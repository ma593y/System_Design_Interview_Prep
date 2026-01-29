# Storage Calculations

## Overview

Storage estimation is one of the most critical skills in system design interviews. Understanding how to calculate data sizes, plan storage capacity, and estimate growth helps you make informed decisions about database choices, sharding strategies, and infrastructure costs. This guide provides formulas, worked examples, and practice problems.

---

## Key Concepts and Formulas

### Data Size Units

```
Bit (b)        = 1 or 0
Byte (B)       = 8 bits
Kilobyte (KB)  = 1,000 bytes (10^3)
Megabyte (MB)  = 1,000,000 bytes (10^6)
Gigabyte (GB)  = 1,000,000,000 bytes (10^9)
Terabyte (TB)  = 1,000,000,000,000 bytes (10^12)
Petabyte (PB)  = 1,000,000,000,000,000 bytes (10^15)

Note: Storage industry uses base-10 (SI units)
      Memory/Computing often uses base-2 (KiB, MiB, GiB)

Binary equivalents (for reference):
KiB = 1,024 bytes
MiB = 1,048,576 bytes
GiB = 1,073,741,824 bytes
```

### Basic Storage Formula

```
Total Storage = Number of Objects x Size per Object x Retention Period

With replication:
Total Storage = Objects x Size x Retention x Replication Factor

With growth:
Storage at Year N = Initial Storage x (1 + Growth Rate)^N
```

---

## Common Data Type Sizes

### Primitive Types

| Type | Size | Notes |
|------|------|-------|
| Boolean | 1 byte | Often padded to 1 byte |
| Char | 1-4 bytes | ASCII: 1B, UTF-8: 1-4B |
| Integer (32-bit) | 4 bytes | Range: -2B to +2B |
| Integer (64-bit) | 8 bytes | For large IDs, timestamps |
| Float | 4 bytes | 32-bit floating point |
| Double | 8 bytes | 64-bit floating point |
| UUID | 16 bytes | 128-bit identifier |
| Timestamp | 8 bytes | Unix timestamp (64-bit) |

### Common Object Sizes

| Object Type | Typical Size | Notes |
|-------------|--------------|-------|
| Tweet/Status | 250-500 bytes | 280 chars + metadata |
| User profile | 1-5 KB | Name, bio, settings |
| Product listing | 2-10 KB | Description, specs |
| Email | 50-100 KB | With small attachments |
| Profile photo | 100-500 KB | Compressed JPEG |
| High-res image | 2-5 MB | PNG/JPEG |
| 1-min video (720p) | 50-100 MB | Compressed |
| 1-min video (4K) | 300-500 MB | Compressed |

### Database Row Overhead

```
MySQL/PostgreSQL row overhead: ~30-50 bytes
MongoDB document overhead: ~16 bytes minimum
Index entry overhead: ~20-30 bytes per entry

Total row size = Data + Row overhead + Index overhead
```

---

## Storage Calculation Examples

### Example 1: URL Shortener

**Requirements**:
- 100 million new URLs per month
- Store for 5 years
- 100:1 read-to-write ratio

**Data Model**:
```
Short URL entry:
- short_code (7 chars)     : 7 bytes
- original_url (avg 200)   : 200 bytes
- user_id (64-bit)         : 8 bytes
- created_at (timestamp)   : 8 bytes
- click_count (32-bit)     : 4 bytes
- Row overhead             : 30 bytes
---------------------------------
Total per URL              : ~257 bytes (round to 300 bytes)
```

**Storage Calculation**:
```
URLs over 5 years:
= 100M/month x 12 months x 5 years
= 6 billion URLs

Raw storage:
= 6B x 300 bytes
= 1.8 TB

With 3x replication:
= 1.8 TB x 3
= 5.4 TB

Adding indexes (short_code, user_id):
Index size = 6B x 50 bytes = 300 GB
Total with indexes: 5.4 TB + (300 GB x 3) = 6.3 TB

Annual growth:
Year 1: 1.26 TB
Year 2: 2.52 TB
Year 3: 3.78 TB
Year 4: 5.04 TB
Year 5: 6.30 TB
```

