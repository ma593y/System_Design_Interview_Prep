# Resilience Patterns

## Overview

Resilience patterns help systems handle failures gracefully, maintaining availability and preventing cascading failures.

## Circuit Breaker

Prevent repeated calls to a failing service.

```
States:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│     ┌─────────┐    failures     ┌─────────┐               │
│     │ CLOSED  │────────────────▶│  OPEN   │               │
│     │(normal) │                 │ (fail)  │               │
│     └────┬────┘                 └────┬────┘               │
│          │                           │                     │
│          │ success                   │ timeout             │
│          │                           ▼                     │
│          │                     ┌───────────┐              │
│          └─────────────────────│HALF-OPEN │              │
│                     success    │  (test)   │              │
│                                └─────┬─────┘              │
│                                      │ failure            │
│                                      ▼                     │
│                                    OPEN                    │
└────────────────────────────────────────────────────────────┘
```

### Implementation

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = "CLOSED"
        self.last_failure_time = None

    def call(self, func):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError("Circuit is open")

        try:
            result = func()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e

    def on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

### Configuration

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| Failure threshold | Failures before opening | 5-10 |
| Recovery timeout | Time before half-open | 30-60 seconds |
| Half-open requests | Test requests allowed | 1-3 |

### When to Use

- Remote service calls
- Database connections
- External API calls
- Any operation that can fail and affect others

## Retry Pattern

Automatically retry failed operations.

```
Request ──▶ Failure ──▶ Wait ──▶ Retry ──▶ Success
                         │
                    Backoff time
```

### Retry Strategies

#### Immediate Retry

```
Attempt 1: Fail
Attempt 2: Fail
Attempt 3: Success

No delay between retries
```

#### Fixed Delay

```
Attempt 1: Fail → Wait 1s
Attempt 2: Fail → Wait 1s
Attempt 3: Success
```

#### Exponential Backoff

```
Attempt 1: Fail → Wait 1s
Attempt 2: Fail → Wait 2s
Attempt 3: Fail → Wait 4s
Attempt 4: Fail → Wait 8s
Attempt 5: Success

delay = base_delay * (2 ^ attempt)
```

#### Exponential Backoff with Jitter

```
Attempt 1: Fail → Wait 1s + random(0-500ms)
Attempt 2: Fail → Wait 2s + random(0-500ms)
Attempt 3: Success

Jitter prevents thundering herd
```

### Implementation

```python
def retry_with_backoff(func, max_retries=3, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except RetryableError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            jitter = random.uniform(0, delay * 0.1)
            time.sleep(delay + jitter)
```

### When to Retry

| Retry | Don't Retry |
|-------|-------------|
| Timeout | Authentication error |
| 503 Service Unavailable | 400 Bad Request |
| Connection refused | 404 Not Found |
| Rate limited (429) | Business logic error |

## Bulkhead Pattern

Isolate failures to prevent cascading.

```
Without Bulkhead:
┌─────────────────────────────────────────┐
│            Thread Pool (100)             │
│                                          │
│  Service A: 50 threads (stuck)          │
│  Service B: 40 threads (stuck)          │
│  Service C: 10 threads (stuck)          │
│                                          │
│  All services affected!                 │
└─────────────────────────────────────────┘

With Bulkhead:
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Service A Pool  │ │ Service B Pool  │ │ Service C Pool  │
│    (50)         │ │    (30)         │ │    (20)         │
│                 │ │                 │ │                 │
│  50 stuck       │ │  30 working     │ │  20 working     │
│                 │ │                 │ │                 │
│ Service A fails │ │ B unaffected    │ │ C unaffected    │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### Types of Bulkheads

#### Thread Pool Isolation

```
┌──────────────────────────────────────────────────┐
│                  Application                      │
│                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Pool: API-A │  │ Pool: API-B │  │Pool: DB  │ │
│  │ (10 threads)│  │ (10 threads)│  │(5 threads)│ │
│  └─────────────┘  └─────────────┘  └──────────┘ │
│                                                   │
└──────────────────────────────────────────────────┘
```

#### Semaphore Isolation

```
Limit concurrent calls without dedicated threads:

API-A semaphore: max 10 concurrent
API-B semaphore: max 10 concurrent

Lighter weight than thread pools
```

### When to Use

- Isolate external service calls
- Protect critical resources
- Different SLAs for different operations

## Timeout Pattern

Fail fast when operations take too long.

```
Request ──▶ Service ──── Processing... ────
     │                         │
     │      Timeout!           │
     │◀── Return error ────────┘
     │
     ▼
  Handle timeout
  (retry or fail)
```

### Timeout Strategies

```
Connect Timeout: Time to establish connection (1-5 seconds)
Read Timeout: Time to receive response (5-30 seconds)
Total Timeout: Overall operation time (including retries)

