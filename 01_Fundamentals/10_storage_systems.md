# Storage Systems

## Overview

Understanding storage systems is essential for designing systems that handle data at scale.

## Storage Types

### Block Storage

Raw storage blocks, like a virtual hard drive.

```
┌─────────────────────────────────────┐
│           Block Device              │
├────┬────┬────┬────┬────┬────┬────┬──┤
│ B0 │ B1 │ B2 │ B3 │ B4 │ B5 │ B6 │..│
└────┴────┴────┴────┴────┴────┴────┴──┘

Direct read/write to blocks
OS manages file system on top
```

**Characteristics:**
- Low latency
- Direct access
- No built-in sharing
- Fixed size allocation

**Use Cases:**
- Database storage
- Boot volumes
- High-performance apps

**Examples:** AWS EBS, Azure Disk, SAN

### File Storage

Hierarchical file system with directories.

```
/
├── home/
│   └── user/
│       └── documents/
│           └── file.txt
└── var/
    └── logs/
        └── app.log
```

**Characteristics:**
- POSIX compatible
- Easy sharing (NFS, SMB)
- Metadata overhead
- Good for structured access

**Use Cases:**
- Shared file systems
- Media processing
- Legacy applications

**Examples:** AWS EFS, Azure Files, NFS

### Object Storage

Flat structure with unique keys.

```
Bucket: my-bucket
├── images/photo1.jpg
├── images/photo2.jpg
├── videos/video1.mp4
└── documents/report.pdf

Each object: Data + Metadata + Unique ID
```

**Characteristics:**
- Unlimited scale
- HTTP access (REST API)
- Rich metadata
- No partial updates (replace entire object)
- Eventually consistent (usually)

**Use Cases:**
- Static content (images, videos)
- Backups
- Data lakes
- CDN origin

**Examples:** AWS S3, Google Cloud Storage, Azure Blob

## Comparison

| Aspect | Block | File | Object |
|--------|-------|------|--------|
| Access | Direct blocks | File path | HTTP/API |
| Protocol | SCSI, iSCSI | NFS, SMB | REST API |
| Scale | TB | TB-PB | Unlimited |
| Latency | Lowest | Low | Higher |
| Sharing | No | Yes | Yes (read) |
| Updates | In-place | In-place | Replace |
| Cost | Highest | Medium | Lowest |

## Distributed File Systems

### HDFS (Hadoop Distributed File System)

For big data processing.

```
┌─────────────────────────────────────────────────┐
│                  NameNode                        │
│         (Metadata, file→block mapping)          │
└───────────────────┬─────────────────────────────┘
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
┌────────┐    ┌────────┐     ┌────────┐
│DataNode│    │DataNode│     │DataNode│
│ Blocks │    │ Blocks │     │ Blocks │
└────────┘    └────────┘     └────────┘

Files split into blocks (128MB default)
Blocks replicated (3x default)
```

**Characteristics:**
- Write once, read many
- Optimized for large files
- High throughput
- Not for low-latency access

### GFS (Google File System) / Colossus

Google's distributed file system.

**Key Features:**
- Single master (metadata)
- Large chunk sizes (64MB)
- Chunk replication
- Atomic append operations

## Database Storage Engines

### B-Tree Based

Traditional database storage (MySQL InnoDB, PostgreSQL).

```
         ┌─────────────────┐
         │   Root Node     │
         │  [P1|30|P2|70|P3]│
         └────┬────┬────┬──┘
              │    │    │
    ┌─────────┘    │    └─────────┐
    ▼              ▼              ▼
┌────────┐   ┌────────┐    ┌────────┐
│ [10,20]│   │ [40,50]│    │ [80,90]│
└────────┘   └────────┘    └────────┘

Balanced tree, O(log n) lookups
Good for both reads and writes
```

### LSM-Tree Based

Log-structured merge-tree (Cassandra, RocksDB, LevelDB).

```
Write Path:
Write → MemTable (RAM) → Flush → SSTable (Disk)

┌──────────────┐
│   MemTable   │ ← Writes go here
└──────┬───────┘
       │ Flush
       ▼
┌──────────────┐
│  SSTable L0  │
└──────┬───────┘
       │ Compaction
       ▼
┌──────────────┐
│  SSTable L1  │
└──────────────┘

Read: Check MemTable → L0 → L1 → ... (use bloom filters)
```

