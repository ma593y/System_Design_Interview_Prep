# Twitter System Design - Social Feed and Timeline

## 1. Problem Statement

Design a social media platform similar to Twitter that allows users to:
- Post short messages (tweets) up to 280 characters
- Follow other users
- View a personalized timeline (home feed) of tweets from followed users
- Like, retweet, and reply to tweets
- Search for tweets and users
- View trending topics

The system must handle millions of concurrent users with real-time feed updates and sub-second latency for timeline generation.

---

## 2. Requirements

### 2.1 Functional Requirements

| Requirement | Description |
|-------------|-------------|
| **Post Tweet** | Users can create tweets with text, images, videos, and links |
| **Follow/Unfollow** | Users can follow/unfollow other users |
| **Home Timeline** | Display tweets from followed users in reverse chronological order |
| **User Timeline** | Display all tweets from a specific user |
| **Like/Retweet** | Users can like and retweet posts |
| **Reply** | Users can reply to tweets (threaded conversations) |
| **Search** | Search tweets and users by keywords |
| **Notifications** | Notify users of likes, retweets, follows, mentions |
| **Trending** | Display trending topics and hashtags |

### 2.2 Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Availability** | 99.99% uptime |
| **Latency** | Timeline load < 200ms (p99) |
| **Consistency** | Eventual consistency acceptable for feeds |
| **Scalability** | Support 500M+ DAU |
| **Durability** | Zero data loss for tweets |

### 2.3 Out of Scope
- Direct messaging (separate system)
- Twitter Spaces (audio rooms)
- Ads and monetization
- Content moderation AI

---

## 3. Capacity Estimation

### 3.1 User and Traffic Estimates

```
Daily Active Users (DAU):     500 million
Monthly Active Users (MAU):   800 million

Tweets per day:               500 million
Average tweets per user:      1 tweet/day (for active tweeters)
Tweet readers vs writers:     1000:1 (read-heavy)

Timeline refreshes per user:  10/day
Total timeline requests:      5 billion/day
```

### 3.2 Queries Per Second (QPS)

```
Tweet writes:
  500M tweets/day
  = 500M / 86,400 seconds
  = ~5,800 tweets/second
  Peak (3x): ~17,400 tweets/second

Timeline reads:
  5B timeline requests/day
  = 5B / 86,400 seconds
  = ~58,000 requests/second
  Peak (3x): ~174,000 requests/second

Search queries:
  Assuming 10% of DAU searches 5 times/day
  = 50M users × 5 searches
  = 250M searches/day
  = ~2,900 QPS
```

### 3.3 Storage Estimation

```
Tweet Storage:
  Average tweet size:
    - tweet_id: 8 bytes
    - user_id: 8 bytes
    - text: 280 bytes (max)
    - created_at: 8 bytes
    - metadata: 100 bytes
    Total: ~400 bytes per tweet

  Daily tweet storage:
    500M tweets × 400 bytes = 200 GB/day

  5-year storage:
    200 GB × 365 × 5 = 365 TB

Media Storage:
  30% tweets have media
  Average media size: 1 MB
  Daily: 500M × 0.3 × 1 MB = 150 TB/day
  5-year: 150 TB × 365 × 5 = 274 PB

User Data:
  800M users × 1 KB/user = 800 GB

Follow Relationships:
  Average followers: 200
  800M × 200 × 16 bytes = 2.56 TB
```

### 3.4 Bandwidth Estimation

```
Incoming (writes):
  Text: 5,800 tweets/sec × 400 bytes = 2.3 MB/s
  Media: 1,740 tweets/sec × 1 MB = 1.74 GB/s
  Total incoming: ~1.75 GB/s

Outgoing (reads):
  Timeline: 58,000 req/sec × 50 tweets × 400 bytes = 1.16 GB/s
  Media: 58,000 req/sec × 10 media × 100 KB = 58 GB/s
  Total outgoing: ~60 GB/s
```

---