Example:
┌────────────────────────────────────────────────────┐
│                Total Timeout: 10s                   │
│                                                     │
│  Connect    Read      Retry     Connect    Read    │
│   [2s]     [3s]       [1s]       [2s]     [2s]    │
│                                                     │
└────────────────────────────────────────────────────┘
```

### Timeouts with Circuit Breaker

```python
def call_with_resilience(func, timeout=5):
    if circuit_breaker.is_open():
        raise CircuitOpenError()

    try:
        with timeout(timeout):
            result = func()
            circuit_breaker.record_success()
            return result
    except TimeoutError:
        circuit_breaker.record_failure()
        raise
```

## Fallback Pattern

Provide alternative when primary fails.

```
Request ──▶ Primary Service ──▶ Failure
                                   │
                                   ▼
                            ┌─────────────┐
                            │  Fallback   │
                            │  Strategy   │
                            └──────┬──────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              ┌──────────┐  ┌──────────┐  ┌──────────┐
              │  Cache   │  │ Default  │  │ Degraded │
              │  Data    │  │  Value   │  │ Service  │
              └──────────┘  └──────────┘  └──────────┘
```

### Fallback Strategies

| Strategy | Description | Example |
|----------|-------------|---------|
| **Cached** | Return cached data | Last known price |
| **Default** | Return static default | Empty list, default config |
| **Degraded** | Reduced functionality | No recommendations |
| **Alternative** | Different service | Backup provider |

### Implementation

```python
def get_product_price(product_id):
    try:
        return pricing_service.get_price(product_id)
    except ServiceUnavailable:
        # Fallback 1: Cache
        cached = cache.get(f"price:{product_id}")
        if cached:
            return cached

        # Fallback 2: Default
        return default_prices.get(product_id, 0)
```

## Rate Limiter

Protect services from overload.

```
Requests ──▶ Rate Limiter ──▶ Service
                  │
                  │ Over limit?
                  ▼
              429 Too Many Requests
```

### Algorithms

#### Token Bucket

```
┌────────────────────────────────────────┐
│         Token Bucket                    │
│                                         │
│    Capacity: 10 tokens                  │
│    Refill: 2 tokens/second              │
│                                         │
│    [●][●][●][●][○][○][○][○][○][○]      │
│     4 tokens available                  │
│                                         │
│    Request uses 1 token                 │
│    If no tokens → reject                │
└────────────────────────────────────────┘
```

#### Sliding Window

```
Window: 60 seconds
Limit: 100 requests

Timeline:
|──────────────────60s───────────────────|
[r1][r2][r3]...                    [r98][r99][r100]
                                              ▲
                                        Next request
                                        rejected
```

## Health Checks

Detect and report service health.

```
┌─────────────────────────────────────────────────────┐
│                  Load Balancer                       │
│                                                      │
│  Health Check: GET /health every 10s                │
└───────────────────────┬─────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ Server1 │    │ Server2 │    │ Server3 │
   │  ✓ 200  │    │  ✓ 200  │    │  ✗ 500  │
   │ Healthy │    │ Healthy │    │Unhealthy│
   └─────────┘    └─────────┘    └─────────┘
        │               │              │
        └───────────────┼──────────────┘
                        │
              Traffic to healthy only
```

### Health Check Types

| Type | Checks |
|------|--------|
| **Liveness** | Is process running? |
| **Readiness** | Can handle traffic? |
| **Dependency** | Are dependencies OK? |

## Combining Patterns

```
Request
   │
   ▼
┌─────────────────────────────────────────────────────┐
│                  Rate Limiter                        │
│              (prevent overload)                      │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                  Circuit Breaker                     │
│              (fail fast if broken)                   │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                  Bulkhead                            │
│              (isolate failures)                      │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                  Retry + Timeout                     │
│              (handle transient failures)             │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                  Fallback                            │
│              (graceful degradation)                  │
└─────────────────────────────────────────────────────┘
```

## Interview Talking Points

1. "Circuit breaker prevents cascading failures"
2. "Exponential backoff with jitter prevents thundering herd"
3. "Bulkheads isolate failures to specific pools"
4. "Combine patterns for defense in depth"
5. "Always have a fallback strategy"

## Common Interview Questions

1. **Q: What is a circuit breaker and when would you use it?**
   A: Prevents repeated calls to failing service. Use for any remote call that can fail and affect system stability.

2. **Q: How do you implement retries without causing thundering herd?**
   A: Exponential backoff with random jitter. Each client waits different time, spreading load.

3. **Q: What's the difference between timeout and circuit breaker?**
   A: Timeout fails individual requests quickly. Circuit breaker fails entire categories of requests when service is consistently failing.

4. **Q: How do resilience patterns work together?**
   A: Rate limiter → Circuit breaker → Bulkhead → Retry/Timeout → Fallback. Each layer provides different protection.

## Key Takeaways

- Build resilience with multiple patterns
- Fail fast with timeouts and circuit breakers
- Isolate failures with bulkheads
- Provide graceful degradation with fallbacks
- Monitor and tune resilience parameters

## Further Reading

- "Release It!" by Michael Nygard
- Hystrix documentation (Netflix)
- Resilience4j documentation
- AWS Well-Architected Framework - Reliability
