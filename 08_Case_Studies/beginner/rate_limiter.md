# Rate Limiter System Design - API Rate Limiting

## 1. Problem Statement

Design a rate limiting system that controls the number of requests a client can make to an API within a specified time window. The system should protect services from abuse, ensure fair usage, and maintain service availability under high load.

**Core Functionality:**
- Limit requests per client (by user ID, API key, or IP address)
- Support multiple rate limiting rules (per endpoint, per user tier)
- Return appropriate headers and error responses
- Handle distributed environments (multiple API servers)
- Provide real-time rate limit status

**Real-World Examples:**
- GitHub API (5000 requests/hour for authenticated users)
- Twitter API (300 requests/15-minute window)
- Stripe API (100 requests/second)
- AWS API Gateway rate limiting

---

## 2. Requirements

### 2.1 Functional Requirements

| Requirement | Description |
|-------------|-------------|
| **Limit Requests** | Block requests exceeding defined thresholds |
| **Multiple Identifiers** | Rate limit by user ID, API key, IP, or combination |
| **Multiple Rules** | Different limits per endpoint, user tier, time window |
| **Response Headers** | Return X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset |
| **Graceful Rejection** | Return 429 Too Many Requests with retry-after |
| **Soft vs Hard Limits** | Support warnings before hard cutoff |
| **Whitelist/Blacklist** | Exempt or block specific clients |

