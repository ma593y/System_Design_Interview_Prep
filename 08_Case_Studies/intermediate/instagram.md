# Instagram System Design (Photo Sharing Platform)

## 1. Problem Statement

Design a photo-sharing social network like Instagram where users can upload photos,
follow other users, view a personalized news feed, post Stories (ephemeral 24-hour
content), and interact through likes and comments. The system must handle hundreds
of millions of users, store billions of photos, and deliver a fast, reliable feed
experience worldwide.

**Core Functionality:**
- Upload and share photos with captions and filters
- Follow/unfollow other users
- View a personalized news feed of posts from followed users
- Like and comment on posts
- Post and view Stories (disappear after 24 hours)
- Explore page with trending and recommended content
- Direct messaging (out of scope, see WhatsApp design)

**Real-World Examples:**
- Instagram
- Flickr
- Pinterest (pin-based variant)
- Snapchat (stories-focused variant)

---

## 2. Requirements Clarification

### Functional Requirements

| Requirement | Description |
|-------------|-------------|
| Photo Upload | Upload photos with captions, tags, location |
| News Feed | View chronological/ranked feed from followed users |
| Follow System | Follow/unfollow users, see follower/following counts |
| Likes & Comments | Like and comment on posts |
| Stories | Post ephemeral content that disappears after 24 hours |
| Explore Page | Discover trending and recommended content |
| Image Processing | Generate thumbnails, apply filters, optimize for devices |
| Search | Search for users, hashtags, and locations |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| High Availability | 99.99% uptime |
| Low Latency | Feed loads in < 200ms, photo upload < 2s |
| Scalability | Support 1 billion users, 500M daily active |
| Durability | Photos must never be lost once uploaded |
| Consistency | Eventual consistency acceptable for feeds |
| Global Reach | Low latency worldwide via CDN |

### Out of Scope
- Direct messaging system
- Video upload (short-form like Reels)
- Advertising and monetization platform
- Content rights management

---

## 3. Capacity Estimation

### Assumptions

```
- 1 billion total users, 500 million DAU
- Average user views feed 7 times per day
- 100 million photos uploaded per day
- Average photo size: 2 MB (original)
- Average 5 thumbnails per photo (various sizes): 500 KB total
- Average user follows 200 accounts
- Average post gets 50 likes, 3 comments
- 200 million Stories posted per day, each with 3 images/clips
```

### Traffic Estimates

```
Feed Requests:
- Feed views per day: 500M DAU * 7 views = 3.5 billion/day
- Feed requests per second: 3.5B / 86,400 ≈ 40,500 req/sec
- Peak (3x): ~121,000 req/sec

Photo Uploads:
- Uploads per day: 100 million
- Uploads per second: 100M / 86,400 ≈ 1,160 uploads/sec
- Peak (3x): ~3,500 uploads/sec

Like Operations:
- Assume each DAU likes 10 posts/day
- Likes per second: 500M * 10 / 86,400 ≈ 57,870 likes/sec

Stories Views:
- Assume each DAU views 30 stories/day
- Story views per second: 500M * 30 / 86,400 ≈ 173,600 req/sec
```

### Storage Estimates

```
Photo Storage (per day):
- Original photos: 100M * 2 MB = 200 TB/day
- Thumbnails: 100M * 500 KB = 50 TB/day
- Total daily: 250 TB/day

Photo Storage (per year):
- 250 TB/day * 365 = 91.25 PB/year

Stories Storage:
- 200M stories * 3 images * 500 KB = 300 TB/day
- Deleted after 24 hours, so steady-state: ~300 TB

Metadata Storage (per year):
- Posts table: 100M posts/day * 500 bytes = 50 GB/day = 18 TB/year
- Users table: 1B users * 1 KB = 1 TB (relatively static)
- Follows: ~100B follow relationships * 16 bytes = 1.6 TB
- Likes: ~5B likes/day * 24 bytes = 120 GB/day = 43 TB/year
- Comments: ~300M comments/day * 500 bytes = 150 GB/day = 54 TB/year
```

### Bandwidth Estimates

```
Upload bandwidth:
- 1,160 uploads/sec * 2 MB = 2.32 GB/s

Download bandwidth (feed images):
- 40,500 feed req/sec * 10 images/feed * 200 KB avg = 81 GB/s
- Most served from CDN, reducing origin load by ~95%
- Origin bandwidth: ~4 GB/s
```

---

## 4. High-Level Design

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              Clients                                     │
│                    (iOS / Android / Web)                                 │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │    API Gateway /    │
                    │    Load Balancer    │
                    │   (Rate Limiting)   │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────────┐
         │                     │                         │
