# System Design Interview Approach

## Overview

A structured approach is essential for system design interviews. This guide presents the **RESHADED** framework for a 45-minute interview.

## Time Management

```
┌─────────────────────────────────────────────────────────────────┐
│                    45-Minute Interview                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Requirements (5 min)     ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
│  Estimation (5 min)       ░░░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
│  High-Level Design (10)   ░░░░░░░░████████████░░░░░░░░░░░░░░░  │
│  Deep Dive (15 min)       ░░░░░░░░░░░░░░░░░░░░████████████████  │
│  Wrap-up (5 min)          ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## The RESHADED Framework

### R - Requirements Clarification (5 minutes)

**Goal:** Understand what you're building and its constraints.

```
┌─────────────────────────────────────────────────────────────────┐
│                    REQUIREMENTS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Functional Requirements (What the system does)                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • Core features the system must support                    │  │
│  │ • User actions and flows                                   │  │
│  │ • Input/output expectations                                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Non-Functional Requirements (How well it does it)              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • Scale: Users, requests per second, data volume          │  │
│  │ • Performance: Latency requirements                       │  │
│  │ • Availability: SLA, downtime tolerance                   │  │
│  │ • Consistency: Strong vs eventual                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Questions to Ask:**
- "What are the main features we need to support?"
- "Who are the users? How many?"
- "What's the expected scale (DAU, requests/sec)?"
- "What are the latency requirements?"
- "Do we need to support global users?"
- "What's more important: consistency or availability?"

**Example:** Design Twitter
```
Functional:
✓ Post tweets (text, images)
✓ Follow users
✓ View timeline (home + user)
✓ Like/retweet

Non-Functional:
✓ 500M DAU
✓ Timeline load < 200ms
✓ 99.9% availability
✓ Eventual consistency OK for timeline
```

### E - Estimation (5 minutes)

**Goal:** Size the system with back-of-envelope calculations.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTIMATION                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Traffic:                                                        │
│  ├── Daily Active Users (DAU)                                   │
│  ├── Requests per second (QPS)                                  │
│  └── Peak vs Average ratio                                      │
│                                                                  │
│  Storage:                                                        │
│  ├── Data size per item                                         │
│  ├── Items per day/year                                         │
│  └── Total storage needed                                       │
│                                                                  │
│  Bandwidth:                                                      │
│  ├── Request size × QPS                                         │
│  └── Inbound vs Outbound                                        │
│                                                                  │
│  Servers:                                                        │
│  └── QPS / (capacity per server)                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Example:** Twitter Estimation
```
Users: 500M DAU
Tweets/day: 500M users × 2 tweets/user = 1B tweets
Tweet size: 280 chars + metadata ≈ 500 bytes

Storage/day: 1B × 500 bytes = 500 GB
Storage/year: 500 GB × 365 = 180 TB

QPS: 1B tweets / 86,400 sec ≈ 12,000 writes/sec
Read QPS: 10x writes = 120,000 reads/sec
Peak: 3x average = 360,000 reads/sec
```

### S - System Interface / API Design (3 minutes)

**Goal:** Define how clients interact with the system.

```
┌─────────────────────────────────────────────────────────────────┐
│                    API DESIGN                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  POST /api/v1/tweets                                            │
│  Body: { content: string, media_ids: [string] }                 │
│  Response: { tweet_id: string, created_at: timestamp }          │
│                                                                  │
│  GET /api/v1/timeline?user_id=123&cursor=abc&limit=20          │
│  Response: { tweets: [...], next_cursor: string }               │
│                                                                  │
│  POST /api/v1/users/{id}/follow                                 │
│  Response: { success: boolean }                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### H - High-Level Design (10 minutes)

**Goal:** Draw the big picture with core components.

```
┌─────────────────────────────────────────────────────────────────┐
│                    HIGH-LEVEL DESIGN                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────┐    ┌─────────────┐    ┌──────────────┐             │
│  │ Client │───▶│ Load        │───▶│ API Gateway  │             │
│  └────────┘    │ Balancer    │    └──────┬───────┘             │
│                └─────────────┘           │                      │
│                                          │                      │
│        ┌─────────────────────────────────┼──────────────┐       │
│        │                                 │              │       │
│        ▼                                 ▼              ▼       │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐              │
│  │ Tweet     │    │ Timeline  │    │ User      │              │
│  │ Service   │    │ Service   │    │ Service   │              │
│  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘              │
│        │                │                │                      │
│        ▼                ▼                ▼                      │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐              │
│  │ Tweet DB  │    │ Timeline  │    │ User DB   │              │
│  │           │    │ Cache     │    │           │              │
│  └───────────┘    └───────────┘    └───────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Walk through the request flow:
1. Client sends request to Load Balancer
2. LB routes to API Gateway
3. Gateway authenticates and routes to appropriate service
4. Service processes and returns response
```

### A - Architecture Deep Dive (15 minutes)

**Goal:** Dive into 2-3 critical components in detail.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEEP DIVE AREAS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Data Model & Database Choice                                │
│     ├── Schema design                                           │
│     ├── SQL vs NoSQL decision                                   │
│     ├── Sharding strategy                                       │
│     └── Indexing                                                │
│                                                                  │
│  2. Core Algorithm/Logic                                        │
│     ├── How does timeline generation work?                      │
│     ├── Push vs Pull trade-offs                                 │
│     └── Ranking algorithm                                       │
│                                                                  │
│  3. Scaling & Performance                                       │
│     ├── Caching strategy                                        │
│     ├── Replication                                             │
│     └── Geographic distribution                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Example Deep Dive: Timeline Service**
```
Timeline Generation Options:

Option 1: Fan-out on Write (Push)
┌─────────────────────────────────────────────────────────────────┐
│  User posts tweet                                                │
│        │                                                         │
│        ▼                                                         │
│  For each follower:                                             │
│     Write tweet_id to follower's timeline cache                 │
│                                                                  │
│  Pros: Fast reads, pre-computed                                 │
│  Cons: Expensive for users with millions of followers           │
└─────────────────────────────────────────────────────────────────┘

Option 2: Fan-out on Read (Pull)
┌─────────────────────────────────────────────────────────────────┐
│  User requests timeline                                          │
│        │                                                         │
│        ▼                                                         │
│  Fetch tweets from all followed users                           │
│  Merge and rank                                                  │
│                                                                  │
│  Pros: Simple write, works for celebrities                      │
│  Cons: Slow reads, compute on every request                     │
└─────────────────────────────────────────────────────────────────┘

Our Solution: Hybrid
- Push for regular users (< 10K followers)
- Pull for celebrities (> 10K followers)
- Merge at read time
```

### D - Design Trade-offs (included in deep dive)

**Goal:** Discuss alternatives and justify decisions.

```
For every major decision, articulate:
1. What options exist?
2. What are the trade-offs?
3. Why did you choose this option?
4. What would change with different requirements?
```

### E - Edge Cases & Error Handling (3 minutes)

**Goal:** Show production-readiness thinking.

```
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE CASES                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Failure Scenarios:                                             │
│  ├── What if the database is down?                              │
│  ├── What if a service is overloaded?                           │
│  ├── What if there's a network partition?                       │
│  └── What if a celebrity tweets? (thundering herd)              │
│                                                                  │
│  Security:                                                       │
│  ├── Authentication/Authorization                               │
│  ├── Rate limiting                                              │
│  └── Data validation                                            │
│                                                                  │
│  Data Issues:                                                    │
│  ├── Duplicate submissions                                      │
│  ├── Data corruption                                            │
│  └── Consistency issues                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### D - Discussion & Wrap-up (5 minutes)

**Goal:** Summarize and discuss improvements.

```
Summary:
1. Recap the main components
2. Highlight key design decisions
3. Mention what you'd do with more time

Future Improvements:
- Analytics pipeline
- Search functionality
- Content moderation
- Internationalization
```

## Signals Interviewers Look For

### Strong Signals ✓

| Signal | How to Demonstrate |
|--------|-------------------|
| Requirements gathering | Ask clarifying questions before designing |
| Trade-off analysis | "Option A gives us X, but at the cost of Y" |
| Scalability thinking | Consider 10x, 100x growth |
| Practical experience | "In my experience..." |
| Communication | Walk through your design clearly |
| Flexibility | Adapt when interviewer redirects |

### Red Flags ✗

| Red Flag | How to Avoid |
|----------|-------------|
| Jumping to solution | Always start with requirements |
| One-sided thinking | Always discuss trade-offs |
| Ignoring scale | Do back-of-envelope calculations |
| Over-engineering | Start simple, add complexity as needed |
| Not listening | Pay attention to interviewer hints |
| Silence | Think out loud, communicate |

## Communication Tips

```
DO:
✓ "Let me start by clarifying requirements..."
✓ "Based on our scale requirements, we need..."
✓ "There's a trade-off here between X and Y..."
✓ "One challenge with this approach is..."
✓ "Does this align with what you had in mind?"

DON'T:
✗ "Obviously, we use [technology]..."
✗ "This is the only way to do it..."
✗ "I don't know anything about that..."
✗ Long silences without explanation
```

## Quick Checklist

Before wrapping up, ensure you've covered:

```
□ Clarified requirements (functional + non-functional)
□ Did capacity estimation
□ Drew high-level architecture
□ Defined data model
□ Discussed at least one deep-dive topic
□ Addressed scalability
□ Mentioned caching strategy
□ Discussed failure scenarios
□ Articulated trade-offs
□ Left time for questions
```

## Key Takeaways

1. **Structure is key** - Follow a framework
2. **Requirements first** - Don't skip this step
3. **Think out loud** - Show your reasoning
4. **Trade-offs matter** - There's no perfect solution
5. **Drive the conversation** - Be proactive
6. **Listen to hints** - Interviewer is guiding you

## Further Reading

- "System Design Interview" by Alex Xu
- "Designing Data-Intensive Applications"
- Company engineering blogs
- Previous interview experiences
