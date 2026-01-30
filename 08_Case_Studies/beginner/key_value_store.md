# Distributed Key-Value Store System Design (Redis / DynamoDB)

## 1. Problem Statement

Design a distributed key-value store that can store and retrieve data across multiple
nodes with high availability, low latency, and strong durability guarantees. The system
should handle node failures gracefully, scale horizontally, and allow tunable consistency
levels similar to Amazon DynamoDB or Apache Cassandra.

**Core Functionality:**
- Store key-value pairs across a distributed cluster
- Retrieve values by key with low latency
- Handle node failures without data loss or downtime
- Scale horizontally by adding more nodes
- Support tunable consistency (strong vs eventual)

**Real-World Examples:**
- Amazon DynamoDB
- Apache Cassandra
- Redis Cluster
- Riak
- Voldemort (LinkedIn)

---

## 2. Requirements Clarification

### Functional Requirements

| Requirement | Description |
|-------------|-------------|
| Put(key, value) | Store a key-value pair in the system |
| Get(key) | Retrieve the value associated with a key |
| Delete(key) | Remove a key-value pair from the system |
| Tunable Consistency | Allow clients to choose consistency level per request |
| Automatic Partitioning | Distribute data across nodes transparently |
| Replication | Replicate data to N nodes for durability |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| High Availability | 99.99% uptime (< 52 min downtime/year) |
| Low Latency | p99 read/write < 10ms |
| Scalability | Handle petabytes of data, millions of ops/sec |
| Durability | No data loss even with multiple node failures |
| Partition Tolerance | Continue operating during network partitions |
| Tunability | Let clients trade off consistency for latency |

### Out of Scope
- Complex query support (range queries, secondary indexes)
- Transactions across multiple keys
- User authentication and authorization

---

## 3. Capacity Estimation

### Assumptions

```
- 100 million daily active users
- Average 50 read operations per user per day
- Average 10 write operations per user per day
- Average key size: 64 bytes
- Average value size: 1 KB
- Replication factor: N = 3
- Data retention: indefinite (with TTL support)
```

### Traffic Estimates

```
Read Operations:
- Daily reads: 100M users * 50 reads = 5 billion reads/day
- Reads per second: 5B / 86,400 ≈ 58,000 reads/sec
- Peak reads (3x average): ~174,000 reads/sec

Write Operations:
- Daily writes: 100M users * 10 writes = 1 billion writes/day
- Writes per second: 1B / 86,400 ≈ 11,600 writes/sec
- Peak writes (3x average): ~35,000 writes/sec

Total operations at peak: ~209,000 ops/sec
```

### Storage Estimates

```
Per key-value pair:
- Key: 64 bytes
- Value: 1 KB (1024 bytes)
- Metadata (timestamps, version, etc.): 64 bytes
- Total per entry: ~1,152 bytes ≈ 1.2 KB

Daily new data:
- New entries per day: 1 billion writes
- Daily storage (before replication): 1B * 1.2 KB = 1.2 TB/day
- Daily storage (with 3x replication): 3.6 TB/day

Annual storage:
- Per year (before replication): 1.2 TB * 365 = 438 TB
- Per year (with replication): 438 TB * 3 = 1.3 PB
```

### Memory Estimates

```
Hot data (20% of total accessed frequently):
- If 30 days of data fits in memory for caching
- 30 days * 1.2 TB/day = 36 TB in memory
- With 64 GB per node: 36 TB / 64 GB = 563 cache nodes

Cluster sizing (storage):
- If each node stores 2 TB on SSD
- For 1 year of data: 1.3 PB / 2 TB = 650 storage nodes
```

### Bandwidth Estimates

```
Read bandwidth:
- 58,000 reads/sec * 1.2 KB = 69.6 MB/s ≈ 70 MB/s
- Peak: 210 MB/s

Write bandwidth:
- 11,600 writes/sec * 1.2 KB = 13.9 MB/s
- With replication (3x): ~42 MB/s
- Peak: 126 MB/s
```

---

