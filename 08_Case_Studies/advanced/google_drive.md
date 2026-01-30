# Design Google Drive / Dropbox (File Sync and Storage)

## 1. Problem Statement

Design a cloud file storage and synchronization service similar to Google Drive or
Dropbox. The system must support file upload/download, real-time synchronization across
multiple devices, file versioning, sharing with permissions, conflict resolution for
concurrent edits, offline access, and storage optimization through deduplication and
compression.

---

## 2. Requirements

### Functional Requirements
- Upload and download files (up to 15 GB per file)
- Chunked, resumable uploads for large files
- Real-time sync across all connected devices
- File versioning with ability to restore previous versions
- Sharing files/folders with configurable permissions (view/edit/comment)
- Conflict resolution for concurrent edits
- Folder hierarchy with nested structures
- Offline access with sync queue
- File search by name, content, and metadata
- Trash/recycle bin with recovery
- Notifications for shared file changes

### Non-Functional Requirements
- High availability: 99.99% uptime
- Durability: 99.999999999% (11 nines) — zero data loss
- Consistency: strong consistency for file metadata; eventual for sync propagation
- Low latency: sync propagation within 5 seconds across devices
- Scalability: support 1 billion users, 500 million daily active
- Storage efficiency: block-level deduplication across all users

---

## 3. Capacity Estimation

### User and File Assumptions
```
Total users:              1 billion
Daily active users (DAU): 500 million
Average files per user:   200 files
Total files:              200 billion files
Average file size:        500 KB
Average file operations:  10 per user per day (save, rename, move, delete)

File operations per day:  500M * 10 = 5 billion ops/day
Operations per second:    5B / 86,400 ~ 58,000 ops/sec (average)
Peak operations/sec:      ~175,000 ops/sec (3x average)
```

### Storage Estimation
```
Raw storage:              200B files * 500 KB = 100 PB
With deduplication (~40% savings): 60 PB effective
With versioning (avg 3 versions):  180 PB logical
With replication (3x):            540 PB physical storage
Daily new data:           5B ops * 20% are writes * 500 KB avg = 500 TB/day
Annual growth:            ~180 PB/year
```

### Bandwidth Estimation
```
Upload:   500M DAU * 2 uploads/day * 500 KB = 500 TB/day = ~46 Gbps
Download: 500M DAU * 5 downloads/day * 500 KB = 1.25 PB/day = ~116 Gbps
Sync metadata:  500M DAU * 50 KB/day = 25 TB/day
Peak bandwidth: ~500 Gbps (combined up+down)
```

### Metadata Storage
```
File metadata record:     ~500 bytes each
200 billion files:        200B * 500 B = 100 TB
Block metadata:           200B files * 10 blocks avg * 100 B = 200 TB
Version records:          600B versions * 200 B = 120 TB
Permission records:       50B shares * 100 B = 5 TB
Total metadata:           ~425 TB
```

---

## 4. High-Level Design

```
  +------------------+      +------------------+     +------------------+
  | Desktop Client   |      | Mobile Client    |     | Web Client       |
  | (sync daemon)    |      | (sync service)   |     | (browser)        |
  +--------+---------+      +--------+---------+     +--------+---------+
           |                         |                         |
           +-------------------------+-------------------------+
                                     |
                            +--------+--------+
                            | Load Balancer   |
                            | (L7 / API GW)   |
                            +--------+--------+
                                     |
           +-------------------------+-------------------------+
           |                         |                         |
  +--------+---------+     +---------+--------+     +----------+-------+
  | Upload/Download  |     | Sync Service     |     | Metadata API     |
  | Service          |     | (WebSocket)      |     | Service          |
  +--------+---------+     +---------+--------+     +----------+-------+
           |                         |                         |
           |                +---------+--------+               |
           |                | Notification     |               |
           |                | Service          |               |
           |                | (push/email)     |               |
           |                +------------------+               |
           |                                                   |
           v                                                   v
  +--------+---------+                              +----------+-------+
  | Block Storage    |                              | Metadata DB      |
  | Service          |                              | (PostgreSQL      |
  |                  |                              |  sharded)        |
  | +-------------+  |                              +----------+-------+
  | | Block Index  |  |                                        |
  | | (hash->loc)  |  |                              +----------+-------+
  | +-------------+  |                              | Redis Cache      |
  |                  |                              | (file tree,      |
  +--------+---------+                              |  sessions,       |
           |                                        |  recent changes) |
           v                                        +------------------+
  +--------+---------+
  | Object Storage   |     +-------------------+
  | (S3 / GCS)       |     | Search Service    |
  |                  |     | (Elasticsearch)   |
  | Hot:  SSD        |     +-------------------+
  | Warm: HDD        |
  | Cold: Glacier    |     +-------------------+
  +------------------+     | Queue Service     |
                           | (Kafka / SQS)     |
                           | - Sync events     |
                           | - Dedup jobs       |
                           | - Notifications   |
                           +-------------------+
```

