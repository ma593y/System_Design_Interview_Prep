# Data Partitioning

## Overview

Data partitioning (sharding) distributes data across multiple nodes to enable horizontal scaling.

```
Before (single node):
┌─────────────────────────────┐
│      All Data (100TB)       │
│   Performance bottleneck    │
└─────────────────────────────┘

After (partitioned):
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│Partition1│ │Partition2│ │Partition3│ │Partition4│
│  (25TB)  │ │  (25TB)  │ │  (25TB)  │ │  (25TB)  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
   Node 1      Node 2       Node 3       Node 4
```

## Why Partition Data?

| Benefit | Description |
|---------|-------------|
| **Scale** | Handle more data than one machine can store |
| **Performance** | Distribute load across machines |
| **Availability** | Failure affects only subset of data |
| **Geography** | Data closer to users |

## Partitioning Strategies

### Hash Partitioning

Use hash function to determine partition.

```
partition = hash(key) % num_partitions

Key: user_123
hash("user_123") = 7823451
7823451 % 4 = 3
→ Goes to Partition 3

┌─────────────────────────────────────────────────────┐
│                  Hash Function                       │
│                                                      │
│  user_100 ──▶ hash() % 4 = 0 ──▶ Partition 0       │
│  user_101 ──▶ hash() % 4 = 1 ──▶ Partition 1       │
│  user_102 ──▶ hash() % 4 = 2 ──▶ Partition 2       │
│  user_103 ──▶ hash() % 4 = 3 ──▶ Partition 3       │
└─────────────────────────────────────────────────────┘
```

**Pros:**
- Even distribution
- Simple to implement
- Works for any key type

**Cons:**
- Adding/removing nodes redistributes many keys
- No range queries
- Hot spots still possible (popular keys)

### Range Partitioning

Partition by key ranges.

```
Partition 0: A-F
Partition 1: G-M
Partition 2: N-S
Partition 3: T-Z

┌────────────────────────────────────────────────────┐
│  Alice (A) ────▶ Partition 0                       │
│  Bob (B) ──────▶ Partition 0                       │
│  George (G) ───▶ Partition 1                       │
│  Tom (T) ──────▶ Partition 3                       │
└────────────────────────────────────────────────────┘
```

**Pros:**
- Efficient range queries
- Data locality for related items
- Intuitive

**Cons:**
- Uneven distribution (some ranges more popular)
- Hot spots (e.g., new users with recent IDs)
- Requires knowing data distribution

### Directory-Based Partitioning

Lookup table maps keys to partitions.

```
┌─────────────────────────┐
│    Partition Directory   │
│                         │
│  user_100 → Partition 2 │
│  user_101 → Partition 1 │
│  user_102 → Partition 0 │
│  user_103 → Partition 3 │
└─────────────────────────┘
          │
          ▼
   Request routed to
   correct partition
```

**Pros:**
- Flexible placement
- Easy to move individual keys
- Custom balancing

**Cons:**
- Directory is a bottleneck
- Directory must be highly available
- Extra lookup latency

## Consistent Hashing

Solves the redistribution problem when adding/removing nodes.

### Basic Concept

```
Hash ring (0 to 2^32-1):

            0
            │
     Node D │  Node A
         \  │  /
          \ │ /
           \│/
   270° ────●──── 90°
           /│\
          / │ \
     Node C │  Node B
            │
           180°

Keys hashed to ring, assigned to first node clockwise
```

### Adding a Node

```
Before:                    After adding Node E:
    ●A                         ●A
   / \                        / \
  C   B                      C   E  ← New node
   \ /                        \ / \
    ●                          ●   B

Only keys between C and E move to E
Other keys stay in place!
```

### Virtual Nodes

Distribute each physical node across multiple points on ring.

```
Without virtual nodes:
Node A may get 40% of keys
Node B may get 20% of keys (unbalanced)

With virtual nodes:
Node A: positions 10, 120, 250, 340
Node B: positions 50, 170, 290, 380

More even distribution
```

```
Ring with virtual nodes:
     A1    B1
      \    /
   B3  \  / A2
        \/
        /\
   A3  /  \ B2
      /    \
     B4    A4

A1, A2, A3, A4 = same physical Node A
```

## Partitioning Key Selection

### Single Key Partitioning

```
Partition by user_id:
- All user data on same partition
- Efficient user-centric queries
- May have hot users

user_id: 123 → Partition 2
```

### Composite Key Partitioning