┌────────▼────────┐  ┌────────▼────────┐   ┌────────────▼──────────┐
│  Post Service   │  │  Feed Service   │   │  User Service         │
│  (Upload,       │  │  (Generation,   │   │  (Profile, Follow,    │
│   Metadata)     │  │   Ranking)      │   │   Search)             │
└───┬─────────┬───┘  └───┬─────────┬───┘   └───┬──────────────────┘
    │         │          │         │            │
    │    ┌────▼──────┐   │    ┌────▼──────┐    │
    │    │  Image    │   │    │  Feed     │    │
    │    │ Processing│   │    │  Cache    │    │
    │    │  Pipeline │   │    │  (Redis)  │    │
    │    └────┬──────┘   │    └───────────┘    │
    │         │          │                     │
    │    ┌────▼──────┐   │                     │
    │    │  Object   │   │                     │
    │    │  Storage  │   │                     │
    │    │  (S3)     │   │                     │
    │    └────┬──────┘   │                     │
    │         │          │                     │
    │    ┌────▼──────┐   │                     │
    │    │   CDN     │   │                     │
    │    │(CloudFront│   │                     │
    │    │ /Akamai)  │   │                     │
    │    └───────────┘   │                     │
    │                    │                     │
┌───▼────────────────────▼─────────────────────▼───┐
│              Database Layer                       │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────┐  │
│  │  Posts DB   │  │  Social Graph│  │ User DB │  │
│  │  (Postgres  │  │  (Cassandra/ │  │(Postgres│  │
│  │   sharded)  │  │   Neo4j)     │  │ /MySQL) │  │
│  └─────────────┘  └──────────────┘  └─────────┘  │
└──────────────────────────────────────────────────┘
         │
┌────────▼─────────────────────────────────────────┐
│              Async Processing                     │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────┐  │
│  │  Message    │  │  Feed Fan-out│  │ Content │  │
│  │  Queue      │  │  Workers     │  │Moderation│ │
│  │  (Kafka)    │  │              │  │  (ML)   │  │
│  └─────────────┘  └──────────────┘  └─────────┘  │
└──────────────────────────────────────────────────┘
```

### Photo Upload Flow

```
┌────────┐     ┌──────────┐     ┌──────────┐     ┌────────────┐
│ Client │────▶│ API      │────▶│ Post     │────▶│ Object     │
│        │     │ Gateway  │     │ Service  │     │ Storage    │
└────────┘     └──────────┘     └────┬─────┘     │ (S3)       │
                                     │           └──────┬─────┘
                                     │                  │
                                     ▼                  │
                              ┌──────────┐              │
                              │ Metadata │              │
                              │ DB Write │              │
                              └────┬─────┘              │
                                   │                    │
                                   ▼                    ▼
                              ┌──────────┐     ┌──────────────┐
                              │  Kafka   │     │   Image      │
                              │  Event   │     │  Processing  │
                              └────┬─────┘     │  Pipeline    │
                                   │           │ (thumbnails, │
                          ┌────────┼────────┐  │  filters)    │
                          │        │        │  └──────┬───────┘
                          ▼        ▼        ▼         │
                    ┌────────┐ ┌──────┐ ┌──────┐      ▼
                    │Fan-out │ │Notify│ │Search│  ┌────────┐
                    │to Feed │ │Follow│ │Index │  │  CDN   │
                    │Cache   │ │ers   │ │Update│  │ Upload │
                    └────────┘ └──────┘ └──────┘  └────────┘
```

---

## 5. API Design

### Photo Upload

```
POST /api/v1/posts
Headers:
  Authorization: Bearer <token>
  Content-Type: multipart/form-data
Body:
  photo: <binary image data>
  caption: "Sunset at the beach #vacation"
  location: {"lat": 37.7749, "lng": -122.4194, "name": "San Francisco"}
  tags: ["user_123", "user_456"]
  filter: "clarendon"

Response 201:
{
  "post_id": "post_98765",
  "photo_url": "https://cdn.instagram.com/p/98765/original.jpg",
  "thumbnails": {
    "150x150": "https://cdn.instagram.com/p/98765/t150.jpg",
    "320x320": "https://cdn.instagram.com/p/98765/t320.jpg",
    "640x640": "https://cdn.instagram.com/p/98765/t640.jpg",
    "1080x1080": "https://cdn.instagram.com/p/98765/t1080.jpg"
  },
  "created_at": "2024-01-25T10:30:00Z",
  "status": "processing"
}
```

### News Feed

```
GET /api/v1/feed?cursor=<cursor>&limit=20
Headers:
  Authorization: Bearer <token>

