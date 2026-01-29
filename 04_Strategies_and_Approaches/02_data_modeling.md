# Data Modeling Strategies

## Overview

Data modeling is the process of designing how data is stored, organized, and accessed in a system. The right data model can dramatically improve performance, simplify queries, and enable scaling, while a poor model can create bottlenecks that are expensive to fix later. In system design interviews, demonstrating thoughtful data modeling shows you understand the trade-offs between normalization, query patterns, and system requirements.

## Key Concepts

### Data Modeling Spectrum

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Data Modeling Spectrum                            │
│                                                                          │
│  Fully Normalized              Hybrid                  Fully Denormalized│
│  (3NF/BCNF)                   Approach                 (Document/NoSQL)  │
│       │                          │                            │          │
│       ▼                          ▼                            ▼          │
│  ┌──────────┐             ┌──────────┐              ┌──────────┐        │
│  │ No Data  │             │ Strategic│              │ Maximum  │        │
│  │Redundancy│             │Redundancy│              │Redundancy│        │
│  │          │             │          │              │          │        │
│  │ Complex  │             │ Balanced │              │ Simple   │        │
│  │ Queries  │             │ Trade-off│              │ Queries  │        │
│  └──────────┘             └──────────┘              └──────────┘        │
│                                                                          │
│  Write Optimized ◄────────────────────────────────► Read Optimized      │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Data Modeling Decision Framework

```
                        ┌─────────────────────┐
                        │ Identify Access     │
                        │ Patterns            │
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │ Analyze Read/Write  │
                        │ Ratio               │
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │ Evaluate Consistency│
                        │ Requirements        │
                        └──────────┬──────────┘
                                   │
                                   ▼
              ┌────────────────────┴────────────────────┐
              │         Choose Modeling Strategy         │
              │                                          │
              │  ┌────────────┐    ┌────────────────┐  │
              │  │ Normalized │    │ Denormalized   │  │
              │  │ (Writes)   │    │ (Reads)        │  │
              │  └────────────┘    └────────────────┘  │
              └──────────────────────────────────────────┘
```

## Normalization Levels

### Normal Forms Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Normal Forms Hierarchy                            │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1NF: Atomic Values                                              │   │
│  │  - No repeating groups                                           │   │
│  │  - Each cell contains single value                               │   │
│  │  - Each row is unique                                            │   │
│  │                                                                   │   │
│  │  Example: Split "John,Jane" into separate rows                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                           │
│                              ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  2NF: Full Functional Dependency                                 │   │
│  │  - Must be in 1NF                                                │   │
│  │  - No partial dependencies on composite key                      │   │
│  │                                                                   │   │
│  │  Example: Move product_name to Products table if it only         │   │
│  │  depends on product_id, not the full composite key               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                           │
│                              ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  3NF: No Transitive Dependencies                                 │   │
│  │  - Must be in 2NF                                                │   │
│  │  - No non-key column depends on another non-key column          │   │
│  │                                                                   │   │
│  │  Example: Move city, state to separate Address table if          │   │
│  │  state depends on city, not directly on primary key              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                           │
│                              ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  BCNF: Boyce-Codd Normal Form                                    │   │
│  │  - Must be in 3NF                                                │   │
│  │  - Every determinant is a candidate key                          │   │
│  │                                                                   │   │
│  │  Stricter version of 3NF for complex key relationships           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Normalized Schema Example

```
E-Commerce Normalized Schema (3NF):

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│     Users       │     │     Orders      │     │   OrderItems    │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│ user_id (PK)    │◄────│ user_id (FK)    │     │ item_id (PK)    │
│ email           │     │ order_id (PK)   │◄────│ order_id (FK)   │
│ name            │     │ status          │     │ product_id (FK) │
│ created_at      │     │ total_amount    │     │ quantity        │
└─────────────────┘     │ created_at      │     │ unit_price      │
                        └─────────────────┘     └────────┬────────┘
                                                         │
┌─────────────────┐     ┌─────────────────┐              │
│   Categories    │     │    Products     │◄─────────────┘
├─────────────────┤     ├─────────────────┤
│ category_id (PK)│◄────│ category_id (FK)│
│ name            │     │ product_id (PK) │
│ parent_id (FK)  │     │ name            │
└─────────────────┘     │ price           │
                        │ description     │
                        └─────────────────┘

Benefits:
- No data redundancy
- Easy to update (single source of truth)
- Data integrity enforced by constraints
- Flexible for ad-hoc queries

Costs:
- Complex queries require JOINs
- Read performance can suffer at scale
- Not optimized for specific access patterns
```