### 2.2 Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Latency** | < 1ms overhead per request |
| **Availability** | 99.99% (rate limiter failure shouldn't block requests) |
| **Accuracy** | < 1% error in rate counting |
| **Scalability** | Handle 1M+ requests/second |
| **Consistency** | Accurate across distributed servers |

### 2.3 Out of Scope
- DDoS protection (network-level)
- Authentication/Authorization
- Request queuing (leaky bucket implementation)
- Billing integration

---

## 3. Capacity Estimation

### 3.1 Traffic Assumptions

```
System Scale:
- Protected APIs: 100 services
- Total QPS across all services: 1,000,000 requests/second
- Unique clients: 10 million active per day
- Rate limit rules: 1000 different rules
- Average rules per request: 3 (global, per-user, per-endpoint)

Rate Limit Checks:
- Checks per second: 1,000,000 × 3 = 3,000,000 checks/second
- Peak (3x): 9,000,000 checks/second
```

### 3.2 Storage Estimates

```
Per Client Counter:
- Client identifier: 32 bytes (hash)
- Counter value: 8 bytes
- Window timestamp: 8 bytes
- TTL metadata: 8 bytes
Total: ~56 bytes per counter

Storage Calculation:
- Active counters: 10M clients × 3 rules = 30M counters
- Memory needed: 30M × 56 bytes = 1.68 GB
- With overhead (2x): ~3.5 GB

Redis Cluster Sizing:
- 3 nodes with 2 GB each = 6 GB total
- Provides redundancy and read scaling
```

### 3.3 Network Estimates

```
Rate Limit Check:
- Request size: ~100 bytes
- Response size: ~50 bytes
- Per check: 150 bytes

Bandwidth:
- 3M checks/second × 150 bytes = 450 MB/s
- Network between app servers and Redis
```

---

## 4. High-Level Design

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                CLIENT REQUEST                                    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              LOAD BALANCER                                       │
│                        (AWS ALB / Nginx / Envoy)                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY                                         │
│                                                                                  │
│   ┌────────────────────────────────────────────────────────────────────────┐   │
│   │                        RATE LIMITER MIDDLEWARE                          │   │
│   │                                                                         │   │
│   │   1. Extract client identifier (API key, user ID, IP)                  │   │
│   │   2. Determine applicable rules (endpoint, user tier)                  │   │
│   │   3. Check rate limits against Redis                                   │   │
│   │   4. Allow/Deny with appropriate headers                               │   │
│   │                                                                         │   │
│   └────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                    │                                        │
                    │                                        │
          ┌─────────┴─────────┐                    ┌────────┴────────┐
          ▼                   ▼                    ▼                  ▼
    ┌──────────┐        ┌──────────┐         ┌──────────┐      ┌──────────┐
    │  Allow   │        │  Reject  │         │  API     │      │  API     │
    │ Request  │        │  429     │         │ Server 1 │      │ Server N │
    └──────────┘        └──────────┘         └──────────┘      └──────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              REDIS CLUSTER                                       │
│                                                                                  │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐            │
│   │   Master 1      │    │   Master 2      │    │   Master 3      │            │
│   │   (Slots 0-5460)│    │(Slots 5461-10922)│   │(Slots 10923-16383)│          │
│   └────────┬────────┘    └────────┬────────┘    └────────┬────────┘            │
│            │                      │                      │                      │
│   ┌────────┴────────┐    ┌────────┴────────┐    ┌────────┴────────┐            │
│   │   Replica 1     │    │   Replica 2     │    │   Replica 3     │            │
│   └─────────────────┘    └─────────────────┘    └─────────────────┘            │
│                                                                                  │
│   Data Structure per Client:                                                    │
│   Key: ratelimit:{client_id}:{rule_id}:{window}                                │
│   Value: counter (integer)                                                      │
│   TTL: window duration                                                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CONFIGURATION STORE                                    │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │   Rate Limit Rules Database                                             │   │
│   │                                                                          │   │
│   │   - Rule definitions (limits, windows, tiers)                           │   │
│   │   - Client tier mappings                                                │   │
│   │   - Whitelist/Blacklist                                                 │   │
│   │   - Cached in Redis with pub/sub for updates                           │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Purpose |
|-----------|---------|
| **Load Balancer** | Distribute traffic, basic DDoS protection |
| **API Gateway** | Host rate limiter middleware |
| **Rate Limiter Middleware** | Core rate limiting logic |
| **Redis Cluster** | Distributed counter storage |
| **Configuration Store** | Rule definitions, client tiers |

---

## 5. API Design

### 5.1 Rate Limited Request Flow

```http
Request:
GET /api/v1/users/123
Authorization: Bearer <token>
X-API-Key: abc123

Response (Allowed - 200 OK):
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1705315200
X-RateLimit-Policy: 1000;w=3600

{
    "id": 123,
    "name": "John Doe"
}

Response (Rejected - 429 Too Many Requests):
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705315200
Retry-After: 3600

{
    "error": {
        "code": "RATE_LIMIT_EXCEEDED",
        "message": "Rate limit exceeded. Please retry after 3600 seconds.",
        "retry_after": 3600
    }
}
```

### 5.2 Rate Limit Configuration API

```http
# Create/Update Rate Limit Rule
POST /admin/api/v1/rate-limits/rules
Authorization: Bearer <admin_token>

Request Body:
{
    "rule_id": "api_default",
    "name": "Default API Rate Limit",
    "limit": 1000,
    "window_seconds": 3600,
    "identifier_type": "api_key",  // api_key, user_id, ip, composite
    "applies_to": {
        "endpoints": ["/api/*"],
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "user_tiers": ["free", "basic"]
    },
    "action": "reject",  // reject, throttle, log_only
    "priority": 10
}

Response (201 Created):
{
    "rule_id": "api_default",
    "status": "active",
    "created_at": "2024-01-15T10:30:00Z"
}

# Get Client Rate Limit Status
GET /api/v1/rate-limits/status
Authorization: Bearer <token>

Response (200 OK):
{
    "client_id": "user_123",
    "limits": [
        {
            "rule_id": "api_default",
            "limit": 1000,
            "remaining": 850,
            "reset_at": "2024-01-15T11:00:00Z",
            "window_seconds": 3600
        },
        {
            "rule_id": "search_endpoint",
            "limit": 100,
            "remaining": 45,
            "reset_at": "2024-01-15T10:35:00Z",
            "window_seconds": 60
        }
    ]
}
```

### 5.3 Whitelist/Blacklist API

```http
# Add to Whitelist
POST /admin/api/v1/rate-limits/whitelist
Authorization: Bearer <admin_token>

Request Body:
{
    "identifier": "api_key_premium_123",
    "identifier_type": "api_key",
    "reason": "Enterprise customer with unlimited access"
}

# Add to Blacklist
POST /admin/api/v1/rate-limits/blacklist
Authorization: Bearer <admin_token>

Request Body:
{
    "identifier": "192.168.1.100",
    "identifier_type": "ip",
    "reason": "Detected abuse pattern",
    "expires_at": "2024-01-16T10:30:00Z"
}
```

---

## 6. Database Schema

### 6.1 Configuration Database (PostgreSQL)

```sql
-- Rate Limit Rules
CREATE TABLE rate_limit_rules (
    id              SERIAL PRIMARY KEY,
    rule_id         VARCHAR(100) UNIQUE NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    limit_count     INT NOT NULL,
    window_seconds  INT NOT NULL,
    identifier_type VARCHAR(50) NOT NULL,  -- api_key, user_id, ip, composite
    endpoint_pattern VARCHAR(500),
    http_methods    VARCHAR(100)[],
    user_tiers      VARCHAR(50)[],
    action          VARCHAR(50) DEFAULT 'reject',  -- reject, throttle, log_only
    priority        INT DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_rule_id (rule_id),
    INDEX idx_active (is_active),
    INDEX idx_priority (priority DESC)
);

-- User Tiers
CREATE TABLE user_tiers (
    id              SERIAL PRIMARY KEY,
    tier_name       VARCHAR(50) UNIQUE NOT NULL,
    description     TEXT,
    is_default      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Client Tier Mappings
CREATE TABLE client_tiers (
    id              SERIAL PRIMARY KEY,
    client_id       VARCHAR(255) NOT NULL,
    identifier_type VARCHAR(50) NOT NULL,
    tier_id         INT REFERENCES user_tiers(id),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(client_id, identifier_type),
    INDEX idx_client (client_id, identifier_type)
);

-- Whitelist
CREATE TABLE rate_limit_whitelist (
    id              SERIAL PRIMARY KEY,
    identifier      VARCHAR(255) NOT NULL,
    identifier_type VARCHAR(50) NOT NULL,
    reason          TEXT,
    created_by      VARCHAR(100),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(identifier, identifier_type),
    INDEX idx_whitelist (identifier, identifier_type)
);

-- Blacklist
CREATE TABLE rate_limit_blacklist (
    id              SERIAL PRIMARY KEY,
    identifier      VARCHAR(255) NOT NULL,
    identifier_type VARCHAR(50) NOT NULL,
    reason          TEXT,
    created_by      VARCHAR(100),
    expires_at      TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(identifier, identifier_type),
    INDEX idx_blacklist (identifier, identifier_type),
    INDEX idx_expires (expires_at)
);

-- Audit Log
CREATE TABLE rate_limit_audit_log (
    id              BIGSERIAL PRIMARY KEY,
    client_id       VARCHAR(255) NOT NULL,
    rule_id         VARCHAR(100),
    action          VARCHAR(50) NOT NULL,  -- allowed, rejected, throttled
    request_count   INT,
    limit_count     INT,
    endpoint        VARCHAR(500),
    ip_address      VARCHAR(45),
    timestamp       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_client_time (client_id, timestamp),
    INDEX idx_action_time (action, timestamp)
);
```

### 6.2 Redis Data Structures

```
# Counter Keys (Sliding Window Log)
Key: ratelimit:counter:{client_id}:{rule_id}:{window_start}
Type: STRING (integer)
TTL: window_seconds + buffer

Example:
SET ratelimit:counter:user_123:api_default:1705312800 "150"
EXPIRE ratelimit:counter:user_123:api_default:1705312800 3660

# Sliding Window Counter (Sorted Set)
Key: ratelimit:sliding:{client_id}:{rule_id}
Type: SORTED SET
Score: request timestamp (milliseconds)
Member: unique request ID

Example:
ZADD ratelimit:sliding:user_123:api_default 1705314523000 "req_abc123"
ZREMRANGEBYSCORE ratelimit:sliding:user_123:api_default 0 1705310923000

# Token Bucket State
Key: ratelimit:bucket:{client_id}:{rule_id}
Type: HASH
Fields: tokens, last_refill

Example:
HSET ratelimit:bucket:user_123:api_default tokens 95 last_refill 1705314500

# Rule Cache
Key: ratelimit:rules:active
Type: HASH
Field: rule_id
Value: serialized rule JSON

# Whitelist Cache
Key: ratelimit:whitelist
Type: SET
Members: identifier strings

# Blacklist Cache
Key: ratelimit:blacklist
Type: HASH
Field: identifier
Value: expiry timestamp
```

---

## 7. Deep Dive: Rate Limiting Algorithms

### 7.1 Algorithm Comparison

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    RATE LIMITING ALGORITHM COMPARISON                            │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. FIXED WINDOW COUNTER                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Window 1 (12:00-12:01)     Window 2 (12:01-12:02)                     │    │
│  │  ████████████░░░░░░░░░░    ░░░░░░░░░░░░░░░░░░░░░                      │    │
│  │  Requests: 80/100          Requests: 0/100                             │    │
│  │                                                                         │    │
│  │  Pros: Simple, memory efficient                                        │    │
│  │  Cons: Burst at window boundary (2x limit possible)                    │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  2. SLIDING WINDOW LOG                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Stores timestamp of each request                                       │    │
│  │  |-----|-----|-----|-----|-----|-----|-----|                           │    │
│  │  12:00 12:01 12:02 12:03 12:04 12:05 12:06                             │    │
│  │    ▲     ▲     ▲▲    ▲     ▲▲▲   ▲                                    │    │
│  │  Count requests in sliding 1-minute window                              │    │
│  │                                                                         │    │
│  │  Pros: Accurate, no boundary issues                                    │    │
│  │  Cons: Memory intensive (stores each request)                          │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  3. SLIDING WINDOW COUNTER (Hybrid)                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Combines fixed window counter with sliding calculation                 │    │
│  │                                                                         │    │
│  │  Previous Window: 84 requests                                          │    │
│  │  Current Window: 36 requests                                           │    │
│  │  Window progress: 25% into current                                     │    │
│  │                                                                         │    │
│  │  Estimated count = (84 × 0.75) + 36 = 99 requests                     │    │
│  │                                                                         │    │
│  │  Pros: Memory efficient, reasonably accurate                           │    │
│  │  Cons: Approximate (not exact)                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  4. TOKEN BUCKET                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Bucket Capacity: 100 tokens                                           │    │
│  │  Refill Rate: 10 tokens/second                                         │    │
│  │                                                                         │    │
│  │  ┌─────────────┐                                                       │    │
│  │  │ ○○○○○○○○○○ │ ← Tokens refill over time                            │    │
│  │  │ ○○○○○○○○○○ │                                                       │    │
│  │  │ ○○○○○○○○   │ ← Each request consumes a token                      │    │
│  │  └─────────────┘                                                       │    │
│  │                                                                         │    │
│  │  Pros: Allows bursts, smooth rate limiting                             │    │
│  │  Cons: Slightly more complex                                           │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  5. LEAKY BUCKET                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Requests enter bucket → Processed at fixed rate                       │    │
│  │                                                                         │    │
│  │  Incoming ──▶ ┌─────────────┐                                          │    │
│  │  Requests     │ ○○○○○○○○○○ │ ← Queue                                  │    │
│  │               │ ○○○○○○○○○○ │                                          │    │
│  │               └──────┬──────┘                                          │    │
│  │                      │ ← Fixed outflow rate                            │    │
│  │                      ▼                                                  │    │
│  │                   Process                                               │    │
│  │                                                                         │    │
│  │  Pros: Smooth output rate, prevents bursts                             │    │
│  │  Cons: Delays requests (not suitable for real-time)                    │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Sliding Window Counter Implementation (Recommended)

```python
import time
import redis
from typing import Tuple

class SlidingWindowRateLimiter:
    """
    Sliding window counter rate limiter using Redis.
    Combines accuracy of sliding window with efficiency of counters.
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def is_allowed(
        self,
        client_id: str,
        rule_id: str,
        limit: int,
        window_seconds: int
    ) -> Tuple[bool, dict]:
        """
        Check if request is allowed and increment counter.
        Returns (is_allowed, rate_limit_info)
        """
        now = time.time()
        window_start = int(now // window_seconds) * window_seconds
        previous_window = window_start - window_seconds

        # Keys for current and previous windows
        current_key = f"ratelimit:{client_id}:{rule_id}:{window_start}"
        previous_key = f"ratelimit:{client_id}:{rule_id}:{previous_window}"

        # Lua script for atomic operation
        lua_script = """
        local current_key = KEYS[1]
        local previous_key = KEYS[2]
        local limit = tonumber(ARGV[1])
        local window_seconds = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        local window_start = tonumber(ARGV[4])

        -- Get counts
        local previous_count = tonumber(redis.call('GET', previous_key) or '0')
        local current_count = tonumber(redis.call('GET', current_key) or '0')

        -- Calculate weighted count (sliding window approximation)
        local elapsed = now - window_start
        local previous_weight = 1 - (elapsed / window_seconds)
        local weighted_count = (previous_count * previous_weight) + current_count

        -- Check if allowed
        if weighted_count >= limit then
            return {0, current_count, previous_count, weighted_count}
        end

        -- Increment current window
        redis.call('INCR', current_key)
        redis.call('EXPIRE', current_key, window_seconds * 2)

        return {1, current_count + 1, previous_count, weighted_count + 1}
        """

        result = self.redis.eval(
            lua_script,
            2,  # number of keys
            current_key,
            previous_key,
            limit,
            window_seconds,
            now,
            window_start
        )

        is_allowed = bool(result[0])
        current_count = int(result[1])
        previous_count = int(result[2])
        weighted_count = float(result[3])

        # Calculate remaining and reset time
        remaining = max(0, limit - int(weighted_count))
        reset_at = window_start + window_seconds

        return is_allowed, {
            'limit': limit,
            'remaining': remaining,
            'reset_at': reset_at,
            'current_count': current_count,
            'weighted_count': weighted_count
        }


class TokenBucketRateLimiter:
    """
    Token bucket rate limiter using Redis.
    Allows bursts while maintaining average rate.
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def is_allowed(
        self,
        client_id: str,
        rule_id: str,
        bucket_capacity: int,
        refill_rate: float,  # tokens per second
        tokens_requested: int = 1
    ) -> Tuple[bool, dict]:
        """
        Check if request is allowed using token bucket algorithm.
        """
        key = f"ratelimit:bucket:{client_id}:{rule_id}"
        now = time.time()

        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])

        -- Get current state
        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or capacity
        local last_refill = tonumber(bucket[2]) or now

        -- Calculate token refill
        local elapsed = now - last_refill
        local refill = elapsed * refill_rate
        tokens = math.min(capacity, tokens + refill)

        -- Check if enough tokens
        if tokens < requested then
            -- Update state (even on rejection for accurate tracking)
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)
            return {0, tokens, capacity}
        end

        -- Consume tokens
        tokens = tokens - requested
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)

        return {1, tokens, capacity}
        """

        result = self.redis.eval(
            lua_script,
            1,
            key,
            bucket_capacity,
            refill_rate,
            now,
            tokens_requested
        )

        is_allowed = bool(result[0])
        remaining_tokens = float(result[1])
        capacity = int(result[2])

        # Calculate time until bucket is full
        if remaining_tokens < capacity:
            time_to_full = (capacity - remaining_tokens) / refill_rate
        else:
            time_to_full = 0

        return is_allowed, {
            'capacity': capacity,
            'remaining': int(remaining_tokens),
            'refill_rate': refill_rate,
            'time_to_full': time_to_full
        }
```

### 7.3 Rate Limiter Middleware

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
import time

class RateLimiterMiddleware:
    def __init__(self, app: FastAPI, rate_limiter, config_service):
        self.app = app
        self.rate_limiter = rate_limiter
        self.config = config_service

    async def __call__(self, request: Request, call_next):
        # Extract client identifier
        client_id = self._get_client_id(request)

        # Check whitelist
        if await self.config.is_whitelisted(client_id):
            return await call_next(request)

        # Check blacklist
        if await self.config.is_blacklisted(client_id):
            return JSONResponse(
                status_code=403,
                content={"error": "Access denied"}
            )

        # Get applicable rules
        rules = await self.config.get_rules_for_request(
            endpoint=request.url.path,
            method=request.method,
            client_id=client_id
        )

        # Check each rule
        headers = {}
        for rule in sorted(rules, key=lambda r: r['priority'], reverse=True):
            is_allowed, info = self.rate_limiter.is_allowed(
                client_id=client_id,
                rule_id=rule['rule_id'],
                limit=rule['limit_count'],
                window_seconds=rule['window_seconds']
            )

            # Set headers for the most restrictive rule
            if 'X-RateLimit-Limit' not in headers:
                headers['X-RateLimit-Limit'] = str(info['limit'])
                headers['X-RateLimit-Remaining'] = str(info['remaining'])
                headers['X-RateLimit-Reset'] = str(int(info['reset_at']))

            if not is_allowed:
                retry_after = int(info['reset_at'] - time.time())
                headers['Retry-After'] = str(max(1, retry_after))

                # Log for monitoring
                await self._log_rejection(client_id, rule, request)

                return JSONResponse(
                    status_code=429,
                    headers=headers,
                    content={
                        "error": {
                            "code": "RATE_LIMIT_EXCEEDED",
                            "message": f"Rate limit exceeded for {rule['name']}",
                            "retry_after": retry_after
                        }
                    }
                )

        # Request allowed - proceed
        response = await call_next(request)

        # Add rate limit headers to response
        for key, value in headers.items():
            response.headers[key] = value

        return response

    def _get_client_id(self, request: Request) -> str:
        """Extract client identifier from request"""
        # Priority: API Key > User ID > IP
        api_key = request.headers.get('X-API-Key')
        if api_key:
            return f"api_key:{api_key}"

        auth = request.headers.get('Authorization', '')
        if auth.startswith('Bearer '):
            # Decode JWT to get user ID (simplified)
            user_id = self._decode_user_id(auth[7:])
            if user_id:
                return f"user:{user_id}"

        # Fallback to IP
        ip = request.client.host
        forwarded = request.headers.get('X-Forwarded-For')
        if forwarded:
            ip = forwarded.split(',')[0].strip()

        return f"ip:{ip}"
```

---

## 8. Scaling Discussion

### 8.1 Redis Cluster Scaling

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         REDIS CLUSTER TOPOLOGY                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Initial Setup (< 1M requests/sec):                                            │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                     3 Masters + 3 Replicas                              │   │
│   │                                                                         │   │
│   │   ┌──────────┐        ┌──────────┐        ┌──────────┐                 │   │
│   │   │ Master 1 │        │ Master 2 │        │ Master 3 │                 │   │
│   │   │ 2GB RAM  │        │ 2GB RAM  │        │ 2GB RAM  │                 │   │
│   │   └────┬─────┘        └────┬─────┘        └────┬─────┘                 │   │
│   │        │                   │                   │                        │   │
│   │   ┌────┴─────┐        ┌────┴─────┐        ┌────┴─────┐                 │   │
│   │   │ Replica  │        │ Replica  │        │ Replica  │                 │   │
│   │   └──────────┘        └──────────┘        └──────────┘                 │   │
│   │                                                                         │   │
│   │   Total: 6GB usable, ~100K ops/sec per node = 300K ops/sec            │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   Scaled Setup (> 1M requests/sec):                                             │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                     9 Masters + 9 Replicas                              │   │
│   │                                                                         │   │
│   │   Hash slots redistributed across more nodes                           │   │
│   │   Total: 18GB usable, ~900K ops/sec                                    │   │
│   │                                                                         │   │
│   │   With read replicas: ~1.8M reads/sec                                  │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Multi-Region Deployment

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         MULTI-REGION ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Option 1: Region-Local Rate Limiting                                          │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                         │   │
│   │   US-EAST                    EU-WEST                   APAC            │   │
│   │   ┌─────────────┐           ┌─────────────┐          ┌─────────────┐   │   │
│   │   │ Rate Limit  │           │ Rate Limit  │          │ Rate Limit  │   │   │
│   │   │ (Local)     │           │ (Local)     │          │ (Local)     │   │   │
│   │   │             │           │             │          │             │   │   │
│   │   │ Redis       │           │ Redis       │          │ Redis       │   │   │
│   │   └─────────────┘           └─────────────┘          └─────────────┘   │   │
│   │                                                                         │   │
│   │   Pros: Low latency, region isolation                                  │   │
│   │   Cons: User can get 3x limit by hitting all regions                   │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   Option 2: Global Rate Limiting with Local Cache                               │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                         │   │
│   │                    ┌─────────────────┐                                 │   │
│   │                    │  Global Redis   │                                 │   │
│   │                    │  (US-EAST)      │                                 │   │
│   │                    └────────┬────────┘                                 │   │
│   │                             │                                          │   │
│   │          ┌──────────────────┼──────────────────┐                      │   │
│   │          │                  │                  │                       │   │
│   │   ┌──────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐              │   │
│   │   │ Local Cache │    │ Local Cache │    │ Local Cache │              │   │
│   │   │ (US-EAST)   │    │ (EU-WEST)   │    │ (APAC)      │              │   │
│   │   └─────────────┘    └─────────────┘    └─────────────┘              │   │
│   │                                                                         │   │
│   │   - Sync to global every N requests or T seconds                       │   │
│   │   - Local allows "soft" limit, global enforces "hard" limit           │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 8.3 Handling Failures

```python
class ResilientRateLimiter:
    """
    Rate limiter with fallback strategies for Redis failures.
    """

    def __init__(self, redis_client, local_cache_size=10000):
        self.redis = redis_client
        self.local_cache = LRUCache(local_cache_size)
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=30
        )

    def is_allowed(self, client_id, rule_id, limit, window_seconds):
        """
        Check rate limit with fallback strategies.
        """
        try:
            if self.circuit_breaker.is_open():
                return self._local_fallback(client_id, rule_id, limit)

            result = self._check_redis(client_id, rule_id, limit, window_seconds)
            self.circuit_breaker.record_success()
            return result

        except redis.RedisError as e:
            self.circuit_breaker.record_failure()
            logging.warning(f"Redis error, using fallback: {e}")
            return self._local_fallback(client_id, rule_id, limit)

    def _local_fallback(self, client_id, rule_id, limit):
        """
        Local in-memory rate limiting as fallback.
        More permissive to avoid false rejections.
        """
        key = f"{client_id}:{rule_id}"

        # Use local LRU cache with 2x limit (more permissive)
        count = self.local_cache.get(key, 0)

        if count >= limit * 2:  # 2x limit as buffer
            return False, {'remaining': 0, 'fallback': True}

        self.local_cache.set(key, count + 1, ttl=60)
        return True, {'remaining': limit * 2 - count - 1, 'fallback': True}