## 4. High-Level Design

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                   CLIENTS                                        │
│                    (Web App, iOS App, Android App, API)                         │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              LOAD BALANCER (L7)                                  │
│                           (AWS ALB / Nginx / HAProxy)                           │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
            ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
            │   API Gateway │    │   API Gateway │    │   API Gateway │
            │   (Rate Limit)│    │   (Auth)      │    │   (Routing)   │
            └──────────────┘    └──────────────┘    └──────────────┘
                    │                   │                   │
                    └───────────────────┼───────────────────┘
                                        │
        ┌───────────────┬───────────────┼───────────────┬───────────────┐
        ▼               ▼               ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│    Tweet     │ │   Timeline   │ │    User      │ │    Search    │ │ Notification │
│   Service    │ │   Service    │ │   Service    │ │   Service    │ │   Service    │
│              │ │              │ │              │ │              │ │              │
│ - Create     │ │ - Home Feed  │ │ - Profile    │ │ - Tweet      │ │ - Push       │
│ - Delete     │ │ - User Feed  │ │ - Follow     │ │ - User       │ │ - Email      │
│ - Like       │ │ - Fan-out    │ │ - Settings   │ │ - Trending   │ │ - In-app     │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
        │               │               │               │               │
        │               │               │               │               │
        ▼               ▼               ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            MESSAGE QUEUE (Kafka)                                 │
│                                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │tweet-created│  │user-followed│  │tweet-liked  │  │tweet-search │            │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘            │
└─────────────────────────────────────────────────────────────────────────────────┘
        │               │               │               │
        ▼               ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Fan-out    │ │   Social     │ │  Analytics   │ │   Search     │
│   Worker     │ │   Graph      │ │   Worker     │ │   Indexer    │
│              │ │   Worker     │ │              │ │              │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
        │               │               │               │
        ▼               ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               DATA STORES                                        │
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │   Tweet Store   │  │  Social Graph   │  │  Timeline Cache │                 │
│  │   (Cassandra)   │  │    (MySQL +     │  │     (Redis)     │                 │
│  │                 │  │     Redis)      │  │                 │                 │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                 │
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │  User Store     │  │  Search Index   │  │   Media Store   │                 │
│  │    (MySQL)      │  │ (Elasticsearch) │  │      (S3 +      │                 │
│  │                 │  │                 │  │      CDN)       │                 │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Component Overview

| Component | Technology | Purpose |
|-----------|------------|---------|
| API Gateway | Kong/Custom | Rate limiting, auth, routing |
| Tweet Service | Go/Java | Tweet CRUD operations |
| Timeline Service | Go/Java | Feed generation and caching |
| User Service | Go/Java | User management, social graph |
| Search Service | Go/Java | Full-text search |
| Fan-out Worker | Go | Distribute tweets to followers |
| Message Queue | Kafka | Async processing, decoupling |
| Tweet Store | Cassandra | High-write tweet storage |
| Timeline Cache | Redis | Pre-computed feeds |
| Search Index | Elasticsearch | Full-text search |
| Media Store | S3 + CloudFront | Images, videos |

---

## 5. API Design

### 5.1 Tweet APIs

```yaml
# Create Tweet
POST /api/v1/tweets
Headers:
  Authorization: Bearer <token>
Body:
  {
    "text": "Hello World! #firsttweet",
    "media_ids": ["media_123", "media_456"],
    "reply_to_tweet_id": "tweet_789",  # optional
    "quote_tweet_id": "tweet_101"       # optional
  }
Response: 201 Created
  {
    "tweet_id": "tweet_12345",
    "text": "Hello World! #firsttweet",
    "user": { "id": "user_1", "username": "johndoe" },
    "created_at": "2024-01-15T10:30:00Z",
    "media": [...],
    "metrics": { "likes": 0, "retweets": 0, "replies": 0 }
  }

# Delete Tweet
DELETE /api/v1/tweets/{tweet_id}
Response: 204 No Content

# Get Tweet
GET /api/v1/tweets/{tweet_id}
Response: 200 OK
  { "tweet_id": "...", "text": "...", ... }

# Like Tweet
POST /api/v1/tweets/{tweet_id}/like
Response: 200 OK

# Retweet
POST /api/v1/tweets/{tweet_id}/retweet
Body:
  { "comment": "Great tweet!" }  # optional for quote tweet
Response: 201 Created
```

