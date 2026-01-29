# Pastebin System Design - Text/Code Storage Service

## 1. Problem Statement

Design a web service like Pastebin that allows users to store and share plain text or code snippets. Users can paste text, get a unique URL, and share it with others. The service should support syntax highlighting, expiration times, and optional password protection.

**Core Functionality:**
- Store text/code snippets with unique shareable URLs
- Support syntax highlighting for various programming languages
- Allow public, unlisted, and private (password-protected) pastes
- Set expiration times for pastes
- View paste history for registered users

**Real-World Examples:**
- Pastebin.com
- GitHub Gists
- Hastebin
- Pastie
- PrivateBin

---

## 2. Requirements

### 2.1 Functional Requirements

| Requirement | Description |
|-------------|-------------|
| **Create Paste** | Store text with optional title, syntax highlighting, expiration |
| **Read Paste** | Retrieve paste content via unique URL |
| **Expire Pastes** | Auto-delete pastes after specified time |
| **Syntax Highlighting** | Support highlighting for 50+ languages |
| **Access Control** | Public, unlisted (link only), private (password) |
| **Edit/Delete** | Registered users can manage their pastes |
| **Raw View** | Provide raw text endpoint for programmatic access |

### 2.2 Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Availability** | 99.9% uptime |
| **Latency** | Read < 100ms, Write < 200ms |
| **Durability** | No data loss for non-expired pastes |
| **Scalability** | Handle 10M+ pastes per day |
| **Security** | Prevent abuse, rate limiting |

### 2.3 Out of Scope
- Real-time collaborative editing
- Version control / diff
- API integrations (Slack, Discord bots)
- Rich text formatting

---

## 3. Capacity Estimation

### 3.1 Traffic Estimates

```
Assumptions:
- 10 million new pastes per day
- Read:Write ratio = 5:1 (most pastes read multiple times)
- Average paste size: 10 KB
- 20% of pastes have expiration < 24 hours
- Data retention: 10 years for permanent pastes

Write Operations:
  10M pastes/day
  = 10M / 86,400 seconds
  = ~116 pastes/second
  Peak (3x): ~350 pastes/second

Read Operations:
  50M reads/day (5:1 ratio)
  = 50M / 86,400 seconds
  = ~580 reads/second
  Peak (3x): ~1,740 reads/second
```

### 3.2 Storage Estimates

```
Per Paste Entry:
  - paste_id: 8 bytes
  - short_key: 8 bytes
  - user_id: 8 bytes (nullable)
  - title: 100 bytes (average)
  - content: 10 KB (average)
  - language: 20 bytes
  - expiration: 8 bytes
  - created_at: 8 bytes
  - password_hash: 64 bytes (nullable)
  - metadata: 100 bytes
  Total: ~10.5 KB per paste

Storage Calculations:
  Daily storage: 10M pastes × 10.5 KB = 105 GB/day
  Yearly storage: 105 GB × 365 = 38.3 TB/year
  10-year storage: 383 TB

  Accounting for 50% expiration:
  Effective storage: ~190 TB over 10 years
```

### 3.3 Bandwidth Estimates

```
Incoming (writes):
  116 pastes/sec × 10.5 KB = 1.2 MB/s
  Peak: 3.6 MB/s

Outgoing (reads):
  580 reads/sec × 10.5 KB = 6.1 MB/s
  Peak: 18.3 MB/s

Total bandwidth: ~22 MB/s peak
```

### 3.4 Memory Estimates (Caching)

```
Cache Strategy: Cache hot pastes (80-20 rule)
  Daily unique reads: 50M / 5 (average reads per paste) = 10M unique pastes
  Cache 20% of daily active pastes: 2M pastes

  Memory needed: 2M × 10.5 KB = 21 GB
  With overhead: ~30 GB Redis cache
```

---

