# Common Numbers Every Engineer Should Memorize

## Overview

This quick reference compiles all the essential numbers, formulas, and benchmarks that system design engineers should know by heart. Memorizing these numbers allows you to make quick, accurate estimations during interviews and real-world capacity planning. Focus on orders of magnitude rather than exact values.

---

## Time Conversions

### Seconds Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIME CONVERSION TABLE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1 minute        =  60 seconds                                  │
│  1 hour          =  3,600 seconds        ≈ 4,000 (4 x 10^3)    │
│  1 day           =  86,400 seconds       ≈ 100,000 (10^5)      │
│  1 week          =  604,800 seconds      ≈ 600,000 (6 x 10^5)  │
│  1 month (30d)   =  2,592,000 seconds    ≈ 2.5 million         │
│  1 year          =  31,536,000 seconds   ≈ 30 million (3 x 10^7)│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Quick Conversions

```
PER DAY TO PER SECOND:
─────────────────────────────────────────────
1 million / day     ≈  12 / second
10 million / day    ≈  120 / second
100 million / day   ≈  1,200 / second
1 billion / day     ≈  12,000 / second
10 billion / day    ≈  120,000 / second

PER MONTH TO PER SECOND:
─────────────────────────────────────────────
1 million / month   ≈  0.4 / second
100 million / month ≈  40 / second
1 billion / month   ≈  400 / second
```

---

## Data Size Units

### Storage Units

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA SIZE REFERENCE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1 Byte (B)      =  8 bits                                      │
│  1 Kilobyte (KB) =  1,000 bytes       = 10^3 bytes              │
│  1 Megabyte (MB) =  1,000 KB          = 10^6 bytes              │
│  1 Gigabyte (GB) =  1,000 MB          = 10^9 bytes              │
│  1 Terabyte (TB) =  1,000 GB          = 10^12 bytes             │
│  1 Petabyte (PB) =  1,000 TB          = 10^15 bytes             │
│  1 Exabyte (EB)  =  1,000 PB          = 10^18 bytes             │
│                                                                  │
│  Binary (for memory):                                           │
│  1 KiB = 1,024 bytes                                            │
│  1 MiB = 1,048,576 bytes                                        │
│  1 GiB = 1,073,741,824 bytes                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Quick Storage Math

```
1 billion x 1 byte   =  1 GB
1 billion x 1 KB     =  1 TB
1 billion x 1 MB     =  1 PB
1 million x 1 KB     =  1 GB
1 million x 1 MB     =  1 TB
1 thousand x 1 GB    =  1 TB
```

---

## Latency Numbers

### Memory Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                    LATENCY HIERARCHY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  L1 cache reference                    0.5 ns     (baseline)    │
│  Branch mispredict                     5 ns       (10x L1)      │
│  L2 cache reference                    7 ns       (14x L1)      │
│  Mutex lock/unlock                     25 ns                    │
│  Main memory reference                 100 ns     (200x L1)     │
│  Compress 1KB with Snappy              3 us       (3,000 ns)    │
│  Send 1KB over 1 Gbps network          10 us                    │
│  Read 4KB randomly from SSD            150 us                   │
│  Read 1MB sequentially from memory     250 us                   │
│  Round trip within datacenter          500 us     (0.5 ms)      │
│  Read 1MB sequentially from SSD        1 ms                     │
│  HDD disk seek                         10 ms                    │
│  Read 1MB sequentially from HDD        20 ms                    │
│  Send packet CA → Netherlands → CA     150 ms                   │
│                                                                  │
│  KEY INSIGHT: Each level is roughly 10x slower                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Latency Comparison

| Operation | Time | Relative |
|-----------|------|----------|
| L1 cache | 1 ns | 1 second |
| L2 cache | 10 ns | 10 seconds |
| RAM | 100 ns | 100 seconds |
| SSD random read | 100 us | 1 day |
| HDD seek | 10 ms | 4 months |
| US to Europe RTT | 150 ms | 5 years |

