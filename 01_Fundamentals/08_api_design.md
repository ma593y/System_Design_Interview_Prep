# API Design

## Overview

APIs (Application Programming Interfaces) define how systems communicate. Good API design is crucial for usability, maintainability, and scalability.

## API Styles

### REST (Representational State Transfer)

Resource-based, stateless, uses HTTP methods.

```
Resources as URLs:
GET    /users          # List users
GET    /users/123      # Get user 123
POST   /users          # Create user
PUT    /users/123      # Replace user 123
PATCH  /users/123      # Update user 123
DELETE /users/123      # Delete user 123
```

**Principles:**
- Stateless
- Uniform interface
- Resource-based
- Cacheable
- Layered system

### GraphQL

Query language for APIs.

```graphql
# Client specifies exactly what data it needs

query {
  user(id: "123") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}

# Response matches query structure
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@mail.com",
      "posts": [
        {"title": "Hello World", "createdAt": "2024-01-01"}
      ]
    }
  }
}
```

**Pros:**
- Client gets exactly what it needs
- Single endpoint
- Strong typing
- Introspection

**Cons:**
- Complexity
- Caching challenges
- N+1 query problems

### gRPC

High-performance RPC using Protocol Buffers.

```protobuf
// user.proto
service UserService {
  rpc GetUser (UserRequest) returns (User);
  rpc ListUsers (ListRequest) returns (stream User);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

**Pros:**
- Very fast (binary protocol)
- Streaming support
- Strong contracts
- Code generation

**Cons:**
- Not browser-friendly
- Less human-readable
- More complex setup

## Comparison

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Protocol | HTTP | HTTP | HTTP/2 |
| Format | JSON | JSON | Protobuf |
| Typing | Weak | Strong | Strong |
| Caching | Easy | Complex | Complex |
| Flexibility | Medium | High | Low |
| Performance | Good | Good | Excellent |
| Best for | Web APIs | Mobile apps | Microservices |

## REST Best Practices

### URL Design

```
Good:
GET /users
GET /users/123
GET /users/123/orders
GET /orders?status=pending

Bad:
GET /getUsers
GET /user/get/123
GET /fetchUserOrders?userId=123
POST /users/delete/123
```

### HTTP Methods

| Method | Purpose | Idempotent | Request Body |
|--------|---------|------------|--------------|
| GET | Read | Yes | No |
| POST | Create | No | Yes |
| PUT | Replace | Yes | Yes |
| PATCH | Partial update | No* | Yes |
| DELETE | Delete | Yes | No |

### Status Codes

```
Success:
200 OK              - Successful GET, PUT, PATCH
201 Created         - Successful POST (include Location header)
204 No Content      - Successful DELETE

Client Errors:
400 Bad Request     - Invalid syntax
401 Unauthorized    - Not authenticated
403 Forbidden       - Authenticated but not authorized
404 Not Found       - Resource doesn't exist
409 Conflict        - State conflict
422 Unprocessable   - Validation error

Server Errors:
500 Internal Error  - Server fault
502 Bad Gateway     - Upstream error
503 Unavailable     - Temporarily down
504 Gateway Timeout - Upstream timeout
```

### Request/Response Format

```json
// Request
POST /users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@mail.com"
}

// Response
HTTP/1.1 201 Created
Location: /users/123

