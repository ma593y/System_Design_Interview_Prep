# URL Shortener System Design (TinyURL / Bitly)

## 1. Problem Statement

Design a URL shortening service like TinyURL or Bitly that takes long URLs and creates short, unique aliases. When users access the short URL, they are redirected to the original long URL.

**Core Functionality:**
- Given a long URL, generate a shorter and unique alias
- When users access the short link, redirect them to the original URL
- Users can optionally create custom short URLs
- Links can have expiration times

**Real-World Examples:**
- TinyURL
- Bitly
- t.co (Twitter)
- goo.gl (deprecated Google service)

---

## 2. Requirements Clarification

### Functional Requirements

| Requirement | Description |
|-------------|-------------|
| URL Shortening | Generate a short URL for any valid long URL |
| URL Redirection | Redirect short URL to original long URL |
| Custom Aliases | Users can optionally specify custom short URLs |
| Link Expiration | URLs can have an optional expiration time |
| Analytics (Optional) | Track click counts, referrers, geographic data |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| High Availability | 99.99% uptime (< 52 minutes downtime/year) |
| Low Latency | Redirection should happen in < 100ms |
| Scalability | Handle billions of URLs and redirections |
| Durability | Once created, URLs should not be lost |
| Security | Prevent malicious URL creation and abuse |

### Out of Scope
- User authentication and management
- Detailed analytics dashboard
- API rate limiting (covered in separate case study)

---

## 3. Capacity Estimation

### Assumptions

```
- 500 million new URL shortenings per month
- Read:Write ratio = 100:1 (reads are much more frequent)
- Average long URL length: 200 characters
- Short URL length: 7 characters
- Data retention: 5 years
```

### Traffic Estimates

```
Write Operations:
- New URLs per month: 500 million
- New URLs per second: 500M / (30 days * 24 hours * 3600 sec)
                     = 500M / 2.6M
                     ≈ 200 URLs/second

Read Operations (Redirections):
- Read:Write ratio = 100:1
- Redirections per second: 200 * 100 = 20,000 redirections/second
```

### Storage Estimates

```
Per URL entry:
- Short URL: 7 bytes
- Long URL: 200 bytes (average)
- Created timestamp: 8 bytes
- Expiration timestamp: 8 bytes
- User ID (optional): 8 bytes
- Total: ~250 bytes per entry

Storage for 5 years:
- Total URLs: 500M/month * 12 months * 5 years = 30 billion URLs
- Storage needed: 30B * 250 bytes = 7.5 TB
```

### Bandwidth Estimates

```
Write bandwidth:
- 200 URLs/sec * 250 bytes = 50 KB/s

Read bandwidth:
- 20,000 redirections/sec * 250 bytes = 5 MB/s
```

### Memory Estimates (Caching)

```
Cache hot URLs (80-20 rule: 20% of URLs get 80% of traffic)
- Daily requests: 20,000/sec * 86,400 sec = 1.7 billion/day
- Cache 20% of daily requests
- Unique URLs to cache: 1.7B * 0.2 = 340 million URLs
- Memory needed: 340M * 250 bytes = 85 GB
```

---

## 4. High-Level Design

```
                                    ┌─────────────────┐
                                    │   CDN / Edge    │
                                    │   (Optional)    │
                                    └────────┬────────┘
                                             │
                                             ▼
┌──────────┐         ┌─────────────────────────────────────────┐
│  Client  │────────▶│            Load Balancer                │
└──────────┘         └─────────────────────────────────────────┘
                                             │
                     ┌───────────────────────┼───────────────────────┐
                     │                       │                       │
                     ▼                       ▼                       ▼
              ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
              │   App       │         │   App       │         │   App       │
              │  Server 1   │         │  Server 2   │         │  Server N   │
              └─────────────┘         └─────────────┘         └─────────────┘
                     │                       │                       │
                     └───────────────────────┼───────────────────────┘
                                             │
                     ┌───────────────────────┴───────────────────────┐
                     │                                               │
                     ▼                                               ▼
              ┌─────────────┐                                 ┌─────────────┐
              │   Redis     │                                 │  Key        │
              │   Cache     │                                 │  Generation │
              │   Cluster   │                                 │  Service    │
              └─────────────┘                                 └─────────────┘
                     │                                               │
                     │                                               │
                     ▼                                               ▼
              ┌─────────────────────────────────────────────────────────────┐
              │                     Database Cluster                        │
              │  ┌─────────┐    ┌─────────┐    ┌─────────┐                 │
              │  │ Shard 1 │    │ Shard 2 │    │ Shard N │                 │
              │  │ Primary │    │ Primary │    │ Primary │                 │
              │  └────┬────┘    └────┬────┘    └────┬────┘                 │
              │       │              │              │                       │
              │  ┌────┴────┐    ┌────┴────┐    ┌────┴────┐                 │
              │  │ Replica │    │ Replica │    │ Replica │                 │
              │  └─────────┘    └─────────┘    └─────────┘                 │
              └─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| Load Balancer | Distribute traffic across app servers |
| App Servers | Handle URL shortening and redirection logic |
| Cache (Redis) | Store frequently accessed URLs for fast retrieval |
| Key Generation Service | Generate unique short URL keys |
| Database | Persistent storage for URL mappings |

---

## 5. API Design

### REST API Endpoints

#### Create Short URL

```http
POST /api/v1/urls
Content-Type: application/json