Response 200:
{
  "posts": [
    {
      "post_id": "post_98765",
      "user": {"id": "user_111", "username": "john", "avatar_url": "..."},
      "photo_url": "https://cdn.instagram.com/p/98765/t1080.jpg",
      "caption": "Sunset at the beach #vacation",
      "location": {"name": "San Francisco"},
      "likes_count": 1523,
      "comments_count": 42,
      "has_liked": false,
      "created_at": "2024-01-25T10:30:00Z"
    }
  ],
  "next_cursor": "eyJsYXN0X3Bvc3RfaWQiOiI5ODc2NCJ9",
  "has_more": true
}
```

### Follow/Unfollow

```
POST /api/v1/users/{user_id}/follow
Headers:
  Authorization: Bearer <token>

Response 200:
{
  "status": "following",
  "follower_count": 1500001,
  "following_count": 200
}

DELETE /api/v1/users/{user_id}/follow
Response 200:
{
  "status": "unfollowed"
}
```

### Like/Unlike

```
POST /api/v1/posts/{post_id}/like
Headers:
  Authorization: Bearer <token>

Response 200:
{
  "likes_count": 1524,
  "has_liked": true
}
```

### Stories

```
POST /api/v1/stories
Headers:
  Authorization: Bearer <token>
  Content-Type: multipart/form-data
Body:
  media: <binary data>
  media_type: "image"
  stickers: [{"type": "poll", "question": "Beach or Mountains?", ...}]

Response 201:
{
  "story_id": "story_55555",
  "media_url": "https://cdn.instagram.com/s/55555/media.jpg",
  "expires_at": "2024-01-26T10:30:00Z"
}

GET /api/v1/stories/feed
Response 200:
{
  "story_trays": [
    {
      "user": {"id": "user_111", "username": "john", "avatar_url": "..."},
      "stories": [
        {
          "story_id": "story_55555",
          "media_url": "...",
          "created_at": "2024-01-25T10:30:00Z",
          "expires_at": "2024-01-26T10:30:00Z",
          "seen": false
        }
      ]
    }
  ]
}
```

---

## 6. Database Schema

### Posts Table (Sharded by user_id)

```
Table: posts
┌─────────────────┬──────────────┬──────────────────────────────┐
│ Column          │ Type         │ Notes                        │
├─────────────────┼──────────────┼──────────────────────────────┤
│ post_id         │ BIGINT (PK)  │ Snowflake ID                 │
│ user_id         │ BIGINT       │ FK to users, shard key       │
│ photo_url       │ VARCHAR(512) │ S3 URL of original photo     │
│ caption         │ TEXT         │ Up to 2200 chars             │
│ filter_type     │ VARCHAR(32)  │ Applied filter name          │
│ latitude        │ DECIMAL(9,6) │ Location latitude            │
│ longitude       │ DECIMAL(9,6) │ Location longitude           │
│ location_name   │ VARCHAR(256) │ Human-readable location      │
│ likes_count     │ INT          │ Denormalized counter         │
│ comments_count  │ INT          │ Denormalized counter         │
│ is_deleted      │ BOOLEAN      │ Soft delete flag             │
│ created_at      │ TIMESTAMP    │ Creation timestamp           │
│ updated_at      │ TIMESTAMP    │ Last update timestamp        │
└─────────────────┴──────────────┴──────────────────────────────┘
Indexes:
  - PRIMARY KEY (post_id)
  - INDEX idx_user_posts (user_id, created_at DESC)
  - INDEX idx_location (latitude, longitude)
```

### Users Table

```
Table: users
┌─────────────────┬──────────────┬──────────────────────────────┐
│ Column          │ Type         │ Notes                        │
├─────────────────┼──────────────┼──────────────────────────────┤
│ user_id         │ BIGINT (PK)  │ Snowflake ID                 │
│ username        │ VARCHAR(30)  │ Unique username              │
│ email           │ VARCHAR(256) │ Unique email                 │
│ display_name    │ VARCHAR(64)  │ Display name                 │
│ bio             │ VARCHAR(150) │ Profile bio                  │
│ avatar_url      │ VARCHAR(512) │ Profile picture URL          │
│ follower_count  │ INT          │ Denormalized counter         │
│ following_count │ INT          │ Denormalized counter         │
│ post_count      │ INT          │ Denormalized counter         │
│ is_verified     │ BOOLEAN      │ Blue checkmark               │
│ is_private      │ BOOLEAN      │ Private account flag         │
│ is_celebrity    │ BOOLEAN      │ Celebrity flag for fan-out   │
│ created_at      │ TIMESTAMP    │ Registration timestamp       │
└─────────────────┴──────────────┴──────────────────────────────┘
Indexes:
  - PRIMARY KEY (user_id)
  - UNIQUE INDEX idx_username (username)
  - UNIQUE INDEX idx_email (email)