## 4. High-Level Design

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                   CLIENTS                                        │
│                      (Web Browser, CLI Tools, API Clients)                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                 CDN (CloudFlare)                                 │
│                    (Static assets, cached public pastes)                         │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              LOAD BALANCER (L7)                                  │
│                           (AWS ALB / Nginx / HAProxy)                            │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                   ┌───────────────────┼───────────────────┐
                   ▼                   ▼                   ▼
           ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
           │  API Server  │    │  API Server  │    │  API Server  │
           │      #1      │    │      #2      │    │      #N      │
           └──────────────┘    └──────────────┘    └──────────────┘
                   │                   │                   │
                   └───────────────────┼───────────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
           ▼                           ▼                           ▼
    ┌─────────────┐           ┌──────────────┐           ┌──────────────┐
    │   Redis     │           │     Key      │           │    Rate      │
    │   Cache     │           │  Generation  │           │   Limiter    │
    │             │           │   Service    │           │   (Redis)    │
    └─────────────┘           └──────────────┘           └──────────────┘
           │                           │
           │                           │
           └───────────────┬───────────┘
                           │
                           ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                        DATABASE LAYER                            │
    │                                                                  │
    │   ┌─────────────────┐              ┌─────────────────┐          │
    │   │   Metadata DB   │              │   Content Store │          │
    │   │    (MySQL)      │              │   (Object Store)│          │
    │   │                 │              │   (S3 / MinIO)  │          │
    │   │  - paste_id     │              │                 │          │
    │   │  - short_key    │              │  - Large pastes │          │
    │   │  - user_id      │              │  - > 64 KB      │          │
    │   │  - metadata     │              │                 │          │
    │   └─────────────────┘              └─────────────────┘          │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │                    BACKGROUND WORKERS                            │
    │                                                                  │
    │   ┌─────────────────┐              ┌─────────────────┐          │
    │   │   Expiration    │              │    Cleanup      │          │
    │   │    Worker       │              │    Worker       │          │
    │   │                 │              │                 │          │
    │   │ - Scan expired  │              │ - Delete old    │          │
    │   │ - Mark deleted  │              │ - Archive data  │          │
    │   └─────────────────┘              └─────────────────┘          │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Load Balancer** | Nginx/ALB | Distribute traffic, SSL termination |
| **API Servers** | Go/Node.js | Handle paste CRUD operations |
| **Redis Cache** | Redis Cluster | Cache hot pastes, rate limiting |
| **Key Generation** | Custom Service | Generate unique short keys |
| **Metadata DB** | MySQL/PostgreSQL | Store paste metadata |
| **Content Store** | S3/MinIO | Store large paste contents |
| **Expiration Worker** | Cron Job | Clean up expired pastes |

---

## 5. API Design

### 5.1 Create Paste

```http
POST /api/v1/pastes
Content-Type: application/json
Authorization: Bearer <token>  (optional)

Request Body:
{
    "content": "def hello():\n    print('Hello, World!')",
    "title": "My Python Script",
    "language": "python",
    "expiration": "1d",           // 10m, 1h, 1d, 1w, 1m, never
    "visibility": "unlisted",     // public, unlisted, private
    "password": "secretpass",     // optional, for private
    "burn_after_read": false      // delete after first view
}

Response (201 Created):
{
    "paste_id": "abc12345",
    "url": "https://paste.example.com/abc12345",
    "raw_url": "https://paste.example.com/raw/abc12345",
    "title": "My Python Script",
    "language": "python",
    "visibility": "unlisted",
    "expires_at": "2024-01-16T10:30:00Z",
    "created_at": "2024-01-15T10:30:00Z"
}

Response (400 Bad Request):
{
    "error": "Content exceeds maximum size (512 KB)",
    "code": "CONTENT_TOO_LARGE"
}
```

### 5.2 Read Paste

```http
GET /api/v1/pastes/{paste_id}
Authorization: Bearer <token>  (optional)

Query Parameters:
  ?password=secretpass    (for private pastes)

Response (200 OK):
{
    "paste_id": "abc12345",
    "title": "My Python Script",
    "content": "def hello():\n    print('Hello, World!')",
    "language": "python",
    "visibility": "unlisted",
    "created_at": "2024-01-15T10:30:00Z",
    "expires_at": "2024-01-16T10:30:00Z",
    "views": 42,
    "user": {
        "id": "user_123",
        "username": "johndoe"
    }
}

Response (404 Not Found):
{
    "error": "Paste not found or expired",
    "code": "PASTE_NOT_FOUND"
}

Response (401 Unauthorized):
{
    "error": "Password required for this paste",
    "code": "PASSWORD_REQUIRED"
}
```

