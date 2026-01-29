# SQL vs NoSQL

## Overview

One of the most critical architectural decisions in system design is choosing between SQL (relational) and NoSQL (non-relational) databases.

## Quick Comparison

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| Data Model | Tables with rows/columns | Documents, key-value, graph, wide-column |
| Schema | Fixed, predefined | Flexible, dynamic |
| Scaling | Vertical (scale up) | Horizontal (scale out) |
| ACID | Full support | Varies (often BASE) |
| Joins | Native support | Limited/application-level |
| Query Language | SQL (standardized) | Database-specific |
| Best For | Complex queries, transactions | Scale, flexibility, speed |

## SQL Databases

### Characteristics

```
┌─────────────────────────────────────────────────────────────────┐
│                    Relational Database                          │
├─────────────────────────────────────────────────────────────────┤
│  Tables:                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ users                      │ orders                      │   │
│  │ ┌────┬────────┬──────────┐│ ┌────┬─────────┬──────────┐ │   │
│  │ │ id │ name   │ email    ││ │ id │ user_id │ total    │ │   │
│  │ ├────┼────────┼──────────┤│ ├────┼─────────┼──────────┤ │   │
│  │ │ 1  │ Alice  │ a@ex.com ││ │ 1  │ 1       │ $50.00   │ │   │
│  │ │ 2  │ Bob    │ b@ex.com ││ │ 2  │ 1       │ $75.00   │ │   │
│  │ └────┴────────┴──────────┘│ └────┴─────────┴──────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Relationships: users.id → orders.user_id (Foreign Key)        │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use SQL

1. **Complex Queries & Reporting**
   - Multi-table JOINs
   - Aggregations, GROUP BY
   - Ad-hoc analytical queries

2. **ACID Transactions**
   - Financial systems
   - Inventory management
   - Order processing

3. **Data Integrity Critical**
   - Foreign key constraints
   - Unique constraints
   - Check constraints

4. **Well-Defined Schema**
   - Requirements are stable
   - Data structure is known
   - Need schema enforcement

### Popular SQL Databases

| Database | Best For |
|----------|----------|
| PostgreSQL | Complex queries, extensions, JSON support |
| MySQL | Web applications, read-heavy workloads |
| SQL Server | Enterprise, .NET ecosystem |
| Oracle | Enterprise, large-scale OLTP |
| SQLite | Embedded, local storage |

### SQL Scaling Strategies

```
Vertical Scaling:
┌──────────────────┐      ┌──────────────────┐
│  Small Server    │ ──▶  │  Larger Server   │
│  4 CPU, 16GB    │      │  32 CPU, 256GB   │
└──────────────────┘      └──────────────────┘

Read Replicas:
                    ┌──────────────┐
              ┌────▶│ Read Replica │
              │     └──────────────┘
┌──────────┐  │     ┌──────────────┐
│  Master  │──┼────▶│ Read Replica │
│  (Write) │  │     └──────────────┘
└──────────┘  │     ┌──────────────┐
              └────▶│ Read Replica │
                    └──────────────┘

Sharding (more complex):
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Shard 1         │  │ Shard 2         │  │ Shard 3         │
│ Users A-H       │  │ Users I-P       │  │ Users Q-Z       │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

## NoSQL Databases

### Types of NoSQL

```
1. Document Store:
   ┌─────────────────────────────────────────────┐
   │ {                                            │
   │   "_id": "user123",                         │
   │   "name": "Alice",                          │
   │   "orders": [                               │
   │     {"id": 1, "total": 50.00},              │
   │     {"id": 2, "total": 75.00}               │
   │   ]                                         │
   │ }                                           │
   └─────────────────────────────────────────────┘
   Examples: MongoDB, CouchDB

2. Key-Value Store:
   ┌───────────────┬──────────────────────────────┐
   │ Key           │ Value                        │
   ├───────────────┼──────────────────────────────┤
   │ session:abc   │ {user_id: 123, expires: ...} │
   │ cache:page:1  │ <html>...</html>             │
   └───────────────┴──────────────────────────────┘
   Examples: Redis, DynamoDB, Memcached

3. Wide-Column Store:
   ┌─────────────────────────────────────────────┐
   │ Row Key    │ Column Family 1 │ Column Fam 2 │
   ├────────────┼─────────────────┼──────────────┤
   │ user:123   │ name:Alice      │ order:1:$50  │
   │            │ email:a@ex.com  │ order:2:$75  │
   └────────────┴─────────────────┴──────────────┘
   Examples: Cassandra, HBase, ScyllaDB

4. Graph Database:
   ┌───────┐    follows    ┌───────┐
   │ Alice │──────────────▶│  Bob  │
   └───────┘               └───────┘
       │                       │
       │ likes                 │ likes
       ▼                       ▼
   ┌───────┐               ┌───────┐
   │ Post1 │               │ Post2 │
   └───────┘               └───────┘
   Examples: Neo4j, Amazon Neptune
```

### When to Use NoSQL