## 4. High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Library                             │
│            (Consistent Hashing Ring + Node Discovery)               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │    Load Balancer /     │
                    │    Coordinator Node    │
                    └───────────┬───────────┘
                                │
            ┌───────────────────┼───────────────────┐
            │                   │                   │
    ┌───────▼───────┐  ┌───────▼───────┐  ┌───────▼───────┐
    │   Node A      │  │   Node B      │  │   Node C      │
    │ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │
    │ │  Request   │ │  │ │  Request   │ │  │ │  Request   │ │
    │ │  Handler   │ │  │ │  Handler   │ │  │ │  Handler   │ │
    │ ├───────────┤ │  │ ├───────────┤ │  │ ├───────────┤ │
    │ │  Memory   │ │  │ │  Memory   │ │  │ │  Memory   │ │
    │ │  Cache     │ │  │ │  Cache     │ │  │ │  Cache     │ │
    │ ├───────────┤ │  │ ├───────────┤ │  │ ├───────────┤ │
    │ │  WAL      │ │  │ │  WAL      │ │  │ │  WAL      │ │
    │ │  (Write-   │ │  │ │  (Write-   │ │  │ │  (Write-   │ │
    │ │  Ahead Log)│ │  │ │  Ahead Log)│ │  │ │  Ahead Log)│ │
    │ ├───────────┤ │  │ ├───────────┤ │  │ ├───────────┤ │
    │ │  SS Table  │ │  │ │  SS Table  │ │  │ │  SS Table  │ │
    │ │  Storage   │ │  │ │  Storage   │ │  │ │  Storage   │ │
    │ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │
    └───────────────┘  └───────────────┘  └───────────────┘
            │                   │                   │
            └───────────────────┼───────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Gossip Protocol     │
                    │   (Membership &       │
                    │    Failure Detection)  │
                    └───────────────────────┘
```

### Consistent Hashing Ring

```
                         Node A (0)
                        /         \
                       /    Hash    \
                      /    Space     \
              VNode A2              VNode B1
              (240)                   (45)
                |                       |
                |    ┌───────────┐      |
                |    │   Keys    │      |
                |    │ mapped to │      |
                |    │  nearest  │      |
                |    │ clockwise │      |
                |    │   node    │      |
                |    └───────────┘      |
                |                       |
              VNode C1              Node B (90)
              (200)                     |
                \                      /
                 \                    /
              VNode C2           VNode A1
              (160)              (120)
                  \              /
                   \            /
                    Node C (180)

  Legend:
  - Ring: 0 to 2^32 hash space
  - Each physical node has multiple virtual nodes (VNodes)
  - Keys hash to a position, stored on next clockwise node
  - Replication: store on next N-1 clockwise distinct physical nodes
```

---

## 5. API Design

### Core Operations

```
PUT /api/v1/keys/{key}
Headers:
  Content-Type: application/octet-stream
  X-Consistency-Level: ONE | QUORUM | ALL
  X-TTL: 3600                              # optional, seconds
Body: <raw value bytes>

Response 200:
{
  "key": "user:12345:profile",
  "version": "v3_1706234567_nodeA",
  "timestamp": 1706234567890,
  "status": "ok"
}

Response 409 (Conflict - if using conditional write):
{
  "error": "conflict",
  "current_version": "v2_1706234500_nodeB",
  "message": "Value has been modified since last read"
}
```

```
GET /api/v1/keys/{key}
Headers:
  X-Consistency-Level: ONE | QUORUM | ALL

Response 200:
{
  "key": "user:12345:profile",
  "value": "<base64 encoded value>",
  "version": "v3_1706234567_nodeA",
  "timestamp": 1706234567890
}

Response 404:
{
  "error": "not_found",
  "key": "user:12345:profile"
}
```

```
DELETE /api/v1/keys/{key}
Headers:
  X-Consistency-Level: ONE | QUORUM | ALL

Response 200:
{
  "key": "user:12345:profile",
  "status": "deleted",
  "tombstone_expiry": 1706320967890
}
```

### Administrative Endpoints

```
GET /api/v1/cluster/status
Response 200:
{
  "nodes": [
    {"id": "nodeA", "status": "healthy", "load": 0.65, "tokens": [0, 120, 240]},
    {"id": "nodeB", "status": "healthy", "load": 0.52, "tokens": [45, 160, 280]},
    {"id": "nodeC", "status": "suspect", "load": 0.78, "tokens": [90, 200, 320]}
  ],
  "ring_version": 42,
  "total_keys": 15000000000
}

POST /api/v1/cluster/nodes
Body:
{
  "node_id": "nodeD",
  "address": "10.0.4.1:7000",
  "num_vnodes": 128
}
```

---

## 6. Data Model & Storage Engine

### LSM Tree Storage Structure

```
┌─────────────────────────────────────────────────┐
│                  Write Path                      │
│                                                  │
│  Client Write                                    │
│       │                                          │
│       ▼                                          │
│  ┌──────────┐    ┌──────────────────────────┐   │
│  │  Write-   │───▶│     MemTable             │   │
│  │  Ahead    │    │  (In-Memory Sorted Map)  │   │
│  │  Log      │    │                          │   │
│  │  (WAL)    │    │  key1 -> val1 (v3)       │   │
│  └──────────┘    │  key2 -> val2 (v1)       │   │
│                   │  key3 -> TOMBSTONE       │   │
│                   └──────────┬───────────────┘   │
│                              │ (when full, ~64MB)│
│                              ▼                   │
│                   ┌──────────────────────────┐   │
│                   │   Immutable MemTable      │   │
│                   │   (being flushed to disk) │   │
│                   └──────────┬───────────────┘   │
│                              ▼                   │
│  ┌───────────────────────────────────────────┐   │
│  │              SSTable Files (Sorted)        │   │
│  │                                            │   │
│  │  Level 0: [SST-1] [SST-2] [SST-3]        │   │
│  │  Level 1: [SST-4          ] [SST-5      ] │   │
│  │  Level 2: [SST-6                        ] │   │
│  │                                            │   │
│  │  Each SSTable has:                         │   │
│  │  ┌────────┬────────┬─────────┬──────────┐ │   │
│  │  │ Data   │ Index  │ Bloom   │ Metadata │ │   │
│  │  │ Blocks │ Block  │ Filter  │ Block    │ │   │
│  │  └────────┴────────┴─────────┴──────────┘ │   │
│  └───────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Key-Value Entry Format