```

### Follows Table (Social Graph - Cassandra)

```
Table: user_follows (who does user X follow?)
┌─────────────────┬──────────────┬──────────────────────────────┐
│ Column          │ Type         │ Notes                        │
├─────────────────┼──────────────┼──────────────────────────────┤
│ follower_id     │ BIGINT       │ Partition key                │
│ followee_id     │ BIGINT       │ Clustering key               │
│ created_at      │ TIMESTAMP    │ When the follow happened     │
└─────────────────┴──────────────┴──────────────────────────────┘
Primary Key: (follower_id, followee_id)

Table: user_followers (who follows user X?)
┌─────────────────┬──────────────┬──────────────────────────────┐
│ Column          │ Type         │ Notes                        │
├─────────────────┼──────────────┼──────────────────────────────┤
│ followee_id     │ BIGINT       │ Partition key                │
│ follower_id     │ BIGINT       │ Clustering key               │
│ created_at      │ TIMESTAMP    │ When the follow happened     │
└─────────────────┴──────────────┴──────────────────────────────┘
Primary Key: (followee_id, follower_id)
```

### Feed Cache (Redis)

```
Key: feed:{user_id}
Type: Sorted Set (score = timestamp)
Value: post_id

Example:
  ZADD feed:user_42 1706234567 post_98765
  ZADD feed:user_42 1706234500 post_98764

  ZREVRANGE feed:user_42 0 19  // Get latest 20 posts
```

### Stories Table

```
Table: stories
┌─────────────────┬──────────────┬──────────────────────────────┐
│ Column          │ Type         │ Notes                        │
├─────────────────┼──────────────┼──────────────────────────────┤
│ story_id        │ BIGINT (PK)  │ Snowflake ID                 │
│ user_id         │ BIGINT       │ FK to users                  │
│ media_url       │ VARCHAR(512) │ S3 URL                       │
│ media_type      │ ENUM         │ image, video                 │
│ sticker_data    │ JSON         │ Sticker/poll metadata        │
│ view_count      │ INT          │ Number of views              │
│ created_at      │ TIMESTAMP    │ Creation timestamp           │
│ expires_at      │ TIMESTAMP    │ Auto-delete after 24h        │
└─────────────────┴──────────────┴──────────────────────────────┘
Indexes:
  - PRIMARY KEY (story_id)
  - INDEX idx_user_stories (user_id, created_at DESC)
  - INDEX idx_expiry (expires_at)   -- for TTL cleanup job
```

---

## 7. Deep Dive

### 7.1 News Feed Generation: Fan-Out on Write vs Read

The feed is the core of Instagram. Two fundamental approaches exist:

**Fan-Out on Write (Push Model):**

```
When User A creates a post:

1. Post saved to Posts DB
2. Fetch all followers of User A from Social Graph
3. For each follower, push post_id into their feed cache

User A posts ──▶ Fetch 1000 followers ──▶ Write to 1000 feed caches

┌──────────┐     ┌──────────┐     ┌────────────────────────┐
│ User A   │────▶│ Fan-out  │────▶│ Redis Feed Caches      │
│ posts    │     │ Worker   │     │                        │
└──────────┘     └──────────┘     │ feed:user_1 [post_A]   │
                                  │ feed:user_2 [post_A]   │
                                  │ feed:user_3 [post_A]   │
                                  │ ...                    │
                                  │ feed:user_1000 [post_A]│
                                  └────────────────────────┘

Feed Read: Simply read from feed:user_X sorted set in Redis.
           O(1) lookup, very fast.

Pros:
+ Feed reads are extremely fast (pre-computed)
+ Simple read path
+ Low read latency

Cons:
- Celebrity problem: user with 100M followers = 100M writes per post
- Wastes storage for inactive users
- Delayed visibility (fan-out takes time)
```

**Fan-Out on Read (Pull Model):**

```
When User B opens their feed:

1. Fetch list of users B follows
2. For each followed user, get their latest N posts
3. Merge and rank all posts
4. Return top posts

User B opens feed ──▶ Fetch 200 following ──▶ Query 200 users' posts ──▶ Merge

Pros:
+ No wasted work for inactive users
+ Celebrity posting is instant (no fan-out)
+ Always fresh data