### 5.3 Get Raw Content

```http
GET /raw/{paste_id}

Response (200 OK):
Content-Type: text/plain

def hello():
    print('Hello, World!')
```

### 5.4 Delete Paste

```http
DELETE /api/v1/pastes/{paste_id}
Authorization: Bearer <token>

Response (204 No Content)

Response (403 Forbidden):
{
    "error": "You don't have permission to delete this paste",
    "code": "FORBIDDEN"
}
```

### 5.5 List User Pastes

```http
GET /api/v1/users/me/pastes?cursor={cursor}&limit={limit}
Authorization: Bearer <token>

Response (200 OK):
{
    "pastes": [
        {
            "paste_id": "abc12345",
            "title": "My Python Script",
            "language": "python",
            "visibility": "unlisted",
            "created_at": "2024-01-15T10:30:00Z",
            "views": 42
        },
        ...
    ],
    "next_cursor": "cursor_xyz",
    "has_more": true
}
```

---

## 6. Database Schema

### 6.1 MySQL - Metadata Tables

```sql
-- Pastes Metadata Table
CREATE TABLE pastes (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    paste_key       VARCHAR(12) UNIQUE NOT NULL,
    user_id         BIGINT NULL,
    title           VARCHAR(255),
    language        VARCHAR(50) DEFAULT 'text',
    content_size    INT NOT NULL,
    content_hash    VARCHAR(64),           -- SHA256 for deduplication
    storage_type    ENUM('inline', 's3') DEFAULT 'inline',
    s3_key          VARCHAR(255) NULL,     -- If stored in S3
    visibility      ENUM('public', 'unlisted', 'private') DEFAULT 'public',
    password_hash   VARCHAR(255) NULL,
    burn_after_read BOOLEAN DEFAULT FALSE,
    view_count      INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at      TIMESTAMP NULL,
    deleted_at      TIMESTAMP NULL,

    INDEX idx_paste_key (paste_key),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at),
    INDEX idx_visibility (visibility),
    INDEX idx_created_at (created_at),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Inline Content Table (for small pastes < 64KB)
CREATE TABLE paste_contents (
    paste_id        BIGINT PRIMARY KEY,
    content         MEDIUMTEXT NOT NULL,

    FOREIGN KEY (paste_id) REFERENCES pastes(id) ON DELETE CASCADE
);

-- Users Table
CREATE TABLE users (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    username        VARCHAR(50) UNIQUE NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    api_key         VARCHAR(64) UNIQUE,
    is_premium      BOOLEAN DEFAULT FALSE,
    paste_count     INT DEFAULT 0,
    storage_used    BIGINT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_username (username),
    INDEX idx_email (email),
    INDEX idx_api_key (api_key)
);

-- Key Pool Table (for pre-generated keys)
CREATE TABLE key_pool (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    paste_key       VARCHAR(12) UNIQUE NOT NULL,
    is_used         BOOLEAN DEFAULT FALSE,
    used_at         TIMESTAMP NULL,

    INDEX idx_unused (is_used)
);
```

### 6.2 Data Model Diagram

```
┌─────────────────────────────────────────────┐
│                   pastes                     │
├─────────────────────────────────────────────┤
│ PK │ id              │ BIGINT               │
│    │ paste_key       │ VARCHAR(12) UNIQUE   │
│ FK │ user_id         │ BIGINT               │
│    │ title           │ VARCHAR(255)         │
│    │ language        │ VARCHAR(50)          │
│    │ content_size    │ INT                  │
│    │ storage_type    │ ENUM                 │
│    │ visibility      │ ENUM                 │
│    │ expires_at      │ TIMESTAMP            │
│    │ created_at      │ TIMESTAMP            │
└─────────────────────────────────────────────┘
              │ 1:1
              ▼
┌─────────────────────────────────────────────┐
│              paste_contents                  │
├─────────────────────────────────────────────┤
│ PK │ paste_id        │ BIGINT               │
│    │ content         │ MEDIUMTEXT           │
└─────────────────────────────────────────────┘

              │ N:1
              ▼
┌─────────────────────────────────────────────┐
│                   users                      │
├─────────────────────────────────────────────┤
│ PK │ id              │ BIGINT               │
│    │ username        │ VARCHAR(50) UNIQUE   │
│    │ email           │ VARCHAR(255) UNIQUE  │
│    │ api_key         │ VARCHAR(64) UNIQUE   │
│    │ is_premium      │ BOOLEAN              │
│    │ paste_count     │ INT                  │
└─────────────────────────────────────────────┘
```

