# Mock System Design Problems

A curated collection of practice problems organized by difficulty level. Use these for self-study or mock interviews.

---

## How to Use This Guide

1. **Start with your current level** - Be honest about where you are
2. **Time yourself** - 45 minutes max per problem
3. **Write it out** - Sketch diagrams, don't just think through it
4. **Review the hints only if stuck** - Try for 10-15 minutes first
5. **Compare against real solutions** - Search for blog posts and tech talks

---

## Beginner Level (8 Problems)

These problems focus on fundamental concepts and have well-documented solutions.

### 1. URL Shortener (TinyURL)

**Core Challenge:** Generate unique short codes, handle high read traffic

**Requirements to Consider:**
- Shorten long URLs to short aliases
- Redirect short URLs to original
- Optional: Custom aliases, expiration, analytics

**Hints:**
<details>
<summary>Hint 1: ID Generation</summary>
Consider base62 encoding of auto-incrementing IDs or pre-generated random strings
</details>

<details>
<summary>Hint 2: Storage</summary>
Key-value store is ideal. Think about read:write ratio (100:1 typical)
</details>

<details>
<summary>Hint 3: Scale</summary>
Cache aggressively - shortened URLs rarely change
</details>

---

### 2. Pastebin / Text Storage Service

**Core Challenge:** Store and retrieve text snippets efficiently

**Requirements to Consider:**
- Store text/code snippets with unique URLs
- Syntax highlighting support
- Expiration policies
- Rate limiting

**Hints:**
<details>
<summary>Hint 1: Storage Strategy</summary>
Separate metadata (SQL) from content (object storage like S3)
</details>

<details>
<summary>Hint 2: Size Limits</summary>
Define max paste size. Consider compression for large pastes
</details>

---

### 3. Rate Limiter

**Core Challenge:** Distributed counting with consistency

**Requirements to Consider:**
- Limit requests per user/IP/API key
- Multiple time windows (per second, minute, hour)
- Low latency overhead
- Handle distributed servers

**Hints:**
<details>
<summary>Hint 1: Algorithms</summary>
Token bucket, leaky bucket, fixed window, sliding window log, sliding window counter
</details>

<details>
<summary>Hint 2: Storage</summary>
Redis is common - atomic operations, TTL support, fast
</details>

<details>
<summary>Hint 3: Distributed Challenge</summary>
Race conditions across servers. Consider eventual consistency tradeoffs
</details>

---

### 4. Key-Value Store

**Core Challenge:** Distributed storage with replication

**Requirements to Consider:**
- Put(key, value) and Get(key) operations
- High availability
- Tunable consistency
- Automatic scaling

**Hints:**
<details>
<summary>Hint 1: Partitioning</summary>
Consistent hashing to distribute keys across nodes
</details>

<details>
<summary>Hint 2: Replication</summary>
Quorum-based reads/writes (W + R > N for strong consistency)
</details>

<details>
<summary>Hint 3: Conflict Resolution</summary>
Vector clocks, last-write-wins, or application-level resolution
</details>

---

### 5. Unique ID Generator

**Core Challenge:** Distributed ID generation without coordination

**Requirements to Consider:**
- Globally unique IDs
- Sortable by time (roughly)
- High throughput (10K+ IDs/second)
- 64-bit numeric IDs

**Hints:**
<details>
<summary>Hint 1: Approaches</summary>
UUID, database auto-increment, Twitter Snowflake, ticket servers
</details>

<details>
<summary>Hint 2: Snowflake Structure</summary>
Timestamp (41 bits) + Machine ID (10 bits) + Sequence (12 bits)
</details>

---

### 6. Task Queue / Job Scheduler

**Core Challenge:** Reliable task execution with retries

**Requirements to Consider:**
- Submit tasks for async processing
- Retry failed tasks
- Priority queues
- Scheduled/delayed tasks

**Hints:**
<details>
<summary>Hint 1: Message Brokers</summary>
Kafka, RabbitMQ, SQS - understand tradeoffs
</details>

<details>
<summary>Hint 2: At-least-once Delivery</summary>
Acknowledgments + retries, but workers must be idempotent
</details>

---

### 7. Leaderboard System