### Example 2: Social Media Platform (Twitter-like)

**Requirements**:
- 500 million DAU
- Each user posts 2 tweets/day
- 30% of tweets have images
- Store for 10 years

**Data Model**:
```
Tweet:
- tweet_id (64-bit)        : 8 bytes
- user_id (64-bit)         : 8 bytes
- content (280 chars UTF-8): 560 bytes (worst case)
- created_at               : 8 bytes
- likes_count              : 4 bytes
- retweet_count            : 4 bytes
- reply_to_id              : 8 bytes
- media_ids (array)        : 24 bytes
- Row overhead             : 50 bytes
---------------------------------
Total per tweet            : ~674 bytes (round to 700 bytes)

Image (compressed thumbnail + full):
- Thumbnail (50 KB)
- Full image (200 KB average)
- Total: 250 KB per image
```

**Storage Calculation**:
```
Daily tweets:
= 500M users x 2 tweets
= 1 billion tweets/day

Annual tweets:
= 1B x 365
= 365 billion tweets/year

10-year tweet storage:
= 365B x 10 x 700 bytes
= 2.555 PB (text only)

Image storage (10 years):
= 365B x 10 x 0.3 x 250 KB
= 273.75 PB

Total storage:
Text: 2.555 PB x 3 (replication) = 7.67 PB
Images: 273.75 PB x 3 = 821.25 PB

Total: ~830 PB over 10 years
```

### Example 3: Video Streaming Platform (YouTube-like)

**Requirements**:
- 500 hours of video uploaded per minute
- Store videos for 10 years
- Multiple quality versions (360p, 720p, 1080p, 4K)

**Data Model**:
```
Video metadata:
- video_id (64-bit)        : 8 bytes
- title (100 chars)        : 100 bytes
- description (5000 chars) : 5000 bytes
- user_id                  : 8 bytes
- upload_date              : 8 bytes
- duration_seconds         : 4 bytes
- view_count               : 8 bytes
- Various flags/settings   : 100 bytes
---------------------------------
Metadata per video         : ~5.2 KB

Video file sizes (per minute):
- 360p:  5 MB/min
- 720p:  25 MB/min
- 1080p: 60 MB/min
- 4K:    300 MB/min
---------------------------------
Total per minute of video: ~390 MB
```

**Storage Calculation**:
```
Daily uploads:
= 500 hours/min x 60 min x 24 hours
= 720,000 hours/day
= 43.2 million minutes/day

Daily storage (video only):
= 43.2M minutes x 390 MB
= 16.85 PB/day

Annual storage:
= 16.85 PB x 365
= 6.15 EB (Exabytes)/year

10-year storage:
= 6.15 EB x 10
= 61.5 EB

With 2x redundancy:
= 123 EB

Plus metadata:
Videos per day: 720,000 hours / 10 min avg = 4.32M videos/day
10-year metadata: 4.32M x 365 x 10 x 5.2 KB = 82 TB
```

---

## Database Storage Considerations

### Index Storage

```
B-tree index storage:
Index size = Number of rows x (Key size + Pointer size)
Pointer size: typically 8-12 bytes

Example (1 billion rows, 8-byte key):
Index size = 1B x (8 + 10) = 18 GB

Compound index (2 columns, 8 bytes each):
Index size = 1B x (16 + 10) = 26 GB
```

### Write Amplification

```
Write amplification factor = Total data written / User data written

Sources of write amplification:
- Write-ahead logging (WAL): 2x
- B-tree page splits: 1.5-3x
- Replication: Nx (N = replica count)
- Compaction (LSM trees): 10-30x

Example: 100 GB of user data
With WAL + B-tree + 3x replication:
Actual writes = 100 GB x 2 x 2 x 3 = 1.2 TB
```

### Compression Ratios

| Data Type | Typical Compression | Notes |
|-----------|---------------------|-------|
| Text (JSON) | 3-5x | gzip/snappy |
| Text (logs) | 5-10x | Repetitive patterns |
| Images (JPEG) | Already compressed | ~0.95x (minimal) |
| Video | Already compressed | ~0.95x (minimal) |
| Database rows | 2-4x | Mixed data |
| Time series | 10-20x | Delta encoding |