class CircuitBreaker:
    """
    Circuit breaker pattern for Redis connection.
    """

    def __init__(self, failure_threshold, recovery_timeout):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = 'closed'  # closed, open, half-open

    def is_open(self):
        if self.state == 'closed':
            return False

        if self.state == 'open':
            # Check if recovery timeout has passed
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = 'half-open'
                return False
            return True

        return False  # half-open allows one request

    def record_success(self):
        self.failures = 0
        self.state = 'closed'

    def record_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()

        if self.failures >= self.failure_threshold:
            self.state = 'open'
```

---

## 9. Trade-offs

### 9.1 Algorithm Selection

| Algorithm | Accuracy | Memory | Burst Handling | Use Case |
|-----------|----------|--------|----------------|----------|
| **Fixed Window** | Low | Very Low | Poor | Simple APIs, non-critical |
| **Sliding Window Log** | High | High | Good | Financial APIs, audit required |
| **Sliding Window Counter** | Medium | Low | Good | Most APIs (recommended) |
| **Token Bucket** | High | Low | Excellent | APIs that allow bursts |
| **Leaky Bucket** | High | Medium | None | Strict rate enforcement |

### 9.2 Centralized vs Distributed

| Approach | Pros | Cons |
|----------|------|------|
| **Centralized (Redis)** | Accurate, consistent | Network latency, SPOF risk |
| **Distributed (Local)** | Fast, no network hop | Inaccurate, client can game it |
| **Hybrid** | Balance of both | Complexity |

### 9.3 Consistency vs Performance

| Choice | Trade-off |
|--------|-----------|
| **Strong consistency** | Every request checks Redis (higher latency) |
| **Eventual consistency** | Local batching, periodic sync (may allow slightly over limit) |
| **Best-effort** | Allow some over-limit (use for non-critical limits) |

---

## 10. Failure Scenarios

### 10.1 Redis Cluster Failure

```
Scenario: All Redis masters become unavailable

