# Consistency vs Availability

## Overview

The trade-off between consistency and availability is fundamental to distributed systems, codified in the CAP theorem.

## CAP Theorem Refresher

```
┌─────────────────────────────────────────────────────────────────┐
│                        CAP Theorem                               │
│                                                                  │
│                      Consistency (C)                             │
│                           /\                                    │
│                          /  \                                   │
│                         /    \                                  │
│                        /  CA  \     (CA: Single node,          │
│                       /        \         no partition)          │
│                      /──────────\                               │
│                     /     CP     \                              │
│                    /              \                             │
│                   /                \                            │
│     Availability ────────AP────────── Partition                 │
│         (A)                            Tolerance (P)            │
│                                                                  │
│  During network partition, choose:                              │
│  • CP: Consistency (reject requests)                            │
│  • AP: Availability (allow stale reads)                         │
└─────────────────────────────────────────────────────────────────┘
```

## The Trade-off Explained

### Strong Consistency (CP)

All nodes see the same data at the same time.

```
Write: x = 5

  Client                    Nodes                         Client
     │                                                       │
     │ ──── Write x=5 ────▶ ┌─────────┐                     │
     │                      │ Node 1  │ x = 5               │
     │                      │ Leader  │────┐                │
     │                      └─────────┘    │                │
     │                                      │ Replicate     │
     │                      ┌─────────┐    │                │
     │                      │ Node 2  │◀───┘                │
     │                      │ x = 5   │                     │
     │                      └─────────┘                     │
     │                                                       │
     │                      ┌─────────┐                     │
     │                      │ Node 3  │ x = 5               │
     │                      └─────────┘                     │
     │                                                       │
     │ ◀─── ACK (after all replicate) ──────────────────────│
     │                                                       │
     │                                 Read x ──────────────▶│
     │                                 Returns: 5 (always)   │
```

**Behavior during partition:**
```
Network Partition:
┌─────────┐         ╳         ┌─────────┐
│ Node 1  │ ────────╳──────── │ Node 2  │
│         │         ╳         │         │
└─────────┘                   └─────────┘

CP System Response:
- Node 1: "Cannot guarantee consistency, rejecting writes"
- Node 2: "Cannot guarantee consistency, rejecting writes"
- Clients: Error responses (unavailable)
```

### High Availability (AP)

System always responds, even if data might be stale.

```
Write: x = 5 (during partition)

  Client A                                           Client B
     │                                                   │
     │ ── Write x=5 ─▶ ┌─────────┐                      │
     │                 │ Node 1  │ x = 5                │
     │                 └─────────┘                      │
     │                       ╳                          │
     │                       ╳ Partition                │
     │                       ╳                          │
     │                 ┌─────────┐ ◀── Read x ──────────│
     │                 │ Node 2  │ x = 0 (stale!)       │
     │                 └─────────┘ ──▶ Returns: 0       │
     │                                                   │
     │ ◀─── ACK ──────                                  │

AP System: Both clients get responses, but different values
```

## Consistency Models

### Spectrum of Consistency

```
Strong                                                      Weak
Consistency ◀───────────────────────────────────────▶ Consistency

  │           │              │              │              │
  ▼           ▼              ▼              ▼              ▼
┌────────┐ ┌────────┐ ┌──────────────┐ ┌────────────┐ ┌────────┐
│Lineariz│ │Sequent-│ │   Causal     │ │  Session   │ │Eventual│
│ able   │ │  ial   │ │ Consistency  │ │ Consistency│ │Consist.│
└────────┘ └────────┘ └──────────────┘ └────────────┘ └────────┘
   Most        │             │              │           Least
   Latency     │             │              │           Latency
```

### 1. Strong/Linearizable Consistency

```
Every read sees the most recent write.

Time ────────────────────────────────────────▶

Client A:  Write(x=1) ───────────────────────▶
                    ↓
Client B:           ├── Read(x) → 1 (always)

No matter which node you read from, see latest value.
```

**Examples:** ZooKeeper, etcd, Spanner

### 2. Sequential Consistency

```
All clients see operations in same order (not necessarily real-time).

Time ────────────────────────────────────────▶

Client A:  Write(x=1) ──────────────────────▶
Client B:       Write(y=2) ─────────────────▶
Client C:                    Read(x)=1, Read(y)=2
Client D:                    Read(x)=1, Read(y)=2

All see same order, but might not reflect real-time order.
```

### 3. Causal Consistency

```
Causally related operations seen in order.

Client A:  Write(x=1) ──▶ Write(y=2) ──────▶
                 │              │
                 │    (y=2 depends on x=1)
                 ▼              ▼
Client B:  Read(x)=1 ───▶ Read(y)=2 ✓

If B sees y=2, B must have seen x=1 (causal order preserved).
```

**Examples:** MongoDB, CockroachDB

### 4. Eventual Consistency

```
Given enough time, all nodes converge.

Time ────────────────────────────────────────▶

Write(x=1) to Node 1
     │
     ├── Node 1: x=1
     │   Node 2: x=0 (stale)      ← Inconsistent window
     │   Node 3: x=0 (stale)
     │
     │... replication ...
     │
     └── Node 1: x=1
         Node 2: x=1              ← Eventually consistent
         Node 3: x=1
```

**Examples:** DynamoDB, Cassandra (configurable)

## PACELC Theorem

Extension of CAP that considers latency trade-offs.