```
┌──────────────────────────────────────────────────┐
│                  KV Entry                         │
├──────────────────────────────────────────────────┤
│ Field           │ Type        │ Size             │
├─────────────────┼─────────────┼──────────────────┤
│ key_length      │ uint16      │ 2 bytes          │
│ key             │ bytes       │ variable (≤ 256B) │
│ value_length    │ uint32      │ 4 bytes          │
│ value           │ bytes       │ variable (≤ 1MB) │
│ timestamp       │ int64       │ 8 bytes          │
│ version_vector  │ bytes       │ variable         │
│ tombstone       │ bool        │ 1 byte           │
│ ttl             │ int64       │ 8 bytes          │
│ checksum        │ uint32 CRC  │ 4 bytes          │
└─────────────────┴─────────────┴──────────────────┘
```

### Metadata Store (per node)

```
Node Metadata:
┌─────────────────────────────────────────────┐
│ Field              │ Description            │
├────────────────────┼────────────────────────┤
│ node_id            │ Unique node identifier │
│ ip_address         │ Node IP address        │
│ port               │ Listening port         │
│ status             │ ALIVE/SUSPECT/DEAD     │
│ tokens[]           │ VNode positions        │
│ heartbeat_counter  │ Monotonic counter      │
│ last_heartbeat_ts  │ Last heartbeat time    │
│ data_size_bytes    │ Total data stored      │
│ load               │ Current load factor    │
└────────────────────┴────────────────────────┘

Partition Map:
┌─────────────────────────────────────────────┐
│ Field              │ Description            │
├────────────────────┼────────────────────────┤
│ token_range_start  │ Start of hash range    │
│ token_range_end    │ End of hash range      │
│ primary_node       │ Node owning this range │
│ replica_nodes[]    │ Replica node IDs       │
│ version            │ Partition map version  │
└────────────────────┴────────────────────────┘
```

---

## 7. Deep Dive

### 7.1 Consistent Hashing & Data Partitioning

Consistent hashing maps both keys and nodes to the same hash ring (0 to 2^32 - 1).
Each key is assigned to the first node found clockwise on the ring.

**Problem with basic consistent hashing:**
- Nodes may be unevenly distributed on the ring
- When a node is added/removed, only adjacent nodes are affected (good), but
  load can become unbalanced (bad)

**Solution: Virtual Nodes (VNodes)**

```
Physical Node A  -->  VNode_A_0 at position 15
                      VNode_A_1 at position 98
                      VNode_A_2 at position 201
                      ...
                      VNode_A_127 at position 350

Physical Node B  -->  VNode_B_0 at position 33
                      VNode_B_1 at position 112
                      ...
```

Each physical node owns 128-256 virtual nodes spread across the ring.

**Benefits of VNodes:**
1. Better load distribution across nodes
2. When a node goes down, its load spreads across many nodes (not just one neighbor)
3. Heterogeneous hardware: give more VNodes to more powerful machines
4. Faster rebalancing when nodes join/leave

**Replication using the ring:**

```
For replication factor N = 3:
- Key K hashes to position P on the ring
- Walk clockwise to find the first node: this is the coordinator (primary)
- Continue clockwise to find next N-1 DISTINCT physical nodes
- Store replicas on all N nodes

Example:
  Key "user:42" hashes to position 50
  Walking clockwise: VNode_B_0(33) is behind us, next is VNode_A_1(98)
  Coordinator = Node A
  Next distinct physical node clockwise = Node C (at position 150)
  Next distinct physical node clockwise = Node B (at position 180)
  Replicas stored on: Node A, Node C, Node B
```

**Handling node addition:**

