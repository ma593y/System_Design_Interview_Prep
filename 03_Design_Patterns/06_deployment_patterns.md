# Deployment Patterns

## Overview

Deployment patterns reduce risk and enable reliable software releases.

## Blue-Green Deployment

Two identical environments, switch traffic between them.

```
Before:
┌─────────────────────────────────────────────────────┐
│                    Load Balancer                     │
│                         │                            │
│                         ▼                            │
│        ┌─────────────────────────────┐              │
│        │      Blue (v1.0)            │ ◀── ACTIVE  │
│        │      Production             │              │
│        └─────────────────────────────┘              │
│                                                      │
│        ┌─────────────────────────────┐              │
│        │      Green (v1.0)           │ ◀── STANDBY │
│        │      Idle                   │              │
│        └─────────────────────────────┘              │
└─────────────────────────────────────────────────────┘

Deploy v1.1 to Green:
┌─────────────────────────────────────────────────────┐
│                    Load Balancer                     │
│                         │                            │
│                         ▼                            │
│        ┌─────────────────────────────┐              │
│        │      Blue (v1.0)            │ ◀── ACTIVE  │
│        └─────────────────────────────┘              │
│                                                      │
│        ┌─────────────────────────────┐              │
│        │      Green (v1.1)           │ ◀── TESTING │
│        │      Deploy + Test          │              │
│        └─────────────────────────────┘              │
└─────────────────────────────────────────────────────┘

Switch traffic to Green:
┌─────────────────────────────────────────────────────┐
│                    Load Balancer                     │
│                         │                            │
│                         ▼                            │
│        ┌─────────────────────────────┐              │
│        │      Blue (v1.0)            │ ◀── STANDBY │
│        └─────────────────────────────┘   (rollback) │
│                                                      │
│        ┌─────────────────────────────┐              │
│        │      Green (v1.1)           │ ◀── ACTIVE  │
│        └─────────────────────────────┘              │
└─────────────────────────────────────────────────────┘
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Instant rollback | Double infrastructure cost |
| Zero downtime | Database migrations complex |
| Full testing before switch | Need to keep environments in sync |

### When to Use

- Critical production systems
- Need instant rollback capability
- Can afford duplicate infrastructure

## Canary Deployment

Gradually roll out to subset of users.

```
Phase 1: 1% Canary
┌─────────────────────────────────────────────────────┐
│                    Load Balancer                     │
│                    ┌────┴────┐                       │
│                99% │         │ 1%                    │
│                    ▼         ▼                       │
│        ┌─────────────┐  ┌─────────────┐            │
│        │  v1.0       │  │  v1.1       │            │
│        │  (stable)   │  │  (canary)   │            │
│        └─────────────┘  └─────────────┘            │
└─────────────────────────────────────────────────────┘

Phase 2: 10% Canary (if metrics OK)
┌─────────────────────────────────────────────────────┐
│                    Load Balancer                     │
│                    ┌────┴────┐                       │
│                90% │         │ 10%                   │
│                    ▼         ▼                       │
│        ┌─────────────┐  ┌─────────────┐            │
│        │  v1.0       │  │  v1.1       │            │
│        └─────────────┘  └─────────────┘            │
└─────────────────────────────────────────────────────┘

Phase 3: 100% (complete rollout)
```

### Canary Metrics to Monitor

| Metric | Threshold |
|--------|-----------|
| Error rate | < baseline + 0.1% |
| Latency p99 | < baseline + 10% |
| CPU/Memory | < baseline + 20% |
| Business metrics | Conversion, revenue |

### Automatic Canary Analysis

```
┌─────────────────────────────────────────────────────┐
│              Canary Analysis Service                 │
│                                                      │
│  Compare canary vs baseline:                        │
│  - Error rates                                      │
│  - Latency percentiles                             │
│  - Custom business metrics                         │
│                                                      │
│  Decision:                                          │
│  ✓ PASS → Continue rollout                         │
│  ✗ FAIL → Automatic rollback                       │
└─────────────────────────────────────────────────────┘
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Limits blast radius | Slower rollout |
| Real production testing | Complex routing |
| Data-driven decisions | Need good metrics |

## Rolling Deployment

Gradually replace instances one at a time.

```
Initial: All v1.0
[v1.0] [v1.0] [v1.0] [v1.0] [v1.0]

Step 1: Replace first instance
[v1.1] [v1.0] [v1.0] [v1.0] [v1.0]
  ↑
  Deploy + health check

Step 2: Replace second instance
[v1.1] [v1.1] [v1.0] [v1.0] [v1.0]

...

Final: All v1.1
[v1.1] [v1.1] [v1.1] [v1.1] [v1.1]
```

### Kubernetes Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Extra pods during update
      maxUnavailable: 0  # Keep all pods available
