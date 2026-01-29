# Capacity Planning

## Overview

Capacity planning is the process of determining the production capacity needed to meet changing demands for your product or service. In system design, this means calculating the number of servers, amount of storage, network bandwidth, and other resources required to handle expected load. This guide covers server sizing, resource estimation formulas, and practical planning strategies.

---

## Capacity Planning Framework

### The Planning Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAPACITY PLANNING PROCESS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. UNDERSTAND REQUIREMENTS                                      │
│     └── Traffic patterns, SLAs, growth projections              │
│                                                                  │
│  2. MEASURE BASELINE                                             │
│     └── Current usage, resource utilization, bottlenecks        │
│                                                                  │
│  3. MODEL DEMAND                                                 │
│     └── Convert business metrics to technical requirements      │
│                                                                  │
│  4. CALCULATE CAPACITY                                           │
│     └── Servers, storage, network, database sizing              │
│                                                                  │
│  5. PLAN FOR GROWTH                                              │
│     └── Scaling strategy, headroom, contingency                 │
│                                                                  │
│  6. VALIDATE AND ITERATE                                         │
│     └── Load testing, monitoring, continuous refinement         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Input Metrics

| Metric | Description | Example |
|--------|-------------|---------|
| DAU/MAU | Daily/Monthly Active Users | 10M DAU |
| Requests per user | Actions per session | 50 requests |
| Data per request | Request/response size | 10 KB average |
| Peak multiplier | Peak vs average ratio | 3x |
| Growth rate | Year-over-year growth | 50% |
| SLA target | Availability requirement | 99.9% |

---

## Server Sizing Formulas

### Basic Server Calculation

```
Servers Needed = Peak QPS / QPS per Server

Where:
- Peak QPS = Average QPS x Peak Multiplier
- QPS per Server = Based on benchmarks and server specs

Example:
- Average QPS: 10,000
- Peak multiplier: 3x
- Peak QPS: 30,000
- Server capacity: 5,000 QPS each

Servers needed = 30,000 / 5,000 = 6 servers
```

### With Redundancy and Headroom

```
Total Servers = (Peak QPS / QPS per Server) x (1 + Redundancy) x (1 + Headroom)

Where:
- Redundancy: typically 0.5 (N+1) to 1.0 (N+N)
- Headroom: typically 0.3 to 0.5 (30-50% buffer)

Example:
Base servers: 6
With N+1 redundancy: 6 x 1.5 = 9
With 30% headroom: 9 x 1.3 = 11.7 ≈ 12 servers
```

### CPU-Bound Workloads

```
Servers = (Total CPU Time per Second) / (Available CPU Time per Server)

CPU Time per Request = Processing time (excluding I/O waits)
Available CPU Time = Cores x 1 second x Target Utilization

Example:
- 10,000 QPS
- 50ms CPU time per request
- 16-core servers
- Target 70% utilization

Total CPU time = 10,000 x 0.05s = 500 CPU-seconds per second
Available per server = 16 x 1 x 0.7 = 11.2 CPU-seconds per second
Servers needed = 500 / 11.2 = 44.6 ≈ 45 servers
```

### Memory-Bound Workloads

```
Servers = Total Memory Required / Memory per Server

Total Memory = Working Set + Cache + Buffer + OS Overhead

Example (Caching layer):
- 100 GB cache required
- 64 GB RAM per server
- OS and app overhead: 8 GB
- Available for cache: 56 GB

Servers needed = 100 GB / 56 GB = 1.79 ≈ 2 servers
With replication (3 copies): 6 servers
```

### I/O-Bound Workloads

```
Servers = Required IOPS / IOPS per Server

IOPS per Server = Storage IOPS x (1 - Utilization Buffer)

Example:
- Required: 100,000 IOPS
- Server with NVMe SSD: 50,000 IOPS
- Target utilization: 70%

Effective IOPS per server = 50,000 x 0.7 = 35,000
Servers needed = 100,000 / 35,000 = 2.86 ≈ 3 servers
```

---

## Resource Estimation Tables

### Server Capacity Benchmarks

