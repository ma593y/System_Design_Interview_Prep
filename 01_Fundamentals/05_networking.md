# Networking Fundamentals

## Overview

Understanding networking is essential for system design. Every distributed system relies on network communication.

## OSI Model (Simplified)

```
┌─────────────────────────────────────────────┐
│ Layer 7: Application  │ HTTP, WebSocket, gRPC│
├───────────────────────┼─────────────────────┤
│ Layer 4: Transport    │ TCP, UDP             │
├───────────────────────┼─────────────────────┤
│ Layer 3: Network      │ IP, Routing          │
├───────────────────────┼─────────────────────┤
│ Layer 1-2: Physical   │ Ethernet, WiFi       │
└─────────────────────────────────────────────┘
```

## DNS (Domain Name System)

Translates domain names to IP addresses.

### DNS Resolution Flow

```
Browser                DNS Resolver        Root DNS        TLD DNS        Auth DNS
   │                        │                  │              │               │
   │──── google.com ───────▶│                  │              │               │
   │                        │──── . ──────────▶│              │               │
   │                        │◀── .com servers ─│              │               │
   │                        │──── .com ────────────────────▶│               │
   │                        │◀── google.com servers ─────────│               │
   │                        │──── google.com ────────────────────────────────▶│
   │                        │◀── 142.250.80.46 ──────────────────────────────│
   │◀─── 142.250.80.46 ────│                  │              │               │
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | `google.com → 142.250.80.46` |
| AAAA | IPv6 address | `google.com → 2607:f8b0::` |
| CNAME | Alias | `www.google.com → google.com` |
| MX | Mail server | `gmail.com → mail.google.com` |
| NS | Name server | `google.com → ns1.google.com` |
| TXT | Text data | SPF, DKIM records |

### DNS Caching

```
TTL (Time To Live): How long to cache

Browser Cache → OS Cache → Router Cache → ISP Cache → DNS Resolver
```

### DNS Load Balancing

Round-robin DNS returns multiple IPs.

```
dig google.com
→ 142.250.80.46
→ 142.250.80.47
→ 142.250.80.48
```

## TCP vs UDP

### TCP (Transmission Control Protocol)

Reliable, ordered, connection-oriented.

```
Three-Way Handshake:

Client                    Server
   │                         │
   │─────── SYN ────────────▶│
   │◀────── SYN-ACK ─────────│
   │─────── ACK ────────────▶│
   │                         │
   │◀──── Data Transfer ────▶│
   │                         │
   │─────── FIN ────────────▶│
   │◀────── ACK ─────────────│
```

**Features:**
- Guaranteed delivery (acknowledgments)
- Ordered packets
- Flow control
- Congestion control

**Use Cases:** HTTP, email, file transfer, databases

### UDP (User Datagram Protocol)

Fast, connectionless, best-effort delivery.

```
Client                    Server
   │                         │
   │─────── Data ───────────▶│  (No handshake)
   │─────── Data ───────────▶│  (May arrive out of order)
   │─────── Data ───────────▶│  (May be lost)
```

**Features:**
- No connection overhead
- No ordering guarantee
- No delivery guarantee
- Low latency

**Use Cases:** Video streaming, gaming, DNS, VoIP

### Comparison

| Aspect | TCP | UDP |
|--------|-----|-----|
| Reliability | Guaranteed | Best-effort |
| Order | Preserved | Not guaranteed |
| Speed | Slower | Faster |
| Overhead | Higher | Lower |
| Connection | Required | Not required |

## HTTP/HTTPS

### HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Read | Yes | Yes |
| POST | Create | No | No |
| PUT | Replace | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove | Yes | No |
| HEAD | Headers only | Yes | Yes |
| OPTIONS | Capabilities | Yes | Yes |

### HTTP Status Codes

| Range | Category | Examples |
|-------|----------|----------|
| 1xx | Informational | 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved, 302 Found, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 404 Not Found |
| 5xx | Server Error | 500 Internal Error, 502 Bad Gateway, 503 Unavailable |

### HTTP/1.1 vs HTTP/2 vs HTTP/3

```
HTTP/1.1:
Request 1 ────────▶ Response 1
Request 2 ────────▶ Response 2  (Head-of-line blocking)
Request 3 ────────▶ Response 3

