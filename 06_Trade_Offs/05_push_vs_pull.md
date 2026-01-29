# Push vs Pull

## Overview

Push and pull are fundamental patterns for data distribution. The choice impacts system design, scalability, and user experience.

## Core Concepts

### Push Model

Server sends data to clients when it becomes available.

```
Push (Fan-out on Write):

Event occurs:
        ┌─────────────────────────────────────────┐
        │                                         │
        ▼                                         │
   ┌─────────┐                                    │
   │ Publisher│──────────────────────────────────┐│
   └─────────┘                                   ││
        │                                        ││
        │ Push to all subscribers               ││
        │                                        ▼▼
   ┌────┴────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
   │Subscriber│  │Subscriber│  │Subscriber│  │Subscriber│
   │    A    │  │    B    │  │    C    │  │    D    │
   └─────────┘  └─────────┘  └─────────┘  └─────────┘

   Mailbox A:  [new message]
   Mailbox B:  [new message]
   Mailbox C:  [new message]
   Mailbox D:  [new message]
```

### Pull Model

Clients request data when they need it.

```
Pull (Fan-in on Read):

When reader wants data:
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │Publisher│  │Publisher│  │Publisher│
   │    A    │  │    B    │  │    C    │
   └────┬────┘  └────┬────┘  └────┬────┘
        │            │            │
        │            │            │ Query all
        │            │            │
        └────────────┴────────────┘
                     │
                     ▼
              ┌─────────────┐
              │   Reader    │
              │ (aggregates)│
              └─────────────┘
```

## Comparison

| Aspect | Push | Pull |
|--------|------|------|
| **Latency** | Low (real-time) | Higher (polling interval) |
| **Write Cost** | High (fan-out) | Low (just store) |
| **Read Cost** | Low (pre-computed) | High (aggregate on read) |
| **Freshness** | Immediate | Depends on poll frequency |
| **Connections** | Many (to all subscribers) | Fewer (on-demand) |
| **Inactive Users** | Wasted work | No wasted work |
| **Hotspots** | Write hotspots | Read hotspots |

## Use Cases

### Push is Better For:

```
✅ Real-time requirements
   - Chat messages
   - Live sports scores
   - Stock tickers

✅ Few subscribers per publisher
   - Private messaging
   - Notification systems

✅ High read-to-write ratio
   - News feeds
   - Social timelines

✅ Subscribers are always active
   - IoT sensors
   - Monitoring systems
```

### Pull is Better For:

```
✅ Many followers per publisher
   - Celebrity accounts (millions of followers)
   - Viral content

✅ Infrequent reads
   - Historical data access
   - Reports and analytics

✅ Variable subscriber activity
   - Email (users check periodically)
   - Inactive users common

✅ Complex aggregation needed
   - Personalized feeds
   - Filtered/ranked content
```

## Twitter/Social Feed Example

### Pure Push (Fan-out on Write)

```
@celebrity posts tweet (10M followers):

┌─────────────────────────────────────────────────────────────────┐
│                    Fan-out on Write                              │
│                                                                  │
│  Tweet by @celebrity                                            │
│        │                                                        │
│        ▼                                                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Write to 10M timelines                        │  │
│  │                                                            │  │
│  │  User1 Timeline: [tweet]                                   │  │
│  │  User2 Timeline: [tweet]                                   │  │
│  │  User3 Timeline: [tweet]                                   │  │
│  │  ...                                                       │  │
│  │  User10M Timeline: [tweet]                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Write: O(n) where n = followers (10M writes!)                  │
│  Read: O(1) - just read pre-built timeline                      │
└─────────────────────────────────────────────────────────────────┘

Problem: Celebrity tweet = 10M writes = slow, expensive
```

### Pure Pull (Fan-in on Read)

```
User opens timeline:

┌─────────────────────────────────────────────────────────────────┐
│                    Fan-in on Read                                │
│                                                                  │
│  User follows 500 accounts                                      │
│        │                                                        │
│        ▼                                                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Query 500 tweet sources                       │  │
│  │                                                            │  │
│  │  Get tweets from @user1                                    │  │
│  │  Get tweets from @user2                                    │  │
│  │  Get tweets from @user3                                    │  │
│  │  ...                                                       │  │
│  │  Merge + Sort + Rank                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Write: O(1) - just store tweet                                 │
│  Read: O(n) where n = following count (500 queries per read!)   │
└─────────────────────────────────────────────────────────────────┘

Problem: Every timeline view = 500 queries = slow reads
```

### Hybrid Approach (What Twitter Does)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hybrid Approach                               │
│                                                                  │
│  Decision based on follower count:                              │
│                                                                  │
│  Regular user posts (< 10K followers):                          │
│  └── PUSH: Fan-out to followers' timelines                      │
│           Fast writes, pre-computed reads                        │
│                                                                  │
│  Celebrity posts (> 10K followers):                             │
│  └── PULL: Store in celebrity tweet cache                       │
│           Merge at read time for their followers                 │
│                                                                  │
│  Read timeline:                                                  │
│  └── Merge: Pre-built timeline + Celebrity tweets              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Timeline = (pushed tweets) ∪ (pulled celebrity tweets)  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Messaging Systems

### Push (WebSockets, Server-Sent Events)

```
Real-time Chat:

┌─────────────────────────────────────────────────────────────────┐
│                    WebSocket Push                                │
│                                                                  │
│  Server                                                         │
│    │                                                            │
│    │ Persistent connections                                     │
│    │                                                            │
│    ├────────────────────▶ Client A (WebSocket)                 │
│    ├────────────────────▶ Client B (WebSocket)                 │
│    └────────────────────▶ Client C (WebSocket)                 │
│                                                                  │
│  New message arrives:                                           │
│    │                                                            │
│    └── Immediately push to all connected clients                │
│                                                                  │
│  Pros: Real-time, low latency                                   │
│  Cons: Connection management, resource usage                    │
└─────────────────────────────────────────────────────────────────┘
```

