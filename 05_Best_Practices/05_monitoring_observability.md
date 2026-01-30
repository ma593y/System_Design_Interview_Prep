# Monitoring & Observability Best Practices

## Table of Contents
1. [Three Pillars of Observability](#1-three-pillars-of-observability)
2. [Metrics Collection](#2-metrics-collection)
3. [Structured Logging](#3-structured-logging)
4. [Distributed Tracing](#4-distributed-tracing)
5. [Alerting Strategy](#5-alerting-strategy)
6. [Dashboards](#6-dashboards)
7. [SLI/SLO/SLA Monitoring](#7-slislosla-monitoring)
8. [Health Check Patterns](#8-health-check-patterns)
9. [Error Budgets and SRE Practices](#9-error-budgets-and-sre-practices)
10. [Log Aggregation](#10-log-aggregation)
11. [Application Performance Monitoring](#11-application-performance-monitoring)
12. [Common Observability Anti-Patterns](#12-common-observability-anti-patterns)
13. [Interview Talking Points](#13-interview-talking-points)

---

## 1. Three Pillars of Observability

```
  +─────────────────────────────────────────────────────────+
  |                  OBSERVABILITY                           |
  |                                                         |
  |  +-----------+   +-----------+   +------------------+   |
  |  |  METRICS  |   |   LOGS    |   |     TRACES       |   |
  |  |           |   |           |   |                  |   |
  |  | What is   |   | What      |   | How does a       |   |
  |  | happening |   | happened  |   | request flow     |   |
  |  | right now?|   | in detail?|   | across services? |   |
  |  |           |   |           |   |                  |   |
  |  | Counters  |   | Structured|   | Span trees       |   |
  |  | Gauges    |   | Events    |   | Timing data      |   |
  |  | Histograms|   | Context   |   | Dependencies     |   |
  |  +-----------+   +-----------+   +------------------+   |
  |                                                         |
  |  Together: Full picture of system behavior              |
  +─────────────────────────────────────────────────────────+
```

### When to Use Each Pillar

| Scenario                              | Metrics | Logs | Traces |
|--------------------------------------|---------|------|--------|
| System is slow                       | First   | Second| Third |
| Error rate spike                     | Detect  | Debug | Root cause |
| Capacity planning                    | Primary | No    | No     |
| Debugging specific request failure   | No      | Primary| Primary|
| Understanding service dependencies   | No      | No    | Primary|
| Long-term trend analysis             | Primary | No    | No     |
| Compliance/audit trail               | No      | Primary| No    |

---

## 2. Metrics Collection

### Metric Types

```
  COUNTER: Monotonically increasing value
  ──────────────────────────────────────
  http_requests_total: 0, 1, 2, 3, 5, 8, 13...
  Use for: Request count, errors, bytes transferred

  GAUGE: Value that goes up and down
  ──────────────────────────────────────
  cpu_utilization: 45%, 62%, 38%, 71%, 55%...
  Use for: CPU, memory, queue depth, connections

  HISTOGRAM: Distribution of values in buckets
  ──────────────────────────────────────
  request_duration_seconds:
    bucket{le="0.05"}: 2400    (50ms)
    bucket{le="0.1"}:  3200    (100ms)
    bucket{le="0.25"}: 3800    (250ms)
    bucket{le="0.5"}:  3950    (500ms)
    bucket{le="1.0"}:  3990    (1s)
    bucket{le="+Inf"}: 4000    (total)
  Use for: Latency distributions, response sizes

  SUMMARY: Pre-calculated quantiles
  ──────────────────────────────────────
  request_duration{quantile="0.5"}:  0.042
  request_duration{quantile="0.9"}:  0.089
  request_duration{quantile="0.99"}: 0.245
  Use for: Client-side quantile calculations
```

### Key Metrics by Component

| Component       | Metrics to Collect                                        |
|----------------|----------------------------------------------------------|
| Web Server     | Request rate, error rate, latency (p50/p95/p99)           |
| Database       | Query latency, connections active/idle, replication lag    |
| Cache          | Hit rate, miss rate, evictions, memory usage               |
| Queue          | Queue depth, consumer lag, processing rate                 |
| Application    | GC pauses, thread count, heap usage, custom business metrics|
| Infrastructure | CPU, memory, disk I/O, network I/O                        |

### Prometheus Architecture

```
  +──────────+     +──────────+     +──────────+
  | Service A|     | Service B|     | Service C|
  | /metrics |     | /metrics |     | /metrics |
  +────┬─────+     +────┬─────+     +────┬─────+
       |                |                |
       +────────────────┼────────────────+
                        |
                   Pull (scrape)
                        |
                 +──────┴──────+
                 | Prometheus  |
                 | Server      |
                 | (TSDB)      |
                 +──────┬──────+
                        |
              +---------+---------+
              |                   |
       +──────┴──────+    +──────┴──────+
       | Grafana     |    | AlertManager|
       | (visualize) |    | (alerts)    |
       +─────────────+    +─────────────+
```

### Tool Comparison

| Tool          | Type          | Strengths                    | Cost Model        |
|--------------|---------------|------------------------------|-------------------|
| Prometheus   | Open source   | Kubernetes native, PromQL    | Free (self-hosted)|
| CloudWatch   | AWS managed   | AWS integration, no setup    | Per metric/alarm  |
| Datadog      | SaaS          | Full stack, easy setup       | Per host/month    |
| Grafana Cloud| SaaS/OSS      | Visualization, multi-source  | Per series/month  |
| InfluxDB     | Open source   | High write throughput        | Free (self-hosted)|

---

## 3. Structured Logging

### Unstructured vs Structured

```
  UNSTRUCTURED (hard to parse):
  2024-01-15 10:23:45 ERROR - Failed to process order 12345 for user john@example.com

  STRUCTURED (JSON - machine parseable):
  {
    "timestamp": "2024-01-15T10:23:45.123Z",
    "level": "ERROR",
    "service": "order-service",
    "instance": "order-svc-pod-abc123",
    "trace_id": "abc-def-123-456",
    "span_id": "span-789",
    "message": "Failed to process order",
    "order_id": "12345",
    "user_id": "user-678",
    "error_type": "PaymentDeclined",
    "error_message": "Insufficient funds",
    "duration_ms": 245
  }
```

### Correlation IDs

```
  Request Flow with Correlation ID:

  Client                API Gateway          Order Service        Payment Service
    |                      |                     |                      |
    |─── POST /orders ───>|                     |                      |
    |  X-Request-ID:      |                     |                      |
    |  req-abc-123        |                     |                      |
    |                      |─── CreateOrder ───>|                      |
    |                      |  trace: req-abc-123 |                      |
    |                      |                     |─── ChargeCard ─────>|
    |                      |                     |  trace: req-abc-123  |
    |                      |                     |<──── Response ──────|
    |                      |<──── Response ─────|                      |
    |<──── 201 Created ───|                     |                      |

  All logs from all services include trace_id: req-abc-123
  Search for "req-abc-123" returns complete request flow
```

### Log Levels

| Level  | When to Use                                       | Example                           |
|--------|--------------------------------------------------|-----------------------------------|
| TRACE  | Detailed debugging (not in production)             | Variable values, loop iterations  |
| DEBUG  | Diagnostic information for developers              | DB query executed, cache hit/miss |
| INFO   | Normal operations, business events                 | Order created, user logged in     |
| WARN   | Unexpected but recoverable situations              | Retry attempt, slow query         |
| ERROR  | Operation failed but service continues             | Payment failed, API timeout       |
| FATAL  | Service cannot continue                            | DB connection lost, OOM           |

### Logging Best Practices

```
  DO:
  - Include context (user_id, request_id, trace_id)
  - Use structured format (JSON)
  - Log at appropriate levels
  - Include timing information
  - Redact sensitive data (PII, passwords, tokens)
  - Set retention policies

  DON'T:
  - Log sensitive data (passwords, credit cards, tokens)
  - Log at DEBUG level in production
  - Use string concatenation for log messages
  - Log in tight loops (use sampling)
  - Log entire request/response bodies (too much data)
  - Ignore log volume costs
```

---

## 4. Distributed Tracing

### Trace Structure

```
  Trace ID: abc-123

  +─────────────────────────────────────────────────────────+
  | API Gateway (span 1)                                     |
  | Duration: 245ms                                          |
  |  +──────────────────────────────────────────────────+    |
  |  | Order Service (span 2)                            |    |
  |  | Duration: 200ms                                   |    |
  |  |  +─────────────────────────+                      |    |
  |  |  | DB Query (span 3)       |                      |    |
  |  |  | Duration: 15ms          |                      |    |
  |  |  +─────────────────────────+                      |    |
  |  |       +──────────────────────────────────────+    |    |
  |  |       | Payment Service (span 4)              |    |    |
  |  |       | Duration: 150ms                       |    |    |
  |  |       |  +──────────────────────+             |    |    |
  |  |       |  | Stripe API (span 5)  |             |    |    |
  |  |       |  | Duration: 120ms      |             |    |    |
  |  |       |  +──────────────────────+             |    |    |
  |  |       +──────────────────────────────────────+    |    |
  |  +──────────────────────────────────────────────────+    |
  +─────────────────────────────────────────────────────────+

  Insight: 120ms of 245ms total (49%) spent in Stripe API call
```

### OpenTelemetry Architecture

```
  +──────────+    +──────────+    +──────────+
  | Service A|    | Service B|    | Service C|
  | (SDK)    |    | (SDK)    |    | (SDK)    |
  +────┬─────+    +────┬─────+    +────┬─────+
       |               |               |
       +───────────────┼───────────────+
                       |
              +────────┴────────+
              | OTel Collector  |
              | (agent/gateway) |
              +────────┬────────+
                       |
          +────────────┼────────────+
          |            |            |
   +──────┴──+  +──────┴──+  +─────┴───+
   | Jaeger  |  |Prometheus|  | Loki    |
   | (traces)|  | (metrics)|  | (logs)  |
   +---------+  +----------+  +---------+
```

### Tool Comparison

| Tool            | Type          | Language Support | Storage Backend     |
|----------------|---------------|------------------|---------------------|
| Jaeger         | Open source   | All major        | Cassandra, ES, Kafka|
| Zipkin         | Open source   | All major        | MySQL, ES, Cassandra|
| AWS X-Ray      | Managed       | AWS SDKs         | AWS managed         |
| Datadog APM    | SaaS          | All major        | Datadog managed     |
| Honeycomb      | SaaS          | All major        | Honeycomb managed   |

### Sampling Strategies

```
  Without sampling: 100% of traces stored
  Cost: $$$$ at high volume

  Head-based sampling (decision at start):
  - Sample 10% of all traces
  - Deterministic based on trace ID
  - Pro: Simple, low overhead
  - Con: May miss interesting traces

  Tail-based sampling (decision at end):
  - Keep all traces with errors
  - Keep all slow traces (> 500ms)
  - Sample 5% of normal traces
  - Pro: Captures all interesting traces
  - Con: Must buffer traces before decision
```

---

## 5. Alerting Strategy

### Severity Levels

| Severity | Description                    | Response Time | Who Gets Paged  |
|----------|-------------------------------|---------------|-----------------|
| P1 / SEV1| Service down, data loss       | Immediate     | On-call + manager|
| P2 / SEV2| Major feature degraded        | < 30 min      | On-call engineer |
| P3 / SEV3| Minor feature affected        | < 4 hours     | Email/Slack      |
| P4 / SEV4| Cosmetic, non-urgent          | Next sprint   | Ticket only      |

### Alert Routing

```
  +──────────────+
  | Alert Fires  |
  +──────┬───────+
         |
  +──────┴───────+
  | AlertManager |
  +──┬──┬──┬──┬──+
     |  |  |  |
     |  |  |  +──> P4: JIRA ticket
     |  |  +─────> P3: Slack #alerts channel
     |  +────────> P2: PagerDuty (on-call)
     +───────────> P1: PagerDuty (on-call + escalation)
```

### Good Alert Design

```
  BAD ALERT:
  "CPU is high on server-123"
  - No context, no action, no severity

  GOOD ALERT:
  +──────────────────────────────────────────────────────+
  | ALERT: [P2] Order Service - High Error Rate          |
  |                                                      |
  | Summary: Error rate is 5.2% (threshold: 1%)          |
  | Impact: ~1 in 20 orders failing                      |
  | Since: 2024-01-15 14:32 UTC (12 minutes ago)         |
  | Service: order-service (us-east-1)                   |
  |                                                      |
  | Dashboard: https://grafana.internal/d/orders         |
  | Runbook: https://wiki.internal/runbooks/order-errors |
  |                                                      |
  | Recent Changes:                                      |
  | - Deploy v2.3.4 at 14:28 UTC                         |
  +──────────────────────────────────────────────────────+
```

### Alerting Best Practices

```
  1. Alert on symptoms, not causes
     BAD:  Alert when CPU > 80%
     GOOD: Alert when response latency p99 > 500ms

  2. Alert on SLO burn rate
     BAD:  Alert on every error
     GOOD: Alert when error budget consumption rate means
           you will exhaust budget within 24 hours

  3. Every alert should be actionable
     If nobody does anything when it fires, delete it

  4. Use multiple notification windows
     Fast burn:  >2% budget in 1 hour  -> page immediately
     Slow burn:  >5% budget in 6 hours -> Slack notification
     Slow drain: >10% budget in 3 days -> email/ticket
```

---

## 6. Dashboards

### Key Dashboards to Build

```
  1. SERVICE OVERVIEW DASHBOARD
  +─────────────────────────────────────────────+
  |  Request Rate     Error Rate    Latency p99 |
  |  [2,450 RPS]     [0.05%]       [89ms]      |
  |                                              |
  |  Request Rate Over Time    Error Rate Chart  |
  |  ┌─────────────────┐      ┌──────────────┐  |
  |  │    ___           │      │              │  |
  |  │___/   \___       │      │   _          │  |
  |  │           \___   │      │__/ \_        │  |
  |  └─────────────────┘      └──────────────┘  |
  |                                              |
  |  Latency Distribution     Active Instances   |
  |  ┌─────────────────┐      ┌──────────────┐  |
  |  │  ___             │      │              │  |
  |  │ /   \            │      │ ████████  8  │  |
  |  │/     \___        │      │              │  |
  |  └─────────────────┘      └──────────────┘  |
  +─────────────────────────────────────────────+

  2. INFRASTRUCTURE DASHBOARD
  - CPU utilization per host/container
  - Memory usage and trends
  - Disk I/O and space
  - Network throughput and errors

  3. DATABASE DASHBOARD
  - Query latency (read/write)
  - Connection pool utilization
  - Replication lag
  - Slow query count
  - Cache hit ratio

  4. BUSINESS METRICS DASHBOARD
  - Orders per minute
  - Revenue per hour
  - User signups
  - Conversion funnel
```

### Dashboard Design Principles

| Principle              | Description                                          |
|-----------------------|------------------------------------------------------|
| Audience appropriate  | Exec dashboard != on-call dashboard                   |
| Use the RED method    | Rate, Errors, Duration for every service              |
| Time ranges matter    | Show last 1h, 24h, 7d for context                    |
| Annotations           | Mark deploys, incidents on charts                     |
| Consistent colors     | Green=good, Yellow=warn, Red=critical everywhere      |
| Drill-down links      | Overview -> Service -> Instance -> Trace              |

---

## 7. SLI/SLO/SLA Monitoring

### Defining SLIs

```
  For an API service:

  Availability SLI:
    Successful requests / Total requests
    (exclude health checks and load balancer checks)

  Latency SLI:
    Requests served < 200ms / Total requests
    (measure at the load balancer, not the application)

  Correctness SLI:
    Correct responses / Total responses
    (verify response body matches expected schema)
```

### SLO Dashboard

```
  +────────────────────────────────────────────────────+
  |  SLO Status - January 2024                         |
  |                                                    |
  |  Availability: 99.95%  Target: 99.9%  [ON TRACK]  |
  |  Budget remaining: 82%                             |
  |  ████████████████████░░░░                          |
  |                                                    |
  |  Latency (p99 < 200ms): 99.7%  Target: 99%        |
  |  Budget remaining: 70%                [ON TRACK]   |
  |  ██████████████████░░░░░░                          |
  |                                                    |
  |  Error Rate: 0.08%  Target: < 0.1%   [AT RISK]    |
  |  Budget remaining: 20%                             |
  |  ████████░░░░░░░░░░░░░░░░                          |
  +────────────────────────────────────────────────────+
```

---

## 8. Health Check Patterns

### Kubernetes Probe Types

```
  Pod Lifecycle:

  Starting ──[startup probe passes]──> Running
                                          |
                                   [liveness probe]
                                   fails 3x -> restart
                                          |
                                   [readiness probe]
                                   fails -> remove from
                                            service endpoints

  Configuration:
  livenessProbe:
    httpGet:
      path: /health/live
      port: 8080
    initialDelaySeconds: 15
    periodSeconds: 10
    failureThreshold: 3

  readinessProbe:
    httpGet:
      path: /health/ready
      port: 8080
    periodSeconds: 5
    failureThreshold: 2

  startupProbe:
    httpGet:
      path: /health/startup
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
```

### Health Check Depth Levels

| Level  | Checks                              | Use Case            |
|--------|-------------------------------------|---------------------|
| Shallow| Process alive, port open             | Liveness probe      |
| Medium | + Database connected, cache reachable| Readiness probe     |
| Deep   | + Full dependency chain verified     | Synthetic monitoring|

---

## 9. Error Budgets and SRE Practices

### Error Budget Calculation

```
  SLO: 99.9% availability
  Budget period: 30 days

  Total minutes:  30 x 24 x 60 = 43,200 minutes
  Error budget:   43,200 x 0.001 = 43.2 minutes of downtime allowed

  Week 1: 5 minutes downtime   -> 38.2 minutes remaining (88%)
  Week 2: 12 minutes downtime  -> 26.2 minutes remaining (61%)
  Week 3: 0 minutes downtime   -> 26.2 minutes remaining (61%)
  Week 4: 10 minutes downtime  -> 16.2 minutes remaining (38%)

  Status: On track but declining. Need to be cautious with deploys.
```

### Error Budget Policy

```
  Budget > 50%:  Normal operations
                 Deploy freely, experiment

  Budget 25-50%: Caution
                 Extra review for risky deploys
                 Increase testing

  Budget < 25%:  Restricted
                 Only critical bug fixes deployed
                 Focus on reliability work

  Budget = 0%:   Freeze
                 No feature deployments
                 All hands on reliability
                 Post-mortem required
```

### Incident Management

```
  Incident Lifecycle:

  Detection ──> Triage ──> Mitigation ──> Resolution ──> Post-mortem
  (minutes)    (minutes)   (minutes-hrs)  (hours-days)   (within 48hrs)

  DORA Metrics:
  +───────────────────────+─────────────────────────────────+
  | Metric                | Elite Performance               |
  +───────────────────────+─────────────────────────────────+
  | Deploy frequency      | Multiple times per day          |
  | Lead time for changes | Less than one hour              |
  | Change failure rate   | 0-15%                           |
  | MTTR                  | Less than one hour              |
  +───────────────────────+─────────────────────────────────+
```

---

## 10. Log Aggregation

### ELK Stack Architecture

```
  +──────────+  +──────────+  +──────────+
  | Service A|  | Service B|  | Service C|
  +────┬─────+  +────┬─────+  +────┬─────+
       |              |              |
  +────┴─────+  +────┴─────+  +────┴─────+
  | Filebeat |  | Filebeat |  | Filebeat |
  +────┬─────+  +────┬─────+  +────┬─────+
       |              |              |
       +──────────────┼──────────────+
                      |
               +──────┴──────+
               |   Logstash  |  (parse, transform, enrich)
               +──────┬──────+
                      |
               +──────┴──────+
               |Elasticsearch|  (index, store, search)
               +──────┬──────+
                      |
               +──────┴──────+
               |   Kibana    |  (visualize, explore)
               +─────────────+
```

### Loki Architecture (Lightweight Alternative)

```
  +──────────+
  | Services |
  +────┬─────+
       |
  +────┴─────+
  | Promtail |  (log shipping agent)
  +────┬─────+
       |
  +────┴─────+
  |   Loki   |  (stores logs, indexes labels only)
  +────┬─────+
       |
  +────┴─────+
  | Grafana  |  (query with LogQL)
  +──────────+

  Key difference: Loki only indexes metadata (labels),
  not full text. Much cheaper to operate.
```

### Log Retention Strategy

| Log Type          | Hot (searchable)| Warm (archived) | Cold (compliance) |
|-------------------|----------------|-----------------|-------------------|
| Application logs  | 7 days         | 30 days         | 1 year            |
| Access logs       | 30 days        | 90 days         | 3 years           |
| Security logs     | 90 days        | 1 year          | 7 years           |
| Debug logs        | 1 day          | Not retained    | Not retained      |

---

## 11. Application Performance Monitoring

### APM Components

```
  +──────────────────────────────────────────────────+
  |               APM Platform                        |
  |                                                   |
  |  +────────────+  +──────────+  +──────────────+  |
  |  | Transaction|  | Error    |  | Infrastructure|  |
  |  | Tracing    |  | Tracking |  | Monitoring    |  |
  |  +────────────+  +──────────+  +──────────────+  |
  |                                                   |
  |  +────────────+  +──────────+  +──────────────+  |
  |  | Real User  |  | Synthetic|  | Service Map  |  |
  |  | Monitoring |  | Monitoring| |               |  |
  |  +────────────+  +──────────+  +──────────────+  |
  +──────────────────────────────────────────────────+
```

### Key APM Features

| Feature               | Purpose                                    |
|----------------------|-------------------------------------------|
| Transaction tracing  | Follow request across all services         |
| Code-level profiling | Identify slow methods/functions            |
| Error grouping       | Aggregate similar errors                   |
| Dependency mapping   | Visualize service relationships            |
| Database monitoring  | Track slow queries, N+1 detection          |
| Real user monitoring | Measure actual user experience (browser)   |
| Synthetic monitoring | Proactive checks from external locations   |

---

## 12. Common Observability Anti-Patterns

| Anti-Pattern                    | Impact                          | Solution                     |
|--------------------------------|--------------------------------|------------------------------|
| Alert fatigue                  | Alerts ignored                  | Reduce noise, alert on SLOs  |
| No runbooks linked to alerts   | Slow incident response          | Attach runbook to every alert|
| Monitoring only infrastructure | Miss application issues         | Add RED metrics per service  |
| No correlation between pillars | Hard to debug                   | Use trace IDs across all three|
| Unbounded log retention        | Runaway storage costs           | Set retention policies       |
| Missing business metrics       | Cannot assess user impact       | Track orders, signups, etc.  |
| Dashboard sprawl               | Nobody looks at dashboards      | Curate key dashboards        |
| High-cardinality metrics       | Prometheus OOM                  | Limit label values           |
| Sampling all traces            | Miss error traces               | Tail-based sampling          |
| No baseline measurements       | Cannot tell if metric is normal | Establish baselines first    |

---

## 13. Interview Talking Points

### When to Bring Up Observability

- When designing any distributed system
- When discussing operational readiness
- When asked how you would debug production issues
- When discussing trade-offs (mention monitoring as validation)

### Key Phrases

```
  "I would instrument the service with the RED method: request rate,
   error rate, and duration. These metrics feed into SLO-based alerts
   so we get notified only when user experience is degraded."

  "For debugging, I would use distributed tracing with correlation IDs
   propagated through every service call. This lets us trace a single
   request across the entire system."

  "I would set up structured logging with JSON format and include
   trace IDs in every log entry, so we can correlate logs with traces
   for any specific request."

  "I would define error budgets based on our SLOs. When we have budget
   remaining, we can ship features aggressively. When budget is low,
   we shift focus to reliability."
```

### Sample Interview Question

**Interviewer:** "How would you monitor this system in production?"

**Strong answer:** "I would implement all three pillars of observability. For metrics, I would
use the RED method on every service, collecting request rate, error rate, and latency
distributions. These would power Grafana dashboards and SLO-based alerts that page on-call
only when error budgets are burning too fast. For logging, I would use structured JSON logs
with correlation IDs, aggregated in a system like Loki or ELK. For tracing, I would use
OpenTelemetry to instrument all service-to-service calls, letting us trace any request across
the system. I would build four key dashboards: service overview, infrastructure, database, and
business metrics. Alerts would be routed through PagerDuty with clear severity levels and
attached runbooks."
