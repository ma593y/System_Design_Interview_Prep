# Back-of-Envelope Calculations

## Overview

Back-of-envelope calculations are quick, approximate estimations used in system design interviews to determine whether a proposed design is feasible. The goal is not precision but rather to get within the right order of magnitude. This guide provides complete step-by-step examples for common systems like URL shortener, Twitter, and YouTube.

---

## The Estimation Framework

### Step-by-Step Process

```
┌─────────────────────────────────────────────────────────────────┐
│              BACK-OF-ENVELOPE ESTIMATION FRAMEWORK              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STEP 1: CLARIFY REQUIREMENTS                                    │
│  └── What are we estimating? Scale? Time period?                │
│                                                                  │
│  STEP 2: STATE ASSUMPTIONS                                       │
│  └── Users, usage patterns, data sizes                          │
│                                                                  │
│  STEP 3: CALCULATE TRAFFIC                                       │
│  └── QPS (read/write), peak multiplier                          │
│                                                                  │
│  STEP 4: CALCULATE STORAGE                                       │
│  └── Data size, retention, replication                          │
│                                                                  │
│  STEP 5: CALCULATE BANDWIDTH                                     │
│  └── Inbound/outbound, peak requirements                        │
│                                                                  │
│  STEP 6: ESTIMATE INFRASTRUCTURE                                 │
│  └── Servers, databases, caches needed                          │
│                                                                  │
│  STEP 7: SANITY CHECK                                            │
│  └── Do numbers make sense? Compare to known systems            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Essential Numbers to Know

```
Time conversions:
- Seconds per day: 86,400 ≈ 100,000 (use 10^5)
- Seconds per month: 2.6 million ≈ 2.5 x 10^6
- Seconds per year: 31.5 million ≈ 3 x 10^7

Data conversions:
- 1 character (ASCII): 1 byte
- 1 character (UTF-8 average): 2 bytes
- 1 million: 10^6
- 1 billion: 10^9
- 1 trillion: 10^12

Throughput shortcuts:
- 1 million/day ≈ 12/second
- 1 billion/day ≈ 12,000/second
- 100 million/day ≈ 1,200/second

Storage shortcuts:
- 1 billion x 1 KB = 1 TB
- 1 million x 1 MB = 1 TB
- 1 thousand x 1 GB = 1 TB
```

---

## Example 1: URL Shortener (like bit.ly)

### Requirements Gathering

```
Functional:
- Shorten long URLs to short URLs
- Redirect short URLs to original URLs
- Custom short URLs (optional)
- Analytics (optional)

Non-functional:
- High availability
- Low latency redirection
- Short URLs should be as short as possible
```

### Assumptions

```
Scale assumptions:
- 100 million new URLs per month
- Read:write ratio = 100:1
- URLs never expire (or 5-year retention)
- Average original URL length: 200 characters

Short URL format:
- Base62 encoding (a-z, A-Z, 0-9)
- 7 characters gives 62^7 = 3.5 trillion combinations
- More than enough for years of operation
```

### Traffic Estimation

```
WRITE TRAFFIC (Creating short URLs):
─────────────────────────────────────
URLs per month: 100 million
URLs per day: 100M / 30 = 3.33 million
URLs per second: 3.33M / 86,400 ≈ 40 QPS

Peak write QPS: 40 x 3 = 120 QPS


READ TRAFFIC (Redirecting):
─────────────────────────────────────
Read:Write ratio = 100:1
Read QPS: 40 x 100 = 4,000 QPS

Peak read QPS: 4,000 x 3 = 12,000 QPS


SUMMARY:
- Write QPS: 40 (peak: 120)
- Read QPS: 4,000 (peak: 12,000)
- Total QPS: 4,040 (peak: 12,120)
```

### Storage Estimation

```
DATA PER URL:
─────────────────────────────────────
- short_url (7 chars):     7 bytes
- original_url (200 avg):  200 bytes
- user_id:                 8 bytes
- created_at:              8 bytes
- expiry_date:             8 bytes
- click_count:             4 bytes
─────────────────────────────────────
Total:                     ~235 bytes → round to 250 bytes