1. **Massive Scale**
   - Billions of records
   - High write throughput
   - Global distribution

2. **Flexible Schema**
   - Rapidly evolving data
   - Varying attributes
   - Denormalized data

3. **Specific Access Patterns**
   - Key-value lookups
   - Time-series data
   - Graph traversals

4. **High Availability**
   - Partition tolerance priority
   - Eventually consistent is OK

### Popular NoSQL Databases

| Database | Type | Best For |
|----------|------|----------|
| MongoDB | Document | General purpose, flexible schema |
| DynamoDB | Key-Value | Serverless, auto-scaling |
| Cassandra | Wide-Column | Write-heavy, time-series |
| Redis | Key-Value | Caching, sessions, real-time |
| Neo4j | Graph | Social networks, recommendations |
| Elasticsearch | Document/Search | Full-text search, logs |

## Decision Framework

### Choose SQL When:

```
✅ Data is highly relational
✅ Need complex JOINs and aggregations
✅ ACID transactions required
✅ Schema is well-defined and stable
✅ Ad-hoc queries are common
✅ Strong consistency required
✅ Data integrity is critical
```

### Choose NoSQL When:

```
✅ Horizontal scaling is required
✅ Schema needs flexibility
✅ Simple read/write patterns
✅ High write throughput needed
✅ Geographic distribution required
✅ Eventually consistent is acceptable
✅ Document or key-value access patterns
```

### Decision Matrix

| Requirement | SQL Score | NoSQL Score |
|-------------|-----------|-------------|
| Complex queries | ⭐⭐⭐ | ⭐ |
| ACID transactions | ⭐⭐⭐ | ⭐ |
| Horizontal scaling | ⭐ | ⭐⭐⭐ |
| Schema flexibility | ⭐ | ⭐⭐⭐ |
| Write performance | ⭐⭐ | ⭐⭐⭐ |
| Strong consistency | ⭐⭐⭐ | ⭐ |
| Simple key lookups | ⭐ | ⭐⭐⭐ |

## Hybrid Approaches

### Polyglot Persistence

```
┌─────────────────────────────────────────────────────────────────┐
│                      E-Commerce System                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ PostgreSQL   │  │  MongoDB     │  │   Redis      │          │
│  │              │  │              │  │              │          │
│  │ Orders       │  │ Product      │  │ Sessions     │          │
│  │ Payments     │  │ Catalog      │  │ Cart Cache   │          │
│  │ Inventory    │  │ Reviews      │  │ Rate Limits  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │ Elasticsearch│  │  Neo4j       │                             │
│  │              │  │              │                             │
│  │ Product      │  │ Recommend-   │                             │
│  │ Search       │  │ ations       │                             │
│  └──────────────┘  └──────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

### NewSQL Databases

Combine SQL features with NoSQL scalability:

| Database | Features |
|----------|----------|
| CockroachDB | Distributed SQL, strong consistency |
| TiDB | MySQL compatible, horizontal scaling |
| YugabyteDB | PostgreSQL compatible, distributed |
| Spanner | Google's globally distributed SQL |

## Real-World Examples

| Company | SQL | NoSQL |
|---------|-----|-------|
| Uber | PostgreSQL (trips, payments) | Cassandra (location history), Redis (caching) |
| Netflix | MySQL (billing) | Cassandra (viewing history), Redis (sessions) |
| Airbnb | PostgreSQL (bookings) | Elasticsearch (search) |
| Twitter | MySQL (users) | Redis (timeline cache), Manhattan (tweets) |

## Interview Talking Points

1. "It's not SQL vs NoSQL, it's about choosing the right tool for each use case"

2. "SQL for transactions and complex queries, NoSQL for scale and flexibility"

3. "Consider hybrid approach - different databases for different data"

4. "Start with SQL unless you have specific NoSQL requirements"

5. "NoSQL doesn't mean no schema - you still need to design your data model"

## Common Interview Questions

1. **Q: When would you choose NoSQL over SQL?**
   A: When I need horizontal scaling, flexible schema, simple access patterns, or very high write throughput. Example: storing user activity logs at scale.

2. **Q: Can you use both SQL and NoSQL?**
   A: Yes, polyglot persistence. Use SQL for transactions and complex queries, NoSQL for specific use cases like caching or search.

3. **Q: How do you handle JOINs in NoSQL?**
   A: Denormalize data (embed related data), or perform JOINs at application level. Design around access patterns.

4. **Q: What about consistency in NoSQL?**
   A: Most NoSQL offer tunable consistency. Some (like MongoDB) support multi-document transactions. Choose based on requirements.

## Key Takeaways

- No universal "better" option - depends on requirements
- SQL: ACID, complex queries, data integrity
- NoSQL: Scale, flexibility, specific access patterns
- Modern systems often use both (polyglot persistence)
- Design around your access patterns and consistency needs

## Further Reading

- "Designing Data-Intensive Applications" - Chapter 2
- MongoDB vs PostgreSQL comparison guides
- AWS database selection guide
- NoSQL Distilled by Martin Fowler
