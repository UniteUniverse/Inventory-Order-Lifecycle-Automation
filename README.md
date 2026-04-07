# Inventory & Order Lifecycle Automation — Architecture Document **Video walkthrough:** [Watch the Loom recording](https://drive.google.com/file/d/18FkknO2luHviZJojAkbs-Wd_cufiyJCQ/view?usp=sharing)

## Overview

This system is implemented as **7 modular n8n workflows** that together form a resilient, event-driven order processing pipeline. Workflows communicate via a message queue (Kafka topics), with all persistent state held in a relational database.

---

## Architecture Principles

| Principle | Implementation |
|-----------|----------------|
| **Idempotency** | Every order checked against a Redis key (`idempotency:{order_id}`) before processing. Duplicate submissions return 409 with no side effects. |
| **Event-driven** | Workflows communicate via Kafka topics, not direct calls. Each stage publishes an event; the next stage subscribes. |
| **No negative stock** | Inventory rows are locked with `SELECT ... FOR UPDATE NOWAIT` before allocation. Decrement only occurs if available qty is sufficient. |
| **Fail-safe state** | Every order has a `status` field. If a workflow crashes mid-run, WF7 catches the error, marks the order `FAILED`, and enqueues it for retry. |
| **Retry with backoff** | HTTP nodes configured with `retryOnFail: true`, `maxTries: 3–5`, `waitBetweenTries: 2000–3000ms`. |
| **Dead-letter queue** | `orders.dead_letter` Kafka topic receives all terminal failures for human review. |
| **Full traceability** | Every state transition appends to the `history` array on the order record. A separate `error_log` table captures all workflow errors. |

---

## Workflow Summary

### WF1 — Order Intake & Validation
**Trigger:** HTTP POST webhook at `/orders`  
**Responsibilities:**
- Validates required fields (`order_id`, `customer.name`, `customer.email`, `items[]`, `priority`)
- Checks Redis for duplicate `order_id` (TTL: 24h)
- Sets idempotency key in Redis (`processing:{timestamp}`)
- Creates initial order record in DB with status `RECEIVED`
- Publishes to `orders.validated` Kafka topic
- Returns `202 Accepted` or `400/409` immediately

**Key design:** The webhook responds immediately after validation — it never waits for fulfillment. This decouples the API response time from all downstream processing.

---

### WF2 — Inventory Allocation
**Trigger:** Kafka consumer on `orders.validated`  
**Responsibilities:**
- Fetches inventory for all requested product IDs with a row-level lock (`SELECT ... FOR UPDATE NOWAIT`)
- Runs a greedy allocation algorithm: fills from the warehouse with highest available stock first
- Prevents negative stock: only decrements up to `available_qty`
- Classifies result as `FULL`, `PARTIAL`, or `BACKORDER`
- Decrements `available_qty` and increments `reserved_qty` atomically
- Publishes to `orders.allocated`

**Multi-warehouse algorithm:**
```
for each item in order:
  remaining = item.quantity
  for each warehouse (sorted by available_qty DESC):
    take = min(remaining, warehouse.available_qty)
    allocate take from this warehouse
    remaining -= take
  if remaining > 0:
    backorder the remainder
```

**Partial fulfillment output:**
- Primary order: contains only the allocated items → goes to dispatch
- Backorder record: `{order_id}-BO` → waits for restock event

---

### WF3 — Order Processor
**Trigger:** Kafka consumer on `orders.allocated`  
**Responsibilities:**
- Routes by `fulfillment_type` using a Switch node:
  - `FULL` → notify customer → mark PROCESSING → publish to `orders.ready_for_dispatch`
  - `PARTIAL` → split into primary order + backorder → notify customer of split → primary goes to dispatch
  - `BACKORDER` → notify customer → persist backorder → wait for restock trigger
- Sends confirmation email via notification service

---

### WF4 — Dispatch & Tracking
**Trigger:** Kafka consumer on `orders.ready_for_dispatch`  
**Responsibilities:**
- Generates tracking ID: `TRK-{timestamp}-{random6}`
- Calls carrier API (retries: 5 attempts, 3s backoff)
- On success: creates tracking record, marks order `DISPATCHED`, notifies customer, publishes `orders.dispatched`
- On failure after all retries: marks order `DISPATCH_FAILED`, publishes to `orders.dead_letter`, alerts Slack ops channel

---

### WF5 — Async Delivery Tracker
**Trigger:** Cron scheduler — every 15 minutes  
**Responsibilities:**
- Queries all active shipments not updated in the last 15 minutes (skips DELIVERED/FAILED/CANCELLED)
- Batches 10 at a time to respect carrier API rate limits
- Maps carrier statuses to internal enum: `IN_TRANSIT`, `OUT_FOR_DELIVERY`, `DELIVERED`, `DELIVERY_FAILED`, `DELIVERY_EXCEPTION`
- Only writes to DB if status has actually changed (avoids unnecessary writes)
- On `DELIVERED`: publishes `orders.delivered`, triggers final customer notification
- Logs all status changes to `tracking_history` for audit

