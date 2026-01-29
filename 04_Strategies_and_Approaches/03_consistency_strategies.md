# Consistency Strategies

## Overview

Consistency in distributed systems refers to how and when updates made by one operation become visible to other operations. Understanding consistency models is critical for system design because they fundamentally affect user experience, system complexity, and performance. The CAP theorem tells us we can't have perfect consistency, availability, and partition tolerance simultaneously - so choosing the right consistency model is about understanding trade-offs.

## Key Concepts

### The Consistency Spectrum

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Consistency Spectrum                                │
│                                                                          │
│  Strongest                                                   Weakest    │
│     │                                                           │        │
│     ▼                                                           ▼        │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────────┐        │
│  │Lineariz│  │Sequent-│  │ Causal │  │ Read   │  │  Eventual  │        │
│  │-able   │  │ial     │  │        │  │ Your   │  │            │        │
│  │        │  │        │  │        │  │ Writes │  │            │        │
│  └────────┘  └────────┘  └────────┘  └────────┘  └────────────┘        │
│                                                                          │
│  ◄─────────── Higher Latency ──────────────────── Lower Latency ──────► │
│  ◄─────────── Lower Throughput ────────────────── Higher Throughput ──► │
│  ◄─────────── Simpler Reasoning ───────────────── Complex Reasoning ──► │
└─────────────────────────────────────────────────────────────────────────┘
```

### CAP Theorem Refresher

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CAP Theorem                                     │
│                                                                          │
│                         Consistency                                      │
│                             /\                                           │
│                            /  \                                          │
│                           /    \                                         │
│                          / CP   \                                        │
│                         /  Systems\                                      │
│                        /   (HBase, \                                     │
│                       /    Spanner) \                                    │
│                      /_______________\                                   │
│                     /        |        \                                  │
│                    /    CA   |   AP    \                                 │
│       Availability/  Systems | Systems  \ Partition                     │
│                  / (Single   |(Cassandra,\ Tolerance                    │
│                 /   Node DB) | DynamoDB)  \                              │
│                                                                          │
│  During a partition, you must choose:                                   │
│  - CP: Reject requests to maintain consistency                          │
│  - AP: Accept requests, allow inconsistency                             │
│                                                                          │
│  Note: CA is only possible without network partitions (single node)     │
└─────────────────────────────────────────────────────────────────────────┘
```

## Strong Consistency

### Definition and Guarantees

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Strong Consistency (Linearizability)                  │
│                                                                          │
│  Guarantee: All operations appear to execute atomically in some         │
│  sequential order that is consistent with the real-time ordering        │
│  of those operations.                                                    │
│                                                                          │
│  Timeline:                                                               │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  Client A:  ───[Write X=1]───────────────────────────────────────────   │
│                      │                                                   │
│  Client B:  ─────────┼───[Read X]───────────────────────────────────    │
│                      │       │                                           │
│                      │       └─► Must return 1 (or block until write    │
│                      │           completes)                              │
│                      │                                                   │
│  Global Order:  Write(X=1) → Read(X) returns 1                          │
│                                                                          │
│  Real-world analogy: A single-threaded program with one variable        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementing Strong Consistency