**Pros:** Very fast writes
**Cons:** Read amplification, compaction overhead

### Comparison

| Aspect | B-Tree | LSM-Tree |
|--------|--------|----------|
| Write | O(log n) | O(1) amortized |
| Read | O(log n) | O(log n) + levels |
| Space | More overhead | More efficient |
| Best for | Balanced workload | Write-heavy |

## Replication Strategies

### Synchronous Replication

```
Client → Primary → Replicas (wait for ACK) → Response

+ Strong consistency
- Higher latency
- Reduced availability
```

### Asynchronous Replication

```
Client → Primary → Response
                 ↓
              Replicas (async)

+ Lower latency
+ Higher availability
- Potential data loss
- Eventual consistency
```

### Semi-Synchronous

```
Client → Primary → At least 1 replica ACK → Response
                ↓
           Other replicas (async)

Balance of consistency and performance
```

## Erasure Coding vs Replication

### Replication (3x)

```
Data: [A][B][C]

Node 1: [A][B][C]
Node 2: [A][B][C]
Node 3: [A][B][C]

Storage overhead: 200%
Fault tolerance: 2 nodes
```

### Erasure Coding (RS 6,3)

```
Data: [A][B][C][D][E][F]
Parity: [P1][P2][P3]

9 chunks across 9 nodes
Any 6 can reconstruct original

Storage overhead: 50%
Fault tolerance: 3 nodes
```

**Trade-offs:**
- Erasure coding: Lower storage cost, higher CPU for reconstruction
- Replication: Simpler, faster recovery

## Tiered Storage

```
┌─────────────────────────────────────────────┐
│              Hot Tier (SSD)                  │
│         Frequently accessed data             │
│            Lowest latency                    │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│              Warm Tier (HDD)                 │
│         Occasionally accessed data           │
│            Medium latency                    │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│              Cold Tier (Archive)             │
│         Rarely accessed data                 │
│         High latency, lowest cost            │
└─────────────────────────────────────────────┘

Automatic tiering based on access patterns
```

## Data Durability

```
Durability = 1 - P(data loss)

AWS S3: 99.999999999% (11 9s)
= 1 in 10 billion chance of losing an object per year

Achieved through:
- Multiple copies
- Geographic distribution
- Checksums
- Automatic repair
```

## Storage Performance Metrics

| Metric | Description | Typical Values |
|--------|-------------|----------------|
| IOPS | Operations per second | SSD: 10K-100K, HDD: 100-200 |
| Throughput | MB/s | SSD: 500MB-7GB/s, HDD: 100-200MB/s |
| Latency | Time per operation | SSD: <1ms, HDD: 5-10ms |

## Cloud Storage Classes

### AWS S3 Example

| Class | Use Case | Retrieval | Cost |
|-------|----------|-----------|------|
| Standard | Frequently accessed | Immediate | $$$ |
| IA (Infrequent Access) | Monthly access | Immediate | $$ |
| Glacier | Archival | Minutes to hours | $ |
| Glacier Deep Archive | Long-term archive | 12+ hours | ¢ |

## Interview Talking Points

1. "Choose storage type based on access patterns and scale needs"
2. "Object storage for unstructured data at scale"
3. "LSM-trees for write-heavy workloads"
4. "Erasure coding reduces storage costs but increases CPU"
5. "Tiered storage optimizes cost vs performance"

## Common Interview Questions

1. **Q: When would you use object storage vs block storage?**
   A: Object storage for large unstructured data at scale (images, videos, backups). Block storage for databases and applications requiring low latency.

2. **Q: How does HDFS handle failures?**
   A: Blocks are replicated (3x default) across nodes. NameNode tracks block locations. Failed blocks are re-replicated automatically.

3. **Q: B-Tree vs LSM-Tree?**
   A: B-Tree for balanced read/write, LSM-Tree for write-heavy workloads. LSM has write amplification benefit but read amplification cost.

4. **Q: How do you design for high durability?**
   A: Multiple copies, geographic distribution, checksums for integrity, automatic repair processes.

## Key Takeaways

- Match storage type to access patterns
- Understand trade-offs between consistency and performance
- Consider cost optimization with tiered storage
- Plan for failure with replication or erasure coding
- Monitor storage performance metrics

## Further Reading

- "Designing Data-Intensive Applications" Chapters 3-4
- Google File System paper
- Amazon Dynamo paper
- HDFS architecture guide