---

### WF6 — SLA Monitor & Escalation
**Trigger:** Cron scheduler — every 30 minutes  
**Responsibilities:**
- Queries orders violating SLA thresholds (defined by priority + current status):

| Priority | Status | SLA Threshold |
|----------|--------|--------------|
| High | RECEIVED | 1 hour |
| High | ALLOCATED / PROCESSING | 2 hours |
| High | DISPATCHED | 24 hours |
| Normal | RECEIVED | 4 hours |
| Normal | ALLOCATED / PROCESSING | 8 hours |
| Normal | DISPATCHED | 72 hours |
| Any | DISPATCH_FAILED | Immediate |
| Any | BACKORDERED | 48 hours |

- Classifies each breach as `CRITICAL` (high priority) or `WARNING` (normal priority)
- Sends per-order Slack alerts with order details and stuck duration
- Logs to `escalation_log` and marks orders `ESCALATED`
- Stores aggregate SLA report per run

---

### WF7 — Global Error Handler
**Trigger:** `errorTrigger` — registered as `errorWorkflow` on WF1–WF5  
**Responsibilities:**
- Parses error context: extracts workflow name, execution ID, last node, order ID (if available)
- Logs error to `error_log` table with unique `error_id`
- If order ID is recoverable: marks order as `FAILED` in DB
- Publishes to `orders.dead_letter` Kafka topic for human review
- Alerts Slack `#errors` channel
- Classifies error as retryable (timeout, rate limit, 5xx) or terminal
- Retryable errors → enqueued in `retry_queue` table for future reprocessing
- Terminal errors → logged and closed

---

## Data Design

### `orders` table
```sql
CREATE TABLE orders (
  order_id          TEXT PRIMARY KEY,
  customer_name     TEXT NOT NULL,
  customer_email    TEXT NOT NULL,
  items             JSONB NOT NULL,        -- [{product_id, quantity}]
  priority          TEXT NOT NULL,         -- 'high' | 'normal'
  status            TEXT NOT NULL,         -- see status enum below
  processing_type   TEXT,                  -- 'FULL' | 'PARTIAL' | 'BACKORDER'
  fulfillment_type  TEXT,
  allocations       JSONB,                 -- [{product_id, warehouse_id, allocated_qty}]
  backorders        JSONB,                 -- [{product_id, quantity_backorder}]
  tracking_id       TEXT,
  invoice_amount    NUMERIC(12,2),
  failure_reason    TEXT,
  error_id          TEXT,
  escalated_at      TIMESTAMPTZ,
  dispatched_at     TIMESTAMPTZ,
  backorder_created_at TIMESTAMPTZ,
  history           JSONB DEFAULT '[]',   -- [{status, ts, note}]
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);
```

### Order Status Enum
```
RECEIVED → VALIDATED → ALLOCATED → PROCESSING → DISPATCHED → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED
                     ↘ PARTIAL_ALLOC          ↗
                     ↘ BACKORDERED           
                                              ↘ DISPATCH_FAILED
                                              ↘ DELIVERY_FAILED
                                              ↘ DELIVERY_EXCEPTION
FAILED (any stage, terminal)
ESCALATED (overlay — order retains its last functional status)
DUPLICATE (terminal, no processing)
CANCELLED (terminal)
```

### `inventory` table
```sql
CREATE TABLE inventory (
  product_id      TEXT NOT NULL,
  warehouse_id    TEXT NOT NULL,
  available_qty   INTEGER NOT NULL CHECK (available_qty >= 0),
  reserved_qty    INTEGER NOT NULL DEFAULT 0,
  total_qty       INTEGER GENERATED ALWAYS AS (available_qty + reserved_qty) STORED,
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (product_id, warehouse_id)
);
```

The `CHECK (available_qty >= 0)` constraint is the database-level guard against negative stock — even if application logic fails.

### `tracking` table
```sql
CREATE TABLE tracking (
  tracking_id        TEXT PRIMARY KEY,
  order_id           TEXT REFERENCES orders(order_id),
  carrier_reference  TEXT,
  status             TEXT NOT NULL DEFAULT 'PENDING',
  carrier_data       JSONB,
  created_at         TIMESTAMPTZ DEFAULT NOW(),
  updated_at         TIMESTAMPTZ DEFAULT NOW()
);
```