## Denormalization Strategies

### When to Denormalize

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHOOSE DENORMALIZATION WHEN:                          │
│                                                                          │
│  1. Read Performance is Critical                                        │
│     └── High read-to-write ratio (10:1 or higher)                       │
│     └── Complex JOINs causing query latency                             │
│     └── Real-time response requirements                                  │
│                                                                          │
│  2. Access Patterns are Well-Known                                      │
│     └── Queries follow predictable patterns                             │
│     └── Data is read together frequently                                │
│     └── Reporting/analytics dashboards                                   │
│                                                                          │
│  3. Write Frequency is Low                                              │
│     └── Reference data that rarely changes                              │
│     └── Aggregated/computed values                                       │
│     └── Historical data (immutable)                                      │
│                                                                          │
│  4. Eventual Consistency is Acceptable                                  │
│     └── Slight staleness is tolerable                                   │
│     └── Background sync processes are feasible                          │
│     └── Users expect some delay in updates                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### Denormalization Techniques

```
Technique 1: Duplicating Columns

Before (Normalized):
┌─────────────┐        ┌─────────────┐
│   Orders    │        │   Users     │
├─────────────┤        ├─────────────┤
│ order_id    │        │ user_id     │
│ user_id ────┼───────►│ email       │
│ total       │        │ name        │
└─────────────┘        └─────────────┘

After (Denormalized):
┌─────────────┐
│   Orders    │
├─────────────┤
│ order_id    │
│ user_id     │
│ user_name   │  ◄── Duplicated for fast reads
│ user_email  │  ◄── Duplicated for fast reads
│ total       │
└─────────────┘

Trade-off: Faster reads, but must update Orders when User changes name


Technique 2: Pre-computed Aggregates

Before (Computed on read):
SELECT seller_id, COUNT(*), AVG(rating)
FROM reviews
GROUP BY seller_id;  -- Slow at scale

After (Pre-computed):
┌───────────────────┐
│  SellerStats      │
├───────────────────┤
│ seller_id         │
│ review_count      │  ◄── Updated on each review
│ average_rating    │  ◄── Updated on each review
│ last_updated      │
└───────────────────┘

Trade-off: O(1) reads, but write complexity increases


Technique 3: Materialized Views

┌─────────────────────────────────────────────────────────────────────┐
│                     Materialized View Pattern                        │
│                                                                      │
│  Source Tables              Materialized View                       │
│  ┌──────────┐              ┌────────────────────────┐              │
│  │ Orders   │              │   DailySalesSummary    │              │
│  └────┬─────┘              ├────────────────────────┤              │
│       │                    │ date                   │              │
│       │    ──Refresh──►    │ total_orders           │              │
│       │    (scheduled)     │ total_revenue          │              │
│  ┌────┴─────┐              │ avg_order_value        │              │
│  │OrderItems│              │ top_category           │              │
│  └──────────┘              └────────────────────────┘              │
│                                                                      │
│  Refresh Strategies:                                                 │
│  - Full refresh (nightly)                                           │
│  - Incremental refresh (hourly)                                     │
│  - Real-time streaming updates                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Schema Design Patterns

### Entity-Attribute-Value (EAV)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Entity-Attribute-Value Pattern                        │
│                                                                          │
│  Traditional Schema (Rigid):        EAV Schema (Flexible):              │
│  ┌──────────────────────┐          ┌──────────────────────┐            │
│  │      Products        │          │    ProductAttributes  │            │
│  ├──────────────────────┤          ├──────────────────────┤            │
│  │ id                   │          │ product_id            │            │
│  │ name                 │          │ attribute_name        │            │
│  │ color       (fixed)  │          │ attribute_value       │            │
│  │ size        (fixed)  │          └──────────────────────┘            │
│  │ weight      (fixed)  │                                               │
│  │ material    (fixed)  │          Sample Data:                        │
│  │ ...100 more columns  │          ┌──────────┬───────────┬─────────┐ │
│  └──────────────────────┘          │product_id│attr_name  │attr_value│ │
│                                     ├──────────┼───────────┼─────────┤ │
│                                     │    1     │ color     │ red     │ │
│                                     │    1     │ size      │ large   │ │
│                                     │    2     │ author    │ Smith   │ │
│                                     │    2     │ pages     │ 342     │ │
│                                     └──────────┴───────────┴─────────┘ │
│                                                                          │
│  Use When:                          Avoid When:                         │
│  - Schema changes frequently        - Attributes are well-known         │
│  - Different entities have          - Need complex queries on attrs     │
│    different attributes             - Referential integrity needed      │
│  - Marketplace with varied products - Performance is critical           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Wide Column / Column Family

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Wide Column Pattern (Cassandra-style)                 │
│                                                                          │
│  Row Key: user_123                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Column Families (grouped by access pattern)                      │   │
│  │                                                                   │   │
│  │ profile:           │ preferences:        │ activity:             │   │
│  │ ┌─────────────────┐│ ┌─────────────────┐│ ┌────────────────────┐│   │
│  │ │ name: "John"    ││ │ theme: "dark"   ││ │ 2024-01: {count:5} ││   │
│  │ │ email: "j@..."  ││ │ lang: "en"      ││ │ 2024-02: {count:12}││   │
│  │ │ created: "..."  ││ │ notify: true    ││ │ 2024-03: {count:8} ││   │
│  │ └─────────────────┘│ └─────────────────┘│ └────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Benefits:                                                               │
│  - Columns can be added dynamically                                     │
│  - Efficient for sparse data                                            │
│  - Good for time-series data                                            │
│  - Column families stored together (read optimization)                  │
│                                                                          │
│  Best For: Time-series, IoT data, event logging, user activity         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Document Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Document Model (MongoDB-style)                        │
│                                                                          │
│  Embedded Documents (Denormalized):                                     │
│  {                                                                       │
│    "_id": "order_123",                                                  │
│    "customer": {                    ◄── Embedded (not referenced)       │
│      "name": "John Doe",                                                │
│      "email": "john@example.com"                                        │
│    },                                                                    │
│    "items": [                       ◄── Array of embedded docs          │
│      {                                                                   │
│        "product_name": "Widget",                                        │
│        "quantity": 2,                                                   │
│        "price": 29.99                                                   │
│      }                                                                   │
│    ],                                                                    │
│    "shipping_address": {            ◄── Embedded                        │
│      "street": "123 Main St",                                           │
│      "city": "Boston"                                                   │
│    },                                                                    │
│    "total": 59.98                                                       │
│  }                                                                       │
│                                                                          │
│  Referenced Documents (Normalized):                                     │
│  {                                                                       │
│    "_id": "order_123",                                                  │
│    "customer_id": "user_456",       ◄── Reference (requires lookup)    │
│    "item_ids": ["item_1", "item_2"],◄── References                     │
│    "total": 59.98                                                       │
│  }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘

Embedding vs Referencing Decision:

┌─────────────────────────────────────────────────────────────────────────┐
│                           EMBED WHEN:                                    │
│                                                                          │
│  ✓ Data is read together frequently                                     │
│  ✓ Child data doesn't exist without parent                              │
│  ✓ One-to-few relationship (not one-to-millions)                        │
│  ✓ Data doesn't change often                                            │
│  ✓ Document size stays under 16MB limit                                 │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                          REFERENCE WHEN:                                 │
│                                                                          │
│  ✓ Data is accessed independently                                       │
│  ✓ Many-to-many relationships                                           │
│  ✓ One-to-many with large "many" side                                   │
│  ✓ Data changes frequently                                              │
│  ✓ Need to query child documents directly                               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Graph Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Graph Model (Neo4j-style)                             │
│                                                                          │
│                    ┌─────────┐                                          │
│           WORKS_AT │  Alice  │ KNOWS                                    │
│         ┌─────────►│ (User)  │◄──────────┐                              │
│         │          └────┬────┘           │                              │
│         │               │                │                              │
│         │               │ MANAGES        │                              │
│         │               ▼                │                              │
│   ┌─────┴─────┐   ┌──────────┐    ┌─────┴─────┐                        │
│   │  Acme Co  │   │   Bob    │    │   Carol   │                        │
│   │ (Company) │   │  (User)  │    │  (User)   │                        │
│   └───────────┘   └────┬─────┘    └───────────┘                        │
│                        │                                                │
│                        │ PURCHASED                                      │
│                        ▼                                                │
│                  ┌──────────┐                                           │
│                  │ Product X│                                           │
│                  │(Product) │                                           │
│                  └──────────┘                                           │
│                                                                          │
│  Query: "Find friends of friends who bought Product X"                  │
│                                                                          │
│  MATCH (u:User)-[:KNOWS*2]->(friend:User)-[:PURCHASED]->(p:Product)    │
│  WHERE u.name = 'Alice' AND p.name = 'Product X'                       │
│  RETURN friend                                                          │
│                                                                          │
│  Best For: Social networks, recommendations, fraud detection,          │
│            knowledge graphs, network topology                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## Indexing Strategies

### Index Types and Use Cases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Index Types                                      │
│                                                                          │
│  B-Tree Index (Default):                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        [M]                                       │   │
│  │                       /   \                                      │   │
│  │                    [D,G]   [P,T]                                 │   │
│  │                   /  |  \  /  |  \                               │   │
│  │                 [A-C][E-F][H-L][N-O][Q-S][U-Z]                   │   │
│  │                                                                   │   │
│  │  Best for: Range queries, sorting, equality checks               │   │
│  │  Examples: WHERE date > '2024-01-01', ORDER BY name             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Hash Index:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  hash("john@example.com") → bucket_42 → [row_ptr1, row_ptr2]    │   │
│  │                                                                   │   │
│  │  Best for: Exact equality lookups only                           │   │
│  │  Examples: WHERE email = 'john@example.com'                     │   │
│  │  NOT for: Range queries, sorting                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Composite Index:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  CREATE INDEX idx ON orders(customer_id, order_date, status);   │   │
│  │                                                                   │   │
│  │  Leftmost Prefix Rule:                                          │   │
│  │  ✓ WHERE customer_id = 1                                         │   │
│  │  ✓ WHERE customer_id = 1 AND order_date > '2024-01-01'          │   │
│  │  ✓ WHERE customer_id = 1 AND order_date = '...' AND status='...'│   │
│  │  ✗ WHERE order_date > '2024-01-01' (first column missing)       │   │
│  │  ✗ WHERE status = 'shipped' (first columns missing)             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Full-Text Index:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Inverted Index:                                                 │   │
│  │  "database" → [doc1, doc5, doc23]                               │   │
│  │  "scaling"  → [doc1, doc7, doc15]                               │   │
│  │  "design"   → [doc5, doc23, doc42]                              │   │
│  │                                                                   │   │
│  │  Best for: Text search, natural language queries                │   │
│  │  Examples: WHERE MATCH(content) AGAINST('database scaling')     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Index Design Principles

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Index Design Principles                               │
│                                                                          │
│  1. Index Selectivity (Higher is Better):                               │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Unique values / Total rows = Selectivity                    │    │
│     │                                                               │    │
│     │  email (unique): 1,000,000 / 1,000,000 = 1.0  ✓ Excellent   │    │
│     │  user_id:        500,000 / 1,000,000 = 0.5    ✓ Good        │    │
│     │  country:        200 / 1,000,000 = 0.0002     ✗ Poor alone  │    │
│     │  is_active:      2 / 1,000,000 = 0.000002     ✗ Very poor   │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  2. Covering Indexes (Include all needed columns):                      │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Query: SELECT name, email FROM users WHERE user_id = 123;   │    │
│     │                                                               │    │
│     │  Regular Index:  INDEX(user_id) → lookup → table → columns  │    │
│     │  Covering Index: INDEX(user_id, name, email) → done!        │    │
│     │                                                               │    │
│     │  Covering index avoids table lookup (index-only scan)        │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  3. Index Column Order (Most selective first):                          │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Query: WHERE tenant_id = 5 AND status = 'active'            │    │
│     │                                                               │    │
│     │  If tenant_id has 1000 unique values, status has 3:          │    │
│     │  Better:  INDEX(tenant_id, status)                           │    │
│     │  Worse:   INDEX(status, tenant_id)                           │    │
│     └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Data Modeling for Specific Use Cases

### Time-Series Data

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Time-Series Data Modeling                             │
│                                                                          │
│  Anti-Pattern (One row per event):                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  sensor_readings                                                 │   │
│  │  ┌───────────┬─────────────────────┬─────────┐                  │   │
│  │  │ sensor_id │ timestamp           │ value   │                  │   │
│  │  ├───────────┼─────────────────────┼─────────┤                  │   │
│  │  │ s1        │ 2024-01-01 00:00:00 │ 23.5    │                  │   │
│  │  │ s1        │ 2024-01-01 00:00:01 │ 23.6    │ ← Billions of   │   │
│  │  │ s1        │ 2024-01-01 00:00:02 │ 23.4    │   rows!         │   │
│  │  └───────────┴─────────────────────┴─────────┘                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Better: Bucketed/Chunked Approach:                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  sensor_readings_hourly                                          │   │
│  │  ┌───────────┬─────────────┬────────────────────────────────┐   │   │
│  │  │ sensor_id │ hour_bucket │ readings (array/json)          │   │   │
│  │  ├───────────┼─────────────┼────────────────────────────────┤   │   │
│  │  │ s1        │ 2024010100  │ [{t:0,v:23.5},{t:1,v:23.6}...] │   │   │
│  │  │ s1        │ 2024010101  │ [{t:0,v:24.1},{t:1,v:24.0}...] │   │   │
│  │  └───────────┴─────────────┴────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  Benefits: Fewer rows, better compression, efficient range queries│   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Time-Series Partitioning:                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                   │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ 2024-01  │ │ 2024-02  │ │ 2024-03  │ │ 2024-04  │           │   │
│  │  │ (active) │ │ (active) │ │ (archive)│ │ (archive)│           │   │
│  │  │  SSD     │ │  SSD     │ │  HDD     │ │  HDD     │           │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │   │
│  │                                                                   │   │
│  │  Hot data on fast storage, cold data on cheap storage            │   │
│  │  Easy to drop old partitions for retention policies              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Multi-Tenant Data

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Multi-Tenant Data Modeling                            │
│                                                                          │
│  Option 1: Shared Tables with Tenant ID                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  orders                                                          │   │
│  │  ┌───────────┬───────────┬────────────────┐                     │   │
│  │  │ tenant_id │ order_id  │ data...        │                     │   │
│  │  ├───────────┼───────────┼────────────────┤                     │   │
│  │  │ tenant_1  │ 1001      │ ...            │                     │   │
│  │  │ tenant_2  │ 1002      │ ...            │                     │   │
│  │  │ tenant_1  │ 1003      │ ...            │                     │   │
│  │  └───────────┴───────────┴────────────────┘                     │   │
│  │                                                                   │   │
│  │  INDEX: (tenant_id, order_id) -- tenant_id FIRST!               │   │
│  │                                                                   │   │
│  │  ✓ Simple to manage                                              │   │
│  │  ✓ Easy cross-tenant analytics                                   │   │
│  │  ✗ Noisy neighbor risk                                           │   │
│  │  ✗ Harder to isolate data for compliance                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Option 2: Schema-per-Tenant                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Database: app_db                                                │   │
│  │  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐      │   │
│  │  │ tenant_1       │ │ tenant_2       │ │ tenant_3       │      │   │
│  │  │ ├─ orders      │ │ ├─ orders      │ │ ├─ orders      │      │   │
│  │  │ ├─ products    │ │ ├─ products    │ │ ├─ products    │      │   │
│  │  │ └─ users       │ │ └─ users       │ │ └─ users       │      │   │
│  │  └────────────────┘ └────────────────┘ └────────────────┘      │   │
│  │                                                                   │   │
│  │  ✓ Better isolation                                              │   │
│  │  ✓ Easier per-tenant backup/restore                              │   │
│  │  ✗ Schema migrations are complex (N schemas to update)          │   │
│  │  ✗ Connection pooling challenges                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Option 3: Database-per-Tenant                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │  tenant_1   │  │  tenant_2   │  │  tenant_3   │             │   │
│  │  │  (database) │  │  (database) │  │  (database) │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                   │   │
│  │  ✓ Complete isolation                                            │   │
│  │  ✓ Easy compliance (data residency)                              │   │
│  │  ✓ Independent scaling                                           │   │
│  │  ✗ Expensive (more resources)                                    │   │
│  │  ✗ Complex connection management                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Trade-offs Summary

### Normalization vs Denormalization

| Aspect | Normalized | Denormalized |
|--------|------------|--------------|
| **Write Performance** | Fast (single update) | Slow (multiple updates) |
| **Read Performance** | Slow (JOINs) | Fast (single read) |
| **Storage Space** | Minimal | Larger (redundancy) |
| **Data Consistency** | Strong (single source) | Eventual (sync needed) |
| **Query Flexibility** | High (ad-hoc queries) | Low (optimized patterns) |
| **Schema Changes** | Easier | Harder |
| **Data Integrity** | Enforced by DB | Application responsibility |

### Data Model Selection Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Data Model Selection Guide                            │
│                                                                          │
│  Use Case                    Recommended Model                          │
│  ─────────────────────────────────────────────────────────────────      │
│  Financial transactions      Relational (ACID compliance)               │
│  User profiles               Document or Relational                     │
│  Product catalog             Document (varied attributes)               │
│  Social connections          Graph                                       │
│  IoT/sensor data             Time-series / Wide column                  │
│  Session storage             Key-Value (Redis)                          │
│  Content management          Document                                    │
│  Search/text                 Inverted index (Elasticsearch)             │
│  Recommendations             Graph + Vector                              │
│  Analytics/reporting         Columnar (data warehouse)                  │
│  Event sourcing              Append-only log (Kafka)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Real-World Examples

### Example 1: E-Commerce Product Catalog

```
Challenge: Products have varying attributes (electronics vs clothing)