---

## 7. Deep Dive: Critical Components

### 7.1 Key Generation Strategy

Similar to URL shortener, we need unique, short keys for paste URLs.

```
Key Requirements:
- Length: 8 characters (62^8 = 218 trillion combinations)
- Character set: a-z, A-Z, 0-9
- No offensive words
- Not easily guessable (for unlisted pastes)
```

**Approach: Pre-Generated Key Pool**

```python
import secrets
import string

class KeyGenerator:
    CHARSET = string.ascii_letters + string.digits
    KEY_LENGTH = 8
    BATCH_SIZE = 10000

    def __init__(self, db_connection, redis_client):
        self.db = db_connection
        self.redis = redis_client
        self.local_keys = []

    def generate_key(self):
        """Generate a cryptographically random key"""
        return ''.join(secrets.choice(self.CHARSET)
                      for _ in range(self.KEY_LENGTH))

    def get_key(self):
        """Get a unique key from the pool"""
        # Try local cache first
        if self.local_keys:
            return self.local_keys.pop()

        # Try Redis pool
        key = self.redis.spop("key_pool:available")
        if key:
            return key.decode('utf-8')

        # Fetch batch from database
        self._refill_local_cache()
        if self.local_keys:
            return self.local_keys.pop()

        # Emergency: generate inline (rare)
        return self._generate_and_reserve()

    def _refill_local_cache(self):
        """Fetch unused keys from database"""
        keys = self.db.fetch_all(
            "SELECT paste_key FROM key_pool WHERE is_used = FALSE LIMIT %s",
            (self.BATCH_SIZE,)
        )

        if keys:
            # Mark as used in database
            key_list = [k['paste_key'] for k in keys]
            self.db.execute(
                "UPDATE key_pool SET is_used = TRUE WHERE paste_key IN %s",
                (tuple(key_list),)
            )
            self.local_keys = key_list

    def _generate_and_reserve(self):
        """Generate a new key and reserve it"""
        for _ in range(10):  # Max 10 attempts
            key = self.generate_key()
            try:
                self.db.execute(
                    "INSERT INTO key_pool (paste_key, is_used, used_at) "
                    "VALUES (%s, TRUE, NOW())",
                    (key,)
                )
                return key
            except DuplicateKeyError:
                continue
        raise Exception("Failed to generate unique key")


# Background job to populate key pool
def populate_key_pool():
    """Run daily to maintain key pool"""
    TARGET_POOL_SIZE = 1_000_000

    current_size = db.fetch_one(
        "SELECT COUNT(*) FROM key_pool WHERE is_used = FALSE"
    )['count']

    needed = TARGET_POOL_SIZE - current_size

    if needed > 0:
        generator = KeyGenerator(db, redis)
        batch = []

        for _ in range(needed):
            key = generator.generate_key()
            batch.append((key, False))

            if len(batch) >= 10000:
                db.execute_many(
                    "INSERT IGNORE INTO key_pool (paste_key, is_used) VALUES (%s, %s)",
                    batch
                )
                batch = []

        if batch:
            db.execute_many(
                "INSERT IGNORE INTO key_pool (paste_key, is_used) VALUES (%s, %s)",
                batch
            )
```

