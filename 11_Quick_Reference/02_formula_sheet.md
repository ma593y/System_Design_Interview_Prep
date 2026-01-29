# System Design Formula Sheet

> All calculations and formulas in one place

---

## Quick Reference Numbers

### Time Conversions

| Unit | Seconds | Rounded |
|------|---------|---------|
| 1 minute | 60 | 60 |
| 1 hour | 3,600 | 3,600 |
| 1 day | 86,400 | ~100,000 (10^5) |
| 1 week | 604,800 | ~600,000 |
| 1 month | 2,592,000 | ~2.5 million |
| 1 year | 31,536,000 | ~30 million (3 x 10^7) |

### Data Size Conversions

| Unit | Bytes | Power |
|------|-------|-------|
| 1 KB | 1,024 | 10^3 |
| 1 MB | 1,048,576 | 10^6 |
| 1 GB | 1,073,741,824 | 10^9 |
| 1 TB | ~1 trillion | 10^12 |
| 1 PB | ~1 quadrillion | 10^15 |

### Powers of 2

| Power | Value | Approx |
|-------|-------|--------|
| 2^10 | 1,024 | 1 thousand |
| 2^20 | 1,048,576 | 1 million |
| 2^30 | 1,073,741,824 | 1 billion |
| 2^40 | 1,099,511,627,776 | 1 trillion |

---

## QPS (Queries Per Second) Calculations

### Daily Active Users to QPS

```
QPS = (DAU × queries_per_user_per_day) / seconds_per_day

QPS = (DAU × queries_per_user) / 86,400
```

**Simplified:**
```
QPS ≈ (DAU × queries_per_user) / 100,000
```

### Examples

| DAU | Queries/User/Day | QPS |
|-----|------------------|-----|
| 1 million | 10 | ~115 |
| 10 million | 10 | ~1,150 |
| 100 million | 10 | ~11,500 |
| 1 billion | 10 | ~115,000 |

### Peak QPS

```
Peak QPS = Average QPS × Peak Factor

Typical Peak Factor: 2x to 10x (use 3x as default)
```

**Example:**
```
Average QPS = 1,000
Peak QPS = 1,000 × 3 = 3,000 QPS
```

---

## Storage Calculations

### Basic Storage Formula

```
Total Storage = Number of Records × Record Size × Retention Period

Daily Storage = DAU × Records per User × Record Size
Monthly Storage = Daily Storage × 30
Yearly Storage = Daily Storage × 365
```

### Storage with Growth

```
Storage (n years) = Daily Rate × 365 × n × (1 + growth_rate)^n
```

### Common Object Sizes

| Object Type | Typical Size |
|-------------|--------------|
| User ID | 8 bytes (64-bit) |
| UUID | 16 bytes (128-bit) |
| Timestamp | 8 bytes |
| Short text (tweet) | 280 bytes |
| Email | 100 KB average |
| JSON metadata | 1-10 KB |
| Thumbnail image | 10-50 KB |
| Standard image | 200 KB - 1 MB |
| HD image | 2-5 MB |
| Video (1 min, compressed) | 10-50 MB |
| Video (1 min, HD) | 100-150 MB |

### Example: Twitter-like Service

```
Users: 500 million
DAU: 200 million
Tweets/day/user: 2
Tweet size: 280 bytes + 100 bytes metadata = 400 bytes

Daily tweets = 200M × 2 = 400M tweets/day
Daily storage = 400M × 400 bytes = 160 GB/day
Yearly storage = 160 GB × 365 = 58.4 TB/year

With media (10% have images, avg 500KB):
Media storage = 400M × 0.1 × 500KB = 20 TB/day
```

---

## Bandwidth Calculations

### Basic Bandwidth Formula

```
Bandwidth = (Data Size × Requests) / Time

Bandwidth (Mbps) = (Bytes × 8) / (Seconds × 10^6)
Bandwidth (Gbps) = (Bytes × 8) / (Seconds × 10^9)
```

### Ingress vs Egress

