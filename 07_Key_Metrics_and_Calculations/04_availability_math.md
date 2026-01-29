# Availability Math - The Nines

## Overview

Availability is a critical metric in system design that measures what percentage of time a system is operational and accessible. Understanding availability calculations, SLA math, and the implications of different availability levels helps you design resilient systems and set realistic expectations. This guide covers the "nines" of availability, downtime calculations, and strategies for achieving high availability.

---

## Understanding the Nines

### Availability Percentage to Downtime

| Availability | Downtime/Year | Downtime/Month | Downtime/Week | Downtime/Day |
|--------------|---------------|----------------|---------------|--------------|
| 90% (one nine) | 36.5 days | 72 hours | 16.8 hours | 2.4 hours |
| 99% (two nines) | 3.65 days | 7.2 hours | 1.68 hours | 14.4 min |
| 99.5% | 1.83 days | 3.6 hours | 50.4 min | 7.2 min |
| 99.9% (three nines) | 8.76 hours | 43.8 min | 10.1 min | 1.44 min |
| 99.95% | 4.38 hours | 21.9 min | 5.04 min | 43.2 sec |
| 99.99% (four nines) | 52.6 min | 4.38 min | 1.01 min | 8.64 sec |
| 99.999% (five nines) | 5.26 min | 26.3 sec | 6.05 sec | 864 ms |
| 99.9999% (six nines) | 31.5 sec | 2.63 sec | 605 ms | 86.4 ms |

### Visual Representation

```
                   AVAILABILITY LADDER

   99.9999% ─────────────────────────────────── Six Nines
      │     Mission-critical systems (banking core)
      │     31.5 seconds downtime/year
      │
   99.999%  ─────────────────────────────────── Five Nines
      │     Enterprise-grade systems
      │     5.26 minutes downtime/year
      │
   99.99%   ─────────────────────────────────── Four Nines
      │     High-availability systems
      │     52.6 minutes downtime/year
      │
   99.9%    ─────────────────────────────────── Three Nines
      │     Standard production systems
      │     8.76 hours downtime/year
      │
   99%      ─────────────────────────────────── Two Nines
      │     Basic web applications
      │     3.65 days downtime/year
      │
   90%      ─────────────────────────────────── One Nine
            Development/staging environments
            36.5 days downtime/year
```

---

## Availability Formulas

### Basic Availability

```
Availability = Uptime / (Uptime + Downtime)

Or equivalently:
Availability = MTBF / (MTBF + MTTR)

Where:
MTBF = Mean Time Between Failures
MTTR = Mean Time To Repair/Recover
```

### Calculating Downtime

```
Downtime per year = (1 - Availability) x 525,600 minutes
Downtime per year = (1 - Availability) x 8,760 hours
Downtime per year = (1 - Availability) x 365 days

Example (99.9% availability):
Downtime = (1 - 0.999) x 525,600 = 525.6 minutes = 8.76 hours
```

### Converting Between Formats

```
Percentage to decimal: 99.9% = 0.999
Nines to percentage: 3 nines = 99.9%
Nines to decimal: 3 nines = 1 - 10^(-3) = 0.999

General formula for N nines:
Availability = 1 - 10^(-N)

Example:
4 nines = 1 - 10^(-4) = 1 - 0.0001 = 0.9999 = 99.99%
```

---

## System Availability Calculations

### Serial (Sequential) Components

When components are in series (all must work for system to work):

```
System Availability = A1 x A2 x A3 x ... x An

Example: Web application with 3 components
- Web Server: 99.9%
- App Server: 99.9%
- Database: 99.9%

System Availability = 0.999 x 0.999 x 0.999 = 0.997 = 99.7%

Downtime increased from 8.76 hours to 26.3 hours per year
```

### Parallel (Redundant) Components

When components are in parallel (any can work for system to work):

```
System Availability = 1 - (1 - A1) x (1 - A2) x ... x (1 - An)

For identical components:
System Availability = 1 - (1 - A)^n

Example: 2 redundant servers, each 99%
System Availability = 1 - (1 - 0.99)^2
                    = 1 - (0.01)^2
                    = 1 - 0.0001
                    = 0.9999 = 99.99%

With 3 redundant servers:
System Availability = 1 - (0.01)^3 = 0.999999 = 99.9999%
```

### Combined Series and Parallel

```
                    ┌─── Server A (99%) ───┐
User ─── LB (99.9%) ┤                      ├─── DB (99.9%)
                    └─── Server B (99%) ───┘

Step 1: Calculate parallel availability (Servers A & B)
Parallel = 1 - (1 - 0.99)^2 = 99.99%

Step 2: Calculate serial availability (LB → Servers → DB)
Total = 0.999 x 0.9999 x 0.999 = 0.9979 = 99.79%

Downtime: ~18.4 hours/year
```