---

## Cloud Storage Cost Estimation

### AWS S3 Pricing Reference (approximate)

```
Storage Classes (per GB/month):
- S3 Standard:           $0.023
- S3 Infrequent Access:  $0.0125
- S3 Glacier:            $0.004
- S3 Glacier Deep:       $0.00099

Data Transfer (per GB):
- Into S3:               Free
- Out to Internet:       $0.09 (first 10 TB)
- Between regions:       $0.02

Requests (per 1000):
- PUT/POST:              $0.005
- GET:                   $0.0004
```

### Cost Calculation Example

```
Scenario: Image hosting service
- 10 TB active images (Standard)
- 100 TB archive images (Glacier)
- 50 TB data transfer out/month
- 100M GET requests/month

Monthly costs:
Storage:
- Standard: 10 TB x $0.023 = $230
- Glacier: 100 TB x $0.004 = $400

Transfer:
- 50 TB x $0.09 = $4,500

Requests:
- 100M GETs x $0.0004/1000 = $40

Total monthly: ~$5,170
Annual: ~$62,000
```

---

## Storage Planning Formulas

### Capacity Planning Formula

```
Required Storage = (Current Data + Growth) x Safety Factor x Replication

Where:
- Current Data = Objects x Size per Object
- Growth = Current x Growth Rate x Time Period
- Safety Factor = 1.3 to 1.5 (for headroom)
- Replication = 2 to 3 (for redundancy)
```

### Time to Fill Storage

```
Time to Fill = Available Storage / Daily Growth Rate

Example:
Available: 100 TB
Daily writes: 1 TB/day
Time to fill: 100 days

With 80% threshold for expansion:
Time to trigger: 80 days
```

### Storage IOPS Requirements

```
IOPS = QPS x Operations per Query

Storage capacity needed:
SSD (100K IOPS): IOPS / 100,000 drives
HDD (150 IOPS): IOPS / 150 drives

Example: 50,000 IOPS required
SSDs needed: 50,000 / 100,000 = 1 SSD
HDDs needed: 50,000 / 150 = 334 HDDs
```

---

## Worked Examples

### Example 1: Chat Application

**Requirements**:
- 100 million DAU
- 50 messages per user per day
- Store messages for 2 years
- 10% of messages have attachments (avg 500 KB)

**Solution**:
```
Step 1: Calculate message volume
Daily messages = 100M x 50 = 5 billion/day
Annual messages = 5B x 365 = 1.825 trillion/year
2-year messages = 3.65 trillion messages

Step 2: Calculate message size
Message schema:
- message_id: 8 bytes
- sender_id: 8 bytes
- receiver_id: 8 bytes
- content (avg 200 chars): 200 bytes
- timestamp: 8 bytes
- status flags: 4 bytes
- overhead: 30 bytes
Total: ~266 bytes (round to 300 bytes)

Step 3: Calculate raw storage
Text storage = 3.65T x 300 bytes = 1.095 PB

Step 4: Calculate attachment storage
Attachments = 3.65T x 0.1 x 500 KB = 182.5 PB

Step 5: Apply replication (3x)
Text: 1.095 PB x 3 = 3.285 PB
Attachments: 182.5 PB x 3 = 547.5 PB

Step 6: Add indexes (message_id, sender_id, receiver_id)
Index size per message: 3 x 20 bytes = 60 bytes
Total indexes: 3.65T x 60 x 3 = 657 TB

Total storage: 3.285 PB + 547.5 PB + 0.657 PB = ~551.5 PB
```

### Example 2: E-commerce Product Catalog

**Requirements**:
- 50 million products
- Each product has 10 images
- Store 5 years of product history
- 20% product turnover per year