---

## 5. API Design

### File Upload (Chunked, Resumable)

```
POST /api/v1/files/upload/init
Headers: Authorization: Bearer <token>
Body: {
    "file_name": "report.pdf",
    "file_size": 52428800,            // 50 MB
    "parent_folder_id": "folder_123",
    "content_hash": "sha256:abc123...",
    "chunk_size": 4194304             // 4 MB chunks
}
Response 200: {
    "upload_id": "upl_789",
    "file_id": "file_456",
    "upload_url": "https://upload.drive.com/v1/upl_789",
    "chunk_count": 13,
    "dedup_result": "none"            // or "full" (skip upload) or "partial"
}

PUT /api/v1/files/upload/{upload_id}/chunks/{chunk_index}
Headers: Content-Type: application/octet-stream
         Content-SHA256: <chunk_hash>
Body: <binary chunk data>
Response 200: {
    "chunk_index": 0,
    "status": "received",
    "dedup": false                    // true if block already exists
}

POST /api/v1/files/upload/{upload_id}/complete
Response 200: {
    "file_id": "file_456",
    "version": 1,
    "status": "ready",
    "size_bytes": 52428800,
    "dedup_savings_bytes": 0
}
```

### File Sync

```
GET /api/v1/sync/changes?cursor=1705340000&limit=100
Response 200: {
    "changes": [
        {
            "file_id": "file_456",
            "action": "modified",
            "version": 3,
            "modified_by": "user_789",
            "modified_at": "2025-01-15T10:30:00Z",
            "file_hash": "sha256:def456...",
            "blocks_changed": [2, 7, 11]
        },
        {
            "file_id": "file_999",
            "action": "created",
            "version": 1,
            "parent_folder_id": "folder_123",
            "file_name": "notes.txt",
            "file_hash": "sha256:ghi789..."
        }
    ],
    "cursor": 1705340060,
    "has_more": false
}

WebSocket: wss://sync.drive.com/v1/ws
Message (server -> client): {
    "type": "file_change",
    "file_id": "file_456",
    "action": "modified",
    "version": 3,
    "changed_blocks": [2, 7, 11],
    "modified_by": "user_789"
}
```

### File Sharing

```
POST /api/v1/files/{file_id}/share
Body: {
    "recipients": [
        { "email": "bob@example.com", "permission": "edit" },
        { "email": "alice@example.com", "permission": "view" }
    ],
    "message": "Please review this document",
    "notify": true,
    "link_sharing": {
        "enabled": true,
        "permission": "view",
        "expiry": "2025-02-15T00:00:00Z"
    }
}
Response 200: {
    "share_id": "shr_123",
    "share_link": "https://drive.com/s/abc123xyz",
    "recipients_notified": 2
}
```

### File Versioning

```
GET /api/v1/files/{file_id}/versions
Response 200: {
    "versions": [
        {
            "version": 3,
            "modified_by": "user_789",
            "modified_at": "2025-01-15T10:30:00Z",
            "size_bytes": 52430000,
            "change_summary": "Modified blocks 2, 7, 11"
        },
        {
            "version": 2,
            "modified_by": "user_456",
            "modified_at": "2025-01-14T09:00:00Z",
            "size_bytes": 52428800
        }
    ]
}

POST /api/v1/files/{file_id}/versions/{version}/restore
Response 200: {
    "file_id": "file_456",
    "restored_version": 2,
    "new_version": 4
}
```

---

## 6. Database Schema