---

## SLA (Service Level Agreement) Calculations

### SLA Components

```
SLA typically includes:
1. Availability target (e.g., 99.9%)
2. Measurement period (monthly, quarterly, annually)
3. Exclusions (planned maintenance, force majeure)
4. Service credits for breaches
```

### Service Credit Calculation

```
Common SLA credit structure:
| Availability Achieved | Credit % |
|-----------------------|----------|
| < 99.9% but >= 99.0%  | 10%      |
| < 99.0% but >= 95.0%  | 25%      |
| < 95.0%               | 50%      |

Example: $10,000/month contract, achieved 98.5% availability
Credit = $10,000 x 25% = $2,500
```

### Monthly SLA Math

```
Minutes in a month (30 days): 43,200

For 99.9% SLA:
Allowed downtime = 43,200 x 0.001 = 43.2 minutes

For 99.95% SLA:
Allowed downtime = 43,200 x 0.0005 = 21.6 minutes

For 99.99% SLA:
Allowed downtime = 43,200 x 0.0001 = 4.32 minutes
```

### Error Budget

```
Error Budget = Allowed downtime based on SLA

Formula:
Error Budget = (1 - SLA Target) x Time Period

Example: 99.9% SLA for a quarter
Error Budget = (1 - 0.999) x 90 days x 24 hours x 60 min
             = 0.001 x 129,600 minutes
             = 129.6 minutes per quarter
             = 43.2 minutes per month

Usage:
- Deployments: 5 min each, budget allows ~8-9 per month
- Incidents: One 30-min incident uses 70% of monthly budget
```

---

## Failure Rate and Reliability

### MTBF and MTTR

```
MTBF (Mean Time Between Failures):
- Measures reliability of a system
- Higher is better
- Typically measured in hours

MTTR (Mean Time To Repair/Recover):
- Measures how quickly you can restore service
- Lower is better
- Includes detection, diagnosis, and repair time

Availability = MTBF / (MTBF + MTTR)

Example:
MTBF = 720 hours (30 days average between failures)
MTTR = 1 hour (average repair time)

Availability = 720 / (720 + 1) = 0.9986 = 99.86%
```

### Improving Availability

```
Two ways to improve availability:

1. Increase MTBF (prevent failures):
   - Better hardware
   - Code quality
   - Testing
   - Monitoring to catch issues early

2. Decrease MTTR (recover faster):
   - Automated failover
   - Better alerting
   - Runbooks and automation
   - Redundancy

Example impact:
Starting: MTBF=100h, MTTR=2h → Availability = 98.04%

Option 1 - Double MTBF (200h):
Availability = 200 / (200 + 2) = 99.01%

Option 2 - Halve MTTR (1h):
Availability = 100 / (100 + 1) = 99.01%

Both have similar impact, but MTTR is often easier to improve!
```

### Failure Rate Calculations

```
Failure Rate (λ) = 1 / MTBF

Probability of failure in time t:
P(failure) = 1 - e^(-λt)

Example: Server with MTBF = 8760 hours (1 year)
λ = 1/8760 = 0.000114 failures/hour

Probability of failure in 30 days (720 hours):
P(failure) = 1 - e^(-0.000114 x 720) = 1 - 0.921 = 7.9%
```

---

## High Availability Architectures

### Single Points of Failure (SPOF)

```
Identifying SPOFs:

                    ┌─────────────┐
     SPOF ─────────►│ Load Balancer│◄───────── SPOF
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼─────┐  ┌──────▼─────┐  ┌──────▼─────┐
    │  Server 1  │  │  Server 2  │  │  Server 3  │ ◄─ Redundant
    └──────┬─────┘  └──────┬─────┘  └──────┬─────┘
           │               │               │
           └───────────────┼───────────────┘
                           │
                    ┌──────▼──────┐
     SPOF ─────────►│   Database  │◄───────── SPOF
                    └─────────────┘
```

### Eliminating SPOFs

```
                    ┌─────────────┐
                    │     DNS     │ (Multiple providers)
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼─────┐ ┌────▼────┐ ┌─────▼──────┐
       │   LB 1     │ │  LB 2   │ │   LB 3     │ (Active-Active)
       └──────┬─────┘ └────┬────┘ └─────┬──────┘
              │            │            │
              └────────────┼────────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │                      │                      │
┌───▼───┐  ┌───────┐  ┌───▼───┐  ┌───────┐  ┌───▼───┐
│Server1│  │Server2│  │Server3│  │Server4│  │Server5│
└───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘
    │          │          │          │          │
    └──────────┼──────────┼──────────┼──────────┘
               │          │          │
        ┌──────▼──────┐  ┌▼──────────▼┐
        │  Primary DB │  │ Replica DB │ (With auto-failover)
        └─────────────┘  └────────────┘
```

