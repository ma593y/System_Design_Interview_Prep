# Event-Driven Architecture

## Overview

Event-driven architecture (EDA) is a software design pattern where the flow of the program is determined by events.

```
Traditional:                Event-Driven:
┌─────────┐                ┌─────────┐
│Service A│──call──▶│Service B│     │Service A│──event──▶[Event Bus]
└─────────┘                └─────────┘                         │
                                           ┌───────────────────┼───────────────────┐
                                           ▼                   ▼                   ▼
                                      ┌─────────┐         ┌─────────┐        ┌─────────┐
                                      │Service B│         │Service C│        │Service D│
                                      └─────────┘         └─────────┘        └─────────┘

Producer doesn't know consumers, fully decoupled
```

## Core Concepts

### Events

Something that happened in the system.

```json
{
  "eventId": "evt-123",
  "eventType": "OrderCreated",
  "timestamp": "2024-01-15T10:30:00Z",
  "source": "order-service",
  "data": {
    "orderId": "order-456",
    "userId": "user-789",
    "items": [...],
    "total": 99.99
  }
}
```

### Event Types

| Type | Description | Example |
|------|-------------|---------|
| **Domain Event** | Business occurrence | OrderPlaced, PaymentReceived |
| **Integration Event** | Cross-service communication | UserCreated |
| **Notification Event** | Alerts/notifications | LowInventoryWarning |

### Producers and Consumers

```
┌──────────────┐          ┌──────────────┐
│   Producer   │──event──▶│  Event Bus   │
│ (emits event)│          │              │
└──────────────┘          └──────┬───────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │Consumer 1│ │Consumer 2│ │Consumer 3│
              └──────────┘ └──────────┘ └──────────┘
```

## Patterns

### Publish-Subscribe (Pub/Sub)

Multiple consumers receive the same event.

```
OrderService ──▶ OrderCreated ──▶ [Topic]
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
              EmailService    InventoryService    AnalyticsService
              (send receipt)  (reserve stock)     (track metrics)
```

### Event Sourcing

Store state as a sequence of events.

```
Traditional:                   Event Sourcing:
┌────────────────────┐        ┌────────────────────────────────────┐
│ Account            │        │ Event Store                        │
│ balance: $500      │        │ 1. AccountOpened($0)              │
└────────────────────┘        │ 2. Deposited($1000)               │
                              │ 3. Withdrawn($300)                 │
Current state only            │ 4. Withdrawn($200)                 │
                              │ Current: $500 (computed from events)
                              └────────────────────────────────────┘

Full history, can replay to any point
```

**Benefits:**
- Complete audit trail
- Time travel (reconstruct past state)
- Event replay for debugging
- Natural fit for EDA

**Challenges:**
- Event schema evolution
- Storage size
- Query complexity

### CQRS (Command Query Responsibility Segregation)

Separate read and write models.

```
                    ┌──────────────────────────────────────────┐
                    │                 Commands                  │
                    │                                          │
Commands ──────────▶│  ┌─────────────┐    ┌──────────────┐   │
                    │  │   Command   │───▶│  Event Store │   │
                    │  │   Handler   │    └──────┬───────┘   │
                    │  └─────────────┘           │           │
                    └────────────────────────────┼───────────┘
                                                 │
                                          Events │
                                                 ▼
                    ┌────────────────────────────────────────┐
                    │                 Queries                 │
                    │                                        │
Queries ───────────▶│  ┌──────────────┐    ┌──────────────┐ │
                    │  │ Query Handler│◀───│  Read Model  │ │
                    │  └──────────────┘    │  (Optimized) │ │
                    │                      └──────────────┘ │
                    └────────────────────────────────────────┘
```

**Benefits:**
- Optimized read models
- Independent scaling
- Complex queries on denormalized data

**Challenges:**
- Eventual consistency
- Increased complexity

### Event Sourcing + CQRS Combined

```
Commands ──▶ Command Handler ──▶ Event Store
                                     │
                                 Events
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
            ┌────────────┐   ┌────────────┐   ┌────────────┐
            │ Read Model │   │ Read Model │   │ Read Model │
            │  (List)    │   │  (Search)  │   │ (Reports)  │
            └────────────┘   └────────────┘   └────────────┘
                  ▲                ▲                ▲
                  │                │                │
               Queries          Queries          Queries

Multiple read models optimized for different queries
```

### Saga Pattern

Manage distributed transactions through events.

```
Choreography Saga (Event-based):

Order ──OrderCreated──▶ Inventory ──StockReserved──▶ Payment
  ▲                         │                           │
  │                         │                           │
  └──OrderCancelled◀────────┘                           │
         (if fails)         ◀──PaymentFailed────────────┘

Each service reacts to events and emits new events
```