### 5.2 Timeline APIs

```yaml
# Get Home Timeline
GET /api/v1/timeline/home?cursor={cursor}&limit={limit}
Response: 200 OK
  {
    "tweets": [
      { "tweet_id": "...", "text": "...", "user": {...}, ... },
      ...
    ],
    "next_cursor": "cursor_abc123",
    "has_more": true
  }

# Get User Timeline
GET /api/v1/users/{user_id}/tweets?cursor={cursor}&limit={limit}
Response: 200 OK
  {
    "tweets": [...],
    "next_cursor": "...",
    "has_more": true
  }
```

### 5.3 User & Social APIs

```yaml
# Follow User
POST /api/v1/users/{user_id}/follow
Response: 200 OK

# Unfollow User
DELETE /api/v1/users/{user_id}/follow
Response: 204 No Content

# Get Followers
GET /api/v1/users/{user_id}/followers?cursor={cursor}&limit={limit}
Response: 200 OK
  {
    "users": [...],
    "next_cursor": "...",
    "has_more": true
  }

# Get Following
GET /api/v1/users/{user_id}/following?cursor={cursor}&limit={limit}
Response: 200 OK
```

### 5.4 Search APIs

```yaml
# Search Tweets
GET /api/v1/search/tweets?q={query}&cursor={cursor}&limit={limit}
Response: 200 OK
  {
    "tweets": [...],
    "next_cursor": "...",
    "has_more": true
  }

# Get Trending Topics
GET /api/v1/trends?location={location}
Response: 200 OK
  {
    "trends": [
      { "name": "#SystemDesign", "tweet_count": 125000 },
      { "name": "#TechNews", "tweet_count": 98000 },
      ...
    ]
  }
```

---

## 6. Database Schema

### 6.1 User Store (MySQL - Sharded by user_id)

```sql
-- Users Table
CREATE TABLE users (
    user_id         BIGINT PRIMARY KEY,
    username        VARCHAR(50) UNIQUE NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    display_name    VARCHAR(100),
    bio             TEXT,
    profile_image   VARCHAR(500),
    banner_image    VARCHAR(500),
    location        VARCHAR(100),
    website         VARCHAR(200),
    is_verified     BOOLEAN DEFAULT FALSE,
    is_private      BOOLEAN DEFAULT FALSE,
    followers_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    tweets_count    INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_username (username),
    INDEX idx_email (email)
);

-- Social Graph (Followers/Following)
CREATE TABLE follows (
    follower_id     BIGINT NOT NULL,
    followee_id     BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id, follower_id),
    INDEX idx_created (follower_id, created_at)
);
```

### 6.2 Tweet Store (Cassandra)

```sql
-- Tweets by ID (main table)
CREATE TABLE tweets (
    tweet_id        BIGINT,
    user_id         BIGINT,
    text            TEXT,
    reply_to_id     BIGINT,
    quote_tweet_id  BIGINT,
    media_urls      LIST<TEXT>,
    hashtags        SET<TEXT>,
    mentions        SET<BIGINT>,
    likes_count     COUNTER,
    retweets_count  COUNTER,
    replies_count   COUNTER,
    created_at      TIMESTAMP,

    PRIMARY KEY (tweet_id)
);

-- User Timeline (tweets by a user)
CREATE TABLE user_tweets (
    user_id         BIGINT,
    created_at      TIMESTAMP,
    tweet_id        BIGINT,

    PRIMARY KEY (user_id, created_at, tweet_id)
) WITH CLUSTERING ORDER BY (created_at DESC, tweet_id DESC);

-- Likes
CREATE TABLE tweet_likes (
    tweet_id        BIGINT,
    user_id         BIGINT,
    created_at      TIMESTAMP,

    PRIMARY KEY (tweet_id, user_id)
);

CREATE TABLE user_likes (
    user_id         BIGINT,
    created_at      TIMESTAMP,
    tweet_id        BIGINT,

    PRIMARY KEY (user_id, created_at, tweet_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### 6.3 Timeline Cache (Redis)

```
# Home Timeline (Pre-computed feed per user)
Key: timeline:{user_id}
Type: Sorted Set
Score: tweet_timestamp
Value: tweet_id

