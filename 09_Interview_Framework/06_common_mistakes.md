# Common Mistakes in System Design Interviews

## Overview

Avoiding common pitfalls can significantly improve your system design interview performance. This guide covers the most frequent mistakes and how to avoid them.

## Top 10 Mistakes

### 1. Jumping to Solution Without Requirements

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #1                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  Interviewer: "Design Twitter"                                  │
│  Candidate: "OK, we'll use Cassandra for storing tweets,        │
│              Redis for caching, and Kafka for messaging..."     │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  Candidate: "Before I start designing, I'd like to clarify      │
│              some requirements. What features should we focus   │
│              on? What scale are we designing for?..."           │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  WHY IT MATTERS:                                                │
│  - You might solve the wrong problem                            │
│  - Shows lack of product thinking                               │
│  - Miss important constraints                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Not Doing Back-of-Envelope Calculations

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #2                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  "We'll have a lot of data, so we need sharding..."             │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  "With 500M DAU and 2 tweets per user per day:                  │
│   - 1B tweets/day                                               │
│   - ~12,000 writes/sec average                                  │
│   - 500 bytes/tweet = 500GB/day storage                         │
│   - Read QPS: ~10x writes = 120,000 reads/sec                   │
│                                                                  │
│   Given these numbers, we need..."                              │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  WHY IT MATTERS:                                                │
│  - Justifies your design decisions                              │
│  - Shows quantitative thinking                                   │
│  - Helps identify bottlenecks                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 3. One-Dimensional Design (No Trade-offs)

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #3                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  "We'll use MongoDB because it's the best database"             │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  "For the timeline data, I'm choosing Cassandra over MySQL.     │
│                                                                  │
│   Cassandra pros:                                               │
│   - Excellent write performance                                 │
│   - Easy horizontal scaling                                     │
│   - Tunable consistency                                         │
│                                                                  │
│   Cassandra cons:                                               │
│   - No JOINs (we'll denormalize)                               │
│   - Eventual consistency (acceptable for feed)                  │
│                                                                  │
│   Given our write-heavy workload and availability              │
│   requirements, these trade-offs work for us."                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Ignoring Scale

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #4                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  Design that works for 1,000 users but breaks at 1,000,000     │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  "This design works at current scale. As we grow to 10x:        │
│   - We'd need to shard the database by user_id                 │
│   - Add more cache nodes                                        │
│   - Consider CDN for static content                             │
│   - May need to move to async processing for [feature]"        │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  ALWAYS CONSIDER:                                               │
│  - What breaks at 10x scale?                                    │
│  - What breaks at 100x scale?                                   │
│  - What's the bottleneck?                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 5. Over-Engineering

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #5                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  "We'll have 50 microservices, Kubernetes, service mesh,       │
│   multiple data centers, ML ranking, blockchain..."             │
│   (for a URL shortener with 1000 users)                        │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  "For our current scale, a simple setup works:                  │
│   - Single database with read replica                           │
│   - Basic caching layer                                         │
│   - Standard load balancer                                      │
│                                                                  │
│   As we scale, we'd evolve to..."                              │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  PRINCIPLE:                                                      │
│  Start simple, add complexity only when needed                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6. Forgetting About Data

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #6                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  Drawing services and APIs but not discussing:                  │
│  - Data model                                                   │
│  - Schema design                                                │
│  - How data flows through the system                           │
│  - Storage requirements                                         │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  "Let me design the data model:                                 │
│                                                                  │
│   Users table:                                                  │
│   - id (PK), username, email, created_at                       │
│                                                                  │
│   Tweets table:                                                 │
│   - id (PK), user_id (FK), content, created_at                 │
│   - Indexed on: user_id, created_at                            │
│                                                                  │
│   Follows table:                                                │
│   - follower_id, followee_id, created_at                       │
│   - Composite PK: (follower_id, followee_id)                   │
│                                                                  │
│   For the timeline, we'll denormalize..."                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7. Not Handling Failure Scenarios

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #7                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  Designing only for the happy path                              │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  "Let me address failure scenarios:                             │
│                                                                  │
│   Database failure:                                             │
│   - Automatic failover to read replica                         │
│   - Connections pool retry with exponential backoff            │
│                                                                  │
│   Cache failure:                                                │
│   - Fall back to database (graceful degradation)               │
│   - Circuit breaker to prevent cascade                         │
│                                                                  │
│   Service failure:                                              │
│   - Health checks and auto-restart                             │
│   - Load balancer removes unhealthy nodes                      │
│   - Rate limiting to protect from overload                     │
│                                                                  │
│   Network partition:                                            │
│   - Choose availability (serve stale data)                     │
│   - Or consistency (return errors until resolved)"             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8. Silence / Not Communicating

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #8                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  *Long silence while thinking*                                  │
│  *Drawing without explaining*                                   │
│  *Not checking in with interviewer*                             │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  "Let me think about this for a moment..."                      │
│  *Brief pause*                                                   │
│  "I'm considering two approaches:                               │
│   1. Push model where we fan out on write                       │
│   2. Pull model where we aggregate on read                      │
│                                                                  │
│   For our use case with celebrity users having millions         │
│   of followers, I'm leaning toward a hybrid approach..."       │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  REMEMBER:                                                       │
│  The interviewer can't give you credit for thoughts            │
│  they can't hear                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 9. Not Listening to Hints

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #9                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  Interviewer: "What about handling hot partitions?"             │
│  Candidate: *continues with original design, ignoring hint*    │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  Candidate: "That's a great point about hot partitions.        │
│              If a celebrity has millions of followers,          │
│              their user_id becomes a hot key.                   │
│                                                                  │
│              To handle this, we could:                          │
│              1. Add a random suffix to spread load             │
│              2. Separate hot users to dedicated shards          │
│              3. Use a different write path for celebrities"    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  INTERVIEWER HINTS OFTEN INDICATE:                              │
│  - Something they want you to address                           │
│  - A flaw in your current design                               │
│  - An opportunity to show depth                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10. Poor Time Management

```
┌─────────────────────────────────────────────────────────────────┐
│                    MISTAKE #10                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ WRONG:                                                       │
│  - 20 minutes on requirements                                   │
│  - 20 minutes on high-level design                             │
│  - Run out of time before deep dive                            │
│                                                                  │
│  ✅ RIGHT:                                                       │
│  - 5 minutes: Requirements                                      │
│  - 5 minutes: Estimation                                        │
│  - 10 minutes: High-level design                               │
│  - 15 minutes: Deep dive on 2-3 components                     │
│  - 5 minutes: Trade-offs and wrap-up                           │
│  - 5 minutes: Questions                                         │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  TIP: Set mental checkpoints                                    │
│  "I should have high-level done by 15 minutes in"              │
└─────────────────────────────────────────────────────────────────┘
```

## Additional Mistakes to Avoid

### Technical Mistakes

| Mistake | Why It's Bad | How to Avoid |
|---------|--------------|--------------|
| Single point of failure | System unreliable | Always add redundancy for critical paths |
| No caching strategy | Poor performance | Consider caching at every layer |
| Ignoring consistency model | Data issues | Explicitly choose strong/eventual |
| Wrong database choice | Won't scale | Match DB to access patterns |
| No API versioning | Breaking changes | Include v1 in API design |

### Process Mistakes

| Mistake | Why It's Bad | How to Avoid |
|---------|--------------|--------------|
| Not drawing diagrams | Hard to follow | Always draw your architecture |
| Solutioning without trade-offs | Seems naive | Every decision has alternatives |
| Defensive when questioned | Bad signal | Welcome feedback, iterate |
| Too much detail too early | Lose big picture | Start broad, go deep when asked |
| Not asking for clarification | Wrong assumptions | Ask when uncertain |

## Mistake-Avoidance Checklist

Before moving to next phase, check:

### After Requirements (5 min)
```
□ Understood core features
□ Got specific scale numbers
□ Clarified consistency requirements
□ Know what's in/out of scope
```

### After High-Level Design (15 min)
```
□ Drew a diagram
□ Identified all major components
□ Showed data flow
□ Mentioned scaling approach
```

### After Deep Dive (30 min)
```
□ Designed data model
□ Discussed key algorithms
□ Addressed at least 2 components in depth
□ Considered failure scenarios
```

### Before Finishing (40 min)
```
□ Summarized trade-offs
□ Mentioned monitoring/observability
□ Discussed future improvements
□ Left time for questions
```

## Red Flags From Interviewer's View

```
🚩 Never asks clarifying questions
🚩 Can't estimate scale
🚩 Only one approach (no alternatives)
🚩 Ignores interviewer's hints
🚩 Can't explain trade-offs
🚩 Design breaks at scale
🚩 No failure handling
🚩 Silent for long periods
🚩 Over-complicated for requirements
🚩 Under-designed for scale
```

## Recovery Strategies

### When You Realize a Mistake

```
"Actually, I want to reconsider this part.
Looking at it now, [original approach] won't work because
[reason]. Let me revise to [new approach]..."
```

### When Interviewer Points Out a Flaw

```
"You're right, I hadn't considered [issue].
To address that, we could [solution].
This would require [trade-off], but given our requirements,
that's acceptable because [reasoning]."
```

### When You're Stuck

```
"I'm not immediately sure how to handle [problem].
My instinct is to [partial solution].
Could you give me a hint about the direction
you'd like me to explore?"
```

## Key Takeaways

1. **Requirements first** - Never skip this step
2. **Quantify everything** - Use numbers, not vague terms
3. **Trade-offs matter** - Every decision has alternatives
4. **Think about failure** - Happy path isn't enough
5. **Communicate constantly** - Silence is your enemy
6. **Listen to hints** - Interviewer is guiding you
7. **Watch the clock** - Pace yourself

## Further Reading

- Mock interview recordings (Pramp, Interviewing.io)
- Company engineering blogs for real designs
- "System Design Interview" by Alex Xu
