# Monolith vs Microservices

## Overview

Choosing between monolithic and microservices architecture is one of the most impactful decisions in system design.

## Architecture Comparison

### Monolithic Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Monolithic Application                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Single Codebase                         │  │
│  │                                                            │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │  │
│  │  │   User   │  │  Order   │  │ Payment  │  │ Inventory│ │  │
│  │  │  Module  │  │  Module  │  │  Module  │  │  Module  │ │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │  │
│  │       │             │             │             │        │  │
│  │       └─────────────┴─────────────┴─────────────┘        │  │
│  │                         │                                 │  │
│  │                  Shared Database                          │  │
│  │                   ┌──────────┐                           │  │
│  │                   │    DB    │                           │  │
│  │                   └──────────┘                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Single Deployment Unit                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Microservices Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   Microservices Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ User Service │  │Order Service │  │Payment Service│          │
│  │              │  │              │  │              │          │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │          │
│  │  │   DB   │  │  │  │   DB   │  │  │  │   DB   │  │          │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         └─────────────────┼─────────────────┘                   │
│                           │                                      │
│                    ┌──────▼──────┐                              │
│                    │ API Gateway │                              │
│                    └─────────────┘                              │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │  Inventory   │  │Notification  │                            │
│  │   Service    │  │  Service     │                            │
│  │  ┌────────┐  │  │  ┌────────┐  │                            │
│  │  │   DB   │  │  │  │   DB   │  │                            │
│  │  └────────┘  │  │  └────────┘  │                            │
│  └──────────────┘  └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

## Comparison Table

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | Single unit | Independent services |
| **Scaling** | Scale entire app | Scale individual services |
| **Development** | Simpler initial setup | Complex infrastructure |
| **Team Structure** | Single team | Multiple autonomous teams |
| **Technology** | Single stack | Polyglot possible |
| **Data** | Shared database | Database per service |
| **Failure** | Total failure risk | Partial failure isolation |
| **Latency** | In-process calls | Network calls |
| **Testing** | Simpler E2E | Complex integration testing |
| **Debugging** | Straightforward | Distributed tracing needed |

## When to Choose Monolith

### Ideal Scenarios

```
✅ Early-stage startup
   - Fast iteration
   - Small team (< 10 developers)
   - Unclear domain boundaries

✅ Simple domain
   - Few business capabilities
   - Limited scaling needs
   - Single team ownership

✅ Tight deadline
   - Need to ship quickly
   - Prove concept first
   - Can refactor later

✅ Limited DevOps expertise
   - No Kubernetes experience
   - Simple deployment preferred
   - Minimal infrastructure
```

### Monolith Advantages

```
1. Simplicity
   └── Single codebase, single deployment, single monitoring

2. Development Speed (initially)
   └── No network calls, easy refactoring, simple testing

3. Debugging
   └── Stack traces, single log stream, IDE debugging

4. Transactions
   └── ACID across all modules, no distributed transactions

5. Performance
   └── In-process calls, shared memory, no serialization
```

### Monolith Disadvantages

```
1. Scaling Limitations
   └── Must scale entire app, even for one hot module

2. Deployment Risk
   └── Small change = full redeployment
   └── One bug can take down everything

3. Technology Lock-in
   └── Stuck with initial choices
   └── Hard to adopt new technologies

4. Team Coordination
   └── Merge conflicts
   └── Stepping on each other's toes
   └── Slower as team grows

5. Long-term Maintainability
   └── Becomes "big ball of mud"
   └── Harder to understand over time
```

## When to Choose Microservices

### Ideal Scenarios

```
✅ Large organization
   - Multiple teams (> 20 developers)
   - Clear domain boundaries
   - Different scaling needs per component

✅ Complex domain
   - Many distinct business capabilities
   - Different reliability requirements
   - Different technology needs

✅ Scale requirements
   - Need to scale components independently
   - High availability critical
   - Global distribution

✅ Strong DevOps culture
   - Container orchestration experience
   - CI/CD pipelines mature
   - Monitoring/observability in place
```

