# DNS and CDN

## DNS (Domain Name System)

### Overview

DNS translates human-readable domain names to IP addresses.

```
Browser: "What is the IP for google.com?"
DNS:     "142.250.80.46"
Browser: "Thanks!" → Connects to 142.250.80.46
```

### DNS Hierarchy

```
                    ┌─────────────┐
                    │ Root DNS    │  (13 root server clusters)
                    │     .       │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
      ┌─────────┐    ┌─────────┐    ┌─────────┐
      │  .com   │    │  .org   │    │  .net   │  TLD Servers
      └────┬────┘    └─────────┘    └─────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌────────┐   ┌────────┐
│google  │   │amazon  │   Authoritative Servers
│.com    │   │.com    │
└────────┘   └────────┘
```

### DNS Resolution Process

```
User                 Resolver           Root        TLD         Auth
 │                      │                │           │            │
 │─── google.com? ─────▶│                │           │            │
 │                      │─── . ─────────▶│           │            │
 │                      │◀── .com NS ────│           │            │
 │                      │─── .com ───────────────▶│            │
 │                      │◀── google.com NS ───────│            │
 │                      │─── google.com ─────────────────────▶│
 │                      │◀── 142.250.80.46 ──────────────────│
 │◀── 142.250.80.46 ───│                │           │            │
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | `google.com → 142.250.80.46` |
| AAAA | IPv6 address | `google.com → 2607:f8b0::` |
| CNAME | Alias to another domain | `www.google.com → google.com` |
| MX | Mail server | `gmail.com → mail.google.com (priority 10)` |
| NS | Name server | `google.com → ns1.google.com` |
| TXT | Text data | SPF, DKIM, verification |
| SOA | Zone authority | Primary NS, admin email, serial |
| PTR | Reverse lookup | `46.80.250.142 → google.com` |

### DNS Caching

```
TTL (Time To Live):

Browser Cache (minutes)
    ↓
OS Cache (minutes)
    ↓
Router Cache (minutes)
    ↓
ISP Resolver Cache (hours)
    ↓
Authoritative Server

Lower TTL = More current, higher load
Higher TTL = Faster, potentially stale
```

### DNS Load Balancing

#### Round Robin DNS

```
dig example.com
→ 192.168.1.1
→ 192.168.1.2
→ 192.168.1.3

Clients cache and use different IPs
Simple but no health checking
```

#### Weighted Round Robin

```
example.com:
  192.168.1.1 (weight: 3) → 60% traffic
  192.168.1.2 (weight: 2) → 40% traffic
```

#### GeoDNS

```
User in US → US datacenter IP
User in EU → EU datacenter IP
User in Asia → Asia datacenter IP

Based on client's resolver location
```

### DNS Security

**DNSSEC (DNS Security Extensions):**
- Digitally signs DNS records
- Prevents DNS spoofing/cache poisoning
- Chain of trust from root

**DNS over HTTPS (DoH) / DNS over TLS (DoT):**
- Encrypts DNS queries
- Prevents eavesdropping
- Privacy protection

---

## CDN (Content Delivery Network)

### Overview

CDN caches content at edge locations worldwide to reduce latency.

```
Without CDN:
User (Tokyo) ──────────────────────▶ Origin (US)
              ~150ms round trip

With CDN:
User (Tokyo) ───▶ Edge (Tokyo) ───▶ Origin (US)
              ~10ms    │
                       │ Cache hit!
                       └─▶ Return immediately
```

### CDN Architecture

```
                    ┌─────────────────────┐
                    │    Origin Server    │
                    │   (Your backend)    │
                    └──────────┬──────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
      ┌──────────┐       ┌──────────┐       ┌──────────┐
      │ Edge PoP │       │ Edge PoP │       │ Edge PoP │
      │  (US)    │       │  (EU)    │       │ (Asia)   │
      └────┬─────┘       └────┬─────┘       └────┬─────┘
           │                  │                  │
    ┌──────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐
    ▼      ▼      ▼    ▼      ▼      ▼    ▼      ▼      ▼
 Users  Users  Users Users  Users  Users Users  Users  Users
```

### How CDN Works

```
1. User requests resource
2. DNS resolves to nearest edge
3. Edge checks cache:
   - HIT: Return cached content
   - MISS: Fetch from origin, cache, return