Request Body:
{
    "long_url": "https://www.example.com/very/long/path/to/resource",
    "custom_alias": "my-custom-url",    // optional
    "expiration_date": "2025-12-31",    // optional
    "user_id": "user123"                // optional
}

Response (201 Created):
{
    "short_url": "https://short.url/abc1234",
    "long_url": "https://www.example.com/very/long/path/to/resource",
    "short_key": "abc1234",
    "created_at": "2024-01-15T10:30:00Z",
    "expires_at": "2025-12-31T23:59:59Z"
}

Response (400 Bad Request):
{
    "error": "Invalid URL format",
    "code": "INVALID_URL"
}
```

#### Redirect to Long URL

```http
GET /{short_key}

Response (302 Found):
Location: https://www.example.com/very/long/path/to/resource

Response (404 Not Found):
{
    "error": "URL not found or expired",
    "code": "URL_NOT_FOUND"
}
```

#### Get URL Info

```http
GET /api/v1/urls/{short_key}

Response (200 OK):
{
    "short_url": "https://short.url/abc1234",
    "long_url": "https://www.example.com/very/long/path/to/resource",
    "created_at": "2024-01-15T10:30:00Z",
    "expires_at": "2025-12-31T23:59:59Z",
    "click_count": 1542
}
```

#### Delete URL

```http
DELETE /api/v1/urls/{short_key}