```
Before: Nodes A, B, C on ring
  Node A: responsible for ranges (C, A]
  Node B: responsible for ranges (A, B]
  Node C: responsible for ranges (B, C]

Adding Node D between A and B:
  Node A: responsible for ranges (C, A]      -- unchanged
  Node D: responsible for ranges (A, D]      -- new, takes from B
  Node B: responsible for ranges (D, B]      -- reduced range
  Node C: responsible for ranges (B, C]      -- unchanged

  Only Node B needs to transfer data to Node D.
  With VNodes, many small ranges transfer across many nodes.
```

### 7.2 Quorum Reads/Writes & Consistency

The system uses quorum-based replication with tunable consistency.

```
N = Total number of replicas (typically 3)
W = Number of replicas that must acknowledge a write
R = Number of replicas that must respond to a read

Consistency guarantee: W + R > N ensures overlap between
read and write quorums (at least one node has latest value).

Common configurations:
┌───────────────┬───┬───┬───┬──────────────────────────────┐
│ Configuration │ N │ W │ R │ Trade-off                    │
├───────────────┼───┼───┼───┼──────────────────────────────┤
│ Strong Read   │ 3 │ 1 │ 3 │ Fast writes, slow reads      │
│ Strong Write  │ 3 │ 3 │ 1 │ Slow writes, fast reads      │
│ Balanced      │ 3 │ 2 │ 2 │ Balanced latency             │
│ Eventual      │ 3 │ 1 │ 1 │ Fastest, may read stale data │
└───────────────┴───┴───┴───┴──────────────────────────────┘
```

**Write Path with Quorum:**

```
Client                Coordinator           Node A        Node B        Node C
  │                       │                   │             │             │
  │── PUT(key,val,W=2) ──▶│                   │             │             │
  │                       │── Write(key,val) ─▶│             │             │
  │                       │── Write(key,val) ──────────────▶│             │
  │                       │── Write(key,val) ─────────────────────────────▶│
  │                       │                   │             │             │
  │                       │◀── ACK ───────────│             │             │
  │                       │◀── ACK ─────────────────────────│             │
  │                       │                   │             │    (slow)   │
  │                       │  W=2 satisfied    │             │             │
  │◀── SUCCESS ───────────│  (2 ACKs received)│             │             │
  │                       │                   │             │             │
  │                       │◀── ACK (late) ──────────────────────────────── │
  │                       │  (3rd ACK arrives, │             │             │
  │                       │   write complete   │             │             │
  │                       │   on all replicas) │             │             │
```

**Read Path with Quorum:**

```
Client                Coordinator           Node A        Node B        Node C
  │                       │                   │             │             │
  │── GET(key, R=2) ─────▶│                   │             │             │
  │                       │── Read(key) ──────▶│             │             │
  │                       │── Read(key) ────────────────────▶│             │
  │                       │── Read(key) ──────────────────────────────────▶│
  │                       │                   │             │             │
  │                       │◀─ val(v3,ts=100) ─│             │             │
  │                       │◀─ val(v2,ts=90) ──────────────── │             │
  │                       │                   │             │             │
  │                       │  R=2 satisfied    │             │             │
  │                       │  Compare versions:│             │             │
  │                       │  v3 > v2, return v3             │             │
  │◀── val(v3) ───────────│                   │             │             │
  │                       │                   │             │             │
  │                       │── Read Repair ─────────────────▶│             │
  │                       │  (send v3 to Node B)            │             │
```

**Read Repair:** When the coordinator detects stale replicas during a read,
it sends the latest version to the outdated nodes in the background.

### 7.3 Conflict Resolution

When concurrent writes happen to different replicas (during a network partition
or with W < N), conflicts can arise. Two primary strategies:

**Strategy 1: Vector Clocks**

```
Vector Clock = { NodeA: counter, NodeB: counter, NodeC: counter }

Timeline of events:

1. Client writes key K via Node A
   Version: { A:1 }

2. Client reads K, gets { A:1 }, writes update via Node A
   Version: { A:2 }

3. Network partition occurs. Two clients write concurrently:

   Client X writes via Node A:
   Version: { A:3 }    (increments A's counter)

   Client Y writes via Node B:
   Version: { A:2, B:1 }   (increments B's counter)

4. Partition heals. Coordinator sees both versions:
   { A:3 } and { A:2, B:1 }
   Neither dominates the other (A:3 > A:2 but missing B)
   This is a CONFLICT -- both versions returned to client.

   The APPLICATION must resolve the conflict.
   (e.g., merge shopping cart items, pick most recent edit)

Dominance rules:
  VC1 dominates VC2 if ALL counters in VC1 >= VC2
  and at least one counter is strictly greater.

  { A:3, B:2 } dominates { A:2, B:1 }  --> no conflict, keep first
  { A:3 } vs { A:2, B:1 }              --> CONFLICT, cannot order
```

**Strategy 2: Last-Write-Wins (LWW)**

