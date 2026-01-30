# Design YouTube (Video Streaming Platform)

## 1. Problem Statement

Design a scalable video sharing and streaming platform similar to YouTube that allows
users to upload, transcode, store, and stream video content to millions of concurrent
viewers. The system must support adaptive bitrate streaming, a recommendation engine,
real-time view counting, comments, search, and live streaming capabilities.

---

## 2. Requirements

### Functional Requirements
- Users can upload videos (up to 256 GB per file)
- System transcodes videos into multiple resolutions and formats (HLS/DASH)
- Users can stream videos with adaptive bitrate
- Video recommendation engine suggests relevant content
- Real-time view counting at scale
- Comment system with threading and moderation
- Search and discovery (title, tags, descriptions, captions)
- Live streaming support
- Channel subscriptions and notification feed
- Like/dislike, playlists, watch history

### Non-Functional Requirements
- High availability: 99.99% uptime
- Low latency: video playback starts within 2 seconds
- Consistency: eventual consistency acceptable for views/likes; strong for uploads
- Durability: zero data loss for uploaded videos
- Scalability: support 2 billion monthly active users
- Global reach: CDN for worldwide low-latency streaming

---

## 3. Capacity Estimation

### User and Traffic Assumptions
```
Total users:              2 billion MAU
Daily active users (DAU): 800 million
Average watch time:       40 minutes/day per DAU
Average video length:     7 minutes
Videos watched per day:   800M * (40/7) ~ 4.5 billion video plays/day
                          ~ 52,000 video plays/second (average)
                          ~ 150,000 video plays/second (peak)

Video uploads per day:    500,000 new videos/day
                          ~ 6 uploads/second (average)
```

### Storage Estimation
```
Average raw upload size:          500 MB
Transcoded output (all formats):  ~3x raw = 1.5 GB per video
Daily new storage:                500,000 * 1.5 GB = 750 TB/day
Annual new storage:               750 TB * 365 = ~274 PB/year
Total stored videos:              ~800 million videos
Total storage (estimated):        ~1.5 exabytes (with replication)
```

### Bandwidth Estimation
```
Average video bitrate (streaming): 5 Mbps (1080p average)
Concurrent viewers (peak):         100 million
Peak egress bandwidth:             100M * 5 Mbps = 500 Tbps
Daily egress:                      4.5B plays * 7 min * 5 Mbps
                                   = 4.5B * 420s * 5 Mbps
                                   = ~1.2 exabytes/day
```

### Database Estimation
```
Video metadata record:   ~2 KB each
800 million videos:      800M * 2 KB = 1.6 TB
User records:            2B * 1 KB = 2 TB
Comments:                ~200 billion total * 0.5 KB = 100 TB
View counts:             800M entries * 16 bytes = ~13 GB (in-memory)
```

---

## 4. High-Level Design

```
                                    +------------------+
                                    |   DNS / Global   |
                                    |   Load Balancer  |
                                    +--------+---------+
                                             |
                    +------------------------+------------------------+
                    |                        |                        |
            +-------+-------+      +---------+--------+     +--------+--------+
            |  Upload API   |      |  Streaming API   |     |  General API    |
            |  Service      |      |  Service         |     |  (metadata,     |
            |               |      |                  |     |   comments,     |
            +-------+-------+      +---------+--------+     |   search)       |
                    |                        |              +--------+--------+
                    v                        v                       |
          +---------+---------+    +---------+---------+            |
          |  Upload Queue     |    |   CDN Layer       |            v
          |  (Kafka)          |    |  (CloudFront /    |   +--------+--------+
          +---------+---------+    |   Akamai)         |   | API Gateway     |
                    |              |                   |   | + Rate Limiter  |
                    v              +---+------+--------+   +--------+--------+
          +---------+---------+        |      |                     |
          |  Transcoding      |        |      |            +--------+--------+
          |  Pipeline         |    +---+--+ +-+----+       |                 |
          |  (FFmpeg workers) |    | Edge | | Edge |       v                 v
          +---------+---------+    | PoP  | | PoP  |  +---+----+     +------+----+
                    |              +------+ +------+  |Metadata |     | Search    |
                    v                                 |Service  |     | Service   |
          +---------+---------+                       +---+----+     | (Elastic) |
          |  Object Storage   |                           |          +-----------+
          |  (S3 / GCS)       |                           v
          |                   |                     +-----+------+
          |  Hot:  SSD tier   |                     | PostgreSQL |
          |  Warm: HDD tier   |                     | (sharded)  |
          |  Cold: Glacier    |                     +-----+------+
          +-------------------+                           |
                                                          v
                                               +----------+---------+
                                               | Redis Cache        |
                                               | (view counts,      |
                                               |  hot metadata)     |
                                               +--------------------+

  +---------------------+     +---------------------+     +---------------------+
  | Recommendation      |     | Notification        |     | Analytics           |
  | Engine (ML)         |     | Service             |     | Pipeline            |
  | - Collaborative     |     | - Push / Email      |     | - Spark / Flink     |
  |   filtering         |     | - Fan-out service   |     | - View aggregation  |
  | - Content-based     |     +---------------------+     +---------------------+
  +---------------------+
```