HTTP/2 (multiplexing):
Request 1 ─┐
Request 2 ─┼──────▶ All responses interleaved
Request 3 ─┘

HTTP/3 (QUIC/UDP):
Same as HTTP/2 but over UDP (no TCP head-of-line blocking)
```

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | No | Yes | Yes |
| Header Compression | No | HPACK | QPACK |
| Server Push | No | Yes | Yes |

## WebSockets

Persistent, bidirectional communication.

```
HTTP Upgrade:
Client ─── GET / (Upgrade: websocket) ──▶ Server
Client ◀── 101 Switching Protocols ────── Server

Full-duplex communication:
Client ◀────────────────────────────────▶ Server
        Messages in both directions
```

**Use Cases:**
- Real-time chat
- Live updates
- Gaming
- Collaborative editing

### WebSocket vs HTTP Polling

```
HTTP Polling:
Client ─── Request ──▶ Server (empty)
Client ─── Request ──▶ Server (empty)
Client ─── Request ──▶ Server (data!)  ← Wasteful

WebSocket:
Client ◀───────────────▶ Server
        ← Data pushed when available
```

## Long Polling

Server holds request until data available.

```
Client ─── Request ──────────────────▶ Server
                    (waits for data)
Client ◀─────────────────── Response ─ Server (when data ready)
Client ─── Request ──────────────────▶ Server (immediately reconnect)
```

**Pros:** Works everywhere, simpler than WebSockets
**Cons:** Higher latency, more overhead than WebSockets

## Server-Sent Events (SSE)

One-way server-to-client streaming.

```
Client ─── GET /events ──▶ Server
Client ◀── event: update ─ Server
Client ◀── event: update ─ Server
Client ◀── event: update ─ Server
```

**Pros:** Simple, automatic reconnection
**Cons:** One-way only, limited browser connections

## Network Latency

### Typical Latencies

| Operation | Latency |
|-----------|---------|
| Same datacenter | 0.5 ms |
| Cross-region (US) | 40 ms |
| Cross-Atlantic | 80 ms |
| Cross-Pacific | 150 ms |

### Reducing Latency

1. **CDN**: Cache content at edge
2. **Connection reuse**: Keep-alive, connection pooling
3. **Compression**: Reduce data size
4. **Protocol optimization**: HTTP/2, QUIC
5. **Geographic distribution**: Deploy closer to users

## Interview Talking Points

1. "TCP for reliability, UDP for speed"
2. "DNS is often the first step in any request"
3. "WebSockets for bidirectional real-time communication"
4. "HTTP/2 multiplexing eliminates head-of-line blocking"
5. "Network latency is often the biggest bottleneck"

## Common Interview Questions

1. **Q: When would you use UDP over TCP?**
   A: When low latency is more important than reliability - video streaming, gaming, real-time communication where occasional packet loss is acceptable.

2. **Q: How does a CDN improve performance?**
   A: Caches content at edge locations closer to users, reducing round-trip time and offloading origin server.

3. **Q: When would you use WebSockets vs HTTP?**
   A: WebSockets for bidirectional real-time communication (chat, gaming). HTTP for request-response patterns.

4. **Q: What happens when you type a URL in browser?**
   A: DNS lookup → TCP connection → TLS handshake (HTTPS) → HTTP request → Server processing → Response → Rendering

## Key Takeaways

- DNS is fundamental to the internet
- TCP guarantees delivery, UDP prioritizes speed
- Modern protocols (HTTP/2, HTTP/3) improve performance
- WebSockets enable real-time bidirectional communication
- Network latency significantly impacts user experience

## Further Reading

- "High Performance Browser Networking" by Ilya Grigorik
- RFC documents for protocols
- Cloudflare Learning Center