```
Simple but lossy approach:
- Each value has a timestamp
- When conflict detected, highest timestamp wins
- Losing write is silently discarded

Pros: Simple, no client-side merge logic needed
Cons: Data loss possible (concurrent writes lost)

Use when: Values are independent, overwrites are acceptable
         (e.g., session data, last-known location)

Avoid when: Values accumulate (shopping carts, counters)
```

### 7.4 Merkle Trees for Anti-Entropy

Background synchronization ensures replicas converge even without read repairs.

```
Merkle Tree (Hash Tree):

Each node builds a Merkle tree over its key-value data.
Leaf nodes = hash of individual key ranges.
Parent nodes = hash of children.

           Root Hash
           H(H12 + H34)
          /             \
      H12                H34
    H(H1+H2)          H(H3+H4)
    /      \           /      \
  H1        H2       H3       H4
hash(K1)  hash(K2) hash(K3)  hash(K4)
 range     range    range     range
[0,25)    [25,50)  [50,75)  [75,100)

Synchronization between Node A and Node B:

1. Exchange root hashes
   - If equal: trees are identical, no sync needed
   - If different: drill down

2. Exchange H12 and H34
   - H12 matches: left subtree identical, skip
   - H34 differs: drill into right subtree

3. Exchange H3 and H4
   - H3 differs: sync key range [50,75)
   - H4 matches: skip

Result: Only the divergent key ranges are transferred.
        O(log N) comparisons instead of O(N) full scan.
```

### 7.5 Gossip Protocol for Membership & Failure Detection

```
Gossip Protocol (Epidemic Protocol):

Every T seconds (e.g., T = 1 second), each node:
1. Picks a random peer node
2. Sends its membership list (node_id, heartbeat_counter, timestamp)
3. Merges the received list with its own

Node A's membership table:
┌──────────┬───────────────────┬──────────────┐
│ Node ID  │ Heartbeat Counter │ Timestamp    │
├──────────┼───────────────────┼──────────────┤
│ A        │ 1042              │ 1706234567   │
│ B        │ 998               │ 1706234565   │
│ C        │ 1015              │ 1706234560   │
│ D        │ 887               │ 1706234520   │  <-- possibly down?
└──────────┴───────────────────┴──────────────┘

Failure Detection:
- If no heartbeat update for T_fail seconds (e.g., 30s):
  Mark node as SUSPECT
- If still no update for T_cleanup seconds (e.g., 60s):
  Mark node as DEAD, remove from ring

Gossip propagation:
- With N nodes, information reaches all nodes in O(log N) rounds
- Example: 100 nodes, gossip period 1s
  All nodes informed in ~7 rounds (7 seconds)
```

```
Failure Handling Sequence:

     Normal Operation           Node C fails             After detection
     ┌───┐ ┌───┐ ┌───┐        ┌───┐ ┌───┐ ┌───┐       ┌───┐ ┌───┐
     │ A │ │ B │ │ C │        │ A │ │ B │ │ C │       │ A │ │ B │
     └─┬─┘ └─┬─┘ └─┬─┘        └─┬─┘ └─┬─┘ └─╳─┘       └─┬─┘ └─┬─┘ ┌───┐
       │     │     │              │     │     X            │     │   │ D │
       │◄────┤     │              │◄────┤                  │◄────┤   └─┬─┘
       │     │◄────┤              │     │                  │     │◄────┤
       │────►│     │              │────►│                  │────►│     │
       │     │────►│              │     │                  │     │────►│
       │     │     │              │     │                  │     │     │
   All healthy              C unreachable           D takes over C's
                            Gossip detects          token ranges via
                            failure                 hinted handoff
```

**Hinted Handoff:**

```
When Node C is down and a write is destined for C:
1. Coordinator writes to Node D instead (a "hint")
2. Node D stores the data with a hint: "this belongs to C"
3. When Node C recovers, Node D forwards all hinted data to C
4. Node D deletes the hinted data after transfer

This ensures writes succeed even during node failures,
maintaining high availability.
```

---

## 8. Scaling Discussion

### Horizontal Scaling

```
Adding a new node to the cluster:

1. New node joins the gossip network
2. Coordinator assigns VNode tokens to new node
3. Existing nodes stream relevant data to new node
4. Once caught up, new node begins serving requests

Data transfer is done in the background:
- No downtime for existing operations
- Throttled to avoid impacting read/write performance
- Verified using Merkle tree comparison

Example:
  Cluster has 10 nodes, each with 1 TB
  Add Node 11: ~1 TB / 10 = 100 GB transferred from each existing node
  At 100 MB/s network: ~17 minutes to complete
```

### Read Scaling