Cons:
- Feed reads are slow (N queries per feed load)
- High read latency at scale
- Heavy load on Posts DB
```

**Hybrid Approach (Instagram's Strategy):**

```
┌─────────────────────────────────────────────────────────┐
│                   Hybrid Fan-Out                         │
│                                                          │
│  User Type      │  Strategy                              │
│  ────────────   │  ────────                              │
│  Regular User   │  Fan-out on WRITE                      │
│  (< 10K follow) │  Push to all followers' feed caches    │
│                 │                                        │
│  Celebrity      │  Fan-out on READ                       │
│  (> 10K follow) │  Merge at read time for each reader    │
│                 │                                        │
│  Feed Read:                                              │
│  1. Read pre-computed feed from Redis (regular users)    │
│  2. Fetch latest posts from followed celebrities         │
│  3. Merge both lists, rank by score                      │
│  4. Return top N posts                                   │
└─────────────────────────────────────────────────────────┘

Celebrity threshold: 10,000 followers (configurable)

Feed Read Flow with Hybrid:
┌────────┐     ┌───────────┐     ┌─────────────┐
│ Client │────▶│Feed       │────▶│ Redis Cache  │──▶ Pre-computed feed
│        │     │Service    │     │ (regular     │    (posts from regular
└────────┘     │           │     │  user posts) │    users you follow)
               │           │     └─────────────┘
               │           │     ┌─────────────┐
               │           │────▶│ Celebrity    │──▶ Latest posts from
               │           │     │ Posts Cache  │    celebrities you follow
               │           │     └─────────────┘
               │           │
               │  MERGE + RANK by engagement score
               │           │
               └───────────┘
```

### 7.2 Image Processing Pipeline

```
Photo Upload ──▶ Original stored in S3

Then async pipeline via message queue:

┌─────────────┐     ┌──────────────┐     ┌──────────────────────┐
│  Original   │────▶│  Validation  │────▶│  Content Moderation  │
│  Upload     │     │  (format,    │     │  (NSFW detection,    │
│  (S3)       │     │   size, EXIF)│     │   violence, spam)    │
└─────────────┘     └──────────────┘     └──────────┬───────────┘
                                                     │
                                          ┌──────────▼───────────┐
                                          │   Filter Application │
                                          │   (if user selected  │
                                          │    a filter)         │
                                          └──────────┬───────────┘
                                                     │
                                          ┌──────────▼───────────┐
                                          │  Thumbnail Generation│
                                          │                      │
                                          │  150x150  (avatar)   │
                                          │  320x320  (feed low) │
                                          │  640x640  (feed med) │
                                          │  1080x1080 (feed hi) │
                                          │  1080x1350 (portrait)│
                                          └──────────┬───────────┘
                                                     │
                                          ┌──────────▼───────────┐
                                          │  Upload to CDN       │
                                          │  (multiple edge      │
                                          │   locations)         │
                                          └──────────┬───────────┘
                                                     │
                                          ┌──────────▼───────────┐
                                          │  Update Post Status  │
                                          │  "processing" ->     │
                                          │  "published"         │
                                          └──────────────────────┘

Processing time: ~5-15 seconds for full pipeline
Client polls or receives push notification when processing complete
```

**Content Moderation Detail:**

```
┌─────────────────────────────────────────────────────────┐
│               Content Moderation Pipeline                │
│                                                          │
│  1. Automated ML Screening (< 1 second)                  │
│     ├── NSFW detection (nudity classifier)               │
│     ├── Violence detection                               │
│     ├── Hate symbol detection                            │
│     └── Spam/scam detection                              │
│                                                          │
│  2. Decision:                                            │
│     ├── Score < 0.3: AUTO APPROVE                        │
│     ├── Score 0.3-0.8: QUEUE FOR HUMAN REVIEW            │
│     └── Score > 0.8: AUTO REJECT + notify user           │
│                                                          │
│  3. Human Review Queue                                   │
│     ├── Priority based on reach (followers count)        │
│     ├── SLA: review within 1 hour                        │
│     └── Appeals process for rejected content             │
└─────────────────────────────────────────────────────────┘
```

### 7.3 Stories System (Ephemeral Content)

```
Stories Architecture:

┌────────────────────────────────────────────────────────────────┐
│                     Stories Service                             │
│                                                                │
│  Upload Flow:                                                  │
│  1. User uploads story media -> S3                             │
│  2. Process media (resize, compress)                           │
│  3. Store metadata with expires_at = now() + 24h               │
│  4. Push story_id to Redis: stories:{user_id} sorted set      │
│  5. Fan-out notification to close friends (if applicable)      │
│                                                                │
│  Read Flow:                                                    │
│  1. Fetch list of followed users                               │
│  2. For each, check stories:{user_id} in Redis                │
│  3. Filter out expired stories (score < now - 24h)             │
│  4. Sort story trays by recency and unseen status              │
│  5. Return story trays with pre-signed S3 URLs                │
│                                                                │
│  Cleanup:                                                      │
│  - Cron job runs every hour                                    │
│  - DELETE FROM stories WHERE expires_at < NOW()                │
│  - Remove expired entries from Redis sorted sets               │
│  - Delete media from S3 (or move to cold storage for          │
│    compliance/legal hold)                                      │
└────────────────────────────────────────────────────────────────┘

