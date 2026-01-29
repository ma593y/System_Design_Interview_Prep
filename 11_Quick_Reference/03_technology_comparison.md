# Technology Comparison Guide

## Databases

### SQL Databases

| Database | Best For | Max Scale | Key Features |
|----------|----------|-----------|--------------|
| **PostgreSQL** | Complex queries, ACID | ~10TB practical | JSON support, extensions, full-text search |
| **MySQL** | Web apps, read-heavy | ~10TB practical | Proven, wide adoption, good replication |
| **SQL Server** | Enterprise, .NET | Very large | Enterprise features, BI integration |
| **SQLite** | Embedded, local | ~140TB theoretical | Zero config, single file, embedded |

### NoSQL Databases

| Database | Type | Best For | Key Features |
|----------|------|----------|--------------|
| **MongoDB** | Document | Flexible schema, rapid dev | JSON-like docs, sharding, aggregation |
| **Cassandra** | Wide-column | Write-heavy, time-series | Linear scaling, no SPOF, tunable consistency |
| **DynamoDB** | Key-value | Serverless, auto-scale | Managed, pay-per-use, global tables |
| **Redis** | Key-value | Caching, sessions | In-memory, pub/sub, data structures |
| **Elasticsearch** | Search | Full-text search, logs | Distributed search, real-time analytics |
| **Neo4j** | Graph | Relationships, recommendations | Graph queries, ACID, visualizations |

### Database Selection Matrix

```
Need ACID transactions?
├── Yes → Need horizontal scaling?
│         ├── Yes → CockroachDB, Spanner, TiDB
│         └── No → PostgreSQL, MySQL
└── No → What's your data model?
         ├── Documents → MongoDB, CouchDB
         ├── Key-Value → Redis, DynamoDB
         ├── Wide-Column → Cassandra, HBase
         ├── Graph → Neo4j, Neptune
         └── Search → Elasticsearch, Solr
```

## Caching

| Cache | Type | Best For | Latency |
|-------|------|----------|---------|
| **Redis** | In-memory | Sessions, counters, pub/sub | < 1ms |
| **Memcached** | In-memory | Simple caching, large values | < 1ms |
| **Varnish** | HTTP cache | Full page caching | < 1ms |
| **CDN** | Edge cache | Static assets, global | Varies by location |

### Cache Comparison

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Rich (lists, sets, hashes) | Simple (strings) |
| Persistence | Optional | No |
| Replication | Yes | No |
| Clustering | Yes | No (client-side) |
| Pub/Sub | Yes | No |
| Max value size | 512MB | 1MB |
| Memory efficiency | Good | Better |

## Message Queues

| Queue | Best For | Ordering | Persistence |
|-------|----------|----------|-------------|
| **Kafka** | Event streaming, high throughput | Per partition | Yes (log-based) |
| **RabbitMQ** | Task queues, routing | Per queue | Optional |
| **SQS** | Serverless, AWS native | FIFO available | Yes |
| **Redis Pub/Sub** | Simple pub/sub, real-time | No guarantee | No |

### Message Queue Comparison

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| Throughput | Very high (millions/sec) | High (tens of thousands) | High |
| Ordering | Per partition | Per queue | FIFO optional |
| Consumer model | Pull | Push | Pull |
| Message replay | Yes | No | No |
| Complexity | High | Medium | Low |
| Managed option | Confluent, MSK | CloudAMQP | Native AWS |

## Load Balancers

| Type | Layer | Best For | Examples |
|------|-------|----------|----------|
| **L4 (Transport)** | TCP/UDP | Simple routing, high performance | HAProxy, NLB |
| **L7 (Application)** | HTTP | Content routing, SSL termination | Nginx, ALB, Envoy |
| **DNS** | DNS | Geographic routing | Route53, Cloudflare |

### Load Balancing Algorithms

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| Round Robin | Rotate through servers | Equal capacity servers |
| Least Connections | Send to server with fewest connections | Variable request duration |
| IP Hash | Hash client IP to server | Session persistence |
| Weighted | Assign weights to servers | Mixed capacity servers |

## API Styles

| Style | Best For | Data Format | Key Features |
|-------|----------|-------------|--------------|
| **REST** | CRUD, public APIs | JSON | Stateless, cacheable, widely adopted |
| **GraphQL** | Flexible queries, mobile | JSON | Single endpoint, client-specified data |
| **gRPC** | Microservices, performance | Protobuf | Binary, streaming, strong typing |
| **WebSocket** | Real-time, bidirectional | Any | Persistent connection, low latency |

