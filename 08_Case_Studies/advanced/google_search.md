# Design a Web Search Engine (Google Search)

## 1. Problem Statement

Design a web-scale search engine capable of crawling billions of web pages, building
an inverted index, ranking results by relevance (including a simplified PageRank),
processing billions of queries per day with sub-second latency, providing spelling
corrections and query suggestions, and integrating advertisements alongside organic
search results.

---

## 2. Requirements

### Functional Requirements
- Crawl and index the public web (~50 billion pages)
- Process search queries and return ranked results within 500ms
- Spelling correction and "did you mean" suggestions
- Autocomplete / query suggestions as user types
- Image, video, and news search verticals
- Snippet generation (extract relevant text from pages)
- Freshness: index new/updated pages within minutes to hours
- Ads integration: display relevant sponsored results
- Safe search filtering
- Localization and language-specific results

### Non-Functional Requirements
- High availability: 99.99% uptime
- Low latency: P99 < 500ms for query results
- Scalability: handle 10 billion queries per day
- Freshness: breaking news indexed within 5 minutes
- Index completeness: > 90% of discoverable public web
- Relevance: high precision and recall in search results

---

## 3. Capacity Estimation

### Web and Query Scale
```
Web pages to index:       50 billion pages
Average page size:        100 KB (HTML + text)
Total raw content:        50B * 100 KB = 5 PB (raw HTML)
Extracted text per page:  ~10 KB average
Total text content:       50B * 10 KB = 500 TB

Queries per day:          10 billion
Queries per second (avg): 10B / 86,400 ~ 115,000 QPS
Queries per second (peak):~350,000 QPS
Average query length:     3 words (~20 characters)
```

### Crawling Scale
```
Pages to re-crawl monthly: ~20 billion (important pages more frequently)
Crawl rate:               20B / 30 days = 667M pages/day
                          = 7,700 pages/second
Crawl bandwidth:          7,700 * 100 KB = 770 MB/s = ~6.2 Gbps
New pages discovered/day: ~50 million
```

### Index Size
```
Unique terms (vocabulary):    ~100 million terms
Inverted index entry:         term -> list of (doc_id, positions, tf)
Average posting list length:  50,000 documents per term
Posting list entry:           12 bytes (4B doc_id + 4B term_freq + 4B position_ptr)
Inverted index size:          100M terms * 50K docs * 12B = 60 TB
                              With compression (~4x): ~15 TB
Forward index (per doc):      50B docs * 500 bytes = 25 TB
PageRank scores:              50B * 8 bytes = 400 GB
Document metadata:            50B * 200 bytes = 10 TB
Total index size:             ~50 TB (compressed)
```

### Infrastructure
```
Serving cluster:     index sharded across ~10,000 machines
                     Each machine holds ~5 TB of index
Crawling cluster:    ~2,000 crawler machines
Indexing pipeline:   ~5,000 machines (MapReduce/Spark)
Query serving:       ~50,000 machines (for redundancy and latency)
Total machines:      ~70,000+ (across all functions)
```

---

## 4. High-Level Design