Stories View Tracking:
┌─────────────────┬──────────────┬──────────────────────────────┐
│ Column          │ Type         │ Notes                        │
├─────────────────┼──────────────┼──────────────────────────────┤
│ story_id        │ BIGINT       │ Partition key                │
│ viewer_user_id  │ BIGINT       │ Clustering key               │
│ viewed_at       │ TIMESTAMP    │ When viewed                  │
└─────────────────┴──────────────┴──────────────────────────────┘

Redis structure for story tray ordering:
  Key: story_tray:{user_id}
  Type: Sorted Set
  Score: latest_story_timestamp of each followed user
  Value: followed_user_id

  This enables fast "which story trays to show first" ordering.
```

---

## 8. Scaling Discussion

### Photo Storage Scaling

```
                    ┌──────────────┐
                    │   API Layer  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼────┐ ┌────▼─────┐ ┌────▼─────┐
        │  S3      │ │  S3      │ │  S3      │
        │ US-East  │ │ EU-West  │ │ AP-South │
        └─────┬────┘ └────┬─────┘ └────┬─────┘
              │            │            │
        ┌─────▼────────────▼────────────▼─────┐
        │          CDN Edge Locations          │
        │   (200+ PoPs worldwide)              │
        │                                      │
        │   Cache hit rate target: 95%+        │
        │   TTL: 30 days for photos            │
        │   TTL: 24 hours for stories          │
        └──────────────────────────────────────┘

S3 cost optimization:
- Original photos: S3 Standard (frequently accessed)
- Old photos (> 1 year): S3 Infrequent Access (30% cheaper)
- Thumbnails: S3 Standard (always accessed via CDN)
```

### Database Sharding Strategy

```
Posts DB Sharding (by user_id):
- Shard key: user_id % num_shards
- Enables efficient "get all posts by user" queries
- Fan-out required for feed generation (cross-shard)

Shard 0: user_id % 1024 == 0   -> posts for users 0, 1024, 2048...
Shard 1: user_id % 1024 == 1   -> posts for users 1, 1025, 2049...
...
Shard 1023: user_id % 1024 == 1023

Initial: 1024 logical shards across 64 physical DB servers
Each server: 16 logical shards, ~90M posts, ~45 GB data
```

### Feed Cache Scaling

```
Redis Cluster for Feed Cache:
- 500M DAU * 200 posts per feed * 8 bytes per post_id = 800 GB
- Add overhead: ~1.2 TB total
- Redis cluster: 40 nodes * 32 GB each = 1.28 TB capacity
- Sharded by user_id
- LRU eviction for inactive users (regenerate on next login)
```

### Celebrity Handling at Scale

```
Problem: Selena Gomez has 400M followers.
Fan-out on write = 400M Redis writes per post = minutes of delay.

Solution: Hybrid approach
1. Celebrity posts stored in a separate "celebrity posts" cache
2. Each user's feed = pre-computed feed + live merge of celebrity posts
3. Celebrity threshold: configurable (start at 10K, tune based on metrics)

Cache structure:
  celebrity_posts:{celebrity_user_id} = sorted set of recent post_ids
  user_celebrities:{user_id} = set of celebrity user_ids this user follows

Feed assembly:
  regular_feed = ZREVRANGE feed:{user_id} 0 49
  for each celeb_id in user_celebrities:{user_id}:
      celeb_posts = ZREVRANGE celebrity_posts:{celeb_id} 0 4
      merge into regular_feed
  sort by ranking score
  return top 20
