# Common Trade-Offs Matrix

## Overview

This document provides a comprehensive reference for the most common trade-offs in system design interviews.

## Master Trade-Off Matrix

| Trade-Off | Option A | Option B | Choose A When | Choose B When |
|-----------|----------|----------|---------------|---------------|
| Consistency vs Availability | Strong Consistency | High Availability | Financial transactions, inventory | Social feeds, caching |
| Latency vs Throughput | Low Latency | High Throughput | Real-time gaming | Batch processing |
| SQL vs NoSQL | Relational DB | NoSQL | Complex queries, ACID | Scale, flexibility |
| Push vs Pull | Push (fan-out) | Pull (fan-in) | Few followers | Many followers |
| Sync vs Async | Synchronous | Asynchronous | Immediate response needed | Background processing |
| Monolith vs Microservices | Monolith | Microservices | Small team, early stage | Large org, clear domains |
| Read vs Write Optimization | Read-optimized | Write-optimized | Read-heavy workloads | Write-heavy workloads |
| Space vs Time | More Memory | More Compute | Fast lookups needed | Memory constrained |
| Simplicity vs Flexibility | Simple design | Flexible design | Clear requirements | Evolving requirements |
| Cost vs Performance | Cost-optimized | Performance-optimized | Non-critical systems | User-facing, SLA-bound |

## Quick Decision Trees

### Database Selection

```
Need ACID transactions?
├── Yes ──▶ Need horizontal scaling?
│           ├── Yes ──▶ NewSQL (CockroachDB, Spanner)
│           └── No ──▶ Traditional SQL (PostgreSQL, MySQL)
└── No ──▶ What's your data model?
            ├── Key-Value ──▶ Redis, DynamoDB
            ├── Document ──▶ MongoDB, CouchDB
            ├── Wide-Column ──▶ Cassandra, HBase
            ├── Graph ──▶ Neo4j, Neptune
            └── Search ──▶ Elasticsearch
```

### Communication Pattern

```
Need real-time response?
├── Yes ──▶ User blocking on response?
│           ├── Yes ──▶ Synchronous (REST, gRPC)
│           └── No ──▶ WebSockets, SSE
└── No ──▶ Need guaranteed delivery?
            ├── Yes ──▶ Message Queue (Kafka, SQS)
            └── No ──▶ Fire-and-forget events
```

### Caching Strategy

```
Data changes frequently?
├── Yes ──▶ Can tolerate stale data?
│           ├── Yes ──▶ Cache with short TTL
│           └── No ──▶ Write-through or no cache
└── No ──▶ Data accessed frequently?
            ├── Yes ──▶ Cache with long TTL
            └── No ──▶ Cache on demand (cache-aside)
```

## Detailed Trade-Off Comparisons

### 1. Normalization vs Denormalization

```
┌─────────────────────────┬─────────────────────────┐
│    NORMALIZED           │    DENORMALIZED         │
├─────────────────────────┼─────────────────────────┤
│ No data duplication     │ Data duplicated         │
│ Complex queries (JOINs) │ Simple queries          │
│ Strong consistency      │ Eventual consistency    │
│ Write efficient         │ Read efficient          │
│ Storage efficient       │ Storage expensive       │
│ Update in one place     │ Update in many places   │
└─────────────────────────┴─────────────────────────┘

Choose Normalized: OLTP, transactional systems
Choose Denormalized: OLAP, read-heavy, NoSQL
```

### 2. Optimistic vs Pessimistic Locking

```
┌─────────────────────────┬─────────────────────────┐
│    OPTIMISTIC           │    PESSIMISTIC          │
├─────────────────────────┼─────────────────────────┤
│ Check at commit         │ Lock before read        │
│ High concurrency        │ Lower concurrency       │
│ Retry on conflict       │ Wait for lock           │
│ Good for read-heavy     │ Good for write-heavy    │
│ Low contention          │ High contention         │
│ No lock wait            │ Can cause deadlocks     │
└─────────────────────────┴─────────────────────────┘

Choose Optimistic: Low conflict rate, read-heavy
Choose Pessimistic: High conflict rate, write-heavy
```