Solution: Hybrid approach
┌─────────────────────────────────────────────────────────────────────────┐
│  Core Product Table (Relational):                                       │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ products: id, name, price, category_id, created_at           │      │
│  │ (Fixed attributes all products share)                        │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                          │
│  Category-Specific Tables:                                              │
│  ┌───────────────────────┐  ┌───────────────────────┐                  │
│  │ electronics_attrs     │  │ clothing_attrs        │                  │
│  │ - product_id          │  │ - product_id          │                  │
│  │ - voltage             │  │ - size                │                  │
│  │ - wattage             │  │ - color               │                  │
│  │ - connectivity        │  │ - material            │                  │
│  └───────────────────────┘  └───────────────────────┘                  │
│                                                                          │
│  Search Document (Elasticsearch):                                       │
│  {                                                                       │
│    "id": "prod_123",                                                    │
│    "name": "Wireless Headphones",                                       │
│    "category": "Electronics",                                           │
│    "price": 79.99,                                                      │
│    "attributes": {                                                       │
│      "connectivity": "Bluetooth 5.0",                                   │
│      "battery_life": "30 hours"                                         │
│    },                                                                    │
│    "searchable_text": "wireless bluetooth headphones music audio"       │
│  }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Example 2: Social Media Feed

```
Challenge: Display personalized feed with user posts, likes, comments

Solution: Denormalized feed with fan-out on write
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  User Posts (Source of Truth):        User Feeds (Denormalized):        │
│  ┌───────────────────────┐           ┌───────────────────────┐         │
│  │ posts                 │           │ user_feeds            │         │
│  │ - post_id             │           │ - user_id             │         │
│  │ - author_id           │  ──────►  │ - feed_position       │         │
│  │ - content             │  fan-out  │ - post_id             │         │
│  │ - created_at          │  on write │ - author_name         │         │
│  │ - like_count          │           │ - author_avatar       │         │
│  │ - comment_count       │           │ - content_preview     │         │
│  └───────────────────────┘           │ - like_count          │         │
│                                       │ - created_at          │         │
│                                       └───────────────────────┘         │
│                                                                          │
│  Feed Query: SELECT * FROM user_feeds                                   │
│              WHERE user_id = ?                                          │
│              ORDER BY feed_position DESC                                │
│              LIMIT 20;                                                  │
│                                                                          │
│  Result: Single query, no JOINs, sub-millisecond response              │
└─────────────────────────────────────────────────────────────────────────┘
```

