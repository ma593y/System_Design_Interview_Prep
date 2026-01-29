# Throughput Calculations

## Overview

Throughput measures how much work a system can handle over time. Understanding throughput calculations is essential for capacity planning, identifying bottlenecks, and ensuring your system can handle expected load. This guide covers QPS, TPS, bandwidth, and related metrics.

---

## Key Metrics and Definitions

### Requests Per Second (RPS/QPS)

**QPS** (Queries Per Second) and **RPS** (Requests Per Second) are interchangeable terms measuring the number of requests a system handles per second.

```
QPS = Total Requests / Time Period (seconds)

Daily Active Users to QPS:
QPS = (DAU × Requests per User per Day) / 86,400 seconds

Peak QPS (rule of thumb):
Peak QPS = Average QPS × 2 to 3
```

### Transactions Per Second (TPS)

**TPS** measures complete transactions (which may involve multiple operations).

```
TPS = Completed Transactions / Time Period (seconds)

Relationship to QPS:
If 1 transaction = 5 database queries:
TPS = QPS / 5
```

### Throughput vs Latency

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Throughput = Work / Time                                   │
│  Latency = Time to complete single unit of work             │
│                                                              │
│  Little's Law:                                              │
│  Concurrent Requests = Throughput × Latency                 │
│                                                              │
│  Example:                                                    │
│  If throughput = 1000 RPS and latency = 100ms               │
│  Concurrent requests = 1000 × 0.1 = 100                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## QPS Calculation Examples

### Example 1: Social Media Platform

**Given**:
- 500 million DAU
- Each user:
  - Opens app 5 times per day
  - Views 10 posts per session
  - Each post view = 1 API call
  - 20% of users post once per day (1 write)
  - 10% of users comment (5 comments average)

**Calculate Read and Write QPS**:

```
Read QPS:
- Post views: 500M × 5 × 10 = 25 billion reads/day
- Read QPS: 25B / 86,400 = ~290,000 QPS

Write QPS:
- Posts: 500M × 0.2 × 1 = 100M writes/day
- Comments: 500M × 0.1 × 5 = 250M writes/day
- Total writes: 350M/day
- Write QPS: 350M / 86,400 = ~4,000 QPS

Peak QPS (3× average):
- Peak Read QPS: ~870,000 QPS
- Peak Write QPS: ~12,000 QPS

Read:Write Ratio = 290,000 : 4,000 = ~70:1
```

### Example 2: E-commerce Platform

**Given**:
- 100 million monthly active users
- Average session: 15 minutes
- 10 page views per session
- 2% conversion rate
- Each checkout = 5 DB transactions

**Calculate**:

```
Sessions per day:
- MAU to DAU ratio: ~30% (typical)
- DAU: 100M × 0.3 = 30M
- Sessions/day: 30M (assume 1 session/user/day)

Page View QPS:
- Total page views: 30M × 10 = 300M/day
- QPS: 300M / 86,400 = ~3,500 QPS

Transaction TPS:
- Checkouts/day: 30M × 0.02 = 600K
- DB transactions: 600K × 5 = 3M/day
- TPS: 3M / 86,400 = ~35 TPS

Peak (assume 3× during sales):
- Peak Page View QPS: ~10,500 QPS
- Peak Checkout TPS: ~105 TPS
```

### Example 3: URL Shortener

**Given**:
- 100 million URLs created per month
- Read:Write ratio = 100:1
- URLs never deleted

**Calculate**:

```
Write QPS:
- URLs/day: 100M / 30 = 3.3M/day
- Write QPS: 3.3M / 86,400 = ~40 QPS

Read QPS:
- 100:1 ratio
- Read QPS: 40 × 100 = 4,000 QPS

Peak QPS:
- Peak Write: ~120 QPS
- Peak Read: ~12,000 QPS
```

---

## Bandwidth Calculations

### Formula

```
Bandwidth = Data Size × Requests per Second

Units:
- 1 Kbps = 1,000 bits per second
- 1 Mbps = 1,000,000 bits per second
- 1 Gbps = 1,000,000,000 bits per second

Bytes to Bits: multiply by 8
```