### `tracking_history` table
```sql
CREATE TABLE tracking_history (
  id              SERIAL PRIMARY KEY,
  tracking_id     TEXT NOT NULL,
  order_id        TEXT NOT NULL,
  old_status      TEXT,
  new_status      TEXT,
  carrier_data    JSONB,
  recorded_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### `error_log` table
```sql
CREATE TABLE error_log (
  error_id        TEXT PRIMARY KEY,
  workflow_name   TEXT NOT NULL,
  workflow_id     TEXT,
  execution_id    TEXT,
  error_message   TEXT,
  order_id        TEXT,
  is_retryable    BOOLEAN,
  retry_at        TIMESTAMPTZ,
  failed_at       TIMESTAMPTZ DEFAULT NOW(),
  raw_error       TEXT
);
```

### `escalation_log` table
```sql
CREATE TABLE escalation_log (
  id               SERIAL PRIMARY KEY,
  order_id         TEXT NOT NULL,
  status           TEXT,
  priority         TEXT,
  escalation_level TEXT,   -- 'CRITICAL' | 'WARNING'
  escalation_reason TEXT,
  hours_in_status  NUMERIC,
  escalated_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### `retry_queue` table
```sql
CREATE TABLE retry_queue (
  id            SERIAL PRIMARY KEY,
  error_id      TEXT NOT NULL,
  order_id      TEXT,
  workflow_name TEXT NOT NULL,
  retry_at      TIMESTAMPTZ NOT NULL,
  attempted     BOOLEAN DEFAULT FALSE,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis Keys
```
idempotency:{order_id}       → "processing:{ISO timestamp}"  TTL: 86400s
```

### Kafka Topics
```
orders.validated             → WF2 subscribes
orders.allocated             → WF3 subscribes
orders.ready_for_dispatch    → WF4 subscribes
orders.dispatched            → WF5 monitors
orders.delivered             → downstream (e.g. analytics, billing)
orders.backorder             → restock service subscribes
orders.dead_letter           → ops review, manual retry tooling
```

---

## Edge Case Handling

### 1. Duplicate Orders
**Detection:** Redis idempotency key checked in WF1 before any DB write.  
**Response:** HTTP 409 returned immediately. No inventory locked, no order created.  
**TTL:** Key expires after 24 hours — allows legitimate re-submission after a day.

### 2. Partial Fulfillment
**Detection:** WF2 allocation algorithm — remaining quantity > 0 after all warehouses exhausted.  
**Handling:**  
- Primary order ships allocated items immediately.  
- `{order_id}-BO` backorder record created with unfulfilled items.  
- Customer notified of split with estimated restock ETA (if available).  
- Backorder picked up when restock event fires (external trigger → WF2).

### 3. API Failures (Carrier)
**Retry strategy:** 5 attempts with 3-second linear backoff (built into n8n's httpRequest node).  
**After max retries:** Order marked `DISPATCH_FAILED`, published to dead-letter topic, ops alerted on Slack.  
**Recovery path:** Manual ops trigger or automatic retry from `retry_queue` on next scheduled sweep.

### 4. Inventory Changes Mid-Process
**Guard:** `SELECT ... FOR UPDATE NOWAIT` in WF2 locks the specific inventory rows for the duration of the allocation + decrement transaction.  
**Database constraint:** `CHECK (available_qty >= 0)` at the DB level rejects any update that would go negative, regardless of application logic.  
**Race condition:** If two WF2 instances try to lock the same rows simultaneously, one gets `NOWAIT` rejection → falls through to error handler → order retried.

### 5. Orders Stuck in Intermediate States
**Detection:** WF6 SLA Monitor queries `orders` table every 30 minutes for rows violating time thresholds.  
**Escalation:** Slack alert per stuck order. Ops can manually trigger force-advance or investigate.  
**Auto-recovery (optional):** `retry_queue` table can be swept by a scheduled workflow to re-trigger stalled workflows from their last known good state.

### 6. Workflow Crash Mid-Execution
**Coverage:** WF7 registered as `errorWorkflow` on all processing workflows.  
**What it captures:** The last successfully executed node, execution ID, and any order ID in scope.  
**Recovery:** Order marked `FAILED` in DB (so it doesn't stay stuck in a transient status). Retryable errors re-queued; terminal errors logged for manual review.

---

## Scalability Notes

- **WF1** (webhook): scales horizontally with n8n instances behind a load balancer. Redis idempotency key ensures exactly-once processing across instances.
- **WF2/WF3/WF4** (Kafka consumers): multiple consumer group members can run in parallel. Kafka's partition assignment distributes load automatically.
- **WF5/WF6** (scheduled): should run as a single instance. If scaled, add a distributed lock (Redis `SETNX`) at the start to prevent duplicate runs.
- **WF7** (error handler): stateless — scales freely.

---

## Environment Variables Required

```
NOTIFICATION_SERVICE_URL    Base URL for email/SMS notification service
CARRIER_API_URL             Base URL for carrier shipment creation API
CARRIER_TRACKING_API_URL    Base URL for carrier tracking status API
SLACK_OPS_CHANNEL           Slack channel ID for ops alerts
SLACK_ERRORS_CHANNEL        Slack channel ID for error alerts
POSTGRES_*                  Standard n8n Postgres credential
REDIS_*                     Standard n8n Redis credential
KAFKA_*                     Standard n8n Kafka credential
```