## Interview Talking Points

1. "I start with normalized schemas to maintain data integrity, then strategically denormalize based on measured query patterns"

2. "The read/write ratio is my first consideration - high read ratios justify denormalization costs"

3. "I choose embedding vs referencing based on whether data is always accessed together"

4. "Indexes are powerful but have write overhead - I create them based on actual query patterns, not speculation"

5. "For time-series data, I use bucketing and partitioning to manage scale efficiently"

6. "Multi-tenant isolation level depends on compliance requirements and customer expectations"

## Common Interview Questions

### Q1: When would you denormalize a database?

**Answer:**
I would denormalize when:

1. **Read performance is critical** - If we're seeing slow queries due to complex JOINs and the system is read-heavy (10:1 ratio or higher)

2. **Access patterns are well-defined** - When we know exactly how data will be queried and can optimize for those specific patterns

3. **Write frequency is low** - When the denormalized data doesn't change often, reducing update complexity

4. **Eventual consistency is acceptable** - When slight delays in propagating updates are tolerable

Example: For a product page showing seller information, I'd embed the seller name directly in the product document rather than JOINing, accepting that seller name changes might take seconds to propagate.

### Q2: How do you design a schema for a multi-tenant SaaS application?

**Answer:**
The approach depends on isolation requirements and scale:

**Shared tables (most common):**
- Add tenant_id to every table
- ALWAYS put tenant_id first in composite indexes
- Use row-level security or application-level filtering
- Works for most B2B SaaS applications

**Schema-per-tenant:**
- When tenants need logical separation
- Useful for different feature sets per tenant
- More complex migrations

**Database-per-tenant:**
- Required for strict compliance (data residency)
- Large enterprise customers with specific requirements
- Most expensive but maximum isolation

I typically start with shared tables and add isolation only when requirements demand it.

### Q3: How would you model a social graph for friend recommendations?

**Answer:**
I'd use a graph database (Neo4j) or a hybrid approach:

**Primary storage:** Graph database for relationship traversal
- Nodes: Users with properties
- Edges: FOLLOWS, FRIENDS_WITH, BLOCKS relationships
- Efficient for queries like "friends of friends who like X"

**Recommendation query:**
```
MATCH (me:User {id: $userId})-[:FOLLOWS]->
      (friend)-[:FOLLOWS]->(recommended)
WHERE NOT (me)-[:FOLLOWS]->(recommended)
RETURN recommended, COUNT(friend) as mutual
ORDER BY mutual DESC LIMIT 10
```

**Caching layer:** Pre-compute recommendations periodically and cache in Redis for fast retrieval