# Example
ZADD timeline:user_123 1705312200 tweet_456
ZADD timeline:user_123 1705312100 tweet_789

# Query latest 50 tweets
ZREVRANGE timeline:user_123 0 49

# User's following list cache
Key: following:{user_id}
Type: Set
Value: followee_user_ids

# Celebrity/VIP tweet cache
Key: celebrity_tweets:{user_id}
Type: Sorted Set
Score: tweet_timestamp
Value: tweet_id
```

### 6.4 Search Index (Elasticsearch)

```json
{
  "mappings": {
    "properties": {
      "tweet_id": { "type": "keyword" },
      "user_id": { "type": "keyword" },
      "username": { "type": "keyword" },
      "text": {
        "type": "text",
        "analyzer": "standard"
      },
      "hashtags": { "type": "keyword" },
      "mentions": { "type": "keyword" },
      "created_at": { "type": "date" },
      "likes_count": { "type": "integer" },
      "retweets_count": { "type": "integer" },
      "language": { "type": "keyword" },
      "location": { "type": "geo_point" }
    }
  }
}
```

---

## 7. Deep Dive: Timeline Generation (Fan-out)

### 7.1 The Fan-out Problem

When a user tweets, their followers need to see it in their timeline. Two approaches:

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **Fan-out on Write** | Push tweet to all followers' timelines at write time | Fast reads | Slow writes for celebrities |
| **Fan-out on Read** | Pull tweets from followed users at read time | Fast writes | Slow reads |

### 7.2 Hybrid Approach (Twitter's Solution)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TWEET CREATED                                 │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │   Is user a celebrity │
                    │   (>10K followers)?   │
                    └───────────────────────┘
                          │           │
                         Yes          No
                          │           │
                          ▼           ▼
              ┌─────────────────┐  ┌─────────────────┐
              │ Store in Tweet  │  │   Fan-out to    │
              │ DB only (pull)  │  │   all followers │
              │                 │  │  (push to Redis)│
              └─────────────────┘  └─────────────────┘
                          │           │
                          └─────┬─────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     TIMELINE READ REQUEST                            │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │  Get pre-computed     │
                    │  timeline from Redis  │
                    └───────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │  Does user follow     │
                    │  any celebrities?     │
                    └───────────────────────┘
                          │           │
                         Yes          No
                          │           │
                          ▼           ▼
              ┌─────────────────┐  ┌─────────────────┐
              │ Fetch celebrity │  │  Return cached  │
              │ tweets from DB  │  │    timeline     │
              │ and merge       │  │                 │
              └─────────────────┘  └─────────────────┘
                          │           │
                          └─────┬─────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │   Return merged &     │
                    │   sorted timeline     │
                    └───────────────────────┘
```

### 7.3 Fan-out Worker Implementation