### Files Table (PostgreSQL, sharded by owner_id)
```sql
CREATE TABLE files (
    file_id         BIGINT PRIMARY KEY,         -- Snowflake ID
    owner_id        BIGINT NOT NULL,
    parent_folder_id BIGINT,                    -- NULL for root
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(50),                -- MIME type
    is_folder       BOOLEAN DEFAULT FALSE,
    current_version INT DEFAULT 1,
    size_bytes      BIGINT,
    content_hash    VARCHAR(128),               -- SHA-256 of full content
    status          ENUM('active','trashed','deleted') DEFAULT 'active',
    trashed_at      TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_parent (parent_folder_id),
    INDEX idx_owner_parent (owner_id, parent_folder_id),
    INDEX idx_owner_updated (owner_id, updated_at),
    INDEX idx_status (status),
    UNIQUE idx_name_parent (parent_folder_id, file_name, status)
);
```

### File Versions Table
```sql
CREATE TABLE file_versions (
    version_id      BIGINT PRIMARY KEY,
    file_id         BIGINT NOT NULL,
    version_number  INT NOT NULL,
    size_bytes      BIGINT,
    content_hash    VARCHAR(128),
    modified_by     BIGINT NOT NULL,
    device_id       VARCHAR(64),
    block_list      JSONB,                      -- ordered list of block hashes
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_file_version (file_id, version_number),
    INDEX idx_file_id (file_id)
);
```

### Blocks Table (Content-Addressable Storage Index)
```sql
CREATE TABLE blocks (
    block_hash      VARCHAR(128) PRIMARY KEY,   -- SHA-256 of block content
    size_bytes      INT NOT NULL,
    storage_url     VARCHAR(2048),              -- S3/GCS path
    reference_count INT DEFAULT 1,              -- for garbage collection
    compression     VARCHAR(10),                -- 'none', 'zstd', 'lz4'
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_refcount (reference_count)
);
```

### Permissions Table
```sql
CREATE TABLE permissions (
    permission_id   BIGINT PRIMARY KEY,
    file_id         BIGINT NOT NULL,
    user_id         BIGINT,                     -- NULL for link sharing
    email           VARCHAR(255),
    permission_type ENUM('owner','edit','comment','view') NOT NULL,
    share_link_token VARCHAR(64),               -- for link-based sharing
    expires_at      TIMESTAMP,
    created_by      BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_file (file_id),
    INDEX idx_user_file (user_id, file_id),
    INDEX idx_share_token (share_link_token),
    UNIQUE idx_file_user (file_id, user_id)
);
```

### Sync Cursors Table
```sql
CREATE TABLE sync_cursors (
    user_id         BIGINT NOT NULL,
    device_id       VARCHAR(64) NOT NULL,
    cursor_value    BIGINT NOT NULL,            -- timestamp or sequence number
    last_sync_at    TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (user_id, device_id)
);
```

### Change Log Table (append-only, for sync)
```sql
CREATE TABLE change_log (
    change_id       BIGINT PRIMARY KEY,         -- monotonically increasing
    file_id         BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    action          ENUM('create','modify','rename','move','trash','restore','delete'),
    old_parent_id   BIGINT,
    new_parent_id   BIGINT,
    old_name        VARCHAR(255),
    new_name        VARCHAR(255),
    version_number  INT,
    timestamp       TIMESTAMP DEFAULT NOW(),

    INDEX idx_user_timestamp (user_id, timestamp),
    INDEX idx_file (file_id)
);
```

---

## 7. Deep Dive

### Deep Dive 1: File Sync Protocol with Block-Level Deduplication

The sync protocol operates at the block level, not the file level. This is the key
insight that makes syncing large files efficient.