5-YEAR STORAGE:
─────────────────────────────────────
URLs per year: 100M x 12 = 1.2 billion
URLs in 5 years: 6 billion

Storage = 6 billion x 250 bytes
        = 1.5 TB

With 3x replication: 4.5 TB

Adding indexes (short_url):
Index size = 6B x 20 bytes = 120 GB
Total with indexes: ~5 TB
```

### Bandwidth Estimation

```
INCOMING (Write requests):
─────────────────────────────────────
Request size: 200 bytes (original URL) + 100 bytes (headers)
Incoming = 120 QPS x 300 bytes = 36 KB/s = 0.3 Mbps


OUTGOING (Redirect responses):
─────────────────────────────────────
Response: 300 bytes (redirect + headers)
Outgoing = 12,000 QPS x 300 bytes = 3.6 MB/s = 28.8 Mbps

With overhead: ~35 Mbps total
```

### Infrastructure Estimation

```
WEB SERVERS:
─────────────────────────────────────
Peak QPS: 12,120
Server capacity: 10,000 QPS per server (simple redirect)
Servers needed: 2 (+ 1 for redundancy) = 3 servers


CACHE (Redis):
─────────────────────────────────────
Hot URLs (20% of traffic, 80% of reads): 1.2B x 0.2 = 240M URLs
Cache size: 240M x 250 bytes = 60 GB
Redis servers (32 GB usable): 2-3 servers


DATABASE:
─────────────────────────────────────
Total storage: 5 TB
Write QPS: 120 (easily handled by single primary)
Read QPS: 12,000 (mostly served by cache)
Cache miss (5%): 600 QPS → 1 replica sufficient

Database setup: 1 primary + 1 replica


