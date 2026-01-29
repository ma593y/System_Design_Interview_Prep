# Indexing

## Overview

Indexes are data structures that improve query speed at the cost of write performance and storage.

```
Without index (full scan):
┌──────────────────────────────────────┐
│ Row1 │ Row2 │ Row3 │ ... │ Row1M     │
└──────────────────────────────────────┘
  Scan every row: O(n)

With index:
┌─────────────┐
│   Index     │ ──▶ Direct pointer to Row47593
└─────────────┘
  Lookup: O(log n) or O(1)
```

## Index Types

### B-Tree Index

Most common database index. Balanced tree structure.

```
                    ┌─────────────────┐
                    │   [30, 60, 90]  │
                    └────┬───┬───┬────┘
                         │   │   │
           ┌─────────────┘   │   └─────────────┐
           ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │[10,20,25]│      │[40,50,55]│      │[70,80,85]│
    └──────────┘      └──────────┘      └──────────┘
         │                 │                 │
    ┌────┴────┐       ┌────┴────┐       ┌────┴────┐
    ▼    ▼    ▼       ▼    ▼    ▼       ▼    ▼    ▼
  Leaf nodes with actual data pointers

Height = O(log n), typically 3-4 levels for millions of rows
```

**Operations:**
- Lookup: O(log n)
- Range scan: O(log n + k) where k = result size
- Insert: O(log n)
- Delete: O(log n)

**Best for:**
- Equality queries (=)
- Range queries (<, >, BETWEEN)
- Sorting (ORDER BY)
- Prefix searches (LIKE 'abc%')

### Hash Index

Direct lookup using hash function.

```
hash(key) → bucket → value

┌────────────────────────────────────┐
│ Bucket 0: key1→val1, key5→val5     │
│ Bucket 1: key2→val2                │
│ Bucket 2: key3→val3, key7→val7     │
│ Bucket 3: key4→val4                │
└────────────────────────────────────┘
```

**Operations:**
- Lookup: O(1) average
- Insert: O(1) average
- No range queries

**Best for:**
- Exact match queries only
- In-memory stores (Redis, Memcached)

### LSM-Tree Index

Log-Structured Merge-tree. Optimized for writes.

```
Write path:
Write ──▶ MemTable (RAM) ──flush──▶ SSTable (Disk)
                                         │
                                    Compaction
                                         │
                                    ┌────┴────┐
                                    ▼         ▼
                              SSTable L0  SSTable L0
                                    │         │
                                 Merge/Compact
                                    │
                                    ▼
                              SSTable L1 (larger, fewer)
```

**Operations:**
- Write: O(1) - just append
- Read: O(log n) - check multiple levels
- Compaction: Background merging

**Best for:**
- Write-heavy workloads
- Append-heavy data (logs, time-series)

**Examples:** RocksDB, LevelDB, Cassandra

### B-Tree vs LSM-Tree

| Aspect | B-Tree | LSM-Tree |
|--------|--------|----------|
| Write | O(log n), in-place | O(1), append-only |
| Read | O(log n), single location | O(log n), multiple levels |
| Space | Fragmentation possible | Space amplification |
| Write amplification | Lower | Higher (compaction) |
| Read amplification | Lower | Higher (multiple levels) |
| Best for | Read-heavy, OLTP | Write-heavy |

## Specialized Indexes

### Full-Text Index (Inverted Index)

For text search.

```
Documents:
Doc1: "the quick brown fox"
Doc2: "the lazy dog"
Doc3: "quick fox jumps"

Inverted Index:
┌────────────┬─────────────┐
│   Term     │  Documents  │
├────────────┼─────────────┤
│ brown      │ [Doc1]      │
│ dog        │ [Doc2]      │
│ fox        │ [Doc1, Doc3]│
│ jumps      │ [Doc3]      │
│ lazy       │ [Doc2]      │
│ quick      │ [Doc1, Doc3]│
│ the        │ [Doc1, Doc2]│
└────────────┴─────────────┘

Query "quick fox" → Intersection: [Doc1, Doc3]
```

**Features:**
- Tokenization
- Stemming (running → run)
- Stop words removal
- TF-IDF scoring
- Fuzzy matching

**Examples:** Elasticsearch, Solr, PostgreSQL FTS

### Geospatial Index

For location-based queries.

```
R-Tree: Hierarchical bounding boxes

┌──────────────────────────────┐
│ ┌──────────┐  ┌───────────┐ │
│ │ ●      ● │  │   ●       │ │
│ │   ●      │  │      ●    │ │
│ └──────────┘  └───────────┘ │
└──────────────────────────────┘

Query: Find points in region
- Quickly eliminate regions that don't intersect
```

**Types:**
- R-Tree: General spatial data
- Quadtree: 2D space partitioning
- Geohash: String encoding of location
- S2: Google's spherical geometry

### Bitmap Index

For columns with few distinct values.

