# Microservices Architecture

## Overview

Microservices architecture structures an application as a collection of loosely coupled, independently deployable services.

```
Monolith:                    Microservices:
┌─────────────────────┐     ┌─────────┐ ┌─────────┐ ┌─────────┐
│                     │     │  User   │ │  Order  │ │ Payment │
│   All Features      │     │ Service │ │ Service │ │ Service │
│   in One Codebase   │     └─────────┘ └─────────┘ └─────────┘
│                     │     ┌─────────┐ ┌─────────┐ ┌─────────┐
│                     │     │Inventory│ │Shipping │ │  Email  │
│                     │     │ Service │ │ Service │ │ Service │
└─────────────────────┘     └─────────┘ └─────────┘ └─────────┘
```

## Key Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Single Responsibility** | Each service does one thing well |
| **Independent Deployment** | Deploy without affecting other services |
| **Decentralized Data** | Each service owns its data |
| **Technology Agnostic** | Different services can use different tech |
| **Fault Isolation** | Failure in one doesn't crash all |

## Service Decomposition

### Domain-Driven Design (DDD)

```
E-commerce Domain:
┌────────────────────────────────────────────────────┐
│                                                    │
│  ┌──────────────┐  ┌──────────────┐              │
│  │   Catalog    │  │   Customer   │              │
│  │  Subdomain   │  │  Subdomain   │              │
│  │              │  │              │              │
│  │ - Products   │  │ - Users      │              │
│  │ - Categories │  │ - Profiles   │              │
│  │ - Search     │  │ - Addresses  │              │
│  └──────────────┘  └──────────────┘              │
│                                                    │
│  ┌──────────────┐  ┌──────────────┐              │
│  │    Order     │  │   Payment    │              │
│  │  Subdomain   │  │  Subdomain   │              │
│  │              │  │              │              │
│  │ - Cart       │  │ - Payments   │              │
│  │ - Orders     │  │ - Refunds    │              │
│  │ - Fulfillment│  │ - Invoices   │              │
│  └──────────────┘  └──────────────┘              │
│                                                    │
└────────────────────────────────────────────────────┘

Each subdomain → One or more microservices
```

### Bounded Contexts

```
Same term, different meanings in different contexts:

Catalog Context:          Order Context:
┌─────────────────┐      ┌─────────────────┐
│     Product     │      │     Product     │
├─────────────────┤      ├─────────────────┤
│ - name          │      │ - productId     │
│ - description   │      │ - quantity      │
│ - category      │      │ - priceAtOrder  │
│ - price         │      │ - discounts     │
│ - inventory     │      └─────────────────┘
└─────────────────┘

Each context defines its own model
```

## Communication Patterns

### Synchronous Communication

#### REST APIs

```
Order Service ──HTTP GET──▶ User Service
              ◀─ JSON ─────

Simple, stateless, widely understood
Best for: Simple request-response
```

#### gRPC

```
Order Service ══ gRPC ══▶ Inventory Service
              ◀═ Protobuf ══

Fast, binary protocol, streaming support
Best for: Internal service-to-service, high performance
```

### Asynchronous Communication

#### Message Queue

```
Order Service ──▶ [Message Queue] ──▶ Email Service
                                  ──▶ Inventory Service
                                  ──▶ Analytics Service

Fire and forget, decoupled
Best for: Non-blocking operations, fan-out
```

#### Event-Driven

```
Order Service ──▶ OrderCreated Event ──▶ [Event Bus]
                                              │
                    ┌─────────────────────────┼─────────────────┐
                    ▼                         ▼                 ▼
              Email Service          Inventory Service    Analytics
              (send receipt)         (reserve stock)      (track)

Services react to events, fully decoupled
```

### Communication Comparison

| Pattern | Latency | Coupling | Use Case |
|---------|---------|----------|----------|
| REST | Higher | Moderate | External APIs |
| gRPC | Lower | Moderate | Internal high-perf |
| Message Queue | Async | Low | Background tasks |
| Events | Async | Lowest | Reactive systems |

## Service Discovery

How services find each other.

### Client-Side Discovery

```
┌─────────────┐     ┌──────────────────┐
│   Client    │────▶│ Service Registry │
│             │◀────│ (Consul/Eureka)  │
└──────┬──────┘     └──────────────────┘
       │
       │ Direct call with discovered address
       ▼
┌─────────────┐
│   Service   │
└─────────────┘

Client fetches addresses, load balances itself
```

