# System Design Interview Preparation

A comprehensive guide for mastering system design interviews, covering fundamentals to advanced topics.

## How to Use This Repository

### Recommended Study Order

1. **Start with Fundamentals** (`01_Fundamentals/`) - Build your foundation
2. **Learn Building Blocks** (`02_Building_Blocks/`) - Understand core components
3. **Study Design Patterns** (`03_Design_Patterns/`) - Learn common architectures
4. **Review Strategies** (`04_Strategies_and_Approaches/`) - Understand decision-making
5. **Practice with Case Studies** (`08_Case_Studies/`) - Apply your knowledge
6. **Master the Interview Framework** (`09_Interview_Framework/`) - Learn how to present
7. **Use Quick Reference** (`11_Quick_Reference/`) - For quick revision

### Study Tips

- **Active Learning**: Don't just read - draw diagrams, explain concepts aloud
- **Practice Estimation**: Do back-of-envelope calculations regularly
- **Mock Interviews**: Practice with peers or use the self-assessment guide
- **Understand Trade-offs**: Every design decision has pros and cons

## Repository Structure

```
├── 00_Study_Roadmap/          # Week-by-week study plan
├── 01_Fundamentals/           # Core concepts (scaling, caching, databases)
├── 02_Building_Blocks/        # Infrastructure components (DNS, CDN, proxies)
├── 03_Design_Patterns/        # Architectural patterns (microservices, event-driven)
├── 04_Strategies_and_Approaches/  # Decision-making frameworks
├── 05_Best_Practices/         # Production-grade practices
├── 06_Trade_Offs/             # Common trade-off analysis
├── 07_Key_Metrics_and_Calculations/  # Numbers and estimation
├── 08_Case_Studies/           # Real-world system designs
│   ├── beginner/              # URL shortener, Pastebin, Rate limiter
│   ├── intermediate/          # Twitter, Uber, WhatsApp
│   └── advanced/              # YouTube, Google Search, Payment systems
├── 09_Interview_Framework/    # How to approach interviews
├── 10_Interview_Questions/    # Common questions by category
├── 11_Quick_Reference/        # Cheat sheets and quick lookups
└── 12_Practice/               # Mock problems and assessments
```

## Key Concepts Overview

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                  │
│                    (Web, Mobile, IoT)                           │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CDN / DNS                                   │
│              (Content Delivery, Name Resolution)                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER                                 │
│            (Traffic Distribution, Health Checks)                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   API GATEWAY                                    │
│        (Authentication, Rate Limiting, Routing)                  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│   Service A   │ │   Service B   │ │   Service C   │
│  (Stateless)  │ │  (Stateless)  │ │  (Stateless)  │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        │                 │                 │
        └────────┬────────┴────────┬────────┘
                 │                 │
        ┌────────▼────────┐ ┌──────▼──────┐
        │     CACHE       │ │   MESSAGE   │
        │  (Redis/Memcached)│ │    QUEUE    │
        └────────┬────────┘ └──────┬──────┘
                 │                 │
        ┌────────▼─────────────────▼────────┐
        │           DATABASES               │
        │   (SQL, NoSQL, Search, Graph)     │
        └───────────────────────────────────┘
```

### Numbers Every Engineer Should Know

| Operation | Latency |
|-----------|---------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory reference | 100 ns |
| SSD random read | 150 μs |
| HDD seek | 10 ms |
| Round trip within datacenter | 500 μs |
| Round trip CA to Netherlands | 150 ms |

### Quick Estimation Guide

- **1 Million users**: Start thinking about scaling
- **10 Million users**: Need horizontal scaling
- **100 Million users**: Need sophisticated architecture

## Getting Started

1. Check your current level with `12_Practice/03_self_assessment.md`
2. Follow the roadmap in `00_Study_Roadmap/roadmap.md`
3. Track your progress with `00_Study_Roadmap/progress_tracker.md`

Good luck with your interviews!
