# Latency Numbers Every Engineer Should Know

## Overview

Understanding latency is fundamental to system design. These numbers help you make informed decisions about caching strategies, data storage choices, and system architecture. Memorizing these orders of magnitude will make your back-of-envelope calculations much more accurate.

---

## Quick Reference Table

| Operation | Latency | Notes |
|-----------|---------|-------|
| L1 cache reference | 0.5 ns | CPU cache, fastest |
| Branch mispredict | 5 ns | CPU pipeline penalty |
| L2 cache reference | 7 ns | Still on-chip |
| Mutex lock/unlock | 25 ns | Thread synchronization |
| Main memory reference | 100 ns | RAM access |
| Compress 1KB with Zippy | 3 us | Fast compression |
| Send 1KB over 1 Gbps network | 10 us | Network I/O |
| Read 4KB randomly from SSD | 150 us | Solid state storage |
| Read 1MB sequentially from memory | 250 us | Bulk memory read |
| Round trip within same datacenter | 500 us | Network RTT |
| Read 1MB sequentially from SSD | 1 ms | Bulk SSD read |
| Disk seek | 10 ms | HDD mechanical delay |
| Read 1MB sequentially from disk | 20 ms | HDD bulk read |
| Send packet CA -> Netherlands -> CA | 150 ms | Intercontinental RTT |

---

## Memory Hierarchy Latencies

### CPU Caches

```
┌─────────────────────────────────────────────────────────────┐
│                      CPU Core                                │
│  ┌─────────┐                                                │
│  │ L1 Cache│  0.5 ns (4 cycles @ 3GHz)                      │
│  │  32KB   │  Reference: 1x                                 │
│  └────┬────┘                                                │
│       │                                                      │
│  ┌────▼────┐                                                │
│  │ L2 Cache│  7 ns (14 cycles)                              │
│  │  256KB  │  Reference: 14x L1                             │
│  └────┬────┘                                                │
│       │                                                      │
│  ┌────▼────┐                                                │
│  │ L3 Cache│  ~20 ns (shared across cores)                  │
│  │  8-32MB │  Reference: 40x L1                             │
│  └────┬────┘                                                │
└───────┼─────────────────────────────────────────────────────┘
        │
   ┌────▼────┐
   │   RAM   │  100 ns
   │  (DRAM) │  Reference: 200x L1
   └─────────┘
```

### Memory Access Patterns

| Access Type | Latency | Throughput |
|-------------|---------|------------|
| L1 cache hit | 0.5 ns | ~1 TB/s |
| L2 cache hit | 7 ns | ~200 GB/s |
| L3 cache hit | 20 ns | ~100 GB/s |
| RAM (random) | 100 ns | ~20 GB/s |
| RAM (sequential) | 100 ns | ~50 GB/s |

**Key Insight**: Sequential memory access is 5-10x faster than random access due to prefetching and cache line optimization.

---

## Storage Latencies

### SSD vs HDD Comparison

```
Operation               SSD          HDD         Ratio
─────────────────────────────────────────────────────────
Random read (4KB)      150 us       10 ms        67x
Random write (4KB)     200 us       10 ms        50x
Sequential read (1MB)  1 ms         20 ms        20x
Sequential write (1MB) 1 ms         20 ms        20x
IOPS (random)          100K         100-200      500-1000x
```

### Storage Latency Hierarchy

```
┌────────────────────────────────────────────────────────────┐
│ Storage Type        │ Random Read │ Sequential Read (1MB)  │
├────────────────────────────────────────────────────────────┤
│ NVMe SSD            │ 20-100 us   │ 0.5 ms                 │
│ SATA SSD            │ 100-200 us  │ 1-2 ms                 │
│ HDD (7200 RPM)      │ 10-15 ms    │ 20-30 ms               │
│ HDD (5400 RPM)      │ 15-20 ms    │ 30-40 ms               │
│ Network Storage     │ 1-10 ms     │ 10-100 ms              │
│ Cloud Block Storage │ 1-5 ms      │ 5-20 ms                │
└────────────────────────────────────────────────────────────┘
```