### 3. REST vs GraphQL vs gRPC

```
┌───────────────┬───────────────┬───────────────┐
│     REST      │   GraphQL     │     gRPC      │
├───────────────┼───────────────┼───────────────┤
│ Simple        │ Flexible      │ Fast          │
│ Cacheable     │ Single request│ Binary/efficient│
│ Over-fetching │ No over-fetch │ Strong typing │
│ Multiple trips│ Complex server│ Not browser   │
│ HTTP/JSON     │ HTTP/JSON     │ HTTP/2+Protobuf│
│ Widely adopted│ Learning curve│ Service-to-svc│
└───────────────┴───────────────┴───────────────┘

Choose REST: Public APIs, simple CRUD
Choose GraphQL: Complex UIs, mobile apps
Choose gRPC: Microservices, performance-critical
```

### 4. Vertical vs Horizontal Scaling

```
┌─────────────────────────┬─────────────────────────┐
│    VERTICAL (Scale Up)  │  HORIZONTAL (Scale Out) │
├─────────────────────────┼─────────────────────────┤
│ Bigger machine          │ More machines           │
│ Simple                  │ Complex                 │
│ Hardware limits         │ Near-infinite scale     │
│ No code changes         │ May need code changes   │
│ Single point of failure │ Fault tolerant          │
│ Expensive (diminishing) │ Cost-effective          │
└─────────────────────────┴─────────────────────────┘

Choose Vertical: Simple scaling, traditional DBs
Choose Horizontal: Massive scale, high availability
```

### 5. CAP Theorem Positions

```
┌─────────────────────────────────────────────────────┐
│                    During Partition                  │
├──────────────────────────┬──────────────────────────┤
│        CP Systems        │        AP Systems        │
├──────────────────────────┼──────────────────────────┤
│ MongoDB (default)        │ Cassandra (default)      │
│ Redis Cluster            │ CouchDB                  │
│ HBase                    │ DynamoDB                 │
│ Zookeeper                │ Riak                     │
│ Etcd                     │ DNS                      │
├──────────────────────────┼──────────────────────────┤
│ Reject writes during     │ Accept writes during     │
│ partition                │ partition                │
│ Strong consistency       │ Eventual consistency     │
│ May be unavailable       │ Always available         │
└──────────────────────────┴──────────────────────────┘
```

### 6. Strong vs Eventual Consistency

```
┌─────────────────────────┬─────────────────────────┐
│    STRONG CONSISTENCY   │  EVENTUAL CONSISTENCY   │
├─────────────────────────┼─────────────────────────┤
│ All reads see latest    │ Reads may see stale     │
│ Higher latency          │ Lower latency           │
│ Lower availability      │ Higher availability     │
│ Simpler app logic       │ Complex app logic       │
│ Banking, inventory      │ Social feeds, analytics │
└─────────────────────────┴─────────────────────────┘
```

### 7. Stateful vs Stateless Services

```
┌─────────────────────────┬─────────────────────────┐
│       STATEFUL          │      STATELESS          │
├─────────────────────────┼─────────────────────────┤
│ Stores session state    │ No session state        │
│ Sticky sessions needed  │ Any instance can serve  │
│ Complex scaling         │ Easy horizontal scaling │
│ Faster (no lookup)      │ Lookup needed           │
│ Complex failover        │ Simple failover         │
│ Gaming, real-time       │ APIs, microservices     │
└─────────────────────────┴─────────────────────────┘

Modern approach: Stateless services + External state store
```

### 8. TCP vs UDP

```
┌─────────────────────────┬─────────────────────────┐
│         TCP             │         UDP             │
├─────────────────────────┼─────────────────────────┤
│ Reliable delivery       │ Best-effort delivery    │
│ Ordered packets         │ No ordering guarantee   │
│ Connection overhead     │ Connectionless          │
│ Flow control            │ No flow control         │
│ Higher latency          │ Lower latency           │
│ HTTP, databases         │ Video, gaming, DNS      │
└─────────────────────────┴─────────────────────────┘
```

