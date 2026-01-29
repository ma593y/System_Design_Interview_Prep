# Latency vs Throughput

## Overview

Latency and throughput are often at odds. Optimizing for one can negatively impact the other.

## Definitions

```
Latency: Time to complete a single operation
┌─────────────────────────────────────────────────────────────────┐
│  Request ─────────────────────────────────────────▶ Response   │
│           └──────────── 50ms latency ──────────────┘           │
└─────────────────────────────────────────────────────────────────┘

Throughput: Operations completed per unit time
┌─────────────────────────────────────────────────────────────────┐
│  1 second:                                                      │
│  Request 1 ──▶ Response 1                                      │
│  Request 2 ──▶ Response 2                                      │
│  Request 3 ──▶ Response 3                                      │
│  ...                                                            │
│  Request 1000 ──▶ Response 1000                                │
│                                                                 │
│  Throughput: 1000 requests/second                              │
└─────────────────────────────────────────────────────────────────┘
```

## The Trade-off

### Lower Latency Often Means Lower Throughput

```
Scenario: Database writes

Low Latency (sync, single write):
Client ──▶ Write ──▶ Disk Flush ──▶ ACK
                     └── 5ms ──┘
Latency: 5ms
Throughput: 200 writes/sec per connection

High Throughput (async, batched):
Client ──▶ Buffer ──────────────────────────────┐
Client ──▶ Buffer                               │
Client ──▶ Buffer ──────────────────────▶ Batch Write ──▶ Disk
Client ──▶ Buffer                        (every 100ms)
                                                │
                                         ACK all ◀──────┘
Latency: up to 100ms
Throughput: 10,000 writes/sec
```

### Higher Throughput Often Means Higher Latency

```
Scenario: Request queuing

Low throughput server (1 worker):
─────▶ ┌───────────┐ ─────▶
       │  Worker   │           Latency: 10ms (no queue)
─────▶ └───────────┘ ─────▶    Throughput: 100/sec

High throughput server (1 worker, queue):
─────▶ ┌───────────────────┐  ┌───────────┐ ─────▶
─────▶ │ Queue (waiting)   │──│  Worker   │ ─────▶
─────▶ └───────────────────┘  └───────────┘ ─────▶
       └── wait time ───┘
                               Latency: 10ms + wait time
                               Throughput: 100/sec (still limited)

Solution: More workers
─────▶ ┌───────────┐ ─────▶
       │ Worker 1  │
─────▶ ├───────────┤ ─────▶   Latency: ~10ms
       │ Worker 2  │           Throughput: 1000/sec
─────▶ ├───────────┤ ─────▶
       │ Worker 3  │
       └───────────┘
```

## Optimization Techniques

### Batching (Throughput over Latency)

```
Individual Operations:
Op1: ──▶ 5ms
Op2: ──▶ 5ms
Op3: ──▶ 5ms
Total: 15ms, 3 operations

Batched:
Ops 1-3: ──▶ 8ms (batch)
Total: 8ms, 3 operations

Trade-off:
- Throughput: ↑ (more ops in less time)
- Latency: ↑ (wait for batch to fill)
```

**Examples:**
- Database bulk inserts
- Kafka message batching
- Network packet coalescing

### Caching (Latency over Throughput)

```
Without Cache:
Client ──▶ API ──▶ DB ──▶ 50ms

With Cache:
Client ──▶ Cache ──▶ 2ms (hit)
Client ──▶ Cache ──▶ API ──▶ DB ──▶ 55ms (miss + cache write)

Trade-off:
- Latency: ↓↓ for cache hits
- Throughput: Depends on hit ratio
- Memory: ↑ (cache storage)
```

### Parallelization (Both, with overhead)

```
Sequential:
Task A ──────────▶ 100ms
                   Task B ──────────▶ 100ms
                                      Task C ──────────▶ 100ms
Total: 300ms

Parallel:
Task A ──────────▶ 100ms
Task B ──────────▶ 100ms    Total: 100ms + overhead
Task C ──────────▶ 100ms

Trade-off:
- Latency: ↓ (parallel execution)
- Throughput: ↑ (more work done)
- Complexity: ↑ (coordination overhead)
- Resources: ↑ (more threads/processes)
```

### Async Processing (Latency for Perceived Latency)

```
Synchronous:
Client ──▶ Process ──▶ Email ──▶ Notify ──▶ Response
           └────────────── 2000ms ──────────┘

Asynchronous:
Client ──▶ Process ──▶ Queue ──▶ Response (200ms)
                         │
                         └──▶ Email ──▶ Notify (background)

Trade-off:
- Perceived latency: ↓↓
- Actual completion: Same or longer
- Complexity: ↑ (queue management)
```

## Scenarios and Decisions

### 1. Real-time Gaming

```
Priority: LATENCY

Every millisecond matters:
┌────────────────────────────────────────────────┐
│ Player Input ──▶ Server ──▶ Game State ──▶ All│
│                                                │
│ Target: < 50ms round trip                      │
│                                                │
│ Techniques:                                    │
│ - UDP over TCP (no retransmit delay)          │
│ - Edge servers (proximity)                     │
│ - Predictive algorithms                        │
│ - Small, frequent updates                      │
└────────────────────────────────────────────────┘
```

### 2. Log Aggregation

```
Priority: THROUGHPUT

Volume matters more than speed:
┌────────────────────────────────────────────────┐
│ Logs ──▶ Buffer ──▶ Batch ──▶ Storage         │
│                                                │
│ Target: 1M+ events/second                      │
│                                                │
│ Techniques:                                    │
│ - Batch writes (seconds of buffering)          │
│ - Async processing                             │
│ - Compression                                  │
│ - Partitioned storage                          │
└────────────────────────────────────────────────┘
```

