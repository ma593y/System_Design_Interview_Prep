# Performance Optimization

## Table of Contents
1. [Latency Reduction Techniques](#1-latency-reduction-techniques)
2. [Throughput Improvement](#2-throughput-improvement)
3. [Database Optimization](#3-database-optimization)
4. [Network Optimization](#4-network-optimization)
5. [Application Optimization](#5-application-optimization)
6. [Frontend Optimization](#6-frontend-optimization)
7. [Caching at Every Layer](#7-caching-at-every-layer)
8. [Load Testing Methodology](#8-load-testing-methodology)
9. [Performance Monitoring and Bottleneck Identification](#9-performance-monitoring-and-bottleneck-identification)
10. [Common Performance Anti-Patterns](#10-common-performance-anti-patterns)

---

## 1. Latency Reduction Techniques

### Latency Overview

```
  User Action ──> Network ──> Load Balancer ──> App Server ──> Database ──> Response
      |                                                                        |
      +──────────────────── Total Latency ────────────────────────────────────+

  Typical Latency Budget (200ms target):
  +───────────────+──────+───────────────────────────────────────+
  | Component     | Time | Visualization                         |
  +───────────────+──────+───────────────────────────────────────+
  | DNS Lookup    | 10ms | ██                                    |
  | TCP Handshake | 15ms | ███                                   |
  | TLS Handshake | 25ms | █████                                 |
  | Network RTT   | 40ms | ████████                              |
  | App Logic     | 30ms | ██████                                |
  | DB Query      | 50ms | ██████████                            |
  | Serialization | 10ms | ██                                    |
  | Response Xfer | 20ms | ████                                  |
  +───────────────+──────+───────────────────────────────────────+
  | Total         |200ms |                                       |
  +───────────────+──────+───────────────────────────────────────+
```

### Key Latency Numbers Every Engineer Should Know

| Operation                           | Latency        |
|------------------------------------|----------------|
| L1 cache reference                  | 0.5 ns         |
| L2 cache reference                  | 7 ns           |
| Main memory reference               | 100 ns         |
| SSD random read                     | 150 us         |
| HDD random read                     | 10 ms          |
| Network round trip (same DC)        | 0.5 ms         |
| Network round trip (cross-continent)| 150 ms         |
| Read 1 MB from memory               | 250 us         |
| Read 1 MB from SSD                  | 1 ms           |
| Read 1 MB from HDD                  | 20 ms          |
| Read 1 MB from network (1 Gbps)     | 10 ms          |

### Caching

The most impactful latency reduction. Move data closer to the consumer.

```
  Without Cache:
  Client ──> App ──> DB (50ms) ──> App ──> Client
  Total: ~100ms

  With Cache:
  Client ──> App ──> Cache (1ms) ──> App ──> Client
  Total: ~20ms (5x improvement)
```

### CDN (Content Delivery Network)

```
  Without CDN:                    With CDN:

  User (Tokyo)                    User (Tokyo)
    |                               |
    | 150ms RTT                     | 5ms RTT
    v                               v
  Origin (US-East)                CDN Edge (Tokyo)
                                    |
                                    | Cache miss? Fetch from origin
                                    v
                                  Origin (US-East)
```

### Connection Pooling

```
  Without Pooling:                With Pooling:

  Request 1: Open ─ Query ─ Close   Request 1: Acquire ─ Query ─ Return
  Request 2: Open ─ Query ─ Close   Request 2: Acquire ─ Query ─ Return
  Request 3: Open ─ Query ─ Close   Request 3: Acquire ─ Query ─ Return

  Each open: ~20ms                  Each acquire: ~0.1ms
  3 requests: 60ms overhead         3 requests: 0.3ms overhead
```

### Precomputation

Compute results ahead of time and store them for instant retrieval.

| Use Case                   | Precomputation Strategy              |
|---------------------------|--------------------------------------|
| News feed                  | Fan-out on write, precompute feeds   |
| Leaderboards               | Materialized view, periodic rebuild  |
| Search suggestions         | Precomputed trie / prefix index      |
| Product recommendations    | Offline ML pipeline, store results   |
| Dashboard metrics          | Aggregate on write, store counters   |

---

## 2. Throughput Improvement

### Horizontal Scaling

```
  Before (1 server, 1000 RPS):

  [All Traffic] ──> [Server] ──> 1000 RPS max

  After (4 servers, 4000 RPS):

  [All Traffic] ──> [Load Balancer]
                       |   |   |   |
                      [S1] [S2] [S3] [S4]  ──> 4000 RPS max
```

### Asynchronous Processing

Move non-critical work off the request path.

```
  Synchronous (slow):
  Client ──> [Create Order] ──> [Send Email] ──> [Update Analytics] ──> Response
  Total: 500ms

  Asynchronous (fast):
  Client ──> [Create Order] ──> Response (100ms)
                  |
                  +──> Queue ──> [Send Email]       (background)
                  +──> Queue ──> [Update Analytics]  (background)
```

### Batching

Combine multiple small operations into fewer large operations.

```
  Without Batching:                With Batching:
  INSERT INTO orders (...)         INSERT INTO orders (...)
  INSERT INTO orders (...)         VALUES
  INSERT INTO orders (...)           (...), (...), (...),
  INSERT INTO orders (...)           (...), (...), (...),
  INSERT INTO orders (...)           (...), (...), (...),
  INSERT INTO orders (...)           (...), (...), (...);
  ... (100 round trips)            (1 round trip)

  100 x 5ms = 500ms               1 x 15ms = 15ms
```

### Parallelization

```
  Sequential:
  [Fetch User] ──> [Fetch Orders] ──> [Fetch Reviews] ──> [Render]
      50ms              80ms               40ms             30ms
  Total: 200ms

  Parallel:
  [Fetch User]    ──┐
      50ms          |
  [Fetch Orders]  ──┼──> [Render]
      80ms          |      30ms
  [Fetch Reviews] ──┘
      40ms
  Total: 80ms + 30ms = 110ms (45% faster)
```

---

## 3. Database Optimization

### Indexing Strategy

```
  Without Index (Full Table Scan):
  +────+──────+──────────+
  | id | name | email    |  Scan all 1M rows
  +────+──────+──────────+  Time: 500ms
  | 1  | ...  | ...      |
  | 2  | ...  | ...      |
  | .. | ...  | ...      |
  | 1M | ...  | ...      |
  +────+──────+──────────+

  With B-Tree Index on email:

       [M]                  Traverse 3-4 levels
      /   \                 Time: 5ms
    [G]   [T]
   / \    / \
  [A] [K][P] [Z]

  Index Scan: O(log n) vs Full Scan: O(n)
```

### Index Types and When to Use Them

| Index Type        | Best For                              | Example                    |
|------------------|---------------------------------------|----------------------------|
| B-Tree (default) | Equality, range, sorting              | WHERE age > 25             |
| Hash             | Exact equality lookups                | WHERE id = 123             |
| Composite        | Multi-column queries                  | WHERE city='NY' AND age>25 |
| Covering         | Queries answered entirely from index  | SELECT name WHERE id=123   |
| Partial          | Queries on a subset of rows           | WHERE status = 'active'    |
| Full-text        | Text search                           | WHERE body MATCH 'system'  |

### Query Optimization Techniques

```
  SLOW: SELECT * FROM orders WHERE YEAR(created_at) = 2024
        (function on column prevents index use)

  FAST: SELECT * FROM orders
        WHERE created_at >= '2024-01-01'
        AND created_at < '2025-01-01'
        (range scan uses index)

  SLOW: SELECT * FROM orders WHERE status != 'cancelled'
        (negative condition scans most rows)

  FAST: SELECT * FROM orders WHERE status IN ('pending','shipped','delivered')
        (positive list, can use index)

  SLOW: SELECT * FROM users u
        JOIN orders o ON u.id = o.user_id
        JOIN items i ON o.id = i.order_id
        (no LIMIT, returns millions of rows)

  FAST: SELECT u.name, o.total FROM users u
        JOIN orders o ON u.id = o.user_id
        WHERE u.id = 123
        ORDER BY o.created_at DESC
        LIMIT 20
        (specific columns, filtered, limited)
```

### Read Replicas

```
  +──────────────+
  |   App Layer  |
  +──┬───────┬───+
     |       |
  Writes   Reads
     |       |
     v       v
  +──────+ +──────+ +──────+
  |Primary| |Read  | |Read  |
  |  DB   |>|Rep 1 | |Rep 2 |
  +──────+ +──────+ +──────+
           async replication
```

Typical read/write split: 80-90% reads, 10-20% writes. Read replicas can handle the majority
of query load.

### Database Connection Pooling

```
  Without Pooling:
  Each request creates new connection
  Max connections hit quickly
  Connection overhead: ~20ms each

  With Pooling (e.g., PgBouncer):
  +──────────────+     +──────────+     +──────────+
  | App Instance |────>| PgBouncer|────>| Postgres |
  | (100 conns)  |     | Pool: 20 |     | Max: 100 |
  +──────────────+     +──────────+     +──────────+
  | App Instance |────>|          |
  | (100 conns)  |     |          |
  +──────────────+     +──────────+

  200 app connections -> 20 DB connections
  Multiplexing reduces DB load dramatically
```

---

## 4. Network Optimization

### Compression

```
  Without Compression:           With Compression (gzip/brotli):

  Response: 500 KB               Response: 80 KB (84% smaller)
  Transfer: 50ms @ 10MB/s        Transfer: 8ms @ 10MB/s

  Content-Encoding: gzip

  Best candidates for compression:
  - JSON/XML responses (60-90% reduction)
  - HTML/CSS/JS (70-90% reduction)
  - NOT already compressed: images, video, zip files
```

### HTTP/2 Advantages

```
  HTTP/1.1:                      HTTP/2:

  [Request 1]────>               [Req 1]──┐
  <────[Response 1]              [Req 2]──┤ Multiplexed
  [Request 2]────>               [Req 3]──┤ on single
  <────[Response 2]              <──[Res 1]┤ connection
  [Request 3]────>               <──[Res 3]┤
  <────[Response 3]              <──[Res 2]┘

  Sequential (HOL blocking)      Parallel (no HOL blocking)
  6 connections per domain       1 connection, many streams
  No header compression          HPACK header compression
  No server push                 Server push supported
```

### Keep-Alive Connections

```
  Without Keep-Alive:
  Request 1: [TCP] [TLS] [HTTP] ──> close
  Request 2: [TCP] [TLS] [HTTP] ──> close   (repeat handshakes)

  With Keep-Alive:
  Request 1: [TCP] [TLS] [HTTP]
  Request 2:              [HTTP]   (reuse connection)
  Request 3:              [HTTP]   (reuse connection)

  Savings: ~35ms per subsequent request (TCP + TLS handshake)
```

### Edge Computing

```
  Traditional:
  User ──(150ms)──> Origin Server ──(50ms)──> DB

  Edge Computing:
  User ──(5ms)──> Edge Server ──(cached or computed locally)
                       |
                       +──(150ms)──> Origin (cache miss only)
```

Use cases: Authentication, A/B test assignment, geo-routing, personalization, request
validation.

---

## 5. Application Optimization

### Profiling and Hotspot Identification

```
  CPU Profile (Flame Graph concept):

  +─────────────────────────────────────────────+
  | main()                                      |
  +──────────────────────┬──────────────────────+
  | handleRequest()      | backgroundTask()     |
  +──────────┬───────────+──────────────────────+
  | parseJSON| queryDB() |
  +──────────+─────┬─────+
             | serialize |
             +───────────+

  Width = time spent
  Identify: queryDB() is the hotspot (widest bar)
```

### Memory Management

| Problem                | Solution                                    |
|-----------------------|---------------------------------------------|
| Memory leaks          | Use weak references, close resources         |
| Large object allocation| Object pooling, buffer reuse               |
| Excessive GC pauses   | Tune heap size, use G1/ZGC collector         |
| High memory usage     | Streaming instead of loading all into memory |
| Cache unbounded growth | Set max size, use LRU eviction              |

### Garbage Collection Tuning

```
  GC Pause Impact on Latency:

  Request Timeline:
  |──────────|──GC──|────────|──GC──|──────────|

  p50 latency: 20ms (no GC during request)
  p99 latency: 200ms (GC pause during request)

  Strategies:
  - Use low-pause collectors (G1GC, ZGC, Shenandoah)
  - Reduce allocation rate (reuse objects)
  - Right-size heap (too small = frequent GC, too large = long pauses)
  - Off-heap memory for large datasets (direct ByteBuffer)
```

### Thread Pool Sizing

```
  CPU-bound tasks:
    Threads = Number of CPU cores
    Example: 8 cores -> 8 threads

  I/O-bound tasks:
    Threads = Number of CPU cores x (1 + Wait Time / Service Time)
    Example: 8 cores, wait=50ms, service=5ms
    Threads = 8 x (1 + 50/5) = 88 threads

  Mixed workloads:
    Separate pools for CPU-bound and I/O-bound work
```

---

## 6. Frontend Optimization

### Lazy Loading

```
  Eager Loading (traditional):
  Page Load ──> Download ALL images ──> Render
  Initial payload: 5 MB, Load time: 4 seconds

  Lazy Loading:
  Page Load ──> Download visible images ──> Render
  As user scrolls ──> Download next images
  Initial payload: 800 KB, Load time: 1.2 seconds
```

### Code Splitting

```
  Without Splitting:
  bundle.js = 2 MB (entire application)
  Download: 2 seconds on 3G

  With Splitting:
  main.js       = 200 KB  (core, loaded immediately)
  dashboard.js  = 300 KB  (loaded when visiting /dashboard)
  settings.js   = 150 KB  (loaded when visiting /settings)
  admin.js      = 500 KB  (loaded only for admin users)

  Initial load: 200 KB = 0.2 seconds
```

### Image Optimization

| Technique               | Impact                                   |
|------------------------|------------------------------------------|
| WebP/AVIF format       | 30-50% smaller than JPEG                  |
| Responsive images      | Serve right size for device               |
| Lazy loading           | Only load visible images                  |
| CDN with transforms    | Resize/compress at edge                   |
| Placeholder (LQIP)     | Show blur while loading                   |
| Sprite sheets          | Combine small icons into one request      |

### Critical Rendering Path

```
  Browser rendering pipeline:

  HTML ──> DOM Tree ──┐
                      +──> Render Tree ──> Layout ──> Paint ──> Composite
  CSS  ──> CSSOM ────┘

  Optimization:
  1. Inline critical CSS (above-the-fold styles)
  2. Defer non-critical CSS
  3. Async/defer JavaScript
  4. Preload key resources: <link rel="preload">
  5. Preconnect to origins: <link rel="preconnect">
```

---

## 7. Caching at Every Layer

```
  Request Flow with Caching:

  [Browser Cache] ──miss──> [CDN Cache] ──miss──> [API Gateway Cache]
       |                        |                        |
      hit                     hit                      miss
       |                        |                        |
    (instant)               (~5ms)                       v
                                                [App Server Cache]
                                                       |
                                                      miss
                                                       v
                                                [Database Cache]
                                                 (query cache)
                                                       |
                                                      miss
                                                       v
                                                   [Disk I/O]
```

### Cache Layer Details

| Layer              | Technology          | TTL        | Hit Rate | Latency  |
|-------------------|---------------------|------------|----------|----------|
| Browser cache     | HTTP Cache-Control   | Minutes-days| 60-80% | 0ms      |
| CDN cache         | CloudFront, Fastly   | Minutes-hours| 80-95% | 1-5ms    |
| API Gateway       | Kong, NGINX          | Seconds    | 50-70%   | 1-2ms    |
| Application cache | Redis, Memcached     | Seconds-min| 85-99%  | 1-5ms    |
| Database cache    | Query cache, buffer  | Auto       | 70-90%   | <1ms     |

### Cache Invalidation Strategies

```
  TTL-based:
  SET key value EX 300     (expire after 5 minutes)
  Simple but data can be stale for up to TTL

  Event-driven:
  UPDATE products SET ...  ──> DELETE cache:product:123
  Precise but requires pub/sub infrastructure

  Write-through:
  Write ──> Cache ──> DB    (both updated simultaneously)
  Consistent but slower writes

  Write-behind:
  Write ──> Cache ──> (async) ──> DB
  Fast writes but risk of data loss

  Cache-aside (Lazy loading):
  Read: Check cache -> miss -> read DB -> populate cache
  Write: Update DB -> invalidate cache
  Most common pattern, simple to implement
```

---

## 8. Load Testing Methodology

### Load Testing Types

```
  Response
  Time
    |
    |              Stress Test
    |             /
    |            / Spike Test
    |           / /
    |     ____/ /
    |    /     |
    |   / Load |  Soak Test (endurance)
    |  /  Test |  ─────────────────────────
    | /        |
    |/_________|_________________________________
    0          Target      Breaking     ──> Load
               Load        Point
```

### Testing Process

| Phase       | Goal                                    | Duration      |
|------------|----------------------------------------|---------------|
| Baseline   | Establish current performance            | 10-15 min     |
| Load test  | Verify system handles expected load      | 30-60 min     |
| Stress test| Find the breaking point                  | Until failure |
| Soak test  | Check for memory leaks, degradation      | 4-24 hours    |
| Spike test | Verify handling of sudden traffic bursts  | 15-30 min     |

### Key Metrics to Capture

```
  During Load Test, monitor:

  +─────────────────────────────────────────+
  | Response Time Distribution              |
  |                                         |
  | p50:  45ms   ████████                   |
  | p90:  120ms  █████████████████          |
  | p95:  250ms  █████████████████████████  |
  | p99:  800ms  ████████████████████████████████|
  | p999: 2.5s   ██████████████████████████████████████|
  +─────────────────────────────────────────+

  Throughput:  2,500 RPS at target load
  Error Rate:  0.01% (target < 0.1%)
  CPU Usage:   65% (target < 80%)
  Memory:      72% (target < 85%)
  DB Conns:    45/100 (target < 80%)
```

### Tool Comparison

| Tool     | Language   | Protocol       | Distributed | Scripting      |
|----------|-----------|----------------|-------------|----------------|
| k6       | JavaScript | HTTP, WS, gRPC | Yes         | JS scripts     |
| JMeter   | Java      | HTTP, FTP, JDBC | Yes        | GUI + XML      |
| Locust   | Python    | HTTP, custom    | Yes         | Python scripts |
| Gatling  | Scala     | HTTP, WS        | Yes         | Scala DSL      |
| wrk      | C/Lua     | HTTP            | No          | Lua scripts    |

---

## 9. Performance Monitoring and Bottleneck Identification

### The USE Method (Brendan Gregg)

For every resource, check: **Utilization, Saturation, Errors**

```
  +──────────────+──────────────+───────────────+──────────────+
  | Resource     | Utilization  | Saturation    | Errors       |
  +──────────────+──────────────+───────────────+──────────────+
  | CPU          | % busy       | Run queue len | HW errors    |
  | Memory       | % used       | Swapping      | OOM kills    |
  | Disk I/O     | % busy       | Wait queue    | Device errors|
  | Network      | Bandwidth %  | Dropped pkts  | Errors/s     |
  | DB Conns     | Active/Max   | Wait queue    | Timeouts     |
  +──────────────+──────────────+───────────────+──────────────+
```

### The RED Method (Tom Wilkie)

For every service, measure: **Rate, Errors, Duration**

```
  +──────────────+──────────────────────────────────────────+
  | Metric       | Description                              |
  +──────────────+──────────────────────────────────────────+
  | Rate         | Requests per second                      |
  | Errors       | Failed requests per second               |
  | Duration     | Distribution of request latency          |
  +──────────────+──────────────────────────────────────────+
```

### Bottleneck Identification Flow

```
  High Latency Detected
         |
         v
  Is CPU > 80%? ──Yes──> Profile application code
         |                Find hot methods
        No
         |
         v
  Is Memory > 85%? ──Yes──> Check for leaks
         |                   Analyze heap dump
        No
         |
         v
  Is DB slow? ──Yes──> Check slow query log
         |              Analyze execution plans
        No              Check connection pool
         |
         v
  Is Network saturated? ──Yes──> Check bandwidth
         |                        Enable compression
        No                        Review payload sizes
         |
         v
  Is there lock contention? ──Yes──> Review synchronization
         |                            Optimize critical sections
        No
         |
         v
  Check external dependencies (APIs, third-party services)
```

---

## 10. Common Performance Anti-Patterns

### The N+1 Query Problem

```
  BAD: Fetch 100 users, then 100 separate queries for their orders

  SELECT * FROM users;                      -- 1 query
  SELECT * FROM orders WHERE user_id = 1;   -- +1
  SELECT * FROM orders WHERE user_id = 2;   -- +1
  ...                                       -- +98
  Total: 101 queries, ~500ms

  GOOD: Fetch users and orders in 2 queries with JOIN or IN clause

  SELECT * FROM users;
  SELECT * FROM orders WHERE user_id IN (1,2,3,...,100);
  Total: 2 queries, ~10ms
```

### Chatty APIs

```
  BAD: Client makes 10 API calls to render a page

  GET /user/123           (20ms)
  GET /user/123/orders    (30ms)
  GET /user/123/address   (15ms)
  GET /user/123/prefs     (10ms)
  ...
  Total: 150ms + 10 round trips

  GOOD: Aggregate API or GraphQL

  POST /graphql
  { user(id:123) { name, orders, address, prefs } }
  Total: 50ms + 1 round trip
```

### Premature Optimization

```
  Anti-pattern: Spending weeks optimizing a function that takes 2ms
  when the database query takes 500ms.

  Always profile first. Optimize the actual bottleneck.

  Amdahl's Law:
  If a component is 5% of total time, even making it infinitely fast
  only improves total performance by 5%.
```

### Other Anti-Patterns

| Anti-Pattern                    | Impact                       | Solution                     |
|--------------------------------|------------------------------|------------------------------|
| Unbounded queries              | Memory exhaustion, timeouts  | Always paginate              |
| Synchronous everything         | Thread starvation            | Async for I/O operations     |
| No connection pooling          | Connection exhaustion        | Use connection pools         |
| Logging in hot path            | I/O overhead per request     | Async logging, sampling      |
| String concatenation in loops  | GC pressure, memory waste    | StringBuilder/buffer         |
| Polling instead of pushing     | Wasted resources             | WebSockets, SSE, webhooks    |
| Cache without eviction policy  | Memory growth until OOM      | Set max size + LRU eviction  |
| Single-threaded processing     | Underutilized hardware       | Worker pools, parallelism    |
| Over-serialization             | CPU overhead                 | Binary formats (protobuf)    |
| Missing database indexes       | Full table scans             | Index frequently queried cols|
