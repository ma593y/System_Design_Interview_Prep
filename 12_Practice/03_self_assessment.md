# Self-Assessment Checklist

A comprehensive evaluation framework to assess your system design interview readiness. Use this to identify gaps and track progress.

---

## How to Use This Assessment

1. **Be honest** - This is for your benefit, not to impress anyone
2. **Score each item** - Use the 0-3 scale consistently
3. **Calculate section scores** - Identify weak areas
4. **Create an action plan** - Focus on low-scoring areas
5. **Reassess regularly** - Track improvement over time

### Scoring Scale

| Score | Meaning |
|-------|---------|
| 0 | Cannot do this / No knowledge |
| 1 | Basic understanding, need significant improvement |
| 2 | Competent, can do with some gaps |
| 3 | Strong, can do confidently and explain to others |

---

## Section 1: Requirements Gathering (Max: 21 points)

How well can you clarify and scope a problem?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 1.1 | Identify functional requirements from a vague prompt | |
| 1.2 | Identify non-functional requirements (scale, latency, availability) | |
| 1.3 | Ask clarifying questions without being told to | |
| 1.4 | Make reasonable assumptions and state them clearly | |
| 1.5 | Prioritize features (MVP vs nice-to-have) | |
| 1.6 | Define success metrics for the system | |
| 1.7 | Scope appropriately for a 45-minute discussion | |

**Section 1 Total: ___ / 21**

### Self-Reflection Questions:
- When given "Design Twitter," what are the first 5 questions you would ask?
- How do you decide what's in scope vs out of scope?
- Can you articulate why latency vs throughput matters differently for different systems?

---

## Section 2: Capacity Estimation (Max: 18 points)

Can you estimate scale and resource needs?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 2.1 | Estimate daily/monthly active users from hints | |
| 2.2 | Calculate requests per second (read and write) | |
| 2.3 | Estimate storage requirements over time | |
| 2.4 | Calculate bandwidth needs | |
| 2.5 | Know order-of-magnitude numbers (latencies, throughput) | |
| 2.6 | Sanity check estimates against real-world systems | |

**Section 2 Total: ___ / 18**

### Key Numbers You Should Know:
- [ ] How many seconds in a day? (~86,400, round to 100K)
- [ ] Typical SSD read latency? (~100 microseconds)
- [ ] Network round trip within datacenter? (~0.5ms)
- [ ] Read 1MB from memory vs SSD vs network?
- [ ] Requests a single server can handle? (~10K-100K/sec depending on work)

---

## Section 3: High-Level Design (Max: 24 points)

Can you architect a system at the component level?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 3.1 | Draw clear system diagrams quickly | |
| 3.2 | Identify major components/services needed | |
| 3.3 | Define APIs between components | |
| 3.4 | Choose appropriate communication patterns (sync/async) | |
| 3.5 | Design for the stated scale, not over/under | |
| 3.6 | Explain data flow through the system | |
| 3.7 | Identify single points of failure | |
| 3.8 | Propose a design that actually works | |

**Section 3 Total: ___ / 24**

### Self-Reflection Questions:
- Can you draw a complete request flow in under 5 minutes?
- Do you naturally think about where data lives?
- Can you explain why you chose microservices vs monolith?

---

## Section 4: Database Design (Max: 27 points)

How well do you understand data storage?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 4.1 | Design relational schemas with proper normalization | |
| 4.2 | Know when to use SQL vs NoSQL | |
| 4.3 | Understand different NoSQL types (document, key-value, column, graph) | |
| 4.4 | Design partition keys for distributed databases | |
| 4.5 | Understand replication strategies | |
| 4.6 | Design indexes for query patterns | |
| 4.7 | Understand consistency models (strong, eventual, causal) | |
| 4.8 | Know when to use caching and cache strategies | |
| 4.9 | Handle data that doesn't fit in one machine | |

**Section 4 Total: ___ / 27**

### Can You Explain:
- [ ] What is consistent hashing and when to use it?
- [ ] Difference between read replicas and sharding?
- [ ] When would you choose Cassandra vs PostgreSQL vs MongoDB?
- [ ] What is a write-ahead log?
- [ ] Cache invalidation strategies?