**Core Challenge:** Real-time ranking at scale

**Requirements to Consider:**
- Update user scores
- Get user rank
- Get top N users
- Time-bounded leaderboards (daily, weekly)

**Hints:**
<details>
<summary>Hint 1: Data Structure</summary>
Redis Sorted Sets - O(log N) updates and rank queries
</details>

<details>
<summary>Hint 2: Sharding Challenge</summary>
Global rankings are hard to shard. Consider approximations or fan-out
</details>

---

### 8. URL Bookmark Service

**Core Challenge:** Personal data management with sync

**Requirements to Consider:**
- Save bookmarks with tags
- Search by title, URL, tags
- Sync across devices
- Import/export

**Hints:**
<details>
<summary>Hint 1: Search</summary>
Elasticsearch for full-text search on titles and descriptions
</details>

<details>
<summary>Hint 2: Sync</summary>
Event sourcing or operational transforms for multi-device sync
</details>

---

## Intermediate Level (10 Problems)

These require balancing multiple concerns and handling higher scale.

### 9. Twitter / Social Feed

**Core Challenge:** Fan-out problem, real-time updates

**Requirements to Consider:**
- Post tweets
- Follow users
- Home timeline (posts from followed users)
- 100M+ DAU scale

**Hints:**
<details>
<summary>Hint 1: Fan-out Strategies</summary>
Fan-out on write (push to follower timelines) vs fan-out on read (pull at request time)
</details>

<details>
<summary>Hint 2: Hybrid Approach</summary>
Push for normal users, pull for celebrities (many followers)
</details>

<details>
<summary>Hint 3: Timeline Cache</summary>
Pre-computed timeline in Redis, limited to recent N posts
</details>

---

### 10. Instagram / Photo Sharing

**Core Challenge:** Media storage and delivery at scale

**Requirements to Consider:**
- Upload photos/videos
- News feed
- Follow system
- Stories (ephemeral content)

**Hints:**
<details>
<summary>Hint 1: Media Pipeline</summary>
Upload -> Object Storage -> Async processing (resize, thumbnails) -> CDN
</details>

<details>
<summary>Hint 2: Feed Generation</summary>
Similar to Twitter but media-heavy. Pre-generate feed, lazy-load media
</details>

---

### 11. Notification System

**Core Challenge:** Multi-channel delivery with preferences

**Requirements to Consider:**
- Push, email, SMS, in-app notifications
- User preferences and rate limiting
- Template management
- Delivery tracking

**Hints:**
<details>
<summary>Hint 1: Architecture</summary>
Event-driven: Event -> Notification Service -> Channel-specific workers
</details>

<details>
<summary>Hint 2: Reliability</summary>
Exactly-once is hard. Dedup on client side, at-least-once on server
</details>

---

### 12. Chat System (WhatsApp/Messenger)

**Core Challenge:** Real-time messaging, presence, offline delivery

**Requirements to Consider:**
- 1:1 and group messaging
- Online/offline status
- Message delivery receipts
- Media sharing

**Hints:**
<details>
<summary>Hint 1: Connection Management</summary>
WebSockets for real-time. Connection servers separate from business logic
</details>

<details>
<summary>Hint 2: Message Storage</summary>
Store-and-forward for offline users. Consider message retention policies
</details>

<details>
<summary>Hint 3: Group Chat</summary>
Fan-out to group members. Optimize for small groups (< 100 members)
</details>

---

### 13. Web Crawler

**Core Challenge:** Distributed crawling with politeness

**Requirements to Consider:**
- Crawl billions of pages
- Respect robots.txt
- Handle duplicates
- Prioritize important pages

**Hints:**
<details>
<summary>Hint 1: Frontier</summary>
URL frontier with priority queues per domain for politeness
</details>

<details>
<summary>Hint 2: Deduplication</summary>
URL normalization + content hashing (SimHash for near-duplicates)
</details>

---

### 14. Autocomplete / Typeahead

**Core Challenge:** Low latency prefix matching

**Requirements to Consider:**
- Suggest completions as user types
- Personalized suggestions
- Trending queries
- < 100ms latency

**Hints:**
<details>
<summary>Hint 1: Data Structure</summary>
Trie with frequency counts. Precompute top suggestions at each node
</details>

