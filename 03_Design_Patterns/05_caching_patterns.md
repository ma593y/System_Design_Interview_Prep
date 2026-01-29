# Caching Patterns

## Overview

Caching patterns determine how data flows between cache and data source, each with different consistency and performance trade-offs.

## Cache-Aside (Lazy Loading)

Application manages cache directly.

```
Read:
┌─────────┐     ┌─────────┐     ┌─────────┐
│  App    │──1──│  Cache  │     │   DB    │
│         │◀─2──│ (miss)  │     │         │
│         │─────────────3───────│         │
│         │◀────────────4───────│         │
│         │──5──│ (store) │     │         │
└─────────┘     └─────────┘     └─────────┘

1. Check cache
2. Cache miss
3. Query database
4. Get data
5. Store in cache

Write:
┌─────────┐     ┌─────────┐     ┌─────────┐
│  App    │──1──│ Invalid │     │   DB    │
│         │─────────────2───────│         │
└─────────┘     └─────────┘     └─────────┘

1. Invalidate cache
2. Write to database
```

### Implementation

```python
def get_user(user_id):
    # 1. Check cache
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached

    # 2. Cache miss - query database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)

    # 3. Store in cache
    cache.set(f"user:{user_id}", user, ttl=3600)

    return user

def update_user(user_id, data):
    # 1. Invalidate cache
    cache.delete(f"user:{user_id}")

    # 2. Update database
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Only caches requested data | Cache miss penalty |
| Simple to implement | Stale data possible |
| Cache failure doesn't break app | N+1 problem (cold start) |

### Best For

- Read-heavy workloads
- Data that can tolerate some staleness
- When cache failures should be transparent

## Read-Through

Cache handles data loading.

```
┌─────────┐     ┌─────────────────────────────┐     ┌─────────┐
│  App    │──1──│         Cache               │     │   DB    │
│         │     │                             │     │         │
│         │     │  Cache miss?                │     │         │
│         │     │  ┌─────────────────────┐    │     │         │
│         │     │  │ Load from database  │────┼──2──│         │
│         │     │  │ Store in cache      │◀───┼──3──│         │
│         │     │  └─────────────────────┘    │     │         │
│         │◀─4──│                             │     │         │
└─────────┘     └─────────────────────────────┘     └─────────┘

Cache transparently loads data on miss
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Simpler application code | Cache must understand data source |
| Consistent loading logic | Less flexibility |
| Prevents cache stampede | Tied to specific cache implementation |

### Best For

- When you want consistent caching logic
- Reducing application complexity
- Using cache products with built-in read-through

## Write-Through

Write to cache and database together.

```
┌─────────┐     ┌─────────────────────────────┐     ┌─────────┐
│  App    │──1──│         Cache               │     │   DB    │
│         │     │                             │     │         │
│         │     │  Write data                 │     │         │
│         │     │  ┌─────────────────────┐    │     │         │
│         │     │  │ Store in cache      │    │     │         │
│         │     │  │ Write to database   │────┼──2──│         │
│         │     │  └─────────────────────┘◀───┼──3──│         │
│         │◀─4──│                             │     │         │
└─────────┘     └─────────────────────────────┘     └─────────┘

1. App writes to cache
2. Cache writes to database
3. Database confirms
4. Cache confirms to app
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Cache always has latest data | Higher write latency |
| No stale reads | All data cached (even if never read) |
| Simpler consistency | Database must support |

### Best For

- Strong consistency requirements
- Read-heavy after initial write
- Tolerating write latency

## Write-Behind (Write-Back)

Write to cache, async write to database.

```
┌─────────┐     ┌─────────────────────────────┐     ┌─────────┐
│  App    │──1──│         Cache               │     │   DB    │
│         │     │                             │     │         │
│         │◀─2──│  Store in cache             │     │         │
│         │     │                             │     │         │
└─────────┘     │  Later (async):             │     │         │
                │  ┌─────────────────────┐    │     │         │
                │  │ Batch writes to DB  │────┼──3──│         │
                │  └─────────────────────┘    │     │         │
                └─────────────────────────────┘     └─────────┘

1. App writes to cache
2. Immediate response
3. Async batch write to database
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Very fast writes | Risk of data loss on crash |
| Reduced database load | Complex to implement |
| Can batch/coalesce writes | Eventual consistency |

### Best For

- Write-heavy workloads
- Acceptable data loss window
- Need to reduce database load

## Write-Around

Write directly to database, bypass cache.

