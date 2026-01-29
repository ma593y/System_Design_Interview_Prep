# Requirements Gathering

## Overview

Requirements gathering is the most critical phase of a system design interview. Getting this wrong leads to designing the wrong system.

## Types of Requirements

### Functional Requirements

**What the system should do.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    FUNCTIONAL REQUIREMENTS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Core Features:                                                  │
│  ├── What actions can users perform?                            │
│  ├── What data does the system handle?                          │
│  └── What are the main use cases?                               │
│                                                                  │
│  User Types:                                                     │
│  ├── Who uses the system?                                       │
│  ├── Different roles/permissions?                               │
│  └── Admin vs regular users?                                    │
│                                                                  │
│  Input/Output:                                                   │
│  ├── What data comes in?                                        │
│  ├── What data goes out?                                        │
│  └── What format?                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Non-Functional Requirements

**How well the system should perform.**

```
┌─────────────────────────────────────────────────────────────────┐
│                NON-FUNCTIONAL REQUIREMENTS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Scale:                                                          │
│  ├── Number of users (DAU/MAU)                                  │
│  ├── Requests per second                                        │
│  ├── Data volume                                                │
│  └── Growth rate                                                │
│                                                                  │
│  Performance:                                                    │
│  ├── Latency requirements                                       │
│  ├── Throughput needs                                           │
│  └── Response time SLAs                                         │
│                                                                  │
│  Availability:                                                   │
│  ├── Uptime requirements (99.9%?)                               │
│  ├── Disaster recovery                                          │
│  └── Maintenance windows?                                       │
│                                                                  │
│  Consistency:                                                    │
│  ├── Strong vs eventual?                                        │
│  ├── Read-after-write needed?                                   │
│  └── Conflict resolution                                        │
│                                                                  │
│  Other:                                                          │
│  ├── Security requirements                                      │
│  ├── Compliance (GDPR, HIPAA)                                   │
│  ├── Geographic distribution                                    │
│  └── Cost constraints                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Questions to Ask By Category

### Scale Questions

```
"How many users do we expect?"
├── Daily Active Users (DAU)
├── Monthly Active Users (MAU)
└── Peak concurrent users

"What's the request rate?"
├── Reads per second
├── Writes per second
└── Peak vs average ratio

"How much data?"
├── Data per user/item
├── Total storage needed
└── Data growth rate

"What's the read/write ratio?"
└── Read-heavy? Write-heavy? Balanced?
```

### Performance Questions

```
"What are the latency requirements?"
├── Average acceptable latency
├── P99 latency requirement
└── User-facing vs background tasks

"Any real-time requirements?"
├── Live updates needed?
├── Notification latency
└── Sync vs async OK?
```

### Availability Questions

```
"What's the uptime requirement?"
├── 99%? 99.9%? 99.99%?
├── Acceptable downtime per year
└── Maintenance windows allowed?

"Geographic requirements?"
├── Single region or global?
├── Data residency requirements
└── Disaster recovery needs
```

### Consistency Questions

```
"What's the consistency requirement?"
├── Strong consistency needed?
├── Eventual consistency OK?
└── Read-your-writes needed?

"How to handle conflicts?"
├── Last-write-wins?
├── Merge conflicts?
└── User resolution?
```

### Feature Questions

```
"What are the core features?"
├── Must-have features
├── Nice-to-have features
└── Out of scope

"Who are the users?"
├── End users
├── Admin users
├── API consumers (developers)

"What platforms?"
├── Web
├── Mobile (iOS/Android)
├── API-only
```

## Example: Gathering Requirements for URL Shortener

### Initial Question
"Design a URL shortener like bit.ly"

### Clarifying Questions & Answers

```
Functional Requirements:
┌─────────────────────────────────────────────────────────────────┐
│ Q: What are the core features?                                   │
│ A: - Shorten long URLs                                          │
│    - Redirect short URLs to original                            │
│    - Optional: analytics, custom aliases                        │
│                                                                  │
│ Q: Can users create custom short URLs?                          │
│ A: Yes, but system-generated is default                         │
│                                                                  │
│ Q: Do URLs expire?                                              │
│ A: Optional expiration, default is permanent                    │
│                                                                  │
│ Q: Any analytics needed?                                        │
│ A: Basic click counts for now                                   │
└─────────────────────────────────────────────────────────────────┘

