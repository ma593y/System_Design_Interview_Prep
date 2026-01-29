# System Design Cheat Sheet

> One-page summary for quick interview review

---

## The 4-Step Framework

| Step | Time | Focus |
|------|------|-------|
| 1. Requirements | 5 min | Functional, non-functional, constraints |
| 2. Estimation | 5 min | Users, QPS, storage, bandwidth |
| 3. High-Level Design | 15 min | Components, data flow, APIs |
| 4. Deep Dive | 15 min | Bottlenecks, trade-offs, scaling |

---

## Key Non-Functional Requirements

| Requirement | Target | How to Achieve |
|-------------|--------|----------------|
| **Availability** | 99.9% - 99.99% | Redundancy, failover, multi-region |
| **Latency** | p99 < 200ms | Caching, CDN, indexing |
| **Throughput** | 10K-1M QPS | Horizontal scaling, async processing |
| **Consistency** | Strong/Eventual | Consensus protocols, replication |
| **Durability** | 99.999999999% | Replication, backups, WAL |

---

## Scaling Strategies

### Vertical vs Horizontal Scaling

| Aspect | Vertical | Horizontal |
|--------|----------|------------|
| Approach | Bigger machine | More machines |
| Limit | Hardware ceiling | Near unlimited |
| Complexity | Low | High |
| Cost | Expensive | Cost-effective at scale |
| Downtime | Required | Zero downtime |

### Database Scaling Progression

```
Single DB → Read Replicas → Vertical Scaling → Sharding → Multi-Region
```

### Sharding Strategies

| Strategy | Use Case | Pros | Cons |
|----------|----------|------|------|
| **Range-based** | Time-series data | Range queries | Hot spots |
| **Hash-based** | Even distribution | Balanced load | No range queries |
| **Directory-based** | Complex routing | Flexible | Lookup overhead |
| **Geographic** | Multi-region | Low latency | Complex sync |

---

## Caching Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Cache-Aside** | App manages cache | General purpose |
| **Read-Through** | Cache loads on miss | Read-heavy |
| **Write-Through** | Write to cache + DB | Strong consistency |
| **Write-Behind** | Async write to DB | Write-heavy |
| **Write-Around** | Write to DB only | Infrequent reads |

### Cache Eviction Policies
- **LRU** (Least Recently Used) - Most common
- **LFU** (Least Frequently Used) - Frequency matters
- **TTL** (Time To Live) - Time-based expiry
- **FIFO** (First In First Out) - Simple

---

## Data Partitioning

### Consistent Hashing
- Virtual nodes for even distribution
- Minimal rebalancing on node changes
- Used by: Cassandra, DynamoDB, Redis Cluster

### Replication Strategies

| Strategy | Consistency | Availability | Use Case |
|----------|-------------|--------------|----------|
| **Single Leader** | Strong | Lower | OLTP, banking |
| **Multi-Leader** | Eventual | High | Multi-DC |
| **Leaderless** | Tunable | Highest | High availability |

---

## CAP Theorem Quick Reference

| Scenario | Sacrifice | Examples |
|----------|-----------|----------|
| **CP** | Availability | MongoDB, HBase, Redis |
| **AP** | Consistency | Cassandra, DynamoDB, CouchDB |
| **CA** | Partition Tolerance | Single-node RDBMS (theoretical) |

> **Reality**: Network partitions happen. Choose CP or AP.

---

## Communication Patterns

| Pattern | Use Case | Latency | Coupling |
|---------|----------|---------|----------|
| **Sync REST** | Simple CRUD | Medium | Tight |
| **Async Queue** | Background jobs | High | Loose |
| **Event Streaming** | Real-time feeds | Low | Loose |
| **gRPC** | Internal services | Low | Medium |
| **WebSocket** | Bidirectional real-time | Very Low | Tight |

---

## Load Balancing Algorithms

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| **Round Robin** | Rotate sequentially | Equal capacity servers |
| **Weighted Round Robin** | Based on capacity | Mixed capacity |
| **Least Connections** | Fewest active | Varying request times |
| **IP Hash** | Consistent routing | Session affinity |
| **Random** | Random selection | Stateless services |

---

## Database Selection Guide

| Need | Choose | Examples |
|------|--------|----------|
| ACID transactions | Relational | PostgreSQL, MySQL |
| Flexible schema | Document | MongoDB, CouchDB |
| High write throughput | Wide-column | Cassandra, HBase |
| Complex relationships | Graph | Neo4j, Amazon Neptune |
| Simple key-value | Key-Value | Redis, DynamoDB |
| Full-text search | Search engine | Elasticsearch, Solr |
| Time-series data | Time-series | InfluxDB, TimescaleDB |

---

## Common Bottlenecks & Solutions

| Bottleneck | Symptoms | Solutions |
|------------|----------|-----------|
| **Database** | High latency, timeouts | Read replicas, caching, sharding |
| **Network** | Packet loss, latency | CDN, compression, regional deployment |
| **CPU** | High utilization | Horizontal scaling, code optimization |
| **Memory** | OOM errors, swapping | Caching layer, data compression |
| **Disk I/O** | Slow queries | SSDs, indexing, denormalization |

---

## Essential Components Checklist

```
[ ] Load Balancer (L4/L7)
[ ] API Gateway (auth, rate limiting)
[ ] CDN (static content)
[ ] Cache Layer (Redis/Memcached)
[ ] Database (SQL/NoSQL)
[ ] Message Queue (Kafka/RabbitMQ)
[ ] Object Storage (S3)
[ ] Search Engine (Elasticsearch)
[ ] Monitoring (metrics, logs, traces)
```

---

## Rate Limiting Algorithms

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| **Token Bucket** | Tokens refill over time | Bursty traffic |
| **Leaky Bucket** | Constant outflow rate | Smooth rate |
| **Fixed Window** | Count per time window | Simple implementation |
| **Sliding Window** | Rolling time window | Accurate limiting |

---

## Consistency Models

| Model | Description | Use Case |
|-------|-------------|----------|
| **Strong** | All reads see latest write | Banking, inventory |
| **Eventual** | Converges over time | Social media, analytics |
| **Causal** | Respects causality | Collaborative editing |
| **Read-your-writes** | User sees own writes | User profiles |

---

## Quick Estimation Numbers

| Metric | Value |
|--------|-------|
| 1 day | 86,400 seconds (~100K) |
| 1 month | 2.5 million seconds |
| 1 year | 31.5 million seconds |
| 1 million requests/day | ~12 QPS |
| 1 billion requests/day | ~12,000 QPS |
| 1 KB * 1 million | 1 GB |
| 1 MB * 1 million | 1 TB |

---

## Interview Red Flags to Avoid

- Starting to design without clarifying requirements
- Jumping into low-level details too early
- Ignoring non-functional requirements
- Not discussing trade-offs
- Over-engineering the solution
- Single points of failure
- Not considering failure scenarios
- Ignoring security and privacy

---

## Success Checklist

- [ ] Clarified requirements and constraints
- [ ] Estimated scale (users, QPS, storage)
- [ ] Drew clear high-level architecture
- [ ] Discussed API design
- [ ] Addressed data model and storage
- [ ] Identified and resolved bottlenecks
- [ ] Discussed trade-offs made
- [ ] Considered failure scenarios
- [ ] Mentioned monitoring and alerting