### 9. Precomputation vs On-Demand

```
┌─────────────────────────┬─────────────────────────┐
│    PRECOMPUTATION       │      ON-DEMAND          │
├─────────────────────────┼─────────────────────────┤
│ Compute ahead of time   │ Compute when requested  │
│ Fast reads              │ Slower reads            │
│ Storage cost            │ Compute cost            │
│ May be stale            │ Always fresh            │
│ Good for hot data       │ Good for cold data      │
│ News feeds, dashboards  │ Reports, search         │
└─────────────────────────┴─────────────────────────┘
```

### 10. Client-Side vs Server-Side Rendering

```
┌─────────────────────────┬─────────────────────────┐
│       CSR (SPA)         │         SSR             │
├─────────────────────────┼─────────────────────────┤
│ Initial load slower     │ Initial load faster     │
│ Better interactivity    │ SEO friendly            │
│ Less server load        │ More server load        │
│ More client resources   │ Less client resources   │
│ Web apps, dashboards    │ Content sites, e-comm   │
└─────────────────────────┴─────────────────────────┘
```

## Common Interview Scenarios

### Scenario 1: High-Traffic Read-Heavy System

```
Trade-offs to consider:
1. Caching layer (Redis, Memcached)
2. Read replicas for database
3. CDN for static content
4. Denormalization for fewer JOINs
5. Eventually consistent where acceptable

Example: Product catalog
- Cache product data aggressively
- Use read replicas
- CDN for images
- Denormalize for product pages
```

### Scenario 2: Financial Transaction System

```
Trade-offs to consider:
1. Strong consistency (CP)
2. ACID transactions
3. Synchronous processing
4. Pessimistic locking
5. Normalized data

Example: Payment processing
- Use RDBMS with ACID
- Strong consistency required
- Synchronous confirmation to user
- Async for notifications
```

### Scenario 3: Social Media Feed

```
Trade-offs to consider:
1. Hybrid push/pull
2. Eventual consistency acceptable
3. Denormalized timeline
4. Heavy caching
5. Async fan-out

Example: Twitter timeline
- Push for regular users
- Pull for celebrities
- Cache precomputed timelines
- Eventual consistency for feed
```

## Interview Talking Points Summary

| Concept | Key Phrase |
|---------|------------|
| Consistency | "Depends on business requirements - banking needs strong, social feeds can be eventual" |
| Scaling | "Start with vertical, move to horizontal when hitting limits" |
| Caching | "Cache at every layer, but consider invalidation strategy" |
| Database | "Use SQL unless you have specific NoSQL requirements" |
| Architecture | "Start with monolith, extract services when boundaries are clear" |
| Communication | "Sync for user-facing, async for background processing" |
| Trade-offs | "No silver bullet - every decision has trade-offs" |

## Quick Reference: What to Choose

| If You Need... | Choose... |
|----------------|-----------|
| Transactions | SQL + ACID |
| Massive scale | NoSQL + eventual consistency |
| Real-time | WebSockets + push |
| Reliability | Message queues + retries |
| Low latency | Caching + CDN + edge |
| High throughput | Batching + async |
| Fault tolerance | Replication + load balancing |
| Flexibility | Microservices (if team is ready) |
| Simplicity | Monolith (for small teams) |

## Key Takeaways

1. **There's no universally correct answer** - it depends on requirements
2. **Understand the trade-offs** before making decisions
3. **Requirements drive architecture** - start there
4. **Start simple, evolve** - don't over-engineer
5. **Hybrid approaches** often work best in practice
6. **Articulate trade-offs clearly** in interviews

## Further Reading

- "Designing Data-Intensive Applications" by Martin Kleppmann
- "System Design Interview" by Alex Xu
- AWS Well-Architected Framework
- Google SRE Book