### 7.2 Content Storage Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE DECISION FLOW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Incoming Paste Content                                         │
│           │                                                      │
│           ▼                                                      │
│   ┌───────────────────┐                                         │
│   │  Content Size?    │                                         │
│   └───────────────────┘                                         │
│        │           │                                             │
│      < 64 KB    >= 64 KB                                        │
│        │           │                                             │
│        ▼           ▼                                             │
│   ┌─────────┐  ┌─────────┐                                      │
│   │  MySQL  │  │   S3    │                                      │
│   │ Inline  │  │ Object  │                                      │
│   └─────────┘  └─────────┘                                      │
│                                                                  │
│   Benefits:                                                      │
│   - MySQL: Fast for small pastes (90%+ of traffic)              │
│   - S3: Cost-effective for large pastes                         │
│   - S3: No database bloat                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Storage Implementation:**

```python
class ContentStorage:
    INLINE_THRESHOLD = 64 * 1024  # 64 KB

    def __init__(self, db, s3_client):
        self.db = db
        self.s3 = s3_client
        self.bucket = "pastebin-content"

    def store(self, paste_id, content):
        """Store content in appropriate storage"""
        content_bytes = content.encode('utf-8')
        content_size = len(content_bytes)
        content_hash = hashlib.sha256(content_bytes).hexdigest()

        # Check for duplicate content (deduplication)
        existing = self.db.fetch_one(
            "SELECT paste_key FROM pastes WHERE content_hash = %s",
            (content_hash,)
        )

        if content_size < self.INLINE_THRESHOLD:
            # Store inline in MySQL
            self._store_inline(paste_id, content)
            return {
                'storage_type': 'inline',
                's3_key': None,
                'content_size': content_size,
                'content_hash': content_hash
            }
        else:
            # Store in S3
            s3_key = self._store_s3(paste_id, content_bytes)
            return {
                'storage_type': 's3',
                's3_key': s3_key,
                'content_size': content_size,
                'content_hash': content_hash
            }

    def _store_inline(self, paste_id, content):
        self.db.execute(
            "INSERT INTO paste_contents (paste_id, content) VALUES (%s, %s)",
            (paste_id, content)
        )

    def _store_s3(self, paste_id, content_bytes):
        s3_key = f"pastes/{paste_id[:2]}/{paste_id}"
        self.s3.put_object(
            Bucket=self.bucket,
            Key=s3_key,
            Body=content_bytes,
            ContentType='text/plain'
        )
        return s3_key

    def retrieve(self, paste):
        """Retrieve content from storage"""
        if paste['storage_type'] == 'inline':
            result = self.db.fetch_one(
                "SELECT content FROM paste_contents WHERE paste_id = %s",
                (paste['id'],)
            )
            return result['content'] if result else None
        else:
            response = self.s3.get_object(
                Bucket=self.bucket,
                Key=paste['s3_key']
            )
            return response['Body'].read().decode('utf-8')
```

### 7.3 Expiration and Cleanup System

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXPIRATION SYSTEM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Two Cleanup Strategies:                                        │
│                                                                  │
│   1. LAZY DELETION (On Read)                                     │
│      ┌─────────────┐                                            │
│      │ Read Request│                                            │
│      └──────┬──────┘                                            │
│             │                                                    │
│             ▼                                                    │
│      ┌─────────────────┐     Yes     ┌─────────────┐           │
│      │  Is Expired?    │────────────▶│ Return 404  │           │
│      └─────────────────┘             │ Mark Deleted│           │
│             │ No                     └─────────────┘           │
│             ▼                                                    │
│      ┌─────────────┐                                            │
│      │Return Paste │                                            │
│      └─────────────┘                                            │
│                                                                  │
│   2. BACKGROUND CLEANUP (Scheduled)                              │
│      ┌─────────────────────────────────────────────────────┐    │
│      │  Cron Job: Every 5 minutes                          │    │
│      │                                                     │    │
│      │  1. SELECT pastes WHERE expires_at < NOW()          │    │
│      │     AND deleted_at IS NULL LIMIT 1000               │    │
│      │                                                     │    │
│      │  2. For each expired paste:                         │    │
│      │     - Delete from S3 (if applicable)                │    │
│      │     - Delete from paste_contents                    │    │
│      │     - Update pastes SET deleted_at = NOW()          │    │
│      │     - Invalidate cache                              │    │
│      │                                                     │    │
│      │  3. Archive metadata (for analytics)                │    │
│      └─────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Cleanup Worker Implementation:**