```
Strategy 1: Increase R (read from fewer replicas)
  - Set R=1 for eventually consistent reads
  - Reduces read latency and load

Strategy 2: Read replicas / follower reads
  - Route reads to any replica, not just coordinator
  - Client-side load balancing across replicas

Strategy 3: Caching layer
  ┌──────────┐     ┌──────────────┐     ┌──────────────┐
  │  Client   │────▶│  Cache Layer │────▶│  KV Cluster  │
  │           │     │  (Memcached) │     │              │
  └──────────┘     └──────────────┘     └──────────────┘

  Cache-aside pattern:
  1. Check cache first
  2. Cache miss -> read from KV store
  3. Populate cache on read
  4. Invalidate cache on write
```

### Write Scaling

```
Strategy 1: Batch writes
  - Client batches multiple writes into one request
  - Reduces network round trips

Strategy 2: Asynchronous replication (W=1)
  - Write returns after one ACK
  - Remaining replicas updated asynchronously
  - Risk: data loss if primary fails before replication

Strategy 3: Partition splitting
  - If a VNode becomes too hot, split its range
  - Creates two smaller ranges, each handled independently
```

### Multi-Datacenter Replication

```
┌─────────────────────┐              ┌─────────────────────┐
│   US-East Cluster    │              │   EU-West Cluster    │
│                      │              │                      │
│  ┌───┐ ┌───┐ ┌───┐  │  Async      │  ┌───┐ ┌───┐ ┌───┐  │
│  │ A │ │ B │ │ C │  │◄────────────▶│  │ D │ │ E │ │ F │  │
│  └───┘ └───┘ └───┘  │  Replication │  └───┘ └───┘ └───┘  │
│                      │              │                      │
│  N=3, W=2, R=2       │              │  N=3, W=2, R=2       │
│  (within DC)         │              │  (within DC)         │
└─────────────────────┘              └─────────────────────┘

- Each DC has its own quorum for local reads/writes
- Cross-DC replication is asynchronous (avoids latency penalty)
- Conflict resolution handles cross-DC divergence
- Clients route to nearest DC for lowest latency
```

---

## 9. Trade-offs

### Consistency vs Availability (CAP Theorem)

```
┌────────────────────────┬────────────────────────────────────┐
│ Choice                 │ Implication                        │
├────────────────────────┼────────────────────────────────────┤
│ Favor Consistency (CP) │ Reject writes during partition.    │
│                        │ Return errors instead of stale     │
│                        │ data. Better for banking, inventory│
├────────────────────────┼────────────────────────────────────┤
│ Favor Availability (AP)│ Accept writes during partition.    │
│                        │ May return stale data. Resolve     │
│                        │ conflicts later. Better for social │
│                        │ media, shopping carts.             │
└────────────────────────┴────────────────────────────────────┘

Our design: AP system with tunable consistency per request.
Default W=2, R=2 gives strong consistency in normal operation
but allows eventual consistency during partitions.
```

### LSM Tree vs B-Tree Storage

```
┌─────────────┬──────────────────────┬──────────────────────┐
│ Aspect      │ LSM Tree (our choice)│ B-Tree               │
├─────────────┼──────────────────────┼──────────────────────┤
│ Write perf  │ Excellent (sequential│ Good (random I/O)    │
│             │ writes, append-only) │                      │
├─────────────┼──────────────────────┼──────────────────────┤
│ Read perf   │ Good (may check      │ Excellent (single    │
│             │ multiple SSTables)   │ tree traversal)      │
├─────────────┼──────────────────────┼──────────────────────┤
│ Space       │ Higher (multiple     │ Lower (in-place      │
│ amplify     │ copies during        │ updates)             │
│             │ compaction)          │                      │
├─────────────┼──────────────────────┼──────────────────────┤
│ Write       │ Higher (compaction   │ Lower                │
│ amplify     │ rewrites data)       │                      │
├─────────────┼──────────────────────┼──────────────────────┤
│ Use case    │ Write-heavy workloads│ Read-heavy workloads │
└─────────────┴──────────────────────┴──────────────────────┘
```

### Vector Clocks vs Last-Write-Wins

```
┌─────────────┬───────────────────────┬──────────────────────┐
│ Aspect      │ Vector Clocks         │ Last-Write-Wins      │
├─────────────┼───────────────────────┼──────────────────────┤
│ Data loss   │ None (conflicts       │ Possible (concurrent │
│             │ surfaced to client)   │ writes silently lost)│
├─────────────┼───────────────────────┼──────────────────────┤
│ Complexity  │ High (client must     │ Low (automatic)      │
│             │ handle merge logic)   │                      │
├─────────────┼───────────────────────┼──────────────────────┤
│ Storage     │ Higher (store vector  │ Lower (single        │
│ overhead    │ per version)          │ timestamp)           │
├─────────────┼───────────────────────┼──────────────────────┤
│ Clock skew  │ Not affected          │ Problematic (NTP     │
│             │ (logical clocks)      │ drift can cause      │
│             │                       │ wrong ordering)      │
└─────────────┴───────────────────────┴──────────────────────┘
```