### 3. E-commerce Checkout

```
Priority: BALANCED (but latency slightly higher)

User experience + reliability:
┌────────────────────────────────────────────────┐
│ Add to Cart: Low latency (< 200ms)             │
│ Checkout: Medium latency (< 2s), high reliability│
│ Inventory: High throughput, eventual consistency │
│                                                 │
│ Techniques:                                     │
│ - Cache product data                           │
│ - Async inventory updates                       │
│ - Sync payment processing                       │
└────────────────────────────────────────────────┘
```

### 4. Video Streaming

```
Priority: THROUGHPUT (sustained)

Continuous data flow:
┌────────────────────────────────────────────────┐
│ Initial buffer: Accept latency (2-5 sec)       │
│ Sustained: High throughput (Mbps)              │
│                                                 │
│ Techniques:                                     │
│ - Large buffers                                │
│ - Adaptive bitrate                             │
│ - CDN distribution                             │
│ - Pre-fetching                                 │
└────────────────────────────────────────────────┘
```

## Latency Percentiles

### Why Percentiles Matter

```
Average latency: 50ms
But...

Distribution:
┌─────────────────────────────────────────────┐
│         ████████████                        │
│         ████████████                        │
│        ████████████████                     │
│       ████████████████████                  │
│      ████████████████████████    ▓▓        │
│     ████████████████████████████ ▓▓▓       │
└──────────────────────────────────────────────
     0ms        50ms        100ms      500ms

p50: 50ms (median)
p95: 120ms (95% of requests)
p99: 300ms (99% of requests)
p99.9: 500ms (worst cases)
```

### Tail Latency Amplification

```
Single service p99: 10ms

Fanout to 100 services:
┌───────────────────────────────────────────────────────────────┐
│ Request ──┬─▶ Service 1 ──▶                                  │
│           ├─▶ Service 2 ──▶                                  │
│           ├─▶ Service 3 ──▶   Wait for ALL to respond       │
│           │   ...                                            │
│           └─▶ Service 100 ──▶                                │
│                                                               │
│ Probability at least one hits p99:                           │
│ 1 - (0.99)^100 = 63%                                         │
│                                                               │
│ Effective p99 of combined request: Much higher!              │
└───────────────────────────────────────────────────────────────┘

Mitigation:
- Hedged requests (send to multiple, take first response)
- Timeouts and fallbacks
- Reduce fanout
```

## Queueing Theory Basics

### Little's Law

```
L = λ × W

L = Average number in system
λ = Arrival rate
W = Average wait time

Example:
λ = 100 requests/second
W = 50ms = 0.05 seconds
L = 100 × 0.05 = 5 requests in system at any time
```

### Utilization and Latency

```
As utilization → 100%, latency → ∞

Utilization (ρ) = λ / μ
Where μ = service rate

Wait time ∝ 1 / (1 - ρ)

┌─────────────────────────────────────────────────────────────┐
│ Latency                                                      │
│    │                                                   ╱     │
│    │                                                 ╱       │
│    │                                               ╱         │
│    │                                             ╱           │
│    │                                          ╱              │
│    │                                       ╱                 │
│    │                                   ╱                     │
│    │                             ╱                           │
│    │                      ─────                              │
│    │             ─────────                                   │
│    │ ─────────────                                           │
│    └──────────────────────────────────────────────────────── │
│    0%              50%              80%   90%  95%  100%     │
│                         Utilization                          │
└─────────────────────────────────────────────────────────────┘

Rule of thumb: Keep utilization below 70-80% for stable latency
```

## Decision Framework

| Scenario | Optimize For | Technique |
|----------|-------------|-----------|
| User-facing API | Latency (p99) | Caching, async, edge |
| Batch processing | Throughput | Batching, parallelism |
| Real-time chat | Latency | WebSockets, edge servers |
| Data pipeline | Throughput | Buffering, streaming |
| Search | Latency + Throughput | Sharding, caching, ranking |

## Interview Talking Points

1. "Batching improves throughput but increases latency"
2. "Caching improves latency but requires memory and cache invalidation"
3. "Tail latency (p99) matters more than average for user experience"
4. "As utilization approaches 100%, latency spikes exponentially"
5. "Async processing reduces perceived latency but not actual completion time"

## Common Interview Questions

1. **Q: How would you reduce latency?**
   A: Caching, edge servers, async processing, reducing serial dependencies, pre-computation, efficient algorithms.

2. **Q: How would you increase throughput?**
   A: Batching, parallelism, horizontal scaling, async processing, efficient data structures, compression.

3. **Q: What's the difference between latency and response time?**
   A: Latency is network/processing delay. Response time = latency + time to transfer data. For large responses, they differ.

4. **Q: Why focus on p99 latency?**
   A: Average hides outliers. If 1% of users have 10x latency, that's millions of unhappy users at scale.

## Key Takeaways

- Latency and throughput often trade off against each other
- Batching: higher throughput, higher latency
- Caching: lower latency, memory cost
- Focus on tail latency (p99, p99.9) for user experience
- Keep utilization below 70-80% for stable latency
- Different operations need different optimizations

## Further Reading

- "High Performance Browser Networking" by Ilya Grigorik
- "Systems Performance" by Brendan Gregg
- Queueing theory fundamentals
- Tail latency papers from Google