```
File Chunking and Block-Level Sync:

  Original File (50 MB)
  +--------+--------+--------+--------+--------+--------+
  | Block  | Block  | Block  | Block  | Block  | Block  |  ...
  |   0    |   1    |   2    |   3    |   4    |   5    |
  | 4 MB   | 4 MB   | 4 MB   | 4 MB   | 4 MB   | 4 MB  |
  | hash:  | hash:  | hash:  | hash:  | hash:  | hash:  |
  | a1b2c3 | d4e5f6 | g7h8i9 | j0k1l2 | m3n4o5 | p6q7r8 |
  +--------+--------+--------+--------+--------+--------+

  Modified File (user edits middle section)
  +--------+--------+--------+--------+--------+--------+
  | Block  | Block  | Block  | Block  | Block  | Block  |  ...
  |   0    |   1    |   2    |   3    |   4    |   5    |
  | 4 MB   | 4 MB   | 4 MB   | 4 MB   | 4 MB   | 4 MB  |
  | hash:  | hash:  | hash:  | hash:  | hash:  | hash:  |
  | a1b2c3 | d4e5f6 | *NEW*  | j0k1l2 | m3n4o5 | p6q7r8 |
  +--------+--------+--------+--------+--------+--------+
                       ^
               Only this block changed!
               Only 4 MB uploaded instead of 50 MB.
```

**Rolling Checksum Algorithm (Rsync-style):**
```
Sync Process:
  1. Client detects file modification (via OS file watcher)
  2. Client splits file into fixed-size blocks (4 MB default)
     - For small files (< 4 MB): single block
     - Variable-size chunking option (Rabin fingerprint) for
       better dedup when content shifts
  3. Client computes SHA-256 hash for each block
  4. Client sends block hash list to server:
     [a1b2c3, d4e5f6, NEW_HASH, j0k1l2, m3n4o5, p6q7r8]
  5. Server compares against previous version's block list:
     [a1b2c3, d4e5f6, g7h8i9, j0k1l2, m3n4o5, p6q7r8]
  6. Server identifies: block 2 has changed (g7h8i9 -> NEW_HASH)
  7. Server checks if NEW_HASH exists in global block index
     - If YES: instant dedup, no upload needed (just reference)
     - If NO:  server requests client upload only block 2
  8. Client uploads only the changed block (4 MB vs 50 MB)
  9. Server stores new block, updates file version with new block list
  10. Server notifies other devices of change (via WebSocket)
```

**Variable-Size Chunking with Rabin Fingerprinting:**
```
Fixed-size chunking problem: inserting 1 byte at start shifts ALL block boundaries
Solution: Content-defined chunking using Rabin fingerprint

Algorithm:
  - Slide a window (48 bytes) over file content
  - Compute rolling hash at each position
  - When hash mod TARGET_SIZE == MAGIC_VALUE, create boundary
  - Minimum block: 1 MB, Maximum block: 8 MB, Target: 4 MB
  - Block boundaries determined by content, not position
  - Result: insertion only affects 1-2 blocks, not all subsequent blocks
```

**Sync State Machine (per device):**
```
                     +----------+
                     |  IDLE    |
                     +----+-----+
                          |
               File change detected
                          |
                     +----v-----+
                     | SCANNING |  (compute block hashes)
                     +----+-----+
                          |
                     +----v-----+
                     | DIFFING  |  (compare with server)
                     +----+-----+
                          |
              +-----------+-----------+
              |                       |
         No changes              Changes found
              |                       |
              v                  +----v-----+
          +---+----+             | UPLOADING|  (upload changed blocks)
          |  IDLE  |             +----+-----+
          +--------+                  |
                                 +----v-----+
                                 | UPDATING |  (update metadata)
                                 +----+-----+
                                      |
                                 +----v-----+
                                 | NOTIFYING|  (push to other devices)
                                 +----+-----+
                                      |
                                 +----v-----+
                                 |   IDLE   |
                                 +----------+
```

### Deep Dive 2: Conflict Resolution for Concurrent Edits

When two users (or two devices) modify the same file simultaneously, we need a robust
conflict resolution strategy.

```
Conflict Scenario:

Timeline:
  t0: File V1 synced to all devices
  t1: User A modifies file on Laptop    (A has V1, creates V2a)
  t2: User B modifies file on Desktop   (B has V1, creates V2b)
  t3: User A uploads V2a to server      (server accepts, V2a becomes V2)
  t4: User B tries to upload V2b        (CONFLICT: base is V1, server has V2)

Detection:
  - Each upload includes base_version (the version the edit was based on)
  - Server checks: base_version == current_version?
  - If yes: accept (fast-forward)
  - If no:  conflict detected
```

**Conflict Resolution Strategies:**