### IOPS Calculations

**IOPS** (Input/Output Operations Per Second):

```
SSD IOPS: 10,000 - 500,000 IOPS
HDD IOPS: 75 - 200 IOPS

Formula for HDD IOPS:
IOPS = 1 / (Seek Time + Rotational Latency)

For 7200 RPM drive:
- Seek time: ~8 ms
- Rotational latency: 60s / 7200 RPM / 2 = 4.17 ms
- IOPS = 1 / (0.008 + 0.00417) = ~82 IOPS
```

---

## Network Latencies

### Network Latency Breakdown

```
┌─────────────────────────────────────────────────────────────┐
│                    Network Latencies                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Same Machine (localhost)     │  ~0.5 us                    │
│  ──────────────────────────────────────────────────────────│
│  Same Rack                    │  ~100-500 us                │
│  ──────────────────────────────────────────────────────────│
│  Same Datacenter              │  ~500 us - 1 ms             │
│  ──────────────────────────────────────────────────────────│
│  Cross-Region (same country)  │  ~10-50 ms                  │
│  ──────────────────────────────────────────────────────────│
│  Cross-Continent              │  ~100-200 ms                │
│  ──────────────────────────────────────────────────────────│
│  Intercontinental Round Trip  │  ~150-300 ms                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Network Operation Latencies

| Operation | Latency |
|-----------|---------|
| TCP handshake (same DC) | ~1-2 ms |
| TCP handshake (cross-region) | ~50-100 ms |
| TLS handshake | ~2-4 RTTs |
| DNS lookup (cached) | ~1 ms |
| DNS lookup (uncached) | ~20-120 ms |
| HTTP request (same DC) | ~5-10 ms |
| HTTP request (cross-region) | ~50-200 ms |

### Data Transfer Times

```
Time to send data over network:

Transfer Time = Data Size / Bandwidth + Latency

1 Gbps network:
- 1 KB: 10 us + latency
- 1 MB: 10 ms + latency
- 1 GB: 10 s + latency

10 Gbps network:
- 1 KB: 1 us + latency
- 1 MB: 1 ms + latency
- 1 GB: 1 s + latency
```

---

## Database Latencies

### Common Database Operations

| Operation | In-Memory DB | SSD-Backed | HDD-Backed |
|-----------|--------------|------------|------------|
| Point lookup | 0.1-1 ms | 1-5 ms | 10-20 ms |
| Range scan (100 rows) | 1-5 ms | 5-20 ms | 50-200 ms |
| Write (single row) | 0.1-1 ms | 1-10 ms | 10-50 ms |
| Write (batch 100) | 1-5 ms | 10-50 ms | 100-500 ms |

### Redis Operations

| Operation | Latency |
|-----------|---------|
| GET (local) | 0.1 ms |
| SET (local) | 0.1 ms |
| GET (network) | 1-2 ms |
| MGET (100 keys) | 1-5 ms |
| Pipeline (100 ops) | 2-5 ms |

### SQL Query Latencies

| Query Type | Simple Index | Complex Join | Full Table Scan |
|------------|--------------|--------------|-----------------|
| 1K rows | 1-5 ms | 10-50 ms | 10-100 ms |
| 100K rows | 5-20 ms | 50-500 ms | 100ms-1s |
| 10M rows | 20-100 ms | 500ms-5s | 1-30s |

---

## Cloud Service Latencies

### AWS Service Latencies (Typical)

| Service | Operation | Latency |
|---------|-----------|---------|
| S3 | GET (first byte) | 50-200 ms |
| S3 | PUT (small object) | 100-300 ms |
| DynamoDB | Read (eventually consistent) | 1-5 ms |
| DynamoDB | Read (strongly consistent) | 5-10 ms |
| DynamoDB | Write | 5-10 ms |
| ElastiCache | GET | 0.5-1 ms |
| Lambda | Cold start | 100ms-3s |
| Lambda | Warm invocation | 1-10 ms |
| SQS | Send message | 10-50 ms |
| SNS | Publish | 10-50 ms |

### CDN Latencies

| Scenario | Latency |
|----------|---------|
| CDN cache hit (edge) | 5-20 ms |
| CDN cache miss | 100-500 ms |
| Origin fetch | +100-300 ms |

---

## Latency Calculation Formulas

### Total Request Latency

```
Total Latency = Network RTT + Processing Time + Queue Wait + Data Transfer