```python
class FanoutWorker:
    CELEBRITY_THRESHOLD = 10000
    BATCH_SIZE = 1000
    TIMELINE_MAX_SIZE = 800

    def process_tweet(self, tweet_event):
        tweet_id = tweet_event['tweet_id']
        author_id = tweet_event['author_id']
        created_at = tweet_event['created_at']

        # Get author's follower count
        follower_count = self.user_service.get_follower_count(author_id)

        if follower_count > self.CELEBRITY_THRESHOLD:
            # Celebrity: Store only in main tweet store
            # Followers will pull during read
            self.mark_as_celebrity_tweet(tweet_id, author_id)
            return

        # Regular user: Fan-out to all followers
        followers = self.get_all_followers(author_id)

        for batch in self.batch_iterator(followers, self.BATCH_SIZE):
            self.push_to_timelines(batch, tweet_id, created_at)

    def push_to_timelines(self, user_ids, tweet_id, timestamp):
        pipeline = self.redis.pipeline()

        for user_id in user_ids:
            timeline_key = f"timeline:{user_id}"

            # Add tweet to timeline
            pipeline.zadd(timeline_key, {tweet_id: timestamp})

            # Trim to max size
            pipeline.zremrangebyrank(timeline_key, 0, -self.TIMELINE_MAX_SIZE - 1)

        pipeline.execute()

    def get_timeline(self, user_id, limit=50, cursor=None):
        # Get pre-computed timeline
        timeline_key = f"timeline:{user_id}"

        if cursor:
            max_score = cursor
            tweet_ids = self.redis.zrevrangebyscore(
                timeline_key, max_score, '-inf', start=0, num=limit
            )
        else:
            tweet_ids = self.redis.zrevrange(timeline_key, 0, limit - 1)

        # Check if user follows celebrities
        celebrity_followees = self.get_celebrity_followees(user_id)

        if celebrity_followees:
            # Fetch recent celebrity tweets
            celebrity_tweets = self.fetch_celebrity_tweets(
                celebrity_followees, limit, cursor
            )
            # Merge and sort
            tweet_ids = self.merge_and_sort(tweet_ids, celebrity_tweets)[:limit]

        # Fetch full tweet objects
        tweets = self.tweet_service.get_tweets_by_ids(tweet_ids)

        return tweets
```

### 7.4 Timeline Cache Strategy

```
Timeline Cache Design:

1. Cache Size per User:
   - Store last 800 tweet IDs in Redis
   - Each entry: tweet_id (8 bytes) + score (8 bytes) = 16 bytes
   - Per user: 800 × 16 = 12.8 KB
   - 500M users: 6.4 TB Redis (sharded)

2. Cache Eviction:
   - LRU eviction for inactive users (>7 days no login)
   - Keep hot users (active in last 24h) always cached

3. Cache Warming:
   - On user login, if cache miss:
     a. Get following list
     b. Query recent tweets from each followee
     c. Merge and cache top 800

4. Cache Invalidation:
   - Tweet deleted: Remove from author's followers' timelines
   - User blocked: Remove blocker's tweets from blocked user's timeline
   - Account deactivated: Background job cleans up
```

---

## 8. Scaling Considerations

### 8.1 Database Sharding Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SHARDING STRATEGY                             │
└─────────────────────────────────────────────────────────────────────┘

1. User Data Sharding (MySQL):
   - Shard key: user_id
   - Shards: 256 shards
   - Routing: user_id % 256
   - Each shard: ~3M users

2. Tweet Data Sharding (Cassandra):
   - Partition key: tweet_id
   - Consistent hashing with virtual nodes
   - Replication factor: 3

3. Timeline Cache Sharding (Redis Cluster):
   - Shard key: user_id
   - 100 Redis nodes
   - Each node: 64 GB RAM
   - Total: 6.4 TB

4. Social Graph Sharding:
   - Primary shard: follower_id
   - Secondary index: followee_id
   - Cross-shard queries for "get followers"
```

### 8.2 Geographic Distribution

```
┌─────────────────────────────────────────────────────────────────────┐
│                     GLOBAL ARCHITECTURE                              │
└─────────────────────────────────────────────────────────────────────┘

                         ┌───────────────┐
                         │   Global DNS  │
                         │  (Route 53)   │
                         └───────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
     ┌────────────┐      ┌────────────┐      ┌────────────┐
     │  US-EAST   │      │  EU-WEST   │      │ AP-SOUTH   │
     │   Region   │      │   Region   │      │   Region   │
     └────────────┘      └────────────┘      └────────────┘
            │                   │                   │
            │     Cross-Region Replication          │
            └───────────────────┼───────────────────┘
                                │
                         ┌─────────────┐
                         │   Primary   │
                         │  US-EAST    │
                         └─────────────┘