### Bandwidth Calculation Examples

#### Example 1: Video Streaming Service

**Given**:
- 10 million concurrent users
- Average video bitrate: 5 Mbps
- 720p quality

```
Total Bandwidth:
= Concurrent Users × Bitrate
= 10M × 5 Mbps
= 50 Tbps (Terabits per second)
= 50,000 Gbps

With multiple quality levels:
- 20% at 1080p (8 Mbps): 2M × 8 = 16 Tbps
- 50% at 720p (5 Mbps): 5M × 5 = 25 Tbps
- 30% at 480p (2 Mbps): 3M × 2 = 6 Tbps
- Total: 47 Tbps
```

#### Example 2: Image-Heavy Social Media

**Given**:
- 100,000 QPS for image requests
- Average image size: 200 KB
- 80% cache hit rate (served from CDN)

```
Origin Server Bandwidth:
- Cache misses: 100,000 × 0.2 = 20,000 QPS
- Bandwidth: 20,000 × 200 KB × 8 bits
= 20,000 × 1,600,000 bits
= 32 Gbps to origin

CDN Bandwidth:
- Cache hits: 100,000 × 0.8 = 80,000 QPS
- Bandwidth: 80,000 × 200 KB × 8
= 128 Gbps from CDN
```

#### Example 3: API Service

**Given**:
- 50,000 QPS
- Average request: 1 KB
- Average response: 10 KB

```
Inbound Bandwidth:
= 50,000 × 1 KB × 8
= 400 Mbps

Outbound Bandwidth:
= 50,000 × 10 KB × 8
= 4 Gbps

Total: ~4.4 Gbps
```

---

## Database Throughput

### Read Throughput

```
Read Throughput = Queries Per Second × Data Per Query

Factors affecting read throughput:
- Index efficiency
- Cache hit rate
- Connection pool size
- Query complexity
```

### Write Throughput

```
Write Throughput = Writes Per Second × Data Per Write

Factors affecting write throughput:
- Write ahead logging (WAL)
- Replication factor
- Index updates
- Transaction isolation level
```

### Database Throughput Benchmarks

| Database | Read QPS (single node) | Write QPS (single node) |
|----------|------------------------|-------------------------|
| MySQL | 10K-50K | 5K-20K |
| PostgreSQL | 10K-50K | 5K-20K |
| Redis | 100K-500K | 100K-500K |
| MongoDB | 20K-100K | 10K-50K |
| Cassandra | 20K-100K | 20K-100K |
| DynamoDB | 25K (per partition) | 25K (per partition) |

### Calculating Database Capacity

```
Example: E-commerce Product Catalog

Requirements:
- 10,000 read QPS
- 100 write QPS
- Single MySQL instance: 30K read QPS, 10K write QPS

Read replicas needed:
- Minimum: 10,000 / 30,000 = 0.33 → 1 replica
- With headroom (50%): 1 replica

Write capacity:
- 100 / 10,000 = 0.01 → 1 master sufficient

For 3× growth:
- Read: 30,000 QPS → 1-2 replicas
- Write: 300 QPS → still 1 master
```

---

## Network Throughput

### Theoretical vs Practical Throughput

```
Network Type     Theoretical    Practical (70-80%)
────────────────────────────────────────────────
1 Gbps           1 Gbps         700-800 Mbps
10 Gbps          10 Gbps        7-8 Gbps
25 Gbps          25 Gbps        17-20 Gbps
100 Gbps         100 Gbps       70-80 Gbps
```

### TCP Throughput Calculation

```
Max TCP Throughput = Window Size / RTT

Example:
- Window size: 64 KB (default)
- RTT: 100 ms

Throughput = 64 KB / 0.1 s = 640 KB/s = 5.12 Mbps

With window scaling (1 MB window):
Throughput = 1 MB / 0.1 s = 10 MB/s = 80 Mbps
```

### Bandwidth-Delay Product