```
                              +-------------------+
                              |   User Query      |
                              +--------+----------+
                                       |
                              +--------v----------+
                              | DNS + Global      |
                              | Load Balancer     |
                              +--------+----------+
                                       |
                    +------------------+------------------+
                    |                                     |
           +--------v---------+                  +--------v---------+
           | Web Frontend     |                  | Ads Service      |
           | (query parsing,  |                  | (bid auction,    |
           |  spell check,    |                  |  ad matching,    |
           |  suggestions)    |                  |  budget mgmt)    |
           +--------+---------+                  +--------+---------+
                    |                                     |
           +--------v---------+                           |
           | Query Router     |                           |
           | (shard selection,|                           |
           |  scatter-gather) |                           |
           +--------+---------+                           |
                    |                                     |
        +-----------+-----------+                         |
        |           |           |                         |
  +-----v----+ +---v------+ +--v-------+                 |
  | Index    | | Index    | | Index    |                  |
  | Shard 0  | | Shard 1  | | Shard N  |                  |
  | (search  | | (search  | | (search  |                  |
  |  + rank) | |  + rank) | |  + rank) |                  |
  +-----+----+ +---+------+ +--+-------+                 |
        |           |           |                         |
        +-----------+-----------+                         |
                    |                                     |
           +--------v---------+                           |
           | Result Merger    |<--------------------------+
           | + Re-Ranker      |
           | + Snippet Gen    |
           +--------+---------+
                    |
           +--------v---------+
           | Search Results   |
           | Page (SERP)      |
           +------------------+


   Offline Pipeline (Crawling + Indexing):

   +------------------+     +------------------+     +------------------+
   | Web Crawler      |---->| Content          |---->| Indexing          |
   | (distributed,    |     | Processing       |     | Pipeline         |
   |  politeness)     |     | (parse, extract, |     | (inverted index, |
   |                  |     |  dedup, lang)    |     |  forward index,  |
   +--------+---------+     +------------------+     |  PageRank)       |
            |                                        +--------+---------+
            v                                                 |
   +--------+---------+                              +--------v---------+
   | URL Frontier     |                              | Index Shards     |
   | (priority queue, |                              | (distributed     |
   |  politeness)     |                              |  serving)        |
   +------------------+                              +------------------+
```

---

## 5. API Design

### Search Query

```
GET /api/v1/search?q=distributed+systems&start=0&num=10&lang=en&safe=moderate&loc=us
Headers: Authorization: Bearer <token> (optional, for personalization)
Response 200: {
    "query": "distributed systems",
    "corrected_query": null,
    "total_results": 48500000,
    "search_time_ms": 230,
    "results": [
        {
            "position": 1,
            "url": "https://example.com/distributed-systems",
            "title": "Introduction to Distributed Systems",
            "snippet": "A distributed system is a collection of independent computers that appear to users as a single coherent system...",
            "cached_url": "https://cache.search.com/q=abc123",
            "published_date": "2025-01-10",
            "site_links": [
                { "title": "CAP Theorem", "url": "https://example.com/cap" },
                { "title": "Consensus", "url": "https://example.com/consensus" }
            ]
        }
    ],
    "ads": [
        {
            "position": "top",
            "title": "Learn Distributed Systems - Online Course",
            "url": "https://ads.example.com/click/ad123",
            "display_url": "www.coursesite.com/distributed",
            "description": "Master distributed systems with hands-on projects...",
            "ad_label": "Sponsored"
        }
    ],
    "related_searches": [
        "distributed systems book",
        "distributed systems interview questions",
        "distributed computing vs distributed systems"
    ],
    "knowledge_panel": null
}
```

### Suggestions / Autocomplete

```
GET /api/v1/suggest?q=distrib&lang=en&num=8
Response 200: {
    "suggestions": [
        { "text": "distributed systems", "score": 0.95 },
        { "text": "distributed computing", "score": 0.88 },
        { "text": "distributed database", "score": 0.82 },
        { "text": "distribution center near me", "score": 0.75 }
    ]
}
```

### Crawl Status (Internal)

```
GET /internal/v1/crawl/status?domain=example.com
Response 200: {
    "domain": "example.com",
    "pages_crawled": 15420,
    "pages_indexed": 14800,
    "last_crawl": "2025-01-15T08:30:00Z",
    "crawl_rate": "2 pages/second",
    "robots_txt": { "disallow": ["/admin", "/private"] },
    "avg_page_quality": 0.72
}
```

---

## 6. Database Schema

