# Scalability

## Overview

Scalability is the ability of a system to handle growing amounts of work by adding resources. A scalable system maintains performance as load increases.

## Types of Scaling

### Vertical Scaling (Scale Up)

Adding more power to existing machines (CPU, RAM, Storage).

```
Before:                 After:
┌─────────────┐         ┌─────────────┐
│   Server    │         │   Server    │
│   4 CPU     │   →     │   16 CPU    │
│   8 GB RAM  │         │   64 GB RAM │
│   100 GB    │         │   1 TB SSD  │
└─────────────┘         └─────────────┘
```

**Pros:**
- Simple to implement
- No code changes required
- No distributed system complexity
- Single point of management

**Cons:**
- Hardware limits (can't scale infinitely)
- Single point of failure
- Expensive at high end
- Downtime during upgrades

**When to Use:**
- Early-stage applications
- Databases that are hard to shard
- Legacy applications
- When simplicity is priority

### Horizontal Scaling (Scale Out)

Adding more machines to distribute the load.

```
Before:                 After:
┌─────────────┐         ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Server    │         │   Server    │  │   Server    │  │   Server    │
│   4 CPU     │   →     │   4 CPU     │  │   4 CPU     │  │   4 CPU     │
│   8 GB RAM  │         │   8 GB RAM  │  │   8 GB RAM  │  │   8 GB RAM  │
└─────────────┘         └─────────────┘  └─────────────┘  └─────────────┘
                               ▲               ▲               ▲
                               └───────────────┴───────────────┘
                                       Load Balancer
```

**Pros:**
- Theoretically unlimited scaling
- Better fault tolerance
- Cost-effective (commodity hardware)
- No single point of failure

**Cons:**
- Distributed system complexity
- Data consistency challenges
- Network overhead
- Requires stateless design

**When to Use:**
- High-traffic applications
- When fault tolerance is critical
- Stateless services
- Microservices architecture

## Scaling Strategies

### Database Scaling

1. **Read Replicas**: Distribute read load
2. **Sharding**: Partition data across databases
3. **Caching**: Reduce database hits
4. **Denormalization**: Trade storage for speed

### Application Scaling

1. **Stateless Design**: Enable any server to handle any request
2. **Session Management**: Use external session storage
3. **Async Processing**: Offload work to queues

### Caching Layers

```
Request → CDN → Application Cache → Database Cache → Database
           ↓           ↓                  ↓              ↓
        Static     Frequent           Query          Actual
        Content    Objects           Results          Data
```

## Bottleneck Identification

### Common Bottlenecks

| Layer | Symptoms | Solutions |
|-------|----------|-----------|
| CPU | High CPU usage, slow processing | Scale out, optimize code |
| Memory | OOM errors, swapping | Add RAM, optimize data structures |
| Disk I/O | Slow reads/writes | SSD, caching, sharding |
| Network | High latency, timeouts | CDN, compression, connection pooling |
| Database | Slow queries, locks | Indexing, read replicas, caching |

### Monitoring Metrics

- **Throughput**: Requests per second
- **Latency**: Response time (p50, p95, p99)
- **Error Rate**: Failed requests percentage
- **Resource Utilization**: CPU, memory, disk, network

## Scaling Patterns

### Stateless Architecture

```
┌─────────┐     ┌─────────┐
│ Client  │────▶│   LB    │
└─────────┘     └────┬────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │Server 1 │  │Server 2 │  │Server 3 │
   └────┬────┘  └────┬────┘  └────┬────┘
        │            │            │
        └────────────┴────────────┘
                     │
              ┌──────▼──────┐
              │   Shared    │
              │   State     │
              │(Redis/DB)   │
              └─────────────┘
```

### Database Sharding

```
User ID Hash → Shard Selection

User 1-1M  → Shard 1 (DB1)
User 1M-2M → Shard 2 (DB2)
User 2M-3M → Shard 3 (DB3)
```

## Real-World Examples

| Company | Strategy |
|---------|----------|
| Netflix | Microservices, horizontal scaling |
| Facebook | Sharded MySQL, Memcached |
| Google | Distributed systems, auto-scaling |
| Amazon | Service-oriented, eventual consistency |

## Interview Talking Points

1. "Start simple with vertical scaling, move to horizontal when needed"
2. "Stateless services are easier to scale horizontally"
3. "Identify bottlenecks before scaling"
4. "Caching is often the first line of defense"
5. "Consider data consistency implications"

## Common Interview Questions

1. **Q: When would you choose vertical over horizontal scaling?**
   A: For databases that are hard to shard, legacy systems, or when simplicity outweighs scale needs.

2. **Q: How do you identify scaling bottlenecks?**
   A: Monitor key metrics (CPU, memory, I/O, network), analyze slow queries, use profiling tools.

3. **Q: What are the challenges of horizontal scaling?**
   A: Data consistency, distributed transactions, service discovery, network partitions.

4. **Q: How do you scale a stateful service?**
   A: Externalize state to shared storage (Redis), use sticky sessions, or partition by user.

## Key Takeaways

- Understand both vertical and horizontal scaling
- Stateless design enables easier horizontal scaling
- Always identify bottlenecks before scaling
- Caching reduces load on backend services
- Trade-offs exist between consistency and availability

## Further Reading

- "Designing Data-Intensive Applications" by Martin Kleppmann
- Google's SRE Book (free online)
- AWS Well-Architected Framework