---

## 5. API Design

### Video Upload

```
POST /api/v1/videos/upload/init
Headers: Authorization: Bearer <token>
Body: {
    "title": "My Video",
    "description": "A great video",
    "tags": ["tech", "tutorial"],
    "privacy": "public",            // public | unlisted | private
    "file_size": 524288000,         // 500 MB in bytes
    "file_type": "video/mp4",
    "chunk_size": 10485760          // 10 MB chunks
}
Response 200: {
    "upload_id": "up_abc123",
    "upload_url": "https://upload.yt.com/v1/up_abc123",
    "chunk_count": 50,
    "expires_at": "2025-01-16T12:00:00Z"
}

PUT /api/v1/videos/upload/{upload_id}/chunks/{chunk_number}
Headers: Content-Type: application/octet-stream
         Content-MD5: <checksum>
Body: <binary chunk data>
Response 200: {
    "chunk_number": 1,
    "status": "received",
    "bytes_received": 10485760
}

POST /api/v1/videos/upload/{upload_id}/complete
Response 200: {
    "video_id": "vid_xyz789",
    "status": "processing",
    "estimated_time_seconds": 300
}
```

### Video Streaming

```
GET /api/v1/videos/{video_id}/manifest.m3u8
Response 200: (HLS Master Playlist)
    #EXTM3U
    #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
    /stream/vid_xyz789/360p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=2400000,RESOLUTION=1280x720
    /stream/vid_xyz789/720p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
    /stream/vid_xyz789/1080p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=12000000,RESOLUTION=3840x2160
    /stream/vid_xyz789/4k/playlist.m3u8
```

### Search

```
GET /api/v1/search?q=system+design&type=video&sort=relevance&page=1&limit=20
Response 200: {
    "results": [
        {
            "video_id": "vid_abc",
            "title": "System Design Interview Guide",
            "channel": { "id": "ch_1", "name": "TechChannel" },
            "thumbnail_url": "https://cdn.yt.com/thumb/vid_abc.jpg",
            "duration_seconds": 1200,
            "view_count": 1500000,
            "published_at": "2025-01-10T08:00:00Z",
            "relevance_score": 0.95
        }
    ],
    "total_results": 12400,
    "next_page_token": "eyJwIjoyfQ=="
}
```

### Comments

```
POST /api/v1/videos/{video_id}/comments
Body: {
    "text": "Great video!",
    "parent_comment_id": null       // null for top-level, ID for reply
}
Response 201: {
    "comment_id": "cmt_456",
    "user_id": "usr_789",
    "text": "Great video!",
    "created_at": "2025-01-15T10:30:00Z",
    "like_count": 0
}

GET /api/v1/videos/{video_id}/comments?sort=top&page=1&limit=20
Response 200: {
    "comments": [...],
    "total_count": 5400,
    "next_page_token": "..."
}
```

### Recommendations

```
GET /api/v1/recommendations?video_id=vid_abc&limit=20
Response 200: {
    "recommendations": [
        {
            "video_id": "vid_def",
            "title": "Advanced System Design",
            "score": 0.92,
            "reason": "similar_content"
        }
    ]
}
```

---

## 6. Database Schema