```
BDP = Bandwidth × RTT

Example:
- 1 Gbps link
- 50 ms RTT

BDP = 1 Gbps × 0.05 s = 50 Mb = 6.25 MB

This is the amount of data "in flight" needed
to fully utilize the link.
```

---

## Throughput Optimization Techniques

### 1. Batching

```
Without batching:
- 1000 requests, 5ms each
- Total: 5 seconds

With batching (100 items per batch):
- 10 batch requests, 20ms each
- Total: 200ms
- 25× improvement

Trade-off: Higher latency per individual item
```

### 2. Connection Pooling

```
Without pooling:
- Each request: connection setup (50ms) + query (5ms)
- 1000 requests: 55 seconds

With pooling (10 connections):
- Connection setup: once
- 1000 requests / 10 connections = 100 sequential queries each
- Total: 100 × 5ms × 10 parallel = 500ms
- 100× improvement
```

### 3. Caching Impact

```
Cache hit rate impact on throughput:

Without cache:
- Database: 10K QPS max
- 10K requests → 10K DB queries

With 90% cache hit:
- 10K requests → 1K DB queries
- Can handle 100K requests (10× throughput)

With 99% cache hit:
- 10K requests → 100 DB queries
- Can handle 1M requests (100× throughput)
```

### 4. Async Processing

```
Synchronous:
- Request → Process → Store → Respond
- Throughput limited by slowest step

Asynchronous:
- Request → Queue → Respond (immediate)
- Background: Process → Store
- Throughput limited only by queue ingestion
```

---

## Throughput Formulas Reference

### Request-Based Calculations

```
QPS from DAU:
QPS = (DAU × requests_per_user) / 86,400

Peak QPS:
Peak_QPS = Average_QPS × peak_factor (typically 2-5)

Read/Write Split:
Read_QPS = Total_QPS × read_percentage
Write_QPS = Total_QPS × write_percentage
```

### Bandwidth Calculations

```
Bandwidth (bps):
Bandwidth = QPS × data_size × 8

Bandwidth with compression:
Effective_Bandwidth = Raw_Bandwidth × compression_ratio

Required bandwidth:
Required = Data_transferred / Time_available
```

### Server Capacity

```
Servers needed:
Servers = QPS / QPS_per_server

With redundancy:
Servers = (QPS / QPS_per_server) × redundancy_factor

Little's Law:
Concurrent_requests = Throughput × Latency
```

---

## Practice Problems

### Problem 1: Twitter-like Service

**Given**:
- 400 million DAU
- Each user tweets 2 times per day
- Each user reads 200 tweets per day
- Tweet size: 500 bytes
- 30% of tweets have images (100 KB average)

Calculate:
1. Read and Write QPS
2. Storage bandwidth
3. Network bandwidth

<details>
<summary>Solution</summary>

```
1. QPS Calculations:
Read QPS:
= (400M × 200) / 86,400
= 80B / 86,400
= ~926,000 QPS

Write QPS:
= (400M × 2) / 86,400
= 800M / 86,400
= ~9,300 QPS

Peak (3× average):
Peak Read: ~2.8M QPS
Peak Write: ~28K QPS

2. Storage Bandwidth:
Text writes:
= 9,300 × 500 bytes
= 4.65 MB/s

Image writes:
= 9,300 × 0.3 × 100 KB
= 279 MB/s

Total write bandwidth: ~284 MB/s

3. Network Bandwidth:
Text reads:
= 926,000 × 500 bytes × 8 bits
= 3.7 Gbps

Image reads (assume 30% include images):
= 926,000 × 0.3 × 100 KB × 8 bits
= 222 Gbps

Total outbound: ~226 Gbps
```

</details>

### Problem 2: Video Upload Service

**Given**:
- 10 million videos uploaded per day
- Average video size: 500 MB
- Processing creates 4 quality versions (total 2 GB)
- Processing time: 10 minutes per video

Calculate:
1. Upload bandwidth
2. Number of processing servers needed
3. Storage write throughput

<details>
<summary>Solution</summary>