```

---

## 9. Trade-offs

### Fan-Out Strategy

```
┌──────────────────┬───────────────────┬───────────────────────┐
│ Aspect           │ Fan-Out on Write  │ Fan-Out on Read       │
├──────────────────┼───────────────────┼───────────────────────┤
│ Feed read speed  │ Very fast O(1)    │ Slow O(following_cnt) │
│ Post publish     │ Slow (fan-out)    │ Instant               │
│ Celebrity posts  │ Very expensive    │ Handled naturally     │
│ Inactive users   │ Wasted writes     │ No wasted work        │
│ Feed freshness   │ Slightly delayed  │ Always fresh          │
│ Storage cost     │ High (N copies)   │ Low (single copy)     │
│ System complex.  │ Moderate          │ Lower                 │
├──────────────────┼───────────────────┼───────────────────────┤
│ Best for         │ Read-heavy, most  │ Write-heavy, few      │
│                  │ users have < 10K  │ followers queried     │
│                  │ followers         │                       │
└──────────────────┴───────────────────┴───────────────────────┘
Decision: Hybrid (fan-out on write for regular, pull for celebrities)
```

### Consistency vs Freshness

```
┌─────────────────────┬─────────────────────────────────────┐
│ Component           │ Consistency Choice                  │
├─────────────────────┼─────────────────────────────────────┤
│ Feed                │ Eventual (seconds delay OK)         │
│ Like counts         │ Eventual (approximate counts OK)    │
│ Follower counts     │ Eventual (approximate counts OK)    │
│ Post content        │ Strong (user sees own post          │
│                     │ immediately after upload)           │
│ Follow relationship │ Strong (unfollow must take effect   │
│                     │ immediately for privacy)            │
│ Stories             │ Eventual (minor delay OK)           │
└─────────────────────┴─────────────────────────────────────┘
```

### Storage Format

```
┌──────────────┬───────────────────────┬──────────────────────┐
│ Aspect       │ Store originals only  │ Pre-generate thumbs  │
├──────────────┼───────────────────────┼──────────────────────┤
│ Storage cost │ Lower (1 copy)        │ Higher (6x copies)   │
│ Read latency │ Higher (resize on     │ Lower (serve direct) │
│              │ the fly)              │                      │
│ CPU cost     │ Higher (on-demand     │ Lower (one-time      │
│              │ processing)           │ processing)          │
│ CDN caching  │ Complex (vary by size)│ Simple (fixed URLs)  │
├──────────────┼───────────────────────┼──────────────────────┤
│ Decision     │ Pre-generate standard sizes. Storage is      │
│              │ cheap; latency matters more for UX.          │
└──────────────┴──────────────────────────────────────────────┘
```

---

## 10. Failure Scenarios & Handling

### Scenario 1: Photo Upload Fails Mid-Upload

```
Problem: User uploads 2 MB photo, connection drops at 80%.

Solution: Resumable uploads
1. Client requests upload URL with Content-Range support
2. Server returns upload_id and pre-signed S3 URL
3. Client uploads in chunks (256 KB each)
4. On failure, client resumes from last acknowledged chunk
5. Server assembles chunks after all received

POST /api/v1/uploads/init -> { upload_id, upload_url }
PUT  /api/v1/uploads/{id}/chunk?offset=0 -> { next_offset }
POST /api/v1/uploads/{id}/complete -> { post_id }
```

### Scenario 2: Feed Cache Redis Node Failure

```
Impact: Feed unavailable for subset of users (sharded by user_id).

Handling:
1. Redis Cluster promotes replica to primary (< 30 seconds)
2. During failover, fall back to fan-out-on-read from Posts DB
3. Rebuild cache for affected users on next feed request
4. Background job pre-populates cache for active users

Mitigation:
- Redis Cluster with 1 primary + 2 replicas per shard
- Cross-AZ replica placement
- Sentinel for automatic failover
```

### Scenario 3: Image Processing Pipeline Backup

```
Problem: Spike in uploads causes processing queue to grow.

Handling:
1. Auto-scale processing workers based on queue depth
2. Prioritize: small images first (faster throughput)
3. Show "processing" status to user with preview of original
4. Dead letter queue for repeatedly failing images
5. Alert ops team if queue age > 5 minutes

Backpressure:
- If queue > 1M items, temporarily disable non-essential processing
  (e.g., skip extra thumbnail sizes, defer filter application)
- Essential processing (original storage, basic thumbnail) always runs
```

### Scenario 4: Celebrity Posts Causing Feed Storms

```
Problem: Celebrity with 100M followers posts, causing massive fan-out
         that overwhelms the system.

