# Proxies

## Overview

A proxy is an intermediary server that sits between clients and backend servers, handling requests on their behalf.

## Types of Proxies

### Forward Proxy

Sits in front of clients, hides client identity from servers.

```
┌────────────────────────────────────────────────────────────┐
│                     Organization                            │
│                                                             │
│  ┌────────┐    ┌───────────────┐                           │
│  │Client 1│───▶│               │                           │
│  └────────┘    │    Forward    │                           │
│  ┌────────┐───▶│     Proxy     │───────▶ Internet ───▶ Server
│  │Client 2│    │               │                           │
│  └────────┘    └───────────────┘                           │
│  ┌────────┐───▶        │                                   │
│  │Client 3│            ▼                                   │
│  └────────┘     (Caching, Filtering,                       │
│                  Access Control)                            │
└────────────────────────────────────────────────────────────┘

Server sees proxy IP, not client IP
```

**Use Cases:**
- Corporate internet access control
- Caching for bandwidth savings
- Anonymity/privacy
- Bypass geo-restrictions

### Reverse Proxy

Sits in front of servers, hides server identity from clients.

```
                    ┌─────────────────────────────────────────┐
                    │              Organization               │
                    │                                         │
                    │    ┌───────────────┐    ┌──────────┐   │
Internet ───▶ Clients ──▶│    Reverse    │───▶│ Server 1 │   │
                    │    │     Proxy     │    └──────────┘   │
                    │    │               │───▶┌──────────┐   │
                    │    │               │    │ Server 2 │   │
                    │    └───────────────┘    └──────────┘   │
                    │           │             ┌──────────┐   │
                    │           ▼         ───▶│ Server 3 │   │
                    │    (Load Balancing,     └──────────┘   │
                    │     SSL, Caching)                      │
                    └─────────────────────────────────────────┘

Client sees proxy, not backend servers
```

**Use Cases:**
- Load balancing
- SSL termination
- Caching
- Security (hide internal architecture)
- Compression

## Reverse Proxy Features

### Load Balancing

```
                    ┌───────────┐
                    │  Reverse  │
Requests ──────────▶│   Proxy   │
                    └─────┬─────┘
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
      ┌─────────┐    ┌─────────┐    ┌─────────┐
      │Server 1 │    │Server 2 │    │Server 3 │
      └─────────┘    └─────────┘    └─────────┘

Algorithms: Round-robin, least connections, IP hash
```

### SSL/TLS Termination

```
Client ══════╗
             ║ HTTPS (encrypted)
             ▼
      ┌─────────────┐
      │   Reverse   │
      │    Proxy    │ ← Decrypts here
      └──────┬──────┘
             │ HTTP (unencrypted)
             ▼
      ┌─────────────┐
      │   Backend   │
      └─────────────┘

Benefits:
- Centralized certificate management
- Reduced backend CPU load
- Easier certificate rotation
```

### Caching

```
Request ──▶ Proxy ──▶ Cache Check
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
           Cache HIT            Cache MISS
              │                     │
              ▼                     ▼
        Return cached         Fetch from origin
          response            Cache and return
```

### Compression

```
Backend Response (1MB) ──▶ Proxy ──▶ Compressed (200KB) ──▶ Client

Saves bandwidth, improves load times
Common: gzip, brotli
```

### Request/Response Modification

```
# Add headers
X-Request-ID: uuid-123
X-Real-IP: client-ip

# Remove headers
Server: nginx  → (removed for security)

# URL rewriting
/api/v1/users → /users (internal)
```

## API Gateway

Specialized reverse proxy for APIs.

```
                    ┌─────────────────────────────────────┐
                    │            API Gateway               │
                    ├─────────────────────────────────────┤
                    │  • Authentication                    │
                    │  • Rate Limiting                     │
                    │  • Request Routing                   │
                    │  • Protocol Translation              │
                    │  • Request/Response Transform        │
                    │  • Analytics & Monitoring            │
                    └───────────────┬─────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
     ┌────────────┐          ┌────────────┐          ┌────────────┐
     │ User       │          │ Order      │          │ Payment    │
     │ Service    │          │ Service    │          │ Service    │
     └────────────┘          └────────────┘          └────────────┘
```

### API Gateway Features

| Feature | Description |
|---------|-------------|
| **Authentication** | Verify JWT, API keys, OAuth |
| **Authorization** | Check permissions |
| **Rate Limiting** | Throttle requests per client |
| **Routing** | Route to appropriate service |
| **Protocol Translation** | REST to gRPC, etc. |
| **Request Aggregation** | Combine multiple service calls |
| **Caching** | Cache API responses |
| **Logging/Monitoring** | Centralized observability |

