# API Best Practices

## Overview

APIs (Application Programming Interfaces) are the contracts that define how different systems communicate. Well-designed APIs are crucial for building scalable, maintainable, and developer-friendly systems. This guide covers essential best practices for designing, implementing, and maintaining robust APIs.

## Table of Contents

1. [API Versioning](#api-versioning)
2. [Pagination](#pagination)
3. [Error Handling](#error-handling)
4. [Documentation](#documentation)
5. [Rate Limiting](#rate-limiting)
6. [Request/Response Design](#requestresponse-design)
7. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
8. [Implementation Guidelines](#implementation-guidelines)
9. [Interview Talking Points](#interview-talking-points)
10. [Key Takeaways](#key-takeaways)

---

## API Versioning

### Why Version APIs?

- **Backward Compatibility**: Allow existing clients to continue working while new features are added
- **Controlled Migration**: Give clients time to upgrade to new versions
- **Clear Communication**: Signal breaking changes explicitly

### Versioning Strategies

#### 1. URL Path Versioning (Recommended)

```
GET /api/v1/users
GET /api/v2/users
```

**Pros**:
- Clear and explicit
- Easy to route at load balancer level
- Simple to cache
- Easy to test and debug

**Cons**:
- URL pollution
- Not truly RESTful (version is not a resource)

#### 2. Header Versioning

```http
GET /api/users
Accept: application/vnd.myapi.v1+json
```

**Pros**:
- Clean URLs
- More RESTful

**Cons**:
- Less discoverable
- Harder to test in browser
- Caching complexity

#### 3. Query Parameter Versioning

```
GET /api/users?version=1
```

**Pros**:
- Easy to implement
- Optional parameter

**Cons**:
- Can be forgotten by clients
- Caching issues with query strings

### Version Lifecycle Management

```
Version Timeline:
|--- v1 (Active) ------|--- v1 (Deprecated) ---|--- v1 (Sunset) ---|
                       |--- v2 (Active) --------|--- v2 (Deprecated) ---|
                                                |--- v3 (Active) --------|

Typical Timeline:
- Active: Full support and features
- Deprecated: 6-12 months warning, no new features
- Sunset: No longer available
```

### Best Practices for Versioning

```python
# Good: Explicit version in path
@app.route('/api/v1/users/<user_id>')
def get_user_v1(user_id):
    return UserSerializerV1(user).to_dict()

@app.route('/api/v2/users/<user_id>')
def get_user_v2(user_id):
    # V2 includes additional fields
    return UserSerializerV2(user).to_dict()

# Version routing middleware
class APIVersionMiddleware:
    def __init__(self, app):
        self.app = app
        self.version_handlers = {}

    def route_to_version(self, request):
        version = self.extract_version(request)
        if version not in self.supported_versions:
            raise UnsupportedVersionError(version)
        return self.version_handlers[version]
```

---

## Pagination

### Why Pagination Matters

- **Performance**: Prevents loading entire datasets into memory
- **Network Efficiency**: Reduces payload size
- **User Experience**: Faster initial load times
- **Resource Protection**: Prevents database overload

### Pagination Strategies

#### 1. Offset-Based Pagination

```http
GET /api/users?offset=20&limit=10
```

```json
{
  "data": [...],
  "pagination": {
    "offset": 20,
    "limit": 10,
    "total": 1000
  }
}
```

**Pros**:
- Simple to implement
- Can jump to any page
- Familiar to users

**Cons**:
- Performance degrades with large offsets (OFFSET 1000000)
- Inconsistent results if data changes between requests
- Not suitable for real-time data

```sql
-- Performance issue with large offset
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 1000000;
-- Database must scan 1,000,010 rows
```

#### 2. Cursor-Based Pagination (Recommended for Large Datasets)

```http
GET /api/users?cursor=eyJpZCI6MTAwfQ&limit=10
```

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTEwfQ",
    "prev_cursor": "eyJpZCI6MTAwfQ",
    "has_more": true
  }
}
```

**Pros**:
- Consistent performance regardless of position
- Stable results even with data changes
- Works well with real-time data

**Cons**:
- Cannot jump to arbitrary pages
- More complex implementation

```python
import base64
import json

def encode_cursor(data: dict) -> str:
    """Encode cursor data to base64 string."""
    return base64.urlsafe_b64encode(
        json.dumps(data).encode()
    ).decode()

def decode_cursor(cursor: str) -> dict:
    """Decode base64 cursor to dict."""
    return json.loads(
        base64.urlsafe_b64decode(cursor.encode())
    )

def get_users_with_cursor(cursor: str = None, limit: int = 10):
    query = User.query.order_by(User.id)

    if cursor:
        cursor_data = decode_cursor(cursor)
        query = query.filter(User.id > cursor_data['id'])

    users = query.limit(limit + 1).all()

    has_more = len(users) > limit
    users = users[:limit]

    next_cursor = None
    if has_more and users:
        next_cursor = encode_cursor({'id': users[-1].id})

    return {
        'data': [user.to_dict() for user in users],
        'pagination': {
            'next_cursor': next_cursor,
            'has_more': has_more
        }
    }
```

#### 3. Keyset Pagination

```http
GET /api/users?after_id=100&limit=10
```

```sql
-- Efficient keyset query
SELECT * FROM users
WHERE id > 100
ORDER BY id
LIMIT 10;
```

### Pagination Response Headers

```http
HTTP/1.1 200 OK
Link: <https://api.example.com/users?cursor=abc>; rel="next",
      <https://api.example.com/users?cursor=xyz>; rel="prev"
X-Total-Count: 1000
X-Page-Size: 10
```

---

## Error Handling

### HTTP Status Codes

```
2xx Success:
  200 OK              - Request succeeded
  201 Created         - Resource created
  202 Accepted        - Request accepted for processing
  204 No Content      - Success with no response body

4xx Client Errors:
  400 Bad Request     - Invalid request syntax
  401 Unauthorized    - Authentication required
  403 Forbidden       - Authenticated but not authorized
  404 Not Found       - Resource doesn't exist
  409 Conflict        - Resource conflict (duplicate)
  422 Unprocessable   - Valid syntax but semantic errors
  429 Too Many Reqs   - Rate limit exceeded

5xx Server Errors:
  500 Internal Error  - Generic server error
  502 Bad Gateway     - Upstream service error
  503 Unavailable     - Service temporarily unavailable
  504 Gateway Timeout - Upstream service timeout
```

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request contains invalid parameters",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Email must be a valid email address"
      },
      {
        "field": "age",
        "code": "OUT_OF_RANGE",
        "message": "Age must be between 0 and 150"
      }
    ],
    "request_id": "req_abc123",
    "documentation_url": "https://api.example.com/docs/errors#VALIDATION_ERROR"
  }
}
```

### Error Handling Implementation

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional

class ErrorCode(Enum):
    VALIDATION_ERROR = "VALIDATION_ERROR"
    NOT_FOUND = "NOT_FOUND"
    UNAUTHORIZED = "UNAUTHORIZED"
    FORBIDDEN = "FORBIDDEN"
    RATE_LIMITED = "RATE_LIMITED"
    INTERNAL_ERROR = "INTERNAL_ERROR"

@dataclass
class FieldError:
    field: str
    code: str
    message: str

@dataclass
class APIError(Exception):
    code: ErrorCode
    message: str
    status_code: int
    details: Optional[List[FieldError]] = None

    def to_dict(self):
        error_dict = {
            "error": {
                "code": self.code.value,
                "message": self.message,
                "request_id": get_request_id()
            }
        }
        if self.details:
            error_dict["error"]["details"] = [
                {"field": d.field, "code": d.code, "message": d.message}
                for d in self.details
            ]
        return error_dict

# Usage
def validate_user(data):
    errors = []
    if not is_valid_email(data.get('email')):
        errors.append(FieldError(
            field='email',
            code='INVALID_FORMAT',
            message='Email must be a valid email address'
        ))

    if errors:
        raise APIError(
            code=ErrorCode.VALIDATION_ERROR,
            message='The request contains invalid parameters',
            status_code=422,
            details=errors
        )
```

### Error Handling Best Practices

1. **Never expose internal errors to clients**

```python
# Bad
except Exception as e:
    return {"error": str(e)}, 500  # May expose sensitive info

# Good
except Exception as e:
    logger.error(f"Internal error: {e}", exc_info=True)
    return {
        "error": {
            "code": "INTERNAL_ERROR",
            "message": "An unexpected error occurred",
            "request_id": request_id
        }
    }, 500
```

2. **Provide actionable error messages**

```json
// Bad
{"error": "Invalid input"}

// Good
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The 'start_date' must be before 'end_date'",
    "details": [
      {
        "field": "start_date",
        "value": "2024-12-01",
        "constraint": "Must be before end_date (2024-11-01)"
      }
    ]
  }
}
```

---

## Documentation

### OpenAPI/Swagger Specification

```yaml
openapi: 3.0.0
info:
  title: User Management API
  version: 1.0.0
  description: API for managing user accounts

paths:
  /users:
    get:
      summary: List all users
      description: Returns a paginated list of users
      parameters:
        - name: cursor
          in: query
          description: Pagination cursor
          schema:
            type: string
        - name: limit
          in: query
          description: Number of results per page
          schema:
            type: integer
            default: 10
            maximum: 100
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        created_at:
          type: string
          format: date-time
      required:
        - id
        - email

    Error:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
            message:
              type: string
```

### Documentation Best Practices

1. **Include request/response examples**
2. **Document all error codes**
3. **Provide code samples in multiple languages**
4. **Keep documentation in sync with code**
5. **Include rate limit information**
6. **Document authentication requirements**

---

## Rate Limiting

### Rate Limiting Strategies

#### 1. Fixed Window

```python
class FixedWindowRateLimiter:
    def __init__(self, redis_client, limit: int, window_seconds: int):
        self.redis = redis_client
        self.limit = limit
        self.window = window_seconds

    def is_allowed(self, client_id: str) -> bool:
        key = f"rate_limit:{client_id}:{int(time.time() // self.window)}"
        current = self.redis.incr(key)

        if current == 1:
            self.redis.expire(key, self.window)

        return current <= self.limit
```

#### 2. Sliding Window Log

```python
class SlidingWindowRateLimiter:
    def __init__(self, redis_client, limit: int, window_seconds: int):
        self.redis = redis_client
        self.limit = limit
        self.window = window_seconds

    def is_allowed(self, client_id: str) -> bool:
        key = f"rate_limit:{client_id}"
        now = time.time()
        window_start = now - self.window

        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)
        pipe.zadd(key, {str(now): now})
        pipe.zcard(key)
        pipe.expire(key, self.window)
        results = pipe.execute()

        return results[2] <= self.limit
```

#### 3. Token Bucket

```python
class TokenBucketRateLimiter:
    def __init__(self, redis_client, capacity: int, refill_rate: float):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second

    def is_allowed(self, client_id: str, tokens: int = 1) -> bool:
        key = f"token_bucket:{client_id}"
        now = time.time()

        bucket = self.redis.hgetall(key)

        if not bucket:
            new_tokens = self.capacity - tokens
            self.redis.hset(key, mapping={
                'tokens': new_tokens,
                'last_update': now
            })
            return True

        last_update = float(bucket['last_update'])
        current_tokens = float(bucket['tokens'])

        # Refill tokens
        elapsed = now - last_update
        current_tokens = min(
            self.capacity,
            current_tokens + elapsed * self.refill_rate
        )

        if current_tokens >= tokens:
            current_tokens -= tokens
            self.redis.hset(key, mapping={
                'tokens': current_tokens,
                'last_update': now
            })
            return True

        return False
```

### Rate Limit Response Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
Retry-After: 60
```

---

## Request/Response Design

### Consistent Response Structure

```json
// Success response
{
  "data": {
    "id": "user_123",
    "email": "user@example.com"
  },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}

// Collection response
{
  "data": [
    {"id": "user_123", "email": "user1@example.com"},
    {"id": "user_456", "email": "user2@example.com"}
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6NDU2fQ",
    "has_more": true
  },
  "meta": {
    "request_id": "req_abc123",
    "total_count": 1000
  }
}
```

### Idempotency

```python
class IdempotencyMiddleware:
    def __init__(self, redis_client, ttl: int = 86400):
        self.redis = redis_client
        self.ttl = ttl

    def process_request(self, request):
        idempotency_key = request.headers.get('Idempotency-Key')

        if not idempotency_key:
            return None  # Process normally

        key = f"idempotency:{idempotency_key}"
        cached = self.redis.get(key)

        if cached:
            return json.loads(cached)

        return None

    def store_response(self, idempotency_key: str, response: dict):
        key = f"idempotency:{idempotency_key}"
        self.redis.setex(key, self.ttl, json.dumps(response))
```

### Field Selection (Sparse Fieldsets)

```http
GET /api/users?fields=id,email,name
```

```python
def get_users(fields: str = None):
    users = User.query.all()

    if fields:
        allowed_fields = {'id', 'email', 'name', 'created_at'}
        requested_fields = set(fields.split(',')) & allowed_fields
        return [
            {k: getattr(user, k) for k in requested_fields}
            for user in users
        ]

    return [user.to_dict() for user in users]
```

---

## Common Mistakes to Avoid

### 1. Breaking Changes Without Versioning

```python
# Bad: Changing response structure without version bump
# Before: {"user_name": "John"}
# After:  {"name": "John"}  # Breaking change!

# Good: Add new field, deprecate old
# {"user_name": "John", "name": "John"}  # Both fields
# Then remove in next major version
```

### 2. Exposing Internal IDs

```python
# Bad: Exposing auto-increment database IDs
{"id": 12345}  # Reveals user count, sequential access

# Good: Use UUIDs or encoded IDs
{"id": "usr_a1b2c3d4e5f6"}
```

### 3. Inconsistent Naming

```python
# Bad: Mixed conventions
{
    "user_name": "John",
    "userEmail": "john@example.com",
    "USER-ID": "123"
}

# Good: Consistent snake_case
{
    "user_name": "John",
    "user_email": "john@example.com",
    "user_id": "usr_123"
}
```

### 4. No Request Validation

```python
# Bad: Trust all input
def create_user(data):
    user = User(**data)  # Dangerous!
    db.session.add(user)

# Good: Validate and sanitize
def create_user(data):
    validated = UserSchema().load(data)  # Validates
    user = User(**validated)
    db.session.add(user)
```

### 5. Chatty APIs

```python
# Bad: Multiple round trips needed
GET /users/123
GET /users/123/orders
GET /users/123/addresses

# Good: Allow embedding related resources
GET /users/123?include=orders,addresses
```

### 6. Ignoring Partial Failures

```python
# Bad: All or nothing batch operations
POST /users/batch
# Returns 500 if any user fails

# Good: Report individual results
POST /users/batch
{
    "results": [
        {"index": 0, "status": "success", "data": {...}},
        {"index": 1, "status": "error", "error": {...}},
        {"index": 2, "status": "success", "data": {...}}
    ],
    "summary": {
        "total": 3,
        "successful": 2,
        "failed": 1
    }
}
```

---

## Implementation Guidelines

### API Design Checklist

```markdown
[ ] Authentication & Authorization
    [ ] API key or OAuth 2.0 implemented
    [ ] HTTPS enforced
    [ ] Token expiration handled

[ ] Versioning
    [ ] Version strategy chosen (URL path recommended)
    [ ] Deprecation policy documented
    [ ] Version headers included in responses

[ ] Request Handling
    [ ] Input validation on all endpoints
    [ ] Request size limits configured
    [ ] Content-Type validation
    [ ] Idempotency keys for mutations

[ ] Response Design
    [ ] Consistent response structure
    [ ] Proper HTTP status codes
    [ ] Error responses include codes and messages
    [ ] Pagination for collections

[ ] Performance
    [ ] Rate limiting implemented
    [ ] Response compression (gzip)
    [ ] Caching headers set appropriately
    [ ] Connection timeouts configured

[ ] Documentation
    [ ] OpenAPI/Swagger spec maintained
    [ ] Examples for all endpoints
    [ ] Error codes documented
    [ ] Changelog maintained
```

### API Gateway Configuration

```yaml
# Kong API Gateway example
services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: users-route
        paths:
          - /api/v1/users
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
      - name: request-transformer
        config:
          add:
            headers:
              - X-Request-ID:$(uuid)
      - name: response-transformer
        config:
          add:
            headers:
              - X-Response-Time:$(latency)
```

---

## Interview Talking Points

### Question: "How would you design API versioning for a large-scale system?"

**Strong Answer Framework**:

1. **Choose URL path versioning** for clarity and routing simplicity
2. **Implement version lifecycle**: Active -> Deprecated -> Sunset
3. **Use feature flags** for gradual rollouts within versions
4. **Maintain backward compatibility** within major versions
5. **Document breaking changes** and migration guides

### Question: "How do you handle pagination for a billion-row table?"

**Strong Answer Framework**:

1. **Use cursor-based pagination** for consistent performance
2. **Create covering indexes** on sort columns
3. **Avoid COUNT queries** for total; use estimates
4. **Implement keyset pagination** with composite keys
5. **Cache first page** results for common queries

### Question: "What's your approach to API error handling?"

**Strong Answer Framework**:

1. **Structured error format** with code, message, details
2. **Appropriate HTTP status codes** for different error types
3. **Request IDs** for debugging and support
4. **Never expose internal errors** - log them instead
5. **Actionable messages** telling clients how to fix issues

### Common Follow-up Questions

- How do you handle API authentication at scale?
- What's your strategy for API deprecation?
- How do you ensure API consistency across teams?
- How do you test APIs for backward compatibility?

---

## Key Takeaways

### 1. Version from Day One
- Plan for change; APIs will evolve
- URL path versioning is most practical
- Have clear deprecation policies

### 2. Pagination is Critical
- Never return unbounded collections
- Cursor-based pagination for large datasets
- Include pagination metadata in responses

### 3. Error Handling is User Experience
- Clear, actionable error messages
- Consistent error format across all endpoints
- Log details internally, summarize externally

### 4. Documentation is a Product
- Keep docs in sync with code (OpenAPI)
- Include examples and error scenarios
- Provide migration guides for version changes

### 5. Rate Limiting Protects Everyone
- Implement rate limiting early
- Use appropriate algorithms (token bucket for bursts)
- Communicate limits via headers

### 6. Consistency Builds Trust
- Consistent naming conventions
- Consistent response structures
- Consistent behavior across endpoints

---

## Quick Reference

| Aspect | Best Practice |
|--------|--------------|
| Versioning | URL path (`/api/v1/`) |
| Pagination | Cursor-based for large datasets |
| Errors | Structured JSON with codes |
| Auth | OAuth 2.0 or API keys over HTTPS |
| Rate Limiting | Token bucket algorithm |
| Documentation | OpenAPI/Swagger spec |
| IDs | UUIDs or prefixed IDs |
| Naming | Consistent snake_case |