```
Ingress (Upload) = Write QPS × Average Request Size
Egress (Download) = Read QPS × Average Response Size

Read:Write ratio typically 10:1 to 100:1
```

### Example: Video Streaming

```
Users watching: 1 million concurrent
Video bitrate: 5 Mbps (1080p)

Egress bandwidth = 1M × 5 Mbps = 5 Tbps
```

### Example: Image Upload Service

```
Uploads: 100 QPS
Average image: 2 MB

Ingress = 100 × 2 MB = 200 MB/s = 1.6 Gbps
```

---

## Availability Calculations

### Uptime Percentages

| Availability | Downtime/Year | Downtime/Month | Downtime/Day |
|--------------|---------------|----------------|--------------|
| 99% (two 9s) | 3.65 days | 7.31 hours | 14.4 min |
| 99.9% (three 9s) | 8.76 hours | 43.8 min | 1.44 min |
| 99.99% (four 9s) | 52.6 min | 4.38 min | 8.64 sec |
| 99.999% (five 9s) | 5.26 min | 26.3 sec | 864 ms |
| 99.9999% (six 9s) | 31.5 sec | 2.63 sec | 86.4 ms |

### Combined Availability

**Sequential (all must work):**
```
Availability = A1 × A2 × A3 × ... × An
```

**Example:** 3 services at 99.9% each
```
Combined = 0.999 × 0.999 × 0.999 = 99.7%
```

**Parallel (at least one must work):**
```
Availability = 1 - (1 - A1) × (1 - A2) × ... × (1 - An)
```

**Example:** 2 servers at 99% each
```
Combined = 1 - (0.01 × 0.01) = 99.99%
```

### Redundancy Impact

| Configuration | Formula | Example (99% each) |
|---------------|---------|-------------------|
| Single | A | 99% |
| 2 parallel | 1 - (1-A)^2 | 99.99% |
| 3 parallel | 1 - (1-A)^3 | 99.9999% |
| 2 sequential | A^2 | 98.01% |
| 3 sequential | A^3 | 97.03% |

---

## Latency Estimations

### Latency Numbers Every Engineer Should Know

| Operation | Time | Rounded |
|-----------|------|---------|
| L1 cache reference | 0.5 ns | 1 ns |
| L2 cache reference | 7 ns | 10 ns |
| Main memory reference | 100 ns | 100 ns |
| SSD random read | 150 μs | 150 μs |
| HDD random read | 10 ms | 10 ms |
| Network round trip (same DC) | 0.5 ms | 500 μs |
| Network round trip (cross DC) | 30-100 ms | 50 ms |
| Network round trip (cross continent) | 100-200 ms | 150 ms |
| Read 1 MB from memory | 250 μs | 250 μs |
| Read 1 MB from SSD | 1 ms | 1 ms |
| Read 1 MB from HDD | 20 ms | 20 ms |
| Read 1 MB over 1 Gbps | 10 ms | 10 ms |

### Latency Comparison

```
Memory is ~100,000x faster than HDD
SSD is ~100x faster than HDD
Network (same DC) is ~5x slower than SSD
```

### P99 Latency Rule of Thumb

```
P99 latency ≈ 3-10x average latency
```

---

## Cache Calculations

### Cache Hit Rate Impact

```
Effective Latency = (Hit Rate × Cache Latency) + (Miss Rate × DB Latency)
```

**Example:**
```
Cache latency: 1 ms
DB latency: 50 ms
Hit rate: 90%

Effective = (0.9 × 1) + (0.1 × 50) = 0.9 + 5 = 5.9 ms
```

### Cache Size Estimation

```
Cache Size = Working Set Size × Overhead Factor

Working Set = Active Users × Data per User
Overhead Factor: 1.5x to 2x for metadata
```

**Example:**
```
Active users: 10 million
Session data: 1 KB per user
Working set = 10M × 1 KB = 10 GB
With overhead = 10 GB × 1.5 = 15 GB cache needed
```

### Cache Efficiency

| Hit Rate | Speedup (Cache 1ms, DB 100ms) |
|----------|-------------------------------|
| 80% | 5x faster |
| 90% | 10x faster |
| 95% | 20x faster |
| 99% | 50x faster |