Response (204 No Content)
```

---

## 6. Database Schema

### Primary Table: urls

```sql
CREATE TABLE urls (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_key       VARCHAR(10) UNIQUE NOT NULL,
    long_url        VARCHAR(2048) NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at      TIMESTAMP NULL,
    user_id         VARCHAR(50) NULL,
    click_count     BIGINT DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,

    INDEX idx_short_key (short_key),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

### Analytics Table (Optional)

```sql
CREATE TABLE url_analytics (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_key       VARCHAR(10) NOT NULL,
    accessed_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    referrer        VARCHAR(2048),
    user_agent      VARCHAR(512),
    ip_address      VARCHAR(45),
    country_code    VARCHAR(2),

    INDEX idx_short_key_time (short_key, accessed_at),
    FOREIGN KEY (short_key) REFERENCES urls(short_key)
);
```

### Key Range Table (For Key Generation)

```sql
CREATE TABLE key_ranges (
    id              INT PRIMARY KEY AUTO_INCREMENT,
    range_start     BIGINT NOT NULL,
    range_end       BIGINT NOT NULL,
    assigned_to     VARCHAR(50),
    assigned_at     TIMESTAMP,
    is_exhausted    BOOLEAN DEFAULT FALSE
);
```

### Data Model Diagram

```
┌─────────────────────────────────────────────┐
│                    urls                      │
├─────────────────────────────────────────────┤
│ PK │ id            │ BIGINT                 │
│    │ short_key     │ VARCHAR(10) UNIQUE     │
│    │ long_url      │ VARCHAR(2048)          │
│    │ created_at    │ TIMESTAMP              │
│    │ expires_at    │ TIMESTAMP              │
│ FK │ user_id       │ VARCHAR(50)            │
│    │ click_count   │ BIGINT                 │
│    │ is_active     │ BOOLEAN                │
└─────────────────────────────────────────────┘
                      │
                      │ 1:N
                      ▼
┌─────────────────────────────────────────────┐
│              url_analytics                   │
├─────────────────────────────────────────────┤
│ PK │ id            │ BIGINT                 │
│ FK │ short_key     │ VARCHAR(10)            │
│    │ accessed_at   │ TIMESTAMP              │
│    │ referrer      │ VARCHAR(2048)          │
│    │ user_agent    │ VARCHAR(512)           │
│    │ ip_address    │ VARCHAR(45)            │
│    │ country_code  │ VARCHAR(2)             │
└─────────────────────────────────────────────┘
```

---

## 7. Deep Dive: Critical Components

### 7.1 Short URL Key Generation

This is the most critical component. We need to generate unique, short keys efficiently.

#### Option 1: Base62 Encoding

```
Characters: a-z, A-Z, 0-9 (62 characters)
Key length: 7 characters
Total combinations: 62^7 = 3.5 trillion unique keys
```

**Algorithm:**
```python
import hashlib

BASE62 = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

def encode_base62(num):
    """Convert a number to base62 string"""
    if num == 0:
        return BASE62[0]

    result = []
    while num > 0:
        result.append(BASE62[num % 62])
        num //= 62

    return ''.join(reversed(result))

def generate_short_key(long_url, counter):
    """Generate short key using hash + counter"""
    # Combine URL with counter to ensure uniqueness
    unique_string = f"{long_url}_{counter}"

    # MD5 hash (we only need uniqueness, not security)
    hash_bytes = hashlib.md5(unique_string.encode()).digest()

    # Convert first 6 bytes to integer
    num = int.from_bytes(hash_bytes[:6], 'big')

    # Encode to base62 (7 characters)
    short_key = encode_base62(num)[:7]

    return short_key.ljust(7, 'a')  # Pad if necessary
```

#### Option 2: Pre-Generated Key Database (Recommended)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Key Generation Service                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────┐        ┌───────────────────┐            │
│   │   Key DB          │        │   Key DB          │            │
│   │   (Unused Keys)   │───────▶│   (Used Keys)     │            │
│   │                   │        │                   │            │
│   │   abc1234         │        │   xyz7890 ✓       │            │
│   │   def5678         │        │   mno3456 ✓       │            │
│   │   ghi9012         │        │                   │            │
│   │   ...             │        │                   │            │
│   └───────────────────┘        └───────────────────┘            │
│                                                                  │
│   Pre-generate millions of keys offline                          │
│   Move key from unused to used when URL is created               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Advantages:**
- No collision handling needed
- Very fast key retrieval (O(1))
- Can pre-validate keys are truly unique

#### Option 3: Counter-Based with Range Allocation

```
┌─────────────────────────────────────────────────────────────────┐
│                   Range Allocation System                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Zookeeper/Coordination Service                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Server 1: Range [1 - 1,000,000]                        │   │
│   │  Server 2: Range [1,000,001 - 2,000,000]                │   │
│   │  Server 3: Range [2,000,001 - 3,000,000]                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Each server generates keys from its allocated range            │
│   No coordination needed between servers                         │
│   Request new range when current is exhausted                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 URL Redirection Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                     Redirection Flow                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   User Request: GET /abc1234                                      │
│                                                                   │
│   ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐   │
│   │ Request │────▶│  Cache  │────▶│   DB    │────▶│Redirect │   │
│   └─────────┘     │  Hit?   │     │  Query  │     │  301/302│   │
│                   └────┬────┘     └────┬────┘     └─────────┘   │
│                        │               │                         │
│                    Yes │           No  │                         │
│                        │               │                         │
│                        ▼               ▼                         │
│                   Return URL      Query DB                       │
│                   from cache      & cache                        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**301 vs 302 Redirect:**

| Redirect Type | Use Case | Caching |
|---------------|----------|---------|
| 301 (Permanent) | URL will never change | Browser caches, reduces server load |
| 302 (Temporary) | Need to track analytics | Each request hits server |

### 7.3 Caching Strategy

```python
class URLCache:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.TTL = 3600 * 24  # 24 hours

    def get_long_url(self, short_key):
        """Try cache first, then database"""
        # Check cache
        long_url = self.redis.get(f"url:{short_key}")

        if long_url:
            return long_url.decode('utf-8')

        # Cache miss - query database
        long_url = self.db_lookup(short_key)

        if long_url:
            # Populate cache
            self.redis.setex(f"url:{short_key}", self.TTL, long_url)

        return long_url

    def cache_url(self, short_key, long_url):
        """Cache a URL mapping"""
        self.redis.setex(f"url:{short_key}", self.TTL, long_url)
```

---

## 8. Scaling Discussion

### Database Sharding

```
Sharding Strategy: Hash-based on short_key

┌───────────────────────────────────────────────────────────────┐
│                    Sharding by Short Key                       │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│   shard_id = hash(short_key) % num_shards                     │
│                                                                │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│   │   Shard 0    │  │   Shard 1    │  │   Shard 2    │        │
│   │  a*, b*, c*  │  │  d*, e*, f*  │  │  g*, h*, i*  │        │
│   └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                                │
│   Benefits:                                                    │
│   - Even distribution                                          │
│   - Easy to locate shard for any key                          │
│   - Can add shards with consistent hashing                    │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### Geographic Distribution

```
┌─────────────────────────────────────────────────────────────────┐
│                    Multi-Region Setup                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐         ┌─────────────┐        ┌─────────────┐│
│   │   US-East   │◀───────▶│  US-West    │◀──────▶│   Europe    ││
│   │   Primary   │         │   Replica   │        │   Replica   ││
│   └─────────────┘         └─────────────┘        └─────────────┘│
│                                                                  │
│   - Writes go to primary region                                  │
│   - Reads served from nearest replica                            │
│   - Async replication between regions                            │
│   - DNS-based routing to nearest data center                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Cache Scaling

```
┌─────────────────────────────────────────────────────────────────┐
│                    Redis Cluster Setup                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Redis Cluster                          │   │
│   │                                                          │   │
│   │   ┌─────────┐     ┌─────────┐     ┌─────────┐           │   │
│   │   │Master 1 │     │Master 2 │     │Master 3 │           │   │
│   │   │Slots    │     │Slots    │     │Slots    │           │   │
│   │   │0-5460   │     │5461-10922│    │10923-16383│         │   │
│   │   └────┬────┘     └────┬────┘     └────┬────┘           │   │
│   │        │               │               │                 │   │
│   │   ┌────┴────┐     ┌────┴────┐     ┌────┴────┐           │   │
│   │   │Replica 1│     │Replica 2│     │Replica 3│           │   │
│   │   └─────────┘     └─────────┘     └─────────┘           │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   - Automatic failover                                           │
│   - Data partitioned across nodes                                │
│   - ~85GB total memory for hot URLs                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Trade-Offs

### Key Generation Approaches

| Approach | Pros | Cons |
|----------|------|------|
| Hash-based | Simple, no coordination | Collision handling needed |
| Pre-generated keys | Fast, no collisions | Storage overhead, sync complexity |
| Counter with ranges | Simple, scalable | Requires coordination service |

### Storage Choices

| Database | Use Case | Trade-off |
|----------|----------|-----------|
| MySQL/PostgreSQL | ACID compliance, complex queries | Scaling limitations |
| Cassandra | High write throughput, easy scaling | Eventually consistent |
| DynamoDB | Managed, auto-scaling | Cost at scale, vendor lock-in |

### Redirect Type Decision

| Choice | When to Use |
|--------|-------------|
| 301 Permanent | Maximum performance, no analytics needed |
| 302 Temporary | Need click tracking, URL might change |

### Cache vs Database Reads

```
Trade-off Analysis:

High Cache TTL (24h):
  + Fewer database reads
  + Lower latency
  - Stale data for longer
  - Higher memory usage

Low Cache TTL (1h):
  + Fresher data
  + Lower memory usage
  - More database load
  - Higher latency on cache miss
```

---

## 10. Bottlenecks & Solutions

### Bottleneck 1: Database Write Throughput

```
Problem: Single database can't handle 200+ writes/second at scale

Solutions:
┌────────────────────────────────────────────────────────────┐
│ 1. Database Sharding                                        │
│    - Distribute writes across multiple database instances   │
│    - Use consistent hashing for even distribution           │
│                                                             │
│ 2. Write-Behind Caching                                     │
│    - Buffer writes in memory/queue                          │
│    - Batch insert to database                               │
│                                                             │
│ 3. NoSQL Database                                           │
│    - Use Cassandra/DynamoDB for better write scaling        │
└────────────────────────────────────────────────────────────┘
```

### Bottleneck 2: Key Generation Collisions

```
Problem: Hash collisions when generating short keys

Solutions:
┌────────────────────────────────────────────────────────────┐
│ 1. Retry with Salt                                          │
│    - On collision, add random salt and regenerate           │
│    - Retry up to N times before failing                     │
│                                                             │
│ 2. Pre-generated Key Pool                                   │
│    - Generate unique keys offline                           │
│    - No runtime collision possible                          │
│                                                             │
│ 3. Bloom Filter                                             │
│    - Quick check if key might exist                         │
│    - Reduce database lookups                                │
└────────────────────────────────────────────────────────────┘
```

### Bottleneck 3: Hot URLs (Viral Content)

```
Problem: Single URL gets millions of requests (viral content)

Solutions:
┌────────────────────────────────────────────────────────────┐
│ 1. CDN Caching                                              │
│    - Cache redirects at edge locations                      │
│    - Reduces load on origin servers                         │
│                                                             │
│ 2. Cache Replication                                        │
│    - Replicate hot keys across all cache nodes              │
│    - Any node can serve the request                         │
│                                                             │
│ 3. Local Caching                                            │
│    - In-memory cache on each app server                     │
│    - LRU eviction for memory management                     │
└────────────────────────────────────────────────────────────┘
```

### Bottleneck 4: Expired URL Cleanup

```
Problem: Billions of expired URLs consuming storage

Solutions:
┌────────────────────────────────────────────────────────────┐
│ 1. Lazy Deletion                                            │
│    - Check expiration on access                             │
│    - Delete if expired                                      │
│                                                             │
│ 2. Background Cleanup Job                                   │
│    - Scheduled job to delete expired URLs                   │
│    - Run during off-peak hours                              │
│                                                             │
│ 3. TTL-based Storage                                        │
│    - Use Cassandra/DynamoDB with TTL feature                │
│    - Automatic expiration handling                          │
└────────────────────────────────────────────────────────────┘
```

---

## 11. Interview Variations

### Common Follow-up Questions

**Q1: How would you handle custom aliases?**
```
Answer:
1. Check if custom alias already exists in database
2. If exists, return error to user
3. If not, use it directly as short_key
4. Consider reserving common words (admin, api, help, etc.)
5. Validate alias format (alphanumeric, length limits)
```

**Q2: How would you implement analytics?**
```
Answer:
1. Use 302 redirects instead of 301
2. Log each click asynchronously (Kafka/RabbitMQ)
3. Process logs in batch with Spark/Flink
4. Store aggregated data in time-series database
5. Consider privacy implications (GDPR compliance)
```

**Q3: How would you prevent abuse?**
```
Answer:
1. Rate limiting per IP/user
2. URL validation (check if URL is reachable)
3. Blacklist malicious domains
4. CAPTCHA for anonymous users
5. Scan destination URLs for malware
6. Report/flag mechanism for users
```

**Q4: How would you make URLs guessable/unguessable?**
```
Answer:
For unguessable (security):
- Use cryptographically random keys
- Longer key length (10+ characters)
- Include special characters if supported

For guessable (convenience):
- Sequential counters with base62 encoding
- Shorter keys are more predictable
```

**Q5: How would you implement URL previews?**
```
Answer:
1. Add preview page before redirect
2. Show destination URL, title, description
3. Fetch metadata asynchronously (Open Graph tags)
4. Cache preview data
5. Allow users to skip directly or view preview
```

**Q6: Design for specific scale (1 billion URLs/day)**
```
Answer:
1. ~12,000 writes/second (vs 200 in original)
2. Need 60x more capacity
3. Heavy database sharding (100+ shards)
4. Larger cache cluster (multiple TB)
5. Multiple data centers for redundancy
6. Consider dedicated key generation cluster
```

### System Design Interview Tips

1. **Start with requirements clarification** - Don't assume
2. **Do back-of-envelope calculations** - Shows analytical thinking
3. **Draw high-level design first** - Then dive deep
4. **Discuss trade-offs explicitly** - There's no perfect solution
5. **Address failure scenarios** - What happens when X fails?
6. **Consider security and abuse** - Real systems need protection

---

## Summary

| Aspect | Decision |
|--------|----------|
| Key Generation | Counter-based with range allocation |
| Database | Sharded MySQL/PostgreSQL or Cassandra |
| Caching | Redis Cluster with 24-hour TTL |
| Redirect Type | 302 for analytics, 301 for performance |
| Scaling | Horizontal scaling with consistent hashing |

The URL shortener is an excellent beginner system design problem because it covers fundamental concepts:
- Unique ID generation
- Database design and sharding
- Caching strategies
- API design
- Scalability patterns
