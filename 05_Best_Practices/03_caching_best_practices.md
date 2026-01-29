# Caching Best Practices

## Overview

Caching is one of the most effective techniques for improving system performance and scalability. By storing frequently accessed data closer to the application, caching reduces latency, decreases database load, and improves overall user experience. This guide covers TTL strategies, invalidation patterns, cache warming, and other crucial caching best practices.

## Table of Contents

1. [Cache Fundamentals](#cache-fundamentals)
2. [TTL Strategies](#ttl-strategies)
3. [Cache Invalidation Patterns](#cache-invalidation-patterns)
4. [Cache Warming](#cache-warming)
5. [Caching Patterns](#caching-patterns)
6. [Distributed Caching](#distributed-caching)
7. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
8. [Implementation Guidelines](#implementation-guidelines)
9. [Interview Talking Points](#interview-talking-points)
10. [Key Takeaways](#key-takeaways)

---

## Cache Fundamentals

### Why Caching Matters

```
Without Cache:
User Request -> Application -> Database -> Response
Latency: 50-500ms

With Cache:
User Request -> Application -> Cache (Hit) -> Response
Latency: 1-10ms

Cache Miss:
User Request -> Application -> Cache (Miss) -> Database -> Cache Update -> Response
```

### Cache Hit Ratio

```
Cache Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)

Target Ratios:
- >95%: Excellent
- 90-95%: Good
- 80-90%: Acceptable
- <80%: Needs optimization
```

### Types of Caches

```
Application Layer:
- In-memory (local): HashMap, Guava Cache
- Distributed: Redis, Memcached

CDN Layer:
- Static assets: CloudFront, Cloudflare
- Dynamic content: Edge caching

Browser Layer:
- HTTP caching: Cache-Control headers
- Service Workers: Offline-first caching

Database Layer:
- Query cache: MySQL query cache
- Buffer pool: PostgreSQL shared buffers
```

---

## TTL Strategies

### Understanding TTL (Time-To-Live)

TTL defines how long cached data remains valid before automatic expiration.

```python
import redis
from datetime import timedelta

redis_client = redis.Redis()

# Fixed TTL
redis_client.setex('user:123', timedelta(hours=1), user_data)

# TTL with sliding expiration
def get_with_sliding_ttl(key: str, ttl: int = 3600):
    """Reset TTL on each access."""
    value = redis_client.get(key)
    if value:
        redis_client.expire(key, ttl)  # Reset TTL
    return value
```

### TTL Strategy Patterns

#### 1. Static TTL

```python
# Same TTL for all entries of a type
CACHE_TTLS = {
    'user_profile': 3600,      # 1 hour
    'product_catalog': 86400,  # 24 hours
    'session': 1800,           # 30 minutes
    'config': 300,             # 5 minutes
}

def cache_with_type_ttl(cache_type: str, key: str, value: Any):
    ttl = CACHE_TTLS.get(cache_type, 3600)
    redis_client.setex(f"{cache_type}:{key}", ttl, json.dumps(value))
```

#### 2. Dynamic TTL Based on Data Characteristics

```python
def calculate_dynamic_ttl(data: dict) -> int:
    """Calculate TTL based on data volatility."""

    # Recently updated data expires faster
    last_updated = data.get('updated_at')
    if last_updated:
        hours_since_update = (datetime.now() - last_updated).total_seconds() / 3600
        if hours_since_update < 1:
            return 300   # 5 minutes for recently changed
        elif hours_since_update < 24:
            return 1800  # 30 minutes

    # High-traffic items cached longer
    if data.get('view_count', 0) > 10000:
        return 7200  # 2 hours

    return 3600  # Default 1 hour
```

#### 3. Probabilistic Early Expiration

```python
import random
import math

def get_with_probabilistic_refresh(key: str, beta: float = 1.0):
    """
    Probabilistically refresh cache before expiration to prevent
    stampede when TTL expires.

    XFetch algorithm: recompute with probability that increases
    as expiration approaches.
    """
    cached = redis_client.get(key)
    if not cached:
        return None

    data = json.loads(cached)
    ttl = redis_client.ttl(key)
    expiry = data.get('cached_at', 0) + data.get('original_ttl', 3600)

    # Calculate probability of early refresh
    now = time.time()
    delta = expiry - now

    # Higher probability as we approach expiration
    if delta > 0:
        probability = math.exp(-delta / (beta * data.get('compute_time', 1)))
        if random.random() < probability:
            return None  # Trigger recomputation

    return data.get('value')
```

#### 4. Adaptive TTL

```python
class AdaptiveTTLCache:
    """Automatically adjust TTL based on access patterns."""

    def __init__(self, redis_client, min_ttl=60, max_ttl=86400):
        self.redis = redis_client
        self.min_ttl = min_ttl
        self.max_ttl = max_ttl

    def get(self, key: str):
        value = self.redis.get(f"cache:{key}")
        if value:
            # Track access frequency
            self.redis.incr(f"hits:{key}")
            self.redis.expire(f"hits:{key}", 3600)
        return value

    def set(self, key: str, value: Any):
        # Calculate TTL based on historical access pattern
        hits = int(self.redis.get(f"hits:{key}") or 0)

        # More hits = longer TTL
        ttl = min(self.max_ttl, max(self.min_ttl, hits * 60))

        self.redis.setex(f"cache:{key}", ttl, json.dumps(value))
        return ttl
```

### TTL Guidelines by Data Type

```
Data Type                    Recommended TTL    Reasoning
---------------------------------------------------------------------------
User session                 15-30 minutes      Security, activity-based
User profile                 1-24 hours         Infrequent changes
Product catalog              1-24 hours         Scheduled updates
Inventory/Stock              1-5 minutes        Frequently changing
Search results               5-15 minutes       Balance freshness/load
Configuration                5-15 minutes       Propagation time acceptable
Static content               24+ hours          Rarely changes
Real-time data               No cache / seconds Freshness critical
```

---

## Cache Invalidation Patterns

### The Challenge

```
"There are only two hard things in Computer Science:
cache invalidation and naming things." - Phil Karlton
```

### Invalidation Strategies

#### 1. Time-Based Expiration (TTL)

```python
# Simplest approach - let cache expire naturally
redis_client.setex('product:123', 3600, product_data)

# Pros: Simple, automatic cleanup
# Cons: Stale data until expiration
```

#### 2. Event-Based Invalidation

```python
from dataclasses import dataclass
from typing import List
import json

@dataclass
class CacheInvalidationEvent:
    entity_type: str
    entity_id: str
    action: str  # 'update', 'delete'

class EventDrivenCache:
    def __init__(self, redis_client, pubsub_client):
        self.redis = redis_client
        self.pubsub = pubsub_client

    def invalidate(self, event: CacheInvalidationEvent):
        """Invalidate cache and publish event for distributed invalidation."""
        # Local invalidation
        cache_key = f"{event.entity_type}:{event.entity_id}"
        self.redis.delete(cache_key)

        # Publish for other nodes
        self.pubsub.publish('cache_invalidation', json.dumps({
            'entity_type': event.entity_type,
            'entity_id': event.entity_id,
            'action': event.action,
            'timestamp': time.time()
        }))

    def handle_invalidation_event(self, message):
        """Handle invalidation events from other nodes."""
        event = json.loads(message)
        cache_key = f"{event['entity_type']}:{event['entity_id']}"
        self.redis.delete(cache_key)

# Usage in application
def update_product(product_id: int, data: dict):
    # Update database
    db.execute("UPDATE products SET ... WHERE id = %s", (product_id,))

    # Invalidate cache
    cache.invalidate(CacheInvalidationEvent(
        entity_type='product',
        entity_id=str(product_id),
        action='update'
    ))
```

#### 3. Write-Through Invalidation

```python
class WriteThroughCache:
    """Update cache synchronously with database writes."""

    def __init__(self, redis_client, db_session):
        self.redis = redis_client
        self.db = db_session

    def update(self, entity_type: str, entity_id: str, data: dict):
        # Begin transaction
        try:
            # Update database
            self.db.execute(
                f"UPDATE {entity_type} SET data = %s WHERE id = %s",
                (json.dumps(data), entity_id)
            )

            # Update cache (same transaction context)
            cache_key = f"{entity_type}:{entity_id}"
            self.redis.set(cache_key, json.dumps(data))

            self.db.commit()
        except Exception as e:
            self.db.rollback()
            self.redis.delete(f"{entity_type}:{entity_id}")
            raise
```

#### 4. Write-Behind (Write-Back) Pattern

```python
class WriteBehindCache:
    """
    Update cache immediately, persist to database asynchronously.
    Better performance but risk of data loss.
    """

    def __init__(self, redis_client, write_queue):
        self.redis = redis_client
        self.queue = write_queue

    def update(self, entity_type: str, entity_id: str, data: dict):
        cache_key = f"{entity_type}:{entity_id}"

        # Update cache immediately
        self.redis.set(cache_key, json.dumps(data))

        # Queue database write
        self.queue.enqueue({
            'operation': 'update',
            'entity_type': entity_type,
            'entity_id': entity_id,
            'data': data,
            'timestamp': time.time()
        })

# Background worker processes queue
def process_write_queue():
    while True:
        write_op = queue.dequeue()
        try:
            db.execute(
                f"UPDATE {write_op['entity_type']} SET data = %s WHERE id = %s",
                (json.dumps(write_op['data']), write_op['entity_id'])
            )
        except Exception as e:
            # Log and potentially retry
            logger.error(f"Write-behind failed: {e}")
            queue.requeue(write_op)
```

#### 5. Cache-Aside with Version Tags

```python
class VersionedCache:
    """Use version tags to detect stale cache entries."""

    def __init__(self, redis_client):
        self.redis = redis_client

    def get(self, key: str) -> Optional[dict]:
        cached = self.redis.get(f"cache:{key}")
        if not cached:
            return None

        data = json.loads(cached)

        # Check if version matches current
        current_version = self.redis.get(f"version:{key}")
        if current_version and data.get('version') != current_version.decode():
            # Stale cache, treat as miss
            return None

        return data.get('value')

    def set(self, key: str, value: Any, ttl: int = 3600):
        version = str(uuid.uuid4())

        # Set both cache and version atomically
        pipe = self.redis.pipeline()
        pipe.setex(f"cache:{key}", ttl, json.dumps({
            'value': value,
            'version': version
        }))
        pipe.set(f"version:{key}", version)
        pipe.execute()

    def invalidate(self, key: str):
        # Just update version - cached data becomes stale
        self.redis.set(f"version:{key}", str(uuid.uuid4()))
```

#### 6. Tag-Based Invalidation

```python
class TaggedCache:
    """
    Associate cache entries with tags for bulk invalidation.
    Useful for invalidating related entries together.
    """

    def __init__(self, redis_client):
        self.redis = redis_client

    def set(self, key: str, value: Any, tags: List[str], ttl: int = 3600):
        pipe = self.redis.pipeline()

        # Store the cached value
        pipe.setex(f"cache:{key}", ttl, json.dumps(value))

        # Associate key with each tag
        for tag in tags:
            pipe.sadd(f"tag:{tag}", key)
            pipe.expire(f"tag:{tag}", ttl)

        pipe.execute()

    def invalidate_by_tag(self, tag: str):
        """Invalidate all cache entries with given tag."""
        keys = self.redis.smembers(f"tag:{tag}")

        if keys:
            pipe = self.redis.pipeline()
            for key in keys:
                pipe.delete(f"cache:{key.decode()}")
            pipe.delete(f"tag:{tag}")
            pipe.execute()

# Usage
cache = TaggedCache(redis_client)

# Cache product with category tags
cache.set(
    key='product:123',
    value=product_data,
    tags=['category:electronics', 'brand:apple', 'featured']
)

# Invalidate all electronics products
cache.invalidate_by_tag('category:electronics')
```

---

## Cache Warming

### What is Cache Warming?

Cache warming is the process of pre-populating the cache with data before it's requested, ensuring high hit rates from the start.

```
Cold Cache (after restart):
First 1000 requests -> 0% hit rate -> Database overload

Warmed Cache:
First 1000 requests -> 95%+ hit rate -> Smooth operation
```

### Cache Warming Strategies

#### 1. Startup Warming

```python
class CacheWarmer:
    def __init__(self, redis_client, db_session):
        self.redis = redis_client
        self.db = db_session

    async def warm_on_startup(self):
        """Warm cache during application startup."""
        warmers = [
            self.warm_popular_products(),
            self.warm_active_users(),
            self.warm_configuration(),
            self.warm_static_content()
        ]
        await asyncio.gather(*warmers)
        logger.info("Cache warming completed")

    async def warm_popular_products(self, limit: int = 1000):
        """Pre-load most viewed products."""
        products = self.db.execute("""
            SELECT * FROM products
            ORDER BY view_count DESC
            LIMIT %s
        """, (limit,)).fetchall()

        pipe = self.redis.pipeline()
        for product in products:
            pipe.setex(
                f"product:{product['id']}",
                3600,
                json.dumps(dict(product))
            )
        pipe.execute()
        logger.info(f"Warmed {len(products)} popular products")

    async def warm_active_users(self):
        """Pre-load recently active user profiles."""
        users = self.db.execute("""
            SELECT * FROM users
            WHERE last_login > NOW() - INTERVAL '24 hours'
            LIMIT 10000
        """).fetchall()

        pipe = self.redis.pipeline()
        for user in users:
            pipe.setex(
                f"user:{user['id']}",
                1800,
                json.dumps(dict(user))
            )
        pipe.execute()
        logger.info(f"Warmed {len(users)} active users")
```

#### 2. Scheduled Warming

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler

class ScheduledCacheWarmer:
    def __init__(self, redis_client, db_session):
        self.redis = redis_client
        self.db = db_session
        self.scheduler = AsyncIOScheduler()

    def start(self):
        # Warm trending items every 5 minutes
        self.scheduler.add_job(
            self.warm_trending,
            'interval',
            minutes=5
        )

        # Warm daily deals at midnight
        self.scheduler.add_job(
            self.warm_daily_deals,
            'cron',
            hour=0,
            minute=0
        )

        # Warm before peak hours
        self.scheduler.add_job(
            self.warm_peak_hour_data,
            'cron',
            hour=8,  # Before 9 AM peak
            minute=30
        )

        self.scheduler.start()

    async def warm_trending(self):
        """Warm currently trending items based on real-time analytics."""
        trending = await self.analytics.get_trending_items(limit=100)

        for item in trending:
            full_data = await self.db.get_item(item['id'])
            await self.redis.setex(
                f"item:{item['id']}",
                600,  # 10 minutes
                json.dumps(full_data)
            )
```

#### 3. Lazy Warming with Background Refresh

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class LazyWarmer:
    """
    On cache miss, return stale data (if available) and
    refresh in background.
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.executor = ThreadPoolExecutor(max_workers=10)

    async def get(self, key: str, fetch_func, ttl: int = 3600):
        # Try cache first
        cached = self.redis.get(f"cache:{key}")

        if cached:
            data = json.loads(cached)
            remaining_ttl = self.redis.ttl(f"cache:{key}")

            # If TTL < 20% remaining, trigger background refresh
            if remaining_ttl < ttl * 0.2:
                asyncio.create_task(
                    self._background_refresh(key, fetch_func, ttl)
                )

            return data

        # Cache miss - fetch and cache
        data = await fetch_func()
        self.redis.setex(f"cache:{key}", ttl, json.dumps(data))
        return data

    async def _background_refresh(self, key: str, fetch_func, ttl: int):
        """Refresh cache in background."""
        try:
            # Use lock to prevent multiple refreshes
            lock_key = f"lock:refresh:{key}"
            if self.redis.set(lock_key, 1, nx=True, ex=60):
                data = await fetch_func()
                self.redis.setex(f"cache:{key}", ttl, json.dumps(data))
                self.redis.delete(lock_key)
        except Exception as e:
            logger.error(f"Background refresh failed for {key}: {e}")
```

#### 4. Predictive Warming

```python
class PredictiveWarmer:
    """
    Use access patterns to predict and pre-warm data
    likely to be requested.
    """

    def __init__(self, redis_client, ml_model):
        self.redis = redis_client
        self.model = ml_model

    async def on_user_login(self, user_id: str):
        """Warm data based on user's predicted behavior."""
        # Get user's historical patterns
        history = await self.get_user_history(user_id)

        # Predict likely next actions
        predictions = self.model.predict_next_items(history)

        # Pre-warm predicted items
        for item_id, probability in predictions:
            if probability > 0.3:  # 30% threshold
                await self.warm_item(item_id)

    async def on_navigation(self, user_id: str, current_page: str):
        """Warm data for likely next pages."""
        # Predict next pages based on current page
        likely_pages = self.model.predict_navigation(current_page)

        for page, probability in likely_pages:
            if probability > 0.5:
                await self.warm_page_data(page)
```

---

## Caching Patterns

### 1. Cache-Aside (Lazy Loading)

```python
class CacheAsidePattern:
    """
    Application manages cache population.
    Most common pattern.
    """

    def get(self, key: str):
        # 1. Check cache
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # 2. Cache miss - fetch from database
        data = self.db.query(key)

        # 3. Populate cache
        if data:
            self.redis.setex(key, 3600, json.dumps(data))

        return data

    def update(self, key: str, data: dict):
        # Update database
        self.db.update(key, data)

        # Invalidate cache (NOT update)
        self.redis.delete(key)

# Flow:
# Read: Cache -> Miss -> DB -> Update Cache -> Return
# Write: DB -> Invalidate Cache
```

### 2. Read-Through Cache

```python
class ReadThroughCache:
    """
    Cache handles data fetching transparently.
    Application only interacts with cache.
    """

    def __init__(self, redis_client, data_loader):
        self.redis = redis_client
        self.loader = data_loader

    def get(self, key: str) -> Any:
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Cache handles the fetch
        data = self.loader.load(key)
        self.redis.setex(key, 3600, json.dumps(data))
        return data

# Application code becomes simpler
def get_product(product_id):
    return cache.get(f"product:{product_id}")
```

### 3. Write-Through Cache

```python
class WriteThroughCache:
    """
    Write to cache and database synchronously.
    Ensures consistency but higher write latency.
    """

    def set(self, key: str, value: Any):
        # Write to database first
        self.db.write(key, value)

        # Then update cache
        self.redis.set(key, json.dumps(value))

# Flow:
# Write -> DB -> Cache -> Return (synchronous)
```

### 4. Write-Behind (Write-Back) Cache

```python
class WriteBehindCache:
    """
    Write to cache immediately, database asynchronously.
    Better performance but data loss risk.
    """

    def set(self, key: str, value: Any):
        # Update cache immediately
        self.redis.set(key, json.dumps(value))

        # Queue async database write
        self.write_queue.enqueue({
            'key': key,
            'value': value,
            'timestamp': time.time()
        })

# Flow:
# Write -> Cache -> Return (immediate)
#       -> Queue -> DB (background)
```

### 5. Cache Stampede Prevention

```python
import threading

class StampedeProtectedCache:
    """
    Prevent cache stampede when popular keys expire.
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.local_locks = {}

    def get_with_lock(self, key: str, fetch_func, ttl: int = 3600):
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        # Try to acquire distributed lock
        lock_key = f"lock:{key}"
        lock_acquired = self.redis.set(lock_key, 1, nx=True, ex=30)

        if lock_acquired:
            try:
                # We got the lock, fetch and cache
                data = fetch_func()
                self.redis.setex(key, ttl, json.dumps(data))
                return data
            finally:
                self.redis.delete(lock_key)
        else:
            # Another process is fetching, wait and retry
            time.sleep(0.1)
            return self.get_with_lock(key, fetch_func, ttl)

    def get_with_stale(self, key: str, fetch_func, ttl: int = 3600):
        """
        Return stale data while refreshing in background.
        Requires storing data with timestamp.
        """
        cached = self.redis.get(key)

        if cached:
            data = json.loads(cached)

            # Check if data is stale but usable
            if data.get('expires_at', 0) < time.time():
                # Return stale, refresh in background
                threading.Thread(
                    target=self._refresh,
                    args=(key, fetch_func, ttl)
                ).start()

            return data.get('value')

        # No cached data, must fetch
        return self._fetch_and_cache(key, fetch_func, ttl)
```

---

## Distributed Caching

### Redis Cluster Setup

```python
from redis.cluster import RedisCluster

# Connect to Redis Cluster
cluster = RedisCluster(
    startup_nodes=[
        {"host": "redis1.example.com", "port": 6379},
        {"host": "redis2.example.com", "port": 6379},
        {"host": "redis3.example.com", "port": 6379},
    ],
    decode_responses=True,
    skip_full_coverage_check=True
)

# Keys are automatically sharded across nodes
cluster.set("user:123", json.dumps(user_data))
```

### Consistent Hashing

```python
import hashlib
from bisect import bisect_left

class ConsistentHash:
    """
    Consistent hashing for distributing cache keys
    across multiple nodes with minimal remapping.
    """

    def __init__(self, nodes: List[str], replicas: int = 100):
        self.replicas = replicas
        self.ring = {}
        self.sorted_keys = []

        for node in nodes:
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """Add node with virtual replicas."""
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = node
            self.sorted_keys.append(hash_value)

        self.sorted_keys.sort()

    def remove_node(self, node: str):
        """Remove node and its virtual replicas."""
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
            self.sorted_keys.remove(hash_value)

    def get_node(self, key: str) -> str:
        """Get node responsible for key."""
        if not self.ring:
            return None

        hash_value = self._hash(key)
        idx = bisect_left(self.sorted_keys, hash_value)

        if idx >= len(self.sorted_keys):
            idx = 0

        return self.ring[self.sorted_keys[idx]]
```

### Multi-Layer Caching

```python
class MultiLayerCache:
    """
    L1: Local in-memory cache (fastest, per-instance)
    L2: Distributed cache (shared across instances)
    L3: Database (source of truth)
    """

    def __init__(self, local_cache, redis_client, db_session):
        self.l1 = local_cache  # e.g., cachetools.TTLCache
        self.l2 = redis_client
        self.db = db_session

    def get(self, key: str) -> Optional[Any]:
        # L1: Check local cache
        if key in self.l1:
            return self.l1[key]

        # L2: Check distributed cache
        cached = self.l2.get(key)
        if cached:
            data = json.loads(cached)
            self.l1[key] = data  # Populate L1
            return data

        # L3: Fetch from database
        data = self.db.query(key)
        if data:
            self.set(key, data)

        return data

    def set(self, key: str, value: Any, ttl: int = 3600):
        # Update all layers
        self.l1[key] = value
        self.l2.setex(key, ttl, json.dumps(value))

    def invalidate(self, key: str):
        # Invalidate all layers
        self.l1.pop(key, None)
        self.l2.delete(key)

        # Publish invalidation for other instances
        self.l2.publish('cache_invalidation', key)
```

### Cache Replication Strategy

```python
class ReplicatedCache:
    """
    Write to primary, read from replicas for geographic distribution.
    """

    def __init__(self, primary_redis, replica_redis_list):
        self.primary = primary_redis
        self.replicas = replica_redis_list
        self.replica_index = 0

    def get(self, key: str) -> Optional[Any]:
        # Round-robin read from replicas
        replica = self.replicas[self.replica_index]
        self.replica_index = (self.replica_index + 1) % len(self.replicas)

        try:
            return replica.get(key)
        except Exception:
            # Fallback to primary
            return self.primary.get(key)

    def set(self, key: str, value: Any, ttl: int = 3600):
        # Write to primary (replicated automatically)
        self.primary.setex(key, ttl, value)
```

---

## Common Mistakes to Avoid

### 1. Caching Null Results

```python
# Bad: Not caching null results (cache penetration)
def get_user(user_id):
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    user = db.query(User).get(user_id)
    if user:
        redis.setex(f"user:{user_id}", 3600, json.dumps(user.to_dict()))
    return user  # Non-existent users always hit DB

# Good: Cache null results with shorter TTL
def get_user(user_id):
    cache_key = f"user:{user_id}"
    cached = redis.get(cache_key)

    if cached == "NULL":
        return None
    if cached:
        return json.loads(cached)

    user = db.query(User).get(user_id)
    if user:
        redis.setex(cache_key, 3600, json.dumps(user.to_dict()))
    else:
        redis.setex(cache_key, 300, "NULL")  # Cache miss for 5 min

    return user
```

### 2. Unbounded Cache Growth

```python
# Bad: No size limits
cache = {}
def cache_data(key, value):
    cache[key] = value  # Memory leak!

# Good: Use LRU with size limit
from cachetools import LRUCache

cache = LRUCache(maxsize=10000)

# Or with Redis maxmemory policy
# redis.conf:
# maxmemory 1gb
# maxmemory-policy allkeys-lru
```

### 3. Cache Key Collisions

```python
# Bad: Simple keys prone to collision
redis.set("user", user_data)  # Which user?
redis.set(str(user_id), user_data)  # What entity?

# Good: Namespaced, descriptive keys
redis.set(f"app:user:profile:{user_id}", user_data)
redis.set(f"app:user:settings:{user_id}", settings_data)

# With version for cache invalidation
redis.set(f"app:v2:user:profile:{user_id}", user_data)
```

### 4. Ignoring Serialization Costs

```python
# Bad: Serializing large objects on every access
def get_large_object(key):
    cached = redis.get(key)
    if cached:
        return pickle.loads(cached)  # Expensive deserialization

    obj = create_large_object()
    redis.set(key, pickle.dumps(obj))  # Expensive serialization
    return obj

# Good: Cache only necessary data, compress if needed
import zlib

def get_large_object(key):
    cached = redis.get(key)
    if cached:
        decompressed = zlib.decompress(cached)
        return json.loads(decompressed)

    obj = create_large_object()
    # Store only necessary fields
    slim_data = {
        'id': obj.id,
        'name': obj.name,
        'computed_value': obj.computed_value
    }
    compressed = zlib.compress(json.dumps(slim_data).encode())
    redis.set(key, compressed)
    return slim_data
```

### 5. No Cache Metrics

```python
# Bad: Flying blind without metrics
def get_cached_data(key):
    return redis.get(key) or fetch_from_db(key)

# Good: Track cache performance
class InstrumentedCache:
    def __init__(self, redis_client, metrics):
        self.redis = redis_client
        self.metrics = metrics

    def get(self, key):
        start = time.time()

        value = self.redis.get(key)

        latency = time.time() - start
        self.metrics.record_histogram('cache.latency', latency)

        if value:
            self.metrics.increment('cache.hits')
        else:
            self.metrics.increment('cache.misses')

        return value
```

### 6. Inconsistent Cache and Database

```python
# Bad: Race condition between cache and DB update
def update_user(user_id, data):
    # Gap between these operations can cause inconsistency
    db.update(user_id, data)
    time.sleep(0.1)  # Something happens here...
    redis.delete(f"user:{user_id}")  # Cache still has old data

# Good: Use transactions or ordered operations
def update_user(user_id, data):
    # Delete cache first, then update DB
    redis.delete(f"user:{user_id}")
    db.update(user_id, data)

    # Or use distributed transaction
    # Or accept eventual consistency with short TTL
```

---

## Implementation Guidelines

### Cache Implementation Checklist

```markdown
[ ] Cache Selection
    [ ] Identify cacheable data
    [ ] Choose cache type (local, distributed, CDN)
    [ ] Select eviction policy (LRU, LFU, TTL)

[ ] TTL Strategy
    [ ] Define TTL per data type
    [ ] Consider data volatility
    [ ] Plan for TTL refresh mechanism

[ ] Invalidation Strategy
    [ ] Choose invalidation approach
    [ ] Handle distributed invalidation
    [ ] Test invalidation scenarios

[ ] Resilience
    [ ] Handle cache failures gracefully
    [ ] Implement circuit breaker
    [ ] Plan for cache warmup

[ ] Monitoring
    [ ] Track hit/miss ratio
    [ ] Monitor latency
    [ ] Alert on hit ratio drops
    [ ] Track memory usage

[ ] Security
    [ ] Encrypt sensitive cached data
    [ ] Isolate cache per tenant
    [ ] Implement access controls
```

### Cache Configuration Template

```yaml
# cache_config.yaml
cache:
  type: redis

  connection:
    host: redis.example.com
    port: 6379
    password: ${REDIS_PASSWORD}
    database: 0

  pool:
    max_connections: 100
    min_idle: 10
    connection_timeout: 5s

  default_ttl: 3600  # 1 hour

  policies:
    user_profile:
      ttl: 1800           # 30 minutes
      warm_on_startup: true
      invalidate_on: [user.updated, user.deleted]

    product_catalog:
      ttl: 86400          # 24 hours
      warm_schedule: "0 */6 * * *"  # Every 6 hours
      invalidate_on: [product.updated, catalog.refresh]

    session:
      ttl: 900            # 15 minutes
      sliding_ttl: true   # Reset on access

  eviction:
    policy: allkeys-lru
    max_memory: 1gb

  monitoring:
    metrics_enabled: true
    slow_log_threshold: 10ms
```

---

## Interview Talking Points

### Question: "How would you implement caching for a high-traffic e-commerce site?"

**Strong Answer Framework**:

1. **Multi-layer caching**: CDN for static, Redis for dynamic, local for hot data
2. **Different TTLs**: Long for catalog (hours), short for inventory (minutes)
3. **Invalidation strategy**: Event-driven for price/inventory updates
4. **Cache warming**: Pre-load popular products before peak hours
5. **Stampede prevention**: Distributed locks or stale-while-revalidate

### Question: "How do you handle cache invalidation in a microservices architecture?"

**Strong Answer Framework**:

1. **Event-driven**: Publish invalidation events to message bus
2. **Tag-based**: Group related entries for bulk invalidation
3. **Version tags**: Increment version on update, stale check on read
4. **TTL as safety net**: Even with events, use TTL as fallback
5. **Eventual consistency**: Accept brief staleness, optimize for availability

### Question: "Explain the cache stampede problem and how to prevent it."

**Key Points**:

1. **What it is**: Many requests hit expired key simultaneously
2. **Why it matters**: Can overwhelm database, cause cascading failures
3. **Solutions**:
   - Distributed locks (only one refreshes)
   - Probabilistic early refresh (stagger expirations)
   - Stale-while-revalidate (serve old, refresh background)
   - Hot key splitting (multiple copies with random selection)

### Common Follow-up Questions

- How do you decide what to cache?
- What metrics would you monitor for cache health?
- How do you handle cache failures?
- When would you choose Redis vs Memcached?

---

## Key Takeaways

### 1. Choose TTL Thoughtfully
- Match TTL to data volatility
- Use adaptive TTL for variable access patterns
- Consider probabilistic early refresh

### 2. Invalidation is Critical
- Event-driven beats time-based for consistency
- Use tags for related data invalidation
- Always have TTL as safety net

### 3. Warm Your Cache
- Pre-populate before traffic hits
- Warm based on predicted access patterns
- Background refresh before expiration

### 4. Prevent Stampedes
- Use distributed locks for popular keys
- Serve stale data while refreshing
- Implement circuit breakers

### 5. Monitor Everything
- Track hit ratio religiously (target >95%)
- Alert on sudden hit ratio drops
- Monitor memory usage and evictions

### 6. Plan for Failure
- Cache should never be required for operation
- Graceful degradation to database
- Circuit breakers for cache failures

---

## Quick Reference

| Aspect | Best Practice |
|--------|--------------|
| Hit Ratio Target | >95% |
| TTL Strategy | Match data volatility |
| Invalidation | Event-driven + TTL backup |
| Stampede Prevention | Distributed lock or stale-serve |
| Cache Warming | Startup + scheduled + predictive |
| Key Format | `app:version:entity:id` |
| Null Caching | Cache misses with short TTL |
| Monitoring | Hit ratio, latency, memory |