Non-Functional Requirements:
┌─────────────────────────────────────────────────────────────────┐
│ Q: How many URLs shortened per day?                             │
│ A: 100 million new URLs/day                                     │
│                                                                  │
│ Q: Read/write ratio?                                            │
│ A: 100:1 (reads >> writes)                                      │
│                                                                  │
│ Q: Latency requirements?                                        │
│ A: Redirect must be < 100ms                                     │
│                                                                  │
│ Q: Availability requirement?                                    │
│ A: 99.9% uptime                                                 │
│                                                                  │
│ Q: How long to store URLs?                                      │
│ A: 10 years retention                                           │
│                                                                  │
│ Q: Global or single region?                                     │
│ A: Global users, need low latency worldwide                     │
└─────────────────────────────────────────────────────────────────┘
```

### Requirements Summary

```
┌─────────────────────────────────────────────────────────────────┐
│             URL SHORTENER REQUIREMENTS SUMMARY                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Functional:                                                     │
│  ✓ Shorten URLs (generate short code)                           │
│  ✓ Redirect short URL to original                               │
│  ✓ Custom aliases (optional)                                    │
│  ✓ URL expiration (optional)                                    │
│  ✓ Basic analytics (click count)                                │
│                                                                  │
│  Non-Functional:                                                 │
│  ✓ 100M new URLs/day (1,150 writes/sec)                        │
│  ✓ 10B redirects/day (115,000 reads/sec)                       │
│  ✓ < 100ms redirect latency                                     │
│  ✓ 99.9% availability                                           │
│  ✓ 10 year data retention                                       │
│  ✓ Global low-latency access                                    │
│                                                                  │
│  Out of Scope:                                                   │
│  ✗ User accounts (for v1)                                       │
│  ✗ Advanced analytics                                           │
│  ✗ API rate limiting (mention but don't deep dive)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Example: Gathering Requirements for Twitter

### Clarifying Questions & Answers

```
Functional:
┌─────────────────────────────────────────────────────────────────┐
│ Q: What features do we need to support?                         │
│ A: Post tweets, follow users, view timeline, like/retweet       │
│                                                                  │
│ Q: What content in tweets?                                      │
│ A: Text (280 chars), images, videos                             │
│                                                                  │
│ Q: What timelines?                                              │
│ A: Home timeline (followed users), user profile timeline        │
│                                                                  │
│ Q: Search functionality?                                        │
│ A: Out of scope for this design                                 │
│                                                                  │
│ Q: Notifications?                                               │
│ A: Basic (mentions, new followers) - don't deep dive            │
└─────────────────────────────────────────────────────────────────┘

Non-Functional:
┌─────────────────────────────────────────────────────────────────┐
│ Q: Number of users?                                             │
│ A: 500 million DAU                                              │
│                                                                  │
│ Q: Tweets per day?                                              │
│ A: Average user tweets twice/day = 1B tweets/day                │
│                                                                  │
│ Q: Timeline loads per day?                                      │
│ A: Each user loads 10x/day = 5B timeline loads                  │
│                                                                  │
│ Q: Timeline latency?                                            │
│ A: < 200ms                                                       │
│                                                                  │
│ Q: Consistency for timeline?                                    │
│ A: Eventual consistency is OK                                   │
│                                                                  │
│ Q: Follower limits?                                             │
│ A: Some users have millions of followers (celebrities)          │
│                                                                  │
│ Q: Geographic distribution?                                     │
│ A: Global users                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Requirements Gathering Template

Use this template for any system design:

```
┌─────────────────────────────────────────────────────────────────┐
│                    REQUIREMENTS TEMPLATE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  System: ___________________                                    │
│                                                                  │
│  FUNCTIONAL REQUIREMENTS                                        │
│  ─────────────────────────                                      │
│  Core Features:                                                 │
│  1. _________________________________                           │
│  2. _________________________________                           │
│  3. _________________________________                           │
│                                                                  │
│  User Types:                                                    │
│  1. _________________________________                           │
│  2. _________________________________                           │
│                                                                  │
│  Out of Scope:                                                  │
│  1. _________________________________                           │
│  2. _________________________________                           │
│                                                                  │
│  NON-FUNCTIONAL REQUIREMENTS                                    │
│  ───────────────────────────                                    │
│  Scale:                                                          │
│  • DAU: _____________                                           │
│  • Read QPS: _____________                                      │
│  • Write QPS: _____________                                     │
│  • Data volume: _____________                                   │
│                                                                  │
│  Performance:                                                    │
│  • Latency (avg): _____________                                 │
│  • Latency (p99): _____________                                 │
│                                                                  │
│  Availability:                                                   │
│  • Target: _____________                                        │
│                                                                  │
│  Consistency:                                                    │
│  • Type: □ Strong  □ Eventual  □ Read-your-writes              │
│                                                                  │
│  Other:                                                          │
│  • Global/Regional: _____________                               │
│  • Security: _____________                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Common Mistakes to Avoid

```
❌ DON'T:
• Jump straight into design
• Assume requirements
• Spend too long on requirements (5 min max)
• Ask about every possible feature
• Forget non-functional requirements

✅ DO:
• Always start with clarifying questions
• Write down the requirements
• Confirm understanding before proceeding
• Prioritize core features
• Get specific numbers for scale
```

## Key Non-Functional Requirements Checklist

| Requirement | Question to Ask | Why It Matters |
|-------------|-----------------|----------------|
| Scale | "How many users/requests?" | Determines architecture |
| Latency | "What response time is needed?" | Affects caching/design |
| Availability | "What uptime is required?" | Affects redundancy |
| Consistency | "Strong or eventual?" | Affects data model |
| Durability | "Can we lose data?" | Affects storage |
| Security | "What data is sensitive?" | Affects design choices |
| Cost | "Any budget constraints?" | Affects technology choices |

## Interview Talking Points

1. "Before I start designing, I'd like to clarify some requirements..."
2. "Let me make sure I understand the scale we're designing for..."
3. "Based on these requirements, the key challenges will be..."
4. "I'll focus on these core features and consider these as out of scope..."
5. "Given the latency requirements, we'll need to think about caching..."

## Key Takeaways

1. **Never skip requirements** - It's the foundation of your design
2. **Ask specific questions** - "Many users" is not a number
3. **Write them down** - Reference throughout the interview
4. **Confirm understanding** - Repeat back to interviewer
5. **Keep it to 5 minutes** - Don't spend too long here
6. **Prioritize** - Focus on must-haves first

## Further Reading

- "The Art of Asking Good Questions"
- Requirements engineering best practices
- Product requirement documents (PRDs)