| Server Type | Typical Capacity | Use Case |
|-------------|------------------|----------|
| Web server (static) | 10,000-50,000 QPS | Static content serving |
| Web server (dynamic) | 1,000-10,000 QPS | Dynamic page generation |
| API server | 5,000-20,000 QPS | JSON API endpoints |
| Application server | 1,000-5,000 QPS | Business logic processing |
| Database (read) | 10,000-50,000 QPS | Simple queries |
| Database (write) | 1,000-10,000 QPS | Transactional writes |
| Cache server | 100,000-500,000 QPS | In-memory key-value |

### Memory Requirements by Workload

| Workload | Memory Estimate | Notes |
|----------|-----------------|-------|
| Web server | 2-4 GB base | Plus connections |
| Application server | 4-16 GB | Depends on framework |
| Database buffer pool | 70-80% of RAM | For working set |
| Cache (Redis/Memcached) | Data size x 1.5 | Overhead for data structures |
| Search (Elasticsearch) | Index size x 2 | Heap + OS cache |
| Message queue | Messages x size x 2 | In-flight + buffer |

### Network Bandwidth Guidelines

| Traffic Type | Bandwidth per Server | Notes |
|--------------|---------------------|-------|
| API traffic | 100-500 Mbps | Small payloads |
| Web pages | 500 Mbps - 1 Gbps | HTML, CSS, JS |
| Image serving | 1-10 Gbps | Depends on image size |
| Video streaming | 10-100 Gbps | High bandwidth |
| Database replication | 100-500 Mbps | Write volume dependent |
| Backup traffic | 1-10 Gbps | Off-peak windows |

---

## Capacity Planning Formulas

### From DAU to Infrastructure

```
Step 1: DAU to QPS
─────────────────────
Daily Requests = DAU x Requests per User per Day
Average QPS = Daily Requests / 86,400
Peak QPS = Average QPS x Peak Multiplier (2-5x)

Step 2: QPS to Servers
─────────────────────
Base Servers = Peak QPS / QPS per Server
Total Servers = Base x Redundancy Factor x Headroom Factor

Step 3: Servers to Other Resources
─────────────────────
Total RAM = Servers x RAM per Server
Total Storage = Data Volume x Replication Factor
Total Bandwidth = Peak QPS x Response Size x 8 bits
```

### Complete Example: Social Media App

```
INPUTS:
- DAU: 50 million
- Requests per user per day: 100
- Average response size: 5 KB
- Peak multiplier: 3x
- Server capacity: 5,000 QPS
- Growth rate: 100% YoY

CALCULATIONS:

Traffic:
Daily requests = 50M x 100 = 5 billion/day
Average QPS = 5B / 86,400 = 57,870 QPS
Peak QPS = 57,870 x 3 = 173,611 QPS

Servers (current):
Base servers = 173,611 / 5,000 = 34.7 ≈ 35
With redundancy (N+1, 50%): 35 x 1.5 = 52.5 ≈ 53
With headroom (30%): 53 x 1.3 = 68.9 ≈ 70 servers

Bandwidth:
Peak bandwidth = 173,611 x 5 KB x 8 = 6.9 Gbps
With overhead (20%): 6.9 x 1.2 = 8.3 Gbps

Growth (Year 2):
DAU: 100M (2x growth)
Peak QPS: 347,222
Servers needed: 140 servers
```

---

## Database Capacity Planning

### Database Sizing

```
Database Capacity Requirements:
1. Storage = Data size x Replication x Index overhead x Headroom
2. IOPS = Read QPS x Reads per query + Write QPS x Writes per operation
3. Connections = Peak concurrent users x Connection multiplier
4. Memory = Working set size (ideally fits in RAM)
```

### Read Replica Calculation

```
Read Replicas Needed = Read QPS / QPS per Replica

Example:
- Total read QPS: 50,000
- Single replica capacity: 20,000 QPS
- Primary handles: 10,000 QPS (reserves capacity for writes)

Read replicas = (50,000 - 10,000) / 20,000 = 2 replicas

With redundancy: 3 read replicas + 1 primary = 4 total
```

### Sharding Calculation