### Document Store (Distributed — Bigtable/Custom)
```
Document Table:
+------------------+------------------+----------------------------------------+
| Field            | Type             | Description                            |
+------------------+------------------+----------------------------------------+
| doc_id           | UINT64           | Unique document identifier             |
| url              | STRING           | Canonical URL                          |
| url_hash         | UINT64           | Hash for fast dedup lookup             |
| domain           | STRING           | Domain name                            |
| title            | STRING           | Page title                             |
| content_text     | TEXT             | Extracted clean text                   |
| content_hash     | UINT64           | For duplicate detection                |
| language         | STRING(5)        | Detected language (ISO 639)            |
| page_rank        | FLOAT64          | PageRank score                         |
| spam_score       | FLOAT32          | Spam probability (0-1)                 |
| last_crawled     | TIMESTAMP        | When last crawled                      |
| last_modified    | TIMESTAMP        | HTTP Last-Modified header              |
| crawl_frequency  | ENUM             | hourly/daily/weekly/monthly            |
| http_status      | INT16            | Last HTTP status code                  |
| outgoing_links   | LIST<UINT64>     | Doc IDs of outgoing links              |
| incoming_links   | INT32            | Count of incoming links                |
+------------------+------------------+----------------------------------------+
Partitioned by: doc_id (hash partitioning)
Indexes: url_hash, domain, last_crawled
```

### Inverted Index (Custom Distributed Structure)
```
Index Structure:
+------------------+------------------+----------------------------------------+
| Field            | Type             | Description                            |
+------------------+------------------+----------------------------------------+
| term             | STRING           | Indexed term (lowercased, stemmed)     |
| term_id          | UINT32           | Numeric ID for the term                |
| doc_frequency    | UINT32           | Number of documents containing term    |
| posting_list     | COMPRESSED_BLOB  | Sorted list of (doc_id, tf, positions) |
+------------------+------------------+----------------------------------------+
Sharding: By doc_id range (document-partitioned) or by term (term-partitioned)
Compression: Variable-byte encoding for doc_id deltas, positions
```

### URL Frontier (Redis / Custom Queue)
```
URL Frontier Entry:
+------------------+------------------+----------------------------------------+
| Field            | Type             | Description                            |
+------------------+------------------+----------------------------------------+
| url              | STRING           | URL to crawl                           |
| url_hash         | UINT64           | For dedup                              |
| domain           | STRING           | Domain (for politeness)                |
| priority         | FLOAT32          | Crawl priority score                   |
| last_crawled     | TIMESTAMP        | When last crawled (NULL if new)        |
| discovery_source | ENUM             | sitemap/link/seed/resubmit             |
| retry_count      | INT8             | Number of failed attempts              |
+------------------+------------------+----------------------------------------+
Partitioned by: domain (ensures politeness per domain)
Priority queue: sorted by priority within each domain partition
```

### Query Log (for analytics and suggestions)
```sql
CREATE TABLE query_log (
    query_id        UUID,
    query_text      VARCHAR(500),
    user_id         BIGINT,                     -- anonymized or null
    session_id      VARCHAR(64),
    country         VARCHAR(5),
    language        VARCHAR(5),
    result_count    INT,
    click_position  INT,                        -- which result was clicked
    click_url       VARCHAR(2048),
    latency_ms      INT,
    timestamp       TIMESTAMP,

    PRIMARY KEY (timestamp, query_id)           -- time-partitioned
);
```

---

## 7. Deep Dive

### Deep Dive 1: Web Crawling at Scale

The web crawler must be distributed, polite, efficient, and adaptive.

```
Crawler Architecture:

  +------------------+     +-------------------+     +------------------+
  | URL Frontier     |     | Crawler Workers   |     | Content Store    |
  | (priority queue  |---->| (distributed,     |---->| (raw HTML +      |
  |  + politeness    |     |  ~2000 machines)  |     |  parsed content) |
  |  scheduler)      |     |                   |     |                  |
  +--------+---------+     +--------+----------+     +------------------+
           ^                        |
           |                        v
           |               +--------+----------+
           |               | Link Extractor    |
           |               | + URL Normalizer  |
           +<--------------| + Dedup Filter    |
                           +-------------------+
```