---

## Network Numbers

### Bandwidth Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    NETWORK BANDWIDTH                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Network Type          Bandwidth        Practical (70%)         │
│  ─────────────────────────────────────────────────────────────  │
│  WiFi (typical)        100 Mbps         70 Mbps                 │
│  Ethernet              1 Gbps           700 Mbps                │
│  Fast Ethernet         10 Gbps          7 Gbps                  │
│  Data Center           25-100 Gbps      17-70 Gbps              │
│                                                                  │
│  DATA TRANSFER TIMES (1 Gbps):                                  │
│  ─────────────────────────────────────────────────────────────  │
│  1 KB                  10 us                                    │
│  1 MB                  10 ms                                    │
│  1 GB                  10 seconds                               │
│  1 TB                  3 hours                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Network Latency

| Route | Latency |
|-------|---------|
| Localhost | < 1 ms |
| Same datacenter | 0.5-1 ms |
| Same region (AWS AZs) | 1-2 ms |
| Cross-region (US) | 30-70 ms |
| US to Europe | 80-120 ms |
| US to Asia | 150-200 ms |
| Around the world | 300+ ms |

---

## Availability Numbers

### The Nines

```
┌─────────────────────────────────────────────────────────────────┐
│                    AVAILABILITY TABLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Availability    Downtime/Year    Downtime/Month   Downtime/Day │
│  ─────────────────────────────────────────────────────────────  │
│  99%             3.65 days        7.2 hours        14.4 min     │
│  99.9%           8.76 hours       43.8 min         1.44 min     │
│  99.95%          4.38 hours       21.9 min         43 sec       │
│  99.99%          52.6 min         4.38 min         8.6 sec      │
│  99.999%         5.26 min         26 sec           864 ms       │
│                                                                  │
│  EASY MEMORY:                                                   │
│  - Each 9 adds = 10x harder, 10x less downtime                  │
│  - 99.9% = about 9 hours/year                                   │
│  - 99.99% = about 1 hour/year                                   │
│  - 99.999% = about 5 minutes/year                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Availability Formulas

```
Serial (all must work):    A = A1 × A2 × A3 × ...
Parallel (any can work):   A = 1 - (1-A1) × (1-A2) × ...