```
Method 1: Single Leader Replication (Synchronous)

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  Client                Primary              Replica 1        Replica 2  │
│    │                      │                    │                │       │
│    │──Write X=1──────────►│                    │                │       │
│    │                      │──Sync Write───────►│                │       │
│    │                      │──Sync Write────────────────────────►│       │
│    │                      │◄──ACK──────────────│                │       │
│    │                      │◄──ACK───────────────────────────────│       │
│    │◄──ACK────────────────│                    │                │       │
│    │                      │                    │                │       │
│                                                                          │
│  Write is acknowledged only after ALL replicas confirm                  │
│  Any replica can serve reads with guaranteed consistency                │
│                                                                          │
│  Trade-offs:                                                             │
│  - High write latency (wait for slowest replica)                        │
│  - Reduced availability (one replica down = writes blocked)            │
│  - Strong consistency guarantee                                          │
└─────────────────────────────────────────────────────────────────────────┘


Method 2: Consensus Protocols (Paxos/Raft)

┌─────────────────────────────────────────────────────────────────────────┐
│                         Raft Consensus                                   │
│                                                                          │
│                      ┌───────────────┐                                  │
│                      │    Leader     │                                  │
│                      │   (Node 1)    │                                  │
│                      └───────┬───────┘                                  │
│                              │                                           │
│               ┌──────────────┼──────────────┐                           │
│               │              │              │                           │
│               ▼              ▼              ▼                           │
│        ┌───────────┐  ┌───────────┐  ┌───────────┐                     │
│        │ Follower  │  │ Follower  │  │ Follower  │                     │
│        │ (Node 2)  │  │ (Node 3)  │  │ (Node 4)  │                     │
│        └───────────┘  └───────────┘  └───────────┘                     │
│                                                                          │
│  Write Process:                                                          │
│  1. Client sends write to leader                                        │
│  2. Leader appends to local log                                         │
│  3. Leader replicates to followers                                      │
│  4. Majority (N/2 + 1) acknowledges                                     │
│  5. Leader commits and responds to client                               │
│  6. Leader notifies followers to commit                                 │
│                                                                          │
│  Requires: 2F+1 nodes to tolerate F failures                            │
│  Example: 5 nodes can tolerate 2 failures                               │
└─────────────────────────────────────────────────────────────────────────┘
```

