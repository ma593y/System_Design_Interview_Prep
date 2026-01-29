# Replication

## Overview

Replication creates copies of data across multiple nodes for fault tolerance, availability, and performance.

```
┌──────────────┐
│   Primary    │
│    Node      │
└──────┬───────┘
       │ Replicate
  ┌────┴────────────────┐
  ▼         ▼           ▼
┌──────┐  ┌──────┐  ┌──────┐
│Replica│  │Replica│  │Replica│
│  1    │  │  2    │  │  3    │
└──────┘  └──────┘  └──────┘

Same data on multiple nodes
```

## Why Replicate?

| Benefit | Description |
|---------|-------------|
| **Fault Tolerance** | Survive node failures |
| **Availability** | System stays up during failures |
| **Read Scalability** | Distribute read load |
| **Geographic** | Data closer to users |

## Replication Strategies

### Single-Leader (Master-Slave)

One leader handles all writes, replicas handle reads.

```
        Writes
           │
           ▼
     ┌──────────┐
     │  Leader  │
     │ (Primary)│
     └────┬─────┘
          │ Replicate
    ┌─────┴─────────┐
    ▼       ▼       ▼
┌──────┐ ┌──────┐ ┌──────┐
│Replica│ │Replica│ │Replica│
└───┬──┘ └───┬──┘ └───┬──┘
    │        │        │
    ▼        ▼        ▼
  Reads    Reads    Reads
```

**Pros:**
- Simple to understand
- No write conflicts
- Easy to implement

**Cons:**
- Single point of failure for writes
- Replication lag for reads
- Failover complexity

**Examples:** PostgreSQL, MySQL, MongoDB

### Multi-Leader (Master-Master)

Multiple leaders can accept writes.

```
  Writes      Writes      Writes
     │           │           │
     ▼           ▼           ▼
┌────────┐  ┌────────┐  ┌────────┐
│Leader 1│◀─│Leader 2│─▶│Leader 3│
│  (DC1) │  │  (DC2) │  │  (DC3) │
└────────┘  └────────┘  └────────┘
     ▲           ▲           ▲
     └───────────┴───────────┘
         Sync changes
```

**Pros:**
- Multi-datacenter writes
- Lower latency for local writes
- Better availability

**Cons:**
- Write conflicts possible
- Complex conflict resolution
- Harder to reason about

**Use Cases:** Multi-datacenter deployments, offline-capable apps

### Leaderless (Peer-to-Peer)

Any node can accept reads and writes.

```
     Write (W=2)
         │
    ┌────┴────┐
    ▼         ▼
┌──────┐  ┌──────┐  ┌──────┐
│Node 1│  │Node 2│  │Node 3│
│  ✓   │  │  ✓   │  │  ✗   │
└──────┘  └──────┘  └──────┘
    ▲         ▲
    └────┬────┘
         │
     Read (R=2)

W + R > N ensures reading latest value
```

**Pros:**
- No single point of failure
- High availability
- Good for write-heavy workloads

**Cons:**
- Eventual consistency
- Need conflict resolution
- More complex reads

**Examples:** Cassandra, DynamoDB, Riak

## Comparison

| Aspect | Single-Leader | Multi-Leader | Leaderless |
|--------|---------------|--------------|------------|
| Write complexity | Simple | Moderate | Complex |
| Conflicts | None | Possible | Possible |
| Availability | Medium | High | Highest |
| Consistency | Strong possible | Eventual | Eventual |
| Latency | Leader bottleneck | Local leader | Any node |

## Synchronous vs Asynchronous

### Synchronous Replication

```
Client ──▶ Leader ──▶ Replica ACK ──▶ Response
              │            ▲
              └────────────┘
           Wait for replica

+ Strong consistency
+ Durability guarantee
- Higher latency
- Lower availability (replica failure blocks)
```

### Asynchronous Replication

```
Client ──▶ Leader ──▶ Response
              │
              └──▶ Replica (later)

+ Lower latency
+ Higher availability
- Replication lag
- Potential data loss
```

### Semi-Synchronous

```
Client ──▶ Leader ──▶ At least 1 replica ACK ──▶ Response
              │
              └──▶ Other replicas (async)

Balance of consistency and performance
```

## Replication Lag

Time between write to leader and propagation to replicas.

```
Timeline:
Leader:  Write x=1 ──────────────────────▶
Replica: ────────── Lag ──── Receives x=1

Problems:
- Read-your-writes: User writes, reads from replica, doesn't see own write
- Monotonic reads: User reads from different replicas, sees time go backward
- Consistent prefix: Causally related writes appear out of order
```

### Read-Your-Writes Consistency

```
Solutions:
1. Read from leader for own data
2. Remember write timestamp, read from replica only if caught up
3. Route to same replica for session

User writes x=1 to leader
User reads from replica where x=1 is already replicated
```