Example - API Call with Database:
- Network RTT (client to server): 50 ms
- Server processing: 5 ms
- Database query: 10 ms
- Response transfer: 5 ms
- Total: ~70 ms
```

### Estimating Sequential Operations

```
Sequential Operations = Sum of all latencies

Example - User Login Flow:
1. DNS lookup: 20 ms
2. TCP handshake: 30 ms
3. TLS handshake: 60 ms (2 RTTs)
4. HTTP request: 30 ms
5. Auth service call: 50 ms
6. Database lookup: 10 ms
7. Response: 30 ms
Total: ~230 ms
```

### P99 Latency Estimation

```
P99 ≈ Average × (2 to 5) for typical distributions

If average = 50ms
P99 ≈ 100-250 ms

For log-normal distributions (common):
P99 ≈ Average × e^(2.33 × σ)
where σ is standard deviation of log(latency)
```

---

## Latency Budget Planning

### Example: 200ms Total Budget

```
┌─────────────────────────────────────────────────────────────┐
│                  Latency Budget: 200ms                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Network (client → LB)      │ 30 ms  │ ████████             │
│  Load Balancer              │ 1 ms   │ █                    │
│  Application Server         │ 20 ms  │ ██████               │
│  Cache Lookup               │ 2 ms   │ █                    │
│  Database Query             │ 50 ms  │ ████████████████     │
│  External API Call          │ 80 ms  │ ████████████████████ │
│  Response Serialization     │ 5 ms   │ ██                   │
│  Network (LB → client)      │ 12 ms  │ ████                 │
│  ───────────────────────────────────────────────────────── │
│  Total                      │ 200 ms │                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Latency SLA Guidelines

| Tier | Latency | Use Case |
|------|---------|----------|
| Real-time | < 10 ms | Gaming, trading |
| Interactive | < 100 ms | Web API, search |
| Responsive | < 1 s | Page loads, reports |
| Batch | > 1 s | Analytics, processing |

---

## Practice Problems

### Problem 1: Estimate Page Load Time

**Scenario**: User loads a webpage that requires:
- 1 HTML file (10 KB)
- 3 CSS files (50 KB total)
- 5 JavaScript files (200 KB total)
- 20 images (2 MB total)
- 3 API calls to backend

**Given**:
- User is 100 ms RTT from server
- Server is on 1 Gbps network
- CDN serves static assets (20 ms RTT)

<details>
<summary>Solution</summary>

```
Initial Request:
- DNS lookup: 50 ms (first time)
- TCP + TLS: 3 × 100 ms = 300 ms
- HTML fetch: 100 ms RTT + 10KB/125MB/s = ~100 ms

Static Assets (parallel, from CDN):
- TCP + TLS: 3 × 20 ms = 60 ms
- Largest asset: 2 MB / 125 MB/s = 16 ms
- CDN RTT: 20 ms
- Total: ~100 ms (parallel)

API Calls (assume sequential):
- 3 × (100 ms RTT + 50 ms processing) = 450 ms

Total estimated: 100 + 300 + 100 + 100 + 450 = ~1050 ms

With HTTP/2 multiplexing and parallel API calls:
~600-800 ms
```

</details>

### Problem 2: Cache vs Database Decision