### Videos Table (PostgreSQL, sharded by video_id)
```sql
CREATE TABLE videos (
    video_id        BIGINT PRIMARY KEY,         -- Snowflake ID
    user_id         BIGINT NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    privacy         ENUM('public','unlisted','private') DEFAULT 'public',
    status          ENUM('uploading','processing','ready','failed','removed'),
    duration_ms     INT,
    original_url    VARCHAR(2048),              -- S3 path to original
    thumbnail_url   VARCHAR(2048),
    view_count      BIGINT DEFAULT 0,
    like_count      INT DEFAULT 0,
    dislike_count   INT DEFAULT 0,
    comment_count   INT DEFAULT 0,
    file_size_bytes BIGINT,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    published_at    TIMESTAMP,

    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_published (published_at),
    INDEX idx_created (created_at)
);
```

### Video Formats Table
```sql
CREATE TABLE video_formats (
    format_id       BIGINT PRIMARY KEY,
    video_id        BIGINT NOT NULL,
    resolution      VARCHAR(10),                -- '360p', '720p', '1080p', '4k'
    codec           VARCHAR(20),                -- 'h264', 'h265', 'vp9', 'av1'
    container       VARCHAR(10),                -- 'mp4', 'webm', 'ts'
    bitrate_kbps    INT,
    file_size_bytes BIGINT,
    storage_url     VARCHAR(2048),              -- S3/GCS path
    status          ENUM('pending','encoding','ready','failed'),
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_video_id (video_id),
    INDEX idx_video_resolution (video_id, resolution)
);
```

### Users Table
```sql
CREATE TABLE users (
    user_id         BIGINT PRIMARY KEY,
    username        VARCHAR(50) UNIQUE NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    display_name    VARCHAR(100),
    avatar_url      VARCHAR(2048),
    subscriber_count BIGINT DEFAULT 0,
    total_views     BIGINT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_username (username),
    INDEX idx_email (email)
);
```

### Comments Table (sharded by video_id)
```sql
CREATE TABLE comments (
    comment_id      BIGINT PRIMARY KEY,
    video_id        BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    parent_id       BIGINT,                     -- NULL for top-level
    text            TEXT NOT NULL,
    like_count      INT DEFAULT 0,
    status          ENUM('active','deleted','flagged') DEFAULT 'active',
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_video_comments (video_id, created_at),
    INDEX idx_video_top (video_id, like_count DESC),
    INDEX idx_parent (parent_id),
    INDEX idx_user (user_id)
);
```

### Subscriptions Table
```sql
CREATE TABLE subscriptions (
    user_id         BIGINT NOT NULL,
    channel_id      BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (user_id, channel_id),
    INDEX idx_channel (channel_id)
);
```

### View Events (Kafka -> Analytics DB / Cassandra)
```sql
CREATE TABLE view_events (
    event_id        UUID,
    video_id        BIGINT,
    user_id         BIGINT,                     -- nullable for anonymous
    session_id      VARCHAR(64),
    watch_duration  INT,                        -- seconds
    device_type     VARCHAR(20),
    country         VARCHAR(5),
    timestamp       TIMESTAMP,

    PRIMARY KEY (video_id, timestamp)           -- Cassandra partitioning
);
```

---

## 7. Deep Dive

### Deep Dive 1: Video Upload and Transcoding Pipeline

The video upload and transcoding pipeline is the backbone of the platform.