Handling:
1. Detect celebrity status (followers > threshold)
2. Skip fan-out for celebrity posts entirely
3. Celebrity posts fetched at read time and merged into feed
4. Cache celebrity recent posts in dedicated high-performance cache
5. Rate-limit celebrity post frequency if needed during incidents
```

---

## 11. Monitoring & Alerting

### Key Metrics

```
┌─────────────────────────────┬────────────────────────────────┐
│ Metric                      │ Alert Threshold                │
├─────────────────────────────┼────────────────────────────────┤
│ Feed load latency p99       │ > 200ms (warn), > 500ms (crit)│
│ Photo upload success rate   │ < 99.5% (warn), < 99% (crit)  │
│ Image processing queue age  │ > 2min (warn), > 5min (crit)  │
│ CDN cache hit rate          │ < 90% (warn), < 80% (crit)    │
│ Fan-out completion time     │ > 5s (warn), > 30s (crit)     │
│ Feed cache miss rate        │ > 10% (warn), > 20% (crit)    │
│ DB replication lag          │ > 1s (warn), > 5s (crit)      │
│ S3 upload error rate        │ > 0.5% (warn), > 1% (crit)    │
│ Content moderation backlog  │ > 10K items (warn)             │
│ Stories cleanup lag         │ > 1h behind (warn)             │
│ API error rate (5xx)        │ > 0.1% (warn), > 1% (crit)    │
│ Active WebSocket connections│ Drop > 10% in 5 min (crit)    │
└─────────────────────────────┴────────────────────────────────┘
```

### Business Metrics Dashboard

```
- DAU / MAU ratio (stickiness)
- Average time spent in feed per session
- Photo upload count per hour (trend)
- Stories creation and view rates
- Like/comment engagement rates
- Feed scroll depth distribution
- Explore page click-through rate
```

---

## 12. Interview Variations

### Common Follow-Up Questions

**Q1: "How would you rank the news feed instead of showing chronological order?"**
```
A: ML-based ranking system:
Features:
- Post recency (time decay)
- User's past engagement with author (likes, comments, DM)
- Post engagement rate (likes/impressions ratio)
- Content type affinity (user prefers photos vs carousels)
- Relationship strength (close friends, family detection)

Architecture:
- Feature store (precomputed features per user-pair)
- Ranking model (gradient boosted trees or neural network)
- Candidate generation -> Ranking -> Re-ranking pipeline
- A/B testing framework to compare ranking algorithms
```

**Q2: "How would you implement the Explore page?"**
```
A: Two-stage recommendation:
1. Candidate Generation:
   - Collaborative filtering (users similar to you liked these)
   - Content-based (posts similar to ones you engaged with)
   - Trending posts (high engagement velocity)
   - Location-based (popular posts near you)

2. Ranking:
   - Score each candidate by predicted engagement probability
   - Diversity filter (avoid too many similar posts)
   - Safety filter (content moderation score)
   - Cache personalized Explore feed, refresh every 30 min
```

**Q3: "How would you handle user privacy (private accounts)?"**
```
A:
- Private account posts excluded from fan-out
- Follow requests require approval for private accounts
- Feed generation checks follow approval status
- Explore page never shows private account content
- Access control check on every post read API call
- Cache invalidation when account switches private/public
```

**Q4: "How would you implement hashtag search?"**
```
A:
- Parse hashtags from captions during post creation
- Store in hashtag_posts table (hashtag -> post_id, score)
- Elasticsearch index for hashtag search with autocomplete
- Trending hashtags: sliding window count of posts per hashtag
  in last 1h / 24h, rank by velocity (growth rate)
- Hashtag page: query hashtag_posts sorted by recency or
  engagement score
```

**Q5: "What if you need to support video uploads (Reels)?"**
```
A: Additional components needed:
- Video transcoding pipeline (multiple resolutions, codecs)
- Adaptive bitrate streaming (HLS/DASH)
- Video CDN with chunked delivery
- Longer processing time (30s-5min for video)
- Higher storage costs (~10x per post vs photo)
- Video-specific content moderation (frame sampling)
- Separate feed mixing strategy (interleave photos and videos)
```

**Q6: "How do you prevent spam and fake accounts?"**
```
A: Multi-layer defense:
1. Registration: CAPTCHA, phone verification, email verification
2. Behavior analysis: posting frequency, follow/unfollow patterns
3. Content analysis: duplicate photo detection (perceptual hashing)
4. Network analysis: graph-based fake account detection
5. Rate limiting: max follows/day, max posts/hour
6. User reports: community flagging with human review
```

---

## Summary: Key Design Decisions

```
┌─────────────────────┬──────────────────────────────────────┐
│ Component           │ Decision                             │
├─────────────────────┼──────────────────────────────────────┤
│ Photo storage       │ S3 + CDN (multi-region)              │
│ Feed generation     │ Hybrid fan-out (write + read)        │
│ Feed cache          │ Redis sorted sets                    │
│ Social graph        │ Cassandra (high write throughput)    │
│ Posts metadata      │ PostgreSQL sharded by user_id        │
│ Image processing    │ Async pipeline via Kafka workers     │
│ Content moderation  │ ML auto-screen + human review queue  │
│ Stories             │ Redis + S3 with 24h TTL cleanup      │
│ Celebrity handling  │ Pull-based merge at read time        │
│ Search              │ Elasticsearch for users, hashtags    │
│ Real-time updates   │ WebSocket for live notifications     │
│ Ranking             │ ML-based with A/B testing            │
└─────────────────────┴──────────────────────────────────────┘
```
