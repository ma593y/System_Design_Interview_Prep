# Data Patterns

## Overview

Data patterns address challenges of managing data in distributed systems: consistency, durability, and coordination.

## Write-Ahead Log (WAL)

Record changes to a log before applying them to the database.

```
Write Request
     │
     ▼
┌─────────────────────────────────────────────┐
│           Write-Ahead Log (WAL)              │
│                                              │
│  1. Write to WAL (sequential, fast)         │
│  2. Acknowledge to client                   │
│  3. Apply to database (async)               │
└─────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ WAL File:                                    │
│ [LSN:1 SET x=1] [LSN:2 SET y=2] [LSN:3 ...]  │
└──────────────────────────────────────────────┘
     │
     ▼ (Apply later)
┌─────────────┐
│  Database   │
└─────────────┘

Recovery: Replay WAL from last checkpoint
```

### Benefits

- **Durability**: Data survives crashes
- **Performance**: Sequential writes faster than random
- **Recovery**: Replay log to restore state
- **Replication**: Ship log to replicas

### Use Cases

- PostgreSQL, MySQL (InnoDB)
- Kafka (commit log)
- Distributed databases

## Saga Pattern

Manage distributed transactions without 2PC.

### Choreography Saga

Services coordinate through events.

```
Order Service ──OrderCreated──▶ [Event Bus]
                                     │
                                     ▼
Inventory Service ◀────────── Subscribes
     │
     ├── Success: ──InventoryReserved──▶ [Event Bus]
     │                                       │
     │                                       ▼
     │              Payment Service ◀─── Subscribes
     │                   │
     │                   ├── Success: PaymentProcessed
     │                   │
     │                   └── Failure: PaymentFailed ──▶
     │                                                 │
     └── Receives PaymentFailed ◀──────────────────────┘
         Compensate: Release inventory
```

**Pros:**
- Simple, no central coordinator
- Loosely coupled

**Cons:**
- Hard to track overall progress
- Complex compensation logic

### Orchestration Saga

Central coordinator manages the flow.

```
┌─────────────────────────────────────────────────────────┐
│                  Order Saga Orchestrator                 │
├─────────────────────────────────────────────────────────┤
│  Saga State: STARTED                                    │
│                                                         │
│  Steps:                                                 │
│  1. [✓] Create Order ─────────▶ Order Service          │
│  2. [✓] Reserve Inventory ────▶ Inventory Service      │
│  3. [✗] Process Payment ──────▶ Payment Service        │
│  4. [ ] Ship Order                                      │
│                                                         │
│  Compensation (reverse order):                          │
│  2c. [✓] Release Inventory ───▶ Inventory Service      │
│  1c. [✓] Cancel Order ────────▶ Order Service          │
│                                                         │
│  Saga State: COMPENSATED                                │
└─────────────────────────────────────────────────────────┘
```

**Pros:**
- Centralized control
- Easier to track progress
- Clear compensation logic

**Cons:**
- Single point of failure
- More coupling to orchestrator

### Saga Comparison

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Coupling | Loose | Tighter |
| Complexity | Distributed | Centralized |
| Visibility | Hard to track | Easy to track |
| Scalability | Better | Coordinator bottleneck |

## Two-Phase Commit (2PC)

Atomic commit across distributed nodes.

```
Phase 1: Prepare
┌─────────────┐
│ Coordinator │
└──────┬──────┘
       │ Prepare?
  ┌────┴────┐
  ▼         ▼
┌───────┐ ┌───────┐
│Node A │ │Node B │
│ Vote  │ │ Vote  │
│  Yes  │ │  Yes  │
└───────┘ └───────┘

Phase 2: Commit (if all voted yes)
┌─────────────┐
│ Coordinator │
└──────┬──────┘
       │ Commit
  ┌────┴────┐
  ▼         ▼
┌───────┐ ┌───────┐
│Node A │ │Node B │
│Commit │ │Commit │
└───────┘ └───────┘

If any node votes No or times out → Abort all
```

### 2PC Problems

| Problem | Issue |
|---------|-------|
| **Blocking** | Nodes locked during voting |
| **Coordinator failure** | System stuck if coordinator fails |
| **Not partition tolerant** | Can't handle network partitions |

### When to Use 2PC

- Within a single datacenter
- Strong consistency requirements
- Short-lived transactions
- All participants trusted

## Outbox Pattern

Reliable event publishing with database transactions.

```
Problem: Non-atomic database + message publish

Order Service:
1. Save order to DB ──────── (success)
2. Publish OrderCreated ──── (failure!) ← System crash
                                         Event never published!

Solution: Outbox Pattern

┌─────────────────────────────────────────────────────────┐
│                  Single Transaction                      │
│                                                          │
│  1. INSERT INTO orders (...)                            │
│  2. INSERT INTO outbox (event_data)                     │
│                                                          │
│  COMMIT                                                  │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│              Outbox Publisher (async)                    │
│                                                          │
│  1. Poll outbox table                                   │
│  2. Publish events to message broker                    │
│  3. Mark events as published                            │
└─────────────────────────────────────────────────────────┘
```