```
Write:
┌─────────┐     ┌─────────┐     ┌─────────┐
│  App    │─────────────1───────│   DB    │
└─────────┘     │ (skip)  │     └─────────┘
                └─────────┘

Read (cache-aside):
┌─────────┐     ┌─────────┐     ┌─────────┐
│  App    │──1──│  Cache  │     │   DB    │
│         │◀─2──│ (miss)  │     │         │
│         │─────────────3───────│         │
│         │◀────────────4───────│         │
│         │──5──│ (store) │     │         │
└─────────┘     └─────────┘     └─────────┘
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Doesn't cache rarely-read data | Higher read latency initially |
| Reduces cache churn | Miss penalty on first read |

### Best For

- Write-heavy with infrequent reads
- Data written once, read rarely
- Large objects written frequently

## Refresh-Ahead

Proactively refresh cache before expiration.

```
Timeline:
──────────────────────────────────────────────────────▶ Time

Cache entry: [──────────────────────|──────]
             ▲                      ▲      ▲
           Created             Refresh   Expire
                             Threshold

When entry accessed after refresh threshold:
- Return cached value immediately
- Async refresh from database

Avoids cache miss latency for hot data
```

### Implementation

```python
def get_with_refresh(key, ttl=3600, refresh_at=0.75):
    entry = cache.get_with_metadata(key)

    if not entry:
        # Cache miss - load and cache
        value = load_from_db(key)
        cache.set(key, value, ttl=ttl)
        return value

    # Check if needs refresh
    age_ratio = entry.age / ttl
    if age_ratio > refresh_at:
        # Async refresh
        async_refresh(key)

    return entry.value
```

### Best For

- Hot data with predictable access
- Low tolerance for cache miss latency
- When background refresh is acceptable

## Pattern Comparison

| Pattern | Read Latency | Write Latency | Consistency | Complexity |
|---------|--------------|---------------|-------------|------------|
| Cache-Aside | Higher (miss) | Low | Eventual | Low |
| Read-Through | Higher (miss) | Low | Eventual | Medium |
| Write-Through | Low | Higher | Strong | Medium |
| Write-Behind | Low | Lowest | Eventual | High |
| Write-Around | Higher (miss) | Low | Eventual | Low |
| Refresh-Ahead | Lowest (hot) | Low | Eventual | High |

## Cache Invalidation Strategies

### Time-Based (TTL)

```
cache.set("user:123", data, ttl=3600)  # Expires in 1 hour

Pros: Simple
Cons: May serve stale data, fixed expiration
```

### Event-Based

```
On user update:
  cache.delete("user:123")

Or publish invalidation event:
  publish("cache:invalidate", "user:123")

Pros: Immediate consistency
Cons: More complex, need event infrastructure
```

### Version-Based

```
cache.set("user:123:v2", data)

On update:
  increment version
  cache.set("user:123:v3", data)

Pros: No explicit invalidation
Cons: Old versions linger, key management
```

## Common Problems

### Cache Stampede

Many requests hit database when cache expires.

```
Solution 1: Locking
if cache_miss:
    if acquire_lock(key):
        data = load_from_db()
        cache.set(key, data)
        release_lock(key)
    else:
        wait_and_retry()

Solution 2: Probabilistic early expiration
if random() < probability_based_on_age:
    async_refresh()
```

### Cache Penetration

Requests for non-existent data bypass cache.

```
Solution: Cache null results
data = db.get(key)
if data is None:
    cache.set(key, NULL_MARKER, ttl=60)  # Short TTL
else:
    cache.set(key, data, ttl=3600)
```

### Hot Key

Single key receives excessive traffic.

```
Solution: Replicate hot keys
for i in range(replicas):
    cache.set(f"{key}:{i}", data)

Read from random replica:
replica = random.randint(0, replicas-1)
cache.get(f"{key}:{replica}")
```

## Interview Talking Points

1. "Cache-aside is most common - simple and flexible"
2. "Write-through for consistency, write-behind for performance"
3. "TTL handles eventual invalidation"
4. "Watch for cache stampede on popular keys"
5. "Match pattern to read/write ratio"

## Common Interview Questions

1. **Q: What caching strategy would you use for a read-heavy application?**
   A: Cache-aside with appropriate TTL. Simple, only caches what's needed, handles cache failures gracefully.

2. **Q: How do you ensure cache consistency with database?**
   A: Depends on requirements. Write-through for strong consistency. Cache-aside with event-based invalidation for near real-time. TTL for eventual consistency.

3. **Q: How do you handle cache stampede?**
   A: Locking (only one request loads), probabilistic early refresh, or pre-warming before expiration.

4. **Q: When would you use write-behind?**
   A: High write volume, can tolerate data loss window, want to reduce database load through batching.

## Key Takeaways

- Choose pattern based on read/write ratio
- Cache-aside is the safe default
- Write patterns trade latency for consistency
- Plan for cache failures
- Monitor hit rates and adjust

## Further Reading

- "Scaling Memcache at Facebook" paper
- Redis caching patterns
- AWS ElastiCache best practices