### API Comparison

| Feature | REST | GraphQL | gRPC |
|---------|------|---------|------|
| Protocol | HTTP/1.1+ | HTTP | HTTP/2 |
| Data format | JSON | JSON | Protobuf (binary) |
| Caching | Easy (HTTP caching) | Complex | Difficult |
| Over-fetching | Common | Solved | Solved |
| Learning curve | Low | Medium | Medium-High |
| Browser support | Native | Native | Needs proxy |

## Container Orchestration

| Platform | Best For | Complexity | Cloud Options |
|----------|----------|------------|---------------|
| **Kubernetes** | Large scale, multi-cloud | High | EKS, GKE, AKS |
| **Docker Swarm** | Simpler setups | Medium | - |
| **ECS** | AWS native | Low-Medium | Native |
| **Nomad** | Multi-workload | Medium | - |

## Search Engines

| Engine | Best For | Key Features |
|--------|----------|--------------|
| **Elasticsearch** | Full-text, logs, analytics | Distributed, real-time, aggregations |
| **Solr** | Enterprise search | Mature, rich features, XML config |
| **Algolia** | Instant search, typo tolerance | Hosted, fast, great UX |
| **Meilisearch** | Simple full-text | Easy setup, typo tolerant |

## Monitoring & Observability

### Metrics

| Tool | Best For | Key Features |
|------|----------|--------------|
| **Prometheus** | Cloud-native metrics | Pull-based, PromQL, alerting |
| **Datadog** | Full observability | SaaS, integrations, APM |
| **CloudWatch** | AWS native | Integrated, alarms, dashboards |
| **Grafana** | Visualization | Multi-source, dashboards, alerts |

### Logging

| Tool | Best For | Key Features |
|------|----------|--------------|
| **ELK Stack** | Log aggregation | Full-text search, visualizations |
| **Splunk** | Enterprise logging | Powerful search, compliance |
| **Loki** | Cloud-native logs | Prometheus-like, efficient |

### Tracing

| Tool | Best For | Key Features |
|------|----------|--------------|
| **Jaeger** | Distributed tracing | Open source, Kubernetes native |
| **Zipkin** | Distributed tracing | Simple, wide adoption |
| **X-Ray** | AWS native | Integrated, service maps |

## Cloud Provider Comparison

### Compute

| Service | AWS | GCP | Azure |
|---------|-----|-----|-------|
| VMs | EC2 | Compute Engine | Virtual Machines |
| Containers | ECS/EKS | GKE | AKS |
| Serverless | Lambda | Cloud Functions | Azure Functions |

### Storage

| Service | AWS | GCP | Azure |
|---------|-----|-----|-------|
| Object | S3 | Cloud Storage | Blob Storage |
| Block | EBS | Persistent Disk | Managed Disks |
| File | EFS | Filestore | Azure Files |

### Database

| Service | AWS | GCP | Azure |
|---------|-----|-----|-------|
| Relational | RDS/Aurora | Cloud SQL | Azure SQL |
| NoSQL | DynamoDB | Firestore/Bigtable | Cosmos DB |
| Cache | ElastiCache | Memorystore | Azure Cache |

## Quick Selection Guide

### "I need a database for..."

| Use Case | Recommendation |
|----------|----------------|
| Web app with complex queries | PostgreSQL |
| High-write time-series | Cassandra or TimescaleDB |
| Flexible schema, rapid dev | MongoDB |
| Caching layer | Redis |
| Full-text search | Elasticsearch |
| Graph relationships | Neo4j |
| Serverless application | DynamoDB |

### "I need a message queue for..."

| Use Case | Recommendation |
|----------|----------------|
| Event streaming, replay | Kafka |
| Task queues, routing | RabbitMQ |
| Simple, managed | SQS |
| Real-time notifications | Redis Pub/Sub |

### "I need to cache..."

| Use Case | Recommendation |
|----------|----------------|
| Session data | Redis |
| Simple key-value | Redis or Memcached |
| Full pages | Varnish or CDN |
| Static assets | CDN |
| Database queries | Redis |

## Key Takeaways

1. **No perfect choice** - Every technology has trade-offs
2. **Match to requirements** - Scale, consistency, team expertise
3. **Start simple** - Don't over-engineer
4. **Consider operations** - Managed services reduce burden
5. **Team expertise matters** - Known tech often beats "perfect" tech
