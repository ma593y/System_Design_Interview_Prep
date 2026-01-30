# Availability Strategies

## Table of Contents
1. [Availability Definitions](#1-availability-definitions)
2. [SLA, SLO, and SLI](#2-sla-slo-and-sli)
3. [Redundancy Patterns](#3-redundancy-patterns)
4. [Failover Strategies](#4-failover-strategies)
5. [Disaster Recovery](#5-disaster-recovery)
6. [Multi-Region Deployment](#6-multi-region-deployment)
7. [Health Checks and Heartbeats](#7-health-checks-and-heartbeats)
8. [Graceful Degradation Patterns](#8-graceful-degradation-patterns)
9. [Data Replication for Availability](#9-data-replication-for-availability)
10. [DNS Failover and Global Load Balancing](#10-dns-failover-and-global-load-balancing)
11. [Real-World Examples](#11-real-world-examples)
12. [Interview Talking Points](#12-interview-talking-points)

---

## 1. Availability Definitions

Availability measures the proportion of time a system is operational and accessible. It is
typically expressed as a percentage of uptime over a given period.

```
                    Uptime
Availability = ─────────────────────
                Uptime + Downtime
```

### The "Nines" Table

| Availability | Downtime/Year   | Downtime/Month | Downtime/Week  | Common Name     |
|-------------|-----------------|----------------|----------------|-----------------|
| 99%         | 3.65 days       | 7.31 hours     | 1.68 hours     | Two nines       |
| 99.9%       | 8.77 hours      | 43.83 minutes  | 10.08 minutes  | Three nines     |
| 99.95%      | 4.38 hours      | 21.92 minutes  | 5.04 minutes   | Three and a half|
| 99.99%      | 52.60 minutes   | 4.38 minutes   | 1.01 minutes   | Four nines      |
| 99.999%     | 5.26 minutes    | 26.30 seconds  | 6.05 seconds   | Five nines      |
| 99.9999%    | 31.56 seconds   | 2.63 seconds   | 0.60 seconds   | Six nines       |

### Key Concepts

- **MTBF (Mean Time Between Failures):** Average time between system breakdowns.
- **MTTR (Mean Time To Recovery):** Average time to restore service after failure.
- **MTTD (Mean Time To Detect):** Average time to detect an issue.

```
                    MTBF
Availability = ─────────────────
                MTBF + MTTR
```

Improving availability means either increasing MTBF (fewer failures) or decreasing MTTR
(faster recovery).

---

## 2. SLA, SLO, and SLI

### Hierarchy

```
  +─────────────────────────────────────────────────────────+
  |  SLA (Service Level Agreement)                          |
  |  External contract with customers                       |
  |  "We guarantee 99.9% uptime or we credit your account" |
  |                                                         |
  |  +──────────────────────────────────────────────────+   |
  |  |  SLO (Service Level Objective)                   |   |
  |  |  Internal target, stricter than SLA              |   |
  |  |  "We target 99.95% uptime internally"            |   |
  |  |                                                  |   |
  |  |  +──────────────────────────────────────────+    |   |
  |  |  |  SLI (Service Level Indicator)           |    |   |
  |  |  |  Actual measured metric                  |    |   |
  |  |  |  "Our uptime was 99.97% this month"      |    |   |
  |  |  +──────────────────────────────────────────+    |   |
  |  +──────────────────────────────────────────────────+   |
  +─────────────────────────────────────────────────────────+
```

### SLI Examples

| Service Type    | Common SLIs                                    |
|----------------|------------------------------------------------|
| Web Application | Request latency (p50, p95, p99), Error rate    |
| Storage System  | Durability, Read/write latency, Throughput     |
| Data Pipeline   | Freshness, Correctness, Throughput             |
| API Service     | Availability, Latency, Error rate              |

### Error Budgets

An error budget is the allowed amount of unreliability based on the SLO. If the SLO is
99.9%, the error budget is 0.1% of total time.

```
Error Budget = 1 - SLO

Example: SLO = 99.9%
  Error Budget = 0.1%
  Monthly budget = 30 days x 24 hours x 60 minutes x 0.001 = 43.2 minutes
```

When the error budget is exhausted, teams should freeze feature deployments and focus on
reliability improvements.

---

## 3. Redundancy Patterns

### Active-Active

Both nodes handle traffic simultaneously. Offers highest throughput and availability.

```
         +──────────+
         |  Client  |
         +────┬─────+
              |
        +─────┴──────+
        | Load       |
        | Balancer   |
        +──┬─────┬───+
           |     |
     +─────┴──+  +──┴─────+
     | Node A |  | Node B  |
     | ACTIVE |  | ACTIVE  |
     +────┬───+  +───┬────+
          |          |
     +────┴──────────┴────+
     |  Shared Database   |
     +────────────────────+
```

**Pros:** Full utilization of all resources, zero-downtime failover, higher throughput.
**Cons:** Complex data synchronization, potential split-brain, higher cost of consistency.

### Active-Passive

One node handles all traffic; the standby takes over on failure.

```
         +──────────+
         |  Client  |
         +────┬─────+
              |
        +─────┴──────+
        | Load       |
        | Balancer   |
        +──┬─────────+
           |
     +─────┴──+     +──────────+
     | Node A |---->| Node B   |
     | ACTIVE |     | PASSIVE  |
     +────┬───+     +───┬──────+
          |             | (replication)
     +────┴─────────────┴────+
     |       Database        |
     +───────────────────────+
```

**Pros:** Simpler consistency model, lower cost, no split-brain risk.
**Cons:** Standby resources idle, failover delay, resource waste.

### N+1 Redundancy

Run N nodes to handle load plus 1 spare for redundancy. If you need 3 nodes to handle
peak traffic, run 4.

```
  Normal Load: 3 nodes needed

  +────────+  +────────+  +────────+  +────────+
  | Node 1 |  | Node 2 |  | Node 3 |  | Node 4 |
  | ACTIVE |  | ACTIVE |  | ACTIVE |  | SPARE  |
  +────────+  +────────+  +────────+  +────────+

  After Node 2 fails:

  +────────+  +────────+  +────────+  +────────+
  | Node 1 |  | Node 2 |  | Node 3 |  | Node 4 |
  | ACTIVE |  | FAILED |  | ACTIVE |  | ACTIVE |
  +────────+  +────────+  +────────+  +────────+
```

### N+2 / 2N Redundancy

- **N+2:** Two spare nodes for extra safety.
- **2N:** Fully doubled capacity. Every component has a complete standby replica. Used for
  mission-critical systems (financial trading, air traffic control).

---

## 4. Failover Strategies

### Hot Standby

The standby is running, fully synchronized, and ready to take over immediately.

```
  +──────────────────────────────────────────+
  |          HOT STANDBY                     |
  |                                          |
  |  Primary ──[sync replication]──> Standby |
  |  (serving)                    (ready)    |
  |                                          |
  |  Failover time: seconds                  |
  |  Data loss: none (synchronous)           |
  |  Cost: HIGH (full duplicate)             |
  +──────────────────────────────────────────+
```

### Warm Standby

The standby is running but may be slightly behind. Uses async replication.

```
  +──────────────────────────────────────────+
  |          WARM STANDBY                    |
  |                                          |
  |  Primary ──[async replication]──> Standby|
  |  (serving)                 (lagging)     |
  |                                          |
  |  Failover time: minutes                  |
  |  Data loss: possible (replication lag)   |
  |  Cost: MEDIUM                            |
  +──────────────────────────────────────────+
```

### Cold Standby

The standby is not running. Must be booted and data restored from backups.

```
  +──────────────────────────────────────────+
  |          COLD STANDBY                    |
  |                                          |
  |  Primary ──[periodic backup]──> Storage  |
  |  (serving)                  (archived)   |
  |                                          |
  |  Failover time: hours                    |
  |  Data loss: since last backup            |
  |  Cost: LOW                               |
  +──────────────────────────────────────────+
```

### Comparison

| Strategy     | Failover Time | Data Loss Risk | Cost    | Use Case                 |
|-------------|---------------|----------------|---------|--------------------------|
| Hot Standby  | Seconds       | None           | High    | Financial, e-commerce    |
| Warm Standby | Minutes       | Minimal        | Medium  | Enterprise applications  |
| Cold Standby | Hours         | Moderate       | Low     | Dev/test, non-critical   |

---

## 5. Disaster Recovery

### RPO and RTO

```
  Timeline of a Disaster:

  Last Good     Disaster      Detection     Recovery
  Backup        Occurs        Complete      Complete
    |              |              |              |
    v              v              v              v
  ──+──────────────+──────────────+──────────────+──────>
    |<── RPO ─────>|              |              |
    |              |<─────────── RTO ───────────>|

  RPO = Recovery Point Objective
        Maximum acceptable data loss measured in time
        "How much data can we afford to lose?"

  RTO = Recovery Time Objective
        Maximum acceptable downtime
        "How fast must we recover?"
```

### DR Tier Classification

| Tier | Strategy               | RPO         | RTO          | Cost     |
|------|------------------------|-------------|--------------|----------|
| 0    | No DR plan             | N/A         | N/A          | None     |
| 1    | Backup & restore       | 24 hours    | Days         | Low      |
| 2    | Pilot light            | Hours       | Hours        | Low-Med  |
| 3    | Warm standby           | Minutes     | Minutes      | Medium   |
| 4    | Multi-site active      | Near zero   | Near zero    | High     |

### Backup Strategies

```
  Full Backup (Sunday):
  [████████████████████] Complete copy

  Incremental (Mon-Sat):
  Mon: [██]           Changes since Sunday
  Tue: [███]          Changes since Monday
  Wed: [█]            Changes since Tuesday
  Thu: [████]         Changes since Wednesday

  Differential (Mon-Sat):
  Mon: [██]           Changes since Sunday
  Tue: [████]         Changes since Sunday
  Wed: [█████]        Changes since Sunday
  Thu: [████████]     Changes since Sunday
```

**Full:** Complete copy each time. Slowest backup, fastest restore.
**Incremental:** Only changes since last backup. Fastest backup, slowest restore.
**Differential:** All changes since last full backup. Balance of backup/restore speed.

### The 3-2-1 Rule

- **3** copies of data
- **2** different storage media
- **1** offsite copy

---

## 6. Multi-Region Deployment

### Active-Active Multi-Region

All regions serve traffic simultaneously.

```
          +──────────────+
          |   Global     |
          |   DNS/LB     |
          +──┬────────┬──+
             |        |
     +───────┴──+  +──┴────────+
     | Region A |  | Region B  |
     | US-East  |  | EU-West   |
     |          |  |           |
     | +──────+ |  | +──────+  |
     | | App  | |  | | App  |  |
     | +──┬───+ |  | +──┬───+  |
     |    |      |  |    |      |
     | +──┴───+ |  | +──┴───+  |
     | | DB    |<──>| | DB    | |
     | +──────+ |  | +──────+  |
     +──────────+  +──────────+
          Bi-directional replication
```

**Challenges:**
- Data consistency across regions
- Conflict resolution for concurrent writes
- Cross-region latency for synchronous operations
- Complex deployment and testing

### Active-Passive Multi-Region

One primary region handles writes; secondary handles reads or stays on standby.

```
     +───────────+            +──────────────+
     | Region A  |  async     | Region B     |
     | (PRIMARY) | ────────>  | (STANDBY)    |
     |           |  replicate |              |
     | Reads +   |            | Read-only or |
     | Writes    |            | Idle         |
     +───────────+            +-─────────────+
```

**Simpler** but higher RTO because failover requires DNS updates and promotion.

### Data Strategies for Multi-Region

| Strategy                | Consistency    | Latency     | Complexity |
|------------------------|----------------|-------------|------------|
| Synchronous replication | Strong         | High        | Medium     |
| Async replication       | Eventual       | Low         | Medium     |
| Conflict-free (CRDTs)  | Eventual       | Low         | High       |
| Region-local writes    | Strong (local) | Low         | Low        |

---

## 7. Health Checks and Heartbeats

### Health Check Types

```
  +---------------------------------------------------------+
  |  LIVENESS CHECK                                         |
  |  "Is the process alive?"                                |
  |  Checks: Process running, not deadlocked               |
  |  Failure action: Restart the container/process          |
  +---------------------------------------------------------+

  +---------------------------------------------------------+
  |  READINESS CHECK                                        |
  |  "Can it handle requests?"                              |
  |  Checks: Dependencies up, warmup complete, pool ready   |
  |  Failure action: Stop sending traffic (don't restart)   |
  +---------------------------------------------------------+

  +---------------------------------------------------------+
  |  STARTUP CHECK                                          |
  |  "Has initialization completed?"                        |
  |  Checks: Config loaded, caches warmed, migrations done  |
  |  Failure action: Wait, then restart if timeout exceeded  |
  +---------------------------------------------------------+
```

### Health Check Implementation

```
  GET /health/live    -> 200 OK (process alive)
  GET /health/ready   -> 200 OK (ready to serve)
  GET /health/startup -> 200 OK (initialization complete)

  Deep Health Check Response:
  {
    "status": "healthy",
    "checks": {
      "database": { "status": "up", "latency_ms": 5 },
      "cache":    { "status": "up", "latency_ms": 1 },
      "queue":    { "status": "up", "latency_ms": 3 },
      "disk":     { "status": "up", "free_gb": 45.2 }
    },
    "uptime_seconds": 86400
  }
```

### Heartbeat Pattern

```
  Service A                    Monitoring System
     |                               |
     |── heartbeat (I'm alive) ────>|
     |                               | (records timestamp)
     |       ... 10 seconds ...      |
     |── heartbeat (I'm alive) ────>|
     |                               | (records timestamp)
     |       ... 30 seconds ...      |
     |                               | (no heartbeat!)
     |                               | --> ALERT: Service A
     |                               |     missed 2 heartbeats
```

**Best practice:** Use a heartbeat interval of T, and alert after missing 3T. This avoids
false positives from network blips.

---

## 8. Graceful Degradation Patterns

### Circuit Breaker

```
                     Success
              +────────────────+
              |                |
              v                |
         +────────+     +─────┴────+     +──────────+
    ────>| CLOSED |────>|   OPEN   |────>| HALF-OPEN|
         +────────+     +──────────+     +──────────+
          (normal)      (failing)        (testing)
              ^          Fail fast          |
              |                            |
              +───── Success ──────────────+

  CLOSED:    Requests flow normally. Track failure count.
  OPEN:      Requests fail immediately. Timer starts.
  HALF-OPEN: Allow limited requests to test recovery.
```

### Bulkhead Pattern

Isolate components so one failure does not cascade to others.

```
  +─────────────────────────────────────────+
  |              Application                |
  |                                         |
  |  +──────────+  +──────────+  +────────+ |
  |  | Payment  |  | Search   |  | User   | |
  |  | Pool:10  |  | Pool:20  |  | Pool:5 | |
  |  | threads  |  | threads  |  | threads| |
  |  +──────────+  +──────────+  +────────+ |
  |                                         |
  |  If Search pool exhausted,              |
  |  Payment and User still work            |
  +─────────────────────────────────────────+
```

### Fallback Strategies

| Strategy            | Description                                   | Example                        |
|--------------------|-----------------------------------------------|--------------------------------|
| Cached response    | Return stale cached data                       | Show cached product catalog    |
| Default value      | Return a safe default                          | Default recommendations list   |
| Reduced feature    | Disable non-essential features                 | Disable reviews, keep checkout |
| Queue for later    | Accept request, process when recovered         | Queue order, confirm later     |
| Static fallback    | Serve static content                           | Static homepage during outage  |

### Load Shedding

When overloaded, intentionally drop low-priority requests to protect critical ones.

```
  Incoming Requests ──> [Priority Classifier]
                              |
                +─────────────┼──────────────+
                |             |              |
           +────┴───+   +────┴───+   +──────┴──+
           | HIGH   |   | MEDIUM |   | LOW     |
           | Always |   | Shed   |   | Shed    |
           | serve  |   | at 80% |   | at 60%  |
           +────────+   +────────+   +─────────+
```

---

## 9. Data Replication for Availability

### Synchronous Replication

```
  Client ──> Primary ──> Replica (wait for ACK) ──> Respond to Client

  Timeline:
  Client    Primary    Replica
    |──Write──>|          |
    |          |──Write──>|
    |          |<──ACK────|
    |<──ACK───|          |
```

**Guarantees:** Zero data loss on failover.
**Tradeoff:** Higher write latency, limited by slowest replica.

### Asynchronous Replication

```
  Client ──> Primary ──> Respond to Client (then replicate)

  Timeline:
  Client    Primary    Replica
    |──Write──>|          |
    |<──ACK───|          |
    |          |──Write──>|  (background)
    |          |<──ACK────|
```

**Guarantees:** Lower latency, higher throughput.
**Tradeoff:** Possible data loss on primary failure (replication lag).

### Semi-Synchronous Replication

Write to primary and at least one replica synchronously. Remaining replicas are async.

```
  Client    Primary    Replica1    Replica2
    |──Write──>|          |           |
    |          |──Write──>|           |
    |          |<──ACK────|           |
    |<──ACK───|          |           |
    |          |────────Write────────>| (async)
```

### Replication Topologies

```
  Single-Leader:          Multi-Leader:         Leaderless:

    Primary               Primary A             Node A
    / | \                  /    \               / | \
   R1 R2 R3           Prim B  Prim C        N-B  N-C  N-D
                        |       |            (quorum reads
                       R1      R2             and writes)
```

---

## 10. DNS Failover and Global Load Balancing

### DNS-Based Failover

```
  +────────────+     +───────────+
  |   Client   |────>| DNS       |
  +────────────+     | Server    |
                     +─────┬─────+
                           |
              Health check determines response:
                           |
           +───────────────┼───────────────+
           |                               |
    +──────┴──────+                +───────┴──────+
    | Primary DC  |                | Secondary DC |
    | 203.0.1.10  |                | 203.0.2.10   |
    +─────────────+                +──────────────+

  Normal: DNS resolves to 203.0.1.10 (Primary)
  Failure: DNS resolves to 203.0.2.10 (Secondary)
```

**Limitation:** DNS TTL means clients may cache the old IP for minutes.

### Global Server Load Balancing (GSLB)

```
  User in Tokyo                    User in London
       |                                |
       v                                v
  +──────────+                    +──────────+
  |   GSLB   |                   |   GSLB   |
  +────┬─────+                   +─────┬────+
       |                               |
  +────┴───────+               +───────┴────+
  | Tokyo DC   |               | London DC  |
  | (closest)  |               | (closest)  |
  +────────────+               +────────────+
```

GSLB considers:
- **Geographic proximity:** Route to nearest datacenter.
- **Health status:** Skip unhealthy endpoints.
- **Load:** Route to least-loaded datacenter.
- **Latency:** Route based on real-time latency measurements.

### Anycast

Multiple servers share the same IP address. BGP routing directs traffic to the nearest one.

```
  All servers announce IP: 1.2.3.4

  Server A (US)  ──┐
  Server B (EU)  ──┼── BGP routes to nearest
  Server C (Asia)──┘
```

Used by CDNs (Cloudflare) and DNS services.

---

## 11. Real-World Examples

### Netflix Availability Architecture

```
  +─────────────────────────────────────────────+
  |           Netflix Architecture              |
  |                                             |
  |  Multi-Region Active-Active (US + EU)       |
  |                                             |
  |  +─────────+  +──────────+  +───────────+  |
  |  | Zuul    |  | Eureka   |  | Hystrix   |  |
  |  | Gateway |  | Service  |  | Circuit   |  |
  |  |         |  | Registry |  | Breaker   |  |
  |  +─────────+  +──────────+  +───────────+  |
  |                                             |
  |  +─────────+  +──────────+  +───────────+  |
  |  | EVCache |  | Cassandra|  | S3        |  |
  |  | (cache) |  | (multi-  |  | (content) |  |
  |  |         |  |  region) |  |           |  |
  |  +─────────+  +──────────+  +───────────+  |
  |                                             |
  |  Chaos Engineering: Simian Army             |
  |  - Chaos Monkey: kills random instances     |
  |  - Chaos Kong: simulates region failure     |
  |  - Latency Monkey: injects delays           |
  +─────────────────────────────────────────────+
```

### AWS Multi-AZ Architecture

```
  +───────────────── AWS Region ──────────────────+
  |                                               |
  |  +──── AZ-a ────+     +──── AZ-b ────+       |
  |  |               |     |               |       |
  |  | +───────────+ |     | +───────────+ |       |
  |  | | EC2 / ECS | |     | | EC2 / ECS | |       |
  |  | +───────────+ |     | +───────────+ |       |
  |  |               |     |               |       |
  |  | +───────────+ |     | +───────────+ |       |
  |  | | RDS       | |     | | RDS       | |       |
  |  | | Primary   |─┼─────┼>| Standby   | |       |
  |  | +───────────+ |     | +───────────+ |       |
  |  |               |     |               |       |
  |  +───────────────+     +───────────────+       |
  |                                               |
  |  ALB distributes across AZs                   |
  |  RDS auto-failover in ~60 seconds             |
  |  S3 automatically replicates across AZs       |
  +───────────────────────────────────────────────+
```

---

## 12. Interview Talking Points

### When to Bring Up Availability

- When discussing any production system requirement
- When asked "What happens if X fails?"
- When defining non-functional requirements at the start

### Key Phrases to Use

1. "For this system, I would target 99.9% availability, giving us about 43 minutes of
   monthly downtime budget."
2. "I would deploy active-active across two availability zones with automatic failover."
3. "To reduce MTTR, I would implement comprehensive health checks and automated recovery."
4. "For disaster recovery, I would set an RPO of 1 hour and RTO of 15 minutes using warm
   standby in a secondary region."

### Common Mistakes to Avoid

| Mistake                                | Better Approach                              |
|---------------------------------------|----------------------------------------------|
| Claiming 100% availability            | State realistic target with error budget     |
| Ignoring failure scenarios            | Proactively discuss what happens when X fails|
| Single point of failure in design     | Add redundancy for every critical component  |
| Over-engineering availability         | Match availability to business requirements  |
| Forgetting DNS/network layer          | Include DNS failover and GSLB                |

### Sample Interview Dialogue

**Interviewer:** "How would you ensure high availability for this service?"

**Strong answer:** "First, I would define the availability target based on the business need.
For a payment system, we might need four nines. I would deploy across multiple AZs with an
active-active setup behind an ALB. The database would use synchronous replication to a standby
in another AZ with automated failover. I would implement circuit breakers to prevent cascade
failures, and have health checks at every layer. For disaster recovery, I would maintain a
warm standby in another region with async replication, giving us an RPO of about 5 minutes and
RTO of about 15 minutes. Finally, I would run regular chaos engineering exercises to validate
our failover mechanisms work as expected."