### Microservices Advantages

```
1. Independent Scaling
   └── Scale only what needs scaling
   └── Cost-efficient resource usage

2. Independent Deployment
   └── Deploy services independently
   └── Faster release cycles
   └── Reduced deployment risk

3. Technology Flexibility
   └── Best tool for each job
   └── Easier to adopt new tech
   └── Gradual modernization

4. Team Autonomy
   └── Teams own services end-to-end
   └── Parallel development
   └── Clear boundaries

5. Fault Isolation
   └── One service failure doesn't take down all
   └── Graceful degradation possible
```

### Microservices Disadvantages

```
1. Operational Complexity
   ┌────────────────────────────────────────────────┐
   │ Need: Service discovery, load balancing,       │
   │       distributed tracing, centralized logging,│
   │       container orchestration, CI/CD pipelines │
   └────────────────────────────────────────────────┘

2. Network Latency
   └── Every service call = network hop
   └── Serialization/deserialization overhead

3. Data Consistency
   └── No ACID across services
   └── Eventual consistency challenges
   └── Saga patterns needed

4. Testing Complexity
   └── Integration testing across services
   └── Contract testing needed
   └── Environment management

5. Debugging Difficulty
   └── Distributed tracing required
   └── Logs across multiple services
   └── Harder to reproduce issues
```

## The Evolution Path

### Starting with Monolith

```
Phase 1: Monolith (Day 1)
┌─────────────────────────────────┐
│         Monolith                │
│  ┌───────────────────────────┐  │
│  │ All modules in one app    │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘

Phase 2: Modular Monolith (Growth)
┌─────────────────────────────────┐
│         Modular Monolith        │
│  ┌─────────┐ ┌─────────┐       │
│  │Module A │ │Module B │ ...   │
│  │(clean   │ │(clean   │       │
│  │boundary)│ │boundary)│       │
│  └─────────┘ └─────────┘       │
└─────────────────────────────────┘

Phase 3: Extract Services (Scale)
┌─────────────┐  ┌─────────────┐
│  Service A  │  │  Remaining  │
│ (extracted) │  │  Monolith   │
└─────────────┘  └─────────────┘
```

### Strangler Fig Pattern

```
Gradually replace monolith functionality:

Step 1: Facade in front of monolith
┌─────────────────────────────────────────────────┐
│                   Facade                         │
│                     │                            │
│          ┌──────────┴──────────┐                │
│          ▼                     ▼                │
│    ┌──────────┐         ┌──────────┐           │
│    │ Monolith │         │ New Svc  │           │
│    │ (all)    │         │ (empty)  │           │
│    └──────────┘         └──────────┘           │
└─────────────────────────────────────────────────┘

Step 2: Move functionality gradually
┌─────────────────────────────────────────────────┐
│                   Facade                         │
│          ┌──────────┴──────────┐                │
│          ▼                     ▼                │
│    ┌──────────┐         ┌──────────┐           │
│    │ Monolith │         │ New Svc  │           │
│    │ (less)   │         │ (more)   │           │
│    └──────────┘         └──────────┘           │
└─────────────────────────────────────────────────┘

Step 3: Complete migration
┌─────────────────────────────────────────────────┐
│                   Facade                         │
│                     │                            │
│                     ▼                            │
│              ┌──────────┐                       │
│              │ New Svc  │                       │
│              │ (all)    │                       │
│              └──────────┘                       │
│     ┌──────────┐                                │
│     │ Monolith │ ← Deprecated                   │
│     │ (empty)  │                                │
│     └──────────┘                                │
└─────────────────────────────────────────────────┘
```

## Hybrid Approaches

### Service-Oriented Monolith