**URL Frontier Design:**
```
The URL Frontier has two main components:

1. Priority Queues (Front Queue):
   +-----------+    +-----------+    +-----------+
   | Priority  |    | Priority  |    | Priority  |
   |   HIGH    |    |  MEDIUM   |    |   LOW     |
   | (news,    |    | (popular  |    | (long-tail|
   |  updated) |    |  sites)   |    |  pages)   |
   +-----------+    +-----------+    +-----------+
         |                |                |
         +-------+--------+--------+-------+
                 |                 |
                 v                 v
2. Politeness Queues (Back Queue):
   One queue per domain, ensures we respect crawl rate limits

   +------------+  +------------+  +------------+  +------------+
   | domain:    |  | domain:    |  | domain:    |  | domain:    |
   | google.com |  | wiki.org   |  | nytimes.com|  | blog.xyz   |
   | rate: 2/s  |  | rate: 5/s  |  | rate: 1/s  |  | rate: 0.5/s|
   | queue: ... |  | queue: ... |  | queue: ... |  | queue: ... |
   +------------+  +------------+  +------------+  +------------+

Crawl Priority Calculation:
  priority = w1 * page_rank +
             w2 * freshness_score +
             w3 * change_frequency +
             w4 * domain_authority

  where:
    page_rank:        higher PageRank = higher priority
    freshness_score:  staleness since last crawl (news sites get re-crawled frequently)
    change_frequency: pages that change often get re-crawled more
    domain_authority: top-level domains with many quality inlinks
```

**Politeness Protocol:**
```
For each domain:
  1. Fetch and cache robots.txt (respect Crawl-delay directive)
  2. Default: max 1 request per second per domain
  3. Adaptive: if server responds slowly (> 2s), reduce rate
  4. If HTTP 429 (Too Many Requests): exponential backoff
  5. Honor robots.txt Disallow directives
  6. Use If-Modified-Since / ETags to avoid re-downloading unchanged pages
  7. Identify crawler via User-Agent string
```

**Duplicate Detection:**
```
Two levels of dedup:

1. URL-level dedup (before crawling):
   - Normalize URLs (lowercase, remove fragments, sort params)
   - Check URL hash against Bloom filter (100B URLs, 1% false positive)
   - Bloom filter size: ~120 GB at 1% FPR for 100B items

2. Content-level dedup (after crawling):
   - Compute SimHash / MinHash fingerprint of page content
   - Compare against existing fingerprints
   - Hamming distance < 3 bits = near-duplicate
   - Reduces index bloat from mirror sites, syndicated content
```

### Deep Dive 2: Indexing Pipeline and PageRank

```
Indexing Pipeline (MapReduce / Spark):

  Raw Crawled Pages
        |
        v
  +-----+------+     +-----------+     +-----------+     +---------------+
  | Parse HTML  |---->| Extract   |---->| Language  |---->| Build Forward |
  | Remove tags |     | Title,    |     | Detection |     | Index         |
  | Extract text|     | Links,    |     | + Encoding|     | (doc -> terms)|
  |             |     | Metadata  |     |           |     |               |
  +-------------+     +-----------+     +-----------+     +-------+-------+
                                                                  |
                                                                  v
  +--------------+     +-----------+     +-----------+     +------+--------+
  | Ship to      |<----| Sort by   |<----| Invert:   |<----| Tokenize,     |
  | Index Servers|     | Term ID   |     | term ->   |     | Stem, Remove  |
  | (shards)     |     | (for      |     | doc_ids   |     | Stop Words    |
  |              |     |  sharding)|     |           |     |               |
  +--------------+     +-----------+     +-----------+     +---------------+
```