```
  Client                    Upload Service           Message Queue          Transcoding Workers
    |                            |                        |                        |
    |---(1) Init Upload--------->|                        |                        |
    |<------ Upload URL----------|                        |                        |
    |                            |                        |                        |
    |---(2) Upload Chunk 1------>|                        |                        |
    |<------ ACK + checksum------|                        |                        |
    |---(3) Upload Chunk 2------>|                        |                        |
    |<------ ACK + checksum------|                        |                        |
    |         ...                |                        |                        |
    |---(N) Upload Chunk N------>|                        |                        |
    |<------ ACK + checksum------|                        |                        |
    |                            |                        |                        |
    |---(4) Complete Upload----->|                        |                        |
    |                            |---(5) Publish Job----->|                        |
    |                            |                        |---(6) Dequeue--------->|
    |                            |                        |                        |
    |                            |                        |            +-----------+-----------+
    |                            |                        |            |  Merge chunks         |
    |                            |                        |            |  Validate integrity   |
    |                            |                        |            |  Extract metadata     |
    |                            |                        |            |  Generate thumbnails  |
    |                            |                        |            +-----------+-----------+
    |                            |                        |                        |
    |                            |                        |            +-----------+-----------+
    |                            |                        |            |  Transcode to:        |
    |                            |                        |            |    - 240p  (400kbps)  |
    |                            |                        |            |    - 360p  (800kbps)  |
    |                            |                        |            |    - 480p  (1.5Mbps)  |
    |                            |                        |            |    - 720p  (2.5Mbps)  |
    |                            |                        |            |    - 1080p (5Mbps)    |
    |                            |                        |            |    - 4K    (12Mbps)   |
    |                            |                        |            +-----------+-----------+
    |                            |                        |                        |
    |                            |                        |            +-----------+-----------+
    |                            |                        |            |  Generate HLS/DASH    |
    |                            |                        |            |  manifests and        |
    |                            |                        |            |  segment files        |
    |                            |                        |            |  (2-10 second chunks) |
    |                            |                        |            +-----------+-----------+
    |                            |                        |                        |
    |                            |                        |            +-----------+-----------+
    |                            |                        |            |  Upload to Object     |
    |                            |                        |            |  Storage + CDN origin |
    |                            |                        |            |  Update DB status     |
    |                            |                        |            +-----------+-----------+
    |<------(7) Webhook/Push: Video Ready---------------------------------|
```

**Chunked Upload Protocol:**
- Client splits file into 10 MB chunks
- Each chunk includes MD5 checksum for integrity
- Server tracks received chunks in Redis (bitmap: upload_id -> chunk_bitmap)
- If upload fails midway, client queries for missing chunks and resumes
- Chunks stored temporarily in fast SSD-backed storage
- Upload session expires after 24 hours if not completed

**Transcoding Architecture:**
- FFmpeg-based workers running on GPU-accelerated instances
- Each video spawns multiple parallel transcoding jobs (one per resolution)
- Jobs submitted via Kafka with priority queues (premium users get priority)
- DAG-based pipeline: merge -> validate -> extract audio -> transcode -> segment -> upload
- Adaptive bitrate encoding: constant quality (CRF) mode for efficiency
- Two-pass encoding for higher quality at lower bitrates
- Hardware acceleration via NVENC (NVIDIA) or QSV (Intel) for throughput

**Segment Generation (HLS):**
- Each transcoded resolution is split into 4-second segments (.ts files)
- Master playlist (.m3u8) references all available quality levels
- Media playlists reference individual segments within each quality level
- Segments are independently seekable — enables instant quality switching
- Key frames aligned across all resolutions for seamless switching

### Deep Dive 2: Content Delivery and Adaptive Bitrate Streaming

```
                              +---------------------------+
                              |    Origin Server (S3)     |
                              |    All video segments     |
                              +-------------+-------------+
                                            |
                              +-------------+-------------+
                              |    Origin Shield (Cache)  |
                              |    Reduces origin load    |
                              +---+---------+---------+---+
                                  |         |         |
                     +------------+    +----+----+    +------------+
                     |                 |         |                 |
               +-----+------+   +-----+------+  +-----+------+   +-----+------+
               |  CDN PoP   |   |  CDN PoP   |  |  CDN PoP   |   |  CDN PoP   |
               |  US-East   |   |  EU-West   |  |  Asia-East |   |  SA-South  |
               +-----+------+   +-----+------+  +-----+------+   +-----+------+
                     |                 |               |                 |
                  Viewers           Viewers          Viewers          Viewers
```

**CDN Strategy:**
- Multi-CDN approach: use Akamai, CloudFront, and Fastly simultaneously
- DNS-based routing directs users to nearest CDN PoP
- Real-time quality monitoring switches CDN if one degrades
- Hot videos (< 48 hours, high views) cached aggressively at edge
- Long-tail videos fetched from origin on demand, cached if popular

