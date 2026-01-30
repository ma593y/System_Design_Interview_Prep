# Cost Optimization

## Table of Contents
1. [Cloud Cost Models](#1-cloud-cost-models)
2. [Right-Sizing Resources](#2-right-sizing-resources)
3. [Auto-Scaling Strategies](#3-auto-scaling-strategies)
4. [Storage Tiering](#4-storage-tiering)
5. [Data Transfer Cost Optimization](#5-data-transfer-cost-optimization)
6. [Database Cost Optimization](#6-database-cost-optimization)
7. [Serverless vs Containers vs VMs](#7-serverless-vs-containers-vs-vms)
8. [Multi-Tenancy for Cost Efficiency](#8-multi-tenancy-for-cost-efficiency)
9. [Cost Monitoring and Alerting](#9-cost-monitoring-and-alerting)
10. [Build vs Buy Decisions](#10-build-vs-buy-decisions)
11. [Interview Relevance](#11-interview-relevance)

---

## 1. Cloud Cost Models

### Pricing Model Overview

```
  Cost
   |
   |  On-Demand ─────────────────────────────── Highest $/hr
   |
   |  Reserved (1yr) ────────────────────────── ~40% savings
   |
   |  Reserved (3yr) ────────────────────────── ~60% savings
   |
   |  Spot/Preemptible ─────────────────────── ~70-90% savings
   |
   |  Free Tier / Always Free ───────────────── $0
   +──────────────────────────────────────────────>
                       Commitment Level
```

### On-Demand Instances

- Pay per second/hour of usage with no upfront commitment.
- Best for: unpredictable workloads, development, short-term spikes.
- Most expensive per unit of compute.

### Reserved Instances / Savings Plans

```
  Example: m5.xlarge in US-East

  On-Demand:     $0.192/hr  x 8760 hrs = $1,682/yr
  1yr Reserved:  $0.120/hr  x 8760 hrs = $1,051/yr  (37% savings)
  3yr Reserved:  $0.075/hr  x 8760 hrs =   $657/yr  (61% savings)

  Break-even for 1yr Reserved: ~7 months of usage
  Break-even for 3yr Reserved: ~14 months of usage
```

**Types of reservations:**
- **All Upfront:** Largest discount, pay entire term upfront.
- **Partial Upfront:** Moderate discount, pay some upfront plus reduced hourly rate.
- **No Upfront:** Smallest discount, commit to 1/3 year term at reduced hourly rate.

### Spot Instances / Preemptible VMs

```
  Spot Pricing Over Time:
  Price
   |     ___
   | ___/   \___       ___
   |/           \___  /   \___
   |                \/        \_________
   +────────────────────────────────────> Time
   On-Demand: $0.192/hr (fixed)
   Spot:      $0.02-$0.06/hr (variable, ~70-90% savings)
```

**Best for:** Batch processing, CI/CD, data analysis, stateless web servers.
**Risk:** Can be terminated with 2-minute notice.
**Mitigation:** Diversify across instance types and AZs, use spot fleets.

### Cost Comparison Table

| Model          | Discount | Commitment | Interruption | Best For              |
|---------------|----------|------------|-------------|------------------------|
| On-Demand     | 0%       | None       | Never       | Variable workloads     |
| Reserved 1yr  | ~40%     | 1 year     | Never       | Steady-state baseline  |
| Reserved 3yr  | ~60%     | 3 years    | Never       | Predictable long-term  |
| Spot          | 70-90%   | None       | Possible    | Fault-tolerant batch   |
| Savings Plans | ~30-40%  | 1-3 years  | Never       | Flexible commitment    |

---

## 2. Right-Sizing Resources

### Identifying Waste

```
  Typical Server Utilization:

  Over-provisioned (common):
  CPU:  [████░░░░░░░░░░░░░░░░] 20%  --> Wasting 80%
  MEM:  [████████░░░░░░░░░░░░] 40%  --> Wasting 60%
  Disk: [██░░░░░░░░░░░░░░░░░░] 10%  --> Wasting 90%

  Right-sized:
  CPU:  [██████████████░░░░░░] 70%  --> 30% headroom
  MEM:  [████████████████░░░░] 80%  --> 20% headroom
  Disk: [████████████░░░░░░░░] 60%  --> 40% headroom
```

### Right-Sizing Process

```
  1. Monitor  ──>  2. Analyze  ──>  3. Resize  ──>  4. Validate
     (2 weeks)     (identify        (change         (verify
                    waste)           instance)       performance)
                                        |
                                        v
                              5. Repeat quarterly
```

### CPU vs Memory Optimization

| Workload Type     | Optimize For  | Instance Family (AWS)   |
|------------------|---------------|-------------------------|
| Web servers      | Balanced       | m5, m6i (general)       |
| Batch processing | CPU            | c5, c6i (compute)       |
| In-memory cache  | Memory         | r5, r6i (memory)        |
| ML training      | GPU            | p4, g5 (accelerated)    |
| Storage-heavy    | Storage IOPS   | i3, d3 (storage)        |

### Graviton / ARM-based Instances

ARM-based processors (AWS Graviton, GCP Tau) offer 20-40% better price-performance
for many workloads compared to x86.

```
  x86 (m5.xlarge):   $0.192/hr, 4 vCPU, 16 GB
  ARM (m6g.xlarge):  $0.154/hr, 4 vCPU, 16 GB  (20% cheaper)
  ARM (m7g.xlarge):  $0.163/hr, 4 vCPU, 16 GB  (15% cheaper, faster)
```

---

## 3. Auto-Scaling Strategies

### Target Tracking Scaling

Automatically adjusts capacity to maintain a target metric value.

```
  Target: CPU Utilization = 60%

  CPU%
  80 |         *
  70 |       *   *        Scale UP triggered
  60 |─ ─ * ─ ─ ─ * ─ ─ Target ─ ─ ─ ─ ─ ─
  50 |  *           *
  40 |*               *   Scale DOWN triggered
  30 |                  * *
     +────────────────────────────> Time
  Instances:
  3   4   5   6   6   5   4   3
```

### Step Scaling

Define discrete steps based on alarm thresholds.

```
  +────────────────────────────────────────────+
  | Alarm Threshold    | Action                |
  +────────────────────────────────────────────+
  | CPU > 80%          | Add 3 instances       |
  | CPU > 70%          | Add 2 instances       |
  | CPU > 60%          | Add 1 instance        |
  | CPU < 30%          | Remove 1 instance     |
  | CPU < 20%          | Remove 2 instances    |
  +────────────────────────────────────────────+
```

### Predictive Scaling

Uses ML to predict traffic patterns and pre-scale before demand hits.

```
  Traffic Pattern (daily):

  Load
   |           ___
   |          /   \
   |         /     \
   |    ____/       \____
   |   /                 \
   |  /                   \___
   |_/                        \___
   +────────────────────────────────> Time
   6am    12pm    6pm    12am

  Predictive scaling provisions capacity 10 minutes before
  predicted spike, avoiding latency during scale-up.
```

### Scaling Strategy Comparison

| Strategy          | React Time | Cost Efficiency | Complexity | Best For             |
|------------------|-----------|-----------------|------------|----------------------|
| Target tracking  | Minutes   | Good            | Low        | Steady metrics       |
| Step scaling     | Minutes   | Good            | Medium     | Known thresholds     |
| Predictive       | Proactive | Excellent       | Low        | Predictable patterns |
| Scheduled        | Proactive | Good            | Low        | Known events         |

---

## 4. Storage Tiering

### S3 Storage Classes

```
  Access Frequency vs Cost:

  Cost/GB
   |
   | S3 Standard           [$0.023/GB]  ──> Frequent access
   |
   | S3 Intelligent-Tier   [$0.023/GB]  ──> Unknown patterns
   |
   | S3 Standard-IA        [$0.0125/GB] ──> Monthly access
   |
   | S3 One Zone-IA        [$0.010/GB]  ──> Reproducible data
   |
   | S3 Glacier Instant    [$0.004/GB]  ──> Quarterly access
   |
   | S3 Glacier Flexible   [$0.0036/GB] ──> Yearly access
   |
   | S3 Glacier Deep       [$0.00099/GB]──> Compliance/archive
   |
   +──────────────────────────────────────>
                  Access Frequency
```

### Lifecycle Policies

```
  Data Lifecycle Automation:

  Day 0          Day 30         Day 90         Day 365
    |              |              |               |
    v              v              v               v
  S3 Standard -> Standard-IA -> Glacier     -> Glacier Deep
  $0.023/GB     $0.0125/GB    Flexible        Archive
                               $0.0036/GB      $0.00099/GB

  Example: 10 TB of logs
  Always Standard:  $230/month = $2,760/year
  With Lifecycle:   $230 + $125 + $36 + ... = ~$800/year (71% savings)
```

### Hot/Warm/Cold/Archive Pattern

| Tier    | Access Pattern    | Storage Type          | Retrieval Time | Cost    |
|---------|------------------|----------------------|----------------|---------|
| Hot     | Real-time         | SSD, Memory, S3 Std  | Milliseconds   | High    |
| Warm    | Daily/Weekly      | HDD, S3 IA           | Milliseconds   | Medium  |
| Cold    | Monthly/Quarterly | S3 Glacier Instant    | Milliseconds   | Low     |
| Archive | Yearly/Never      | S3 Glacier Deep       | Hours          | Lowest  |

---

## 5. Data Transfer Cost Optimization

### Data Transfer Cost Model

```
  +---------+                    +---------+
  |  AWS    |  Cross-region:     |  AWS    |
  | Region  | $0.02/GB           | Region  |
  | US-East |====================>| EU-West |
  +---------+                    +---------+
       |                              |
       | To Internet:                 |
       | $0.09/GB                     |
       | (first 10 TB)               |
       v                              v
  [Internet Users]             [Internet Users]

  INBOUND: FREE (in most cases)
  WITHIN AZ: FREE (private IP)
  CROSS AZ: $0.01/GB each way
  CROSS REGION: $0.02/GB
  TO INTERNET: $0.05-$0.09/GB (tiered)
```

### Cost Reduction Strategies

```
  Strategy 1: CDN to reduce origin egress

  Without CDN:
  1M requests x 500KB avg = 500 TB/month
  Cost: 500 TB x $0.085/GB = $42,500/month

  With CDN (90% cache hit):
  Origin transfer: 50 TB x $0.085/GB = $4,250
  CDN transfer: 500 TB x $0.085/GB = $42,500
  CDN is often cheaper than direct origin transfer
  Plus CDN has tiered pricing that drops significantly

  Strategy 2: Compression

  Uncompressed JSON: 10 KB avg response
  Compressed (gzip):  2 KB avg response (80% reduction)
  Transfer cost savings: 80%

  Strategy 3: Keep traffic within the same AZ where possible

  Service A (AZ-a) -> Service B (AZ-a): FREE (private IP)
  Service A (AZ-a) -> Service B (AZ-b): $0.01/GB each way
```

### VPC Endpoints

```
  Without VPC Endpoint:
  EC2 ──> Internet Gateway ──> S3
  Cost: Data transfer charges apply

  With VPC Endpoint (Gateway):
  EC2 ──> VPC Endpoint ──> S3
  Cost: FREE for S3 and DynamoDB gateway endpoints
```

---

## 6. Database Cost Optimization

### Read Replicas vs Cache

```
  Option A: Scale reads with Read Replicas

  Primary + 3 Read Replicas (db.r5.2xlarge)
  Cost: 4 x $0.96/hr = $3.84/hr = $2,803/month

  Option B: Add Redis Cache

  Primary DB (db.r5.xlarge) + Redis (r5.large)
  Cost: $0.48 + $0.25 = $0.73/hr = $533/month

  If cache hit rate > 80%, Option B handles same read load
  at ~80% lower cost.
```

### Serverless Databases

```
  Traditional Provisioned:
  Cost
   |  ████████████████████████████  $500/month (always on)
   +──────────────────────────────> Time

  Serverless (Aurora Serverless, DynamoDB On-Demand):
  Cost
   |  ██  ████  ██ █████ ██  ██   $120/month (pay per use)
   +──────────────────────────────> Time

  Best for: Variable/unpredictable workloads, dev/test environments
  Avoid for: Steady high-throughput production workloads
```

### DynamoDB Pricing: Provisioned vs On-Demand

| Metric                | Provisioned         | On-Demand           |
|-----------------------|--------------------|---------------------|
| Write cost            | $0.00065 per WCU/hr | $1.25 per million   |
| Read cost             | $0.00013 per RCU/hr | $0.25 per million   |
| Best for              | Predictable traffic  | Variable traffic    |
| Auto-scaling          | Yes (with delay)    | Instant             |
| Cost at steady state  | Lower               | Higher              |
| Cost at variable load | Wastes capacity     | Pay exact usage     |

---

## 7. Serverless vs Containers vs VMs

### Cost Comparison at Different Scales

```
  Monthly Cost
   |
   |                                           / VMs
   |                                         /
   |                                  ______/
   |                           ______/
   |                    ______/   Containers
   |             ______/     ___/
   |      ______/       ___/
   |_____/ Serverless__/
   |  /___/
   +────────────────────────────────────────> Requests/month
   0    1M    10M    100M    1B    10B

  Serverless cheapest at low volume
  Containers cheapest at medium-high volume
  VMs cheapest at very high, steady volume
```

### Detailed Comparison

| Factor            | Serverless           | Containers (ECS/K8s) | VMs (EC2)          |
|------------------|---------------------|---------------------|---------------------|
| Pricing model    | Per request + duration| Per container hour  | Per instance hour   |
| Min cost         | $0 (free tier)       | ~$15/month          | ~$4/month (nano)    |
| Idle cost        | $0                   | Base cluster cost   | Full instance cost  |
| Scale-to-zero    | Yes                  | With KEDA           | No                  |
| Cold start       | 100ms-10s            | Seconds             | Minutes             |
| Max duration     | 15 min (Lambda)      | Unlimited           | Unlimited           |
| Management       | None                 | Medium (orchestration)| High (OS, patches)|
| Cost at 1M req   | ~$20                 | ~$50                | ~$30                |
| Cost at 1B req   | ~$20,000             | ~$2,000             | ~$1,500             |

### When to Use What

```
  Decision Tree:

  Is traffic predictable?
  ├── No ──> Is request duration < 15 min?
  |          ├── Yes ──> Serverless (Lambda/Cloud Functions)
  |          └── No  ──> Containers with auto-scaling
  └── Yes ──> Is it steady 24/7?
              ├── Yes ──> Reserved VMs or Containers
              └── No  ──> Containers with scheduled scaling
```

---

## 8. Multi-Tenancy for Cost Efficiency

### Tenancy Models

```
  Single-Tenant:                Multi-Tenant:

  +──────────+  +──────────+   +────────────────────+
  | Tenant A |  | Tenant B |   |    Shared Infra    |
  | [App]    |  | [App]    |   | [App]              |
  | [DB]     |  | [DB]     |   | Tenant A | Tenant B|
  | [Cache]  |  | [Cache]  |   | [Shared DB]        |
  +──────────+  +──────────+   | [Shared Cache]     |
                               +────────────────────+
  Cost: 2x                    Cost: 1.2-1.5x
```

### Multi-Tenancy Levels

| Level               | Isolation | Cost/Tenant | Complexity | Use Case             |
|---------------------|-----------|-------------|------------|----------------------|
| Separate infra      | Highest   | Highest     | Low        | Enterprise/regulated |
| Shared app, sep DB  | High      | Medium      | Medium     | SaaS standard        |
| Shared app+DB       | Medium    | Low         | High       | Consumer SaaS        |
| Schema per tenant   | Medium    | Low         | Medium     | B2B SaaS             |
| Row-level isolation | Lowest    | Lowest      | High       | High-scale consumer  |

### Noisy Neighbor Problem

```
  Shared Resources:

  +──────────────────────────────────────+
  | Shared Database (1000 IOPS)          |
  |                                      |
  | Tenant A: 100 IOPS (normal)          |
  | Tenant B: 800 IOPS (spike!)          |
  | Tenant C: 100 IOPS (normal)          |
  |                                      |
  | Total: 1000 IOPS = capacity!         |
  | Tenant A and C degraded              |
  +──────────────────────────────────────+

  Mitigation:
  - Rate limiting per tenant
  - Resource quotas per tenant
  - Separate hot tenants to dedicated infra
  - Burstable capacity with hard limits
```

---

## 9. Cost Monitoring and Alerting

### AWS Cost Management Tools

```
  +──────────────────────────────────────────────+
  |            Cost Management Stack              |
  |                                               |
  |  AWS Cost Explorer ──> Visualize spending     |
  |  AWS Budgets ──> Set alerts on thresholds     |
  |  Cost Anomaly Detection ──> ML-based alerts   |
  |  Trusted Advisor ──> Optimization recs        |
  |  Compute Optimizer ──> Right-sizing recs      |
  |  Savings Plans Rec ──> Commitment recs        |
  +──────────────────────────────────────────────+
```

### Cost Allocation Tags

```
  Every resource tagged with:

  +────────────────+─────────────────+
  | Tag            | Example Value   |
  +────────────────+─────────────────+
  | Environment    | production      |
  | Team           | platform        |
  | Service        | user-service    |
  | CostCenter     | CC-1234         |
  | Project        | project-phoenix |
  +────────────────+─────────────────+

  Enables: Cost breakdown by team, service, environment
```

### Budget Alerts

```
  Budget: $10,000/month

  Alert Thresholds:
  +──────────────────────────────────────+
  | 50% ($5,000)  ──> Email notification |
  | 80% ($8,000)  ──> Slack + email      |
  | 100% ($10,000)──> Page on-call       |
  | 120% ($12,000)──> Auto-remediation   |
  +──────────────────────────────────────+

  Anomaly Detection:
  Expected: $330/day
  Actual:   $1,200/day ──> Alert: 3.6x expected spend
```

### Cost Optimization Checklist

```
  Weekly:
  [ ] Review cost anomaly alerts
  [ ] Check spot instance interruption rate

  Monthly:
  [ ] Review top 10 cost items
  [ ] Check for idle/unused resources
  [ ] Review data transfer costs
  [ ] Evaluate reserved instance coverage

  Quarterly:
  [ ] Right-sizing analysis
  [ ] Storage tier review
  [ ] Reserved instance/Savings Plan evaluation
  [ ] Architecture review for cost efficiency
```

---

## 10. Build vs Buy Decisions

### Decision Framework

```
  Build vs Buy Matrix:

                    Core Differentiator
                    |
              Build |  BUILD
              (high |  Custom solution
              value)|  Full control
                    |
  ──────────────────┼──────────────────
                    |
              Buy   |  BUY
              (low  |  Managed service
              value)|  Focus on core product
                    |
                    Non-Core / Commodity
```

### Cost Comparison Framework

| Factor                    | Build                     | Buy (SaaS/Managed)       |
|--------------------------|---------------------------|--------------------------|
| Upfront cost             | High (dev time)            | Low (subscription)       |
| Ongoing cost             | Maintenance, ops, on-call  | Subscription fee         |
| Time to market           | Months                     | Days/weeks               |
| Customization            | Unlimited                  | Limited to features      |
| Operational burden       | Full (you manage it)       | Reduced (vendor manages) |
| Switching cost           | Low (you own it)           | High (vendor lock-in)    |
| Scaling responsibility   | Yours                      | Vendor                   |

### Examples

| Component           | Build                          | Buy                           |
|--------------------|--------------------------------|-------------------------------|
| Email sending      | Build SMTP infra (avoid)       | SendGrid, SES ($0.10/1K)     |
| Search             | Elasticsearch cluster          | Algolia, Elastic Cloud        |
| Authentication     | Custom auth system             | Auth0, Cognito, Firebase Auth |
| Monitoring         | Prometheus + Grafana           | Datadog, New Relic            |
| Payment processing | PCI compliance infra (avoid)   | Stripe (2.9% + $0.30)        |
| CDN                | Build edge network (avoid)     | CloudFront, Cloudflare        |

---

## 11. Interview Relevance

### When to Discuss Cost

- When the interviewer explicitly asks about cost
- When comparing two valid architectural approaches
- When discussing scale (millions of users = significant cost)
- When proposing managed services vs self-hosted

### Key Phrases for Interviews

```
  "For this component, I would start with a managed service like X to reduce
   operational overhead, then evaluate building a custom solution if we hit
   scale where the cost becomes prohibitive."

  "I would use spot instances for the batch processing workers since they
   are fault-tolerant and can retry on interruption, saving about 70%
   compared to on-demand."

  "At this scale, caching becomes essential not just for performance but
   for cost. Serving from Redis at $0.001 per request is far cheaper than
   hitting the database at $0.01 per query."

  "I would implement storage tiering so that only the last 30 days of data
   lives on hot storage. Older data moves to cold storage automatically,
   reducing storage costs by 80%."
```

### Common Interview Mistakes

| Mistake                          | Better Approach                         |
|---------------------------------|----------------------------------------|
| Ignoring cost entirely           | Mention cost when comparing approaches  |
| Over-optimizing for cost         | Balance cost with reliability and speed  |
| Using most expensive option only | Propose tiered approach (spot + reserved)|
| Not considering operational cost | Include engineering time in TCO          |
| Proposing only build or only buy | Evaluate both, justify your choice       |

### Cost-Aware Architecture Example

```
  System: Image Processing Pipeline (10M images/day)

  Cost-Naive Design:
  API (On-Demand EC2) -> Process (On-Demand EC2) -> Store (S3 Standard)
  Monthly cost: ~$15,000

  Cost-Optimized Design:
  API (Reserved EC2)    -> Queue (SQS)      -> Process (Spot EC2)
  Store: S3 Standard (30 days) -> S3 IA -> Glacier
  Monthly cost: ~$4,500 (70% savings)

  Key optimizations:
  1. Reserved for steady API servers ($3,000 -> $1,800)
  2. Spot for batch processing ($6,000 -> $900)
  3. Storage tiering ($3,000 -> $500)
  4. SQS decouples and enables spot interruption handling
```