**Inverted Index Structure:**
```
Term: "distributed"
  Document Frequency: 2,500,000
  Posting List (compressed):
  +--------+----+----------+
  | doc_id | tf | positions|
  +--------+----+----------+
  | 1042   | 3  | [5,22,87]|
  | 1055   | 1  | [14]     |
  | 1089   | 7  | [2,8,15,.|
  | ...    |    |          |
  +--------+----+----------+

  Compression: delta-encode doc_ids, variable-byte encode
  Original: [1042, 1055, 1089, ...]
  Deltas:   [1042,   13,   34, ...]
  VByte:    fewer bytes for small deltas

  Skip Pointers (every 128 entries):
  [1042, ptr] -> [15000, ptr] -> [30000, ptr] -> ...
  Enables fast intersection of posting lists
```

**Simplified PageRank:**
```
PageRank Algorithm:

  PR(A) = (1 - d) / N  +  d * SUM( PR(Ti) / C(Ti) )

  where:
    d = damping factor (0.85)
    N = total number of pages
    Ti = pages that link to A
    C(Ti) = number of outgoing links from Ti

  Iterative computation (MapReduce):

  Iteration 0: PR(every page) = 1/N

  Map Phase (for each page P):
    For each outgoing link L from P:
      Emit (L, PR(P) / outgoing_count(P))

  Reduce Phase (for each page Q):
    new_PR(Q) = (1-d)/N + d * SUM(all contributions)

  Repeat for 40-50 iterations until convergence
  (delta < 0.0001 for all pages)

  Scale: 50 billion pages, ~200 bytes per page in link graph
         Link graph size: ~10 TB
         Each iteration processes full graph: ~30 minutes on large cluster
         Total: ~25 hours for full PageRank computation
         Run weekly (incremental updates for daily freshness)
```

**Index Sharding Strategies:**
```
Option A: Document-Partitioned Index
  +----------+  +----------+  +----------+
  | Shard 0  |  | Shard 1  |  | Shard 2  |  ...
  | docs     |  | docs     |  | docs     |
  | 0-5B     |  | 5B-10B   |  | 10B-15B  |
  | ALL terms|  | ALL terms|  | ALL terms|
  +----------+  +----------+  +----------+

  Query: search ALL shards, merge results
  Pro: each shard is independent, easy to add shards
  Con: every query hits every shard (fan-out)

Option B: Term-Partitioned Index
  +----------+  +----------+  +----------+
  | Shard 0  |  | Shard 1  |  | Shard 2  |  ...
  | terms    |  | terms    |  | terms    |
  | a-f      |  | g-m      |  | n-s      |
  | ALL docs |  | ALL docs |  | ALL docs |
  +----------+  +----------+  +----------+

  Query: route to specific shards based on query terms
  Pro: less fan-out for single-term queries
  Con: multi-term queries need cross-shard join, harder to balance

Choice: Document-Partitioned (Google uses this)
  - Simpler to build and maintain
  - Each shard returns local top-K
  - Merge top-K from all shards for global ranking
  - Handle with tiered index: Tier 1 (top 10% by PageRank, fast)
    Tier 2 (remaining 90%, slower, only if Tier 1 insufficient)
```

### Deep Dive 3: Query Processing and Ranking

```
Query Processing Pipeline:

  User Query: "distrbuted system design"
       |
       v
  +----+-------------+
  | Query Parser      |
  | - Tokenize        |
  | - Lowercase       |
  | - Remove stopwords|
  | tokens: ["distrbuted", "system", "design"]
  +----+--------------+
       |
       v
  +----+-------------+
  | Spell Correction  |
  | - Edit distance   |
  | - Language model   |
  | corrected: "distributed system design"
  | "Did you mean: distributed system design?"
  +----+--------------+
       |
       v
  +----+-------------+
  | Query Expansion   |
  | - Synonyms        |
  | - Stemming        |
  | expanded: ["distributed", "distribute",
  |            "system", "systems",
  |            "design", "designing"]
  +----+--------------+
       |
       v
  +----+-------------+
  | Query Router      |     Scatter to all index shards
  | (scatter-gather)  |---> Each shard returns local top-100
  +----+--------------+
       |
       v
  +----+-------------+
  | Result Merger     |
  | + Global Ranking  |
  | Merge top-100 from each shard
  | Re-rank globally using full feature set
  +----+--------------+
       |
       v
  +----+-------------+
  | Snippet Generator |
  | Extract most relevant text passage
  | Highlight query terms in bold
  +----+--------------+
       |
       v
  Search Results Page
```