```
Shards Needed = Max(Storage shards, QPS shards)

Storage-based:
Shards = Total Data Size / Max Size per Shard

QPS-based:
Shards = Total QPS / Max QPS per Shard

Example:
- Total data: 10 TB
- Max per shard: 500 GB
- Total QPS: 100,000
- Max QPS per shard: 10,000

Storage shards = 10,000 GB / 500 GB = 20 shards
QPS shards = 100,000 / 10,000 = 10 shards

Take maximum: 20 shards needed
```

### Connection Pool Sizing

```
Pool Size = (Concurrent Users x Connections per User) / Servers

Or using Little's Law:
Pool Size = QPS x Average Query Time

Example:
- Peak QPS to DB: 10,000
- Average query time: 10ms
- 5 application servers

Total connections = 10,000 x 0.01 = 100 concurrent
Per server = 100 / 5 = 20 connections per pool

Add buffer (50%): 30 connections per server pool
Total DB connections: 30 x 5 = 150 connections
```

---

## Cache Capacity Planning

### Cache Size Estimation

```
Cache Size = Number of Objects x Size per Object x (1 + Overhead)

Where overhead includes:
- Key storage (typically 50-100 bytes)
- Data structure overhead (10-20%)
- Memory fragmentation (10-20%)

Example:
- 10 million users
- 5 KB profile data each
- Cache hit rate target: 90%

Active users (likely in cache): 10M x 0.3 = 3M
Cache size = 3M x 5 KB x 1.3 = 19.5 GB

With replication (3 copies): 58.5 GB
Redis servers (32 GB each, 24 GB usable): 3 servers
```

### Cache Hit Rate Impact

```
Database Load = Total QPS x (1 - Cache Hit Rate)

Example:
- Total read QPS: 100,000
- Cache hit rate: 95%
- Database QPS: 100,000 x 0.05 = 5,000

Improving cache hit rate:
- 95% hit rate: 5,000 DB QPS
- 99% hit rate: 1,000 DB QPS (5x reduction)
- 99.9% hit rate: 100 DB QPS (50x reduction)
```

### Cache Throughput

```
Operations per Server = Memory Bandwidth / Operation Size

Redis single-thread benchmark:
- GET: 100,000-150,000 ops/sec
- SET: 80,000-120,000 ops/sec
- Pipeline (100 ops): 500,000+ ops/sec effective

Example capacity plan:
- Required: 500,000 ops/sec
- Single Redis: 100,000 ops/sec
- Redis servers needed: 5 (without replication)
- With 3 replicas for HA: 15 Redis instances
```

---

## Network Capacity Planning

### Bandwidth Calculation

```
Required Bandwidth = QPS x Response Size x 8 bits x (1 + Overhead)

Protocol overhead:
- TCP/IP: 40 bytes per packet
- HTTP headers: 500-1000 bytes
- TLS: 5-10% overhead

Example:
- 50,000 QPS
- 10 KB average response
- HTTP overhead: 1 KB
- TLS overhead: 10%

Base bandwidth = 50,000 x 11 KB x 8 = 4.4 Gbps
With TLS: 4.4 x 1.1 = 4.84 Gbps
With headroom (30%): 6.3 Gbps
```

### Load Balancer Sizing

```
Load Balancer Capacity:
- Connections per second: 50,000-500,000
- Concurrent connections: 1-10 million
- Throughput: 10-100 Gbps

Example sizing:
- Peak QPS: 100,000
- Average connection duration: 200ms
- Concurrent connections = 100,000 x 0.2 = 20,000

Small LB (50K concurrent): Sufficient
Add redundancy: 2 load balancers (active-passive)
```

---

## Growth Planning

### Capacity Projection Formula

```
Future Capacity = Current Capacity x (1 + Growth Rate)^Years

Example:
- Current servers: 50
- Growth rate: 50% per year

Year 1: 50 x 1.5 = 75 servers
Year 2: 75 x 1.5 = 112 servers
Year 3: 112 x 1.5 = 168 servers

Compound: 50 x (1.5)^3 = 168.75 ≈ 169 servers
```

### Scaling Triggers

