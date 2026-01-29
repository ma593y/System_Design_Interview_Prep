# Caching

## Overview

Caching stores frequently accessed data in fast storage (memory) to reduce latency and load on backend systems.

```
Request Flow with Cache:

┌────────┐    ┌─────────┐    ┌──────────┐    ┌──────────┐
│ Client │───▶│  Cache  │───▶│  Server  │───▶│ Database │
└────────┘    └─────────┘    └──────────┘    └──────────┘
                  │
            Cache Hit?
           ┌─────┴─────┐
          Yes          No
           │            │
     Return data    Query backend,
                    store in cache,
                    return data
```

## Why Cache?

| Benefit | Description |
|---------|-------------|
| Reduced Latency | Memory access: ~100ns vs Disk: ~10ms |
| Reduced Load | Fewer database queries |
| Cost Savings | Less compute for repeated operations |
| Improved UX | Faster page loads |

## Cache Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Browser Cache                         │
│              (CSS, JS, Images - local)                   │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                         CDN                              │
│            (Static content - edge servers)               │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                   Application Cache                      │
│          (API responses, computed data - Redis)          │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                    Database Cache                        │
│              (Query cache, buffer pool)                  │
└─────────────────────────────────────────────────────────┘
```

## Caching Strategies

### Cache-Aside (Lazy Loading)

Application manages cache. Most common pattern.

```
Read:
1. Check cache
2. If miss, read from DB
3. Store in cache
4. Return data

Write:
1. Write to DB
2. Invalidate cache
```

**Pros:**
- Simple to implement
- Only caches data that's requested
- Cache failure doesn't break the app

**Cons:**
- Cache miss penalty (3 trips)
- Data can become stale
- Cold start problem

### Read-Through

Cache sits between app and database.

```
Read:
1. Request data from cache
2. Cache fetches from DB if miss
3. Cache stores and returns data
```

**Pros:**
- Simpler application code
- Consistent read logic

**Cons:**
- Cache must understand data source
- Same staleness issues

### Write-Through

Data written to cache and database synchronously.

```
Write:
1. Write to cache
2. Cache writes to DB
3. Return success
```

**Pros:**
- Cache always has latest data
- No staleness

**Cons:**
- Write latency (2 writes)
- Cache churn for rarely-read data

### Write-Behind (Write-Back)

Data written to cache, async write to database.

```
Write:
1. Write to cache
2. Return success
3. Async: Cache writes to DB
```

**Pros:**
- Fast writes
- Batch database writes

**Cons:**
- Risk of data loss
- Complexity

### Refresh-Ahead

Proactively refresh cache before expiration.

**Pros:**
- Reduced cache miss latency
- Smoother performance

**Cons:**
- Complexity
- May refresh unused data

## Comparison Table

| Strategy | Read Performance | Write Performance | Consistency |
|----------|------------------|-------------------|-------------|
| Cache-Aside | Good (after warm) | Good | Eventual |
| Read-Through | Good | Good | Eventual |
| Write-Through | Excellent | Slower | Strong |
| Write-Behind | Excellent | Excellent | Eventual |

## Cache Eviction Policies

### LRU (Least Recently Used)

Evicts items not accessed for longest time.

```
Access: A, B, C, D, A, E (cache size: 4)

Cache state:
[A] → [A,B] → [A,B,C] → [A,B,C,D]
→ [B,C,D,A] → [C,D,A,E] (B evicted)
```

**Best for:** General purpose, temporal locality

### LFU (Least Frequently Used)

Evicts items with lowest access count.

**Best for:** When frequency matters more than recency

### FIFO (First In First Out)

Evicts oldest items first.

**Best for:** Simple implementations

### TTL (Time To Live)

Items expire after set duration.

```
cache.set("user:123", userData, TTL=3600)  // Expires in 1 hour
```

**Best for:** Data that becomes stale over time

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Strategies

1. **Time-based (TTL)**: Simple but may serve stale data
2. **Event-based**: Invalidate on write
3. **Version-based**: Include version in cache key

### Patterns

```
// Key with version
cache_key = "user:123:v2"

// Key with timestamp
cache_key = "user:123:1642000000"

// Pub/Sub invalidation
on_user_update(user_id):
    publish("cache:invalidate", f"user:{user_id}")
```

## Distributed Caching

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data Types | Strings, Lists, Sets, Hashes | Strings only |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Yes | No |
| Pub/Sub | Yes | No |
| Lua Scripting | Yes | No |
| Memory Efficiency | Lower | Higher |
| Multi-threading | Single (mostly) | Multi-threaded |

**Choose Redis when:** Need persistence, complex data types, pub/sub
**Choose Memcached when:** Simple caching, memory efficiency critical

### Cache Cluster Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│          Consistent Hashing             │
└─────────────────────────────────────────┘
       │
  ┌────┴────┬────────────┐
  ▼         ▼            ▼
┌─────┐  ┌─────┐     ┌─────┐
│Node1│  │Node2│     │Node3│
└─────┘  └─────┘     └─────┘
```

## Cache Problems

### Cache Stampede (Thundering Herd)

Many requests hit database when cache expires.

**Solutions:**
- Locking (only one request fetches)
- Probabilistic early expiration
- Background refresh

```
// Locking example
if cache_miss:
    if acquire_lock(key):
        data = fetch_from_db()
        cache.set(key, data)
        release_lock(key)
    else:
        wait_and_retry()
```

### Cache Penetration

Requests for non-existent data bypass cache.

**Solutions:**
- Cache null results (with short TTL)
- Bloom filter to check existence

### Cache Breakdown

Hot key expires, causing DB spike.

**Solutions:**
- Never expire hot keys
- Mutex lock during refresh
- Multiple cache copies

## CDN Caching

Content Delivery Network caches at edge locations.

```
User (Tokyo) → Edge (Tokyo) → Origin (US)
                   │
              Cache Hit!
              [Return immediately]
```

### What to Cache on CDN

- Static assets (CSS, JS, images)
- API responses (with care)
- HTML pages (for static sites)

### CDN Headers

```
Cache-Control: public, max-age=31536000
Cache-Control: private, no-cache
Cache-Control: no-store
```

## Interview Talking Points

1. "Cache-aside is the most common pattern for its simplicity"
2. "TTL is the simplest invalidation but may serve stale data"
3. "Redis for features, Memcached for pure speed"
4. "Always plan for cache failures - app should still work"
5. "CDN caching reduces latency for static content"

## Common Interview Questions

1. **Q: How do you handle cache invalidation?**
   A: Depends on consistency needs. TTL for simplicity, event-based invalidation for stronger consistency, or version-based keys for fine control.

2. **Q: What happens if the cache goes down?**
   A: With cache-aside, requests fall through to database. Need to ensure DB can handle the load. Consider circuit breakers.

3. **Q: How do you prevent cache stampede?**
   A: Use locking (only one request refreshes), probabilistic early expiration, or background refresh before expiry.

4. **Q: When would you NOT use caching?**
   A: When data changes frequently, when strong consistency is required, or when data is unique per request.

## Key Takeaways

- Caching dramatically improves performance
- Choose strategy based on read/write patterns
- Plan for invalidation from the start
- Cache at multiple layers for best results
- Always have a fallback when cache fails

## Further Reading

- Redis documentation
- "Caching at Scale" - various tech blogs
- CDN provider documentation (Cloudflare, Fastly)
