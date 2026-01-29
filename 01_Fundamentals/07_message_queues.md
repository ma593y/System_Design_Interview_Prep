# Message Queues

## Overview

Message queues enable asynchronous communication between services, decoupling producers from consumers.

```
┌──────────┐     ┌─────────────┐     ┌──────────┐
│ Producer │────▶│   Message   │────▶│ Consumer │
│ Service  │     │    Queue    │     │ Service  │
└──────────┘     └─────────────┘     └──────────┘
     │                                     │
     └─ Fire and forget ─────────────────┘
        (Decoupled)
```

## Why Use Message Queues?

| Benefit | Description |
|---------|-------------|
| **Decoupling** | Services don't need to know about each other |
| **Async Processing** | Non-blocking operations |
| **Load Leveling** | Handle traffic spikes |
| **Reliability** | Messages persisted until processed |
| **Scalability** | Add consumers independently |

## Core Concepts

### Messages

Unit of data passed through the queue.

```json
{
  "id": "msg-123",
  "type": "order.created",
  "timestamp": "2024-01-15T10:30:00Z",
  "payload": {
    "orderId": "order-456",
    "userId": "user-789",
    "total": 99.99
  }
}
```

### Producers

Services that send messages to the queue.

### Consumers

Services that receive and process messages.

### Topics/Queues

Logical channels for organizing messages.

## Messaging Patterns

### Point-to-Point (Queue)

One message → One consumer.

```
Producer ──▶ [Queue] ──▶ Consumer 1
                    └──▶ Consumer 2 (competing)
                    └──▶ Consumer 3 (competing)

Each message processed by exactly one consumer
```

**Use Cases:** Task distribution, work queues

### Publish-Subscribe (Pub/Sub)

One message → Multiple consumers.

```
Producer ──▶ [Topic] ──▶ Subscriber 1 (gets copy)
                    ──▶ Subscriber 2 (gets copy)
                    ──▶ Subscriber 3 (gets copy)

Each subscriber gets a copy of every message
```

**Use Cases:** Event broadcasting, notifications

### Fan-Out

Combine queue and pub/sub.

```
                    ┌──▶ [Queue 1] ──▶ Consumer 1a, 1b
Producer ──▶ [Topic]├──▶ [Queue 2] ──▶ Consumer 2a, 2b
                    └──▶ [Queue 3] ──▶ Consumer 3a, 3b

Topic fans out to queues, each queue has competing consumers
```

## Message Delivery Guarantees

### At-Most-Once

Message delivered zero or one time.

```
Producer ──▶ Queue ──▶ Consumer
                        │
                    (Crash before ACK)
                        │
                    Message lost
```

**Pros:** Fastest, no duplicates
**Cons:** May lose messages
**Use Cases:** Metrics, logging

### At-Least-Once

Message delivered one or more times.

```
Producer ──▶ Queue ──▶ Consumer
                        │
                    (Process + Crash before ACK)
                        │
                    Redelivered (duplicate)
```

**Pros:** No message loss
**Cons:** May have duplicates
**Use Cases:** Most applications (with idempotency)

### Exactly-Once

Message delivered exactly one time.

```
Requires:
- Idempotent consumers
- Transaction support
- Deduplication

Most complex to implement
```

**Pros:** Ideal semantics
**Cons:** Complex, performance overhead
**Use Cases:** Financial transactions

## Message Ordering

### FIFO (First In, First Out)

Messages processed in order sent.

```
Send: [A, B, C, D]
Receive: [A, B, C, D]
```

**Challenge:** Hard to maintain with multiple consumers

### Partition-Based Ordering

Order guaranteed within partition.

```
Partition Key: user_id

User 1 messages: [A, B, C] → Always in order
User 2 messages: [X, Y, Z] → Always in order
Across users: No guarantee
```

## Dead Letter Queues (DLQ)

Store messages that fail processing.

```
Main Queue ──▶ Consumer ──(fail 3x)──▶ Dead Letter Queue
                  │                         │
                  └─ Process successfully   └─ Manual review
```

**Use Cases:**
- Debug failed messages
- Prevent blocking main queue
- Alert on processing issues

## Popular Message Queue Systems

### RabbitMQ

Traditional message broker with rich routing.

```
Features:
- AMQP protocol
- Complex routing (exchanges, bindings)
- Multiple protocols
- Management UI

Best for:
- Complex routing needs
- Legacy integration
- Smaller scale
```