```
Gender column (M/F):
Row:     1  2  3  4  5  6  7  8
Value:   M  F  M  M  F  M  F  M

Bitmap:
Male:    1  0  1  1  0  1  0  1
Female:  0  1  0  0  1  0  1  0

Query (Male AND Age>30):
Male:    1  0  1  1  0  1  0  1
Age>30:  0  1  1  0  1  1  0  0
AND:     0  0  1  0  0  1  0  0
```

**Best for:**
- Low cardinality columns
- Data warehousing
- Analytical queries

## Composite Index

Index on multiple columns.

```sql
CREATE INDEX idx_name_age ON users(last_name, first_name, age);
```

```
Index structure:
┌────────────────────────────────────┐
│ (Adams, Alice, 25) → Row pointer   │
│ (Adams, Bob, 30) → Row pointer     │
│ (Brown, Carol, 28) → Row pointer   │
│ (Brown, Carol, 35) → Row pointer   │
└────────────────────────────────────┘

Leftmost prefix rule:
✓ WHERE last_name = 'Adams'
✓ WHERE last_name = 'Adams' AND first_name = 'Alice'
✓ WHERE last_name = 'Adams' AND first_name = 'Alice' AND age = 25
✗ WHERE first_name = 'Alice' (doesn't use index)
✗ WHERE age = 25 (doesn't use index)
```

### Column Order Matters

```sql
-- Index: (a, b, c)

-- Uses full index
WHERE a = 1 AND b = 2 AND c = 3

-- Uses index for a, b
WHERE a = 1 AND b = 2

-- Uses index for a only
WHERE a = 1 AND c = 3  -- Skip b, can only use a

-- Cannot use index
WHERE b = 2 AND c = 3  -- Missing leftmost column
```

## Covering Index

Index includes all columns needed by query.

```sql
-- Query
SELECT name, email FROM users WHERE id = 123;

-- Regular index (requires table lookup)
CREATE INDEX idx_id ON users(id);
-- Index lookup → Row pointer → Table lookup for name, email

-- Covering index (no table lookup)
CREATE INDEX idx_id_covering ON users(id) INCLUDE (name, email);
-- Index lookup → Return name, email directly
```

## Index Selection Strategy

### When to Create Indexes

| Good Candidates | Poor Candidates |
|-----------------|-----------------|
| Primary key columns | Small tables |
| Foreign key columns | Frequently updated columns |
| WHERE clause columns | Low-selectivity columns |
| JOIN columns | Wide columns (large strings) |
| ORDER BY columns | Tables with many writes |

### Index Selectivity

```
Selectivity = Distinct values / Total rows

High selectivity (good for index):
- user_id: 1,000,000 distinct / 1,000,000 rows = 1.0

Low selectivity (poor for index):
- gender: 2 distinct / 1,000,000 rows = 0.000002
- status: 5 distinct / 1,000,000 rows = 0.000005
```

## Index Maintenance

### Fragmentation

```
Over time, inserts/deletes cause fragmentation:

┌───┬───┬───┬───┬───┐     ┌───┬───┬▓▓▓┬───┬▓▓▓┐
│ A │ B │ C │ D │ E │ ──▶ │ A │ C │░░░│ E │░░░│
└───┴───┴───┴───┴───┘     └───┴───┴───┴───┴───┘
                                   Gaps (fragmentation)

Solutions:
- REINDEX / REBUILD
- Online reorganization
- Fill factor (leave space for growth)
```

### Index Overhead

```
Storage: Each index stores copies of indexed columns
Write cost: Each INSERT/UPDATE/DELETE must update all indexes

Trade-off:
More indexes → Faster reads, slower writes
Fewer indexes → Slower reads, faster writes
```

## Interview Talking Points

1. "B-tree for balanced read/write, LSM for write-heavy workloads"
2. "Composite index order matters - leftmost prefix rule"
3. "Covering indexes avoid table lookups"
4. "High selectivity makes indexes more effective"
5. "Every index has write overhead"

## Common Interview Questions

1. **Q: When would you use a hash index vs B-tree?**
   A: Hash for exact match only (faster O(1)). B-tree for range queries, sorting, and prefix searches.

2. **Q: How does a full-text search index work?**
   A: Inverted index maps terms to documents. Includes tokenization, stemming, and scoring (TF-IDF).

3. **Q: Why does column order matter in composite indexes?**
   A: Indexes are sorted by columns left to right. Can only use prefix of index columns.

4. **Q: How do you decide what to index?**
   A: Analyze query patterns, check selectivity, consider write overhead. Index columns in WHERE, JOIN, ORDER BY.

## Key Takeaways

- Indexes trade write performance for read performance
- B-tree is the default choice for most databases
- LSM-tree excels at write-heavy workloads
- Composite index column order is critical
- Monitor and maintain indexes regularly

## Further Reading

- "Database Internals" by Alex Petrov
- "Use The Index, Luke" (online resource)
- PostgreSQL/MySQL indexing documentation