**Ranking Algorithm (Simplified):**
```
Score(query, document) = w1 * BM25(query, document)
                       + w2 * PageRank(document)
                       + w3 * Freshness(document)
                       + w4 * ClickThroughRate(query, document)
                       + w5 * DomainAuthority(document)
                       + w6 * URLMatch(query, document)
                       + w7 * TitleMatch(query, document)
                       + w8 * UserPersonalization(query, user)

BM25 Scoring:
  BM25(q, d) = SUM for each term t in q:
    IDF(t) * (tf(t,d) * (k1 + 1)) / (tf(t,d) + k1 * (1 - b + b * |d|/avgdl))

  where:
    IDF(t) = log((N - df(t) + 0.5) / (df(t) + 0.5))
    tf(t,d) = term frequency of t in document d
    |d| = document length
    avgdl = average document length
    k1 = 1.2 (term frequency saturation)
    b = 0.75 (length normalization)

Two-Phase Ranking:
  Phase 1 (per shard, fast): BM25 + PageRank + basic signals
           Select top 100 candidates per shard
  Phase 2 (global, slower):  Full ML model with 1000+ features
           Re-rank merged candidates from all shards
           Neural model (BERT-based) for semantic understanding
```

**Spelling Correction:**
```
Approach: Noisy Channel Model + Language Model

P(correction | misspelling) proportional to:
  P(misspelling | correction) * P(correction)

Components:
  1. Edit Distance Model:
     - Weighted edit distance (common typos have lower cost)
     - "distrbuted" -> "distributed" (edit distance 1: insert 'i')
     - Pre-compute candidates within edit distance 2

  2. Language Model (N-gram):
     - P("distributed system") >> P("distrbuted system")
     - Trained on query logs and web text
     - Trigram model: P(w3 | w1, w2)

  3. Query Log Evidence:
     - "distrbuted" was corrected to "distributed" by 95% of users
     - Historical correction patterns

Decision: suggest correction if:
  P(correction | query) > threshold AND
  correction has significantly more search results
```

---

## 8. Scaling Discussion

### Index Serving Scaling
- Each index shard replicated 3-5 times for redundancy and load balancing
- 10,000 shards x 5 replicas = 50,000 index servers
- Two-tier index: Tier 1 (high PageRank docs, fast) serves 90% of queries
- Tier 2 (long-tail docs) invoked only when Tier 1 results are insufficient
- Cache frequent query results (top 10% of queries cover 50% of traffic)

### Crawling Scaling
- Distributed across 2,000+ machines, each crawling ~4 pages/second
- Geographically distributed crawlers (US, EU, Asia) for locality
- Adaptive crawl scheduling: popular/changing pages crawled hourly, static pages monthly
- Separate fast-track crawler for news (refreshes every 5 minutes)

### Indexing Pipeline Scaling
- Batch indexing via MapReduce/Spark for full re-indexing (weekly)
- Real-time indexing pipeline for freshness (Kafka + Flink)
- New content indexed and searchable within 5-30 minutes
- Incremental index updates (merge new postings into existing shards)

### Query Processing Scaling
- Geo-distributed query serving clusters (US, EU, Asia, etc.)
- DNS routes users to nearest cluster
- Each cluster has full copy of index
- Auto-scaling: add more replicas during peak hours
- Result caching: Memcached layer caches top queries (cache hit rate ~30%)