### Monotonic Reads

```
Ensure user always reads from same replica (or more up-to-date):

User: Read from Replica 1 (x=1)
User: Next read from Replica 1 (x=2)  ✓
NOT:  Read from Replica 2 (x=0)      ✗ (went backward)
```

## Conflict Resolution

When multi-leader or leaderless systems have concurrent writes.

### Last Write Wins (LWW)

```
Node 1: Write x=1 at t=100
Node 2: Write x=2 at t=101
Resolution: x=2 (higher timestamp wins)

Simple but may lose data
```

### Merge Values

```
Shopping cart conflict:
Node 1: cart = [A, B]
Node 2: cart = [A, C]
Resolution: cart = [A, B, C] (union)
```

### Version Vectors

```
Track causality per node:
[A:1, B:0] vs [A:0, B:1]

These are concurrent - can't determine order
Application must resolve (show conflict to user)
```

### CRDTs

Conflict-free Replicated Data Types - mathematically guaranteed to merge.

```
G-Counter (grow-only counter):
Node A: {A:5}
Node B: {B:3}
Merge: {A:5, B:3} = total 8

OR-Set (observed-remove set):
Track adds and removes with unique IDs
Merge correctly handles concurrent add/remove
```

## Failover

### Automatic Failover

```
1. Detect leader failure (heartbeat timeout)
2. Elect new leader (consensus among replicas)
3. Reconfigure clients to use new leader
4. Old leader demoted when it recovers

Leader fails ──▶ Detection ──▶ Election ──▶ Promotion
     │                            │
     ▼                            ▼
  Timeout                    Replica 2
  (30 sec)                  becomes leader
```

### Failover Challenges

| Issue | Description |
|-------|-------------|
| **Split Brain** | Two nodes think they're leader |
| **Data Loss** | Async replica missing recent writes |
| **Inconsistency** | Old leader's uncommitted writes |
| **Cascade** | Overloaded new leader fails too |

### Fencing

Prevent old leader from accepting writes after failover.

```
Fencing tokens:
- New leader gets token 34
- Old leader had token 33
- Storage rejects writes with token < 34

Old leader    New leader
Token 33      Token 34
   │             │
   └──▶ Write ───│──▶ Storage: Reject token 33
                 │
            Accept token 34
```

## Replication Topologies

### Star (Hub and Spoke)

```
        ┌────────┐
        │ Leader │
        └───┬────┘
      ┌─────┼─────┐
      ▼     ▼     ▼
    ┌───┐ ┌───┐ ┌───┐
    │ R1│ │ R2│ │ R3│
    └───┘ └───┘ └───┘
```

### Circular

```
  ┌───┐
  │ A │───┐
  └───┘   │
    ▲     ▼
    │   ┌───┐
    │   │ B │
    │   └───┘
    │     │
  ┌───┐   ▼
  │ D │◀─┌───┐
  └───┘  │ C │
         └───┘
```

### All-to-All

```
  ┌───┐◀──────▶┌───┐
  │ A │        │ B │
  └───┘        └───┘
    ▲ \      / ▲
    │  \    /  │
    │   \  /   │
    ▼    \/    ▼
  ┌───┐  /\  ┌───┐
  │ D │ /  \ │ C │
  └───┘◀────▶└───┘

Most fault tolerant but complex
```

## Interview Talking Points

1. "Single-leader is simplest but limits write scalability"
2. "Async replication has lag but better performance"
3. "Leaderless requires quorum for consistency"
4. "LWW is simple but may lose data"
5. "Failover must handle split brain and data loss"

## Common Interview Questions

1. **Q: When would you use multi-leader replication?**
   A: Multi-datacenter deployments where you need low-latency writes in each location. Also for offline-capable apps that sync later.

2. **Q: How do you handle replication lag?**
   A: Read-your-writes (read from leader for own data), monotonic reads (same replica per session), or accept eventual consistency.

3. **Q: What is the quorum formula?**
   A: W + R > N ensures overlap. W=write quorum, R=read quorum, N=replicas. Trade-off between consistency and availability.

4. **Q: How do you handle conflicts in multi-leader systems?**
   A: LWW (simple, loses data), merge values (domain-specific), version vectors (detect conflicts), or CRDTs (automatic merge).

## Key Takeaways

- Replication provides fault tolerance and read scalability
- Choose strategy based on consistency and availability needs
- Understand trade-offs between sync and async replication
- Plan for failover scenarios
- Have a conflict resolution strategy

## Further Reading

- "Designing Data-Intensive Applications" Chapter 5
- Cassandra architecture documentation
- PostgreSQL replication documentation
- CRDTs papers and implementations