### When to Use Strong Consistency

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   CHOOSE STRONG CONSISTENCY WHEN:                        │
│                                                                          │
│  1. Financial Transactions                                              │
│     └── Bank transfers must not allow double-spending                   │
│     └── Stock trades must reflect accurate balances                     │
│     └── Payment processing requires exact state                         │
│                                                                          │
│  2. Inventory Management                                                │
│     └── Cannot oversell limited stock items                             │
│     └── Reservation systems (flights, hotels)                           │
│     └── Ticket sales for popular events                                 │
│                                                                          │
│  3. Coordination Services                                               │
│     └── Distributed locks (ZooKeeper, etcd)                             │
│     └── Leader election                                                  │
│     └── Configuration management                                         │
│                                                                          │
│  4. Critical State Machines                                             │
│     └── Order state transitions                                          │
│     └── Workflow engines                                                 │
│     └── Approval processes                                               │
└─────────────────────────────────────────────────────────────────────────┘
```

## Eventual Consistency

### Definition and Guarantees

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Eventual Consistency                               │
│                                                                          │
│  Guarantee: If no new updates are made, eventually all replicas will    │
│  converge to the same value. No guarantee about when this happens.      │
│                                                                          │
│  Timeline:                                                               │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  Client A:  ───[Write X=1]───────────────────────────────────────────   │
│                      │                                                   │
│  Replica 1: ─────────X=1─────────────────────────────────────────────   │
│                      │                                                   │
│  Replica 2: ─────────│──────────────X=1──────────────────────────────   │
│                      │               ▲                                   │
│                      │      (Replication delay)                          │
│                      │                                                   │
│  Client B           │                                                    │
│  (reads R2): ────────│──[Read X]─────────────────────────────────────   │
│                      │      │                                            │
│                      │      └─► May return old value (or undefined)     │
│                      │                                                   │
│  Eventually:   All replicas have X=1                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Eventual Consistency Challenges

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Eventual Consistency Challenges                       │
│                                                                          │
│  1. Read Your Own Writes Problem:                                       │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  User writes profile picture                                 │    │
│     │  Redirected to different server                              │    │
│     │  Page shows old profile picture!                             │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  2. Monotonic Reads Problem:                                            │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Request 1: Read balance = $100                              │    │
│     │  Request 2: Read balance = $80 (older replica)               │    │
│     │  Time appears to go backward!                                 │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  3. Conflicting Writes:                                                 │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Client A: Set name = "Alice"  (Replica 1)                   │    │
│     │  Client B: Set name = "Bob"    (Replica 2)                   │    │
│     │  Which one wins? Need conflict resolution strategy.          │    │
│     └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Conflict Resolution Strategies

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Conflict Resolution Strategies                        │
│                                                                          │
│  1. Last-Writer-Wins (LWW):                                             │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Compare timestamps, keep the latest write                   │    │
│     │                                                               │    │
│     │  Client A: X=1 @ t=100                                       │    │
│     │  Client B: X=2 @ t=101                                       │    │
│     │  Result: X=2 (timestamp 101 wins)                            │    │
│     │                                                               │    │
│     │  Problem: Clock skew can cause data loss                     │    │
│     │  Used by: Cassandra, DynamoDB                                │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  2. Version Vectors (Detect conflicts):                                 │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Track version per replica                                    │    │
│     │                                                               │    │
│     │  Replica A: {A:1, B:0} writes X=1                            │    │
│     │  Replica B: {A:0, B:1} writes X=2                            │    │
│     │                                                               │    │
│     │  Neither dominates → CONFLICT DETECTED                       │    │
│     │  Application must resolve (or keep both as siblings)         │    │
│     │                                                               │    │
│     │  Used by: Riak, some DynamoDB configurations                 │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  3. CRDTs (Conflict-free Replicated Data Types):                        │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Data structures designed to merge automatically             │    │
│     │                                                               │    │
│     │  G-Counter (grow-only):                                      │    │
│     │    Node A: {A:5, B:3} → total = 8                            │    │
│     │    Node B: {A:5, B:5} → total = 10                           │    │
│     │    Merge:  {A:5, B:5} → total = 10 (take max per node)       │    │
│     │                                                               │    │
│     │  Other CRDTs: Sets, Maps, Registers, Sequences               │    │
│     │  Used by: Redis (CRDT mode), Riak                            │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  4. Application-Level Merge:                                            │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Custom logic based on domain knowledge                       │    │
│     │                                                               │    │
│     │  Shopping cart: Union of items from both versions            │    │
│     │  Document editing: Operational transformation                 │    │
│     │  User profile: Prompt user to resolve                        │    │
│     └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### When to Use Eventual Consistency

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   CHOOSE EVENTUAL CONSISTENCY WHEN:                      │
│                                                                          │
│  1. High Availability is Critical                                       │
│     └── System must remain operational during partitions               │
│     └── Multi-region deployments                                        │
│     └── User-facing applications with global traffic                   │
│                                                                          │
│  2. High Throughput Requirements                                        │
│     └── Social media feeds, likes, views                               │
│     └── Analytics and telemetry data                                   │
│     └── Logging and monitoring                                          │
│                                                                          │
│  3. Conflicts are Rare or Resolvable                                    │
│     └── Single-writer patterns (user owns their data)                  │
│     └── Append-only data (events, logs)                                │
│     └── Mergeable data structures (counters, sets)                     │
│                                                                          │
│  4. Stale Reads are Acceptable                                          │
│     └── Content feeds (posts, comments)                                 │
│     └── Product catalogs                                                │
│     └── Caching layers                                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Causal Consistency

### Definition and Guarantees

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Causal Consistency                                 │
│                                                                          │
│  Guarantee: If operation A causally precedes operation B, then every   │
│  node sees A before B. Concurrent operations may be seen in any order. │
│                                                                          │
│  Causality Examples:                                                     │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  1. Read-then-Write:                                                     │
│     Alice reads X=1, then writes Y=2                                    │
│     Anyone who sees Y=2 must also see X=1                               │
│                                                                          │
│  2. Same-Session Writes:                                                │
│     Alice writes X=1, then writes X=2                                   │
│     Everyone sees X=1 before X=2                                        │
│                                                                          │
│  3. Communication Chain:                                                 │
│     Alice writes "Hello"                                                 │
│     Bob sees "Hello", replies "Hi"                                      │
│     Carol sees "Hi" → Carol must also see "Hello"                       │
│                                                                          │
│  Non-Causal (Concurrent):                                               │
│     Alice writes X=1 (no knowledge of Bob)                              │
│     Bob writes Y=2 (no knowledge of Alice)                              │
│     Different nodes may see these in different orders                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementing Causal Consistency

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  Causal Consistency Implementation                       │
│                                                                          │
│  Method: Vector Clocks / Lamport Timestamps                             │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Node A          Node B          Node C                          │   │
│  │    │               │               │                              │   │
│  │    │ [A:1]         │               │                              │   │
│  │    │ Write X=1     │               │                              │   │
│  │    │───────────────►               │                              │   │
│  │    │            [A:1,B:1]          │                              │   │
│  │    │            Read X=1           │                              │   │
│  │    │            Write Y=2          │                              │   │
│  │    │               │───────────────►                              │   │
│  │    │               │           [A:1,B:1,C:1]                      │   │
│  │    │               │           Read X=1 ✓                        │   │
│  │    │               │           Read Y=2 ✓                        │   │
│  │    │               │               │                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Each operation carries a vector timestamp                              │
│  Before applying operation, check if dependencies are satisfied         │
│  If not, buffer the operation until dependencies arrive                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### Causal Consistency Use Cases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   CHOOSE CAUSAL CONSISTENCY WHEN:                        │
│                                                                          │
│  1. Social Applications                                                 │
│     └── Comments must appear after parent post                         │
│     └── Replies must follow original message                           │
│     └── "Like" must be on a visible post                               │
│                                                                          │
│  2. Collaborative Editing                                               │
│     └── Edits must preserve cause-effect relationship                  │
│     └── Undo must reverse correct operation                            │
│     └── References must point to existing content                      │
│                                                                          │
│  3. Multi-Step Workflows                                                │
│     └── Dependent operations in correct order                          │
│     └── Workflow state transitions                                      │
│     └── Audit trail integrity                                           │
│                                                                          │
│  Benefits over Strong Consistency:                                      │
│  - Better availability (no global coordination)                         │
│  - Lower latency (local reads possible)                                 │
│  - Intuitive for users (cause before effect)                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## Session Guarantees

### Read Your Own Writes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Read Your Own Writes                                 │
│                                                                          │
│  Problem:                                                                │
│  ─────────────────────────────────────────────────────────────────────  │
│  User updates profile → Redirected to different server → Sees old data │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  User                Server A              Server B               │ │
│  │    │                    │                      │                  │ │
│  │    │──Update Profile───►│                      │                  │ │
│  │    │◄──Success──────────│                      │                  │ │
│  │    │                    │──Async Replication──►│                  │ │
│  │    │                    │                      │                  │ │
│  │    │──────────Read Profile────────────────────►│                  │ │
│  │    │◄─────────OLD DATA (replication lag)───────│                  │ │
│  │    │                    │                      │                  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Solutions:                                                              │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  1. Sticky Sessions:                                                     │
│     └── Route user to same server that received write                  │
│     └── Use session cookie or consistent hashing                       │
│                                                                          │
│  2. Read from Primary:                                                   │
│     └── After write, read from primary for N seconds                   │
│     └── Or read from primary if reading user's own data               │
│                                                                          │
│  3. Version/Timestamp Tracking:                                         │
│     └── Client tracks last write timestamp                              │
│     └── Server rejects reads from replicas behind that timestamp       │
│                                                                          │
│  4. Write-Through Cache:                                                │
│     └── Update cache synchronously on write                            │
│     └── Read from cache first                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Monotonic Reads

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Monotonic Reads                                    │
│                                                                          │
│  Guarantee: If a read returns value X, subsequent reads will never     │
│  return a value older than X (time doesn't go backward).               │
│                                                                          │
│  Problem Scenario:                                                       │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Timeline:                                                        │ │
│  │                                                                    │ │
│  │  Primary:     ──[X=1]────[X=2]────[X=3]─────────────────────────  │ │
│  │                    │         │         │                          │ │
│  │  Replica A:   ────[X=1]─────[X=2]─────[X=3]────────────────────   │ │
│  │                                                                    │ │
│  │  Replica B:   ────[X=1]────────────────────────────────────────   │ │
│  │                    ▲         (lagging behind)                     │ │
│  │                    │                                              │ │
│  │  User reads:  Read1=X=2(A)  Read2=X=1(B)  ← Time went backward!   │ │
│  │                                                                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Solutions:                                                              │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  1. Session Stickiness:                                                  │
│     └── Same replica for entire session                                │
│                                                                          │
│  2. Version Tracking:                                                    │
│     └── Client sends last-seen version                                 │
│     └── Server waits until replica catches up                          │
│                                                                          │
│  3. Quorum Reads:                                                        │
│     └── Read from multiple replicas, return highest version            │
└─────────────────────────────────────────────────────────────────────────┘
```