```python
class ExpirationWorker:
    BATCH_SIZE = 1000

    def __init__(self, db, s3, redis, metrics):
        self.db = db
        self.s3 = s3
        self.redis = redis
        self.metrics = metrics

    def run(self):
        """Main cleanup loop"""
        while True:
            try:
                deleted_count = self._cleanup_batch()
                self.metrics.increment('pastes_expired', deleted_count)

                if deleted_count < self.BATCH_SIZE:
                    # No more expired pastes, wait before next check
                    time.sleep(60)
            except Exception as e:
                self.metrics.increment('cleanup_errors')
                logging.error(f"Cleanup error: {e}")
                time.sleep(5)

    def _cleanup_batch(self):
        """Clean up a batch of expired pastes"""
        expired = self.db.fetch_all("""
            SELECT id, paste_key, storage_type, s3_key
            FROM pastes
            WHERE expires_at < NOW()
            AND deleted_at IS NULL
            LIMIT %s
            FOR UPDATE SKIP LOCKED
        """, (self.BATCH_SIZE,))

        if not expired:
            return 0

        for paste in expired:
            self._delete_paste(paste)

        return len(expired)

    def _delete_paste(self, paste):
        """Delete a single paste"""
        try:
            # Delete S3 content
            if paste['storage_type'] == 's3' and paste['s3_key']:
                self.s3.delete_object(
                    Bucket='pastebin-content',
                    Key=paste['s3_key']
                )

            # Delete inline content
            self.db.execute(
                "DELETE FROM paste_contents WHERE paste_id = %s",
                (paste['id'],)
            )

            # Soft delete paste metadata
            self.db.execute(
                "UPDATE pastes SET deleted_at = NOW() WHERE id = %s",
                (paste['id'],)
            )

            # Invalidate cache
            self.redis.delete(f"paste:{paste['paste_key']}")

        except Exception as e:
            logging.error(f"Failed to delete paste {paste['paste_key']}: {e}")
            raise
```

---

## 8. Scaling Discussion

### 8.1 Database Scaling

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE SCALING STRATEGY                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Phase 1: Vertical Scaling (< 100M pastes)                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Single MySQL instance with read replicas               │   │
│   │  - Master: Writes                                       │   │
│   │  - Replicas: Reads                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Phase 2: Horizontal Sharding (100M+ pastes)                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Shard by paste_key hash                                │   │
│   │                                                         │   │
│   │  shard_id = hash(paste_key) % num_shards               │   │
│   │                                                         │   │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │
│   │  │ Shard 0 │  │ Shard 1 │  │ Shard 2 │  │ Shard N │   │   │
│   │  │ a-f     │  │ g-l     │  │ m-r     │  │ s-z     │   │   │
│   │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   User Data: Separate database, sharded by user_id              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Caching Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-TIER CACHING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Layer 1: CDN (CloudFlare)                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  - Cache public paste HTML pages                        │   │
│   │  - Cache raw content endpoints                          │   │
│   │  - TTL: 5 minutes (balance freshness vs performance)    │   │
│   │  - Bypass for private/unlisted                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Layer 2: Application Cache (Redis)                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Key: paste:{paste_key}                                 │   │
│   │  Value: serialized paste object (metadata + content)    │   │
│   │  TTL: 1 hour for hot pastes, 5 min for others          │   │
│   │                                                         │   │
│   │  Key: user_pastes:{user_id}                            │   │
│   │  Value: list of user's paste IDs                       │   │
│   │  TTL: 10 minutes                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Layer 3: Local In-Memory Cache                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  - LRU cache on each app server                        │   │
│   │  - Hot keys only (> 10 requests/minute)                │   │
│   │  - Size: 100 MB per server                             │   │
│   │  - TTL: 30 seconds                                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 Read vs Write Path Optimization