---

## 10. Failure Scenarios & Handling

### Scenario 1: Single Node Failure

```
Detection: Gossip protocol marks node as SUSPECT after 30s,
           then DEAD after 60s.

Handling:
1. Hinted handoff: writes destined for dead node go to a
   temporary stand-in node
2. Read requests served by remaining replicas (R=2 still possible
   with N=3, one node down)
3. When node recovers:
   a. Receives hinted handoff data
   b. Merkle tree sync fills any remaining gaps
   c. Gossip marks node as ALIVE again

Impact: None if R <= 2 and W <= 2. Operations continue normally.
```

### Scenario 2: Network Partition

```
    ┌──────────┐           ┌──────────┐
    │ Partition │           │ Partition │
    │    1      │     X     │    2      │
    │           │     X     │           │
    │  Node A   │     X     │  Node C   │
    │  Node B   │     X     │  Node D   │
    │           │     X     │           │
    └──────────┘           └──────────┘
         X = Network partition

AP Mode (our default):
- Both partitions continue accepting reads and writes
- Data may diverge between partitions
- When partition heals, anti-entropy reconciles differences
- Conflicts resolved via vector clocks or LWW

CP Mode (if configured):
- Partition with majority of replicas continues operating
- Minority partition rejects writes (returns 503)
- Reads may still be served if data is available locally
```

### Scenario 3: Datacenter Failure

```
Handling:
1. DNS/Load balancer redirects traffic to surviving DC
2. Surviving DC has full replica of all data (async replication)
3. May have slightly stale data (replication lag)
4. When DC recovers, anti-entropy synchronizes divergent data

RPO (Recovery Point Objective): ~seconds of replication lag
RTO (Recovery Time Objective): ~minutes for DNS failover
```

### Scenario 4: Disk Failure on a Node

```
Handling:
1. Node detects disk failure via I/O errors
2. Node marks itself as degraded in gossip
3. Data reconstructed from replicas on other nodes
4. New disk provisioned, data re-replicated

Prevention:
- RAID for disk redundancy within a node
- Regular disk health monitoring (SMART)
- Checksums on all data blocks (CRC32)
```

### Scenario 5: Hot Key (Single Key Receives Disproportionate Traffic)

```
Problem: One key (e.g., celebrity profile) gets millions of reads/sec,
         overwhelming the node responsible for that key.

Solutions:
1. Client-side caching with short TTL
2. Read replicas: allow reads from any replica, not just primary
3. Key splitting: "hot_key" -> "hot_key_1", "hot_key_2", ... "hot_key_N"
   Client randomly picks a shard, aggregates if needed
4. In-memory cache on coordinator node for hot keys
```

---

## 11. Monitoring & Alerting

### Key Metrics

```
┌─────────────────────────────┬────────────────────────────────┐
│ Metric                      │ Alert Threshold                │
├─────────────────────────────┼────────────────────────────────┤
│ Read latency p99            │ > 10ms (warn), > 50ms (crit)  │
│ Write latency p99           │ > 15ms (warn), > 100ms (crit) │
│ Read throughput (ops/sec)   │ Drop > 20% from baseline       │
│ Write throughput (ops/sec)  │ Drop > 20% from baseline       │
│ Disk usage per node         │ > 80% (warn), > 90% (crit)    │
│ Memory usage per node       │ > 85% (warn), > 95% (crit)    │
│ Replication lag             │ > 1s (warn), > 10s (crit)     │
│ Gossip convergence time     │ > 30s (warn)                   │
│ Node status changes         │ Any SUSPECT/DEAD transition    │
│ Compaction backlog          │ > 10 pending (warn)            │
│ Hinted handoff queue size   │ > 10,000 hints (warn)          │
│ Merkle tree diff count      │ > 5% keys divergent (crit)    │
│ Failed read/write requests  │ Error rate > 0.1% (crit)      │
│ Cross-DC replication lag    │ > 5s (warn), > 30s (crit)     │
└─────────────────────────────┴────────────────────────────────┘
```

### Monitoring Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  KV Node     │     │  KV Node     │     │  KV Node     │
│  ┌────────┐  │     │  ┌────────┐  │     │  ┌────────┐  │
│  │Metrics │  │     │  │Metrics │  │     │  │Metrics │  │
│  │Agent   │  │     │  │Agent   │  │     │  │Agent   │  │
│  └───┬────┘  │     │  └───┬────┘  │     │  └───┬────┘  │
└──────┼───────┘     └──────┼───────┘     └──────┼───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                   ┌────────▼────────┐
                   │  Prometheus /   │
                   │  InfluxDB       │
                   └────────┬────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────▼───┐  ┌─────▼──────┐  ┌───▼────────┐
     │  Grafana   │  │  PagerDuty │  │  Alert     │
     │  Dashboard │  │  Alerts    │  │  Manager   │
     └────────────┘  └────────────┘  └────────────┘