Request Flow:
User → DNS → Edge (cache check) → Origin (if miss)
                  ↓
            Return response
```

### CDN Caching Strategies

#### Cache-Control Headers

```
# Cache for 1 year (immutable assets)
Cache-Control: public, max-age=31536000, immutable

# Cache for 1 hour, revalidate after
Cache-Control: public, max-age=3600, must-revalidate

# No caching
Cache-Control: no-store

# Private (not on CDN)
Cache-Control: private, max-age=3600
```

#### Cache Key

```
Default cache key:
URL + Host + Query String

Custom cache key might include:
- URL path only
- Specific headers
- Cookies
- Device type
```

### CDN Features

| Feature | Description |
|---------|-------------|
| **Edge Caching** | Cache static content at edge |
| **Dynamic Acceleration** | Optimize dynamic content delivery |
| **SSL Termination** | Handle HTTPS at edge |
| **DDoS Protection** | Absorb attack traffic |
| **WAF** | Web Application Firewall |
| **Image Optimization** | Resize, compress on the fly |
| **Video Streaming** | Adaptive bitrate delivery |

### Cache Invalidation

```
Methods:
1. TTL expiration (automatic)
2. Purge by URL
3. Purge by tag/key
4. Purge all

Best practice:
- Use versioned URLs for assets
- /static/app.v2.js (cache forever)
- Avoids purge complexity
```

### CDN for Different Content Types

| Content | Strategy |
|---------|----------|
| Static assets (JS, CSS) | Long TTL, versioned URLs |
| Images | Long TTL, image optimization |
| HTML pages | Short TTL or no cache |
| API responses | Vary by auth, short TTL |
| Video | Chunked delivery, long TTL |

### Multi-CDN Strategy

```
┌─────────────────────────────────────────────┐
│              Traffic Manager                 │
│   (Route53, Cloudflare, custom logic)       │
└───────────────┬─────────────────────────────┘
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│ CDN A  │ │ CDN B  │ │ CDN C  │
└────────┘ └────────┘ └────────┘

Benefits:
- Redundancy
- Best performance per region
- Cost optimization
```

### Popular CDN Providers

| Provider | Strengths |
|----------|-----------|
| Cloudflare | DDoS protection, free tier |
| Akamai | Enterprise, largest network |
| AWS CloudFront | AWS integration |
| Fastly | Real-time purge, edge compute |
| Google Cloud CDN | GCP integration |

## DNS + CDN Integration

```
DNS Resolution → CDN Edge → Origin

example.com
    │
    ▼ (CNAME to CDN)
cdn.example.com → Cloudflare edge
    │
    ▼ (Cache miss)
origin.example.com → Your server
```

### Anycast Routing

```
Same IP announced from multiple locations:

User in US ──▶ 1.2.3.4 (routes to US edge)
User in EU ──▶ 1.2.3.4 (routes to EU edge)
User in Asia ──▶ 1.2.3.4 (routes to Asia edge)

BGP routing finds closest edge
```

## Interview Talking Points

1. "DNS is the first step in any web request"
2. "CDN reduces latency by serving content from edge locations"
3. "Cache-Control headers are crucial for CDN behavior"
4. "Use versioned URLs to avoid cache invalidation complexity"
5. "Anycast enables automatic routing to nearest edge"

## Common Interview Questions

1. **Q: How does DNS work?**
   A: Recursive resolution through hierarchy: Root → TLD → Authoritative. Each level cached with TTL.

2. **Q: How would you reduce DNS latency?**
   A: Use DNS providers with many PoPs, optimize TTLs, implement DNS prefetching, use GeoDNS.

3. **Q: What content should be cached on CDN?**
   A: Static assets (long TTL), images (with optimization), video chunks. Not sensitive data or highly dynamic content.

4. **Q: How do you handle CDN cache invalidation?**
   A: Prefer versioned URLs. If purging needed, use cache tags or specific URL purge. Avoid purge-all.

## Key Takeaways

- DNS is hierarchical and heavily cached
- CDN reduces latency and origin load
- Proper cache headers are essential
- Version static assets instead of purging
- Consider multi-CDN for reliability

## Further Reading

- Cloudflare Learning Center
- AWS CloudFront documentation
- "High Performance Browser Networking" Chapter 9
