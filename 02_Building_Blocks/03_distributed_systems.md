# Distributed Systems

## Overview

A distributed system is a collection of independent computers that appears as a single coherent system to users.

```
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Node 1  │──│ Node 2  │──│ Node 3  │──│ Node 4  │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
      │           │           │           │
      └───────────┴───────────┴───────────┘
                      │
              ┌───────▼───────┐
              │ Appears as    │
              │ single system │
              └───────────────┘
```

## Key Challenges

### The Eight Fallacies of Distributed Computing

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

### Fundamental Problems

| Problem | Description |
|---------|-------------|
| **Partial Failure** | Some components fail while others work |
| **No Global Clock** | Nodes can't agree on exact time |
| **Network Partitions** | Nodes can't communicate |
| **Concurrent Operations** | Multiple nodes modify data simultaneously |

## Consensus Algorithms

### The Consensus Problem

Getting all nodes to agree on a value.

```
Node 1: "Value is 5"
Node 2: "Value is 7"
Node 3: "Value is 5"

Consensus: Agree on one value
```

### Paxos

Classic consensus algorithm.

```
Roles:
- Proposer: Proposes values
- Acceptor: Votes on proposals
- Learner: Learns agreed value

Basic Paxos (single value):
1. Prepare: Proposer sends prepare(n) to acceptors
2. Promise: Acceptors promise not to accept lower n
3. Accept: Proposer sends accept(n, value)
4. Accepted: Acceptors accept if n is highest promised
```

### Raft

More understandable consensus algorithm.

```
States: Leader, Follower, Candidate

Normal Operation:
┌────────┐    ┌────────┐    ┌────────┐
│ Leader │───▶│Follower│    │Follower│
│        │───▶│        │    │        │
└────────┘    └────────┘    └────────┘
    │              ▲             ▲
    └──────────────┴─────────────┘
         Replicate log entries

Leader Election:
1. Followers become candidates after timeout
2. Candidate requests votes
3. Majority wins, becomes leader
4. Leader sends heartbeats
```

**Raft Log Replication:**
```
Leader:  [1:A] [2:B] [3:C] [4:D]
                              │
                              ▼ Replicate
Follower: [1:A] [2:B] [3:C] [4:D]
Follower: [1:A] [2:B] [3:C] [4:D]

Commit after majority acknowledgment
```

### Comparison

| Aspect | Paxos | Raft |
|--------|-------|------|
| Understandability | Complex | Simpler |
| Leader | Optional | Required |
| Log | Not specified | Central concept |
| Adoption | Older systems | Newer systems |

## Leader Election

### Why Leaders?

- Simplifies coordination
- Avoids conflicts
- Single decision maker

### Election Algorithms

#### Bully Algorithm

```
When a node detects leader failure:
1. Send election message to all higher-ID nodes
2. If no response, become leader
3. If response, wait for new leader announcement

ID: 1   ID: 2   ID: 3   ID: 4 (Leader fails)
 │       │       │       ✕
 └───────┴───────┘
         │
    Node 3 becomes leader
```

#### Ring Algorithm

```
Nodes in logical ring:
1 → 2 → 3 → 4 → 1

Election message passed around ring
Highest ID in message becomes leader
```

## Distributed Clocks

### Physical Clocks

Problem: Clocks drift, hard to synchronize perfectly.

**NTP (Network Time Protocol):**
- Synchronizes to time servers
- Millisecond accuracy typically
- Not suitable for strict ordering

### Logical Clocks

#### Lamport Clocks

```
Each event gets a logical timestamp:
- Local event: increment counter
- Send message: attach counter
- Receive message: max(local, received) + 1

Node A: 1 → 2 → 3 ─────────────────▶ 4
              │                       ▲
              └─ msg(3) ─▶ Node B: 1 → 4 → 5

If A → B, then LC(A) < LC(B)
But LC(A) < LC(B) doesn't mean A → B
```

#### Vector Clocks

```
Track time per node:
[A:0, B:0, C:0]

Node A event: [A:1, B:0, C:0]
Node B event: [A:0, B:1, C:0]
A sends to B:  B receives [A:1, B:0, C:0]
               B's clock: [A:1, B:2, C:0]

Can detect concurrent events (neither happened before other)
[A:1, B:0] vs [A:0, B:1] = concurrent
```

## Failure Detection

### Heartbeat

```
┌────────┐         ┌────────┐
│ Node A │── ♥ ───▶│ Node B │
└────────┘         └────────┘

If no heartbeat for X seconds → considered failed
```