| Metric | Warning Threshold | Critical Threshold | Action |
|--------|------------------|-------------------|--------|
| CPU utilization | 60% | 80% | Add servers |
| Memory utilization | 70% | 85% | Add memory/servers |
| Disk utilization | 70% | 85% | Add storage |
| Network utilization | 60% | 80% | Add bandwidth |
| Error rate | 0.1% | 1% | Investigate and scale |
| Latency (P99) | 2x baseline | 5x baseline | Scale or optimize |

### Scaling Strategies

```
VERTICAL SCALING (Scale Up)
├── Pros: Simple, no code changes
├── Cons: Hardware limits, downtime
└── Use when: Quick fix, small scale

HORIZONTAL SCALING (Scale Out)
├── Pros: Linear scaling, fault tolerant
├── Cons: Complexity, data consistency
└── Use when: Large scale, high availability

HYBRID APPROACH
├── Scale up first (simpler)
├── Scale out when limits reached
└── Plan architecture for horizontal from start
```

---

## Worked Examples

### Example 1: Video Streaming Service

**Requirements**:
- 10 million concurrent viewers
- Average bitrate: 5 Mbps
- 3 quality levels (1080p, 720p, 480p)
- 99.9% availability SLA

**Capacity Calculation**:

```
Step 1: Bandwidth calculation
Concurrent viewers: 10M
Average bitrate: 5 Mbps
Total bandwidth = 10M x 5 Mbps = 50 Tbps

Step 2: CDN capacity
CDN POP capacity: 100 Gbps each
POPs needed = 50 Tbps / 100 Gbps = 500 POPs

With 30% headroom: 650 POPs

Step 3: Origin capacity
Cache hit rate: 95%
Origin traffic: 50 Tbps x 0.05 = 2.5 Tbps

Origin servers (10 Gbps each): 250 servers
With redundancy: 375 origin servers

Step 4: Storage for video library
Videos: 10 million
Average duration: 60 minutes
Storage per video: 10 GB (all qualities)
Total: 100 PB

With 3x replication: 300 PB storage

Step 5: Metadata/API servers
Concurrent users making API calls: 10M x 0.1 = 1M QPS
API server capacity: 10,000 QPS
Servers: 100 (+ 50 for redundancy) = 150 API servers
```

### Example 2: Ride-Sharing Platform

**Requirements**:
- 1 million concurrent rides
- Location updates every 4 seconds
- Matching algorithm: 100ms target
- Store 90 days of ride history

**Capacity Calculation**:

```
Step 1: Location update throughput
Updates per second = 1M / 4 = 250,000 updates/sec

Location service servers:
Server capacity: 10,000 updates/sec
Servers needed: 25 (base) + 12 (redundancy) + 8 (headroom) = 45 servers

Step 2: Matching service
Match requests: 50,000/minute (new ride requests)
QPS: 833 matches/sec
Processing time per match: 100ms
Concurrent matches: 833 x 0.1 = 83

CPU-seconds needed: 833 x 0.1 = 83.3 CPU-sec/sec
8-core servers at 70%: 5.6 CPU-sec capacity
Servers: 83.3 / 5.6 = 15 servers (+ redundancy) = 23 servers

Step 3: Database capacity
Active rides: 1M concurrent
Write QPS (location): 250,000/sec

Use time-series DB (InfluxDB/TimescaleDB):
Write capacity: 100,000 writes/sec per node
Nodes needed: 3 (+ replication) = 9 nodes

Step 4: Ride history storage
Rides per day: 10M
Data per ride: 50 KB (GPS trace + metadata)
Daily: 500 GB
90 days: 45 TB
With 3x replication: 135 TB

Step 5: Real-time messaging (driver-rider)
Active conversations: 1M
Messages per minute per conversation: 2
Message QPS: 2M / 60 = 33,333 QPS

Messaging server capacity: 50,000 QPS
Servers: 1 (+ redundancy) = 3 messaging servers
```

### Example 3: E-commerce Flash Sale

**Requirements**:
- Normal traffic: 10,000 QPS
- Flash sale traffic: 500,000 QPS (50x spike)
- Sale duration: 2 hours
- Inventory: 10,000 items
- 99.99% availability during sale

**Capacity Calculation**:

```
Step 1: Web/API tier scaling
Normal servers: 10,000 / 5,000 = 2 servers
Flash sale servers: 500,000 / 5,000 = 100 servers

Auto-scaling approach:
- Pre-scale to 50 servers before sale
- Auto-scale to 120 servers (with headroom)
- Scale down after sale

Step 2: Cache layer (prevent DB overload)
Cache all product data:
- 10,000 products x 5 KB = 50 MB
- Replicate across 5 cache servers for throughput
- Cache read: 500,000 QPS / 5 = 100,000 QPS per server (OK for Redis)

Step 3: Inventory service (critical path)
Dedicated inventory microservice:
- 500,000 QPS attempting purchase
- Inventory check: 10,000 items
- Use Redis for real-time inventory counting
- Single Redis master for consistency
- Read replicas for inventory display

Step 4: Order processing
Assuming 2% conversion: 10,000 orders (all inventory)
Orders per second: 10,000 / 7,200 sec = 1.4 orders/sec

But burst at start: potentially 10,000 in first minute
Burst QPS: 167 orders/sec
Order server capacity: 100 orders/sec
Servers: 2 (+ redundancy) = 4 order servers

Queue orders and process asynchronously:
Message queue: 10,000 message burst capacity
Order processors: 10 workers at 10 orders/sec each

Step 5: Database
Use queue to smooth writes:
Sustained write: 10 orders/sec
Peak: 100 orders/sec (queue buffered)
Single primary DB sufficient

Connection pool: 50 connections (for order processors)
```

---

## Practice Problems

### Problem 1: Chat Application

**Given**:
- 50 million DAU
- 100 messages sent per user per day
- 500 messages received per user per day
- Message size: 1 KB
- Real-time delivery required (<1 second)
- Store messages for 1 year

Calculate server, storage, and network requirements.

<details>
<summary>Solution</summary>

```
Message throughput:
Sent: 50M x 100 / 86,400 = 57,870 msg/sec
Received: 50M x 500 / 86,400 = 289,352 msg/sec
Total: ~350,000 msg/sec

WebSocket servers:
Concurrent connections: 50M x 0.2 (20% online) = 10M
Connections per server: 100,000
Servers: 100 (+ 50 redundancy) = 150 WebSocket servers

Message routing servers:
350,000 msg/sec / 50,000 per server = 7 servers
With redundancy: 12 routing servers

Storage (1 year):
Messages/year: 50M x 100 x 365 = 1.825 trillion
Storage: 1.825T x 1 KB = 1.825 PB
With 3x replication: 5.5 PB
With compression (3x): 1.83 PB

Network:
Outbound: 289,352 x 1 KB x 8 = 2.3 Gbps
With overhead: 3 Gbps

Database:
Write QPS: 57,870 (messages created)
Read QPS: Low (messages pushed, not pulled)
Shards: 1.825 PB / 500 GB = 3,650 shards
```

</details>

### Problem 2: Search Engine

**Given**:
- 100 million searches per day
- Index size: 100 billion documents
- 1 KB per document
- P99 latency target: 200ms
- Index updated hourly

Calculate capacity requirements.

<details>
<summary>Solution</summary>

```
Search QPS:
Average: 100M / 86,400 = 1,157 QPS
Peak (3x): 3,471 QPS

Index size:
100B docs x 1 KB = 100 TB raw
Inverted index: ~2x raw = 200 TB

Search servers:
Assume 500 QPS per server at 200ms P99
Servers: 3,471 / 500 = 7 servers per shard

Sharding:
200 TB / 50 GB per shard = 4,000 shards
Total search servers: 4,000 x 7 = 28,000 servers

With 2 replicas per shard: 84,000 servers

Memory (for fast search):
Hot data in memory: 20 TB (top 10%)
Servers with 256 GB RAM: 80 servers
These handle 80% of queries

Index build servers:
100B docs / hour = 27.8M docs/sec
Processing at 1000 docs/sec/server: 27,800 servers

Storage:
Index: 200 TB x 3 replicas = 600 TB
Document store: 100 TB x 3 = 300 TB
Total: 900 TB
```

</details>

### Problem 3: Gaming Leaderboard

**Given**:
- 10 million concurrent players
- Score updates: 10 per minute per player
- Leaderboard queries: 5 per minute per player
- Top 1000 players shown
- Real-time updates required