{
  "id": 123,
  "name": "Alice",
  "email": "alice@mail.com",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

## Pagination

### Offset-Based

```
GET /users?offset=0&limit=20
GET /users?offset=20&limit=20
GET /users?offset=40&limit=20
```

**Pros:** Simple, random access
**Cons:** Inconsistent with changes, slow for large offsets

### Cursor-Based

```
GET /users?limit=20
GET /users?cursor=eyJpZCI6MTAwfQ&limit=20

Response:
{
  "data": [...],
  "nextCursor": "eyJpZCI6MTIwfQ",
  "hasMore": true
}
```

**Pros:** Consistent, efficient
**Cons:** No random access, more complex

### Keyset-Based

```
GET /users?limit=20&after_id=100
```

**Pros:** Very efficient
**Cons:** Requires sortable field

## Filtering, Sorting, Search

```
# Filtering
GET /users?status=active&role=admin

# Sorting
GET /users?sort=created_at&order=desc
GET /users?sort=-created_at  # Alternative syntax

# Search
GET /users?search=alice
GET /users?q=alice&fields=name,email

# Complex filtering
GET /products?price[gte]=10&price[lte]=100
GET /products?category[in]=electronics,clothing
```

## Versioning

### URL Versioning

```
GET /v1/users
GET /v2/users
```

**Pros:** Clear, easy to implement
**Cons:** URL pollution

### Header Versioning

```
GET /users
Accept: application/vnd.api+json;version=1
```

**Pros:** Clean URLs
**Cons:** Hidden, harder to test

### Query Parameter

```
GET /users?version=1
```

**Pros:** Easy to use
**Cons:** Less RESTful

## Rate Limiting

Protect APIs from abuse and ensure fair usage.

### Response Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 998
X-RateLimit-Reset: 1642000000
```

### Rate Limit Exceeded

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{
  "error": "Rate limit exceeded",
  "retryAfter": 60
}
```

### Algorithms

| Algorithm | Description |
|-----------|-------------|
| Token Bucket | Tokens added at fixed rate, requests consume tokens |
| Leaky Bucket | Requests queue, processed at fixed rate |
| Fixed Window | Count requests per time window |
| Sliding Window | Rolling count over time period |

## Authentication & Authorization

### API Keys

```
GET /users
Authorization: Api-Key sk-abc123
```

**Use for:** Server-to-server, simple cases

### JWT (JSON Web Tokens)

```
GET /users
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

JWT Structure:
Header.Payload.Signature

Payload:
{
  "sub": "user123",
  "exp": 1642000000,
  "roles": ["admin"]
}
```

**Use for:** Stateless auth, user sessions

### OAuth 2.0

```
Authorization Code Flow:
1. User redirected to auth server
2. User authenticates
3. Auth server returns code
4. Exchange code for access token
5. Use access token for API calls
```

**Use for:** Third-party access, delegated auth

## Error Handling

### Consistent Error Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      },
      {
        "field": "age",
        "message": "Must be at least 18"
      }
    ],
    "requestId": "req-abc123"
  }
}
```

### Error Categories

```
Validation Errors (400):
- Invalid field format
- Missing required fields
- Business rule violations

Authentication Errors (401):
- Missing credentials
- Invalid token
- Expired token

Authorization Errors (403):
- Insufficient permissions
- Resource access denied

Not Found (404):
- Resource doesn't exist
```

## API Documentation

### OpenAPI (Swagger)

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0

paths:
  /users/{id}:
    get:
      summary: Get user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
```

## Interview Talking Points

1. "REST for web APIs, gRPC for internal microservices"
2. "GraphQL solves over-fetching and under-fetching"
3. "Cursor pagination for large datasets"
4. "Rate limiting protects against abuse and ensures fair usage"
5. "Consistent error handling improves developer experience"

## Common Interview Questions

1. **Q: REST vs GraphQL - when to use which?**
   A: REST for simple, cacheable APIs. GraphQL when clients need flexibility in data fetching (mobile apps, complex UIs).

2. **Q: How do you handle API versioning?**
   A: URL versioning is most common (`/v1/users`). Maintain backward compatibility, deprecate gracefully.

3. **Q: How do you implement rate limiting?**
   A: Token bucket or sliding window. Use Redis for distributed limiting. Return proper headers (429, Retry-After).

4. **Q: How do you design APIs for mobile apps?**
   A: Consider bandwidth, batch endpoints, field filtering, compression, efficient pagination.

## Key Takeaways

- Choose API style based on use case
- Follow REST conventions for consistency
- Implement proper pagination for large datasets
- Always include rate limiting
- Design consistent error responses
- Document your APIs thoroughly

## Further Reading

- REST API Design Rulebook
- GraphQL specification
- gRPC documentation
- OpenAPI specification
