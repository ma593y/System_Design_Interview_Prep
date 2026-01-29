# Acronyms & Glossary

## Common Acronyms

### Architecture & Design

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **API** | Application Programming Interface | Contract for how software components communicate |
| **REST** | Representational State Transfer | Architectural style for web APIs |
| **SOA** | Service-Oriented Architecture | Design pattern using loosely coupled services |
| **CRUD** | Create, Read, Update, Delete | Basic data operations |
| **DDD** | Domain-Driven Design | Software design based on business domain |
| **MVC** | Model-View-Controller | Architectural pattern separating concerns |
| **CQRS** | Command Query Responsibility Segregation | Separate read and write models |
| **EDA** | Event-Driven Architecture | Architecture based on event production/consumption |

### Databases

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **SQL** | Structured Query Language | Language for relational databases |
| **NoSQL** | Not Only SQL | Non-relational databases |
| **ACID** | Atomicity, Consistency, Isolation, Durability | Transaction properties |
| **BASE** | Basically Available, Soft state, Eventually consistent | NoSQL consistency model |
| **OLTP** | Online Transaction Processing | Real-time transactional systems |
| **OLAP** | Online Analytical Processing | Analytical/reporting systems |
| **ORM** | Object-Relational Mapping | Maps objects to database tables |
| **DDL** | Data Definition Language | Schema definition commands |
| **DML** | Data Manipulation Language | Data modification commands |

### Distributed Systems

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **CAP** | Consistency, Availability, Partition tolerance | Distributed systems theorem |
| **RPC** | Remote Procedure Call | Calling functions on remote servers |
| **gRPC** | Google Remote Procedure Call | High-performance RPC framework |
| **2PC** | Two-Phase Commit | Distributed transaction protocol |
| **RAFT** | (Not an acronym) | Consensus algorithm |
| **CDC** | Change Data Capture | Tracking database changes |
| **LSM** | Log-Structured Merge tree | Write-optimized data structure |

### Networking

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **TCP** | Transmission Control Protocol | Reliable transport protocol |
| **UDP** | User Datagram Protocol | Fast, unreliable transport |
| **HTTP** | Hypertext Transfer Protocol | Web protocol |
| **HTTPS** | HTTP Secure | Encrypted HTTP |
| **DNS** | Domain Name System | Translates domains to IPs |
| **CDN** | Content Delivery Network | Distributed content caching |
| **SSL/TLS** | Secure Sockets Layer / Transport Layer Security | Encryption protocols |
| **IP** | Internet Protocol | Network layer protocol |
| **LB** | Load Balancer | Distributes traffic across servers |
| **VPN** | Virtual Private Network | Secure network tunnel |

### Performance & Metrics

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **QPS** | Queries Per Second | Request rate metric |
| **TPS** | Transactions Per Second | Transaction rate metric |
| **RPS** | Requests Per Second | Same as QPS |
| **SLA** | Service Level Agreement | Contractual performance guarantee |
| **SLO** | Service Level Objective | Internal performance target |
| **SLI** | Service Level Indicator | Metric used to measure SLO |
| **MTBF** | Mean Time Between Failures | Reliability metric |
| **MTTR** | Mean Time To Recovery | Recovery speed metric |
| **P99** | 99th Percentile | Latency that 99% of requests meet |
| **TTL** | Time To Live | Expiration time |

### DevOps & Infrastructure

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **CI/CD** | Continuous Integration/Deployment | Automated build and deploy |
| **K8s** | Kubernetes | Container orchestration |
| **VM** | Virtual Machine | Virtualized computer |
| **IaC** | Infrastructure as Code | Managing infra via code |
| **APM** | Application Performance Monitoring | Performance tracking tools |
| **SRE** | Site Reliability Engineering | Operations discipline |

### Security

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **OAuth** | Open Authorization | Authorization framework |
| **JWT** | JSON Web Token | Token-based authentication |
| **SSO** | Single Sign-On | One login for multiple apps |
| **RBAC** | Role-Based Access Control | Permission model |
| **MFA/2FA** | Multi/Two-Factor Authentication | Additional auth factors |
| **DDoS** | Distributed Denial of Service | Attack flooding servers |