### Availability by Architecture

| Architecture | Typical Availability | Notes |
|--------------|---------------------|-------|
| Single server | 99% | Basic deployment |
| Active-passive | 99.9% | Manual/auto failover |
| Active-active (2 nodes) | 99.99% | Load balanced |
| Active-active (3+ nodes) | 99.99%+ | Distributed |
| Multi-region | 99.999% | Geographic redundancy |
| Multi-cloud | 99.9999% | Ultimate redundancy |

---

## Worked Examples

### Example 1: Three-Tier Web Application

**Given**:
- Load balancer: 99.99% availability
- Web servers: 2 servers, each 99% available
- App servers: 3 servers, each 99% available
- Database: Primary + replica with failover, 99.9% combined

**Calculate system availability**:

```
Step 1: Web server tier (parallel)
A_web = 1 - (1 - 0.99)^2 = 1 - 0.0001 = 99.99%

Step 2: App server tier (parallel)
A_app = 1 - (1 - 0.99)^3 = 1 - 0.000001 = 99.9999%

Step 3: System availability (serial)
A_system = 0.9999 x 0.9999 x 0.999999 x 0.999
         = 0.9999 x 0.9999 x 0.999999 x 0.999
         = 0.9988 = 99.88%

Downtime per year: ~10.5 hours
```

### Example 2: Multi-Region Deployment

**Given**:
- 3 regions, each with 99.9% availability
- Traffic can failover to any region
- Cross-region latency acceptable

**Calculate system availability**:

```
With geographic redundancy (any region can serve traffic):
A_system = 1 - (1 - 0.999)^3
         = 1 - (0.001)^3
         = 1 - 0.000000001
         = 0.999999999 = 99.9999999%

This is theoretical - in practice, failover adds delay:
- Failover detection: ~30 seconds
- DNS propagation: ~60 seconds
- Total: ~90 seconds per failure

Realistic availability with failover overhead:
If 3 failures per year, 90 seconds each = 4.5 minutes
4.5 minutes / 525,600 minutes = 0.0000086
Practical availability: 99.999% (five nines)
```

### Example 3: Database Availability

**Given**:
- Primary database: 99.5% availability
- Synchronous replica: 99.5% availability
- Failover time: 30 seconds

**Calculate combined availability**:

```
Step 1: Parallel availability (either can serve reads)
A_parallel = 1 - (1 - 0.995)^2 = 1 - 0.000025 = 99.9975%

Step 2: Account for failover time
Average failures per year (for 99.5%):
Downtime = 0.005 x 8760 hours = 43.8 hours
If MTTR = 2 hours, failures = 43.8 / 2 = ~22 failures/year

Failover overhead: 22 x 30 seconds = 11 minutes/year

Step 3: Final availability
Total downtime: (1 - 0.999975) x 525,600 + 11 min
              = 13.14 + 11 = 24.14 minutes/year

Availability = 1 - (24.14 / 525,600) = 99.995%
```

---

## Common Availability Targets by System Type

### Industry Standards

| System Type | Typical SLA | Rationale |
|-------------|-------------|-----------|
| Internal tools | 99% | Non-critical |
| B2B SaaS | 99.9% | Standard enterprise |
| E-commerce | 99.95% | Revenue impact |
| Financial services | 99.99% | Regulatory |
| Healthcare | 99.99% | Life-critical |
| Cloud infrastructure | 99.99% | Platform dependency |
| Telecommunications | 99.999% | Critical infrastructure |

### Cloud Provider SLAs

| Provider | Service | SLA |
|----------|---------|-----|
| AWS | EC2 | 99.99% |
| AWS | S3 | 99.99% |
| AWS | RDS Multi-AZ | 99.95% |
| Google Cloud | Compute Engine | 99.99% |
| Azure | Virtual Machines | 99.99% |
| Cloudflare | CDN | 100% (with credits) |

---

## Practice Problems

### Problem 1: E-commerce Platform

**Given**:
An e-commerce platform has:
- CDN: 99.99%
- Web tier: 4 servers, 99% each
- API tier: 3 servers, 99.5% each
- Database: 99.9%
- Payment gateway: 99.95%

Calculate overall availability for a purchase flow.

<details>
<summary>Solution</summary>

```
Step 1: Web tier (parallel)
A_web = 1 - (1 - 0.99)^4 = 1 - 0.00000001 = 99.999999%

Step 2: API tier (parallel)
A_api = 1 - (1 - 0.995)^3 = 1 - 0.000000125 = 99.9999875%

Step 3: Serial combination (all required for purchase)
A_total = 0.9999 x 0.99999999 x 0.999999875 x 0.999 x 0.9995
        = 0.9999 x 0.999999 x 0.999999 x 0.999 x 0.9995
        = 0.9984 = 99.84%

Downtime per year: ~14 hours

Bottleneck: Database (99.9%) and Payment Gateway (99.95%)
```