```

### Operational Dashboards

```
Dashboard 1: Cluster Health
- Node map with status (green/yellow/red)
- Ring visualization showing data distribution
- Token ownership and balance

Dashboard 2: Performance
- Read/write latency histograms (p50, p95, p99)
- Throughput time series (ops/sec)
- Queue depths and backlogs

Dashboard 3: Storage
- Disk usage per node
- Compaction progress and frequency
- SSTable count and sizes per level

Dashboard 4: Replication
- Replication lag across nodes
- Hinted handoff queue sizes
- Anti-entropy sync status
```

---

## 12. Interview Variations

### Common Follow-Up Questions

**Q1: "How would you handle a hot partition?"**
```
A: Multiple strategies:
1. Add more virtual nodes for the hot range
2. Split the hot partition into smaller ranges
3. Add a caching layer (Redis/Memcached) in front
4. Client-side caching with short TTL
5. Key-level sharding: append random suffix to hot keys,
   read from random shard, write to all shards
```

**Q2: "How would you add support for range queries?"**
```
A: Standard KV stores use hash-based partitioning which
breaks ordering. To support range queries:
1. Use range-based partitioning instead of hash-based
2. Keys within a partition are stored sorted (LSM/B-Tree)
3. Coordinator determines which partitions overlap the
   query range and fans out to those nodes
4. Trade-off: range partitioning can create hot spots
   (e.g., all today's timestamps go to one partition)
```

**Q3: "What happens if the coordinator node fails mid-write?"**
```
A: Client timeout and retry logic:
1. Client has a timeout (e.g., 5 seconds)
2. On timeout, client picks a new coordinator
3. Idempotency key prevents duplicate writes
4. If some replicas got the write but not all,
   anti-entropy will eventually propagate it
5. Client can increase W for critical writes
```

**Q4: "How do you handle schema evolution of stored values?"**
```
A: KV stores are typically schema-agnostic (opaque bytes).
For application-level evolution:
1. Version field in the value (deserializer handles versions)
2. Use a serialization format like Protobuf or Avro
   that supports backward/forward compatibility
3. Lazy migration: convert old format on read, write new format
```

**Q5: "How would you implement transactions across multiple keys?"**
```
A: Several approaches:
1. Single-partition transactions: if keys share a partition,
   use local locks/MVCC on that node
2. Two-phase commit (2PC): coordinator orchestrates across
   nodes. Blocks on coordinator failure.
3. Paxos/Raft consensus for distributed transactions
4. Application-level saga pattern for eventual consistency
Trade-off: each adds latency and complexity
```

**Q6: "How do you prevent data corruption?"**
```
A: Multiple layers of protection:
1. CRC32 checksum on every data block
2. Write-ahead log for crash recovery
3. Merkle trees detect divergence between replicas
4. Background scrubbing: periodically verify checksums
5. Replicas act as backup: corrupted data restored from healthy replica
```

**Q7: "Compare this design to Redis Cluster vs DynamoDB."**
```
A:
Redis Cluster:
- In-memory primary storage (faster, less capacity)
- Uses hash slots (16384 slots) instead of consistent hashing
- Asynchronous replication (potential data loss on failover)
- No tunable consistency (always eventual)

DynamoDB:
- Managed service (no operational burden)
- Uses consistent hashing with virtual nodes (similar to our design)
- Supports both eventually consistent and strongly consistent reads
- Auto-scaling based on throughput
- Global tables for multi-region replication

Our design is closest to DynamoDB/Cassandra architecture.
```

---

## Summary: Key Design Decisions

```
┌─────────────────────┬──────────────────────────────────────┐
│ Component           │ Decision                             │
├─────────────────────┼──────────────────────────────────────┤
│ Partitioning        │ Consistent hashing with VNodes       │
│ Replication         │ Leaderless, N=3 replicas             │
│ Consistency         │ Tunable quorum (W + R > N)           │
│ Conflict resolution │ Vector clocks (default) or LWW       │
│ Storage engine      │ LSM Tree (write-optimized)           │
│ Failure detection   │ Gossip protocol                      │
│ Recovery            │ Hinted handoff + Merkle tree sync    │
│ Membership          │ Gossip-based decentralized           │
│ Caching             │ In-memory MemTable + optional cache  │
│ Durability          │ WAL + SSTable persistence            │
└─────────────────────┴──────────────────────────────────────┘
```