---

## 9. Trade-offs

| Decision | Option A | Option B | Choice & Rationale |
|----------|----------|----------|-------------------|
| Index partitioning | Document-based | Term-based | Document-based: simpler fan-out model, each shard independent, easier to add capacity. Term-based has lower fan-out but complex multi-term queries. |
| Freshness vs completeness | Real-time index | Batch re-index | Hybrid: real-time for news/popular sites, batch for long-tail. Balance freshness with indexing quality. |
| Ranking model | BM25 (traditional) | Neural (BERT) | Both: BM25 for Phase 1 (fast candidate selection), neural for Phase 2 (precise re-ranking of top candidates). |
| Crawl storage | Store full HTML | Store extracted text only | Both: full HTML for re-processing, extracted text for indexing. Full HTML enables re-indexing without re-crawling. |
| Spell correction | Dictionary-based | Statistical (ML) | Statistical: handles novel misspellings, personalizable, learns from query logs. Dictionary misses domain-specific terms. |
| Cache granularity | Full result pages | Posting lists | Both: cache full results for top queries, cache posting lists for common terms. Different cache tiers for different access patterns. |
| Snippet generation | Pre-compute | On the fly | On the fly: snippets depend on query (highlight different passages for different queries). Cache popular query-doc snippets. |

---

## 10. Failure Scenarios & Handling

### Index Shard Failure
- **Detection:** Health check failure, query timeouts from shard
- **Handling:** Route queries to replica shards; mark failed shard as unhealthy
- **Recovery:** Replace machine, copy index from replica
- **Prevention:** 3-5 replicas per shard; spread across racks/AZs

### Crawler Overloading Target Site
- **Detection:** HTTP 429 responses, connection refused, robots.txt changes
- **Handling:** Immediately back off; reduce crawl rate for domain
- **Recovery:** Gradually increase crawl rate after cooldown period
- **Prevention:** Strict politeness enforcement; respect robots.txt; adaptive rate limiting

### Index Corruption
- **Detection:** Checksum verification failures during query serving
- **Handling:** Take corrupted shard offline; serve from replicas
- **Recovery:** Rebuild shard from source data (re-index from content store)
- **Prevention:** Checksums on all index files; periodic integrity audits

### Stale Index (Freshness Degradation)
- **Detection:** Monitoring shows increasing index age; user complaints about outdated results
- **Handling:** Prioritize re-crawling of stale domains; boost real-time indexing pipeline
- **Recovery:** Emergency batch re-index of affected domains
- **Prevention:** Adaptive crawl scheduling; real-time feed from sitemaps and news sources

### Query Serving Latency Spike
- **Detection:** P99 latency exceeds 500ms threshold
- **Handling:** Shed load (return fewer results, skip Phase 2 re-ranking); use cached results
- **Recovery:** Scale out query servers; investigate root cause (hot shard, slow network)
- **Prevention:** Over-provision capacity; latency budgets per pipeline stage

### Spam Content Infiltration
- **Detection:** Spam classifiers flag increase in spam pages in index
- **Handling:** Demote or remove identified spam; update spam model
- **Recovery:** Re-crawl and re-index affected domains with updated spam filters
- **Prevention:** ML-based spam detection during indexing; link analysis for link farms

---

## 11. Monitoring & Alerting

### Key Metrics

```
+-------------------------------+-----------------+-------------------+
| Metric                        | Target          | Alert Threshold   |
+-------------------------------+-----------------+-------------------+
| Query latency (P50)           | < 200 ms        | > 400 ms          |
| Query latency (P99)           | < 500 ms        | > 1 second        |
| Query success rate            | > 99.99%        | < 99.9%           |
| Index freshness (avg age)     | < 24 hours      | > 72 hours        |
| News freshness                | < 5 minutes     | > 15 minutes      |
| Crawl throughput              | > 600M pages/day| < 400M pages/day  |
| Index coverage                | > 90% of web    | < 85%             |
| Spam rate in results          | < 0.1%          | > 1%              |
| Zero-result rate              | < 5%            | > 10%             |
| Click-through rate (top 3)    | > 40%           | < 30%             |
| Spell correction accuracy     | > 95%           | < 90%             |
| Ad relevance score            | > 0.7           | < 0.5             |
+-------------------------------+-----------------+-------------------+
```

