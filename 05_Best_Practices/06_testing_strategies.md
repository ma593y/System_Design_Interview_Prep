# Testing Strategies for Distributed Systems

## Table of Contents
1. [Testing Pyramid](#1-testing-pyramid)
2. [Load Testing](#2-load-testing)
3. [Stress Testing and Breaking Point Analysis](#3-stress-testing-and-breaking-point-analysis)
4. [Chaos Engineering](#4-chaos-engineering)
5. [Fault Injection Testing](#5-fault-injection-testing)
6. [Contract Testing for Microservices](#6-contract-testing-for-microservices)
7. [Canary Testing and Progressive Rollouts](#7-canary-testing-and-progressive-rollouts)
8. [A/B Testing Infrastructure](#8-ab-testing-infrastructure)
9. [Shadow Testing / Dark Launching](#9-shadow-testing--dark-launching)
10. [Data Migration Testing](#10-data-migration-testing)
11. [Performance Regression Testing](#11-performance-regression-testing)
12. [Disaster Recovery Testing](#12-disaster-recovery-testing)
13. [Interview Relevance](#13-interview-relevance)

---

## 1. Testing Pyramid

```
                    /\
                   /  \           End-to-End Tests
                  / E2E\          - Few, slow, expensive
                 / Tests \        - Test full user flows
                /──────────\      - Catch integration issues
               /            \
              / Integration  \    Integration Tests
             /    Tests       \   - Moderate number
            /──────────────────\  - Test component interactions
           /                    \ - API contracts, DB queries
          /     Unit Tests       \
         /       (Most)           \  Unit Tests
        /──────────────────────────\ - Many, fast, cheap
                                     - Test individual functions
                                     - Pure logic, no I/O
```

### Test Distribution

| Level         | Count    | Speed      | Cost    | Scope              |
|--------------|----------|------------|---------|---------------------|
| Unit         | Hundreds | Milliseconds| Low    | Single function     |
| Integration  | Dozens   | Seconds    | Medium  | Component boundary  |
| E2E          | Few      | Minutes    | High    | Full system path    |

### What Each Level Tests in Distributed Systems

```
  Unit Tests:
  - Business logic (calculate price, validate input)
  - Serialization/deserialization
  - Algorithm correctness
  - Error handling paths

  Integration Tests:
  - Database queries with real DB (testcontainers)
  - API endpoint request/response
  - Message queue publish/consume
  - Cache read/write operations
  - External API client behavior

  End-to-End Tests:
  - User registration -> email verification -> login
  - Place order -> payment -> inventory update -> notification
  - Search -> filter -> add to cart -> checkout
```

---

## 2. Load Testing

### Load Testing Process

```
  Phase 1: Define          Phase 2: Script         Phase 3: Execute
  +--------------+         +--------------+         +--------------+
  | Target RPS   |   -->   | Write test   |   -->   | Ramp up      |
  | User journeys|         | scenarios    |         | gradually    |
  | Success      |         | Parameterize |         | Monitor      |
  | criteria     |         | data         |         | all metrics  |
  +--------------+         +--------------+         +--------------+

  Phase 4: Analyze         Phase 5: Optimize        Phase 6: Repeat
  +--------------+         +--------------+         +--------------+
  | Review       |   -->   | Fix          |   -->   | Validate     |
  | percentiles  |         | bottlenecks  |         | improvements |
  | Find         |         | Tune configs |         | Regression   |
  | bottlenecks  |         | Scale up/out |         | baseline     |
  +--------------+         +--------------+         +--------------+
```

### Load Test Scenario Example (k6)

```
  Stages:
  +─────────────────────────────────────────────────────+
  |                                                     |
  | Users                    ___________                |
  |   500 |                 /           \               |
  |       |               /             \              |
  |   300 |    __________/               \             |
  |       |   /                           \            |
  |   100 |  /                             \           |
  |       | /                               \          |
  |     0 |/                                 \_______  |
  |       +────────────────────────────────────────── |
  |       0   5min  10min  15min  20min  25min  30min |
  |       Ramp  Hold   Ramp    Peak   Ramp    Cool   |
  |        up   steady  up     Hold    down   down    |
  +─────────────────────────────────────────────────────+
```

### Tool Comparison

| Tool    | Language   | Protocol       | Distributed | Cloud Option  | Learning Curve |
|---------|-----------|----------------|-------------|---------------|---------------|
| k6      | JavaScript| HTTP, WS, gRPC | Yes         | Grafana Cloud | Low           |
| JMeter  | Java/GUI  | HTTP, JDBC, FTP| Yes         | BlazeMeter    | Medium        |
| Locust  | Python    | HTTP, custom   | Yes         | Locust Cloud  | Low           |
| Gatling | Scala     | HTTP, WS       | Yes         | Gatling Ent.  | Medium        |
| Artillery| YAML/JS  | HTTP, WS       | Yes         | Artillery.io  | Low           |

### Key Metrics to Monitor During Load Tests

```
  +──────────────────────────────────────────────+
  | During load test, watch for:                  |
  |                                               |
  | Response Time:                                |
  |   p50 < 100ms                                 |
  |   p95 < 300ms                                 |
  |   p99 < 1000ms                                |
  |                                               |
  | Throughput:                                   |
  |   Requests/sec matches target                 |
  |   No plateauing (sign of bottleneck)          |
  |                                               |
  | Error Rate:                                   |
  |   < 0.1% under normal load                    |
  |   < 1% under peak load                        |
  |                                               |
  | Resources:                                    |
  |   CPU < 80%                                   |
  |   Memory < 85%                                |
  |   DB connections < 80% of pool                |
  |   Network bandwidth < 70%                     |
  +──────────────────────────────────────────────+
```

---

## 3. Stress Testing and Breaking Point Analysis

### Stress Test vs Load Test

```
  Load
   |
   |                     Stress Zone
   |                    /
   |              _____/ Breaking
   |             /       Point!
   |     _______/         X
   |    /                 |
   |   / Load Test Zone   |
   |  /                   |
   | /                    |
   |/_____________________|______________> Time
   Normal     Peak    Overload   Failure

  Load Test: Can we handle expected peak?
  Stress Test: At what point does the system break?
```

### Breaking Point Analysis

```
  Throughput
     |          ___________
     |         /           \
     |        /             \  <-- Throughput drops
     |       /               \     after breaking point
     |      /                 \
     |     /                   \
     |    /                     \____
     |   /
     |  /
     +────────────────────────────────> Concurrent Users
     0   100   500   1000  2000  5000

  Findings:
  - Linear scaling up to 1000 users
  - Breaking point at 2000 users
  - Root cause: Database connection pool exhausted
  - Action: Increase pool from 50 to 200, add read replica
```

### What to Observe at Breaking Point

| Symptom                    | Likely Cause                        |
|---------------------------|-------------------------------------|
| Latency spike             | CPU saturation, lock contention     |
| Errors spike (500s)       | Memory exhaustion, OOM kills        |
| Throughput plateau         | Connection pool exhausted            |
| Cascading failures        | No circuit breakers, no timeouts    |
| Unresponsive health checks| Thread pool exhaustion              |

---

## 4. Chaos Engineering

### Principles of Chaos Engineering

```
  +──────────────────────────────────────────────────+
  | CHAOS ENGINEERING PROCESS                         |
  |                                                   |
  | 1. Define "steady state" (normal behavior)        |
  |    - p99 latency < 200ms                          |
  |    - Error rate < 0.1%                            |
  |    - Orders processed > 100/min                   |
  |                                                   |
  | 2. Hypothesize that steady state will continue    |
  |    - "If we kill 1 instance, the system           |
  |      continues operating within SLO"              |
  |                                                   |
  | 3. Introduce real-world events                    |
  |    - Kill instances, inject latency, fill disk    |
  |                                                   |
  | 4. Disprove the hypothesis                        |
  |    - Measure actual vs expected behavior          |
  |    - If system degrades beyond SLO, fix it        |
  |                                                   |
  | 5. Minimize blast radius                          |
  |    - Start small (one instance)                   |
  |    - Expand gradually                             |
  |    - Have abort button ready                      |
  +──────────────────────────────────────────────────+
```

### Netflix Simian Army

```
  +───────────────────────────────────────────────────+
  |  SIMIAN ARMY                                       |
  |                                                    |
  |  Chaos Monkey      Kill random EC2 instances       |
  |  Chaos Kong        Simulate full region failure    |
  |  Latency Monkey    Inject network delays           |
  |  Conformity Monkey Check instances follow rules    |
  |  Doctor Monkey     Health check all instances      |
  |  Janitor Monkey    Clean unused resources          |
  |  Security Monkey   Find security violations        |
  +───────────────────────────────────────────────────+
```

### Chaos Engineering Tools

| Tool           | Provider     | Key Features                            |
|---------------|-------------|----------------------------------------|
| Chaos Monkey  | Netflix OSS  | Random instance termination             |
| Gremlin       | Commercial   | Full platform, safe experiments          |
| Litmus        | Open source  | Kubernetes-native chaos                  |
| Chaos Mesh    | Open source  | Kubernetes-native, many fault types      |
| AWS FIS       | AWS          | Managed fault injection for AWS services |

### Chaos Experiments by Maturity

```
  Level 1 (Beginner):
  - Kill a single application instance
  - Restart a database node
  - Simulate a downstream API timeout

  Level 2 (Intermediate):
  - Kill an entire availability zone
  - Inject 500ms latency to database calls
  - Fill disk to 95% on a node
  - Corrupt DNS responses

  Level 3 (Advanced):
  - Simulate region failure
  - Network partition between services
  - Clock skew between nodes
  - Data corruption scenarios
  - Simultaneous multi-component failures
```

---

## 5. Fault Injection Testing

### Fault Types

```
  +────────────────────────────────────────────────────+
  | INFRASTRUCTURE FAULTS                               |
  | - Instance termination                              |
  | - Network partition                                 |
  | - Disk failure                                      |
  | - CPU/memory pressure                               |
  +────────────────────────────────────────────────────+
  | APPLICATION FAULTS                                  |
  | - Exception injection                               |
  | - Latency injection (slow responses)                |
  | - HTTP error code injection (500, 503)              |
  | - Resource exhaustion (thread pool, connections)    |
  +────────────────────────────────────────────────────+
  | DATA FAULTS                                         |
  | - Corrupted messages in queue                       |
  | - Database replication lag simulation               |
  | - Stale cache data                                  |
  | - Clock skew between services                       |
  +────────────────────────────────────────────────────+
```

### Fault Injection Implementation

```
  Using HTTP middleware for latency/error injection:

  Request ──> [Fault Injection Middleware] ──> Service
                     |
                     v
              Is injection enabled?
              (feature flag / config)
                     |
              +──────┼──────+
              |      |      |
          No fault  Latency  Error
          (pass     (sleep   (return
          through)  500ms)   503)

  Configuration:
  {
    "faults": {
      "payment-service": {
        "latency_ms": 500,
        "latency_percentage": 10,
        "error_code": 503,
        "error_percentage": 5
      }
    }
  }
```

---

## 6. Contract Testing for Microservices

### The Problem

```
  Without Contract Testing:

  Service A (Consumer)              Service B (Provider)
  Expects:                          Returns:
  { "id": 123,                     { "id": 123,
    "name": "Alice",      !=         "full_name": "Alice",  <-- RENAMED!
    "email": "a@b.com" }              "email": "a@b.com" }

  Service A breaks in production because Service B changed "name" to
  "full_name" and nobody caught it.
```

### Consumer-Driven Contracts (Pact)

```
  Step 1: Consumer defines contract
  +───────────────────────────────────+
  | Consumer (Service A)              |
  | "I expect Provider to return:    |
  |  { name: string, email: string } |
  |  when I call GET /users/123"     |
  +───────────────────┬───────────────+
                      |
                    Pact file
                      |
                      v
  Step 2: Provider verifies contract
  +───────────────────────────────────+
  | Provider (Service B)              |
  | Runs Pact verification:           |
  | "Can I satisfy all consumer      |
  |  contracts?"                      |
  |                                   |
  | Result: FAIL - "name" field      |
  |         renamed to "full_name"    |
  +───────────────────────────────────+

  Caught BEFORE deployment!
```

### Contract Testing vs E2E Testing

| Aspect           | Contract Testing      | E2E Testing              |
|-----------------|-----------------------|--------------------------|
| Speed           | Seconds               | Minutes                  |
| Scope           | API boundary          | Full system              |
| Environment     | Isolated              | Full stack required      |
| Maintenance     | Low                   | High                     |
| Failure clarity | Exact field/endpoint  | "Something is broken"    |
| When to run     | Every PR              | Pre-release              |

---

## 7. Canary Testing and Progressive Rollouts

### Canary Deployment

```
  Phase 1: Deploy to canary (1% traffic)

  Users ──> Load Balancer
            ├── 99% ──> v1.0 (stable)    [10 instances]
            └──  1% ──> v1.1 (canary)    [1 instance]

  Monitor for 15 minutes:
  - Error rate: canary vs stable
  - Latency: canary vs stable
  - Business metrics: conversion rate

  Phase 2: If healthy, expand to 10%
  Phase 3: If healthy, expand to 50%
  Phase 4: If healthy, promote to 100%

  AUTO-ROLLBACK if:
  - Error rate > 2x baseline
  - p99 latency > 2x baseline
  - Any critical alerts fire
```

### Progressive Rollout Strategies

```
  Linear Rollout:
  Traffic %
  100|                              ████████
     |                        ██████
     |                  ██████
     |            ██████
     |      ██████
     |██████
   0 +──────────────────────────────────────> Time
     0    15min  30min  45min  60min

  Exponential Rollout:
  Traffic %
  100|                              ████████
     |                           ███
     |                        ███
     |                     ███
     |                  ███
     |         █████████
   0 +──────────────────────────────────────> Time
     1%   2%   5%   10%  25%  50%  100%
```

### Blue-Green vs Canary

| Aspect          | Blue-Green              | Canary                   |
|----------------|-------------------------|--------------------------|
| Traffic split  | 0% or 100%              | Gradual (1% -> 100%)     |
| Risk           | All-or-nothing          | Limited blast radius     |
| Rollback speed | Instant (switch back)   | Instant (route to old)   |
| Infrastructure | 2x capacity needed      | +1 instance minimum      |
| Validation time| Quick (minutes)         | Longer (hours)           |

---

## 8. A/B Testing Infrastructure

### Architecture

```
  +──────────────+
  | User Request |
  +──────┬───────+
         |
  +──────┴───────+
  | Assignment   |     User 123 -> Experiment: new_checkout
  | Service      |     Variant: B (treatment)
  +──────┬───────+     Sticky: hash(user_id + experiment_id)
         |
  +──────┴───────+
  | Feature Flag |     if (variant == 'B') { showNewCheckout() }
  | Evaluation   |
  +──────┬───────+
         |
  +──────┴───────+
  | Event        |     Track: checkout_completed, revenue, time_spent
  | Tracking     |
  +──────┬───────+
         |
  +──────┴───────+
  | Analysis     |     Statistical significance: p < 0.05
  | Pipeline     |     Treatment vs Control comparison
  +──────────────+
```

### Key Requirements

| Requirement              | Description                                     |
|-------------------------|------------------------------------------------|
| Consistent assignment   | Same user always sees same variant               |
| Even distribution       | 50/50 split across variants                      |
| Mutual exclusion        | Avoid conflicting experiments                    |
| Statistical significance| Enough sample size for valid conclusions         |
| Segmentation            | Target specific user groups                      |
| Kill switch             | Immediately disable any experiment               |

---

## 9. Shadow Testing / Dark Launching

### Shadow Traffic Pattern

```
  +──────────────+
  | User Request |
  +──────┬───────+
         |
  +──────┴───────+
  | Load Balancer|
  +──┬───────┬───+
     |       |
     |   (duplicate)
     |       |
  +──┴──+  +─┴──────+
  | v1  |  | v2     |
  | Live |  | Shadow |
  +──┬──+  +──┬─────+
     |        |
  Response  Response
  to user   DISCARDED
     |        |
     +────> Compare responses for correctness
```

### When to Use Shadow Testing

| Use Case                            | Benefit                              |
|------------------------------------|------------------------------------|
| Database migration                  | Verify new DB returns same results  |
| API rewrite                        | Ensure backward compatibility       |
| New ML model                       | Compare predictions with production |
| Performance validation             | Benchmark new version under real load|
| Third-party service migration      | Verify new vendor matches behavior  |

### Implementation Considerations

```
  IMPORTANT: Shadow traffic must be read-only or idempotent!

  SAFE to shadow:
  - GET /users/123                  (read-only)
  - Search queries                   (read-only)
  - Recommendation requests          (read-only)
  - Price calculations               (read-only)

  UNSAFE to shadow (without modification):
  - POST /orders                    (creates real order!)
  - POST /payments/charge           (charges real money!)
  - DELETE /users/123               (deletes real data!)
  - POST /emails/send               (sends real email!)

  Solution for write operations:
  - Route to isolated environment with test data
  - Intercept side effects (mock payment, email)
  - Compare request/response only, don't execute
```

---

## 10. Data Migration Testing

### Migration Testing Strategy

```
  +────────────────────────────────────────────────────+
  | DATA MIGRATION TEST PLAN                            |
  |                                                     |
  | Phase 1: Schema Validation                          |
  | - New schema supports all existing data types       |
  | - Constraints (FK, unique, NOT NULL) preserved      |
  | - Indexes exist for all query patterns              |
  |                                                     |
  | Phase 2: Data Integrity                             |
  | - Row counts match (source vs destination)          |
  | - Checksums match for sampled rows                  |
  | - Edge cases: NULL values, special characters,      |
  |   Unicode, max-length strings, boundary numbers     |
  |                                                     |
  | Phase 3: Performance Validation                     |
  | - Run top 20 queries against new DB                 |
  | - Compare execution plans                           |
  | - Verify latency within acceptable range            |
  |                                                     |
  | Phase 4: Application Compatibility                  |
  | - Run integration tests against new DB              |
  | - Shadow traffic comparison                         |
  | - Rollback procedure verified                       |
  +────────────────────────────────────────────────────+
```

### Double-Write Pattern Testing

```
  Migration Phase:

  Application
      |
      +──> Write to OLD DB ──> (primary, serves reads)
      |
      +──> Write to NEW DB ──> (secondary, validation only)

  Verification Job (runs continuously):
  - Read from both DBs
  - Compare results
  - Log discrepancies
  - Alert if mismatch rate > threshold

  Cutover:
  1. Verify 0 discrepancies for 24 hours
  2. Switch reads to NEW DB
  3. Monitor for errors
  4. Stop writes to OLD DB after confidence period
  5. Keep OLD DB for 7 days as rollback option
```

---

## 11. Performance Regression Testing

### CI/CD Integration

```
  Code Commit ──> Build ──> Unit Tests ──> Integration Tests
                                                  |
                                          [Performance Gate]
                                                  |
                                    +-────────────┼────────────+
                                    |             |            |
                                  PASS         WARNING       FAIL
                                 (<5% slower)  (5-15%)      (>15%)
                                    |             |            |
                                 Deploy        Deploy +     Block
                                              Alert team   deployment
```

### Performance Baseline

```
  Establish baseline metrics for each endpoint:

  +─────────────────────+──────+──────+──────+──────────+
  | Endpoint            | p50  | p95  | p99  | RPS Max  |
  +─────────────────────+──────+──────+──────+──────────+
  | GET /users/:id      | 12ms | 35ms | 80ms | 5,000    |
  | POST /orders        | 45ms | 120ms| 350ms| 2,000    |
  | GET /search?q=      | 30ms | 85ms | 200ms| 3,000    |
  | GET /feed           | 25ms | 60ms | 150ms| 4,000    |
  +─────────────────────+──────+──────+──────+──────────+

  Each PR runs benchmark suite and compares against baseline.
  Regression detected if any metric degrades beyond threshold.
```

### Regression Detection Methods

| Method                  | Description                              | Accuracy    |
|------------------------|------------------------------------------|-------------|
| Fixed threshold        | Fail if p99 > 200ms                       | Low (noisy) |
| Relative comparison    | Fail if > 10% slower than baseline        | Medium      |
| Statistical comparison | Fail if statistically significant slowdown | High        |
| Trend analysis         | Detect gradual degradation over time       | High        |

---

## 12. Disaster Recovery Testing

### Game Day Framework

```
  +────────────────────────────────────────────────────+
  | GAME DAY PLANNING                                   |
  |                                                     |
  | Before:                                             |
  | - Define scenario (region failure, DB corruption)   |
  | - Establish success criteria (RTO < 15 min)         |
  | - Notify stakeholders                               |
  | - Prepare rollback plan                             |
  | - Designate observer and facilitator                |
  |                                                     |
  | During:                                             |
  | - Execute failure scenario                          |
  | - Time detection (MTTD)                             |
  | - Time recovery (MTTR)                              |
  | - Document all actions taken                        |
  | - Note unexpected behaviors                         |
  |                                                     |
  | After:                                              |
  | - Blameless post-mortem                             |
  | - Document gaps found                               |
  | - Create action items                               |
  | - Update runbooks                                   |
  | - Schedule next game day                            |
  +────────────────────────────────────────────────────+
```

### DR Test Scenarios

| Scenario                    | Tests                                  | Frequency  |
|----------------------------|----------------------------------------|------------|
| Single instance failure    | Auto-scaling, health checks             | Monthly    |
| AZ failure                 | Multi-AZ failover, data replication     | Quarterly  |
| Region failure             | Multi-region failover, DNS failover     | Bi-annually|
| Database failure           | Backup restore, replica promotion       | Monthly    |
| Data corruption            | Point-in-time recovery, backup validity | Quarterly  |
| Network partition          | Split-brain handling, quorum            | Quarterly  |
| Dependency outage          | Circuit breakers, graceful degradation  | Monthly    |

### Tabletop Exercises

```
  When full DR testing is too risky or expensive, run tabletop exercises:

  Facilitator: "It is 3 AM. The primary database in US-East is
  unreachable. PagerDuty has fired. Walk me through your response."

  Engineer 1: "I check the CloudWatch dashboard to confirm the outage.
  I see the RDS instance is unreachable."

  Engineer 2: "I initiate the failover runbook. Step 1: Promote the
  read replica in US-East-2 to primary."

  Facilitator: "The replica is 30 seconds behind. You now have a
  promoted primary missing 30 seconds of writes. What do you do?"

  This reveals gaps in runbooks and training without risking
  production systems.
```

---

## 13. Interview Relevance

### When to Discuss Testing

- When presenting any system design (always mention testing strategy)
- When discussing reliability and availability
- When talking about deployment strategy
- When asked "How would you validate this works?"

### Key Phrases

```
  "Before deploying, I would run load tests simulating 2x expected peak
   traffic to verify the system handles our growth projections."

  "I would implement canary deployments, routing 1% of traffic to the
   new version and automatically rolling back if error rates spike."

  "To build confidence in our failover mechanisms, I would schedule
   quarterly game days where we simulate AZ and region failures."

  "I would use contract testing between our microservices so that API
   changes are caught at PR time rather than in production."

  "I would set up shadow testing for the database migration, sending
   duplicate reads to both the old and new databases and comparing
   results before cutting over."
```

### Common Interview Mistakes

| Mistake                              | Better Approach                       |
|-------------------------------------|---------------------------------------|
| Not mentioning testing at all       | Include test strategy in every design |
| Only mentioning unit tests          | Cover all levels of the pyramid       |
| Skipping chaos engineering          | Mention chaos experiments for HA      |
| No deployment safety                | Include canary/blue-green deployment  |
| Ignoring data migration risks       | Describe double-write and validation  |
