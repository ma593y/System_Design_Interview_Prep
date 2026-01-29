# Load Balancing

## Overview

Load balancing distributes incoming network traffic across multiple servers to ensure no single server becomes overwhelmed, improving reliability and availability.

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │    Load     │
                    │  Balancer   │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │  Server 1 │    │  Server 2 │    │  Server 3 │
    └───────────┘    └───────────┘    └───────────┘
```

## Types of Load Balancers

### Layer 4 (Transport Layer)

Operates at TCP/UDP level. Routes based on IP address and port.

**Characteristics:**
- Fast (minimal packet inspection)
- Simple configuration
- Cannot make decisions based on content
- Good for non-HTTP traffic

**Use Cases:**
- Gaming servers
- Streaming services
- Database connections

### Layer 7 (Application Layer)

Operates at HTTP/HTTPS level. Can inspect request content.

**Characteristics:**
- Content-aware routing
- Can terminate SSL
- URL-based routing
- Header manipulation
- Slower than L4 (more processing)

**Use Cases:**
- Web applications
- API gateways
- Microservices routing

```
Layer 7 Routing Example:

/api/*     → API Servers
/static/*  → CDN
/admin/*   → Admin Servers
/         → Web Servers
```

## Load Balancing Algorithms

### Round Robin

Distributes requests sequentially across servers.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (repeat)
```

**Pros:** Simple, even distribution
**Cons:** Ignores server capacity and health

### Weighted Round Robin

Assigns weights based on server capacity.

```
Server A (weight: 3): Handles 3 out of 6 requests
Server B (weight: 2): Handles 2 out of 6 requests
Server C (weight: 1): Handles 1 out of 6 requests
```

**Pros:** Accounts for different server capacities
**Cons:** Doesn't adapt to real-time load

### Least Connections

Routes to server with fewest active connections.

```
Server A: 10 connections → Skip
Server B: 5 connections  → Skip
Server C: 2 connections  → Route here
```

**Pros:** Adapts to server load
**Cons:** Doesn't account for request complexity

### Weighted Least Connections

Combines least connections with server weights.

**Pros:** Best of both approaches
**Cons:** More complex to configure

### IP Hash

Routes based on client IP address hash.

```
hash(client_IP) % num_servers = target_server
```

**Pros:** Session persistence without cookies
**Cons:** Uneven distribution if IPs are clustered

### Least Response Time

Routes to server with fastest response time.

**Pros:** Optimizes for user experience
**Cons:** Requires constant monitoring

## Comparison Table

| Algorithm | Best For | Drawback |
|-----------|----------|----------|
| Round Robin | Homogeneous servers | Ignores load |
| Weighted RR | Mixed capacity servers | Static weights |
| Least Connections | Variable request length | Complexity |
| IP Hash | Session persistence | Uneven distribution |
| Least Response Time | Performance-critical | Monitoring overhead |

## Health Checks

Load balancers must detect unhealthy servers.

### Types of Health Checks

1. **TCP Check**: Can establish connection?
2. **HTTP Check**: Returns 200 OK?
3. **Custom Check**: Application-specific validation

### Health Check Configuration

```
Health Check Settings:
- Interval: 10 seconds
- Timeout: 5 seconds
- Unhealthy threshold: 3 failures
- Healthy threshold: 2 successes
```

## Session Persistence (Sticky Sessions)

Ensures a client always reaches the same server.

### Methods

1. **Cookie-based**: LB sets a cookie with server ID
2. **IP-based**: Route by client IP
3. **URL-based**: Encode server in URL

**Pros:**
- Required for stateful applications
- Better cache utilization

**Cons:**
- Uneven load distribution
- Problematic when servers fail

## High Availability Setup

### Active-Passive

```
┌─────────────┐     ┌─────────────┐
│   Active    │     │   Passive   │
│     LB      │────▶│     LB      │
│  (Primary)  │     │  (Standby)  │
└─────────────┘     └─────────────┘
      │
      ▼
  [Servers]
```

### Active-Active

```
┌─────────────┐     ┌─────────────┐
│   Active    │     │   Active    │
│    LB 1     │     │    LB 2     │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └─────────┬─────────┘
                 ▼
             [Servers]
```

## Global Load Balancing (GSLB)

Distributes traffic across data centers globally.

```
                    ┌─────────────┐
                    │   DNS/GSLB  │
                    └──────┬──────┘
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
┌────────────┐      ┌────────────┐      ┌────────────┐
│  US-East   │      │  US-West   │      │   Europe   │
│    DC      │      │    DC      │      │     DC     │
└────────────┘      └────────────┘      └────────────┘
```

**Strategies:**
- Geographic proximity
- Latency-based
- Failover

## Common Load Balancers

| Name | Type | Use Case |
|------|------|----------|
| HAProxy | Software L4/L7 | High performance |
| NGINX | Software L7 | Web traffic |
| AWS ALB | Cloud L7 | HTTP/HTTPS |
| AWS NLB | Cloud L4 | TCP/UDP |
| F5 | Hardware | Enterprise |

## Interview Talking Points

1. "Choose L4 for speed, L7 for content-aware routing"
2. "Least connections works well when request processing times vary"
3. "Health checks are critical for reliability"
4. "Sticky sessions solve stateful problems but create scaling challenges"
5. "Always have redundant load balancers"

## Common Interview Questions

1. **Q: When would you use L4 vs L7 load balancing?**
   A: L4 for raw speed and non-HTTP traffic. L7 when you need content-based routing, SSL termination, or header manipulation.

2. **Q: How do you handle session state with load balancing?**
   A: Best approach is stateless design with external session store. If not possible, use sticky sessions with cookie-based affinity.

3. **Q: What happens when a server behind the LB fails?**
   A: Health checks detect the failure, mark server unhealthy, stop routing traffic to it. When recovered, it's gradually reintroduced.

4. **Q: How do you scale the load balancer itself?**
   A: Use DNS round robin, active-active setup, or cloud-managed LBs that auto-scale.

## Key Takeaways

- Load balancers are critical for scalability and availability
- Choose algorithm based on your specific use case
- Always implement health checks
- Plan for LB redundancy
- Prefer stateless design to avoid sticky session complexity

## Further Reading

- HAProxy documentation
- NGINX load balancing guide
- AWS Elastic Load Balancing documentation