**Adaptive Bitrate (ABR) Streaming:**
```
Client ABR Algorithm:
  1. Start with lowest quality (fast startup)
  2. Measure download throughput for each segment
  3. Estimate available bandwidth using EWMA:
     bandwidth_estimate = 0.7 * current_measurement + 0.3 * previous_estimate
  4. Select highest quality where bitrate < 0.8 * bandwidth_estimate
  5. If buffer drops below 5 seconds, immediately downgrade
  6. If buffer exceeds 30 seconds, attempt upgrade
  7. Track quality switches — avoid oscillation with hysteresis
```

**Storage Tiering:**
```
+------------------+---------------------------+------------------+
| Tier             | Criteria                  | Storage Type     |
+------------------+---------------------------+------------------+
| Hot              | < 7 days old OR           | SSD-backed S3    |
|                  | > 1000 views/day          | Standard         |
+------------------+---------------------------+------------------+
| Warm             | 7-90 days old AND         | HDD-backed S3    |
|                  | 10-1000 views/day         | Infrequent Access|
+------------------+---------------------------+------------------+
| Cold             | > 90 days old AND         | S3 Glacier       |
|                  | < 10 views/day            | (retrieval ~min) |
+------------------+---------------------------+------------------+
| Archive          | > 1 year, < 1 view/month  | Glacier Deep     |
|                  |                           | (retrieval ~hrs) |
+------------------+---------------------------+------------------+
```

### Deep Dive 3: View Counting at Scale and Recommendation Engine

**Approximate View Counting:**

Exact counting at 52,000 views/second is impractical with strong consistency.
We use a multi-stage approximate counting pipeline:

```
  Client View Event
        |
        v
  +-----+------+      +-----------+      +-------------+      +-----------+
  | Edge Server |----->| Kafka     |----->| Flink       |----->| Cassandra |
  | (dedup by  |      | Topic:    |      | Stream      |      | (durable  |
  |  session+  |      | "views"   |      | Processor   |      |  counts)  |
  |  video in  |      |           |      |             |      |           |
  |  local     |      +-----------+      | - Window:   |      +-----------+
  |  bloom     |                         |   1 min     |            |
  |  filter)   |                         | - Aggregate |            v
  +------------+                         |   per video |      +-----------+
                                         | - HLL for  |      | Redis     |
                                         |   unique   |      | (real-time|
                                         |   viewers  |      |  counter) |
                                         +-------------+      +-----------+
```

**Deduplication Strategy:**
- Edge server maintains local Bloom filter (per-minute window) to reject obvious duplicates
- Kafka partitioned by video_id ensures all events for same video go to same Flink instance
- Flink maintains per-video HyperLogLog for unique viewer approximation
- Count incremented only if: session > 30 seconds watch time AND not in recent dedup set
- Redis holds real-time approximate count (updated every 5 seconds)
- Cassandra stores durable hourly/daily aggregates
- Public-facing count updated every ~30 seconds (not real-time for anti-fraud)

**Recommendation Engine:**

```
+------------------+     +-------------------+     +------------------+
| Offline Pipeline |     | Nearline Pipeline |     | Online Serving   |
| (daily batch)    |     | (hourly updates)  |     | (real-time)      |
+------------------+     +-------------------+     +------------------+
|                  |     |                   |     |                  |
| Collaborative    |     | User embedding    |     | Candidate        |
| Filtering:       |     | updates from      |     | Generation:      |
| - User-User      |     | recent watch      |     | - Retrieve top   |
|   similarity     |     | history           |     |   500 candidates |
| - Item-Item      |     |                   |     |   from each      |
|   similarity     |     | Trending topics   |     |   source         |
|                  |     | and viral content |     |                  |
| Content-Based:   |     | detection         |     | Ranking:         |
| - Video          |     |                   |     | - ML model       |
|   embeddings     |     |                   |     |   scores each    |
|   (title, tags,  |     |                   |     |   candidate      |
|   transcript)    |     |                   |     | - Features:      |
|                  |     |                   |     |   watch history, |
| Matrix           |     |                   |     |   user profile,  |
| Factorization:   |     |                   |     |   video features,|
| - ALS on         |     |                   |     |   context (time, |
|   user-video     |     |                   |     |   device)        |
|   interaction    |     |                   |     |                  |
|   matrix         |     |                   |     | Diversity:       |
|                  |     |                   |     | - MMR to avoid   |
+------------------+     +-------------------+     |   repetition     |
                                                   +------------------+
```