### Monotonic Writes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Monotonic Writes                                   │
│                                                                          │
│  Guarantee: Writes from the same client are applied in order.          │
│                                                                          │
│  Problem Without Monotonic Writes:                                      │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Client:      Write1(X=1)         Write2(X=2)                     │ │
│  │                    │                   │                          │ │
│  │                    ▼                   ▼                          │ │
│  │  Replica A:   [X=1]──────────────[X=2]───────  Correct order     │ │
│  │                                                                    │ │
│  │  Replica B:   ─────[X=2]────[X=1]────────────  Wrong order!      │ │
│  │                     ▲         ▲                                   │ │
│  │                     │         └── Write1 arrived late            │ │
│  │                     └── Write2 arrived first (network reorder)   │ │
│  │                                                                    │ │
│  │  Result: Replica B has X=1, Replica A has X=2 (inconsistent!)    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Solutions:                                                              │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  1. Sequence Numbers:                                                   │
│     └── Each write carries client-assigned sequence number             │
│     └── Replicas apply in sequence order, buffer out-of-order         │
│                                                                          │
│  2. Single Writer Path:                                                 │
│     └── Route all writes from client through same node                 │
│                                                                          │
│  3. Lamport Timestamps:                                                  │
│     └── Logical clocks ensure ordering                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Quorum-Based Consistency