<details>
<summary>Hint 2: Updates</summary>
Batch updates to trie (hourly/daily). Real-time layer for trending
</details>

---

### 15. Dropbox / File Storage

**Core Challenge:** Efficient sync and deduplication

**Requirements to Consider:**
- Upload/download files
- Sync across devices
- File versioning
- Sharing and permissions

**Hints:**
<details>
<summary>Hint 1: Chunking</summary>
Split files into chunks (4-8MB). Only sync changed chunks
</details>

<details>
<summary>Hint 2: Deduplication</summary>
Content-addressable storage. Hash chunks, store once if duplicate
</details>

<details>
<summary>Hint 3: Sync Protocol</summary>
Track file metadata separately. Use checksums to detect changes
</details>

---

### 16. Uber / Ride Sharing

**Core Challenge:** Real-time location matching

**Requirements to Consider:**
- Rider requests ride
- Match with nearby drivers
- Real-time location updates
- ETA calculation

**Hints:**
<details>
<summary>Hint 1: Geospatial Index</summary>
Geohash or QuadTree for location-based queries
</details>

<details>
<summary>Hint 2: Location Updates</summary>
Drivers send updates every few seconds. Write-heavy workload
</details>

<details>
<summary>Hint 3: Matching</summary>
Find nearby available drivers, rank by ETA, driver rating
</details>

---

### 17. Yelp / Nearby Places

**Core Challenge:** Geospatial search and ranking

**Requirements to Consider:**
- Search businesses by location
- Filter by category, rating, price
- Business pages with reviews
- Photo galleries

**Hints:**
<details>
<summary>Hint 1: Geospatial Index</summary>
QuadTree, Geohash, or PostGIS for location queries
</details>

<details>
<summary>Hint 2: Search</summary>
Elasticsearch with geo_distance queries and faceted filters
</details>

---

### 18. News Feed Ranking

**Core Challenge:** Personalized content ranking

**Requirements to Consider:**
- Rank posts for each user
- Balance relevance, recency, engagement
- A/B testing framework
- Real-time updates

**Hints:**
<details>
<summary>Hint 1: Ranking Signals</summary>
Affinity (user-author relationship), edge weight (content type), time decay
</details>

<details>
<summary>Hint 2: Architecture</summary>
Candidate generation -> Ranking -> Final selection
</details>

---

## Advanced Level (9 Problems)

These involve complex distributed systems, strong consistency requirements, or unique technical challenges.

### 19. Google Search

**Core Challenge:** Index the web, rank results, sub-second latency

**Requirements to Consider:**
- Crawl and index billions of pages
- Inverted index for keyword search
- PageRank-style authority scoring
- Query understanding and spelling correction

**Hints:**
<details>
<summary>Hint 1: Index Structure</summary>
Inverted index: word -> list of (docID, positions, metadata)
</details>

<details>
<summary>Hint 2: Serving</summary>
Shard index by document, replicate for availability. Scatter-gather query pattern
</details>

<details>
<summary>Hint 3: Ranking</summary>
Hundreds of signals. Separate indexing (offline) from serving (online)
</details>

---

### 20. YouTube / Video Streaming

**Core Challenge:** Video processing pipeline, adaptive streaming

**Requirements to Consider:**
- Upload and process videos
- Adaptive bitrate streaming
- Recommendations
- Comments and engagement

**Hints:**
<details>
<summary>Hint 1: Processing Pipeline</summary>
Upload -> Transcoding (multiple resolutions) -> CDN distribution
</details>

<details>
<summary>Hint 2: Streaming</summary>
HLS/DASH adaptive streaming. Chunk videos into segments
</details>

<details>
<summary>Hint 3: Scale</summary>
Edge caching critical. Popular videos at edge, long-tail at origin
</details>

---

### 21. Google Docs / Collaborative Editing

**Core Challenge:** Real-time collaboration with conflict resolution

**Requirements to Consider:**
- Multiple users editing simultaneously
- Real-time cursor positions
- Version history
- Offline support

**Hints:**
<details>
<summary>Hint 1: Algorithms</summary>
Operational Transformation (OT) or CRDTs for conflict resolution
</details>