**Two-Stage Recommendation Architecture:**

Stage 1 - Candidate Generation (fast, recall-focused):
- Collaborative filtering: find videos watched by similar users
- Content-based: find videos with similar embeddings to recently watched
- Subscription feed: recent uploads from subscribed channels
- Trending: globally and regionally popular content
- Result: ~2000 candidate videos

Stage 2 - Ranking (slower, precision-focused):
- Deep neural network scores each candidate
- Features: user history, video metadata, engagement signals, freshness
- Calibrated to predict P(click), P(watch>50%), P(like), P(subscribe)
- Multi-objective optimization balancing engagement and satisfaction
- Result: top 20 ranked recommendations

---

## 8. Scaling Discussion

### Video Storage Scaling
- Object storage (S3/GCS) scales horizontally — no practical limit
- Separate hot and cold storage pools to optimize cost
- Erasure coding (e.g., Reed-Solomon 6+3) instead of 3x replication saves 50% storage
- Geo-replicate hot content to multiple regions for faster CDN origin pulls

### Transcoding Scaling
- Use spot/preemptible GPU instances (70% cost savings)
- Auto-scale transcoding fleet based on queue depth
- Priority queues: live streams > premium users > free users
- Distribute transcoding per resolution across different workers (parallelize)

### Database Scaling
- Shard video metadata by video_id (hash-based)
- Shard comments by video_id (co-locate with video for efficient queries)
- Read replicas for search and browse queries
- Separate OLTP (PostgreSQL) from OLAP (BigQuery/Redshift)

### CDN Scaling
- Multi-CDN with real-time failover reduces dependency on single provider
- Pre-warm CDN caches for anticipated viral content
- Use multicast or P2P protocols for live events with massive concurrent viewership
- Edge computing for personalized thumbnail selection

---

## 9. Trade-offs

| Decision | Option A | Option B | Choice & Rationale |
|----------|----------|----------|-------------------|
| Streaming protocol | HLS | DASH | Both: HLS for Apple, DASH for others. HLS has wider device support but DASH is more flexible. |
| View consistency | Strong | Eventual | Eventual: exact counts not needed in real-time; batch corrections every hour are acceptable. |
| Video codec | H.264 | VP9/AV1 | All three: H.264 for compatibility, VP9/AV1 for bandwidth savings on modern devices. Encode in all formats. |
| Segment duration | 2 seconds | 10 seconds | 4 seconds: balances latency (shorter = faster quality switching) vs overhead (longer = fewer requests). |
| Comment storage | SQL | NoSQL | SQL (PostgreSQL): comments need threading, sorting, and moderation queries. Shard by video_id. |
| Recommendation model | Collaborative | Content-based | Hybrid: collaborative filtering finds non-obvious connections; content-based handles cold start. |
| Thumbnail storage | Per resolution | Single + resize | Per resolution: pre-compute common sizes (120x90, 320x180, 480x360) to avoid on-the-fly resizing. |

---

## 10. Failure Scenarios & Handling

### Upload Failure Mid-Transfer
- **Detection:** Client timeout or HTTP error on chunk upload
- **Handling:** Client queries server for received chunks, resumes from last successful chunk
- **Prevention:** Server persists chunk receipt in Redis with 24-hour TTL

### Transcoding Worker Crash
- **Detection:** Heartbeat timeout (worker stops reporting to coordinator)
- **Handling:** Job remains in Kafka (not ACKed), another worker picks it up
- **Prevention:** Idempotent transcoding jobs; output keyed by (video_id, format, resolution)

### CDN Cache Miss Storm (Thundering Herd)
- **Detection:** Origin server sees spike in requests for same content
- **Handling:** Request coalescing at origin shield — only one request reaches storage
- **Prevention:** Proactive cache warming for trending videos; stale-while-revalidate headers

### Database Shard Failure
- **Detection:** Health check failure, connection timeouts
- **Handling:** Promote read replica to primary; redirect writes
- **Recovery:** Rebuild failed shard from replica + WAL replay
- **Prevention:** Multi-AZ deployment with synchronous replication to standby