QUICK EXAMPLES:
2 servers at 99%:          1 - (0.01)^2 = 99.99%
3 servers at 99%:          1 - (0.01)^3 = 99.9999%
3 components at 99.9%:     0.999^3 = 99.7%
```

---

## Database Numbers

### Capacity Benchmarks

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE BENCHMARKS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DATABASE           READ QPS        WRITE QPS      NOTES        │
│  ─────────────────────────────────────────────────────────────  │
│  MySQL (single)     10K-50K         5K-20K         ACID         │
│  PostgreSQL         10K-50K         5K-20K         ACID         │
│  Redis              100K-500K       100K-500K      In-memory    │
│  Memcached          100K-1M         100K-1M        In-memory    │
│  MongoDB            20K-100K        10K-50K        Document     │
│  Cassandra          20K-100K        20K-100K       Wide column  │
│  DynamoDB           25K/partition   25K/partition  Managed      │
│  Elasticsearch      10K-50K         5K-10K         Search       │
│                                                                  │
│  RULE OF THUMB:                                                 │
│  - Relational: 10K-50K QPS single node                          │
│  - In-memory: 100K+ QPS                                         │
│  - NoSQL: 20K-100K QPS                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Storage Capacity

| Database Type | Recommended Max Size |
|---------------|---------------------|
| MySQL/PostgreSQL | 1-5 TB per instance |
| MongoDB | 2-5 TB per node |
| Cassandra | 1-2 TB per node |
| Elasticsearch | 20-50 GB per shard |
| Redis | 25-100 GB per instance |

---

## Server Capacity Numbers

### Server Benchmarks

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVER CAPACITY GUIDE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SERVER TYPE             QPS CAPACITY     NOTES                 │
│  ─────────────────────────────────────────────────────────────  │
│  Static web server       10K-100K         Nginx serving files   │
│  Dynamic web server      1K-10K           PHP/Ruby/Python       │
│  API server              5K-20K           JSON APIs             │
│  Application server      1K-5K            Business logic        │
│  WebSocket server        50K-100K         Connections (not QPS) │
│                                                                  │
│  MEMORY GUIDANCE:                                               │
│  ─────────────────────────────────────────────────────────────  │
│  Web server              2-4 GB base                            │
│  Application server      4-16 GB                                │
│  Cache server            90% of RAM for data                    │
│  Database                60-80% of RAM for buffer pool          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Connection Limits

| Component | Typical Limit |
|-----------|---------------|
| MySQL connections | 150-500 per instance |
| PostgreSQL connections | 100-300 per instance |
| Redis connections | 10,000+ |
| WebSocket server | 50,000-100,000 per server |
| Load balancer | 100,000-1,000,000 concurrent |

---

## Common Data Sizes

### Object Size Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON OBJECT SIZES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRIMITIVE TYPES:                                               │
│  ─────────────────────────────────────────────────────────────  │
│  Boolean                 1 byte                                 │
│  Char (ASCII)            1 byte                                 │
│  Char (UTF-8)            1-4 bytes (avg 2)                      │
│  Integer (32-bit)        4 bytes                                │
│  Integer (64-bit)        8 bytes                                │
│  Float                   4 bytes                                │
│  Double                  8 bytes                                │
│  UUID                    16 bytes                               │
│  Timestamp               8 bytes                                │
│                                                                  │
│  COMMON OBJECTS:                                                │
│  ─────────────────────────────────────────────────────────────  │
│  Tweet (280 chars)       ~500 bytes                             │
│  URL                     100-200 bytes                          │
│  Email address           50 bytes                               │
│  User profile            1-5 KB                                 │
│  Product listing         2-10 KB                                │
│  Web page (HTML)         50-200 KB                              │
│                                                                  │
│  MEDIA:                                                         │
│  ─────────────────────────────────────────────────────────────  │
│  Thumbnail               10-50 KB                               │
│  Profile photo           100-500 KB                             │
│  High-res image          2-5 MB                                 │
│  Video (1 min, 720p)     50-100 MB                              │
│  Video (1 min, 4K)       300-500 MB                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Traffic Patterns

### Typical Ratios

```
READ/WRITE RATIOS:
─────────────────────────────────────────────
Social media feed:        100:1 to 1000:1
E-commerce catalog:       100:1
URL shortener:            100:1
Chat application:         1:1 to 5:1
Banking transactions:     10:1
Analytics writes:         1:100 (write heavy)

PEAK MULTIPLIERS:
─────────────────────────────────────────────
Normal daily variation:   2x average
Weekly pattern:           2-3x average
Major events/sales:       5-10x average
Viral content:            10-100x average

