# Database Best Practices

## Overview

Databases are the backbone of most applications, storing and managing critical data. Proper database design and optimization are essential for building scalable, performant systems. This guide covers connection pooling, query optimization, indexing strategies, and other crucial database best practices.

## Table of Contents

1. [Connection Pooling](#connection-pooling)
2. [Query Optimization](#query-optimization)
3. [Indexing Strategies](#indexing-strategies)
4. [Schema Design](#schema-design)
5. [Transaction Management](#transaction-management)
6. [Replication and Sharding](#replication-and-sharding)
7. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
8. [Implementation Guidelines](#implementation-guidelines)
9. [Interview Talking Points](#interview-talking-points)
10. [Key Takeaways](#key-takeaways)

---

## Connection Pooling

### Why Connection Pooling Matters

Creating database connections is expensive:
- TCP handshake overhead
- Authentication negotiation
- Memory allocation on both client and server
- Connection setup can take 20-50ms

### Connection Pool Architecture

```
Application Servers          Connection Pool          Database
    ┌─────────┐                                      ┌─────────┐
    │ Request │───┐                                  │         │
    └─────────┘   │      ┌─────────────────┐         │         │
    ┌─────────┐   │      │ ┌─────┐ ┌─────┐ │         │         │
    │ Request │───┼─────►│ │Conn │ │Conn │ │────────►│   DB    │
    └─────────┘   │      │ └─────┘ └─────┘ │         │         │
    ┌─────────┐   │      │ ┌─────┐ ┌─────┐ │         │         │
    │ Request │───┘      │ │Conn │ │Conn │ │         │         │
    └─────────┘          │ └─────┘ └─────┘ │         │         │
                         └─────────────────┘         └─────────┘
                             Pool Size: 4
```

### Connection Pool Configuration

```python
# Python with SQLAlchemy
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:password@localhost/db',
    poolclass=QueuePool,
    pool_size=10,           # Number of persistent connections
    max_overflow=20,        # Additional connections when pool is full
    pool_timeout=30,        # Seconds to wait for available connection
    pool_recycle=1800,      # Recycle connections after 30 minutes
    pool_pre_ping=True      # Verify connection health before use
)
```

```java
// Java with HikariCP (recommended for Java)
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost/db");
config.setUsername("user");
config.setPassword("password");

// Pool sizing
config.setMinimumIdle(5);           // Minimum idle connections
config.setMaximumPoolSize(20);      // Maximum pool size
config.setIdleTimeout(600000);      // 10 minutes idle timeout
config.setMaxLifetime(1800000);     // 30 minutes max lifetime
config.setConnectionTimeout(30000); // 30 seconds connection timeout

// Health checks
config.setConnectionTestQuery("SELECT 1");
config.setValidationTimeout(5000);

HikariDataSource dataSource = new HikariDataSource(config);
```

### Pool Sizing Formula

```
Optimal Pool Size = (Core Count * 2) + Effective Spindle Count

For SSD storage:
Pool Size = Core Count * 2 + 1

Example:
- 4 core CPU with SSD
- Pool Size = (4 * 2) + 1 = 9 connections
```

### Connection Pool Monitoring

```python
class ConnectionPoolMonitor:
    def __init__(self, pool):
        self.pool = pool

    def get_stats(self):
        return {
            'pool_size': self.pool.size(),
            'checked_out': self.pool.checkedout(),
            'overflow': self.pool.overflow(),
            'checked_in': self.pool.checkedin(),
            'invalidated': self.pool.invalidated()
        }

    def log_stats(self):
        stats = self.get_stats()
        logger.info(f"Pool Stats: "
                   f"size={stats['pool_size']}, "
                   f"active={stats['checked_out']}, "
                   f"available={stats['checked_in']}")
```

### External Connection Poolers

For high-scale applications, consider external poolers:

#### PgBouncer (PostgreSQL)

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool settings
pool_mode = transaction        # session, transaction, or statement
default_pool_size = 20
max_client_conn = 1000
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
```

---

## Query Optimization

### Understanding EXPLAIN Plans

```sql
-- PostgreSQL EXPLAIN
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name
ORDER BY order_count DESC
LIMIT 10;

-- Output interpretation
/*
Limit  (cost=1234.56..1234.78 rows=10 width=48) (actual time=45.123..45.130 rows=10 loops=1)
  ->  Sort  (cost=1234.56..1256.78 rows=8888 width=48) (actual time=45.121..45.125 rows=10 loops=1)
        Sort Key: (count(o.id)) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  HashAggregate  (cost=1000.00..1088.88 rows=8888 width=48) (actual time=40.000..43.000 rows=8500 loops=1)
              Group Key: u.id
              ->  Hash Left Join  (cost=100.00..900.00 rows=10000 width=40) (actual time=5.000..30.000 rows=10000 loops=1)
                    Hash Cond: (u.id = o.user_id)
                    ->  Seq Scan on users u  (cost=0.00..500.00 rows=8888 width=36) (actual time=0.010..10.000 rows=8500 loops=1)
                          Filter: (created_at > '2024-01-01'::date)
                          Rows Removed by Filter: 1500
                    ->  Hash  (cost=50.00..50.00 rows=4000 width=8) (actual time=4.000..4.000 rows=4000 loops=1)
                          Buckets: 4096  Batches: 1  Memory Usage: 189kB
                          ->  Seq Scan on orders o  (cost=0.00..50.00 rows=4000 width=8) (actual time=0.005..2.000 rows=4000 loops=1)
Planning Time: 0.500 ms
Execution Time: 45.500 ms
*/
```

### Query Optimization Techniques

#### 1. Avoid SELECT *

```sql
-- Bad: Fetches all columns
SELECT * FROM users WHERE id = 1;

-- Good: Only fetch needed columns
SELECT id, name, email FROM users WHERE id = 1;
```

#### 2. Use Proper JOINs

```sql
-- Bad: Implicit join (hard to read, error-prone)
SELECT u.name, o.total
FROM users u, orders o
WHERE u.id = o.user_id;

-- Good: Explicit JOIN
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

#### 3. Optimize Subqueries

```sql
-- Bad: Correlated subquery (N+1 problem)
SELECT u.name,
       (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
FROM users u;

-- Good: JOIN with aggregation
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

#### 4. Use EXISTS Instead of IN for Large Sets

```sql
-- Slower with large subsets
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 100);

-- Faster with EXISTS
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 100
);
```

#### 5. Batch Operations

```python
# Bad: Individual inserts
for user in users:
    cursor.execute("INSERT INTO users (name, email) VALUES (%s, %s)",
                   (user['name'], user['email']))

# Good: Batch insert
cursor.executemany(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    [(u['name'], u['email']) for u in users]
)

# Even better: COPY or bulk insert
from io import StringIO
import csv

buffer = StringIO()
writer = csv.writer(buffer)
for user in users:
    writer.writerow([user['name'], user['email']])

buffer.seek(0)
cursor.copy_from(buffer, 'users', columns=('name', 'email'), sep=',')
```

### N+1 Query Problem

```python
# Bad: N+1 queries
users = User.query.all()  # 1 query
for user in users:
    orders = user.orders  # N queries (one per user)

# Good: Eager loading
users = User.query.options(
    joinedload(User.orders)
).all()  # 1 or 2 queries total

# Or with explicit join
users = db.session.query(User, Order)\
    .outerjoin(Order)\
    .all()
```

---

## Indexing Strategies

### Types of Indexes

#### 1. B-Tree Index (Default)

```sql
-- Best for: equality and range queries
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created ON orders(created_at);

-- Supports these operations efficiently:
-- =, <, >, <=, >=, BETWEEN, IN, IS NULL
-- LIKE 'prefix%' (but NOT '%suffix')
```

#### 2. Hash Index

```sql
-- Best for: exact equality comparisons only
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- Only supports: =
-- Faster for equality but no range queries
```

#### 3. GiST Index (Generalized Search Tree)

```sql
-- Best for: geometric data, full-text search, range types
CREATE INDEX idx_locations_point ON locations USING gist(coordinates);
CREATE INDEX idx_events_timerange ON events USING gist(time_range);
```

#### 4. GIN Index (Generalized Inverted Index)

```sql
-- Best for: array containment, full-text search, JSONB
CREATE INDEX idx_posts_tags ON posts USING gin(tags);
CREATE INDEX idx_docs_content ON documents USING gin(to_tsvector('english', content));
CREATE INDEX idx_data_jsonb ON data_table USING gin(metadata jsonb_path_ops);
```

### Composite Indexes

```sql
-- Order matters! Left-to-right prefix matching
CREATE INDEX idx_orders_composite ON orders(user_id, status, created_at);

-- This index supports:
-- WHERE user_id = ?                          -- Uses index
-- WHERE user_id = ? AND status = ?           -- Uses index
-- WHERE user_id = ? AND status = ? AND created_at > ?  -- Uses index
-- WHERE status = ?                           -- Does NOT use index!
-- WHERE created_at > ?                       -- Does NOT use index!
```

### Partial Indexes

```sql
-- Index only relevant subset of data
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- Much smaller index, faster updates
-- Only used when query includes WHERE status = 'pending'
```

### Covering Indexes

```sql
-- Include extra columns to avoid table lookup
CREATE INDEX idx_users_email_include ON users(email) INCLUDE (name, created_at);

-- Query satisfied entirely from index
SELECT name, email, created_at FROM users WHERE email = 'test@example.com';
```

### Index Maintenance

```sql
-- Check index usage (PostgreSQL)
SELECT
    schemaname,
    relname as table_name,
    indexrelname as index_name,
    idx_scan as times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- Find unused indexes
SELECT
    schemaname || '.' || relname AS table,
    indexrelname AS index,
    pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size
FROM pg_stat_user_indexes ui
JOIN pg_index i ON ui.indexrelid = i.indexrelid
WHERE NOT indisunique
  AND idx_scan = 0
  AND pg_relation_size(relid) > 5 * 8192;

-- Rebuild bloated indexes
REINDEX INDEX CONCURRENTLY idx_users_email;
```

### Indexing Decision Framework

```
Should I create an index?

1. Query Frequency
   - High frequency queries benefit most
   - Rare queries may not justify index overhead

2. Table Size
   - Small tables (<1000 rows) often faster with seq scan
   - Large tables benefit significantly

3. Write vs Read Ratio
   - Write-heavy: fewer indexes
   - Read-heavy: more indexes acceptable

4. Selectivity
   - High selectivity (few matches): good for index
   - Low selectivity (many matches): may not help

5. Column Cardinality
   - High cardinality (unique values): good for index
   - Low cardinality (few distinct values): bitmap index or partial index
```

---

## Schema Design

### Normalization vs Denormalization

```sql
-- Normalized (3NF) - Better for write-heavy workloads
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255) UNIQUE
);

CREATE TABLE addresses (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    street VARCHAR(255),
    city VARCHAR(100),
    country VARCHAR(100)
);

-- Denormalized - Better for read-heavy workloads
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    -- Embedded address
    address_street VARCHAR(255),
    address_city VARCHAR(100),
    address_country VARCHAR(100),
    -- Computed/cached fields
    order_count INTEGER DEFAULT 0,
    total_spent DECIMAL(10,2) DEFAULT 0
);
```

### Data Types Best Practices

```sql
-- Use appropriate types
CREATE TABLE products (
    -- Use UUID for distributed systems
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,

    -- Use VARCHAR with reasonable limits
    name VARCHAR(255) NOT NULL,
    description TEXT,  -- Use TEXT for unlimited length

    -- Use DECIMAL for money (never FLOAT!)
    price DECIMAL(10, 2) NOT NULL,

    -- Use appropriate integer sizes
    quantity SMALLINT,      -- -32768 to 32767
    view_count INTEGER,     -- -2B to 2B
    global_rank BIGINT,     -- Very large numbers

    -- Use BOOLEAN for flags
    is_active BOOLEAN DEFAULT true,

    -- Use TIMESTAMPTZ for timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    -- Use JSONB for flexible data
    metadata JSONB DEFAULT '{}'
);
```

### Soft Deletes

```sql
-- Soft delete pattern
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    deleted_at TIMESTAMPTZ DEFAULT NULL,

    -- Unique constraint that allows multiple deleted records
    CONSTRAINT unique_active_email
        UNIQUE (email) WHERE deleted_at IS NULL
);

-- Partial index for active records
CREATE INDEX idx_users_active ON users(email)
WHERE deleted_at IS NULL;

-- Query active users
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Audit Trails

```sql
-- Audit table pattern
CREATE TABLE user_audit (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    action VARCHAR(10) NOT NULL, -- INSERT, UPDATE, DELETE
    old_data JSONB,
    new_data JSONB,
    changed_by INTEGER,
    changed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger function
CREATE OR REPLACE FUNCTION audit_user_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO user_audit (user_id, action, new_data, changed_by)
        VALUES (NEW.id, 'INSERT', to_jsonb(NEW), current_setting('app.user_id')::int);
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO user_audit (user_id, action, old_data, new_data, changed_by)
        VALUES (NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_setting('app.user_id')::int);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO user_audit (user_id, action, old_data, changed_by)
        VALUES (OLD.id, 'DELETE', to_jsonb(OLD), current_setting('app.user_id')::int);
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_user_changes();
```

---

## Transaction Management

### ACID Properties

```
A - Atomicity:   All operations succeed or all fail
C - Consistency: Database remains in valid state
I - Isolation:   Concurrent transactions don't interfere
D - Durability:  Committed data survives failures
```

### Isolation Levels

```sql
-- Read Uncommitted: Can see uncommitted changes (dirty reads)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Read Committed: Only see committed changes (PostgreSQL default)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Repeatable Read: Same query returns same results within transaction
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Serializable: Full isolation, transactions appear sequential
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### Transaction Best Practices

```python
# Keep transactions short
def transfer_money(from_account: int, to_account: int, amount: Decimal):
    with db.session.begin():
        # Acquire locks in consistent order to prevent deadlocks
        accounts = Account.query\
            .filter(Account.id.in_([from_account, to_account]))\
            .order_by(Account.id)\
            .with_for_update()\
            .all()

        from_acc = next(a for a in accounts if a.id == from_account)
        to_acc = next(a for a in accounts if a.id == to_account)

        if from_acc.balance < amount:
            raise InsufficientFundsError()

        from_acc.balance -= amount
        to_acc.balance += amount
        # Transaction commits automatically on exit

# Handle retries for serialization failures
from tenacity import retry, retry_if_exception_type

@retry(retry=retry_if_exception_type(SerializationError), stop=stop_after_attempt(3))
def process_order(order_id: int):
    with db.session.begin():
        order = Order.query.get(order_id)
        # Process order...
```

### Optimistic vs Pessimistic Locking

```python
# Optimistic locking (version-based)
class Product(Base):
    id = Column(Integer, primary_key=True)
    name = Column(String)
    quantity = Column(Integer)
    version = Column(Integer, default=1)

    __mapper_args__ = {"version_id_col": version}

def update_product_optimistic(product_id: int, new_quantity: int):
    try:
        product = Product.query.get(product_id)
        product.quantity = new_quantity
        db.session.commit()  # Fails if version changed
    except StaleDataError:
        db.session.rollback()
        raise ConcurrentModificationError()

# Pessimistic locking (SELECT FOR UPDATE)
def update_product_pessimistic(product_id: int, new_quantity: int):
    with db.session.begin():
        product = Product.query\
            .filter_by(id=product_id)\
            .with_for_update()\
            .first()  # Blocks until lock acquired

        product.quantity = new_quantity
```

---

## Replication and Sharding

### Read Replicas

```python
# Route queries to appropriate database
class DatabaseRouter:
    def __init__(self, primary_url: str, replica_urls: List[str]):
        self.primary = create_engine(primary_url)
        self.replicas = [create_engine(url) for url in replica_urls]
        self.replica_index = 0

    def get_engine(self, readonly: bool = False) -> Engine:
        if readonly and self.replicas:
            # Round-robin replica selection
            engine = self.replicas[self.replica_index]
            self.replica_index = (self.replica_index + 1) % len(self.replicas)
            return engine
        return self.primary

# Usage
@contextmanager
def get_session(readonly: bool = False):
    engine = router.get_engine(readonly=readonly)
    session = Session(engine)
    try:
        yield session
    finally:
        session.close()

# Read from replica
with get_session(readonly=True) as session:
    users = session.query(User).all()

# Write to primary
with get_session(readonly=False) as session:
    session.add(User(name='John'))
    session.commit()
```

### Sharding Strategies

```python
# Hash-based sharding
def get_shard(user_id: int, num_shards: int) -> int:
    return hash(user_id) % num_shards

# Range-based sharding
def get_shard_by_date(created_at: datetime) -> str:
    year = created_at.year
    month = created_at.month
    return f"orders_{year}_{month:02d}"

# Directory-based sharding
class ShardDirectory:
    def __init__(self, redis_client):
        self.redis = redis_client

    def get_shard(self, entity_id: str) -> str:
        shard = self.redis.hget('shard_map', entity_id)
        if not shard:
            shard = self.assign_shard(entity_id)
        return shard

    def assign_shard(self, entity_id: str) -> str:
        # Find shard with least data
        shard_sizes = self.get_shard_sizes()
        shard = min(shard_sizes, key=shard_sizes.get)
        self.redis.hset('shard_map', entity_id, shard)
        return shard
```

### Cross-Shard Queries

```python
# Scatter-gather pattern
async def search_all_shards(query: str) -> List[Result]:
    shards = get_all_shard_connections()

    async def search_shard(shard):
        return await shard.execute(
            "SELECT * FROM products WHERE name ILIKE %s",
            (f'%{query}%',)
        )

    # Execute on all shards in parallel
    results = await asyncio.gather(*[
        search_shard(shard) for shard in shards
    ])

    # Merge and sort results
    merged = list(chain.from_iterable(results))
    return sorted(merged, key=lambda x: x.relevance, reverse=True)[:100]
```

---

## Common Mistakes to Avoid

### 1. Not Using Parameterized Queries (SQL Injection)

```python
# DANGEROUS: SQL Injection vulnerable
query = f"SELECT * FROM users WHERE email = '{email}'"

# SAFE: Parameterized query
query = "SELECT * FROM users WHERE email = %s"
cursor.execute(query, (email,))
```

### 2. Missing Indexes on Foreign Keys

```sql
-- Foreign key without index
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id)  -- No index!
);

-- JOINs will be slow without index
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### 3. Over-Indexing

```sql
-- Too many indexes hurt write performance
-- Each INSERT/UPDATE must update all indexes

-- Bad: Index on every column
CREATE INDEX idx_1 ON users(name);
CREATE INDEX idx_2 ON users(email);
CREATE INDEX idx_3 ON users(created_at);
CREATE INDEX idx_4 ON users(updated_at);
CREATE INDEX idx_5 ON users(status);
CREATE INDEX idx_6 ON users(name, email);
CREATE INDEX idx_7 ON users(email, name);

-- Good: Index based on actual query patterns
CREATE INDEX idx_users_email ON users(email);  -- Login lookup
CREATE INDEX idx_users_status_created ON users(status, created_at) WHERE status = 'active';
```

### 4. Not Handling Connection Failures

```python
# Bad: No retry logic
def get_user(user_id):
    return db.session.query(User).get(user_id)

# Good: With retry and circuit breaker
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(OperationalError)
)
def get_user(user_id):
    try:
        return db.session.query(User).get(user_id)
    except OperationalError as e:
        db.session.rollback()
        raise
```

### 5. Ignoring Query Plans

```sql
-- Bad: Assuming the query is fine
SELECT * FROM orders WHERE status = 'pending';

-- Good: Check the plan regularly
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
-- Look for Seq Scan on large tables (potential missing index)
```

### 6. Storing Large Objects in Database

```python
# Bad: Storing files in database
class Document(Base):
    id = Column(Integer, primary_key=True)
    content = Column(LargeBinary)  # Entire file in DB

# Good: Store reference, file in object storage
class Document(Base):
    id = Column(Integer, primary_key=True)
    storage_path = Column(String)  # S3 path
    content_type = Column(String)
    size_bytes = Column(BigInteger)
```

---

## Implementation Guidelines

### Database Selection Checklist

```markdown
[ ] Data Model Fit
    [ ] Relational data -> PostgreSQL, MySQL
    [ ] Document data -> MongoDB, CouchDB
    [ ] Key-value -> Redis, DynamoDB
    [ ] Time-series -> TimescaleDB, InfluxDB
    [ ] Graph -> Neo4j, Amazon Neptune

[ ] Scalability Requirements
    [ ] Read-heavy -> Read replicas
    [ ] Write-heavy -> Sharding, Cassandra
    [ ] Both -> Hybrid approach

[ ] Consistency Requirements
    [ ] Strong consistency -> PostgreSQL, MySQL
    [ ] Eventual consistency -> Cassandra, DynamoDB
    [ ] Tunable -> CockroachDB

[ ] Operational Considerations
    [ ] Managed service available
    [ ] Team expertise
    [ ] Backup/recovery options
    [ ] Monitoring tools
```

### Performance Monitoring Queries

```sql
-- PostgreSQL: Long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state != 'idle';

-- Table sizes
SELECT
    schemaname,
    relname as table_name,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    pg_size_pretty(pg_relation_size(relid)) as table_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) as index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Cache hit ratio (should be >99%)
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM pg_statio_user_tables;
```

---

## Interview Talking Points

### Question: "How would you optimize a slow database query?"

**Strong Answer Framework**:

1. **Analyze**: Use EXPLAIN ANALYZE to understand the query plan
2. **Identify Bottlenecks**: Look for sequential scans, nested loops, high row estimates
3. **Index**: Add appropriate indexes based on WHERE/JOIN conditions
4. **Rewrite**: Consider query restructuring (JOINs vs subqueries)
5. **Denormalize**: If read patterns justify it
6. **Cache**: Add application-level caching for frequent queries

### Question: "How do you handle database scaling?"

**Strong Answer Framework**:

1. **Vertical Scaling**: Upgrade hardware (limited)
2. **Read Replicas**: Offload read traffic
3. **Caching Layer**: Redis/Memcached for hot data
4. **Connection Pooling**: PgBouncer for connection management
5. **Sharding**: Horizontal partitioning for write scaling
6. **Data Archival**: Move old data to cold storage

### Question: "Explain connection pooling and its importance."

**Key Points**:

1. Connection creation overhead (20-50ms)
2. Database connection limits (memory per connection)
3. Pool sizing based on cores and workload
4. Connection health checks and recycling
5. External poolers for microservices (PgBouncer)

### Common Follow-up Questions

- How do you prevent deadlocks?
- When would you choose eventual consistency?
- How do you handle schema migrations at scale?
- What's your approach to database backup and recovery?

---

## Key Takeaways

### 1. Connection Pooling is Essential
- Always use connection pools in production
- Size pools based on CPU cores and workload
- Consider external poolers for high-scale apps

### 2. Index Strategically
- Index based on actual query patterns
- Remember composite index column order
- Monitor and remove unused indexes

### 3. Optimize Queries Proactively
- Use EXPLAIN ANALYZE regularly
- Watch for N+1 queries
- Batch operations when possible

### 4. Choose Appropriate Isolation
- Understand ACID trade-offs
- Use lowest isolation level that meets requirements
- Handle serialization failures gracefully

### 5. Plan for Scale
- Design for read replicas from the start
- Consider sharding strategy early
- Separate hot and cold data

### 6. Monitor Everything
- Track slow queries
- Monitor connection pool usage
- Alert on cache hit ratio drops

---

## Quick Reference

| Aspect | Best Practice |
|--------|--------------|
| Pool Size | (CPU cores * 2) + 1 |
| Index Type | B-tree for most, GIN for JSONB/arrays |
| Isolation | Read Committed (default) |
| Locking | Optimistic for low contention |
| Batch Size | 1000-5000 rows per batch |
| Query Timeout | 30 seconds max |
| Connection Timeout | 5-10 seconds |
| Pool Recycle | 30 minutes |