```

### Pros and Cons

| Pros | Cons |
|------|------|
| No extra infrastructure | Slow rollback |
| Gradual rollout | Two versions running |
| Resource efficient | Must handle version compatibility |

## A/B Testing

Route traffic based on user segments for experimentation.

```
┌─────────────────────────────────────────────────────┐
│                    Traffic Router                    │
│                                                      │
│  User in experiment group A?                        │
│  ┌────────────────────┬────────────────────┐        │
│  │ Yes (50%)          │ No (50%)           │        │
│  ▼                    ▼                    │        │
│  ┌─────────────┐     ┌─────────────┐              │
│  │  Feature A  │     │  Feature B  │              │
│  │  (variant)  │     │  (control)  │              │
│  └─────────────┘     └─────────────┘              │
│                                                      │
│  Measure: Conversion, engagement, revenue           │
└─────────────────────────────────────────────────────┘
```

### A/B vs Canary

| Aspect | A/B Testing | Canary |
|--------|-------------|--------|
| Purpose | Feature comparison | Safe rollout |
| Metrics | Business metrics | Technical metrics |
| Duration | Until statistical significance | Until full rollout |
| Routing | User-based | Random/percentage |

## Feature Flags

Toggle features without deployment.

```
┌─────────────────────────────────────────────────────┐
│                  Feature Flag Service                │
│                                                      │
│  Flags:                                             │
│  ├── new_checkout_flow: false                       │
│  ├── dark_mode: true                               │
│  ├── premium_features:                             │
│  │   └── enabled_for: [user_group: "beta"]         │
│  └── search_v2:                                    │
│      └── enabled_for: [percentage: 10]             │
└─────────────────────────────────────────────────────┘

Code:
if feature_flags.is_enabled("new_checkout_flow", user):
    return new_checkout()
else:
    return old_checkout()
```

### Flag Types

| Type | Use Case |
|------|----------|
| Release flag | Control feature rollout |
| Experiment flag | A/B testing |
| Ops flag | Kill switches, maintenance mode |
| Permission flag | Feature access control |

### Best Practices

```
1. Clean up old flags
   - Set expiration dates
   - Remove after full rollout

2. Avoid flag dependencies
   - Flag A depends on Flag B = complexity

3. Test both states
   - On and off paths

4. Document flag purpose
   - Why it exists, when to remove
```

## Shadow Deployment

Run new version in parallel without affecting users.

```
┌─────────────────────────────────────────────────────┐
│                   Traffic Splitter                   │
│                         │                            │
│              ┌──────────┴──────────┐                │
│              ▼ (real)              ▼ (copy)         │
│        ┌──────────┐          ┌──────────┐          │
│        │  v1.0    │          │  v1.1    │          │
│        │ (active) │          │ (shadow) │          │
│        └────┬─────┘          └────┬─────┘          │
│             │                     │                 │
│        User gets                Logs only          │
│        this response            (compare)          │
└─────────────────────────────────────────────────────┘
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Test with real traffic | Double compute cost |
| No user impact | No write testing |
| Compare responses | Complex setup |

### When to Use

- Major refactors
- Performance testing
- Migration validation

## Comparison Table

| Pattern | Risk | Speed | Rollback | Cost |
|---------|------|-------|----------|------|
| Blue-Green | Low | Fast | Instant | High |
| Canary | Low | Slow | Fast | Medium |
| Rolling | Medium | Medium | Slow | Low |
| A/B Test | Medium | N/A | Fast | Medium |
| Shadow | None | N/A | N/A | High |

## Deployment Decision Tree

```
Need instant rollback?
├── Yes → Blue-Green
└── No
    └── Need real-world testing?
        ├── Yes
        │   └── Business metrics or technical?
        │       ├── Business → A/B Testing
        │       └── Technical → Canary
        └── No
            └── Resource constrained?
                ├── Yes → Rolling
                └── No → Blue-Green
```

## Database Migrations

### Expand and Contract

```
Phase 1: Expand (add new)
┌─────────────────────────────────────────────────────┐
│ Table: users                                         │
│ ┌──────────────────────────────────────────────────┐│
│ │ name (old)  │  full_name (new)  │               ││
│ │ "Alice"     │  NULL             │               ││
│ └──────────────────────────────────────────────────┘│
│                                                      │
│ Code writes to both columns                         │
└─────────────────────────────────────────────────────┘

Phase 2: Migrate data
┌─────────────────────────────────────────────────────┐
│ ┌──────────────────────────────────────────────────┐│
│ │ name (old)  │  full_name (new)  │               ││
│ │ "Alice"     │  "Alice"          │               ││
│ └──────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘

Phase 3: Contract (remove old)
┌─────────────────────────────────────────────────────┐
│ ┌──────────────────────────────────────────────────┐│
│ │ full_name (new)  │                              ││
│ │ "Alice"          │                              ││
│ └──────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

## Interview Talking Points

1. "Blue-green for instant rollback, canary for gradual rollout"
2. "Feature flags separate deployment from release"
3. "Rolling deployment is resource-efficient but slower"
4. "A/B testing is for business metrics, canary for technical"
5. "Database migrations need expand-contract pattern"

## Common Interview Questions

1. **Q: How would you deploy a critical production change?**
   A: Canary deployment with automated rollback. Start at 1%, monitor metrics, gradually increase if healthy.

2. **Q: What's the difference between canary and A/B testing?**
   A: Canary tests technical stability (errors, latency). A/B tests business impact (conversion, engagement).

3. **Q: How do you handle database schema changes during deployment?**
   A: Expand-contract pattern. Add new columns, migrate data, then remove old columns. Ensures backward compatibility.

4. **Q: When would you use feature flags instead of canary?**
   A: When you need user-targeted rollout, quick kill switch, or testing features independent of deployment.

## Key Takeaways

- Match deployment strategy to risk tolerance
- Automate rollback based on metrics
- Feature flags enable flexible rollouts
- Database migrations need special handling
- Monitor everything during deployments

## Further Reading

- "Continuous Delivery" by Jez Humble
- Kubernetes deployment strategies
- LaunchDarkly feature flags
- Spinnaker deployment pipelines