```
Strategy 1: Last Writer Wins (LWW)
  - Simplest approach
  - Most recent write overwrites
  - Risk: data loss (User A's changes lost)
  - Use case: settings files, caches

Strategy 2: Fork and Merge (Used by Google Drive / Dropbox)
  - Server keeps both versions
  - Creates conflict copy: "report (User B's conflicting copy).pdf"
  - User manually resolves
  - Best for binary files (images, PDFs) where auto-merge is impossible

Strategy 3: Operational Transform (Google Docs style)
  - For text/document files only
  - Track individual operations (insert char at pos X, delete range Y-Z)
  - Transform operations against concurrent ops
  - Automatic merge without user intervention
  - Complex to implement correctly

Strategy 4: Block-Level Merge (Dropbox approach)
  - If users modified DIFFERENT blocks: auto-merge (no conflict)
  - If users modified SAME block: create conflict copy
  - Granularity reduces false conflicts significantly
```

**Implementation of Block-Level Merge:**
```
  File Version 1 (server): blocks [A, B, C, D, E]

  User X edits block B:    blocks [A, B', C, D, E]
  User Y edits block D:    blocks [A, B, C, D', E]

  Server receives X first: blocks [A, B', C, D, E]  (V2)
  Server receives Y:       base=V1, but V2 exists

  Block-level comparison:
    Y changed: block D -> D'  (index 3)
    V2 changed: block B -> B' (index 1)
    No overlap!

  Auto-merge result (V3):  blocks [A, B', C, D', E]

  If both had changed block B:
    Create conflict: "file (Y's conflicting copy)" with [A, B_Y, C, D', E]
    Keep server version:  [A, B', C, D, E]
    Notify both users of conflict
```

### Deep Dive 3: Notification System and Real-Time Sync

```
Notification Flow for File Changes:

  +----------+     +-----------+     +-----------+     +-------------+
  | User A   |---->| Upload    |---->| Change    |---->| Notification|
  | modifies |     | Service   |     | Log       |     | Service     |
  | file     |     |           |     | (Kafka)   |     |             |
  +----------+     +-----------+     +-----------+     +------+------+
                                                              |
                          +-----------------------------------+----------+
                          |                                   |          |
                   +------v------+                     +------v------+  |
                   | WebSocket   |                     | Push / Email|  |
                   | Manager     |                     | Service     |  |
                   +------+------+                     +-------------+  |
                          |                                             |
               +----------+----------+                          +-------v------+
               |                     |                          | User C       |
        +------v------+      +------v------+                    | (email       |
        | User A      |      | User B      |                    |  notification|
        | Device 2    |      | (shared     |                    |  - offline)  |
        | (real-time  |      |  collaborator|                    +--------------+
        |  sync)      |      |  real-time) |
        +-------------+      +-------------+
```

**WebSocket Connection Management:**
```
Connection Registry (Redis):
  Key: ws:user:{user_id}:devices
  Value: Hash {
      device_1: { ws_server: "ws-node-03", connected_at: "...", last_ping: "..." },
      device_2: { ws_server: "ws-node-07", connected_at: "...", last_ping: "..." }
  }

Fan-out Process:
  1. File change event published to Kafka topic "file-changes"
  2. Notification service consumes event
  3. Determine affected users:
     - File owner (all their devices)
     - Users with file shared to them
  4. For each affected user:
     a. Look up connected devices in Redis
     b. For each connected device:
        - Route message to correct WebSocket server via internal pub/sub
        - WebSocket server pushes to client
     c. For offline devices:
        - Store pending notification in database
        - Send push notification (mobile) or email (based on preferences)
  5. Client receives notification and fetches changed blocks
```

**Offline Access and Sync Queue:**
```
Client Offline Behavior:
  1. User marks files/folders for offline access
  2. Client downloads and caches all blocks locally
  3. When connection lost:
     - File operations continue against local copy
     - Changes recorded in local sync queue (SQLite DB on device)
     - Queue entry: { file_id, action, timestamp, block_changes, base_version }
  4. When connection restored:
     - Client replays sync queue in order
     - For each queued change:
       a. Upload changed blocks
       b. If conflict detected: apply conflict resolution
       c. Mark queue entry as synced
     - Download any server-side changes made while offline
     - Merge using block-level comparison
```

---

## 8. Scaling Discussion