```
Write Path (Optimized for Durability):
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Client  │───▶│ Server  │───▶│ Database│───▶│ Response│
└─────────┘    └─────────┘    │ (Sync)  │    └─────────┘
                              └─────────┘
                                   │
                                   ▼ (Async)
                              ┌─────────┐
                              │  Cache  │
                              │ Warmup  │
                              └─────────┘

Read Path (Optimized for Speed):
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Client  │───▶│  CDN    │───▶│ Return  │  (Cache Hit)
└─────────┘    └─────────┘    └─────────┘
                    │
                    ▼ (Cache Miss)
               ┌─────────┐    ┌─────────┐
               │  Redis  │───▶│ Return  │  (Cache Hit)
               └─────────┘    └─────────┘
                    │
                    ▼ (Cache Miss)
               ┌─────────┐    ┌─────────┐
               │   DB    │───▶│ Return  │
               └─────────┘    │& Cache  │
                              └─────────┘
```

---

## 9. Trade-offs

### 9.1 Key Generation Approaches

| Approach | Pros | Cons |
|----------|------|------|
| **Pre-generated pool** | Fast, no collisions | Storage overhead, complexity |
| **Hash-based** | Simple | Collision handling needed |
| **Snowflake ID** | Distributed, ordered | Longer keys (19 chars) |
| **UUID** | Unique, simple | Very long (36 chars) |

**Decision: Pre-generated pool** - Best balance of speed and short URLs.

### 9.2 Storage Choices

| Choice | Use Case | Trade-off |
|--------|----------|-----------|
| **All in MySQL** | Simple, ACID | Database bloat, slow backups |
| **All in S3** | Scalable, cheap | Latency for small pastes |
| **Hybrid (MySQL + S3)** | Optimized for both | Complexity in code |

**Decision: Hybrid** - MySQL for < 64KB, S3 for larger pastes.

### 9.3 Expiration Strategies

| Strategy | Pros | Cons |
|----------|------|------|
| **Lazy deletion** | No background jobs | Stale data in storage |
| **Active cleanup** | Clean storage | CPU/IO overhead |
| **TTL in database** | Automatic (Cassandra/DynamoDB) | Limited to specific DBs |

**Decision: Both** - Lazy deletion for immediate 404s, active cleanup for storage.

### 9.4 Syntax Highlighting

| Approach | Pros | Cons |
|----------|------|------|
| **Server-side (highlight.js)** | Cached HTML, fast display | CPU intensive on server |
| **Client-side** | Offload processing | Slower page load |
| **Pre-rendered** | Instant display | Storage overhead |

**Decision: Client-side** - Offload CPU, better scalability.

---

## 10. Failure Scenarios

### 10.1 Database Failure

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE FAILURE HANDLING                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Scenario: Primary MySQL goes down                             │
│                                                                  │
│   Impact:                                                        │
│   - Writes: BLOCKED (no new pastes)                             │
│   - Reads: DEGRADED (cache hits only)                           │
│                                                                  │
│   Mitigation:                                                    │
│   1. Automatic failover to replica (< 30 seconds)               │
│   2. Circuit breaker for database calls                         │
│   3. Queue writes in Kafka during outage                        │
│   4. Replay queued writes after recovery                        │
│                                                                  │
│   Recovery Steps:                                                │
│   1. Promote replica to primary                                 │
│   2. Update connection strings                                  │
│   3. Process queued writes                                      │
│   4. Rebuild failed primary as new replica                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Cache Failure

```
Impact: Increased database load, higher latency
Mitigation:
- Redis Cluster with automatic failover
- Local in-memory cache as fallback
- Database connection pooling
- Graceful degradation (serve from DB)
```

### 10.3 S3 Failure

```
Impact: Large pastes unavailable
Mitigation:
- S3 cross-region replication
- Retry with exponential backoff
- Show error message for affected pastes
- Metadata still accessible
```

### 10.4 Key Generation Service Failure

```
Impact: Cannot create new pastes
Mitigation:
- Local key buffer on each server (1000 keys)
- Fallback to inline generation with collision check
- Multiple key generation instances
- Pre-generate large key pool (1M+ keys)
```

---

## 11. Interview Variations

### 11.1 Common Follow-up Questions