### API Gateway Patterns

#### Gateway Routing

```
GET /users/* ────────▶ User Service
GET /orders/* ───────▶ Order Service
GET /products/* ─────▶ Product Service
```

#### Gateway Aggregation

```
GET /user-dashboard

API Gateway:
├── GET /users/123      ───▶ User Service
├── GET /orders?user=123 ──▶ Order Service
└── GET /notifications   ──▶ Notification Service

Response: Combined data from all services
```

#### Gateway Offloading

```
Offload to gateway:
- SSL termination
- Authentication
- CORS handling
- Compression
- Rate limiting

Backend services stay simple
```

## Service Mesh vs API Gateway

```
API Gateway (North-South traffic):
External ──▶ Gateway ──▶ Services

Service Mesh (East-West traffic):
Service A ◀──▶ Sidecar ◀──▶ Sidecar ◀──▶ Service B
```

| Aspect | API Gateway | Service Mesh |
|--------|-------------|--------------|
| Traffic | External to internal | Internal to internal |
| Deployment | Centralized | Distributed (sidecars) |
| Use Case | Client-facing APIs | Inter-service communication |
| Examples | Kong, Apigee | Istio, Linkerd |

## Popular Proxy Software

### NGINX

```nginx
# Reverse proxy configuration
upstream backend {
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### HAProxy

```
# HAProxy configuration
frontend http_front
    bind *:80
    default_backend servers

backend servers
    balance roundrobin
    server server1 192.168.1.1:80 check
    server server2 192.168.1.2:80 check
    server server3 192.168.1.3:80 check
```

### Comparison

| Feature | NGINX | HAProxy | Envoy |
|---------|-------|---------|-------|
| Type | Web server + proxy | Pure proxy | Service mesh proxy |
| Config | File-based | File-based | API-driven |
| HTTP/2 | Yes | Yes | Yes |
| gRPC | Yes | Limited | Native |
| Service Discovery | Limited | Limited | Native |
| Hot Reload | Yes | Yes | Yes |

## Proxy Protocols

### HTTP/HTTPS Proxy

```
CONNECT method for HTTPS tunneling:
Client → Proxy: CONNECT server.com:443
Proxy → Server: TCP connection
Client ◀═══ Encrypted tunnel ═══▶ Server
```

### SOCKS Proxy

```
General-purpose proxy for any protocol
Operates at TCP level
Supports authentication
Used for: VPNs, Tor, general tunneling
```

### Transparent Proxy

```
Client doesn't know about proxy
Intercepts traffic at network level
Used for: Content filtering, caching at ISP level
```

## High Availability

### Active-Passive

```
┌──────────────┐
│   Primary    │◀─── Traffic
│    Proxy     │
└──────┬───────┘
       │ Heartbeat
       ▼
┌──────────────┐
│   Standby    │    (Takes over if primary fails)
│    Proxy     │
└──────────────┘
```

### Active-Active

```
       ┌─────────────────────────┐
       │        DNS/LB           │
       └───────────┬─────────────┘
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
   ┌──────────┐        ┌──────────┐
   │ Proxy 1  │        │ Proxy 2  │
   │ (Active) │        │ (Active) │
   └──────────┘        └──────────┘
```

## Interview Talking Points

1. "Forward proxy protects clients, reverse proxy protects servers"
2. "Reverse proxies enable SSL termination, caching, and load balancing"
3. "API gateways add authentication, rate limiting, and routing"
4. "Use service mesh for internal service-to-service communication"
5. "Always deploy proxies in HA configuration"

## Common Interview Questions

1. **Q: What's the difference between forward and reverse proxy?**
   A: Forward proxy sits in front of clients (client-side), reverse proxy sits in front of servers (server-side). Forward hides clients, reverse hides servers.

2. **Q: Why use an API gateway?**
   A: Centralized authentication, rate limiting, routing, protocol translation, and monitoring. Simplifies backend services.

3. **Q: How do you handle proxy failures?**
   A: Deploy in HA pairs (active-passive or active-active), use health checks, implement failover with DNS or load balancer.

4. **Q: When would you use a service mesh over an API gateway?**
   A: Service mesh for internal service-to-service (east-west) traffic. API gateway for external client traffic (north-south). Often use both.

## Key Takeaways

- Proxies are fundamental for security, performance, and scaling
- Reverse proxies are essential in modern architectures
- API gateways provide crucial cross-cutting concerns
- Choose the right proxy software based on requirements
- Always plan for high availability

## Further Reading

- NGINX documentation
- Envoy proxy architecture
- Kong API Gateway documentation
- Service mesh concepts (Istio)