Impact:
- Rate limiting stops working
- All requests could be allowed (or all blocked)

Mitigation:
1. Circuit breaker detects failure
2. Fall back to local in-memory rate limiting
3. Use more permissive limits during fallback
4. Alert on-call team

Recovery:
1. Redis auto-failover promotes replicas
2. Circuit breaker enters half-open state
3. Gradual recovery to normal operation
4. Verify counter consistency
```

### 10.2 Configuration Service Failure

```
Scenario: Cannot fetch rate limit rules

Impact:
- New rules not applied
- Rule changes not propagated

Mitigation:
1. Cache rules locally with long TTL
2. Use last known good configuration
3. Default rules as fallback
4. Alert on stale configuration

Best Practice:
- Keep default rules in code
- Cache rules aggressively
- Async rule updates (don't block requests)
```

### 10.3 Clock Skew Across Servers

```
Scenario: Servers have different times

Impact:
- Window calculations inconsistent
- Client may get extra/fewer requests

Mitigation:
1. Use NTP to sync all servers
2. Use Redis server time (not local)
3. Add small buffer to windows
4. Monitor for clock drift
```

---

## 11. Interview Variations

### 11.1 Common Follow-up Questions

**Q1: How would you handle rate limiting for a multi-tenant SaaS?**
```
Answer:
1. Hierarchical limits:
   - Tenant-level limit (total for organization)
   - User-level limit (per user within tenant)
   - API key limit (per integration)

2. Composite keys:
   ratelimit:{tenant_id}:global
   ratelimit:{tenant_id}:{user_id}
   ratelimit:{tenant_id}:{api_key}

3. Check all levels, most restrictive wins
4. Allow tenant admins to configure sub-limits
```

**Q2: How would you implement graduated rate limiting?**
```
Answer:
1. Define tiers (free, basic, pro, enterprise)
2. Store tier in user profile
3. Load tier-specific rules at request time
4. Example:
   - Free: 100 requests/hour
   - Basic: 1000 requests/hour
   - Pro: 10000 requests/hour
   - Enterprise: Custom/unlimited
```

**Q3: How would you rate limit expensive operations differently?**
```
Answer:
1. Assign "cost" to each endpoint
   - GET /users: cost = 1
   - POST /reports: cost = 10
   - POST /export: cost = 100

2. Token bucket with weighted consumption
3. Deduct cost instead of 1 per request
4. Users get X "credits" per time window
```

**Q4: How would you implement rate limiting without Redis?**
```
Answer:
1. In-memory with sticky sessions (load balancer)
   - Route same client to same server
   - Local counters only

2. Database-based (PostgreSQL)
   - Use SELECT FOR UPDATE
   - Higher latency but simpler

3. Application-level only
   - Per-process limits
   - Good for single-server apps
```

### 11.2 Design Variations

**Variation 1: Rate Limiting at Network Level (API Gateway)**
```
Focus on:
- Integration with Kong, AWS API Gateway, Envoy
- Configuration via YAML/JSON
- Observability and metrics
- Request/response transformation
```

**Variation 2: DDoS-Resistant Rate Limiting**
```
Additional considerations:
- IP reputation scoring
- Captcha challenges
- JavaScript challenges
- Behavioral analysis
- Integration with CDN (Cloudflare, Akamai)
```

**Variation 3: Rate Limiting for Microservices**
```
Focus on:
- Service-to-service rate limiting
- Service mesh integration (Istio, Linkerd)
- Cascading failures prevention
- Bulkhead pattern
```

### 11.3 Red Flags to Avoid

| Mistake | Why It's Wrong |
|---------|----------------|
| No fallback strategy | Single point of failure |
| Checking DB for each request | Too slow, will become bottleneck |
| Ignoring distributed nature | Inaccurate counting |
| No monitoring/alerting | Can't detect failures |
| Blocking on Redis failure | Should fail open or use fallback |

### 11.4 Good Signs in Your Answer

| Indicator | Why It's Good |
|-----------|---------------|
| Discuss algorithm trade-offs | Shows depth of knowledge |
| Mention atomic operations | Understands race conditions |
| Consider failure modes | Production-ready thinking |
| Talk about headers (429, Retry-After) | API design awareness |
| Mention monitoring/metrics | Operational maturity |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        RATE LIMITER DESIGN SUMMARY                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Scale: 1M+ requests/second, 3M+ rate limit checks/second                      │
│                                                                                  │
│   Key Decisions:                                                                │
│   - Algorithm: Sliding Window Counter (accuracy + efficiency)                   │
│   - Storage: Redis Cluster (distributed, fast)                                  │
│   - Fallback: Local in-memory with circuit breaker                             │
│                                                                                  │
│   Implementation:                                                               │
│   - Lua scripts for atomic operations                                          │
│   - Middleware pattern for easy integration                                    │
│   - Standard headers (X-RateLimit-*)                                           │
│                                                                                  │
│   Key Numbers:                                                                  │
│   - Latency overhead: < 1ms                                                    │
│   - Memory per client: ~56 bytes per rule                                      │
│   - Redis cluster: 6+ nodes for HA                                             │
│                                                                                  │
│   Trade-offs:                                                                   │
│   - Accuracy vs Performance (sliding window counter is good balance)           │
│   - Centralized vs Distributed (centralized for accuracy)                      │
│   - Fail-open vs Fail-closed (usually fail-open with logging)                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```
