# Questions to Ask in System Design Interviews

## Overview

Asking the right questions demonstrates maturity and ensures you design the right system. This guide organizes questions by category and system type.

## Universal Questions (Ask for Every System)

### Scale Questions

```
┌─────────────────────────────────────────────────────────────────┐
│                    SCALE QUESTIONS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. "How many users do we need to support?"                     │
│     └── DAU, MAU, peak concurrent                               │
│                                                                  │
│  2. "What's the expected request rate?"                         │
│     └── QPS, peak vs average                                    │
│                                                                  │
│  3. "How much data will we store?"                              │
│     └── Per item, total, growth rate                            │
│                                                                  │
│  4. "What's the read/write ratio?"                              │
│     └── Read-heavy, write-heavy, balanced                       │
│                                                                  │
│  5. "What's the expected growth?"                               │
│     └── 10x in a year? Gradual?                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Performance Questions

```
┌─────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE QUESTIONS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. "What are the latency requirements?"                        │
│     └── Average, P99, user-facing vs background                 │
│                                                                  │
│  2. "Is real-time critical?"                                    │
│     └── Seconds delay OK? Minutes?                              │
│                                                                  │
│  3. "What's the availability target?"                           │
│     └── 99.9%? 99.99%?                                          │
│                                                                  │
│  4. "Are there SLAs we need to meet?"                           │
│     └── Contractual obligations                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Feature Questions

```
┌─────────────────────────────────────────────────────────────────┐
│                    FEATURE QUESTIONS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. "What are the must-have features?"                          │
│     └── Core functionality                                      │
│                                                                  │
│  2. "What can we defer to v2?"                                  │
│     └── Nice-to-haves                                           │
│                                                                  │
│  3. "Who are the users?"                                        │
│     └── End users, admins, developers                           │
│                                                                  │
│  4. "What platforms need support?"                              │
│     └── Web, mobile, API                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Consistency Questions

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY QUESTIONS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. "Do we need strong consistency?"                            │
│     └── Or is eventual OK?                                      │
│                                                                  │
│  2. "Does the user need to see their own writes immediately?"   │
│     └── Read-your-writes consistency                            │
│                                                                  │
│  3. "How important is data accuracy?"                           │
│     └── Financial data vs social feed                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Questions By System Type

### Social Media / Feed Systems

```
Questions for Twitter, Instagram, Facebook:

□ "How many users and what's the average follow count?"
□ "What's the distribution of followers?" (celebrities vs regular)
□ "How quickly must new posts appear in followers' feeds?"
□ "Do we need real-time notifications?"
□ "What content types?" (text, images, video)
□ "Is the feed chronological or ranked?"
□ "Do we need direct messaging?"
```

### Messaging Systems

```
Questions for WhatsApp, Slack:

□ "One-to-one or group messaging?"
□ "Maximum group size?"
□ "Do messages need to be persisted forever?"
□ "Do we need delivery receipts? Read receipts?"
□ "Encryption requirements?"
□ "Offline message delivery?"
□ "Message ordering guarantees?"
□ "Media sharing?" (images, files, voice)
```

### E-commerce Systems

```
Questions for Amazon, Shopify:

□ "How many products in the catalog?"
□ "Order volume per day?"
□ "Do prices change frequently?"
□ "Inventory tracking across warehouses?"
□ "Payment integration scope?"
□ "Return/refund handling?"
□ "Search functionality depth?"
□ "Recommendation system needed?"
```

### Video/Streaming Systems

```
Questions for YouTube, Netflix:

□ "User-generated or licensed content?"
□ "Video quality levels needed?"
□ "Live streaming or VOD only?"
□ "Upload volume per day?"
□ "Average video length/size?"
□ "Global CDN requirements?"
□ "Offline viewing support?"
□ "Recommendation system?"
```

### Search Systems

```
Questions for Search Engines:

□ "How large is the corpus to search?"
□ "Update frequency of data?"
□ "Type of search?" (full-text, structured, fuzzy)
□ "Autocomplete/suggestions needed?"
□ "Relevance ranking complexity?"
□ "Personalization requirements?"
□ "Query latency requirements?"
```

### Location-Based Systems

```
Questions for Uber, DoorDash:

□ "How many active drivers/couriers?"
□ "Location update frequency?"
□ "Matching algorithm requirements?"
□ "ETA calculation needs?"
□ "Surge pricing?"
□ "Geographic coverage?"
□ "Offline operation?"
```

### Storage Systems

```
Questions for Dropbox, Google Drive:

□ "File size limits?"
□ "Total storage per user?"
□ "Versioning required?"
□ "Sharing/collaboration features?"
□ "Sync across devices?"
□ "Conflict resolution for concurrent edits?"
□ "Deduplication needed?"
```

### Real-Time Systems

```
Questions for Gaming, Trading:

□ "Latency tolerance?"
□ "Concurrent users in same session?"
□ "State synchronization needs?"
□ "Anti-cheat considerations?"
□ "Replay/history requirements?"
□ "Matchmaking complexity?"
```

## Questions By Technical Area

### Database Design

```
□ "What are the main entities and relationships?"
□ "What queries will be most common?"
□ "Are there any hot keys/partitions?"
□ "What's the write pattern?" (append, update, delete)
□ "Do we need full-text search?"
□ "ACID requirements?"
□ "Backup/recovery needs?"
```

### Caching

```
□ "What data is cacheable?"
□ "Cache invalidation strategy?"
□ "TTL requirements?"
□ "Cache-through vs cache-aside?"
□ "Local vs distributed cache?"
□ "Cold start handling?"
```

### APIs

```
□ "Who are the API consumers?"
□ "Backward compatibility requirements?"
□ "Rate limiting needs?"
□ "Authentication method?"
□ "Synchronous or async responses?"
□ "Pagination requirements?"
□ "Webhook support?"
```

### Security

```
□ "What data is sensitive?"
□ "Encryption requirements?" (at rest, in transit)
□ "Compliance requirements?" (GDPR, HIPAA, PCI)
□ "Authentication mechanism?"
□ "Authorization model?"
□ "Audit logging needs?"
```

## Question Flow Template

```
Step 1: Understand the product
├── "What problem are we solving?"
├── "Who are the users?"
└── "What are the core features?"

Step 2: Understand the scale
├── "How many users?"
├── "How many requests/sec?"
├── "How much data?"
└── "What's the growth expectation?"

Step 3: Understand the performance
├── "What latency is acceptable?"
├── "What availability is required?"
└── "Any real-time requirements?"

Step 4: Understand the constraints
├── "Strong or eventual consistency?"
├── "Any compliance requirements?"
├── "Budget or technology constraints?"
└── "What's out of scope?"
```

## Question Cheat Sheet (Quick Reference)

| Category | Quick Questions |
|----------|-----------------|
| Scale | "How many users? QPS? Data size?" |
| Performance | "Latency target? Availability?" |
| Features | "Core features? Out of scope?" |
| Consistency | "Strong or eventual?" |
| Data | "Read/write ratio? Retention?" |
| Distribution | "Global or regional?" |
| Security | "Sensitive data? Compliance?" |

## Questions NOT to Ask

```
❌ Don't ask obvious questions:
   "Should it be fast?" (obviously yes)
   "Should it be reliable?" (obviously yes)

❌ Don't ask about implementation details early:
   "Should we use Redis or Memcached?" (too early)
   "How many servers?" (figure out from requirements)

❌ Don't ask about every possible feature:
   Focus on core requirements, not edge cases
```

## How Interviewers View Questions

### Strong Signal Questions

```
✓ "What's our consistency requirement for this use case?"
   → Shows understanding of trade-offs

✓ "What's the read/write ratio we should optimize for?"
   → Shows system design thinking

✓ "Are there any hot keys or uneven access patterns?"
   → Shows experience with scaling

✓ "What happens if this component fails?"
   → Shows production mindset
```

### Weak Signal Questions

```
✗ "What technology should I use?"
   → Should evaluate and recommend

✗ Generic questions without context
   → Should be specific to the problem

✗ Too many questions without synthesis
   → Should use answers to drive design
```

## Interview Talking Points

1. "Before I design, I want to understand the requirements..."
2. "The scale you mentioned will affect our architecture choices..."
3. "Given the latency requirements, we'll need to consider..."
4. "Let me confirm I understand the core features..."
5. "Based on what you've told me, I'll focus on..."

## Key Takeaways

1. **Ask purposefully** - Each question should inform your design
2. **Listen actively** - Interviewers often drop hints
3. **Synthesize answers** - Connect requirements to design decisions
4. **Keep it brief** - 5 minutes max for requirements
5. **Write things down** - Reference throughout the interview
6. **Prioritize** - Focus on what affects your design most

## Further Reading

- "Interviewing for System Design" articles
- STAR method for behavioral questions
- Product requirements documents