**Solution**:
```
Step 1: Calculate product data
Product schema:
- product_id: 8 bytes
- name (100 chars): 100 bytes
- description (2000 chars): 2000 bytes
- category_ids: 50 bytes
- price history: 100 bytes
- attributes (JSON): 1000 bytes
- timestamps: 24 bytes
- overhead: 50 bytes
Total: ~3.3 KB per product

Step 2: Calculate image storage
Images per product: 10
Average image size: 200 KB
Total per product: 2 MB

Step 3: Product accumulation over 5 years
Year 1: 50M products
Year 2: 50M + (50M x 0.2) = 60M cumulative active
        But storing history: 50M + 10M new = 60M total
Year 3: 60M + 12M = 72M
Year 4: 72M + 14.4M = 86.4M
Year 5: 86.4M + 17.3M = 103.7M total products stored

Step 4: Calculate total storage
Product data: 103.7M x 3.3 KB = 342 GB
Images: 103.7M x 2 MB = 207.4 TB

Step 5: Apply replication (3x)
Product data: 342 GB x 3 = 1.03 TB
Images: 207.4 TB x 3 = 622.2 TB

Total: ~623 TB
```

### Example 3: IoT Sensor Data

**Requirements**:
- 10 million sensors
- Each sensor reports every 5 seconds
- Store raw data for 30 days, aggregated for 2 years
- Aggregate = 1-minute and 1-hour rollups

**Solution**:
```
Step 1: Calculate data volume
Readings per sensor per day:
= (24 x 60 x 60) / 5 = 17,280 readings/day

Total readings per day:
= 10M x 17,280 = 172.8 billion readings/day

Step 2: Calculate reading size
Reading schema:
- sensor_id: 8 bytes
- timestamp: 8 bytes
- value (double): 8 bytes
- quality_flag: 1 byte
Total: 25 bytes per reading

Step 3: Raw data storage (30 days)
= 172.8B x 30 x 25 bytes
= 129.6 TB raw data

Step 4: Aggregated storage
1-minute aggregates:
= 10M sensors x 1440 min/day x 365 days x 2 years
= 10.5 trillion aggregates
= 10.5T x 50 bytes (with min/max/avg) = 525 TB

1-hour aggregates:
= 10M x 24 x 365 x 2 = 175.2 billion
= 175.2B x 50 bytes = 8.76 TB

Step 5: Apply compression (time series ~10x)
Raw: 129.6 TB / 10 = 13 TB
Aggregates: (525 + 8.76) TB / 10 = 53.4 TB

Step 6: Apply replication (3x)
Total = (13 + 53.4) TB x 3 = 199 TB
```

---

## Quick Reference Tables

### Storage Quick Conversions

| From | To | Multiply by |
|------|-----|-------------|
| KB | MB | / 1,000 |
| MB | GB | / 1,000 |
| GB | TB | / 1,000 |
| TB | PB | / 1,000 |
| 1 million | 1 billion | x 1,000 |
| Per day | Per year | x 365 |
| Per second | Per day | x 86,400 |

### Data Size Estimates

| Data Type | Quick Estimate |
|-----------|---------------|
| Character (ASCII) | 1 byte |
| Character (UTF-8 avg) | 2 bytes |
| Integer | 4-8 bytes |
| Timestamp | 8 bytes |
| UUID/GUID | 16 bytes |
| Short text (tweet) | 500 bytes |
| User profile | 5 KB |
| Web page | 2-3 MB |
| Photo | 200-500 KB |
| 1 min video (720p) | 50-100 MB |

### Storage Multipliers

| Factor | Multiplier | When to Apply |
|--------|------------|---------------|
| Replication | 2-3x | Always for production |
| Indexes | 1.2-1.5x | Based on index count |
| Overhead | 1.1-1.2x | DB row overhead |
| Safety margin | 1.3-1.5x | Capacity planning |
| Peak growth | 2x | Seasonal variation |

---

## Practice Problems

### Problem 1: Email Service

**Given**:
- 1 billion users
- Average 40 emails received per day per user
- Average email size: 75 KB
- Store for 7 years
- 3x replication

Calculate total storage required.

<details>
<summary>Solution</summary>