```
┌─────────────────────────────────────────────────┐
│              Service-Oriented Monolith           │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │        Internal Service Boundaries        │   │
│  │                                          │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐    │   │
│  │  │User Svc│  │Order   │  │Payment │    │   │
│  │  │(module)│  │(module)│  │(module)│    │   │
│  │  └────────┘  └────────┘  └────────┘    │   │
│  │       │           │           │         │   │
│  │       └───────────┴───────────┘         │   │
│  │            Clean interfaces              │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  Single deployment, clear boundaries            │
└─────────────────────────────────────────────────┘

Benefits:
- Simpler operations than microservices
- Clear module boundaries
- Easy to extract later
```

### Macro Services

```
Not micro, not mono - something in between:

┌────────────────────────────────────────────────┐
│              Macro Services                     │
│                                                 │
│  ┌───────────────────┐  ┌───────────────────┐ │
│  │   User Domain     │  │   Order Domain    │ │
│  │                   │  │                   │ │
│  │  - Auth          │  │  - Orders        │ │
│  │  - Profile       │  │  - Cart          │ │
│  │  - Preferences   │  │  - Checkout      │ │
│  │                   │  │  - Inventory     │ │
│  └───────────────────┘  └───────────────────┘ │
│                                                 │
│  Fewer, larger services (domain-oriented)      │
└────────────────────────────────────────────────┘
```

## Decision Framework

### Size and Stage Matrix

| Team Size | Stage | Recommendation |
|-----------|-------|----------------|
| < 5 | MVP | Monolith |
| 5-15 | Growth | Modular Monolith |
| 15-50 | Scale | Start extracting services |
| 50+ | Enterprise | Microservices (if needed) |

### Decision Checklist

```
Choose Microservices if most are TRUE:
□ Multiple teams (> 3) working on the system
□ Different components have vastly different scaling needs
□ Need to use different technologies for different components
□ Have strong DevOps/platform team
□ Clear domain boundaries exist
□ Independent deployability is critical
□ Can afford operational complexity

Choose Monolith if most are TRUE:
□ Small team (< 10)
□ Need to move fast
□ Domain boundaries unclear
□ Limited DevOps expertise
□ Tight deadline
□ Simple deployment preferred
```

## Real-World Examples

| Company | Architecture | Notes |
|---------|--------------|-------|
| Amazon | Started monolith → Microservices | Famously "two-pizza teams" |
| Netflix | Microservices | Pioneers of cloud microservices |
| Etsy | Monolith | Successful with well-organized monolith |
| Shopify | Modular Monolith | Chose not to go microservices |
| Uber | Microservices | Thousands of services |
| Basecamp | Monolith | Small team, simple deployment |

## Interview Talking Points

1. "Start with a monolith unless you have a specific reason for microservices"

2. "The biggest driver for microservices is organizational - team independence"

3. "Microservices trade development complexity for operational complexity"

4. "Modular monolith gives you the best of both - clear boundaries, simple operations"

5. "Don't split until you have clear domain boundaries"

## Common Interview Questions

1. **Q: When would you choose microservices over monolith?**
   A: When I have multiple teams needing independent deployment, vastly different scaling needs, or need technology flexibility. Also when domain boundaries are clear and I have DevOps expertise.

2. **Q: What are the main challenges with microservices?**
   A: Distributed systems complexity, network latency, data consistency across services, debugging/tracing, and operational overhead.

3. **Q: How would you migrate from monolith to microservices?**
   A: Strangler fig pattern - gradually extract services while routing through a facade. Start with clear, independent domains.

4. **Q: Can a monolith scale?**
   A: Yes, through horizontal scaling (multiple instances), database sharding, caching, and read replicas. Microservices just offer more granular scaling.

## Key Takeaways

- Start simple (monolith), evolve if needed
- Organizational structure often drives architecture
- Modular monolith is a valid middle ground
- Microservices add operational complexity
- Clear domain boundaries are prerequisite for good services

## Further Reading

- "Monolith First" by Martin Fowler
- "Building Microservices" by Sam Newman
- "Microservices Patterns" by Chris Richardson
- Netflix and Amazon engineering blogs
