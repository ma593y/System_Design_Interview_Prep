# Databases

## Overview

Databases are organized collections of data. Choosing the right database is crucial for system design.

## SQL vs NoSQL

### Relational Databases (SQL)

Structured data with predefined schemas, ACID transactions.

```
Users Table:
┌────┬──────────┬─────────────────┬─────────┐
│ id │   name   │      email      │  age    │
├────┼──────────┼─────────────────┼─────────┤
│ 1  │ Alice    │ alice@mail.com  │   28    │
│ 2  │ Bob      │ bob@mail.com    │   32    │
└────┴──────────┴─────────────────┴─────────┘

Orders Table:
┌────┬─────────┬────────────┬─────────┐
│ id │ user_id │   total    │  date   │
├────┼─────────┼────────────┼─────────┤
│ 1  │    1    │   99.99    │ 2024-01 │
│ 2  │    1    │   49.99    │ 2024-01 │
└────┴─────────┴────────────┴─────────┘
```

**Examples:** PostgreSQL, MySQL, Oracle, SQL Server

**Strengths:**
- ACID compliance
- Complex queries (JOINs)
- Data integrity (foreign keys)
- Mature ecosystem

**Weaknesses:**
- Rigid schema
- Horizontal scaling is hard
- Not ideal for unstructured data

### NoSQL Databases

Flexible schemas, designed for scale.

#### Document Stores

Store JSON-like documents.

```json
{
  "_id": "user123",
  "name": "Alice",
  "email": "alice@mail.com",
  "orders": [
    {"id": 1, "total": 99.99},
    {"id": 2, "total": 49.99}
  ]
}
```

**Examples:** MongoDB, CouchDB
**Use Cases:** Content management, catalogs, user profiles

#### Key-Value Stores

Simple key-value pairs, extremely fast.

```
"user:123" → {"name": "Alice", "email": "alice@mail.com"}
"session:abc" → {"user_id": 123, "expires": 1642000000}
```

**Examples:** Redis, DynamoDB, Memcached
**Use Cases:** Caching, sessions, real-time data

#### Wide-Column Stores

Data stored in column families.

```
Row Key: user123
┌────────────────┬──────────────────┬─────────────────┐
│  profile:name  │  profile:email   │  stats:logins   │
├────────────────┼──────────────────┼─────────────────┤
│     Alice      │  alice@mail.com  │      150        │
└────────────────┴──────────────────┴─────────────────┘
```

**Examples:** Cassandra, HBase, ScyllaDB
**Use Cases:** Time-series, IoT, analytics

#### Graph Databases

Nodes and edges for relationship-heavy data.

```
    (Alice)──FOLLOWS──▶(Bob)
       │                  │
    LIKES              POSTED
       │                  │
       ▼                  ▼
    (Post1)◀──COMMENT──(Charlie)
```

**Examples:** Neo4j, Amazon Neptune
**Use Cases:** Social networks, recommendations, fraud detection

## Comparison Table

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| Schema | Fixed | Flexible |
| Scaling | Vertical (mostly) | Horizontal |
| Transactions | ACID | BASE (usually) |
| Queries | Complex JOINs | Simple lookups |
| Consistency | Strong | Eventual (often) |
| Use Case | Structured data | Unstructured/semi |

## ACID Properties

| Property | Description | Example |
|----------|-------------|---------|
| **A**tomicity | All or nothing | Transfer fails = no money moved |
| **C**onsistency | Valid state to valid state | Balance can't go negative |
| **I**solation | Transactions don't interfere | Concurrent reads see consistent data |
| **D**urability | Committed = permanent | Power failure doesn't lose data |

## BASE Properties

| Property | Description |
|----------|-------------|
| **B**asically **A**vailable | System always responds |
| **S**oft state | State may change without input |
| **E**ventual consistency | Will become consistent eventually |

## Database Scaling

### Read Replicas

```
┌──────────────┐
│    Master    │◀── Writes
│   (Primary)  │
└──────┬───────┘
       │ Replication
  ┌────┴────┬────────────┐
  ▼         ▼            ▼
┌─────┐  ┌─────┐     ┌─────┐
│Rep 1│  │Rep 2│     │Rep 3│◀── Reads
└─────┘  └─────┘     └─────┘
```