```
Orchestration Saga (Coordinator-based):

┌─────────────────────────────────────────────────────┐
│                  Saga Orchestrator                   │
├─────────────────────────────────────────────────────┤
│  1. Create Order                                    │
│  2. Reserve Inventory ──▶ Inventory Service         │
│  3. Process Payment ───▶ Payment Service            │
│  4. Ship Order ────────▶ Shipping Service           │
│                                                      │
│  On failure: Execute compensating actions           │
└─────────────────────────────────────────────────────┘
```

## Message Brokers

### Apache Kafka

```
┌─────────────────────────────────────────────────────┐
│                    Kafka Cluster                     │
│                                                      │
│  Topic: orders                                       │
│  ┌───────────────────────────────────────────────┐  │
│  │ Partition 0: [msg1][msg2][msg3][msg4]         │  │
│  │ Partition 1: [msg5][msg6][msg7]               │  │
│  │ Partition 2: [msg8][msg9][msg10][msg11]       │  │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  Consumer Groups:                                    │
│  - Group A: Consumer1(P0), Consumer2(P1,P2)         │
│  - Group B: Consumer3(P0,P1,P2)                     │
└─────────────────────────────────────────────────────┘

Partitioned, replicated, high-throughput
```

### RabbitMQ

```
┌─────────────────────────────────────────────────────┐
│                    RabbitMQ                          │
│                                                      │
│  Exchange ──routing──▶ Queue 1 ──▶ Consumer 1       │
│     │                                                │
│     ├──routing──▶ Queue 2 ──▶ Consumer 2            │
│     │                                                │
│     └──routing──▶ Queue 3 ──▶ Consumer 3            │
│                                                      │
│  Exchange Types: Direct, Fanout, Topic, Headers      │
└─────────────────────────────────────────────────────┘

Flexible routing, traditional message broker
```

### Comparison

| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| Model | Log (append-only) | Queue (remove on consume) |
| Retention | Configurable (days/size) | Until consumed |
| Replay | Yes | No |
| Ordering | Per partition | Per queue |
| Throughput | Very high | High |
| Routing | Basic (partitions) | Flexible (exchanges) |

## Event Design Best Practices

### Event Structure

```json
{
  "metadata": {
    "eventId": "uuid",
    "eventType": "OrderCreated",
    "version": "1.0",
    "timestamp": "ISO8601",
    "correlationId": "trace-id",
    "source": "order-service"
  },
  "payload": {
    // Domain-specific data
  }
}
```

### Event Versioning

```
Version 1:
{
  "type": "UserCreated",
  "data": { "name": "Alice" }
}

Version 2 (added field):
{
  "type": "UserCreated",
  "data": { "name": "Alice", "email": "alice@mail.com" }
}

Strategies:
- Add optional fields (backward compatible)
- Create new event type for breaking changes
- Use schema registry
```

### Idempotency

```
Consumer must handle duplicate events:

// Bad: Creates duplicate orders
def handle_order_created(event):
    create_invoice(event.orderId)

// Good: Idempotent handling
def handle_order_created(event):
    if not invoice_exists(event.orderId):
        create_invoice(event.orderId)

Use event ID for deduplication
```

## Eventual Consistency

```
Timeline:
Write ────────────▶ Event ────────────▶ Read Model Update
                                              │
                              ┌───────────────┘
                              ▼
                    Query may see stale data
                    until read model catches up

Solutions:
- Accept eventual consistency
- UI shows "processing" status
- Optimistic UI updates
- Polling for updates
```

## Interview Talking Points

1. "Events decouple producers from consumers"
2. "Event sourcing provides complete audit trail"
3. "CQRS optimizes reads and writes independently"
4. "Sagas handle distributed transactions without 2PC"
5. "Kafka for high throughput, RabbitMQ for flexible routing"

## Common Interview Questions

1. **Q: What is event sourcing?**
   A: Storing state as a sequence of events rather than current state. Enables audit trail, time travel, and event replay.

2. **Q: How do you handle event ordering?**
   A: Use partitioning by key (same entity goes to same partition). Accept out-of-order across partitions.

3. **Q: How do you ensure events are processed exactly once?**
   A: At-least-once delivery + idempotent consumers. Use event IDs for deduplication.

4. **Q: When would you use CQRS?**
   A: When read and write models have different requirements, need multiple optimized read views, or have complex queries.

## Key Takeaways

- Events represent facts that happened
- Pub/Sub decouples producers and consumers
- Event sourcing stores state as event history
- CQRS separates read and write models
- Design for eventual consistency and idempotency

## Further Reading

- "Designing Event-Driven Systems" by Ben Stopford
- Martin Fowler's CQRS and Event Sourcing articles
- Kafka documentation
- Event Storming methodology