### Outbox Table

```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(100),
    event_data JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP
);

-- Insert in same transaction as business data
BEGIN;
  INSERT INTO orders (...);
  INSERT INTO outbox (event_type, event_data)
    VALUES ('OrderCreated', '{"orderId": 123}');
COMMIT;
```

### Change Data Capture (CDC)

Alternative to outbox polling.

```
Database ──CDC (Debezium)──▶ Kafka ──▶ Consumers

CDC reads transaction log, publishes changes as events
No polling, no outbox table needed
```

## Transactional Outbox with CDC

```
┌───────────────────────────────────────────────────────┐
│                      Database                          │
│  ┌─────────────┐    ┌─────────────────────────────┐   │
│  │   Orders    │    │          WAL/Binlog         │   │
│  │   Table     │───▶│ (Transaction Log)           │   │
│  └─────────────┘    └──────────────┬──────────────┘   │
└────────────────────────────────────┼──────────────────┘
                                     │
                              ┌──────▼──────┐
                              │   Debezium  │
                              │    (CDC)    │
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │    Kafka    │
                              └─────────────┘
```

## Idempotency Key Pattern

Ensure operations can be safely retried.

```
Client Request:
POST /payments
Idempotency-Key: pay-abc-123

Server:
┌─────────────────────────────────────────────────────────┐
│  1. Check if key exists                                 │
│     - Yes: Return cached response                       │
│     - No: Continue                                      │
│                                                         │
│  2. Process payment                                     │
│                                                         │
│  3. Store key + response                                │
│                                                         │
│  4. Return response                                     │
└─────────────────────────────────────────────────────────┘

Idempotency Store:
┌────────────────────┬─────────────────────────────────┐
│  pay-abc-123      │  {status: "success", txn: 456}  │
└────────────────────┴─────────────────────────────────┘

Retry with same key → Same response
```

## Optimistic vs Pessimistic Locking

### Optimistic Locking

Assume no conflicts, detect at commit.

```sql
-- Read with version
SELECT id, data, version FROM items WHERE id = 1;
-- version = 5, data = "old"

-- Update with version check
UPDATE items
SET data = 'new', version = 6
WHERE id = 1 AND version = 5;

-- If 0 rows affected → Conflict, retry
```

**Best for:** Low contention, short transactions

### Pessimistic Locking

Lock before reading.

```sql
-- Lock the row
SELECT * FROM items WHERE id = 1 FOR UPDATE;

-- Safely modify
UPDATE items SET data = 'new' WHERE id = 1;

-- Release lock
COMMIT;
```

**Best for:** High contention, data integrity critical

## Soft Delete

Mark records as deleted instead of removing.

```sql
-- Soft delete
UPDATE users SET deleted_at = NOW() WHERE id = 1;

-- Query excludes deleted
SELECT * FROM users WHERE deleted_at IS NULL;

-- Can restore
UPDATE users SET deleted_at = NULL WHERE id = 1;
```

**Benefits:**
- Audit trail
- Recovery possible
- Referential integrity preserved

**Drawbacks:**
- Storage overhead
- Query complexity
- Performance impact

## Interview Talking Points

1. "WAL provides durability and enables replication"
2. "Sagas manage distributed transactions through compensation"
3. "Outbox pattern ensures reliable event publishing"
4. "Optimistic locking for low contention, pessimistic for high"
5. "2PC works but has limitations in distributed systems"

## Common Interview Questions

1. **Q: How do you handle distributed transactions?**
   A: Prefer Saga pattern over 2PC. Choreography for simple flows, orchestration for complex ones.

2. **Q: What is the outbox pattern?**
   A: Write events to an outbox table in the same transaction as business data, then publish asynchronously. Ensures atomic database + event operations.

3. **Q: When would you use optimistic vs pessimistic locking?**
   A: Optimistic for low contention (most updates succeed). Pessimistic when conflicts are common or data integrity is critical.

4. **Q: How does WAL help with recovery?**
   A: All changes logged before applying. On crash, replay log from last checkpoint to restore consistent state.

## Key Takeaways

- WAL is fundamental for durability and replication
- Sagas replace distributed transactions with compensation
- Outbox pattern ensures reliable event publishing
- Choose locking strategy based on contention level
- Design for idempotency in distributed systems

## Further Reading

- "Designing Data-Intensive Applications" Chapters 7, 9
- Saga pattern documentation
- Debezium (CDC) documentation
- PostgreSQL WAL documentation