**Pros:** Improves read throughput
**Cons:** Replication lag, writes still limited

### Sharding (Horizontal Partitioning)

Distribute data across multiple databases.

```
Sharding by User ID:

User 1-1M    → Shard 1
User 1M-2M   → Shard 2
User 2M-3M   → Shard 3

┌─────────┐  ┌─────────┐  ┌─────────┐
│ Shard 1 │  │ Shard 2 │  │ Shard 3 │
│ (1-1M)  │  │(1M-2M)  │  │(2M-3M)  │
└─────────┘  └─────────┘  └─────────┘
```

**Sharding Strategies:**
- **Hash-based**: `shard = hash(key) % num_shards`
- **Range-based**: Ranges of keys per shard
- **Directory-based**: Lookup table for shard mapping

**Challenges:**
- Cross-shard queries (JOINs)
- Rebalancing when adding shards
- Distributed transactions

### Vertical Partitioning

Split tables by columns.

```
Before:
Users (id, name, email, profile_pic, bio, preferences)

After:
Users_Core (id, name, email)           → Frequently accessed
Users_Profile (id, profile_pic, bio)   → Large, less frequent
```

## Indexing

Indexes speed up queries at the cost of write performance.

### B-Tree Index

Default for most databases. Good for range queries.

```
         [M]
        /   \
     [D,G]  [P,T]
     /|\    /|\
   [A-C][E-F][H-L] ...
```

**Best for:** Equality, range queries, sorting

### Hash Index

Direct lookup, O(1) for equality.

**Best for:** Exact match queries only

### Composite Index

Index on multiple columns.

```sql
CREATE INDEX idx_user_status_date ON orders(user_id, status, created_at);
```

**Column order matters!** Left-to-right usage.

## Denormalization

Trade storage for read performance.

### Normalized (3NF)
```
Users: id, name
Orders: id, user_id, total
Products: id, name, price
OrderItems: order_id, product_id, quantity
```

### Denormalized
```
Orders: id, user_id, user_name, total, items_json
```

**Pros:** Faster reads, no JOINs
**Cons:** Data duplication, harder updates

## Database Selection Guide

| Requirement | Consider |
|-------------|----------|
| ACID transactions | PostgreSQL, MySQL |
| High write throughput | Cassandra, ScyllaDB |
| Flexible schema | MongoDB, DynamoDB |
| Relationships/graphs | Neo4j |
| Real-time/caching | Redis |
| Full-text search | Elasticsearch |
| Time-series | InfluxDB, TimescaleDB |
| Analytics | ClickHouse, BigQuery |

## Interview Talking Points

1. "Choose SQL for complex queries and ACID, NoSQL for scale and flexibility"
2. "Read replicas handle read scaling, sharding handles write scaling"
3. "Denormalization trades storage for read performance"
4. "Indexing speeds reads but slows writes"
5. "Consider data access patterns when choosing database"

## Common Interview Questions

1. **Q: When would you choose NoSQL over SQL?**
   A: When you need horizontal scaling, flexible schema, high write throughput, or when data doesn't fit relational model well.

2. **Q: How do you handle database scaling?**
   A: Start with read replicas for read-heavy workloads, then consider sharding for write scaling, with caching in front.

3. **Q: What are the trade-offs of sharding?**
   A: Pros: Horizontal scale. Cons: Complex queries across shards, rebalancing difficulty, distributed transaction challenges.

4. **Q: How do you choose a sharding key?**
   A: Choose a key that distributes data evenly, is frequently used in queries, and minimizes cross-shard operations.

## Key Takeaways

- No one-size-fits-all database
- Understand ACID vs BASE trade-offs
- Scaling reads is easier than scaling writes
- Indexing is crucial for query performance
- Consider access patterns when designing schema

## Further Reading

- "Designing Data-Intensive Applications" by Martin Kleppmann
- Database vendor documentation
- Use The Index, Luke (indexing guide)