### Apache Kafka

Distributed event streaming platform.

```
Features:
- High throughput
- Message replay
- Partitioning
- Consumer groups
- Long retention

Best for:
- Event streaming
- Log aggregation
- High throughput
- Data pipelines
```

### Amazon SQS

Managed queue service.

```
Features:
- Fully managed
- Standard and FIFO queues
- Dead letter queues
- Long polling

Best for:
- AWS ecosystem
- Simple queuing
- Serverless architectures
```

### Amazon SNS

Pub/sub messaging service.

```
Features:
- Pub/sub
- Push to multiple endpoints
- Fan-out to SQS
- Mobile push

Best for:
- Notifications
- Fan-out patterns
- Multi-protocol delivery
```

## Comparison Table

| Feature | RabbitMQ | Kafka | SQS |
|---------|----------|-------|-----|
| Model | Queue | Log | Queue |
| Ordering | FIFO (per queue) | Partition | FIFO (optional) |
| Retention | Until consumed | Time/size based | 14 days max |
| Replay | No | Yes | No |
| Throughput | Medium | Very high | Medium |
| Managed | Self/Cloud | Self/Cloud | AWS only |
| Complexity | Medium | High | Low |

## Kafka Deep Dive

### Architecture

```
┌─────────────────────────────────────────────────┐
│                    Kafka Cluster                 │
│                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │Broker 1 │  │Broker 2 │  │Broker 3 │         │
│  │ P0(L)   │  │ P1(L)   │  │ P2(L)   │         │
│  │ P1(R)   │  │ P2(R)   │  │ P0(R)   │         │
│  └─────────┘  └─────────┘  └─────────┘         │
│                                                  │
│  P = Partition, L = Leader, R = Replica         │
└─────────────────────────────────────────────────┘

Producer ──▶ Partition (based on key) ──▶ Consumer Group
```

### Consumer Groups

```
Topic with 4 partitions:

Consumer Group A:
- Consumer 1: P0, P1
- Consumer 2: P2, P3

Consumer Group B:
- Consumer 1: P0, P1, P2, P3
```

Each partition consumed by one consumer per group.

### Offset Management

```
Partition: [msg0, msg1, msg2, msg3, msg4, msg5]
                              ↑
                        Consumer offset

Consumer tracks position, can replay from any offset
```

## Best Practices

### Message Design

```json
{
  "messageId": "uuid",           // Deduplication
  "timestamp": "ISO8601",        // Ordering context
  "version": "1.0",              // Schema evolution
  "type": "event.type",          // Routing
  "correlationId": "uuid",       // Tracing
  "payload": {}                  // Actual data
}
```

### Idempotency

Make consumers handle duplicates.

```python
def process_message(message):
    if already_processed(message.id):
        return  # Skip duplicate

    do_work(message)
    mark_processed(message.id)
```

### Error Handling

```
Retry Strategy:
1. Immediate retry (transient error)
2. Exponential backoff
3. Move to DLQ after N failures
4. Alert for manual review
```

## Interview Talking Points

1. "Use message queues to decouple services and handle async work"
2. "At-least-once delivery with idempotent consumers is common pattern"
3. "Kafka for event streaming, RabbitMQ for traditional messaging"
4. "Partition by key to maintain ordering per entity"
5. "Dead letter queues prevent blocking on failures"

## Common Interview Questions

1. **Q: When would you use a message queue?**
   A: Async processing, decoupling services, handling traffic spikes, ensuring reliability for background tasks.

2. **Q: How do you handle message ordering?**
   A: Use FIFO queues or partition by key (e.g., user_id). Full ordering limits parallelism.

3. **Q: How do you prevent duplicate processing?**
   A: Make consumers idempotent using message IDs, database constraints, or deduplication stores.

4. **Q: Kafka vs RabbitMQ?**
   A: Kafka for high throughput, event streaming, replay. RabbitMQ for complex routing, lower scale, simpler setup.

## Key Takeaways

- Message queues enable reliable async communication
- Choose delivery guarantee based on requirements
- Design for idempotency to handle duplicates
- Use DLQs to handle failures gracefully
- Partition for ordering while maintaining parallelism

## Further Reading

- "Designing Data-Intensive Applications" Chapter 11
- Kafka documentation
- RabbitMQ tutorials
- AWS SQS/SNS documentation