### Live Stream Origin Failure
- **Detection:** Stream health monitoring detects gap in segments
- **Handling:** Failover to backup ingest server; client player reconnects automatically
- **Prevention:** Dual-ingest from streamer to two independent origin servers

### Recommendation Service Degradation
- **Detection:** P99 latency exceeds 200ms or error rate > 1%
- **Handling:** Fall back to pre-computed static recommendations (trending, popular)
- **Prevention:** Circuit breaker pattern; recommendation results cached per user for 5 minutes

---

## 11. Monitoring & Alerting

### Key Metrics

```
+---------------------------+------------------+-------------------+
| Metric                    | Target           | Alert Threshold   |
+---------------------------+------------------+-------------------+
| Video start time (P50)    | < 1 second       | > 2 seconds       |
| Video start time (P99)    | < 3 seconds      | > 5 seconds       |
| Rebuffer rate             | < 0.5%           | > 1%              |
| Upload success rate       | > 99.5%          | < 99%             |
| Transcoding time (P50)    | < 10 minutes     | > 30 minutes      |
| Search latency (P99)      | < 200 ms         | > 500 ms          |
| API error rate            | < 0.1%           | > 0.5%            |
| CDN cache hit rate        | > 95%            | < 90%             |
| View count lag            | < 1 minute       | > 5 minutes       |
| Comment post latency      | < 500 ms         | > 2 seconds       |
+---------------------------+------------------+-------------------+
```

### Dashboards
- **Streaming Quality:** rebuffer ratio, bitrate distribution, quality switches per session
- **Upload Pipeline:** queue depth, transcoding backlog, success/failure rates
- **Infrastructure:** CDN bandwidth, origin load, storage utilization, cache hit ratios
- **Business:** DAU, watch time, uploads per day, creator retention
- **Search:** query latency, zero-result rate, click-through rate

### Alerting Strategy
- Page-level alerts (PagerDuty): streaming failures, upload pipeline down, CDN outage
- Warning alerts (Slack): elevated latency, cache hit rate drop, queue depth growing
- Informational (dashboard): daily aggregates, trend analysis, capacity forecasts

---

## 12. Interview Variations

### Common Follow-up Questions

**Q: How would you handle a viral video that suddenly gets millions of views?**
A: Pre-warm CDN edges when view velocity exceeds threshold. Use request coalescing at
origin shield. Scale out CDN capacity dynamically. Rate-limit non-critical features
(comments, recommendations) to preserve streaming bandwidth.

**Q: How would you implement copyright detection?**
A: Content ID system — compute perceptual fingerprints (audio + video) during upload.
Compare against database of copyrighted material using locality-sensitive hashing.
Flag matches for review. Options: block, monetize (ads to copyright holder), or allow.

**Q: How would you design the live streaming component?**
A: RTMP ingest from streamer to edge server. Server segments into HLS/DASH chunks
(2-second segments for low latency). Push segments to CDN immediately. Use WebRTC
for ultra-low-latency (sub-second) for interactive streams. Separate live chat system
using WebSocket connections.

**Q: How do you handle different video aspect ratios and formats?**
A: Normalize during transcoding to standard aspect ratios with letterboxing/pillarboxing.
Support vertical video (9:16) natively for mobile. Store original aspect ratio in metadata.
Player adapts layout based on video dimensions and device screen.

**Q: How do you design the notification system for subscriptions?**
A: Fan-out-on-write for channels with < 10M subscribers (write to each subscriber's
notification inbox). Fan-out-on-read for mega-channels (> 10M subscribers) to avoid
write amplification. Hybrid approach with priority tiers based on subscriber engagement.

**Q: How would you implement video chapters and timestamps?**
A: Store chapter metadata as structured data (timestamp, title) in video record. Allow
creators to define chapters in description using timestamp format (0:00, 1:30, etc.).
Auto-generate chapters using ML-based scene detection on video content. Display as
segmented progress bar in player.

**Q: How do you ensure video quality during peak hours?**
A: Monitor bandwidth per region in real-time. Pre-position popular content at edge PoPs
before peak hours. Implement graceful degradation — reduce default quality from 1080p
to 720p during extreme load. Use traffic shaping to prioritize video segments over
thumbnails and non-critical API calls.
