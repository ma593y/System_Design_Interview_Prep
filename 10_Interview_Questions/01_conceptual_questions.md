# Conceptual Questions for System Design Interviews

This document contains theory and concept questions commonly asked in system design interviews, along with detailed answers and what interviewers are looking for.

---

## Table of Contents

1. [CAP Theorem](#1-cap-theorem)
2. [Consistency Models](#2-consistency-models)
3. [ACID vs BASE](#3-acid-vs-base)
4. [Horizontal vs Vertical Scaling](#4-horizontal-vs-vertical-scaling)
5. [Database Sharding](#5-database-sharding)
6. [Caching Strategies](#6-caching-strategies)
7. [Load Balancing Algorithms](#7-load-balancing-algorithms)
8. [Microservices vs Monolith](#8-microservices-vs-monolith)
9. [Message Queues](#9-message-queues)
10. [SQL vs NoSQL](#10-sql-vs-nosql)
11. [Replication Strategies](#11-replication-strategies)
12. [Consensus Algorithms](#12-consensus-algorithms)
13. [CDN Architecture](#13-cdn-architecture)
14. [API Design](#14-api-design)
15. [Rate Limiting](#15-rate-limiting)
16. [Idempotency](#16-idempotency)
17. [Event-Driven Architecture](#17-event-driven-architecture)
18. [Data Partitioning](#18-data-partitioning)

---

## 1. CAP Theorem

### Question
**"Explain the CAP theorem and how it applies to distributed system design."**

### Detailed Answer

The CAP theorem, proposed by Eric Brewer, states that a distributed system can only guarantee two out of three properties simultaneously:

**Consistency (C):** Every read receives the most recent write or an error. All nodes see the same data at the same time.

**Availability (A):** Every request receives a non-error response, without guarantee that it contains the most recent write. The system remains operational 100% of the time.

**Partition Tolerance (P):** The system continues to operate despite network partitions (communication breakdowns between nodes).

**Why You Can't Have All Three:**
In a distributed system, network partitions are inevitable. When a partition occurs, you must choose between:
- **CP (Consistency + Partition Tolerance):** Reject requests to maintain consistency (e.g., banking systems)
- **AP (Availability + Partition Tolerance):** Return potentially stale data to remain available (e.g., social media feeds)

**Real-World Examples:**
- **CP Systems:** HBase, MongoDB (in certain configurations), Zookeeper
- **AP Systems:** Cassandra, DynamoDB, CouchDB
- **CA Systems:** Traditional RDBMS (but only in single-node, non-distributed setups)

**Modern Interpretation (PACELC):**
The PACELC theorem extends CAP: "If there is a Partition, choose between Availability and Consistency; Else, when the system is running normally, choose between Latency and Consistency."

### What Interviewers Are Looking For
- Understanding that partition tolerance is non-negotiable in distributed systems
- Ability to reason about trade-offs in real scenarios
- Knowledge of which systems fall into which category
- Awareness of the PACELC extension

### Key Points to Mention
- Network partitions are unavoidable in distributed systems
- The choice is really between CP and AP
- Different parts of a system can make different CAP trade-offs
- Consistency can be tunable (eventual vs strong)

### Common Follow-Ups
- "How would you handle a network partition in a payment system?"
- "Can you give an example of when you'd choose AP over CP?"
- "What is eventual consistency and when is it acceptable?"

---

## 2. Consistency Models

### Question
**"Explain the different consistency models in distributed systems. When would you use each?"**

### Detailed Answer

**Strong Consistency:**
- Every read reflects the most recent write
- All clients see the same data at the same time
- Achieved through synchronous replication
- **Use case:** Financial transactions, inventory management
- **Trade-off:** Higher latency, reduced availability

**Eventual Consistency:**
- If no new updates are made, eventually all reads will return the last updated value
- Temporary inconsistencies are acceptable
- **Use case:** Social media feeds, DNS, shopping carts
- **Trade-off:** Users may see stale data temporarily

**Causal Consistency:**
- Operations that are causally related are seen by all nodes in the same order
- Concurrent operations may be seen in different orders
- **Use case:** Collaborative editing, messaging systems
- **Trade-off:** More complex to implement than eventual consistency

**Read-Your-Writes Consistency:**
- A user always sees their own updates immediately
- Other users may see stale data
- **Use case:** User profile updates, comment systems
- **Trade-off:** May require session affinity

**Monotonic Read Consistency:**
- Once a user reads a value, subsequent reads will never return older values
- Prevents "going back in time"
- **Use case:** News feeds, timelines
- **Trade-off:** May require tracking read versions

**Linearizability:**
- Strongest consistency model
- Operations appear instantaneous and in real-time order
- **Use case:** Distributed locks, leader election
- **Trade-off:** Highest latency cost

### What Interviewers Are Looking For
- Understanding of the consistency spectrum
- Ability to match consistency models to use cases
- Recognition of performance implications
- Knowledge of implementation techniques

### Key Points to Mention
- Consistency is a spectrum, not binary
- Stronger consistency = higher latency and lower availability
- Different data in the same system can have different consistency requirements
- Conflict resolution strategies (last-write-wins, vector clocks, CRDTs)

### Common Follow-Ups
- "How would you implement read-your-writes consistency?"
- "What are vector clocks and when would you use them?"
- "How does Cassandra achieve tunable consistency?"

---

## 3. ACID vs BASE

### Question
**"Compare ACID and BASE properties. When would you choose one over the other?"**

### Detailed Answer

**ACID Properties (Traditional RDBMS):**

- **Atomicity:** Transactions are all-or-nothing. If any part fails, the entire transaction is rolled back.
- **Consistency:** Database moves from one valid state to another. Constraints are always maintained.
- **Isolation:** Concurrent transactions don't interfere with each other. Multiple isolation levels exist (Read Uncommitted, Read Committed, Repeatable Read, Serializable).
- **Durability:** Once committed, data survives system failures.

**BASE Properties (NoSQL/Distributed):**

- **Basically Available:** System guarantees availability per CAP theorem.
- **Soft State:** State may change over time even without input due to eventual consistency.
- **Eventually Consistent:** System will become consistent over time.

**When to Choose ACID:**
- Financial transactions requiring strict correctness
- Inventory management where overselling is unacceptable
- Systems with complex relationships between entities
- Regulatory requirements (banking, healthcare)
- Low to moderate scale with predictable growth

**When to Choose BASE:**
- High-volume, globally distributed systems
- Social media, content delivery, analytics
- When availability is more critical than immediate consistency
- Systems that can tolerate and handle temporary inconsistencies
- Massive scale with unpredictable traffic patterns

**Hybrid Approaches:**
Many modern systems use both:
- Core financial data: ACID
- User activity, analytics: BASE
- NewSQL databases (CockroachDB, Spanner) offer ACID at scale

### What Interviewers Are Looking For
- Clear understanding of both paradigms
- Ability to reason about business requirements
- Knowledge of hybrid approaches
- Practical experience with trade-offs

### Key Points to Mention
- ACID = correctness over availability; BASE = availability over correctness
- Modern systems often use both paradigms for different data types
- NewSQL bridges the gap between ACID and scalability
- Saga pattern for distributed transactions

### Common Follow-Ups
- "How would you implement transactions across microservices?"
- "What is the saga pattern?"
- "How does two-phase commit work and what are its limitations?"

---

## 4. Horizontal vs Vertical Scaling

### Question
**"Explain horizontal and vertical scaling. When would you use each approach?"**

### Detailed Answer

**Vertical Scaling (Scale Up):**
- Adding more resources (CPU, RAM, storage) to existing machines
- **Advantages:**
  - Simple to implement - no code changes needed
  - No distributed system complexity
  - Data consistency is straightforward
  - Lower operational overhead
- **Disadvantages:**
  - Hardware limits (can't scale indefinitely)
  - Single point of failure
  - Expensive at high end
  - Downtime during upgrades
- **Best for:** Databases with complex joins, legacy applications, small-medium workloads

**Horizontal Scaling (Scale Out):**
- Adding more machines to the system
- **Advantages:**
  - Theoretically unlimited scaling
  - Better fault tolerance
  - Cost-effective with commodity hardware
  - No single point of failure
  - Can scale incrementally
- **Disadvantages:**
  - Distributed system complexity
  - Data consistency challenges
  - Network overhead
  - More complex operations
- **Best for:** Stateless services, high-availability requirements, web tiers, big data

**Scaling Strategies by Component:**

| Component | Primary Strategy | Notes |
|-----------|-----------------|-------|
| Web Servers | Horizontal | Stateless, easy to scale |
| Application Servers | Horizontal | Keep stateless where possible |
| Cache | Horizontal | Use consistent hashing |
| Database (reads) | Horizontal | Read replicas |
| Database (writes) | Vertical first, then shard | Most challenging |
| Message Queues | Horizontal | Partitioned topics |

### What Interviewers Are Looking For
- Understanding of when each approach is appropriate
- Recognition that databases are the hardest to scale horizontally
- Knowledge of stateless design principles
- Practical scaling experience

### Key Points to Mention
- Start with vertical scaling for simplicity, plan for horizontal
- Stateless services are easier to scale horizontally
- Databases often require vertical scaling before sharding
- Modern cloud makes horizontal scaling easier

### Common Follow-Ups
- "How do you make a service stateless?"
- "At what point would you move from vertical to horizontal scaling?"
- "How do you handle session state in a horizontally scaled system?"

---

## 5. Database Sharding

### Question
**"Explain database sharding. What are the different sharding strategies and their trade-offs?"**

### Detailed Answer

**What is Sharding?**
Sharding is horizontal partitioning of data across multiple database instances. Each shard contains a subset of the total data and operates independently.

**Sharding Strategies:**

**1. Range-Based Sharding:**
- Data distributed based on value ranges (e.g., A-M on shard 1, N-Z on shard 2)
- **Pros:** Simple to implement, efficient range queries
- **Cons:** Uneven distribution (hotspots), rebalancing is difficult
- **Use case:** Time-series data, alphabetical lookups

**2. Hash-Based Sharding:**
- Hash function determines shard (e.g., hash(user_id) % num_shards)
- **Pros:** Even distribution, predictable routing
- **Cons:** Range queries require scatter-gather, resharding is painful
- **Use case:** User data, session storage

**3. Consistent Hashing:**
- Nodes arranged in a ring, data assigned to nearest node
- **Pros:** Minimal reshuffling when adding/removing nodes
- **Cons:** More complex, potential for slight imbalance
- **Use case:** Distributed caches, DynamoDB-style systems

**4. Directory-Based Sharding:**
- Lookup service maintains mapping of keys to shards
- **Pros:** Flexible, supports complex sharding logic
- **Cons:** Lookup service becomes bottleneck/SPOF
- **Use case:** Multi-tenant systems, complex partitioning requirements

**5. Geographic Sharding:**
- Data partitioned by geographic region
- **Pros:** Lower latency for users, data locality compliance
- **Cons:** Cross-region queries are expensive
- **Use case:** Global applications, GDPR compliance

**Challenges:**
- **Cross-shard queries:** Expensive scatter-gather operations
- **Cross-shard transactions:** Require 2PC or saga pattern
- **Resharding:** Data migration is complex and risky
- **Referential integrity:** Foreign keys across shards don't work
- **Hotspots:** Uneven distribution leads to bottlenecks

### What Interviewers Are Looking For
- Understanding of different strategies and trade-offs
- Recognition of challenges and how to mitigate them
- Knowledge of shard key selection importance
- Practical experience with sharded systems

### Key Points to Mention
- Shard key selection is critical and hard to change
- Avoid cross-shard operations where possible
- Consider application-level sharding vs database-level
- Consistent hashing minimizes data movement

### Common Follow-Ups
- "How would you choose a shard key for a social media application?"
- "How do you handle resharding without downtime?"
- "What happens when a shard goes down?"

---

## 6. Caching Strategies

### Question
**"Explain different caching strategies. How do you decide what caching pattern to use?"**

### Detailed Answer

**Cache-Aside (Lazy Loading):**
```
Read: App checks cache -> if miss, read from DB -> write to cache -> return
Write: App writes to DB -> invalidate/update cache
```
- **Pros:** Only requested data is cached, cache failure doesn't break the system
- **Cons:** Cache miss penalty, potential for stale data
- **Use case:** General-purpose caching, read-heavy workloads

**Write-Through:**
```
Write: App writes to cache -> cache synchronously writes to DB
Read: App reads from cache (always a hit if data exists)
```
- **Pros:** Cache always consistent with DB, no stale reads
- **Cons:** Write latency (two writes), cache may contain unread data
- **Use case:** Read-heavy with consistency requirements

**Write-Behind (Write-Back):**
```
Write: App writes to cache -> cache asynchronously writes to DB
Read: App reads from cache
```
- **Pros:** Low write latency, batching possible
- **Cons:** Risk of data loss, complex consistency
- **Use case:** Write-heavy workloads that can tolerate some data loss

**Read-Through:**
```
Read: App reads from cache -> cache loads from DB on miss
Write: App writes to DB directly
```
- **Pros:** Simplified application code, cache manages loading
- **Cons:** First request always misses, cold start issues
- **Use case:** When cache can be tightly integrated with data source

**Refresh-Ahead:**
```
Cache proactively refreshes frequently accessed data before expiration
```
- **Pros:** Reduced latency, no cache miss penalty for hot data
- **Cons:** Complexity, may refresh data that's not needed
- **Use case:** Predictable access patterns, hot data

**Cache Invalidation Strategies:**
- **TTL (Time-To-Live):** Simple but may serve stale data
- **Event-Based:** Invalidate on updates, more complex but accurate
- **Version-Based:** Include version in cache key

### What Interviewers Are Looking For
- Understanding of different patterns and trade-offs
- Ability to match patterns to use cases
- Knowledge of cache invalidation challenges
- Awareness of cache stampede and thundering herd problems

### Key Points to Mention
- "There are only two hard things in CS: cache invalidation and naming things"
- Consider TTL based on data volatility
- Cache stampede prevention (locking, probabilistic expiration)
- Distributed caching challenges (consistency, network overhead)

### Common Follow-Ups
- "How do you prevent cache stampede?"
- "How do you handle cache warming?"
- "What's the difference between Redis and Memcached?"

---

## 7. Load Balancing Algorithms

### Question
**"Explain different load balancing algorithms and when you would use each."**

### Detailed Answer

**Round Robin:**
- Requests distributed sequentially to each server
- **Pros:** Simple, fair distribution for homogeneous servers
- **Cons:** Doesn't account for server capacity or current load
- **Use case:** Identical servers with similar request processing times

**Weighted Round Robin:**
- Servers assigned weights based on capacity
- Higher weight = more requests
- **Pros:** Accounts for different server capacities
- **Cons:** Doesn't consider current load
- **Use case:** Heterogeneous server fleet

**Least Connections:**
- Directs traffic to server with fewest active connections
- **Pros:** Adapts to varying request processing times
- **Cons:** Doesn't account for server capacity
- **Use case:** Long-lived connections, varying request complexity

**Weighted Least Connections:**
- Combines weights with connection count
- **Pros:** Accounts for both capacity and current load
- **Cons:** More complex to implement
- **Use case:** Heterogeneous servers with varying request complexity

**IP Hash:**
- Client IP determines server (hash(IP) % servers)
- **Pros:** Session persistence without cookies
- **Cons:** Uneven distribution if IP distribution is skewed
- **Use case:** Stateful applications, simple session affinity

**Consistent Hashing:**
- Minimizes redistribution when servers added/removed
- **Pros:** Smooth scaling, good for caching
- **Cons:** More complex, virtual nodes needed for balance
- **Use case:** Distributed caches, session stores

**Least Response Time:**
- Routes to server with lowest latency
- **Pros:** Optimizes user experience
- **Cons:** Requires latency monitoring
- **Use case:** Geographically distributed servers

**Random:**
- Randomly selects a server
- **Pros:** Simple, stateless
- **Cons:** Not optimal distribution
- **Use case:** When simplicity trumps optimization

### What Interviewers Are Looking For
- Understanding of algorithm characteristics
- Ability to match algorithms to scenarios
- Knowledge of session affinity implications
- Awareness of health checking importance

### Key Points to Mention
- Health checks are essential regardless of algorithm
- Layer 4 vs Layer 7 load balancing
- Global vs local load balancing
- Connection draining for graceful server removal

### Common Follow-Ups
- "How do you handle sticky sessions?"
- "What's the difference between L4 and L7 load balancing?"
- "How do you perform health checks?"

---

## 8. Microservices vs Monolith

### Question
**"Compare microservices and monolithic architectures. When would you choose each?"**

### Detailed Answer

**Monolithic Architecture:**
- Single deployable unit containing all functionality
- Shared database, in-process communication

**Advantages:**
- Simple development and deployment
- Easy debugging and testing
- No network latency between components
- Strong consistency with shared database
- Lower operational complexity

**Disadvantages:**
- Scaling requires scaling entire application
- Technology lock-in
- Long deployment cycles
- Single point of failure
- Team coordination challenges at scale

**Microservices Architecture:**
- Application as collection of loosely coupled services
- Each service has its own database and deployment

**Advantages:**
- Independent scaling of services
- Technology diversity (right tool for the job)
- Faster deployment cycles
- Fault isolation
- Team autonomy
- Easier to understand individual services

**Disadvantages:**
- Distributed system complexity
- Network latency and failure handling
- Data consistency challenges
- Operational overhead (monitoring, logging, tracing)
- Testing complexity
- Service discovery and orchestration

**When to Choose Monolith:**
- Early-stage startups (speed to market)
- Small teams (< 10 engineers)
- Simple domain
- Uncertain requirements
- Limited DevOps expertise

**When to Choose Microservices:**
- Large, complex domains
- Multiple teams working independently
- Different scaling requirements per component
- Need for technology diversity
- High availability requirements
- Clear domain boundaries

**Migration Path:**
1. Start with modular monolith
2. Identify bounded contexts
3. Extract services at natural boundaries
4. Strangler fig pattern for gradual migration

### What Interviewers Are Looking For
- Balanced view of both architectures
- Understanding of organizational implications (Conway's Law)
- Recognition that microservices aren't always better
- Knowledge of migration strategies

### Key Points to Mention
- "Microservices are a solution to organizational scaling, not technical scaling"
- Conway's Law: system design mirrors organization structure
- Start monolithic, evolve to microservices
- Microservices require mature DevOps practices

### Common Follow-Ups
- "How do you handle transactions across microservices?"
- "How do you debug a request spanning multiple services?"
- "What is the strangler fig pattern?"

---

## 9. Message Queues

### Question
**"Explain message queues and their role in distributed systems. What are the different messaging patterns?"**

### Detailed Answer

**What is a Message Queue?**
An asynchronous communication mechanism where producers send messages to a queue, and consumers process them independently.

**Benefits:**
- **Decoupling:** Producers and consumers are independent
- **Load Leveling:** Absorbs traffic spikes
- **Fault Tolerance:** Messages persist if consumer fails
- **Scalability:** Add consumers to increase throughput
- **Async Processing:** Improves response times

**Messaging Patterns:**

**1. Point-to-Point (Queue):**
- One message, one consumer
- Load balancing across consumers
- **Use case:** Task queues, work distribution

**2. Publish-Subscribe (Topic):**
- One message, multiple subscribers
- Each subscriber gets a copy
- **Use case:** Event notification, fan-out

**3. Request-Reply:**
- Synchronous-like pattern over async infrastructure
- Correlation IDs match responses to requests
- **Use case:** RPC over message queue

**Delivery Guarantees:**

| Guarantee | Description | Trade-off |
|-----------|-------------|-----------|
| At-most-once | Message may be lost, never duplicated | Highest performance |
| At-least-once | Message never lost, may be duplicated | Requires idempotent consumers |
| Exactly-once | Message delivered exactly once | Highest complexity/latency |

**Message Queue Technologies:**
- **RabbitMQ:** Traditional message broker, rich routing
- **Apache Kafka:** Distributed log, high throughput, replay capability
- **Amazon SQS:** Managed queue service, simple
- **Apache Pulsar:** Multi-tenancy, tiered storage
- **Redis Streams:** Lightweight, in-memory

**Key Considerations:**
- Message ordering (per-partition in Kafka, not guaranteed in SQS)
- Dead letter queues for failed messages
- Message retention and replay
- Consumer group management
- Backpressure handling

### What Interviewers Are Looking For
- Understanding of async communication benefits
- Knowledge of different patterns and guarantees
- Ability to choose appropriate technology
- Awareness of failure handling

### Key Points to Mention
- Prefer at-least-once with idempotent consumers
- Dead letter queues are essential
- Consider message ordering requirements
- Kafka for event sourcing and replay; RabbitMQ for routing

### Common Follow-Ups
- "How do you handle poison messages?"
- "What's the difference between Kafka and RabbitMQ?"
- "How do you implement exactly-once semantics?"

---

## 10. SQL vs NoSQL

### Question
**"Compare SQL and NoSQL databases. How do you decide which to use?"**

### Detailed Answer

**SQL (Relational) Databases:**
- Structured data with predefined schema
- ACID transactions
- Complex queries with JOINs
- Examples: PostgreSQL, MySQL, Oracle

**Advantages:**
- Strong consistency
- Powerful query language
- Mature tooling and expertise
- Referential integrity
- Complex aggregations

**Disadvantages:**
- Schema changes can be painful
- Horizontal scaling is challenging
- May be overkill for simple access patterns

**NoSQL Databases:**

**1. Document Stores (MongoDB, Couchbase):**
- Flexible schema, JSON-like documents
- Good for: Content management, catalogs, user profiles
- Trade-off: Limited joins, eventual consistency

**2. Key-Value Stores (Redis, DynamoDB):**
- Simple get/put operations
- Good for: Caching, session storage, leaderboards
- Trade-off: No complex queries

**3. Wide-Column Stores (Cassandra, HBase):**
- Column families, good for sparse data
- Good for: Time-series, IoT, analytics
- Trade-off: Limited query patterns

**4. Graph Databases (Neo4j, Amazon Neptune):**
- Optimized for relationships
- Good for: Social networks, fraud detection, recommendations
- Trade-off: Not for non-graph data

**Decision Framework:**

| Factor | Choose SQL | Choose NoSQL |
|--------|-----------|--------------|
| Data structure | Well-defined, relational | Flexible, hierarchical |
| Consistency needs | Strong consistency | Eventual acceptable |
| Query patterns | Complex, ad-hoc | Simple, known patterns |
| Scale | Moderate, vertical | Massive, horizontal |
| Development speed | Schema design upfront | Iterate quickly |

**Polyglot Persistence:**
Modern systems often use multiple databases:
- User accounts: PostgreSQL (ACID)
- Product catalog: MongoDB (flexible schema)
- Session cache: Redis (speed)
- Analytics: Cassandra (write throughput)
- Recommendations: Neo4j (relationships)

### What Interviewers Are Looking For
- Understanding of different NoSQL categories
- Ability to match database to use case
- Knowledge of polyglot persistence
- Practical experience with trade-offs

### Key Points to Mention
- NoSQL isn't "No SQL" but "Not Only SQL"
- Access patterns should drive database choice
- SQL is often underestimated for scaling
- NewSQL bridges the gap (CockroachDB, TiDB)

### Common Follow-Ups
- "How would you migrate from SQL to NoSQL?"
- "When would you use a graph database?"
- "What is polyglot persistence?"

---

## 11. Replication Strategies

### Question
**"Explain database replication strategies and their trade-offs."**

### Detailed Answer

**Single-Leader (Master-Slave) Replication:**
- One leader handles all writes
- Followers replicate from leader, serve reads

**Advantages:**
- Simple to implement and understand
- Strong consistency for writes
- Read scaling via replicas

**Disadvantages:**
- Leader is a single point of failure
- Write bottleneck
- Failover complexity

**Synchronous vs Asynchronous:**
- **Synchronous:** Write confirmed after all replicas acknowledge. Durable but slow.
- **Asynchronous:** Write confirmed after leader persists. Fast but risk of data loss.
- **Semi-synchronous:** At least one replica must acknowledge.

**Multi-Leader (Master-Master) Replication:**
- Multiple nodes accept writes
- Changes replicated to all leaders

**Advantages:**
- Better write availability
- Geographic distribution
- No single point of failure for writes

**Disadvantages:**
- Conflict resolution required
- More complex consistency model
- Write conflicts possible

**Conflict Resolution:**
- Last-write-wins (LWW)
- Merge/CRDT-based
- Application-level resolution
- Vector clocks for ordering

**Leaderless Replication:**
- All nodes are equal, any can accept writes
- Quorum-based consistency (R + W > N)

**Advantages:**
- High availability
- No leader election needed
- Tolerates node failures

**Disadvantages:**
- Eventual consistency
- Read repair and anti-entropy needed
- More complex operations

**Quorum Formulas:**
- N = total replicas
- W = write quorum
- R = read quorum
- W + R > N for strong consistency
- W > N/2 for write consistency

### What Interviewers Are Looking For
- Understanding of trade-offs between strategies
- Knowledge of consistency implications
- Ability to configure quorums appropriately
- Awareness of conflict resolution approaches

### Key Points to Mention
- Replication for availability and read scaling, not write scaling
- Asynchronous replication can lose data on leader failure
- Quorum selection affects consistency and availability
- Geographic replication has latency implications

### Common Follow-Ups
- "How do you handle leader failover?"
- "What is split-brain and how do you prevent it?"
- "How does Cassandra handle write conflicts?"

---

## 12. Consensus Algorithms

### Question
**"Explain consensus algorithms and why they're important in distributed systems."**

### Detailed Answer

**What is Consensus?**
Consensus algorithms allow distributed systems to agree on a single value or sequence of values despite failures. Essential for leader election, distributed locks, and replicated state machines.

**Properties Required:**
- **Agreement:** All correct nodes agree on the same value
- **Validity:** The agreed value was proposed by some node
- **Termination:** All correct nodes eventually decide
- **Integrity:** Each node decides at most once

**Paxos:**
- Classic consensus algorithm (Lamport)
- Three roles: Proposers, Acceptors, Learners
- Two-phase protocol: Prepare and Accept

**Phases:**
1. Prepare: Proposer sends prepare request with proposal number
2. Promise: Acceptors promise not to accept lower-numbered proposals
3. Accept: Proposer sends value to acceptors
4. Accepted: Acceptors accept and inform learners

**Challenges:** Complex to understand and implement correctly

**Raft:**
- Designed for understandability
- Leader-based consensus
- Widely implemented (etcd, Consul)

**Key Components:**
- Leader election (randomized timeouts)
- Log replication (leader appends, followers replicate)
- Safety (committed entries never lost)

**Why Raft over Paxos:**
- Clearer separation of concerns
- Easier to implement correctly
- Better documentation

**Zab (Zookeeper Atomic Broadcast):**
- Used by Apache Zookeeper
- Optimized for primary-backup systems
- Ordering guarantees for state machine replication

**Byzantine Fault Tolerance (BFT):**
- Handles malicious nodes
- Requires 3f+1 nodes to tolerate f failures
- PBFT, Tendermint
- **Use case:** Blockchain, high-security systems

### What Interviewers Are Looking For
- Understanding of why consensus is hard
- Knowledge of common algorithms
- Ability to explain leader election
- Awareness of FLP impossibility result

### Key Points to Mention
- Consensus is solved problem but implementations are tricky
- Raft is go-to for most applications
- FLP: consensus impossible with even one faulty node in async system
- Timeouts and randomization work around FLP practically

### Common Follow-Ups
- "What is the FLP impossibility theorem?"
- "How does Raft handle leader failure?"
- "What's the difference between Paxos and Raft?"

---

## 13. CDN Architecture

### Question
**"Explain how CDNs work and their role in system design."**

### Detailed Answer

**What is a CDN?**
A Content Delivery Network is a geographically distributed network of proxy servers that cache content closer to users, reducing latency and origin server load.

**How CDNs Work:**

1. **Request Routing:**
   - DNS-based: CDN's DNS returns nearest edge server IP
   - Anycast: Same IP, routed to nearest location
   - HTTP redirect: Origin redirects to edge

2. **Caching:**
   - Edge servers cache content from origin
   - Cache headers control behavior (Cache-Control, ETag, Expires)
   - Hit/miss determines if origin is contacted

3. **Content Delivery:**
   - User connects to nearest edge
   - If cached, content served directly
   - If not, edge fetches from origin, caches, serves

**CDN Benefits:**
- **Reduced latency:** Content closer to users
- **Origin offload:** Fewer requests to origin
- **DDoS protection:** Distributed absorption
- **High availability:** Redundant edge locations
- **Bandwidth savings:** Less origin egress

**Types of Content:**

| Type | Cacheability | TTL |
|------|-------------|-----|
| Static (images, CSS, JS) | High | Long (days/weeks) |
| Dynamic (HTML) | Varies | Short (seconds/minutes) |
| Streaming | Chunked | Session-based |
| API responses | Carefully | Very short or none |

**Cache Invalidation:**
- **TTL expiration:** Content expires after time
- **Purge:** Explicit invalidation via API
- **Versioned URLs:** New version = new URL
- **Soft purge:** Serve stale while revalidating

**CDN Architectures:**

**Pull CDN:**
- Edge fetches from origin on cache miss
- Simpler to set up
- First request always goes to origin

**Push CDN:**
- Content pushed to edges proactively
- Better for predictable content
- More control, more complexity

### What Interviewers Are Looking For
- Understanding of how CDNs reduce latency
- Knowledge of caching strategies
- Awareness of cache invalidation challenges
- Ability to design CDN integration

### Key Points to Mention
- CDNs are essential for global applications
- Cache invalidation is the hard part
- Consider cache-busting for deployments
- Edge compute is evolving CDNs (Cloudflare Workers)

### Common Follow-Ups
- "How do you handle cache invalidation on deployment?"
- "What content would you not cache on a CDN?"
- "How does DNS-based routing work?"

---

## 14. API Design

### Question
**"Compare REST, GraphQL, and gRPC. When would you use each?"**

### Detailed Answer

**REST (Representational State Transfer):**
- Resource-based URLs, HTTP methods
- Stateless, cacheable
- JSON/XML payloads

**Advantages:**
- Simple, well-understood
- Excellent tooling and caching
- Works everywhere (browsers, mobile)

**Disadvantages:**
- Over-fetching/under-fetching
- Multiple round trips for related data
- Versioning challenges

**Use cases:** Public APIs, web applications, mobile backends

**GraphQL:**
- Query language for APIs
- Client specifies exact data needs
- Single endpoint

**Advantages:**
- No over-fetching/under-fetching
- Strong typing with schema
- Introspection and documentation
- Single round trip for complex data

**Disadvantages:**
- Caching is complex
- N+1 query problems
- Learning curve
- Potential for expensive queries

**Use cases:** Complex UIs, mobile apps (bandwidth sensitive), BFF pattern

**gRPC:**
- Protocol Buffers for serialization
- HTTP/2 with streaming
- Strongly typed contracts

**Advantages:**
- High performance (binary, HTTP/2)
- Bi-directional streaming
- Strong contracts
- Code generation

**Disadvantages:**
- Not browser-friendly (needs proxy)
- Less human-readable
- Smaller ecosystem

**Use cases:** Microservice communication, real-time systems, polyglot environments

**Comparison Table:**

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Data format | JSON/XML | JSON | Protocol Buffers |
| Typing | Optional (OpenAPI) | Strong | Strong |
| Performance | Good | Good | Excellent |
| Browser support | Excellent | Excellent | Limited |
| Caching | Excellent | Complex | Limited |
| Streaming | Polling/WebSocket | Subscriptions | Native |

### What Interviewers Are Looking For
- Understanding of trade-offs between approaches
- Ability to match API style to use case
- Knowledge of performance implications
- Practical experience with multiple styles

### Key Points to Mention
- REST for public APIs and browser compatibility
- GraphQL for complex client data needs
- gRPC for internal microservice communication
- Often use multiple styles in same system

### Common Follow-Ups
- "How do you handle versioning in REST?"
- "How do you prevent expensive GraphQL queries?"
- "How do you expose gRPC to browsers?"

---

## 15. Rate Limiting

### Question
**"Explain rate limiting algorithms and how you would implement rate limiting in a distributed system."**

### Detailed Answer

**Why Rate Limiting?**
- Prevent abuse and DDoS
- Fair resource allocation
- Cost control
- Maintain service quality

**Rate Limiting Algorithms:**

**1. Token Bucket:**
- Bucket holds tokens, refilled at constant rate
- Request consumes token; if empty, rejected
- Allows bursts up to bucket size

**Parameters:** Bucket size (burst), Refill rate (sustained)
**Use case:** Most common, good balance of burst and sustained

**2. Leaky Bucket:**
- Requests enter bucket, processed at constant rate
- Overflow is rejected
- Smooths traffic to constant rate

**Parameters:** Bucket size, Leak rate
**Use case:** Traffic shaping, steady output rate needed

**3. Fixed Window:**
- Count requests in fixed time windows
- Reset count at window boundary
- Simple but allows 2x burst at boundary

**Parameters:** Window size, Request limit
**Use case:** Simple implementation, approximate limiting

**4. Sliding Window Log:**
- Store timestamp of each request
- Count requests in sliding window
- Most accurate but memory-intensive

**Parameters:** Window size, Request limit
**Use case:** When accuracy is critical

**5. Sliding Window Counter:**
- Hybrid of fixed window and sliding log
- Weighted average of current and previous window
- Good balance of accuracy and efficiency

**Distributed Rate Limiting:**

**Centralized (Redis):**
```
INCR user:{id}:requests
EXPIRE user:{id}:requests 60
```
- Accurate but adds latency
- Redis is single point of failure

**Distributed (Local + Sync):**
- Local rate limiting with periodic sync
- Less accurate but more resilient
- Good for high-volume systems

**Sticky Sessions:**
- Route user to same server
- Local rate limiting
- Risk of imbalance

**Response Handling:**
- HTTP 429 Too Many Requests
- Retry-After header
- X-RateLimit-* headers for transparency

### What Interviewers Are Looking For
- Knowledge of different algorithms
- Understanding of distributed challenges
- Ability to choose appropriate algorithm
- Awareness of user experience implications

### Key Points to Mention
- Token bucket for most use cases
- Consider different limits for different operations
- Rate limit by user, IP, API key
- Graceful degradation over hard failures

### Common Follow-Ups
- "How do you handle rate limiting with multiple API keys per user?"
- "How do you prevent race conditions in distributed rate limiting?"
- "What headers would you return to clients?"

---

## 16. Idempotency

### Question
**"What is idempotency and why is it important in distributed systems?"**

### Detailed Answer

**Definition:**
An operation is idempotent if performing it multiple times has the same effect as performing it once. In distributed systems, this ensures safety when retrying failed requests.

**Why It Matters:**
- Network failures cause request retries
- Can't always distinguish timeout from failure
- "At-least-once" delivery requires idempotent handlers
- Prevents duplicate orders, payments, etc.

**Naturally Idempotent Operations:**
- GET, PUT, DELETE (in REST)
- Setting absolute values (x = 5)
- Upserts with unique keys

**Non-Idempotent Operations:**
- POST (creates new resource)
- Incrementing values (x += 1)
- Appending to lists
- Sending notifications

**Making Operations Idempotent:**

**1. Idempotency Keys:**
- Client generates unique key per operation
- Server checks if key was already processed
- Return same response for duplicate requests

```
POST /orders
Idempotency-Key: abc-123

Server: Check if abc-123 exists
  - If yes: return cached response
  - If no: process, store result with key, return
```

**2. Conditional Operations:**
- Include expected state in request
- Only proceed if state matches
- ETag, If-Match headers

**3. Request Deduplication:**
- Track processed requests (database, cache)
- Time-bounded (can't store forever)
- Consider distributed storage for multi-node

**4. Natural Keys:**
- Use deterministic IDs from request data
- hash(user_id + order_details + timestamp_bucket)

**Implementation Considerations:**
- Key expiration (how long to remember?)
- Storage (database vs cache)
- Consistency (what if check and process aren't atomic?)
- Response storage (return same response)

### What Interviewers Are Looking For
- Understanding of why idempotency matters
- Knowledge of implementation patterns
- Awareness of edge cases
- Practical experience with payment/order systems

### Key Points to Mention
- Essential for payment systems
- Client must generate idempotency key
- Consider key expiration policy
- Atomic check-and-process

### Common Follow-Ups
- "How long would you store idempotency keys?"
- "How do you handle idempotency in a distributed system?"
- "What if the idempotency check and operation aren't atomic?"

---

## 17. Event-Driven Architecture

### Question
**"Explain event-driven architecture and its patterns."**

### Detailed Answer

**What is Event-Driven Architecture?**
A software architecture pattern where the flow of the program is determined by events - significant changes in state that are published and consumed asynchronously.

**Core Concepts:**
- **Event:** Immutable fact that something happened
- **Producer:** Creates and publishes events
- **Consumer:** Subscribes to and processes events
- **Event Bus/Broker:** Routes events from producers to consumers

**Event Types:**

**Domain Events:**
- Significant business occurrences
- Example: OrderPlaced, PaymentReceived, UserRegistered

**Integration Events:**
- Cross-service communication
- Example: Inventory updated after order

**Event Sourcing:**
- Store events, not current state
- Rebuild state by replaying events
- Complete audit trail

**Patterns:**

**1. Event Notification:**
- Simple notification of change
- Consumers may fetch details separately
- Loose coupling

**2. Event-Carried State Transfer:**
- Event contains all needed data
- No callback to source required
- Higher bandwidth but more autonomous

**3. Event Sourcing:**
- Events are source of truth
- State derived from event stream
- Time travel and replay possible

**4. CQRS (Command Query Responsibility Segregation):**
- Separate models for read and write
- Often paired with event sourcing
- Optimized read models

**Benefits:**
- Loose coupling between services
- Scalability and resilience
- Temporal decoupling
- Audit trail and replay
- Easier to add new consumers

**Challenges:**
- Eventual consistency
- Event ordering
- Debugging complexity
- Event schema evolution
- Duplicate event handling

### What Interviewers Are Looking For
- Understanding of event-driven benefits
- Knowledge of different patterns
- Awareness of challenges
- Practical experience with event systems

### Key Points to Mention
- Events are immutable facts
- Eventually consistent by nature
- Schema versioning is important
- Consider compensating transactions for failures

### Common Follow-Ups
- "How do you handle event ordering?"
- "What is event sourcing and when would you use it?"
- "How do you version events as schema changes?"

---

## 18. Data Partitioning

### Question
**"Explain data partitioning strategies beyond sharding."**

### Detailed Answer

**Types of Partitioning:**

**1. Horizontal Partitioning (Sharding):**
- Rows distributed across partitions
- Same schema in each partition
- Covered in sharding section

**2. Vertical Partitioning:**
- Columns distributed across partitions
- Split table by column groups
- Example: User profile vs user activity

**Benefits:**
- Frequently accessed columns together
- Different storage for different access patterns
- Can use different technologies

**Use case:** Separating BLOB columns, hot vs cold data

**3. Functional Partitioning:**
- Data partitioned by business function
- Different databases for different domains
- Aligns with microservices

**Example:**
- Users service: User database
- Orders service: Orders database
- Inventory service: Inventory database

**4. Time-Based Partitioning:**
- Data partitioned by time periods
- Common for logs, events, metrics
- Easy to archive/delete old data

**Example:**
- logs_2024_01, logs_2024_02, etc.
- Hot data in SSD, cold in HDD

**5. Geographic Partitioning:**
- Data partitioned by region
- Supports data residency requirements
- Reduces latency for local users

**Example:**
- EU user data in EU region
- US user data in US region
- Satisfies GDPR requirements

**Partition Strategies Comparison:**

| Strategy | Key Benefit | Challenge |
|----------|-------------|-----------|
| Horizontal | Scale writes | Cross-partition queries |
| Vertical | Optimize access patterns | Joins across partitions |
| Functional | Service isolation | Cross-service transactions |
| Time-based | Easy archival | Historical queries |
| Geographic | Compliance/latency | Cross-region operations |

**Partition Management:**
- Automatic vs manual partitioning
- Partition pruning for query optimization
- Rebalancing strategies
- Monitoring partition skew

### What Interviewers Are Looking For
- Understanding of different partitioning types
- Ability to match strategy to requirements
- Knowledge of trade-offs
- Practical partition management experience

### Key Points to Mention
- Multiple partitioning strategies can be combined
- Consider query patterns when designing partitions
- Time-based is excellent for logs/analytics
- Geographic partitioning for compliance

### Common Follow-Ups
- "How would you partition a multi-tenant system?"
- "How do you handle queries spanning partitions?"
- "What is partition pruning?"

---

## Summary

These conceptual questions test your understanding of fundamental distributed systems principles. Key themes across all topics:

1. **Trade-offs are everywhere** - There's no perfect solution, only appropriate trade-offs for your context
2. **Consistency vs availability** - Understand when to prioritize each
3. **Scale considerations** - What works at small scale may not work at large scale
4. **Failure handling** - Assume everything will fail
5. **Practical experience** - Interviewers value real-world application of concepts

When answering these questions:
- Start with the definition/explanation
- Discuss trade-offs
- Give concrete examples
- Relate to your experience when possible
- Acknowledge what you don't know