<details>
<summary>Hint 2: Architecture</summary>
WebSocket for real-time. Document server maintains authoritative state
</details>

---

### 22. Distributed Message Queue (Kafka)

**Core Challenge:** Durable, ordered, high-throughput messaging

**Requirements to Consider:**
- Publish/subscribe
- Message ordering guarantees
- Exactly-once semantics
- Consumer groups

**Hints:**
<details>
<summary>Hint 1: Ordering</summary>
Partition by key for ordering. Only guarantee order within partition
</details>

<details>
<summary>Hint 2: Durability</summary>
Replicated log. Leader handles writes, followers replicate
</details>

---

### 23. Payment System

**Core Challenge:** Strong consistency, exactly-once transactions

**Requirements to Consider:**
- Process payments
- Handle refunds
- Fraud detection
- Compliance and audit trails

**Hints:**
<details>
<summary>Hint 1: Idempotency</summary>
Idempotency keys prevent duplicate charges. Store transaction state
</details>

<details>
<summary>Hint 2: Architecture</summary>
Event sourcing for audit trail. Saga pattern for distributed transactions
</details>

---

### 24. Stock Exchange / Trading Platform

**Core Challenge:** Ultra-low latency, fairness, ordering

**Requirements to Consider:**
- Order matching engine
- Price-time priority
- Market data distribution
- Fault tolerance

**Hints:**
<details>
<summary>Hint 1: Matching Engine</summary>
In-memory order book. Sorted structures for bid/ask
</details>

<details>
<summary>Hint 2: Sequencing</summary>
Single sequencer for fairness. Replicate for fault tolerance
</details>

---

### 25. Distributed Lock Service (Chubby/ZooKeeper)

**Core Challenge:** Consensus and fault tolerance

**Requirements to Consider:**
- Acquire/release locks
- Leader election
- Configuration management
- Fault tolerance (survive node failures)

**Hints:**
<details>
<summary>Hint 1: Consensus</summary>
Paxos or Raft for agreement. Majority quorum for decisions
</details>

<details>
<summary>Hint 2: Sessions</summary>
Session-based locks with heartbeats. Lock released if session expires
</details>

---

### 26. Ad Click Aggregation

**Core Challenge:** Real-time aggregation at massive scale

**Requirements to Consider:**
- Count clicks per ad in real-time
- Handle late-arriving data
- Exactly-once counting
- Query aggregates by time window

**Hints:**
<details>
<summary>Hint 1: Stream Processing</summary>
Kafka + Flink/Spark Streaming for real-time aggregation
</details>

<details>
<summary>Hint 2: Lambda Architecture</summary>
Real-time (approximate) + batch (accurate) layers
</details>

---

### 27. Hotel Reservation System

**Core Challenge:** Inventory management with overbooking prevention

**Requirements to Consider:**
- Search available rooms
- Book rooms with dates
- Prevent double-booking
- Handle cancellations

**Hints:**
<details>
<summary>Hint 1: Inventory Model</summary>
Date-based inventory. Track available count per room type per date
</details>

<details>
<summary>Hint 2: Concurrency</summary>
Optimistic locking or database transactions for booking
</details>

---

## Problem Selection Guide

| Goal | Recommended Problems |
|------|---------------------|
| First system design interview | 1, 2, 5, 9 |
| Practice fundamentals | 3, 4, 6, 7 |
| FAANG preparation | 9, 12, 15, 20, 21 |
| Backend-focused roles | 4, 6, 22, 23, 25 |
| Data-intensive roles | 13, 19, 20, 26 |
| Real-time systems | 12, 16, 21, 24 |

---

## Practice Checklist

For each problem, make sure you can:

- [ ] Clarify requirements and scope (5 min)
- [ ] Make reasonable capacity estimates (5 min)
- [ ] Design high-level architecture (10 min)
- [ ] Deep dive into 2-3 components (15 min)
- [ ] Discuss tradeoffs you made (5 min)
- [ ] Address failure scenarios (5 min)

---

## Additional Resources

After attempting these problems:

1. Read engineering blogs from companies that built these systems
2. Watch system design videos that walk through solutions
3. Practice explaining your design out loud
4. Get feedback from peers or mock interview platforms
