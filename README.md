# Inventory-Order-Lifecycle-Automation
This system is implemented as 7 modular n8n workflows that together form a resilient, event-driven order processing pipeline. Workflows communicate via a message queue (Kafka topics), with all persistent state held in a relational database. 