---

## Database Calculations

### Index Size Estimation

```
Index Size ≈ Number of Rows × Key Size × 2 (B-tree overhead)
```

**Example:**
```
Rows: 100 million
Primary key: 8 bytes
Index size ≈ 100M × 8 × 2 = 1.6 GB
```

### Sharding Calculations

```
Number of Shards = Total Data Size / Target Shard Size
Number of Shards = Total QPS / QPS per Shard
```

**Example:**
```
Total data: 10 TB
Target shard: 500 GB
Shards needed = 10 TB / 500 GB = 20 shards
```

### Replication Factor Impact

```
Storage with Replication = Raw Storage × Replication Factor
Write Throughput = Raw Throughput / Replication Factor
```

---

## Server Capacity Planning

### Web Server Capacity

```
Servers Needed = Peak QPS / QPS per Server

Typical single server: 1,000 - 10,000 QPS
With safety margin: multiply by 1.5
```

### Connection Limits

| Server Type | Typical Limits |
|-------------|----------------|
| Web server connections | 10,000 - 50,000 |
| Database connections | 100 - 500 |
| Redis connections | 10,000 |

### Memory Calculation

```
Memory per Connection = Connection Object + Buffer Size
Total Memory = Connections × Memory per Connection
```

**Example:**
```
Connections: 10,000
Memory per connection: 100 KB
Total = 10,000 × 100 KB = 1 GB for connections
```

---

## Message Queue Calculations

### Throughput Estimation

```
Messages/Second = Events per User × Active Users / 86,400
Queue Lag = Queue Depth / Processing Rate
```

### Partition Count

```
Partitions = Target Throughput / Throughput per Partition
Partitions ≥ Number of Consumers (for parallelism)
```

**Kafka example:**
```
Target: 100,000 msg/sec
Per partition: 10,000 msg/sec
Partitions needed = 100,000 / 10,000 = 10 partitions
```

---

## CDN and Edge Calculations

### CDN Cache Hit Benefits

```
Origin Requests = Total Requests × (1 - Cache Hit Rate)
Bandwidth Savings = Total Bandwidth × Cache Hit Rate
```

**Example:**
```
Total requests: 1 million/day
CDN hit rate: 95%
Origin requests = 1M × 0.05 = 50,000/day
```

### Geographic Latency Reduction

```
Without CDN: User → Origin (200ms)
With CDN: User → Edge (20ms) for cached content
Improvement: 10x faster for cached content
```

---

## Quick Estimation Templates

### Template 1: Social Media Platform

```
Users: 1 billion total, 500M DAU
Posts/day/user: 0.5
Reads/day/user: 100

Write QPS = 500M × 0.5 / 86400 ≈ 3,000 QPS
Read QPS = 500M × 100 / 86400 ≈ 580,000 QPS
Peak Read QPS = 580,000 × 3 ≈ 1.7M QPS
```

### Template 2: E-commerce Platform

```
Products: 10 million
DAU: 50 million
Page views/user: 20
Orders/user: 0.05

Browse QPS = 50M × 20 / 86400 ≈ 11,500 QPS
Order QPS = 50M × 0.05 / 86400 ≈ 30 QPS
Search QPS ≈ Browse QPS × 0.3 ≈ 3,500 QPS
```

### Template 3: Chat Application

```
DAU: 100 million
Messages/user/day: 50
Message size: 500 bytes

Message QPS = 100M × 50 / 86400 ≈ 58,000 QPS
Daily storage = 100M × 50 × 500 = 2.5 TB/day
```

---

## Sanity Check Numbers

| Metric | Reasonable Range |
|--------|------------------|
| QPS per web server | 1K - 10K |
| QPS per DB server | 5K - 20K (depends on queries) |
| QPS per cache server | 100K - 1M |
| Latency (same region) | 1-10 ms |
| Latency (cross region) | 50-200 ms |
| CDN hit rate | 80-95% |
| Cache hit rate | 80-99% |
| Read:Write ratio | 10:1 to 1000:1 |