### Dashboards
- **Query Quality:** latency distribution, zero-result rate, click-through by position
- **Crawl Health:** pages crawled/day, error rates by type, domain coverage
- **Index Health:** shard status, freshness by domain category, index size growth
- **Ranking Quality:** NDCG scores, A/B test metrics, spam detection rates
- **Infrastructure:** server utilization, network bandwidth, storage capacity

### Alerting Strategy
- Critical (PagerDuty): query serving down, index corruption, P99 > 2 seconds
- Warning (Slack): freshness degradation, crawl rate drop, spam rate increase
- Informational: daily crawl statistics, index growth, ranking quality metrics

---

## 12. Interview Variations

### Common Follow-up Questions

**Q: How would you handle the "freshness" problem for news content?**
A: Dedicated news crawling pipeline with 5-minute refresh cycles for major news sources.
RSS/Atom feed monitoring for real-time updates. Push-based ingestion from news partners.
Separate "freshness index" that gets merged into main index. Boost freshness signal in
ranking for queries that appear to be about current events (detect via trending query
analysis).

**Q: How would you implement Google Knowledge Graph / Knowledge Panels?**
A: Build structured knowledge base from Wikipedia, Wikidata, and authoritative sources.
Entity recognition in queries (detect when query refers to a known entity). Link entities
to knowledge base entries. Display structured data (biography, facts, images) in a
knowledge panel alongside organic results. Use entity relationships for query understanding
(e.g., "Obama wife" -> Michelle Obama).

**Q: How do you handle personalization in search results?**
A: Use user search history, click patterns, and location for personalization signals.
Apply as a re-ranking boost (not a filter) to base results. Location-based results for
local queries ("restaurants near me"). Language preference from user settings. Keep
personalization weight low (< 10% of total score) to avoid filter bubbles. Allow users
to disable personalization.

**Q: How does the ads auction work alongside organic results?**
A: Generalized Second-Price (GSP) auction runs in parallel with organic search. Advertisers
bid on keywords. Ad rank = bid * quality_score. Top ads shown above organic results,
bottom ads below. Quality score considers ad relevance, landing page quality, expected
click-through rate. Advertiser pays minimum bid needed to maintain their position.
Strict separation between organic ranking and ad ranking (no paid influence on organic).

**Q: How do you handle multiple languages and internationalization?**
A: Language detection during crawling (HTTP headers + content analysis). Separate
indexes per language or language-annotated unified index. Language-specific tokenization
(CJK needs segmentation, Arabic needs right-to-left handling). Query language detection
to route to appropriate index. Cross-language search using translation models for
multilingual queries. Local domain boosting (prefer .de domains for German queries).

**Q: How would you implement image search?**
A: Crawl images alongside web pages. Extract alt text, surrounding text, and page title
as text signals. Compute visual features using CNN embeddings. Build visual similarity
index using approximate nearest neighbors (HNSW). Multi-modal search combining text
matching (alt text, page content) with visual similarity. Support reverse image search
using visual feature matching.

**Q: What is the difference between indexing 1 billion vs 50 billion pages?**
A: At 50 billion scale, every component must be distributed. Index cannot fit on a single
machine. Need tiered serving (small fast tier for popular pages, large slow tier for
long-tail). Crawling requires thousands of machines with careful scheduling. PageRank
computation takes hours even on large clusters. Deduplication becomes critical — web has
massive redundancy. Quality signals matter more (need to filter low-quality content).