### Cloud & Storage

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| **AWS** | Amazon Web Services | Cloud provider |
| **GCP** | Google Cloud Platform | Cloud provider |
| **S3** | Simple Storage Service | AWS object storage |
| **EC2** | Elastic Compute Cloud | AWS virtual machines |
| **RDS** | Relational Database Service | AWS managed databases |
| **EBS** | Elastic Block Store | AWS block storage |
| **SSD** | Solid State Drive | Fast storage |
| **HDD** | Hard Disk Drive | Traditional storage |
| **IOPS** | Input/Output Operations Per Second | Storage performance metric |

## Glossary of Terms

### A

**Availability** - Percentage of time a system is operational and accessible.

**Asynchronous** - Operations that don't block waiting for completion.

**Auto-scaling** - Automatically adjusting resources based on demand.

### B

**Backpressure** - Mechanism to slow producers when consumers can't keep up.

**Bloom Filter** - Probabilistic data structure for set membership testing.

**Bulkhead** - Pattern isolating failures to prevent cascade.

### C

**Cache** - Fast storage layer for frequently accessed data.

**Caching Aside** - Cache pattern where application manages cache.

**Circuit Breaker** - Pattern preventing calls to failing services.

**Consistent Hashing** - Hashing that minimizes redistribution when nodes change.

**Containerization** - Packaging applications with dependencies.

### D

**Data Lake** - Repository storing structured and unstructured data.

**Data Warehouse** - Central repository for analytical data.

**Denormalization** - Adding redundant data for read performance.

**Distributed System** - System with components on networked computers.

### E

**Elasticity** - Ability to scale resources up and down.

**Event Sourcing** - Storing state as sequence of events.

**Eventually Consistent** - Data will become consistent over time.

### F

**Failover** - Switching to backup system on failure.

**Fan-out** - Distributing data/work to multiple destinations.

**Fault Tolerance** - Ability to continue operating despite failures.

### H

**Horizontal Scaling** - Adding more machines (scale out).

**Hot Spot** - Unevenly loaded server/partition.

### I

**Idempotent** - Operation producing same result regardless of repetition.

**Index** - Data structure improving query performance.

### L

**Latency** - Time to complete an operation.

**Leader Election** - Process of choosing a coordinator node.

**Load Balancing** - Distributing work across multiple servers.

### M

**Message Broker** - Intermediary for message passing.

**Microservices** - Architecture using small, independent services.

**Middleware** - Software connecting components/applications.

### N

**Normalization** - Organizing data to reduce redundancy.

**Node** - Individual server or instance in a cluster.

### P

**Partition** - Division of data across multiple storage units.

**Partition Tolerance** - Ability to operate during network splits.

**Primary/Replica** - Leader/follower database replication model.

### Q

**Quorum** - Minimum nodes needed for consensus.

**Queue** - Data structure for ordered message passing.

### R

**Replication** - Copying data across multiple nodes.

**Rate Limiting** - Controlling request frequency.

**Redundancy** - Duplication for fault tolerance.

### S

**Saga** - Pattern for distributed transactions using compensation.

**Scalability** - Ability to handle growth in load.

**Sharding** - Horizontal partitioning of data.

**Stateless** - Service not storing client state between requests.

**Strong Consistency** - All reads see most recent write.

### T

**Throughput** - Operations completed per time unit.

**Tombstone** - Marker for deleted data (in distributed systems).

### V

**Vertical Scaling** - Adding more resources to a machine (scale up).

**Virtual IP (VIP)** - IP address not tied to specific hardware.

### W

**Write-Ahead Log (WAL)** - Log writes before applying to database.

**Write-Through** - Cache pattern writing to both cache and database.

## Quick Reference Cards

### The "Nines" of Availability

| Availability | Downtime/Year | Downtime/Month |
|--------------|---------------|----------------|
| 99% | 3.65 days | 7.2 hours |
| 99.9% | 8.76 hours | 43.8 minutes |
| 99.99% | 52.6 minutes | 4.38 minutes |
| 99.999% | 5.26 minutes | 26.3 seconds |

### CAP Theorem Quick Reference

```
During Network Partition:
- CP: Choose Consistency (reject operations)
- AP: Choose Availability (allow stale reads)
- CA: Not possible during partition
```

### ACID vs BASE

```
ACID (SQL):           BASE (NoSQL):
- Atomicity           - Basically Available
- Consistency         - Soft state
- Isolation           - Eventually consistent
- Durability
```

## Key Takeaways

1. **Know the acronyms** - They're used constantly in interviews
2. **Understand the concepts** - Don't just memorize definitions
3. **Context matters** - Same term can mean different things
4. **Keep learning** - New terms emerge regularly