- Writes: Routed to primary region (US-EAST)
- Reads: Served from nearest region
- Timeline cache: Replicated to all regions
- Replication lag: ~100-500ms acceptable
```

### 8.3 Caching Strategy

| Cache Layer | Technology | Purpose | TTL |
|-------------|------------|---------|-----|
| **CDN** | CloudFront | Static assets, media | 24h |
| **API Cache** | Varnish | Common API responses | 60s |
| **Timeline Cache** | Redis Cluster | Pre-computed feeds | Persistent |
| **Tweet Cache** | Redis | Hot tweets | 24h |
| **User Cache** | Redis | User profiles | 1h |
| **Search Cache** | Elasticsearch | Query results | 5m |

---

## 9. Trade-offs Discussed

### 9.1 Consistency vs Availability

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Timeline delivery** | Eventual consistency | Missing a tweet for a few seconds is acceptable |
| **Like/Retweet counts** | Eventual consistency | Counts can be slightly stale |
| **User profile** | Strong consistency | Profile changes should be immediate |
| **Tweet creation** | Strong consistency | Tweet must not be lost |

### 9.2 Push vs Pull Trade-offs

| Scenario | Approach | Reason |
|----------|----------|--------|
| Regular users (<10K followers) | Push (fan-out on write) | Fast reads, manageable write load |
| Celebrities (>10K followers) | Pull (fan-out on read) | Avoid write amplification |
| Inactive users (>7 days) | Pull | Save storage, rebuild on demand |

### 9.3 Storage Trade-offs

| Choice | Decision | Trade-off |
|--------|----------|-----------|
| **Cassandra for tweets** | Yes | High write throughput, eventual consistency |
| **MySQL for users** | Yes | Strong consistency, complex queries |
| **Redis for timelines** | Yes | Speed vs cost (expensive at scale) |
| **Denormalization** | Heavy | Query speed vs storage cost |

---

## 10. Bottlenecks & Solutions

### 10.1 Hot Partition Problem

**Problem**: Celebrity tweets cause hot partitions in the timeline cache.

**Solution**:
```
1. Hybrid fan-out (already discussed)
2. Cache celebrity tweets separately
3. Read replicas for hot data

Celebrity Tweet Cache:
┌─────────────────┐
│ celebrity_tweets│ ──► Replicated across
│   :{user_id}    │     all Redis nodes
└─────────────────┘
```

### 10.2 Thundering Herd

**Problem**: Many users refreshing timeline simultaneously (e.g., after downtime).

**Solution**:
```python
def get_timeline_with_jitter(user_id):
    cache_key = f"timeline:{user_id}"

    # Try cache first
    cached = redis.get(cache_key)
    if cached:
        return cached

    # Cache miss - use distributed lock with jitter
    lock_key = f"lock:timeline:{user_id}"

    # Add random jitter (0-100ms) to spread load
    time.sleep(random.uniform(0, 0.1))

    if redis.setnx(lock_key, 1, ex=5):
        # This instance rebuilds cache
        timeline = rebuild_timeline(user_id)
        redis.setex(cache_key, 3600, timeline)
        redis.delete(lock_key)
        return timeline
    else:
        # Wait for other instance to rebuild
        return wait_for_cache(cache_key)
```

### 10.3 Database Write Amplification

**Problem**: One tweet = N writes (one per follower).

**Solution**:
```
1. Batch writes to Redis (pipeline)
2. Async processing via Kafka
3. Celebrity hybrid approach
4. Limit timeline size (800 tweets)

Write Amplification Calculation:
- Average followers: 200
- Tweets per second: 5,800
- Without optimization: 5,800 × 200 = 1.16M Redis writes/sec

With celebrity cutoff (10K followers):
- 95% users < 10K followers
- Effective writes: 5,510 × 200 = 1.1M writes/sec
- Celebrity tweets (5%): Stored once = 290 writes/sec
- Total: ~1.1M writes/sec (manageable with Redis cluster)
```

### 10.4 Search Indexing Lag

**Problem**: New tweets not immediately searchable.

**Solution**:
```
1. Near real-time indexing via Kafka
2. Two-tier search:
   - Hot index: Last 24h tweets (fast updates)
   - Cold index: Older tweets (batch updates)