**Hybrid approach:** Store user data in PostgreSQL, relationships in a graph database, and cache hot recommendations.

### Q4: How do you handle schema migrations with zero downtime?

**Answer:**
I follow the expand-contract pattern:

1. **Expand phase:**
   - Add new columns/tables (nullable or with defaults)
   - Deploy application code that writes to both old and new
   - Backfill existing data

2. **Migrate phase:**
   - Update application to read from new schema
   - Verify data consistency

3. **Contract phase:**
   - Remove old column/table access from code
   - Drop old columns/tables

Example: Renaming a column from `user_name` to `full_name`:
- Add `full_name` column
- Write to both columns
- Backfill `full_name` from `user_name`
- Switch reads to `full_name`
- Stop writing to `user_name`
- Drop `user_name` column

This ensures no downtime and provides rollback capability at each step.

## Key Takeaways

- Design schemas based on access patterns, not just data relationships
- Normalization prevents anomalies; denormalization improves read performance
- Start normalized, denormalize strategically based on measured bottlenecks
- Index design should follow the leftmost prefix rule and consider selectivity
- Time-series data benefits from bucketing and partitioning strategies
- Multi-tenant isolation level should match compliance and customer requirements
- Schema migrations should follow expand-contract for zero downtime
- Polyglot persistence (multiple databases) is often the right answer at scale

## Further Reading

- "Designing Data-Intensive Applications" by Martin Kleppmann
- "SQL Performance Explained" by Markus Winand
- "MongoDB Data Modeling" - Official MongoDB Documentation
- "Graph Databases" by Ian Robinson, Jim Webber, and Emil Eifrem
- "The Data Warehouse Toolkit" by Ralph Kimball