**Q1: How would you implement burn-after-read?**
```
Answer:
1. Set burn_after_read = TRUE in database
2. On read request:
   - Fetch paste content
   - Immediately delete from all storage
   - Invalidate cache
   - Return content (one-time only)
3. Use database transaction to ensure atomicity
4. Consider race conditions (use distributed lock)
```

**Q2: How would you prevent abuse?**
```
Answer:
1. Rate limiting per IP (10 pastes/hour for anonymous)
2. Rate limiting per API key (100 pastes/hour)
3. Content scanning for malware/spam
4. CAPTCHA for suspicious activity
5. Blacklist known spam patterns
6. Limit paste size (512 KB max)
7. Report mechanism for users
```

**Q3: How would you implement search?**
```
Answer:
1. Elasticsearch for full-text search
2. Index only public pastes
3. Index fields: title, content, language, username
4. Async indexing via message queue
5. Consider privacy implications
6. Rate limit search queries
```

**Q4: How would you handle code execution (like Replit)?**
```
Answer:
1. Sandboxed containers (Docker/Firecracker)
2. Strict resource limits (CPU, memory, time)
3. Network isolation
4. Separate infrastructure from main service
5. Queue-based execution
6. Output streaming via WebSocket
```

**Q5: How would you implement collaborative editing?**
```
Answer:
1. Operational Transformation (OT) or CRDT
2. WebSocket for real-time sync
3. Document versioning
4. Conflict resolution
5. Cursor/selection sharing
6. This significantly changes architecture (see Google Docs design)
```

### 11.2 Variations

**Variation 1: Design Pastebin for Enterprise**
```
Additional Requirements:
- SSO integration (SAML, OAuth)
- Team/organization support
- Access control lists
- Audit logging
- Data residency compliance
- End-to-end encryption
```

**Variation 2: Design Pastebin at 100x Scale**
```
Changes Needed:
- Global distribution (multi-region)
- More aggressive caching
- Sharding from day 1
- CDN for all content
- Async writes with eventual consistency
- Read replicas in each region
```

**Variation 3: Design a Code Snippet Manager (GitHub Gist)**
```
Additional Features:
- Version history (Git-based)
- Forking and stars
- Comments and discussions
- Multiple files per gist
- Embed support
- API for CI/CD integration
```

### 11.3 Red Flags to Avoid

| Mistake | Why It's Wrong |
|---------|----------------|
| Storing all content in single DB | Will become bottleneck |
| No expiration strategy | Unbounded storage growth |
| Sequential key generation | Predictable URLs |
| No rate limiting | Abuse vulnerability |
| Synchronous everything | Performance issues |

### 11.4 Good Signs in Your Answer

| Indicator | Why It's Good |
|-----------|---------------|
| Discuss key generation early | Core challenge identified |
| Separate metadata from content | Good data modeling |
| Multiple caching layers | Performance awareness |
| Consider abuse prevention | Security mindset |
| Async cleanup process | Scalability thinking |

---

## Summary

| Aspect | Decision |
|--------|----------|
| **Key Generation** | Pre-generated pool (8-char alphanumeric) |
| **Storage** | Hybrid: MySQL (< 64KB), S3 (>= 64KB) |
| **Caching** | CDN + Redis + Local LRU |
| **Expiration** | Lazy deletion + Background cleanup |
| **Database** | MySQL with read replicas, shard at scale |

```
┌─────────────────────────────────────────────────────────────────┐
│                    PASTEBIN DESIGN SUMMARY                       │
├─────────────────────────────────────────────────────────────────┤
│ Scale: 10M pastes/day, 50M reads/day                            │
│                                                                  │
│ Key Challenges:                                                  │
│   1. Unique short key generation                                │
│   2. Efficient storage for varying content sizes                │
│   3. Automatic expiration handling                              │
│   4. Abuse prevention                                           │
│                                                                  │
│ Storage Estimates:                                              │
│   - Daily: 105 GB                                               │
│   - 10-year: ~190 TB (with expirations)                        │
│                                                                  │
│ Key Numbers:                                                    │
│   - Write QPS: ~116 (peak 350)                                  │
│   - Read QPS: ~580 (peak 1,740)                                 │
│   - Latency target: < 100ms reads                               │
└─────────────────────────────────────────────────────────────────┘
```