### Quorum Configuration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Quorum-Based Consistency                             │
│                                                                          │
│  N = Total replicas                                                      │
│  W = Write quorum (must acknowledge)                                    │
│  R = Read quorum (must query)                                           │
│                                                                          │
│  Strong Consistency: W + R > N                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  Example: N=3, W=2, R=2                                                 │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  Write (W=2):         Read (R=2):                                 │ │
│  │  ┌────────┐           ┌────────┐                                  │ │
│  │  │Node 1  │✓          │Node 1  │ → X=5                           │ │
│  │  └────────┘           └────────┘                                  │ │
│  │  ┌────────┐           ┌────────┐                                  │ │
│  │  │Node 2  │✓          │Node 2  │ → X=5                           │ │
│  │  └────────┘           └────────┘                                  │ │
│  │  ┌────────┐           ┌────────┐                                  │ │
│  │  │Node 3  │(slow)     │Node 3  │ (not queried)                   │ │
│  │  └────────┘           └────────┘                                  │ │
│  │                                                                    │ │
│  │  W + R = 4 > N = 3 → At least one node has latest write          │ │
│  │                                                                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Common Configurations:                                                  │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  N=3, W=2, R=2:  Balanced read/write, strong consistency               │
│  N=3, W=3, R=1:  Fast reads, slow writes, strong consistency           │
│  N=3, W=1, R=3:  Fast writes, slow reads, strong consistency           │
│  N=3, W=1, R=1:  Fast both, eventual consistency (W+R=2 ≤ N=3)        │
│  N=3, W=2, R=1:  Fast reads, eventual consistency (W+R=3 ≤ N=3)       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Sloppy Quorums and Hinted Handoff

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  Sloppy Quorums and Hinted Handoff                       │
│                                                                          │
│  Problem: Strict quorum fails when designated nodes are unavailable    │
│                                                                          │
│  Sloppy Quorum:                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│  Accept writes on ANY W nodes, not just designated replicas            │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Key K should go to: Node 1, Node 2, Node 3                       │ │
│  │                                                                    │ │
│  │  Node 1: ✓ Available                                              │ │
│  │  Node 2: ✗ Down                                                   │ │
│  │  Node 3: ✓ Available                                              │ │
│  │  Node 4: ✓ Available (not designated, but used for quorum)       │ │
│  │                                                                    │ │
│  │  Write accepted on Node 1, 3, 4 (sloppy quorum met)              │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Hinted Handoff:                                                         │
│  ─────────────────────────────────────────────────────────────────────  │
│  When designated node recovers, transfer data back                      │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Node 4 stores:                                                    │ │
│  │  {                                                                 │ │
│  │    data: "value",                                                  │ │
│  │    hint: "This belongs to Node 2"                                 │ │
│  │  }                                                                 │ │
│  │                                                                    │ │
│  │  When Node 2 recovers:                                             │ │
│  │  Node 4 ──sends hinted data──► Node 2                             │ │
│  │  Node 4 deletes the hint                                          │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Trade-off: Higher availability, but NOT strong consistency during     │
│  partition (sloppy quorum may not overlap with later reads)            │
└─────────────────────────────────────────────────────────────────────────┘
```

## Trade-offs Summary

### Consistency Model Comparison

| Model | Latency | Availability | Complexity | Use Case |
|-------|---------|--------------|------------|----------|
| **Strong** | High | Lower | Medium | Financial, inventory |
| **Eventual** | Low | High | High (conflicts) | Social, analytics |
| **Causal** | Medium | High | High | Messaging, collaboration |
| **Read-Your-Writes** | Medium | High | Medium | User-facing apps |

### Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Consistency Decision Matrix                           │
│                                                                          │
│                        Conflict Resolution                               │
│                        Easy         Hard                                 │
│                          │           │                                   │
│  Strong        ┌─────────┼───────────┼─────────┐                        │
│  Needed   Yes  │ Strong  │  Strong   │         │                        │
│                │ Consist │  Consist  │         │                        │
│                ├─────────┼───────────┤         │                        │
│           No   │ Eventual│  Causal/  │         │                        │
│                │ (fast)  │  Custom   │         │                        │
│                └─────────┴───────────┴─────────┘                        │
│                                                                          │
│  Questions to Ask:                                                       │
│  1. What happens if two users see different values temporarily?         │
│  2. Can we detect and resolve conflicts?                                │
│  3. What are the latency requirements?                                  │
│  4. Is the system read-heavy or write-heavy?                            │
│  5. Do we need multi-region deployment?                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

## Real-World Examples

### Example 1: Banking System (Strong Consistency)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Banking Transfer System                               │
│                                                                          │
│  Requirements:                                                           │
│  - No double-spending                                                    │
│  - Accurate balances                                                     │
│  - Audit trail                                                           │
│                                                                          │
│  Architecture:                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  Transfer Request                                                  │ │
│  │       │                                                            │ │
│  │       ▼                                                            │ │
│  │  ┌─────────────┐    ┌─────────────┐                               │ │
│  │  │ API Gateway │───►│ Transaction │                               │ │
│  │  └─────────────┘    │   Service   │                               │ │
│  │                     └──────┬──────┘                               │ │
│  │                            │                                       │ │
│  │                     ┌──────▼──────┐                               │ │
│  │                     │  Database   │                               │ │
│  │                     │  (Primary)  │                               │ │
│  │                     │             │                               │ │
│  │                     │ BEGIN TX;   │                               │ │
│  │                     │ CHECK bal   │                               │ │
│  │                     │ UPDATE A    │                               │ │
│  │                     │ UPDATE B    │                               │ │
│  │                     │ COMMIT;     │                               │ │
│  │                     └──────┬──────┘                               │ │
│  │                            │ Sync replication                     │ │
│  │                     ┌──────▼──────┐                               │ │
│  │                     │  Standby    │                               │ │
│  │                     │  (for HA)   │                               │ │
│  │                     └─────────────┘                               │ │
│  │                                                                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Key Decisions:                                                          │
│  - Single primary for writes (no multi-master)                          │
│  - Synchronous replication for standby                                  │
│  - Serializable isolation level                                          │
│  - Two-phase commit for distributed transactions                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Example 2: Social Media Feed (Eventual Consistency)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Social Media Feed System                              │
│                                                                          │
│  Requirements:                                                           │
│  - High availability                                                     │
│  - Low latency reads                                                     │
│  - Global distribution                                                   │
│  - Slight delay in seeing posts is acceptable                           │
│                                                                          │
│  Architecture:                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  User Posts                                                        │ │
│  │       │                                                            │ │
│  │       ▼                                                            │ │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │ │
│  │  │  Region A   │    │  Region B   │    │  Region C   │            │ │
│  │  │  (US-East)  │◄──►│  (EU)       │◄──►│  (Asia)     │            │ │
│  │  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘            │ │
│  │         │                  │                  │                    │ │
│  │         │     Async Replication (Kafka)       │                    │ │
│  │         │◄────────────────────────────────────►                    │ │
│  │         │                                                          │ │
│  │         ▼                                                          │ │
│  │  ┌─────────────────────────────────────────────────────────────┐  │ │
│  │  │ Feed Cache (Redis)                                          │  │ │
│  │  │ - Pre-computed feeds per user                               │  │ │
│  │  │ - Fan-out on write for active users                        │  │ │
│  │  │ - Pull model for inactive users                             │  │ │
│  │  └─────────────────────────────────────────────────────────────┘  │ │
│  │                                                                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Key Decisions:                                                          │
│  - Eventual consistency across regions                                  │
│  - Read from local region (low latency)                                 │
│  - Async replication via message queue                                  │
│  - Last-writer-wins for conflict resolution                            │
│  - Read-your-writes via session affinity                               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Example 3: Collaborative Document (Causal Consistency)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Collaborative Document System                         │
│                                                                          │
│  Requirements:                                                           │
│  - Real-time collaboration                                              │
│  - Operations must be causally ordered                                  │
│  - Offline support                                                       │
│                                                                          │
│  Architecture:                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  User A (Browser)          Server              User B (Browser)   │ │
│  │  ┌────────────────┐    ┌─────────────┐    ┌────────────────┐     │ │
│  │  │ Local Doc      │    │  Document   │    │ Local Doc      │     │ │
│  │  │ + Operation Log│◄──►│   Service   │◄──►│ + Operation Log│     │ │
│  │  │ + Vector Clock │    │             │    │ + Vector Clock │     │ │
│  │  └────────────────┘    └──────┬──────┘    └────────────────┘     │ │
│  │                               │                                    │ │
│  │  Operations:                  │                                    │ │
│  │  {                            ▼                                    │ │
│  │    id: "op_123",        ┌─────────────┐                           │ │
│  │    type: "insert",      │  Operation  │                           │ │
│  │    position: 5,         │  Transform  │                           │ │
│  │    char: "x",           │  Engine     │                           │ │
│  │    vectorClock: {       └─────────────┘                           │ │
│  │      A: 3, B: 1                                                    │ │
│  │    },                                                              │ │
│  │    causalDeps: ["op_122"]                                         │ │
│  │  }                                                                 │ │
│  │                                                                    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Key Decisions:                                                          │
│  - Operational Transformation (OT) or CRDT for merging                 │
│  - Vector clocks track causality                                        │
│  - Buffer out-of-order operations until dependencies arrive            │
│  - Local-first with sync to server                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

## Interview Talking Points

1. "I choose consistency models based on business requirements - banking needs strong consistency, social feeds can tolerate eventual"

2. "Quorum reads and writes give us tunable consistency - W+R>N for strong, W+R<=N for eventual"

3. "Read-your-writes is often the minimum acceptable UX - users must see their own updates immediately"

4. "CRDTs are powerful for collaborative features where conflicts are inherent"

5. "Causal consistency provides intuitive ordering without the cost of strong consistency"

6. "Conflict resolution strategy is as important as the consistency model itself"

## Common Interview Questions

### Q1: Explain the difference between strong and eventual consistency with examples.

**Answer:**
**Strong consistency** guarantees that after a write completes, all subsequent reads see that write. It's like a single-threaded program.

Example: Bank balance. If I transfer $100, the next balance check must reflect it. Otherwise, I might overdraw or think I have more money.

**Eventual consistency** guarantees that if no new writes occur, all replicas will eventually converge to the same value, but there's no guarantee when.

Example: Social media likes. If a post shows 100 likes to one user and 99 to another for a few seconds, that's acceptable. Eventually, both will see the correct count.

The trade-off is latency and availability. Strong consistency requires coordination, which adds latency and reduces availability during network partitions.

### Q2: How would you implement read-your-writes consistency?

**Answer:**
Several approaches:

1. **Sticky sessions:** Route the user to the same server that processed their write. Works but limits load balancing flexibility.

2. **Read from primary:** After a write, read from the primary database for a short window (e.g., 5 seconds). Simple but increases primary load.

3. **Version tracking:** The client includes the version of their last write. The server ensures the replica has at least that version before responding.

4. **Write-through cache:** Update a user-specific cache synchronously on write. Read from cache first.

I typically use version tracking as it's the most robust and doesn't require affinity.

### Q3: What is the CAP theorem and how does it apply to system design?

**Answer:**
CAP states that during a network partition, a distributed system must choose between Consistency and Availability - you can't have both.

**CP systems** (like HBase, Spanner): Reject operations rather than return inconsistent data. Use for financial systems.

**AP systems** (like Cassandra, DynamoDB): Accept operations and allow temporary inconsistency. Use for high-availability user-facing apps.

In practice, partitions are rare, so the real trade-off is latency vs consistency in normal operation. Many systems let you tune this per-operation - for example, DynamoDB allows strong consistency reads at higher latency.

### Q4: How do you handle conflicts in an eventually consistent system?

**Answer:**
Several strategies depending on use case:

1. **Last-writer-wins (LWW):** Use timestamps, keep the latest. Simple but can lose data if clocks are skewed.

2. **Version vectors:** Detect conflicts and let the application resolve them. More complex but no silent data loss.

3. **CRDTs:** Use data structures that merge automatically (counters, sets). Best for specific use cases like collaborative editing.

4. **Application-level merge:** Custom logic based on domain knowledge. Shopping cart might union items; document might prompt user.

For a shopping cart, I'd use a CRDT set that takes the union of items from concurrent updates - the user likely wants all items they added.

## Key Takeaways

- Consistency is a spectrum, not binary - choose the weakest model that meets requirements
- Strong consistency adds latency and reduces availability - use only when necessary
- Eventual consistency requires conflict resolution strategy
- Session guarantees (read-your-writes, monotonic reads) improve UX without full strong consistency
- Quorum configurations let you tune the consistency-availability trade-off
- Causal consistency provides intuitive ordering with better availability than strong consistency
- Always consider what happens during network partitions

## Further Reading

- "Designing Data-Intensive Applications" by Martin Kleppmann (Chapter 5, 9)
- "Consistency Models" - Jepsen.io
- "CRDTs: Conflict-free Replicated Data Types" by Shapiro et al.
- "Dynamo: Amazon's Highly Available Key-value Store" - Amazon Paper
- "Spanner: Google's Globally Distributed Database" - Google Paper