### Pull (Polling)

```
Email-style:

┌─────────────────────────────────────────────────────────────────┐
│                    HTTP Polling                                  │
│                                                                  │
│  Client                          Server                         │
│    │                               │                            │
│    │ ── Any new messages? ───────▶ │                            │
│    │ ◀── No ────────────────────── │                            │
│    │                               │                            │
│    │ (wait 5 seconds)              │                            │
│    │                               │                            │
│    │ ── Any new messages? ───────▶ │                            │
│    │ ◀── Yes, here's 3 ─────────── │                            │
│    │                               │                            │
│                                                                  │
│  Pros: Simple, works everywhere, no connection management       │
│  Cons: Latency (up to poll interval), wasted requests           │
└─────────────────────────────────────────────────────────────────┘
```

### Long Polling (Hybrid)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Long Polling                                  │
│                                                                  │
│  Client                          Server                         │
│    │                               │                            │
│    │ ── Any new messages? ───────▶ │                            │
│    │                               │ (holds request)            │
│    │                               │ (waits for data)           │
│    │                               │                            │
│    │                               │ ← Message arrives          │
│    │ ◀── Yes, here's message ───── │                            │
│    │                               │                            │
│    │ ── Any new messages? ───────▶ │ (immediately reconnect)   │
│    │                               │                            │
│                                                                  │
│  Pros: Near real-time, works with HTTP                          │
│  Cons: Connection overhead, timeout handling                    │
└─────────────────────────────────────────────────────────────────┘
```

## Message Queues: Push vs Pull

### Push (Most message brokers)

```
RabbitMQ style:

┌─────────────────────────────────────────────────────────────────┐
│  Producer ──▶ Queue ──push──▶ Consumer                          │
│                                                                  │
│  Broker pushes messages to consumers as they arrive             │
│  Consumer must handle backpressure                              │
│                                                                  │
│  Pros: Low latency, simple consumer logic                       │
│  Cons: Consumer can be overwhelmed                              │
└─────────────────────────────────────────────────────────────────┘
```

### Pull (Kafka style)

```
Kafka style:

┌─────────────────────────────────────────────────────────────────┐
│  Producer ──▶ Topic ◀──pull──  Consumer                         │
│                                                                  │
│  Consumer pulls at its own pace                                 │
│  Consumer tracks its own offset                                 │
│                                                                  │
│  Pros: Consumer controls rate, replay possible                  │
│  Cons: Slightly higher latency, consumer complexity             │
└─────────────────────────────────────────────────────────────────┘
```

## Decision Framework

### Choose Push When:

```
┌─────────────────────────────────────────────────────────────────┐
│  □ Real-time requirements (< 1 second latency)                  │
│  □ Subscribers are mostly active                                │
│  □ Publisher has few subscribers                                │
│  □ High read-to-write ratio                                     │
│  □ Consistent subscription patterns                             │
└─────────────────────────────────────────────────────────────────┘
```

### Choose Pull When:

```
┌─────────────────────────────────────────────────────────────────┐
│  □ Publisher has many subscribers (fan-out problem)             │
│  □ Subscribers are often inactive                               │
│  □ Complex aggregation/filtering needed                         │
│  □ Consumer needs to control rate                               │
│  □ Need replay capability                                       │
└─────────────────────────────────────────────────────────────────┘
```

### Choose Hybrid When:

```
┌─────────────────────────────────────────────────────────────────┐
│  □ Mixed characteristics                                        │
│  □ Some publishers have few followers, some have millions       │
│  □ Need real-time for some, batch for others                    │
│  □ Complex social graph                                         │
└─────────────────────────────────────────────────────────────────┘
```

## Interview Talking Points

1. "Push has O(n) write cost, Pull has O(n) read cost - choose based on ratio"

2. "For social feeds, hybrid is often best - push for regular users, pull for celebrities"

3. "Push is great for real-time, but consider inactive users"

4. "Kafka uses pull because it gives consumers control and enables replay"

5. "WebSockets for real-time push, polling for simplicity"

## Common Interview Questions

1. **Q: How would you design a news feed?**
   A: Hybrid approach. Push tweets from regular users to followers' timelines. Pull tweets from celebrities/followed accounts with millions of followers at read time. Merge both for final feed.

2. **Q: What's the trade-off between push and pull?**
   A: Push trades write cost for read speed (good when reads >> writes). Pull trades read cost for write simplicity (good when writes >> reads or many inactive readers).

3. **Q: When would you use WebSockets vs polling?**
   A: WebSockets for real-time bidirectional (chat, gaming). Polling for simpler setups, less frequent updates, or when WebSockets aren't supported.

4. **Q: Why does Kafka use pull instead of push?**
   A: Consumer controls rate (backpressure), can replay messages, simpler broker logic, and consumers can process at their own pace.

## Key Takeaways

- Push: Fast reads, expensive writes (fan-out)
- Pull: Cheap writes, expensive reads (fan-in)
- Social feeds often use hybrid approaches
- Consider: activity patterns, follower counts, real-time needs
- Real-time = push (WebSockets, SSE)
- Batch/periodic = pull (polling)

## Further Reading

- Twitter engineering blog (feed architecture)
- Kafka design documentation
- "Designing Data-Intensive Applications" - Chapter 11
- Facebook TAO paper