```
┌─────────────────────────────────────────────────────────────────┐
│                         PACELC                                   │
│                                                                  │
│  If there's a Partition (P):                                    │
│    Choose between Availability (A) and Consistency (C)          │
│                                                                  │
│  Else (E):                                                       │
│    Choose between Latency (L) and Consistency (C)               │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  System          │ Partition  │ Normal Operation          │  │
│  ├───────────────────┼────────────┼───────────────────────────┤  │
│  │  DynamoDB        │ PA         │ EL (low latency)          │  │
│  │  Cassandra       │ PA         │ EL (tunable)              │  │
│  │  MongoDB         │ PC         │ EC (single primary)       │  │
│  │  Spanner         │ PC         │ EC (strong consistency)   │  │
│  │  CockroachDB     │ PC         │ EC (serializable)         │  │
│  └───────────────────┴────────────┴───────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## When to Choose What

### Choose Strong Consistency (CP)

```
✅ Financial transactions
   "Transfer $100 from A to B" - must not lose money

✅ Inventory systems
   "Only 1 item left" - must not oversell

✅ Leader election
   "Only one leader at a time" - must not have split-brain

✅ Distributed locks
   "Only one process holds lock" - must be exclusive

✅ Configuration management
   "All nodes use same config" - must be synchronized
```

### Choose High Availability (AP)

```
✅ Social media feeds
   "Show latest posts" - stale is OK briefly

✅ Product catalogs
   "Show product info" - cached data acceptable

✅ Analytics/metrics
   "Show visitor count" - approximate is fine

✅ User sessions
   "Keep user logged in" - availability critical

✅ DNS
   "Resolve domain name" - cached OK, must respond
```

## Strategies to Handle Trade-offs

### 1. Read-Your-Writes Consistency

```
User writes, then immediately reads their own write.

Write Request:
Client ──▶ Primary ──▶ Write to DB
               │
               └──▶ Route subsequent reads to same node
                    OR
                    Wait for replication before ACK

Result: User always sees their own writes.
```

### 2. Quorum Reads/Writes

```
N = Total replicas
W = Write quorum
R = Read quorum

Strong consistency if: W + R > N

Example: N=3, W=2, R=2

Write (W=2):
┌─────────┐
│ Node 1  │ ◀── Write ✓
│ (ACK)   │
└─────────┘
┌─────────┐
│ Node 2  │ ◀── Write ✓  ──▶ Return ACK (2 nodes)
│ (ACK)   │
└─────────┘
┌─────────┐
│ Node 3  │ ◀── (pending)
│         │
└─────────┘

Read (R=2):
Must read from 2 nodes, guaranteed to hit at least 1 with latest data.
```

### 3. Conflict Resolution

```
For AP systems, handle conflicts when they occur:

1. Last-Write-Wins (LWW):
   Timestamp: 10:00:01 - x = 5
   Timestamp: 10:00:02 - x = 7  ← Winner

2. Vector Clocks:
   Track causality, detect conflicts
   [A:1, B:0] vs [A:0, B:1] = Conflict → Application resolves

3. CRDTs (Conflict-free Replicated Data Types):
   Data structures that automatically merge
   Counter: {A: +5, B: +3} → Total: 8
```

### 4. Saga Pattern for Distributed Transactions

```
Instead of distributed ACID, use compensating transactions:

┌──────────────────────────────────────────────────────────────┐
│                    Order Saga                                 │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  T1: Reserve Inventory ──▶ T2: Charge Payment ──▶ T3: Ship   │
│          │                        │                    │      │
│          │                        │ (if fail)          │      │
│          ▼                        ▼                    ▼      │
│  C1: Release Inventory ◀── C2: Refund Payment  (compensate) │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

## Real-World Examples

| System | Consistency Choice | Rationale |
|--------|-------------------|-----------|
| Banking | Strong (CP) | Cannot lose transactions |
| Twitter Timeline | Eventual (AP) | Slight delay acceptable |
| Stock Trading | Strong (CP) | Price accuracy critical |
| Netflix Streaming | Eventual (AP) | Availability over perfect sync |
| Google Docs | Causal | Collaborative editing |
| Shopping Cart | Eventual (AP) | Must always be accessible |
| Inventory | Strong (CP) | Cannot oversell |

## Interview Talking Points

1. "CAP is about trade-offs during network partitions - you can't have both perfect consistency and availability"

2. "Most systems are on a spectrum - you don't have to choose binary"

3. "Use strong consistency for financial transactions, eventual consistency for social feeds"

4. "Quorum systems let you tune the consistency-availability balance"

5. "PACELC extends CAP - even without partitions, there's latency vs consistency"

## Common Interview Questions

1. **Q: Explain CAP theorem with an example.**
   A: During network partition, a banking system (CP) rejects transactions to prevent inconsistency, while a social media feed (AP) serves potentially stale data to remain available.

2. **Q: How do you ensure consistency in a distributed system?**
   A: Quorum reads/writes (W+R>N), synchronous replication, consensus protocols (Paxos, Raft), or distributed transactions with 2PC.

3. **Q: What is eventual consistency?**
   A: All replicas will eventually converge to the same value given no new updates. Temporary inconsistency is allowed. Used when availability and performance are prioritized.

4. **Q: How would you design a system that needs both?**
   A: Different consistency levels for different operations. Strong consistency for checkout/payment, eventual consistency for product catalog display.

## Key Takeaways

- Network partitions are inevitable - plan for them
- Choose consistency model based on business requirements
- Most systems use a mix of consistency levels
- Quorum systems offer tunable consistency
- Design for eventual consistency where possible (more scalable)

## Further Reading

- "Designing Data-Intensive Applications" - Chapters 5, 7, 9
- Google Spanner paper (strong consistency at scale)
- Dynamo paper (eventual consistency)
- Jepsen.io (consistency testing)