### Gossip Protocol

```
Node periodically shares membership info with random peers:

Node A: {A:alive, B:alive, C:alive}
        ↓ gossip
Node B: {A:alive, B:alive, C:alive}
        ↓ gossip
Node C: ...

Eventually consistent view of cluster state
```

### Phi Accrual Failure Detector

```
Instead of binary alive/dead:
- Track heartbeat arrival times
- Calculate probability of failure
- Threshold-based decision

More adaptive to network conditions
```

## Quorum

Minimum nodes needed for operation.

```
N = Total nodes
W = Write quorum
R = Read quorum

For strong consistency: W + R > N

Example (N=5, W=3, R=3):
Write to 3 nodes, Read from 3 nodes
At least 1 node has latest value

┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
│ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │
└───┘ └───┘ └───┘ └───┘ └───┘
  W     W     W    (R)   (R)
             (R)
       │           │     │
       └───────────┴─────┘
         Read sees latest write
```

### Quorum Trade-offs

| Configuration | Behavior |
|---------------|----------|
| W=N, R=1 | Slow writes, fast reads |
| W=1, R=N | Fast writes, slow reads |
| W=N/2+1, R=N/2+1 | Balanced, strict consistency |
| W=1, R=1 | Fast, but inconsistent |

## Two-Phase Commit (2PC)

Atomic commits across distributed nodes.

```
Coordinator                 Participants
     │                          │
     │─── Prepare ─────────────▶│ All
     │◀── Vote (Yes/No) ────────│
     │                          │
     │ If all Yes:              │
     │─── Commit ──────────────▶│ All
     │                          │
     │ If any No:               │
     │─── Abort ───────────────▶│ All
```

**Problems:**
- Coordinator failure blocks everything
- Not partition tolerant
- Blocking protocol

## Three-Phase Commit (3PC)

Adds pre-commit phase to avoid blocking.

```
Coordinator                 Participants
     │─── CanCommit? ──────────▶│
     │◀── Vote ─────────────────│
     │─── PreCommit ───────────▶│
     │◀── ACK ──────────────────│
     │─── DoCommit ────────────▶│
```

## Saga Pattern

Long-running transactions using compensating actions.

```
Order Saga:
1. Create Order ─────────▶ (compensate: Cancel Order)
2. Reserve Inventory ────▶ (compensate: Release Inventory)
3. Charge Payment ───────▶ (compensate: Refund Payment)
4. Ship Order

If Step 3 fails:
- Execute compensation for Step 2
- Execute compensation for Step 1
```

## Split Brain

When network partition creates two independent leaders.

```
Network Partition:
┌─────────────┐     ║     ┌─────────────┐
│ Partition A │     ║     │ Partition B │
│  Leader 1   │  ═══╣═══  │  Leader 2   │
│  Nodes 1,2  │     ║     │  Nodes 3,4  │
└─────────────┘     ║     └─────────────┘

Both think they're the leader!
```

**Solutions:**
- Quorum-based decisions (minority can't elect leader)
- Fencing tokens (old leader's tokens rejected)
- STONITH (Shoot The Other Node In The Head)

## Interview Talking Points

1. "Consensus is fundamental - Paxos and Raft are the main algorithms"
2. "Vector clocks help track causality in distributed systems"
3. "Quorums enable consistency without coordinating all nodes"
4. "2PC provides atomicity but is blocking"
5. "Split brain is a critical failure mode to handle"

## Common Interview Questions

1. **Q: What is consensus and why is it hard?**
   A: Getting distributed nodes to agree on a value. Hard because of network failures, node failures, and no global clock.

2. **Q: How does Raft work?**
   A: Leader election via majority vote, leader handles all writes, replicates log entries to followers, commits after majority acknowledgment.

3. **Q: What are vector clocks used for?**
   A: Tracking causality between events. Can determine if events are concurrent or one happened before another.

4. **Q: How do you handle split brain?**
   A: Use quorum (minority partition can't operate), fencing tokens, or STONITH. Design for partition tolerance.

## Key Takeaways

- Distributed systems have unique failure modes
- Consensus algorithms (Raft/Paxos) are fundamental
- No perfect global time - use logical clocks
- Quorums balance availability and consistency
- Always plan for network partitions

## Further Reading

- "Designing Data-Intensive Applications" Chapters 8-9
- Raft paper (In Search of an Understandable Consensus Algorithm)
- Lamport's Time, Clocks paper
- Google Spanner paper