DAU/MAU RATIOS:
─────────────────────────────────────────────
Social networks:          50-70%
Messaging apps:           70-80%
E-commerce:               20-30%
Enterprise SaaS:          30-50%
Gaming (casual):          10-20%
Gaming (hardcore):        40-60%
```

---

## Quick Formulas

### Essential Calculations

```
┌─────────────────────────────────────────────────────────────────┐
│                    ESSENTIAL FORMULAS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TRAFFIC:                                                       │
│  QPS = Daily Requests / 86,400                                  │
│  Peak QPS = Average QPS × 2 to 5                                │
│                                                                  │
│  STORAGE:                                                       │
│  Storage = Objects × Size × Retention × Replication             │
│                                                                  │
│  BANDWIDTH:                                                     │
│  Bandwidth (bps) = QPS × Data Size × 8                          │
│                                                                  │
│  SERVERS:                                                       │
│  Servers = Peak QPS / QPS per Server × 1.5 (redundancy)         │
│                                                                  │
│  CACHE:                                                         │
│  DB Load = Total QPS × (1 - Cache Hit Rate)                     │
│  Cache Size = Hot Objects × Object Size × 1.3 (overhead)        │
│                                                                  │
│  CONNECTIONS:                                                   │
│  Concurrent = Throughput × Latency (Little's Law)               │
│  Pool Size = QPS × Query Time                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Capacity Planning Multipliers

| Factor | Multiplier | When to Use |
|--------|------------|-------------|
| Peak traffic | 2-5x | Always for production |
| Redundancy (N+1) | 1.5x | Standard HA |
| Redundancy (N+N) | 2x | Critical systems |
| Headroom | 1.3-1.5x | Growth buffer |
| Replication (DB) | 3x | Standard durability |
| Replication (files) | 2-3x | Object storage |
| Index overhead | 1.2-1.5x | Database storage |
| Compression | 0.1-0.5x | Text/logs |

---

## Cheat Sheet Cards

### Card 1: Time & Data

```
╔═══════════════════════════════════════════════════════════════╗
║                    TIME & DATA CHEAT SHEET                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                                ║
║  TIME:                           DATA:                         ║
║  1 day = 86,400s ≈ 10^5         1 billion × 1 KB = 1 TB       ║
║  1 year = 31.5M s ≈ 3×10^7      1 million × 1 MB = 1 TB       ║
║                                                                ║
║  QPS SHORTCUTS:                 SIZE SHORTCUTS:                ║
║  1M/day = 12/sec                ASCII char = 1 byte            ║
║  1B/day = 12K/sec               Integer = 4-8 bytes            ║
║  100M/day = 1.2K/sec            UUID = 16 bytes                ║
║                                                                ║
╚═══════════════════════════════════════════════════════════════╝
```

### Card 2: Latency

```
╔═══════════════════════════════════════════════════════════════╗
║                    LATENCY CHEAT SHEET                         ║
╠═══════════════════════════════════════════════════════════════╣
║                                                                ║
║  MEMORY:                         NETWORK:                      ║
║  L1 cache     = 1 ns            Same DC    = 0.5-1 ms         ║
║  L2 cache     = 10 ns           US cross   = 50-70 ms         ║
║  RAM          = 100 ns          US-Europe  = 100-150 ms       ║
║                                                                ║
║  STORAGE:                        DATABASE:                     ║
║  SSD random   = 100 us          Cache (Redis) = 1 ms          ║
║  SSD seq 1MB  = 1 ms            Index lookup  = 5-10 ms       ║
║  HDD seek     = 10 ms           Full scan     = 100+ ms       ║
║  HDD seq 1MB  = 20 ms                                         ║
║                                                                ║
║  REMEMBER: Each level is ~10-100× slower than the previous    ║
║                                                                ║
╚═══════════════════════════════════════════════════════════════╝
```

### Card 3: Availability

```
╔═══════════════════════════════════════════════════════════════╗
║                    AVAILABILITY CHEAT SHEET                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                                ║
║  THE NINES:                                                    ║
║  99%      = 3.65 days/year     (one nine)                     ║
║  99.9%    = 8.76 hours/year    (three nines)                  ║
║  99.99%   = 52.6 min/year      (four nines)                   ║
║  99.999%  = 5.26 min/year      (five nines)                   ║
║                                                                ║
║  FORMULAS:                                                     ║
║  Serial:    A = A1 × A2 × A3                                  ║
║  Parallel:  A = 1 - (1-A1) × (1-A2)                           ║
║                                                                ║
║  QUICK FACTS:                                                  ║
║  2 servers at 99% = 99.99%                                    ║
║  3 components at 99.9% = 99.7%                                ║
║                                                                ║
╚═══════════════════════════════════════════════════════════════╝
```

### Card 4: Capacity

```
╔═══════════════════════════════════════════════════════════════╗
║                    CAPACITY CHEAT SHEET                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                                ║
║  SERVER CAPACITY:                                              ║
║  Web server        = 10K-50K QPS                              ║
║  API server        = 5K-20K QPS                               ║
║  Database (SQL)    = 10K-50K read, 5K-20K write               ║
║  Cache (Redis)     = 100K-500K ops/sec                        ║
║                                                                ║
║  SIZING FORMULA:                                               ║
║  Servers = (Peak QPS / Server QPS) × 1.5 × 1.3                ║
║            (base)     (redundancy)   (headroom)               ║
║                                                                ║
║  MULTIPLIERS:                                                  ║
║  Peak traffic    = 2-5× average                               ║
║  DB replication  = 3×                                         ║
║  Storage headroom = 1.5×                                      ║
║                                                                ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## Practice: Self-Test

### Quick Calculations

Test yourself on these common conversions:

1. **500 million requests per day = ? QPS**
   <details><summary>Answer</summary>500M / 86,400 ≈ 5,800 QPS (or ~6,000)</details>

2. **10 billion objects × 500 bytes = ? storage**
   <details><summary>Answer</summary>10B × 500 = 5 TB</details>

3. **99.9% availability = ? downtime per month**
   <details><summary>Answer</summary>43.8 minutes (or ~44 minutes)</details>

4. **Three 99% servers in parallel = ? availability**
   <details><summary>Answer</summary>1 - (0.01)^3 = 99.9999%</details>

5. **20,000 QPS at 5ms latency = ? concurrent requests**
   <details><summary>Answer</summary>20,000 × 0.005 = 100 concurrent</details>

6. **1 GB file over 1 Gbps = ? seconds**
   <details><summary>Answer</summary>8 seconds (1 GB = 8 Gb)</details>

7. **100 million users × 50 KB profile = ? storage**
   <details><summary>Answer</summary>100M × 50 KB = 5 TB</details>

8. **If cache hit rate is 95%, database QPS reduced by ?**
   <details><summary>Answer</summary>20× (only 5% of requests hit DB)</details>

---

## Interview Quick Reference

### Before the Interview

Memorize these key numbers:
- 86,400 seconds/day (use 100,000)
- 1M/day = 12/sec, 1B/day = 12K/sec
- L1: 1ns, RAM: 100ns, SSD: 100us, HDD: 10ms
- 99.9% = 9 hours downtime/year
- Redis: 100K ops/sec, SQL: 10-50K QPS

### During the Interview

1. State assumptions clearly
2. Use round numbers (powers of 10)
3. Show calculations step by step
4. Apply multipliers: peak (3x), redundancy (1.5x), headroom (1.3x)
5. Sanity check against known systems

### Common Estimation Pattern

```
1. Users → QPS:      Users × actions / 86,400
2. QPS → Servers:    Peak QPS / server capacity × 1.5
3. Data → Storage:   Objects × size × years × replication
4. QPS → Bandwidth:  QPS × response size × 8
5. Always add:       Redundancy, headroom, indexes
```

---

## Summary

The most critical numbers to memorize:

| Category | Key Number | Why It Matters |
|----------|------------|----------------|
| Time | 86,400 sec/day | QPS calculations |
| Traffic | 1M/day ≈ 12/sec | Quick QPS estimates |
| Latency | RAM=100ns, SSD=100us, HDD=10ms | Architecture decisions |
| Availability | 99.9% = 9 hours/year | SLA planning |
| Capacity | SQL=10K QPS, Redis=100K ops | Server sizing |
| Data | 1B × 1KB = 1TB | Storage estimates |

Remember: The goal is **order of magnitude** accuracy, not precision. If you're within 2-3x of the right answer, you're doing great in an interview.