**Scenario**: You have an endpoint that:
- Receives 10,000 requests per second
- Currently hits database (10 ms average)
- Cache hit would be 1 ms
- Expected cache hit rate: 95%

What's the improvement in average latency?

<details>
<summary>Solution</summary>

```
Without cache:
Average latency = 10 ms

With cache (95% hit rate):
Average latency = (0.95 × 1 ms) + (0.05 × 10 ms)
                = 0.95 ms + 0.5 ms
                = 1.45 ms

Improvement: 10 ms → 1.45 ms = 6.9x faster

Database load reduction:
Before: 10,000 QPS
After: 10,000 × 0.05 = 500 QPS (95% reduction)
```

</details>

### Problem 3: Cross-Region Latency

**Scenario**: Design a system where:
- Users are in US-East, US-West, and Europe
- Data must be consistent
- Target latency: < 100 ms for reads

Should you use single-region or multi-region deployment?

<details>
<summary>Solution</summary>

```
Single Region (US-East):
- US-East users: 20 ms
- US-West users: 70 ms
- Europe users: 150 ms ❌ Exceeds target

Multi-Region with Replication:
- Each region: 20 ms for local reads
- Write propagation: eventual (async) or strong (sync)

For strong consistency:
- Writes require quorum across regions
- Write latency = max(region RTTs) = ~150 ms

For eventual consistency:
- Local reads: 20 ms ✓
- Writes: 20 ms (local ack)
- Replication lag: 100-500 ms

Recommendation: Multi-region with eventual consistency
for reads, single-leader for writes to maintain
consistency with acceptable write latency.
```

</details>

---

## Interview Tips

### Do's

1. **Memorize orders of magnitude** - Know that RAM is ~100 ns, SSD is ~100 us, HDD is ~10 ms
2. **State your assumptions** - "I'm assuming a 1 Gbps network and SSD storage"
3. **Use round numbers** - "Let's say 100 ms for cross-region latency"
4. **Consider P99 vs average** - "The average is 50 ms, but P99 might be 200 ms"
5. **Think about tail latency** - "At high percentiles, we need to account for..."

### Don'ts

1. **Don't memorize exact numbers** - Orders of magnitude matter more
2. **Don't forget network latency** - It's often the dominant factor
3. **Don't ignore cold start** - First request is often much slower
4. **Don't assume infinite bandwidth** - Network can be a bottleneck

### Common Interview Questions

1. "Why is this system slow?" - Start with latency breakdown
2. "How would you reduce latency?" - Caching, CDN, async processing
3. "What's the theoretical minimum latency?" - Speed of light calculation
4. "How do you handle tail latency?" - Hedged requests, timeouts

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│              LATENCY NUMBERS CHEAT SHEET                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1 ns    = L1 cache (×2)                                    │
│  10 ns   = L2 cache, branch mispredict                      │
│  100 ns  = RAM access, mutex lock                           │
│  1 us    = 1KB network send                                 │
│  10 us   = SSD read, context switch                         │
│  100 us  = SSD random read, same-rack network               │
│  1 ms    = SSD sequential MB, same-DC network               │
│  10 ms   = HDD seek, cross-region network                   │
│  100 ms  = intercontinental RTT, TLS handshake              │
│  1 s     = bad response time                                │
│                                                              │
│  Remember: Each step is roughly 10× slower                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

- **Memory**: L1 (0.5 ns) → L2 (7 ns) → RAM (100 ns)
- **Storage**: SSD (100 us) → HDD (10 ms) - 100× difference
- **Network**: Same DC (0.5 ms) → Cross-region (50 ms) → Intercontinental (150 ms)
- **Database**: Cache (1 ms) → Index lookup (5 ms) → Table scan (100+ ms)

The key insight is understanding **orders of magnitude** - each category is roughly 10-100× different from the next. This helps you quickly identify bottlenecks and make architecture decisions.