┌─────────┐     ┌─────────┐     ┌──────────────────┐
│  Tweet  │────►│  Kafka  │────►│  Search Indexer  │
│ Service │     │         │     │  (< 30s latency) │
└─────────┘     └─────────┘     └──────────────────┘
                                        │
                          ┌─────────────┴─────────────┐
                          ▼                           ▼
                   ┌────────────┐              ┌────────────┐
                   │ Hot Index  │              │ Cold Index │
                   │  (24h)     │              │  (Archive) │
                   └────────────┘              └────────────┘
```

---

## 11. Interview Variations and Follow-ups

### 11.1 Common Follow-up Questions

| Question | Key Points to Address |
|----------|----------------------|
| "How would you handle a celebrity with 50M followers?" | Pure pull model, cache their tweets separately, merge at read time |
| "How do you ensure tweets are never lost?" | Write-ahead log, synchronous replication, at-least-once delivery |
| "How would you implement trending topics?" | Sliding window, count-min sketch, geographic segmentation |
| "How do you handle tweet deletion?" | Soft delete, async cleanup of fan-out, tombstone in Cassandra |
| "How would you add real-time updates?" | WebSocket connections, long polling fallback, pub/sub for online users |

### 11.2 Variations

#### Variation 1: Design Twitter with Real-Time Updates

```
Additional Components:
1. WebSocket Gateway for live connections
2. Pub/Sub system (Redis Pub/Sub or custom)
3. Presence service to track online users

Flow:
Tweet Created → Kafka → Fan-out Worker →
  → Check if follower is online
  → If online: Push via WebSocket
  → If offline: Update Redis timeline only
```

#### Variation 2: Design Twitter Search

```
Focus Areas:
1. Inverted index design
2. Ranking algorithm (recency, engagement, relevance)
3. Typeahead suggestions
4. Trending hashtags computation
5. Spam filtering in search results
```

#### Variation 3: Design Twitter DM System

```
Separate System Design:
1. End-to-end encryption
2. Message delivery guarantees
3. Presence (online/offline status)
4. Read receipts
5. Media sharing in DMs
```

### 11.3 Red Flags to Avoid

| Mistake | Why It's Wrong |
|---------|----------------|
| Ignoring celebrity problem | Will cause massive write amplification |
| Single database | Won't scale to Twitter's volume |
| Synchronous fan-out | Will make tweet posting slow |
| Not discussing consistency | Critical for distributed systems |
| Forgetting cache invalidation | Stale data problems |

### 11.4 Good Signs in Your Answer

| Indicator | Why It's Good |
|-----------|---------------|
| Mention hybrid fan-out early | Shows awareness of key challenge |
| Discuss trade-offs clearly | Shows system design maturity |
| Calculate numbers | Shows quantitative thinking |
| Consider failure modes | Shows production readiness |
| Ask clarifying questions | Shows communication skills |

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    TWITTER DESIGN SUMMARY                        │
├─────────────────────────────────────────────────────────────────┤
│ Scale: 500M DAU, 500M tweets/day, 5B timeline reads/day         │
│                                                                  │
│ Key Challenge: Fan-out problem for celebrities                   │
│ Solution: Hybrid push/pull based on follower count               │
│                                                                  │
│ Storage:                                                         │
│   - Tweets: Cassandra (high write throughput)                   │
│   - Users: MySQL (strong consistency)                           │
│   - Timelines: Redis (pre-computed feeds)                       │
│   - Search: Elasticsearch                                        │
│   - Media: S3 + CDN                                             │
│                                                                  │
│ Key Numbers:                                                     │
│   - Tweet write: ~6K/sec (peak 18K/sec)                         │
│   - Timeline read: ~58K/sec (peak 174K/sec)                     │
│   - Timeline latency: <200ms p99                                │
│                                                                  │
│ Trade-offs:                                                      │
│   - Eventual consistency for feeds                              │
│   - Heavy denormalization for read speed                        │
│   - Celebrity tweets pulled at read time                        │
└─────────────────────────────────────────────────────────────────┘
```