---

## Section 5: Scalability Patterns (Max: 30 points)

Do you know patterns for handling scale?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 5.1 | Load balancing strategies | |
| 5.2 | Horizontal vs vertical scaling tradeoffs | |
| 5.3 | Caching at different layers (CDN, application, database) | |
| 5.4 | Asynchronous processing with message queues | |
| 5.5 | Database sharding strategies | |
| 5.6 | Read replicas for read-heavy workloads | |
| 5.7 | Connection pooling | |
| 5.8 | Rate limiting and throttling | |
| 5.9 | Batch processing vs stream processing | |
| 5.10 | CDN usage for static and dynamic content | |

**Section 5 Total: ___ / 30**

### Pattern Recognition:
For each scenario, what pattern would you use?
- 10x more reads than writes: _______________
- Expensive computation repeated often: _______________
- Need to process 1M events/second: _______________
- Users distributed globally: _______________
- Sudden traffic spikes: _______________

---

## Section 6: Reliability & Fault Tolerance (Max: 24 points)

Can you design systems that don't fail?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 6.1 | Understand availability percentages (99.9% vs 99.99%) | |
| 6.2 | Design for redundancy at each layer | |
| 6.3 | Implement health checks and circuit breakers | |
| 6.4 | Handle partial failures gracefully | |
| 6.5 | Design idempotent operations | |
| 6.6 | Understand consensus algorithms (Paxos, Raft) conceptually | |
| 6.7 | Plan for disaster recovery | |
| 6.8 | Implement retry strategies with backoff | |

**Section 6 Total: ___ / 24**

### Scenario Questions:
- What happens when your primary database goes down?
- How do you handle a failed payment mid-transaction?
- What is the CAP theorem and what does it mean for your design?

---

## Section 7: Specific System Knowledge (Max: 30 points)

Do you understand common system components?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 7.1 | How CDNs work | |
| 7.2 | How load balancers work (L4 vs L7) | |
| 7.3 | How message queues work (Kafka, RabbitMQ) | |
| 7.4 | How search engines work (inverted index) | |
| 7.5 | How caches work (Redis, Memcached) | |
| 7.6 | How DNS works | |
| 7.7 | How WebSockets work | |
| 7.8 | How blob storage works (S3) | |
| 7.9 | How distributed locks work | |
| 7.10 | How API gateways work | |

**Section 7 Total: ___ / 30**

---

## Section 8: Communication Skills (Max: 21 points)

Can you explain your design effectively?

| # | Skill | Score (0-3) |
|---|-------|-------------|
| 8.1 | Structure explanation logically (top-down) | |
| 8.2 | Adjust depth based on interviewer cues | |
| 8.3 | Use precise technical terminology | |
| 8.4 | Draw clear, readable diagrams | |
| 8.5 | Acknowledge tradeoffs without being asked | |
| 8.6 | Handle pushback and alternative suggestions | |
| 8.7 | Manage time effectively across phases | |

**Section 8 Total: ___ / 21**

### Communication Checklist:
- [ ] Do you think out loud or go silent while thinking?
- [ ] Do you ask "Does this make sense?" periodically?
- [ ] Can you explain complex concepts simply?
- [ ] Do you get defensive when challenged?

---

## Section 9: Problem-Specific Readiness (Max: 27 points)

Can you design these common systems?

| # | System | Score (0-3) |
|---|--------|-------------|
| 9.1 | URL Shortener | |
| 9.2 | Rate Limiter | |
| 9.3 | Key-Value Store | |
| 9.4 | Social Media Feed (Twitter/Instagram) | |
| 9.5 | Chat System (WhatsApp/Messenger) | |
| 9.6 | File Storage (Dropbox/Google Drive) | |
| 9.7 | Video Streaming (YouTube/Netflix) | |
| 9.8 | Ride Sharing (Uber/Lyft) | |
| 9.9 | Search System (Google/Elasticsearch) | |

**Section 9 Total: ___ / 27**

---

