# CAP Theorem

## Overview

The CAP theorem states that a distributed system can only guarantee two of three properties simultaneously:

- **C**onsistency
- **A**vailability
- **P**artition Tolerance

```
        Consistency
            /\
           /  \
          /    \
         /  CA  \
        /________\
       /\        /\
      /  \  CP  /  \
     / AP \    /    \
    /______\  /______\
Availability ── Partition
                Tolerance
```

## The Three Properties

### Consistency (C)

Every read receives the most recent write or an error.

```
Write x=1 to Node A

Node A (x=1) ──sync──▶ Node B (x=1)
                       Node C (x=1)

All nodes return x=1 at the same time
```

**Strong Consistency:** All nodes see the same data at the same time.

### Availability (A)

Every request receives a response (without guarantee it's the latest).

```
Node A responds: x=1
Node B responds: x=0  ← May be stale
Node C responds: x=1

All respond, even if data differs
```

**High Availability:** System always responds, even if data is stale.

### Partition Tolerance (P)

System continues operating despite network partitions.

```
┌─────────────────────────────────────┐
│           Network Partition          │
│                                      │
│  [Node A]─────X─────[Node B]        │
│     │                   │            │
│     └───────X───────────┘            │
│                                      │
└─────────────────────────────────────┘

Nodes cannot communicate but system still operates
```

## Why "Pick Two" is Misleading

In reality, **partition tolerance is not optional** for distributed systems. Network failures happen.

So the real choice is:

- **CP**: Sacrifice availability during partition
- **AP**: Sacrifice consistency during partition

```
During Partition:

CP System:
Client ─── Request ──▶ Node A
Client ◀── Error ───── Node A  (Cannot reach Node B to confirm)

AP System:
Client ─── Request ──▶ Node A
Client ◀── Response ── Node A  (Returns possibly stale data)
```

## CAP in Practice

### CP Systems (Consistency + Partition Tolerance)

Prefer consistency over availability.

```
Examples:
- Traditional RDBMS (single-node)
- MongoDB (with majority writes)
- HBase
- Redis (in certain configs)
- Zookeeper
- Etcd

Use Cases:
- Financial transactions
- Inventory management
- Coordination services
```

**Behavior during partition:**
- May reject writes
- May become unavailable
- Guarantees data consistency

### AP Systems (Availability + Partition Tolerance)

Prefer availability over consistency.

```
Examples:
- Cassandra
- DynamoDB
- CouchDB
- Riak

Use Cases:
- Social media feeds
- Shopping carts
- User sessions
- DNS
```

**Behavior during partition:**
- Always accepts writes
- Returns possibly stale reads
- Reconciles conflicts later

## PACELC Theorem

Extends CAP to consider behavior during normal operation.

```
If Partition (P):
  Choose Availability (A) or Consistency (C)
Else (E):
  Choose Latency (L) or Consistency (C)
```

### PACELC Classifications

| System | P+A/P+C | E+L/E+C |
|--------|---------|---------|
| Cassandra | PA | EL |
| DynamoDB | PA | EL |
| MongoDB | PC | EC |
| MySQL | PC | EC |
| PostgreSQL | PC | EC |

**PA/EL:** Prioritizes availability and low latency (most NoSQL)
**PC/EC:** Prioritizes consistency always (most SQL)

## Consistency Models

### Strong Consistency

All reads return the latest write.

```
Timeline:
Write x=1 ────▶ All subsequent reads return x=1

Implementation: Synchronous replication, consensus protocols
```

### Eventual Consistency

Given enough time, all reads will return the latest write.

```
Timeline:
Write x=1 ─▶ Read x=0 ─▶ Read x=0 ─▶ Read x=1

There's a window where stale data may be returned
```

### Read-Your-Writes Consistency

A client always sees its own writes.

```
Client A: Write x=1 ─▶ Read x=1 (always sees own write)
Client B: Read x=0 (may see stale data)
```

### Monotonic Reads

Once you read a value, you won't read an older value.

```
Client reads:
x=1 ─▶ x=2 ─▶ x=2 ─▶ x=3  ✓ (always moving forward)
x=1 ─▶ x=2 ─▶ x=1         ✗ (went backward)
```

### Causal Consistency

Causally related operations appear in order.

```
Client A: Write "Hello"
Client A: Write "World"

All clients see "Hello" before "World"
```

## Comparison Table

| Model | Guarantees | Latency | Use Case |
|-------|------------|---------|----------|
| Strong | Latest data always | High | Financial |
| Eventual | Data converges | Low | Social feeds |
| Read-Your-Writes | See own writes | Medium | User sessions |
| Causal | Ordered operations | Medium | Messaging |

## Real-World Trade-offs

### Banking System (CP)

```
Requirement: Account balance must always be accurate

Approach:
- Synchronous replication
- Reject transactions during partition
- Users see errors rather than wrong data
```

### Social Media Feed (AP)

```
Requirement: Always show something

Approach:
- Async replication
- Show potentially stale posts
- Eventually consistent
- Better UX than errors
```

### Shopping Cart (AP with conflict resolution)

```
Requirement: Never lose items

Approach:
- AP for availability
- Merge conflicts (union of items)
- Resolve at checkout
```

## Conflict Resolution Strategies

When AP systems have concurrent writes:

### Last Write Wins (LWW)

Latest timestamp wins.

```
Node A: x=1 at t=100
Node B: x=2 at t=101
Result: x=2 (higher timestamp)
```

**Risk:** May lose data

### Vector Clocks

Track causal history.

```
[A:1] writes x=1
[A:1,B:1] writes x=2
Conflict detected: [A:2] vs [A:1,B:1]
Resolution: Application decides
```

### CRDTs (Conflict-free Replicated Data Types)

Data structures that automatically merge.

```
G-Counter: Only increments, sums all nodes
OR-Set: Union of all adds, tracks removes
```

## Interview Talking Points

1. "CAP is really about choosing between CP and AP during partitions"
2. "Most systems are neither purely CP nor AP - they're tunable"
3. "Strong consistency has latency costs"
4. "Eventual consistency is fine for many use cases"
5. "PACELC extends CAP to consider normal operation trade-offs"

## Common Interview Questions

1. **Q: Explain CAP theorem in simple terms.**
   A: In a distributed system during a network partition, you must choose between availability (always respond) or consistency (always correct data). You can't have both.

2. **Q: Is partition tolerance optional?**
   A: No. Network failures are inevitable. The real choice is between CP and AP behavior during partitions.

3. **Q: How does DynamoDB handle consistency?**
   A: It's AP by default (eventually consistent reads) but offers strongly consistent reads at higher latency cost.

4. **Q: When would you choose consistency over availability?**
   A: Financial systems, inventory management, any system where incorrect data causes real-world problems.

## Key Takeaways

- Partition tolerance is mandatory for distributed systems
- Real choice is between consistency and availability during partitions
- Most modern systems allow tuning this trade-off
- Choose based on business requirements
- Eventual consistency is often acceptable

## Further Reading

- "CAP Twelve Years Later" by Eric Brewer
- "Designing Data-Intensive Applications" Chapter 9
- "Consistency Models" by Aphyr (jepsen.io)