</details>

### Problem 2: SLA Breach Calculation

**Given**:
- Monthly SLA: 99.9%
- Incidents this month:
  - Incident 1: 15 minutes
  - Incident 2: 8 minutes
  - Incident 3: 25 minutes
- Service credit structure: 10% for <99.9%, 25% for <99%

Did we breach SLA? What's the credit?

<details>
<summary>Solution</summary>

```
Total downtime: 15 + 8 + 25 = 48 minutes

Minutes in month (30 days): 43,200

Allowed downtime for 99.9%:
= 43,200 x 0.001 = 43.2 minutes

Actual availability:
= (43,200 - 48) / 43,200 = 0.99889 = 99.889%

SLA breached? Yes (99.889% < 99.9%)

Credit tier: <99.9% but >= 99.0% = 10% credit

For $50,000/month contract:
Credit = $50,000 x 10% = $5,000
```

</details>

### Problem 3: Designing for Five Nines

**Given**:
- Target: 99.999% availability
- Current single-server setup: 99.5%
- Can add redundant servers (same 99.5% each)
- Failover time between servers: 10 seconds

How many servers needed?

<details>
<summary>Solution</summary>

```
Target availability: 99.999%
Required: 1 - (1 - 0.995)^n >= 0.99999

Solving:
(1 - 0.995)^n <= 0.00001
(0.005)^n <= 0.00001

Taking log:
n x log(0.005) <= log(0.00001)
n x (-2.301) <= -5
n >= 2.17

So n = 3 servers gives:
A = 1 - (0.005)^3 = 1 - 0.000000125 = 99.9999875%

But we need to account for failover time:
Expected failures per year per server:
Downtime at 99.5% = 43.8 hours/year
If MTTR = 4 hours, failures = ~11 per server

With 3 servers, probability of needing failover:
P(2 fail simultaneously) is very low, but failovers occur

Estimate ~15 failovers/year x 10 seconds = 2.5 minutes

Additional unavailability: 2.5 / 525,600 = 0.0000048

Final availability: 99.9999875% - 0.00048% = 99.9995%

This exceeds 99.999%, so 3 servers is sufficient.
```

</details>

---

## Interview Tips

### Do's

1. **Know the nines table by heart** - Quick mental math impresses
2. **Identify SPOFs immediately** - Show you think about failure modes
3. **Consider both MTBF and MTTR** - Recovery time often matters more
4. **Calculate error budgets** - Shows operational maturity
5. **Mention monitoring** - Can't manage what you can't measure

### Don'ts

1. **Don't promise five nines casually** - It's extremely hard
2. **Don't forget planned maintenance** - It counts against availability
3. **Don't ignore cascading failures** - One failure can cause others
4. **Don't assume perfect failover** - Always has latency
5. **Don't forget partial failures** - Degraded != down

### Key Talking Points

1. "Each nine of availability is 10x harder and more expensive"
2. "Focus on MTTR - it's usually easier to improve than MTBF"
3. "The weakest link determines overall availability"
4. "Redundancy helps, but you need to eliminate correlation"
5. "Error budgets help balance reliability and velocity"

---

## Quick Reference Card

```
AVAILABILITY QUICK REFERENCE

The Nines:
99%     = 3.65 days downtime/year
99.9%   = 8.76 hours downtime/year
99.99%  = 52.6 minutes downtime/year
99.999% = 5.26 minutes downtime/year

Formulas:
Serial:   A_total = A1 x A2 x A3 x ...
Parallel: A_total = 1 - (1-A1) x (1-A2) x ...
MTBF/MTTR: A = MTBF / (MTBF + MTTR)

Quick Math:
Minutes/month = 43,200
Minutes/year = 525,600
Hours/year = 8,760

Rule of Thumb:
- 2 servers at 99% = 99.99%
- 3 servers at 99% = 99.9999%
- Adding a serial component reduces availability
- Adding a parallel component increases availability
```

---

## Summary

Availability math is fundamental to system design:

1. **Understand the nines** - Each nine is 10x more difficult
2. **Serial vs parallel** - Serial multiplies, parallel compounds
3. **MTBF and MTTR** - Focus on recovery time
4. **SLA and error budgets** - Balance reliability with velocity
5. **No single points of failure** - Redundancy at every layer
6. **Real-world factors** - Failover time, correlated failures, maintenance

The key insight: **achieving high availability requires eliminating single points of failure at every layer while ensuring fast detection and recovery from failures.**