## Overall Score Summary

| Section | Your Score | Max Score | Percentage |
|---------|------------|-----------|------------|
| 1. Requirements Gathering | | 21 | |
| 2. Capacity Estimation | | 18 | |
| 3. High-Level Design | | 24 | |
| 4. Database Design | | 27 | |
| 5. Scalability Patterns | | 30 | |
| 6. Reliability & Fault Tolerance | | 24 | |
| 7. Specific System Knowledge | | 30 | |
| 8. Communication Skills | | 21 | |
| 9. Problem-Specific Readiness | | 27 | |
| **TOTAL** | | **222** | |

---

## Score Interpretation

| Total Score | Readiness Level | Recommendation |
|-------------|-----------------|----------------|
| 0-66 (0-30%) | Beginning | Focus on fundamentals. Study basic concepts before practicing problems. |
| 67-111 (30-50%) | Developing | Good foundation. Practice more problems and fill knowledge gaps. |
| 112-155 (50-70%) | Competent | Ready for interviews with more practice. Focus on weak sections. |
| 156-188 (70-85%) | Strong | Well-prepared. Polish communication and edge cases. |
| 189-222 (85-100%) | Expert | Excellent. Maintain skills and help others practice. |

---

## Gap Analysis

### Your Weakest Sections (Score < 50%):
1. _______________
2. _______________
3. _______________

### Priority Actions:
Based on your scores, here's what to focus on:

**If Requirements Gathering is low:**
- Practice with vague problem statements
- Write down clarifying questions for 10 different problems
- Study how experienced engineers scope problems

**If Capacity Estimation is low:**
- Memorize key latency numbers
- Practice back-of-envelope calculations daily
- Estimate capacity for apps you use

**If High-Level Design is low:**
- Draw architecture diagrams for systems you use
- Study published architectures (blog posts, papers)
- Practice going from requirements to diagram in 10 minutes

**If Database Design is low:**
- Take a database course or read a database book
- Practice schema design problems
- Learn when to use different database types

**If Scalability Patterns is low:**
- Study each pattern individually
- Implement small versions of patterns
- Read scaling stories from engineering blogs

**If Reliability is low:**
- Study distributed systems fundamentals
- Read post-mortems from major outages
- Practice failure scenario analysis

**If System Knowledge is low:**
- Deep dive into one technology per week
- Read documentation and architecture docs
- Build small projects using these technologies

**If Communication is low:**
- Practice explaining designs out loud
- Record yourself and review
- Do mock interviews with feedback

---

## Progress Tracker

Track your scores over time:

| Date | Total Score | Weakest Section | Notes |
|------|-------------|-----------------|-------|
| | | | |
| | | | |
| | | | |
| | | | |
| | | | |

---

## 30-Day Improvement Plan

### Week 1: Foundation
- [ ] Complete this assessment
- [ ] Review fundamentals for lowest-scoring sections
- [ ] Practice 2 beginner problems
- [ ] Memorize capacity estimation numbers

### Week 2: Building Skills
- [ ] Deep dive into database design
- [ ] Study 3 scalability patterns
- [ ] Practice 2 intermediate problems
- [ ] Record yourself solving a problem

### Week 3: Integration
- [ ] Practice 3 timed exercises
- [ ] Do 2 mock interviews with peers
- [ ] Study reliability patterns
- [ ] Reassess weak sections

### Week 4: Polish
- [ ] Practice 3 advanced problems
- [ ] Focus on communication refinement
- [ ] Do final mock interview
- [ ] Complete reassessment

---

## Ready for Interview Checklist

Before your interview, confirm you can:

- [ ] Clarify requirements for any vague problem
- [ ] Do capacity estimation in under 5 minutes
- [ ] Draw a high-level design in 10 minutes
- [ ] Justify database choices with tradeoffs
- [ ] Explain at least 5 scalability patterns
- [ ] Discuss failure scenarios proactively
- [ ] Design 5+ common systems confidently
- [ ] Communicate clearly while drawing
- [ ] Handle interviewer challenges gracefully
- [ ] Complete a full design in 45 minutes