```
Daily emails: 1B x 40 = 40 billion emails/day
Annual emails: 40B x 365 = 14.6 trillion emails/year
7-year emails: 14.6T x 7 = 102.2 trillion emails

Raw storage: 102.2T x 75 KB = 7.67 EB (Exabytes)

With replication: 7.67 EB x 3 = 23 EB

Adding 20% for indexes and metadata: 23 EB x 1.2 = 27.6 EB

With compression (3x for text):
Compressed: 27.6 EB / 3 = ~9.2 EB
```

</details>

### Problem 2: Ride-Sharing GPS Data

**Given**:
- 10 million active rides per day
- Average ride: 20 minutes
- GPS update every 3 seconds
- Each GPS point: 32 bytes
- Store for 90 days

Calculate storage requirements.

<details>
<summary>Solution</summary>

```
GPS points per ride:
= 20 min x 60 sec / 3 sec = 400 points/ride

Daily GPS points:
= 10M rides x 400 points = 4 billion points/day

90-day storage:
= 4B x 90 = 360 billion points

Raw storage:
= 360B x 32 bytes = 11.52 TB

With 3x replication:
= 11.52 TB x 3 = 34.56 TB

With time-series compression (10x):
= 34.56 TB / 10 = 3.46 TB

Adding indexes (ride_id, timestamp):
Index size: 360B x 20 bytes x 3 = 21.6 TB / 10 = 2.16 TB

Total: 3.46 TB + 2.16 TB = ~5.6 TB
```

</details>

### Problem 3: Log Aggregation System

**Given**:
- 10,000 servers
- Each server: 100 MB logs per hour
- Retention: 30 days hot, 1 year cold
- Hot storage: SSD with 3x replication
- Cold storage: HDD with 2x replication

Calculate storage and estimate monthly cost.

<details>
<summary>Solution</summary>

```
Hourly logs: 10,000 x 100 MB = 1 TB/hour
Daily logs: 1 TB x 24 = 24 TB/day

Hot storage (30 days):
Raw: 24 TB x 30 = 720 TB
With compression (10x): 72 TB
With 3x replication: 216 TB SSD

Cold storage (11 months, after hot):
Raw: 24 TB x 335 days = 8.04 PB
With compression (10x): 804 TB
With 2x replication: 1.61 PB HDD

Cost estimation (approximate):
Hot (SSD, $0.10/GB/month): 216 TB x $100 = $21,600/month
Cold (HDD, $0.02/GB/month): 1,610 TB x $20 = $32,200/month

Total monthly: ~$53,800
Annual: ~$645,600
```

</details>

---

## Interview Tips

### Do's

1. **Start with clear assumptions** - State the data model upfront
2. **Round numbers for easy math** - 86,400 seconds ~ 100,000
3. **Show your work** - Interviewers want to see the process
4. **Consider compression** - Most data compresses 2-10x
5. **Always include replication** - Production systems need redundancy
6. **Think about access patterns** - Hot vs cold storage

### Don'ts

1. **Don't forget indexes** - Can add 20-50% to storage
2. **Don't ignore metadata** - Timestamps, IDs, flags add up
3. **Don't assume infinite retention** - Ask about data lifecycle
4. **Don't forget growth** - Plan for 2-3 years ahead
5. **Don't mix units** - Be consistent with bytes vs bits

### Key Questions to Ask

1. "What's the data retention policy?"
2. "What's the expected growth rate?"
3. "What's the read/write ratio?"
4. "What are the access patterns (hot vs cold data)?"
5. "What's the replication requirement?"

---

## Summary

Storage calculation follows a consistent pattern:

1. **Identify the data model** - What are you storing?
2. **Calculate object size** - Sum all fields plus overhead
3. **Estimate volume** - Users x Actions x Time
4. **Apply multipliers** - Replication, indexes, safety margin
5. **Consider compression** - Especially for text and time series
6. **Plan for growth** - 2-3 year horizon minimum

Key numbers to remember:
- 1 billion rows x 1 KB = 1 TB
- 86,400 seconds per day (use 100K for easy math)
- Replication: 3x for databases, 2x for files
- Compression: 3-5x for text, 10x for time series