Calculate capacity requirements.

<details>
<summary>Solution</summary>

```
Write throughput:
10M players x 10 updates/min = 100M updates/min
= 1.67M updates/sec

Read throughput:
10M players x 5 queries/min = 50M queries/min
= 833,000 queries/sec

Redis sorted set solution:
Write: ZADD - 200,000 ops/sec per instance
Read: ZRANGE (top 1000) - 100,000 ops/sec per instance

Write instances: 1.67M / 200,000 = 9 instances
Read instances: 833,000 / 100,000 = 9 instances

Sharding by game/region:
10 shards, each handles 1/10th of load
Per shard: ~1 write instance, ~1 read instance

With redundancy: 30 Redis instances

Memory:
10M players x 100 bytes (score + ID) = 1 GB
Negligible - fits in single instance

API servers:
Combined QPS: 2.5M ops/sec
API capacity: 20,000 QPS per server
Servers: 125 (+ redundancy) = 175 API servers

WebSocket (for real-time push):
10M connections / 50,000 per server = 200 servers
With redundancy: 300 WebSocket servers
```

</details>

---

## Interview Tips

### Do's

1. **Start with requirements** - DAU, QPS, storage, SLA
2. **Show your math** - Write formulas, then calculate
3. **Use round numbers** - 86,400 ≈ 100,000 for easy math
4. **Consider all resources** - CPU, memory, storage, network
5. **Plan for failure** - Include redundancy in all calculations
6. **Think about peaks** - Normal load vs peak load

### Don'ts

1. **Don't forget growth** - Plan for 2-3 years
2. **Don't ignore network** - Often the bottleneck
3. **Don't assume linear scaling** - Coordination overhead exists
4. **Don't forget dependencies** - Databases, caches, external services
5. **Don't optimize prematurely** - Start simple, scale as needed

### Quick Estimation Tips

```
Quick Math Shortcuts:
- 86,400 seconds/day ≈ 100,000
- 1 million/day ≈ 12/second
- 1 billion/day ≈ 12,000/second

Typical Ratios:
- Peak:Average = 2-5x
- Read:Write = 10:1 to 1000:1
- Cache hit rate = 90-99%

Standard Server Capacity:
- Web server: 5,000-10,000 QPS
- Database: 10,000 read, 5,000 write QPS
- Cache: 100,000+ ops/sec
```

---

## Quick Reference Card

```
CAPACITY PLANNING QUICK REFERENCE

Core Formulas:
─────────────────────────────────────────────
Servers = Peak QPS / QPS per server x (1 + redundancy) x (1 + headroom)
Storage = Objects x Size x Retention x Replication
Bandwidth = QPS x Response Size x 8 bits
DB Connections = QPS x Query Time

Standard Multipliers:
─────────────────────────────────────────────
Redundancy: 1.5x (N+1) to 2x (N+N)
Headroom: 1.3x to 1.5x
Peak traffic: 2x to 5x average
Replication: 3x for databases
Index overhead: 1.2x to 1.5x

Capacity Benchmarks:
─────────────────────────────────────────────
Web server: 5,000-10,000 QPS
API server: 5,000-20,000 QPS
Database (single node): 10,000-50,000 read QPS
Redis: 100,000-500,000 ops/sec
Network per server: 1-10 Gbps

Time Conversions:
─────────────────────────────────────────────
1 day = 86,400 seconds ≈ 100,000
1 week = 604,800 seconds ≈ 600,000
1 month = 2.6M seconds ≈ 2.5M
1 year = 31.5M seconds ≈ 30M
```

---

## Summary

Capacity planning is essential for building scalable systems:

1. **Understand the requirements** - Users, traffic, data, SLAs
2. **Calculate baseline capacity** - QPS, storage, bandwidth
3. **Account for peaks** - 2-5x average traffic
4. **Add redundancy** - N+1 or N+N for availability
5. **Include headroom** - 30-50% for unexpected growth
6. **Plan for growth** - 2-3 year projection
7. **Validate with testing** - Load test before production

The key is to be methodical: gather requirements, apply formulas, add safety factors, and iterate based on real-world data.