### Server-Side Discovery

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│   Client    │────▶│  Load Balancer  │────▶│   Service   │
└─────────────┘     └────────┬────────┘     └─────────────┘
                             │
                    ┌────────▼────────┐
                    │ Service Registry│
                    └─────────────────┘

LB handles discovery and routing
```

## Data Management

### Database per Service

```
┌─────────────────┐     ┌─────────────────┐
│  Order Service  │     │  User Service   │
└────────┬────────┘     └────────┬────────┘
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │Order DB │            │ User DB │
    │(Postgres)│           │(MongoDB)│
    └─────────┘            └─────────┘

Each service has its own database
No direct database access between services
```

### Data Consistency Patterns

#### Saga Pattern

```
Distributed transaction using compensating actions:

1. Order Service: Create order
2. Inventory Service: Reserve stock
3. Payment Service: Charge card
4. Shipping Service: Create shipment

If step 3 fails:
- Compensate step 2: Release stock
- Compensate step 1: Cancel order
```

#### Event Sourcing + CQRS

```
Commands ──▶ Event Store ──▶ Events ──▶ Read Models
                                            │
                                      Queries
```

## API Gateway

Single entry point for all clients.

```
┌─────────────────────────────────────────────────────┐
│                    API Gateway                       │
├─────────────────────────────────────────────────────┤
│  • Authentication                                    │
│  • Rate Limiting                                     │
│  • Request Routing                                   │
│  • Response Aggregation                              │
│  • Protocol Translation                              │
└───────────────────────┬─────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ Service │    │ Service │    │ Service │
   │    A    │    │    B    │    │    C    │
   └─────────┘    └─────────┘    └─────────┘
```

## Challenges

### Distributed System Complexity

| Challenge | Solution |
|-----------|----------|
| Network failures | Retry, circuit breaker, timeout |
| Data consistency | Saga, eventual consistency |
| Debugging | Distributed tracing |
| Testing | Contract testing |

### Operational Overhead

```
Monolith: Deploy 1 thing
Microservices: Deploy 50+ things

Need:
- Container orchestration (Kubernetes)
- CI/CD pipelines
- Centralized logging
- Monitoring/alerting
- Service mesh
```

## When to Use Microservices

### Good Fit

- Large teams (Conway's Law)
- Need independent scaling
- Different tech requirements
- Complex, evolving domain
- High availability requirements

### Poor Fit

- Small team (< 10 engineers)
- Simple domain
- Early-stage startup
- Unclear domain boundaries
- Tight deadlines

## Migration Strategy

### Strangler Fig Pattern

```
Phase 1:           Phase 2:           Phase 3:
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Monolith   │   │   Monolith   │   │ (deprecated) │
│              │   │  (smaller)   │   │              │
│  ┌────────┐  │   │              │   │              │
│  │Feature │  │   │              │   │              │
│  │   A    │  │   │              │   │              │
│  └────────┘  │   └──────────────┘   └──────────────┘
└──────────────┘           │
                    ┌──────▼──────┐   ┌──────────────┐
                    │  Service A  │   │  Service A   │
                    └─────────────┘   │  Service B   │
                                      │  Service C   │
                                      └──────────────┘

Gradually extract services from monolith
```

## Interview Talking Points

1. "Microservices trade development simplicity for operational complexity"
2. "Service boundaries should align with domain boundaries"
3. "Each service should own its data"
4. "Async communication reduces coupling"
5. "Start with a monolith, extract services when needed"

## Common Interview Questions

1. **Q: How do you decompose a monolith into microservices?**
   A: Use DDD to identify bounded contexts, start with clear domain boundaries, extract services incrementally (strangler pattern).

2. **Q: How do you handle transactions across services?**
   A: Saga pattern with compensating transactions, or embrace eventual consistency with event-driven architecture.

3. **Q: How do microservices communicate?**
   A: Sync (REST, gRPC) for immediate response, async (messages, events) for decoupling. Choose based on use case.

4. **Q: What are the drawbacks of microservices?**
   A: Operational complexity, distributed system challenges, network latency, data consistency challenges, debugging difficulty.

## Key Takeaways

- Microservices enable independent deployment and scaling
- Domain-driven design helps define service boundaries
- Choose communication patterns based on coupling needs
- Each service owns its data
- Start simple, add complexity when needed

## Further Reading

- "Building Microservices" by Sam Newman
- "Domain-Driven Design" by Eric Evans
- Netflix, Uber engineering blogs
- Microservices.io patterns catalog