```
Partition by (user_id, timestamp):
- Distribute single user's data
- Prevents hot spots for active users

(123, 2024-01-01) → Partition 1
(123, 2024-01-02) → Partition 3
```

### Choosing a Partition Key

| Factor | Consideration |
|--------|---------------|
| Query patterns | Key should match common queries |
| Distribution | Key values should spread evenly |
| Growth | Consider future data growth |
| Hot spots | Avoid keys that create hot partitions |

## Rebalancing

### Fixed Number of Partitions

```
Create more partitions than nodes:
10 nodes, 1000 partitions
Each node: ~100 partitions

Adding node:
- Move some partitions to new node
- No re-partitioning needed

Node 1: P1-P100
Node 2: P101-P200
...
New Node 11: Takes P1, P101, P201... (from each)
```

### Dynamic Partitioning

```
Split when partition too large:
Partition 1 (50GB) → Split → P1a (25GB) + P1b (25GB)

Merge when partition too small:
P1 (1GB) + P2 (1GB) → Merge → P1 (2GB)
```

### Partition Proportional to Nodes

```
Partitions = nodes × partitions_per_node

Adding a node:
- Take random partitions from existing nodes
- Split them with new node
```

## Hot Spot Handling

### Celebrity Problem

One key receives disproportionate traffic.

```
Taylor Swift's user_id → Same partition
Millions of requests to one node

Solutions:
1. Add random suffix: key_1, key_2, key_3
2. Dedicated handling for known hot keys
3. Caching layer
4. Read replicas for hot partitions
```

### Write Hot Spots

```
Append random suffix to spread writes:
order_id_0, order_id_1, order_id_2

Reading requires scatter-gather:
Read from all suffixed keys, merge results
```

## Secondary Indexes

### Local Index (Document-Partitioned)

```
Each partition has its own index:

Partition 1:             Partition 2:
┌─────────────────┐     ┌─────────────────┐
│ Data: Users A-M │     │ Data: Users N-Z │
│ Index: A-M      │     │ Index: N-Z      │
└─────────────────┘     └─────────────────┘

Query by index:
- Must query ALL partitions (scatter-gather)
- High latency for index queries
```

### Global Index (Term-Partitioned)

```
Separate index partitions:

Data Partitions:         Index Partitions:
┌─────────────┐          ┌────────────────┐
│ Users A-M   │          │ Terms A-M      │
│ Users N-Z   │          │ Terms N-Z      │
└─────────────┘          └────────────────┘

Query by index:
- Single partition for index lookup
- But writes update multiple partitions
- Index updates often async (eventually consistent)
```

## Cross-Partition Queries

### Scatter-Gather

```
Query: SELECT * FROM users WHERE age > 25

Coordinator:
├── Query Partition 1 → results1
├── Query Partition 2 → results2
├── Query Partition 3 → results3
└── Query Partition 4 → results4
         │
         ▼
    Merge results
```

### Denormalization

```
Avoid joins by duplicating data:

Original:
- Users table (partitioned by user_id)
- Orders table (partitioned by order_id)
- Join needed for user orders

Denormalized:
- Users table (partitioned by user_id)
  - Includes: user_name, recent_orders[]
- Orders table (partitioned by user_id)
  - Includes: user_name, order_details

No cross-partition joins needed
```

## Interview Talking Points

1. "Choose partitioning strategy based on query patterns"
2. "Consistent hashing minimizes redistribution when scaling"
3. "Virtual nodes improve load balancing"
4. "Hot spots require special handling"
5. "Secondary indexes have trade-offs between local and global"

## Common Interview Questions

1. **Q: How do you choose a partition key?**
   A: Consider query patterns, data distribution, and growth. Key should spread data evenly and support common queries without cross-partition operations.

2. **Q: What is consistent hashing and why use it?**
   A: Keys and nodes mapped to a ring. Adding/removing nodes only affects adjacent keys. Minimizes data movement during scaling.

3. **Q: How do you handle hot spots?**
   A: Add random suffixes to spread load, use caching, dedicate resources to known hot keys, or add read replicas.

4. **Q: How do secondary indexes work with partitioning?**
   A: Local indexes require scatter-gather for queries. Global indexes require single lookup but writes touch multiple partitions.

## Key Takeaways

- Partitioning enables horizontal scaling
- Hash partitioning for even distribution, range for range queries
- Consistent hashing minimizes rebalancing
- Carefully choose partition key based on access patterns
- Plan for hot spots and cross-partition queries

## Further Reading

- "Designing Data-Intensive Applications" Chapter 6
- DynamoDB partitioning documentation
- Cassandra partition key best practices