### Storage Scaling
- Object storage (S3) for blocks — scales horizontally without limit
- Block deduplication reduces storage by 30-50% across all users
- Content-addressable storage: same block stored once regardless of how many files reference it
- Tiered storage: hot (SSD) for recent files, warm (HDD) for older, cold (Glacier) for trash/versions

### Metadata Database Scaling
- Shard PostgreSQL by owner_id (user's files co-located)
- Read replicas for search and listing operations
- Separate change_log table could use Cassandra for append-only high-throughput writes
- Cache file trees in Redis (invalidate on change)

### Sync Service Scaling
- WebSocket servers are stateless (connection registry in Redis)
- Horizontal scaling: add more WebSocket nodes behind load balancer
- Sticky sessions by user_id for connection affinity
- Kafka partitioned by file_id for ordered processing per file

### Upload/Download Scaling
- Direct-to-S3 uploads with pre-signed URLs (bypass application servers)
- CDN for downloads (cache popular shared files at edge)
- Parallel chunk upload from client (4 chunks simultaneously)
- Regional upload endpoints (minimize latency for large files)

### Deduplication Scaling
- Global block index (hash -> storage_url) in distributed hash table
- Bloom filter at upload service for fast negative lookups (block not seen before)
- Background dedup worker for cross-user deduplication
- Per-account dedup is fast (inline); cross-account dedup is async (background)

---

## 9. Trade-offs

| Decision | Option A | Option B | Choice & Rationale |
|----------|----------|----------|-------------------|
| Chunking | Fixed-size | Content-defined (Rabin) | Content-defined for frequently edited files, fixed for others. Content-defined is better for dedup but more CPU intensive. |
| Sync model | Push (WebSocket) | Pull (polling) | Push for active users (real-time), pull for reconnection catch-up. Push reduces latency from minutes to seconds. |
| Conflict resolution | Last writer wins | Fork + manual merge | Fork for binary files (user chooses), auto-merge for text (block-level). LWW causes data loss. |
| Block size | 1 MB | 4 MB | 4 MB: balances dedup granularity (smaller=better dedup) vs metadata overhead (smaller=more blocks to track). |
| Metadata DB | PostgreSQL | DynamoDB | PostgreSQL: need folder hierarchy queries, permission joins, and ACID for rename/move operations. |
| Version storage | Full copy | Delta only | Delta (store only changed blocks per version). Full copies waste storage but simplify restore. Deltas save 80%+ storage. |
| Sync consistency | Strong | Eventual | Eventual for sync propagation (few-second delay acceptable). Strong for metadata writes (prevent duplicate names in folder). |

---

## 10. Failure Scenarios & Handling

### Upload Failure During Chunk Transfer
- **Detection:** Client receives timeout or HTTP error for chunk upload
- **Handling:** Client retries with exponential backoff; server tracks received chunks by upload_id
- **Recovery:** Resume from last acknowledged chunk (not from beginning)
- **Prevention:** Each chunk upload is idempotent (keyed by upload_id + chunk_index)

### Sync Server Crash
- **Detection:** WebSocket connections drop; health check failure
- **Handling:** Client reconnects to different server via load balancer; fetches changes since last cursor
- **Recovery:** No state loss — connection registry in Redis, file state in database
- **Prevention:** Stateless sync servers; all state external (Redis, DB, Kafka)

### Split Brain During Network Partition
- **Detection:** Multiple devices make conflicting changes during partition
- **Handling:** Each device operates on local state; conflicts resolved on reconnection using vector clocks
- **Recovery:** Block-level merge for non-overlapping changes; conflict copies for overlapping changes
- **Prevention:** Cannot prevent entirely; robust conflict resolution is the mitigation

### Storage Backend Failure
- **Detection:** S3/GCS returns errors for read/write operations
- **Handling:** Retry with fallback to secondary region; serve cached copies from CDN
- **Recovery:** Object storage has built-in replication and self-healing
- **Prevention:** Multi-region replication; 11 nines durability guarantee from cloud provider

### Metadata Database Failure
- **Detection:** Connection errors, replication lag alerts
- **Handling:** Promote read replica to primary; route writes to new primary
- **Recovery:** Rebuild failed node from replica + WAL
- **Prevention:** Synchronous replication to standby; multi-AZ deployment

### Deduplication Index Corruption
- **Detection:** Block hash lookups return incorrect storage URLs; integrity check failures
- **Handling:** Fall back to non-dedup mode (store blocks regardless); queue re-indexing job
- **Recovery:** Rebuild block index from object storage metadata scan
- **Prevention:** Checksums on index entries; periodic consistency audits

---

## 11. Monitoring & Alerting

### Key Metrics

```
+------------------------------+------------------+-------------------+
| Metric                       | Target           | Alert Threshold   |
+------------------------------+------------------+-------------------+
| Sync propagation latency     | < 3 seconds      | > 10 seconds      |
| Upload success rate          | > 99.9%          | < 99.5%           |
| Download latency (P99)       | < 500 ms         | > 2 seconds       |
| File operation latency (P50) | < 100 ms         | > 500 ms          |
| WebSocket connection rate    | Baseline +/- 20% | > 50% deviation   |
| Dedup savings rate           | > 35%            | < 20%             |
| Block storage utilization    | < 80%            | > 90%             |
| Conflict rate                | < 0.1% of syncs  | > 1%              |
| Offline queue depth          | < 100 per device | > 1000            |
| Metadata DB replication lag  | < 100 ms         | > 1 second        |
+------------------------------+------------------+-------------------+
```

### Dashboards
- **Sync Health:** propagation latency distribution, WebSocket connections, sync throughput
- **Storage:** utilization by tier, dedup ratio, block reference distribution
- **Upload Pipeline:** upload rates, chunk retry rates, transcoding queue depth
- **Sharing:** active shares, permission changes, link sharing usage
- **Client Health:** sync errors by platform (Windows/Mac/Linux/iOS/Android), offline durations

### Alerting Strategy
- Critical (PagerDuty): sync propagation > 30s, upload failure rate > 5%, data loss detected
- Warning (Slack): elevated conflict rates, dedup savings dropping, queue depth growing
- Informational: daily storage growth, user activity patterns, feature usage analytics

---

## 12. Interview Variations

### Common Follow-up Questions

**Q: How would you handle a user with 1 million files in a single folder?**
A: Paginate folder listings with cursor-based pagination. Use database index on
(parent_folder_id, file_name) for efficient listing. Cache folder contents in Redis
with incremental updates. Client should lazy-load folder contents as user scrolls.
Consider server-side sorting and filtering to reduce client-side processing.

**Q: How would you implement real-time collaborative editing?**
A: For documents, use Operational Transform (OT) or CRDT (Conflict-free Replicated Data
Types). Each keystroke is an operation sent to a central coordination service. Operations
are transformed against concurrent operations to maintain consistency. This is a separate
system from file sync — it operates at the character/operation level rather than block level.

**Q: How do you handle files shared with thousands of users?**
A: For widely shared files, use indirect permission model. Create a "share group" with
a single permission entry pointing to a membership list. For notifications, use fan-out-on-read
instead of fan-out-on-write (query share membership at read time rather than pre-computing
notification lists for thousands of users).

**Q: How do you prevent abuse (e.g., users uploading illegal content)?**
A: Scan uploaded content with automated detection systems (hash matching against known
bad content databases like PhotoDNA). Apply rate limits per user on upload volume. Monitor
for unusual patterns (rapid upload of many files). Implement reporting mechanism for shared
content. Legal compliance team reviews flagged content.

**Q: How would you implement search across file contents?**
A: Extract text from documents during upload (OCR for images, text extraction for PDFs).
Index content in Elasticsearch. Search query runs against both filename index and content
index. Rank results by relevance (content match > filename match > metadata match).
Apply permission filters — only return results the user has access to view.

**Q: How would you design the storage quota system?**
A: Track per-user storage usage in a dedicated counter table (updated atomically on
upload/delete). Quota check at upload initiation (reject if would exceed). Count original
file sizes (not deduplicated sizes) for user-facing quota. Shared files count against
owner's quota only. Version history has separate quota or time-based retention policy.

**Q: How do you handle very large files (10+ GB)?**
A: Use larger chunk sizes (64 MB) for very large files. Support parallel chunk uploads
(8 concurrent streams). Implement server-side assembly to avoid re-uploading on network
interruption. Consider direct S3 multipart upload with pre-signed URLs to bypass
application servers entirely. Provide progress reporting via polling endpoint.