SUMMARY:
─────────────────────────────────────
Web servers: 3
Cache servers: 3 (Redis cluster)
Database: 2 (primary + replica)
Total storage: 5 TB
Total bandwidth: 35 Mbps
```

---

## Example 2: Twitter-like Service

### Requirements Gathering

```
Functional:
- Post tweets (280 characters max)
- Follow/unfollow users
- Home timeline (tweets from followed users)
- User timeline (user's own tweets)
- Search tweets

Non-functional:
- High availability
- Timeline generation < 200ms
- Eventually consistent (acceptable delay for timeline)
```

### Assumptions

```
Scale assumptions:
- 500 million total users
- 200 million DAU
- Each user:
  - Posts 2 tweets per day
  - Views timeline 10 times per day
  - Each timeline view shows 20 tweets
- Average user follows 200 people
- 20% of tweets have media (image/video)
- Media: 1 MB average (compressed)
- Retention: Forever (10 years for estimation)
```

### Traffic Estimation

```
TWEET CREATION (Writes):
─────────────────────────────────────
Tweets per day: 200M DAU x 2 = 400 million
Tweets per second: 400M / 86,400 ≈ 4,600 QPS

Peak write QPS: 4,600 x 3 = 13,800 QPS


TIMELINE READS:
─────────────────────────────────────
Timeline requests per day: 200M x 10 = 2 billion
Timeline QPS: 2B / 86,400 ≈ 23,000 QPS

Peak read QPS: 23,000 x 3 = 69,000 QPS

Each timeline = 20 tweet fetches (fanout on read)
Effective read QPS: 69,000 x 20 = 1.38M QPS
(This shows why caching/pre-computation is critical)


MEDIA REQUESTS:
─────────────────────────────────────
Timeline views with media: 2B x 0.2 x 0.5 (assume half load media)
Media requests/day: 200 million
Media QPS: 200M / 86,400 ≈ 2,300 QPS

Peak media QPS: 6,900 QPS


SUMMARY:
- Tweet write QPS: 4,600 (peak: 13,800)
- Timeline read QPS: 23,000 (peak: 69,000)
- Media read QPS: 2,300 (peak: 6,900)
```

### Storage Estimation

```
TWEET STORAGE:
─────────────────────────────────────
Tweet data:
- tweet_id: 8 bytes
- user_id: 8 bytes
- content (280 UTF-8): 560 bytes
- created_at: 8 bytes
- likes_count: 4 bytes
- retweet_count: 4 bytes
- reply_to_id: 8 bytes
- media_ids: 20 bytes
─────────────────────────────────────
Total per tweet: ~620 bytes → round to 700 bytes

Tweets per year: 400M x 365 = 146 billion
10-year tweets: 1.46 trillion

Tweet storage = 1.46T x 700 bytes = 1.02 PB

With 3x replication: 3.06 PB


MEDIA STORAGE:
─────────────────────────────────────
Daily media uploads: 400M x 0.2 = 80 million
Media per year: 80M x 365 = 29.2 billion
10-year media: 292 billion files

Media storage = 292B x 1 MB = 292 PB

With 3x replication: 876 PB ≈ 900 PB


USER DATA:
─────────────────────────────────────
User profile: 5 KB per user
500M users x 5 KB = 2.5 TB
With replication: 7.5 TB


FOLLOW GRAPH:
─────────────────────────────────────
Follow relationship: 16 bytes (follower_id + followee_id)
Total relationships: 500M users x 200 avg follows = 100 billion
Storage: 100B x 16 bytes = 1.6 TB
With replication: 4.8 TB


TOTAL STORAGE SUMMARY:
─────────────────────────────────────
Tweets: 3 PB
Media: 900 PB (dominates!)
Users: 8 TB
Follow graph: 5 TB
─────────────────────────────────────
Total: ~903 PB ≈ 1 EB (Exabyte)
```

### Bandwidth Estimation

```
TWEET CREATION:
─────────────────────────────────────
Text tweets: 13,800 x 700 bytes = 9.7 MB/s
Media uploads: 13,800 x 0.2 x 1 MB = 2.76 GB/s

Incoming bandwidth: ~2.8 GB/s = 22.4 Gbps


TIMELINE READS:
─────────────────────────────────────
Timeline data: 69,000 x 20 tweets x 700 bytes = 966 MB/s
Media serving: 6,900 x 1 MB = 6.9 GB/s

Outgoing bandwidth: ~8 GB/s = 64 Gbps


TOTAL BANDWIDTH:
Incoming: 22.4 Gbps (mostly media uploads)
Outgoing: 64 Gbps (mostly media serving)

Note: CDN handles most media, reducing origin bandwidth significantly
With 95% CDN hit rate: Origin outgoing = 3.2 Gbps
```

### Infrastructure Estimation

```
TWEET SERVICE:
─────────────────────────────────────
Write QPS: 13,800
Server capacity: 5,000 QPS
Servers: 3 (+ redundancy) = 5 servers


TIMELINE SERVICE:
─────────────────────────────────────
Two approaches:

Approach 1: Fanout on Read
- Fetch tweets from all followed users on request
- Pros: Simple, instant consistency
- Cons: Slow for users with many followers

Approach 2: Fanout on Write (Recommended)
- Pre-compute timelines when tweet posted
- Pros: Fast reads, predictable latency
- Cons: Celebrity problem, storage overhead

Timeline cache size:
- 200M DAU x 20 tweets x 8 bytes (tweet_id) = 32 GB
- Add tweet content for hot tweets: +200 GB
- Total: ~250 GB across Redis cluster

Timeline servers: 10 servers (Redis + application)


DATABASE CLUSTER:
─────────────────────────────────────
Tweet DB:
- 3 PB storage
- Shard by tweet_id (time-based)
- 500 shards x 6 TB each
- Each shard: 1 primary + 2 replicas = 1,500 DB instances

User DB:
- Much smaller, 3 replicas sufficient

Follow Graph:
- Graph database (Neo4j) or sharded relational
- 10 shards sufficient


MEDIA STORAGE:
─────────────────────────────────────
Object storage (S3-like): 900 PB
CDN POPs: 100+ globally


SEARCH:
─────────────────────────────────────
Elasticsearch cluster for tweet search
Index size: ~500 TB
Nodes: 100+ search nodes


SUMMARY:
─────────────────────────────────────
Tweet service: 5 servers
Timeline service: 10 servers
Cache: 20 Redis nodes
Tweet DB: 1,500 instances
Media storage: 900 PB (object storage)
Search: 100 nodes
CDN: Global distribution
```

---

## Example 3: YouTube-like Video Platform

### Requirements Gathering

```
Functional:
- Upload videos
- Stream videos
- Search videos
- Like/comment on videos
- Recommendations
- Subscriptions

Non-functional:
- High availability
- Low latency streaming
- Multiple video qualities
- Global reach
```

### Assumptions

```
Scale assumptions:
- 2 billion monthly active users
- 800 million DAU
- 5 million video uploads per day
- Average video length: 10 minutes
- Average video watch time per user: 30 minutes/day
- Video qualities: 360p, 720p, 1080p, 4K
- Average consumed quality: 720p (5 Mbps bitrate)
```

### Traffic Estimation

```
VIDEO UPLOADS:
─────────────────────────────────────
Uploads per day: 5 million
Uploads per second: 5M / 86,400 ≈ 58 uploads/sec

Peak upload QPS: 58 x 3 = 174 uploads/sec

Upload size (original): 500 MB average (10 min at high quality)


VIDEO VIEWS:
─────────────────────────────────────
Daily watch time: 800M users x 30 min = 24 billion minutes
Videos watched (avg 10 min): 2.4 billion videos/day

Video start QPS: 2.4B / 86,400 ≈ 27,800 QPS
Peak video start QPS: 83,400 QPS


CONCURRENT VIEWERS:
─────────────────────────────────────
At any moment, assume 10% of DAU watching:
Concurrent viewers: 800M x 0.1 = 80 million


API REQUESTS (search, browse, comments):
─────────────────────────────────────
Each video view = 5-10 API calls (metadata, recommendations, etc.)
API QPS: 27,800 x 7 = 194,600 QPS
Peak API QPS: 583,800 QPS


SUMMARY:
- Upload QPS: 58 (peak: 174)
- Video start QPS: 27,800 (peak: 83,400)
- Concurrent streams: 80 million
- API QPS: 195,000 (peak: 584,000)
```

### Storage Estimation

```
VIDEO STORAGE:
─────────────────────────────────────
Daily uploads: 5 million
Average raw video: 500 MB
Daily raw upload: 2.5 PB

Transcoded sizes (per video):
- 360p: 150 MB
- 720p: 500 MB
- 1080p: 1.5 GB
- 4K: 5 GB
─────────────────────────────────────
Total per video: ~7.15 GB

Daily transcoded: 5M x 7.15 GB = 35.75 PB
Annual: 35.75 PB x 365 = 13 EB (Exabytes)
5-year: 65 EB

With 2x redundancy: 130 EB


VIDEO METADATA:
─────────────────────────────────────
Per video:
- video_id: 8 bytes
- title: 100 bytes
- description: 5 KB
- tags: 200 bytes
- thumbnails: 50 KB
- user_id: 8 bytes
- upload_date: 8 bytes
- duration: 4 bytes
- view_count: 8 bytes
- statistics: 200 bytes
─────────────────────────────────────
Total: ~56 KB per video

5-year videos: 5M x 365 x 5 = 9.125 billion
Metadata storage: 9.125B x 56 KB = 511 TB

With 3x replication: 1.5 PB


USER DATA:
─────────────────────────────────────
Users: 2 billion
Profile data: 10 KB per user
User storage: 20 TB

Watch history: 1 KB per video watched
Average 1000 videos per user
History storage: 2B x 1000 x 1 KB = 2 PB


TOTAL STORAGE SUMMARY:
─────────────────────────────────────
Videos (5 years): 130 EB
Metadata: 1.5 PB
User data: 2 PB
─────────────────────────────────────
Total: ~130 EB (videos dominate)
```

### Bandwidth Estimation

```
VIDEO UPLOAD BANDWIDTH:
─────────────────────────────────────
Upload rate: 174 uploads/sec x 500 MB
= 87 GB/s = 696 Gbps (peak)

Sustained: ~230 Gbps


VIDEO STREAMING BANDWIDTH:
─────────────────────────────────────
Concurrent viewers: 80 million
Average bitrate: 5 Mbps (720p)

Streaming bandwidth = 80M x 5 Mbps = 400 Tbps

This is served globally via CDN


CDN TO ORIGIN:
─────────────────────────────────────
Cache hit rate: 95% (popular videos)
Origin traffic: 400 Tbps x 0.05 = 20 Tbps

Origin bandwidth: 20 Tbps


TOTAL BANDWIDTH:
─────────────────────────────────────
Upload: 230 Gbps (sustained), 700 Gbps (peak)
Download (CDN): 400 Tbps (global)
Origin: 20 Tbps
```

### Infrastructure Estimation

```
UPLOAD SERVICE:
─────────────────────────────────────
Upload QPS: 174 (peak)
Large file handling, needs chunked upload
Servers: 50 upload servers (globally distributed)


VIDEO PROCESSING:
─────────────────────────────────────
Videos to process: 5M / day = 58/sec
Processing time: 1 hour per video (all qualities)
Processing capacity per server: 10 videos/hour

Servers needed: 58 x 60 x 60 / 10 = 20,880
With headroom: 25,000 transcoding servers


API SERVERS:
─────────────────────────────────────
Peak QPS: 584,000
Server capacity: 10,000 QPS
Servers: 59 (+ redundancy) = 100 API servers


RECOMMENDATION ENGINE:
─────────────────────────────────────
ML model serving
Batch inference for most users
Real-time for active sessions
GPU servers: 500 (for model serving)


DATABASE:
─────────────────────────────────────
Video metadata: 1.5 PB
Sharded by video_id
Shards: 1,500 (1 TB each)
With replicas: 4,500 DB instances

User database:
Sharded by user_id
Shards: 500
With replicas: 1,500 instances


CACHE:
─────────────────────────────────────
Hot video metadata: 100M videos x 56 KB = 5.6 TB
Distributed cache: 200 Redis nodes


CDN:
─────────────────────────────────────
Global POPs: 200+ locations
Edge storage: 1 EB total (hot videos)
POP capacity: 2-10 Tbps each


OBJECT STORAGE:
─────────────────────────────────────
Video files: 130 EB
Custom object storage system (like GCS/S3)
Multiple data centers for redundancy


SUMMARY:
─────────────────────────────────────
Upload servers: 50
Transcoding: 25,000 servers
API servers: 100
Recommendation: 500 GPU servers
Metadata DB: 6,000 instances
Cache: 200 nodes
Object storage: 130 EB
CDN: 200+ POPs
```

---

## Quick Reference: Estimation Templates

### Template 1: Generic Web Application

```
INPUTS:
- DAU: ___
- Actions per user per day: ___
- Data per action: ___
- Retention period: ___

CALCULATIONS:

Traffic:
- Daily actions = DAU x actions_per_user
- QPS = daily_actions / 86,400
- Peak QPS = QPS x 3

Storage:
- Annual data = daily_actions x 365 x data_per_action
- Total = annual x years x replication

Bandwidth:
- Incoming = write_QPS x request_size x 8
- Outgoing = read_QPS x response_size x 8

Servers:
- Compute = peak_QPS / QPS_per_server
- Add redundancy (50%) and headroom (30%)
```

### Template 2: Social Media Platform

```
INPUTS:
- Total users: ___
- DAU: ___
- Posts per user per day: ___
- Content consumption per day: ___
- Followers per user: ___
- % with media: ___

CALCULATIONS:

Post traffic:
- Post QPS = DAU x posts / 86,400

Feed traffic:
- Feed QPS = DAU x views / 86,400
- Effective QPS = feed_QPS x items_per_feed

Storage:
- Posts = total_posts x post_size
- Media = posts_with_media x media_size
- Social graph = users x followers x edge_size

Timeline strategy:
- Fanout on write for most users
- Fanout on read for celebrities
```

### Template 3: Streaming Platform

```
INPUTS:
- MAU: ___
- DAU: ___
- Watch time per user: ___
- Uploads per day: ___
- Content length: ___

CALCULATIONS:

Upload traffic:
- Upload QPS = uploads / 86,400
- Upload bandwidth = QPS x file_size

Streaming:
- Concurrent viewers = DAU x online_ratio
- Stream bandwidth = viewers x bitrate

Storage:
- Daily = uploads x (sum of all qualities)
- Annual = daily x 365
- N-year = annual x N x replication

Processing:
- Transcoding servers = uploads_per_hour / videos_per_server_per_hour
```

---

## Common Mistakes to Avoid

### 1. Unit Confusion

```
WRONG:
- Mixing bits and bytes
- Forgetting to multiply by 8 for bandwidth

RIGHT:
- Always specify: "10 KB" or "80 Kbps"
- Bandwidth = data_size x 8 (bits per byte)
```

### 2. Forgetting Peak Traffic

```
WRONG:
- Sizing for average load

RIGHT:
- Peak = Average x 2 to 5
- Size for peak + headroom
```

### 3. Ignoring Replication

```
WRONG:
- Raw storage only

RIGHT:
- Database: 3x replication
- Files: 2-3x replication
- Cache: depends on architecture
```

### 4. Missing Components

```
WRONG:
- Only calculating primary data

RIGHT:
- Include indexes (20-50% of data)
- Include metadata
- Include logs and backups
```

### 5. Unrealistic Server Capacity

```
WRONG:
- Assuming 100% CPU utilization
- Using theoretical maximums

RIGHT:
- Target 60-70% utilization
- Use real-world benchmarks
- Account for overhead
```

---

## Practice Problems

### Problem 1: Instagram Stories

Design capacity for Instagram Stories feature.

<details>
<summary>Solution</summary>

```
ASSUMPTIONS:
- 500M DAU
- 20% post stories daily (100M)
- Average 3 stories per user posting
- Story = 15 seconds video, 10 MB
- View stories: 80% of DAU
- Average 20 stories viewed per user
- Stories expire after 24 hours

TRAFFIC:
Story uploads: 100M x 3 / 86,400 = 3,472 QPS
Story views: 400M x 20 / 86,400 = 92,593 QPS

STORAGE (Rolling 24-hour):
Daily stories: 300M x 10 MB = 3 PB
With replication: 9 PB (rolling, not cumulative)

BANDWIDTH:
Upload: 3,472 x 10 MB = 34.7 GB/s = 278 Gbps
Download: 92,593 x 10 MB = 926 GB/s = 7.4 Tbps

INFRASTRUCTURE:
Upload servers: 100
Storage: 9 PB (SSD for fast access)
CDN: Aggressive caching (content expires quickly)
```

</details>

### Problem 2: Uber Surge Pricing

Design capacity for real-time surge pricing.

<details>
<summary>Solution</summary>

```
ASSUMPTIONS:
- 1M concurrent active rides
- 5M driver locations tracked
- Location update every 4 seconds
- Surge calculation per city zone (10,000 zones globally)
- Recalculate surge every 30 seconds

TRAFFIC:
Location updates: 5M / 4 = 1.25M updates/sec
Surge calculations: 10,000 zones / 30 sec = 333 calcs/sec
Price queries: 1M rides x 0.1 (10% new requests) / sec = 100K QPS

STORAGE:
Location history (last hour): 5M x 900 x 32 bytes = 144 GB
Zone definitions: 10,000 x 10 KB = 100 MB
Historical surge data: 10,000 x 365 x 48 x 100 bytes = 175 GB

PROCESSING:
Each surge calc: aggregate supply/demand in zone
Processing time: 10ms
CPU-seconds: 333 x 0.01 = 3.33/sec
Servers (16 core, 70%): 1 server sufficient

INFRASTRUCTURE:
Location service: 50 servers (1.25M writes/sec)
Surge compute: 5 servers (with redundancy)
Redis cluster: 10 nodes (location caching)
Time-series DB: 20 nodes (historical data)
```

</details>

### Problem 3: Netflix Download Feature

Design capacity for offline downloads.

<details>
<summary>Solution</summary>

```
ASSUMPTIONS:
- 200M subscribers
- 10% use download feature monthly
- Average 5 downloads per month
- Average download: 2 GB (720p movie)
- Downloads stored for 30 days
- Peak download time: 6-10 PM local

TRAFFIC:
Monthly downloads: 20M x 5 = 100M
Daily downloads: 100M / 30 = 3.33M
Download QPS: 3.33M / 86,400 = 39 QPS average

Peak (4-hour window): 39 x 6 = 234 QPS

BANDWIDTH:
Average: 39 x 2 GB / 3600 = 21.7 MB/s = 173 Mbps
Peak: 234 x 2 GB / 3600 = 130 MB/s = 1 Gbps

Per-region (assume 10 regions):
Peak per region: 100 Mbps

STORAGE (on Netflix side):
Just serving from existing video storage
No additional storage needed

CLIENT CONSIDERATIONS:
Active downloads: 20M users x 5 movies x 2 GB = 200 PB total on devices
Average per user: 10 GB

INFRASTRUCTURE:
Download servers: 50 globally (low volume)
DRM license servers: 20 (license checks)
Leverage existing CDN infrastructure
```

</details>

---

## Interview Tips

### Do's

1. **State assumptions clearly** - "I'm assuming 100 million DAU"
2. **Round numbers** - Use powers of 10 for easy math
3. **Show your work** - Walk through calculations step by step
4. **Sanity check** - "This is similar to X which has Y, so this seems reasonable"
5. **Consider trade-offs** - "We could reduce storage by compressing..."

### Don'ts

1. **Don't aim for precision** - Order of magnitude is enough
2. **Don't forget peaks** - Systems must handle peak load
3. **Don't skip units** - Always specify bytes, bits, seconds
4. **Don't assume perfect efficiency** - Add overhead, redundancy
5. **Don't rush** - Take time to organize your approach

### Useful Comparisons

```
KNOWN SYSTEM SCALES:

Twitter (2024):
- 500M+ tweets/day
- 200M+ DAU

YouTube:
- 500 hours video uploaded/minute
- 1B+ hours watched/day

Netflix:
- 200M+ subscribers
- 15% of global internet bandwidth

Uber:
- 100M+ monthly users
- 15M+ trips/day
```

---

## Summary

Back-of-envelope calculations follow a consistent pattern:

1. **Start with users** - MAU, DAU, concurrent users
2. **Convert to actions** - Requests per user, frequency
3. **Calculate QPS** - Actions / seconds in day
4. **Estimate storage** - Objects x size x time x replication
5. **Estimate bandwidth** - QPS x data size x 8
6. **Size infrastructure** - QPS / capacity per server
7. **Add safety factors** - Peak (3x), redundancy (1.5x), headroom (1.3x)

Key numbers to memorize:
- 86,400 seconds/day (use 100,000)
- 1 million/day = 12/second
- 1 billion/day = 12,000/second
- Peak = Average x 2-5
- Replication = 2-3x