```
1. Upload Bandwidth:
Total upload data: 10M × 500 MB = 5 PB/day
Upload bandwidth: 5 PB / 86,400 s
= 57.9 GB/s = 463 Gbps

Peak (2× average): ~926 Gbps

2. Processing Servers:
Videos to process: 10M / day = 115.7 videos/second

Processing capacity per server:
- 10 min per video
- 6 videos/hour = 144 videos/day

Servers needed:
= 10M / 144 = ~69,500 servers

With 50% headroom: ~100,000 processing servers

3. Storage Write Throughput:
Output data: 10M × 2 GB = 20 PB/day
Write throughput: 20 PB / 86,400 s
= 231.5 GB/s = 1.85 Tbps
```

</details>

### Problem 3: Real-time Messaging

**Given**:
- 100 million concurrent users
- Average user sends 50 messages per day
- Average user receives 200 messages per day
- Message size: 1 KB
- 99.9% delivery within 100ms required

Calculate:
1. Message throughput
2. Server requirements if each server handles 50K connections

<details>
<summary>Solution</summary>

```
1. Message Throughput:
Assume concurrent users represent 30% of DAU
DAU = 100M / 0.3 = 333M

Messages sent per day: 333M × 50 = 16.7B
Messages received per day: 333M × 200 = 66.7B

Send QPS: 16.7B / 86,400 = ~193K QPS
Receive QPS: 66.7B / 86,400 = ~772K QPS

Peak QPS (3×):
Peak Send: ~579K QPS
Peak Receive: ~2.3M QPS

2. Server Requirements:
Connection servers:
= 100M connections / 50K per server
= 2,000 servers

For redundancy (N+1):
= 2,000 × 1.5 = 3,000 servers

Message routing bandwidth:
= 193K × 1 KB × 8 = 1.5 Gbps outbound
```

</details>

---

## Interview Tips

### Key Points to Remember

1. **Always start with user numbers**
   - DAU/MAU → requests per user → QPS

2. **Distinguish read vs write**
   - Most systems are read-heavy (10:1 to 1000:1)
   - Helps determine architecture (replicas, caching)

3. **Account for peak traffic**
   - Use 2-5× multiplier
   - Consider time-of-day patterns

4. **Include overhead**
   - Protocol overhead (TCP/HTTP headers)
   - Metadata and indexes
   - Replication factor

### Common Mistakes to Avoid

1. **Forgetting to convert units**
   - Always specify bytes vs bits
   - Be explicit about time units

2. **Ignoring network overhead**
   - HTTP headers add ~500-1000 bytes
   - TCP/IP adds ~40 bytes per packet

3. **Not considering redundancy**
   - Production needs N+1 or N+2
   - Data replication multiplies storage

4. **Assuming linear scaling**
   - Systems rarely scale linearly
   - Account for coordination overhead

### Quick Estimation Shortcuts

```
1 million / day ≈ 12 per second
1 billion / day ≈ 12,000 per second
86,400 ≈ 100,000 (for easy math)

1 GB/s = 8 Gbps
1 TB/day = 100 Mbps
1 PB/day = 100 Gbps
```

---

## Quick Reference Table

| Metric | Formula | Example |
|--------|---------|---------|
| QPS | requests / seconds | 1M/day = 12 QPS |
| Peak QPS | avg QPS × 2-5 | 12 × 3 = 36 QPS |
| Bandwidth | QPS × size × 8 | 1000 × 1KB × 8 = 8 Mbps |
| Servers | QPS / QPS_per_server | 10K / 1K = 10 servers |
| Concurrent | throughput × latency | 1000 × 0.1 = 100 |
| Cache benefit | 1 / (1 - hit_rate) | 90% hit = 10× capacity |

---

## Summary

Throughput calculations are fundamental to system design. Remember:

1. **Start with business metrics** - DAU, requests per user, data sizes
2. **Convert to technical metrics** - QPS, bandwidth, IOPS
3. **Account for peaks** - Systems must handle 2-5× average load
4. **Consider read/write ratios** - Determines caching and replication strategy
5. **Include all overhead** - Protocol headers, replication, redundancy
6. **Use Little's Law** - Connects throughput, latency, and concurrency
