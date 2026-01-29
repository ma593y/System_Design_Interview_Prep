# System Design Study Roadmap

## Overview

This roadmap provides a structured approach to preparing for system design interviews. Adjust the timeline based on your experience level and available time.

---

## Phase 1: Foundation (Week 1-2)

### Week 1: Core Concepts

**Day 1-2: Scalability & Load Balancing**
- [ ] Read `01_Fundamentals/01_scalability.md`
- [ ] Read `01_Fundamentals/02_load_balancing.md`
- [ ] Practice: Explain vertical vs horizontal scaling

**Day 3-4: Caching & Databases**
- [ ] Read `01_Fundamentals/03_caching.md`
- [ ] Read `01_Fundamentals/04_databases.md`
- [ ] Practice: When would you use Redis vs Memcached?

**Day 5-6: CAP Theorem & Consistency**
- [ ] Read `01_Fundamentals/06_cap_theorem.md`
- [ ] Read `04_Strategies_and_Approaches/03_consistency_strategies.md`
- [ ] Practice: Explain CAP theorem with real examples

**Day 7: Review & Practice**
- [ ] Review all concepts from the week
- [ ] Do estimation practice from `07_Key_Metrics_and_Calculations/06_back_of_envelope.md`

### Week 2: More Fundamentals

**Day 1-2: Networking & APIs**
- [ ] Read `01_Fundamentals/05_networking.md`
- [ ] Read `01_Fundamentals/08_api_design.md`
- [ ] Practice: Design a RESTful API for a simple service

**Day 3-4: Message Queues & Storage**
- [ ] Read `01_Fundamentals/07_message_queues.md`
- [ ] Read `01_Fundamentals/10_storage_systems.md`
- [ ] Practice: When would you use async processing?

**Day 5-6: Security & Concurrency**
- [ ] Read `01_Fundamentals/09_security.md`
- [ ] Read `01_Fundamentals/11_concurrency.md`
- [ ] Practice: Explain OAuth 2.0 flow

**Day 7: Review & First Case Study**
- [ ] Complete `08_Case_Studies/beginner/url_shortener.md`

---

## Phase 2: Building Blocks & Patterns (Week 3-4)

### Week 3: Infrastructure Components

**Day 1-2: DNS, CDN & Proxies**
- [ ] Read `02_Building_Blocks/01_dns_and_cdn.md`
- [ ] Read `02_Building_Blocks/02_proxies.md`
- [ ] Practice: Trace a request from browser to server

**Day 3-4: Distributed Systems**
- [ ] Read `02_Building_Blocks/03_distributed_systems.md`
- [ ] Read `02_Building_Blocks/04_data_partitioning.md`
- [ ] Practice: Explain consistent hashing

**Day 5-6: Replication & Indexing**
- [ ] Read `02_Building_Blocks/05_replication.md`
- [ ] Read `02_Building_Blocks/06_indexing.md`
- [ ] Practice: Compare B-trees vs LSM trees

**Day 7: Case Study**
- [ ] Complete `08_Case_Studies/beginner/rate_limiter.md`

### Week 4: Design Patterns

**Day 1-2: Microservices & Event-Driven**
- [ ] Read `03_Design_Patterns/01_microservices.md`
- [ ] Read `03_Design_Patterns/02_event_driven.md`
- [ ] Practice: When to use monolith vs microservices?

**Day 3-4: Data & Resilience Patterns**
- [ ] Read `03_Design_Patterns/03_data_patterns.md`
- [ ] Read `03_Design_Patterns/04_resilience_patterns.md`
- [ ] Practice: Explain circuit breaker pattern

**Day 5-6: Caching & Deployment Patterns**
- [ ] Read `03_Design_Patterns/05_caching_patterns.md`
- [ ] Read `03_Design_Patterns/06_deployment_patterns.md`
- [ ] Practice: Compare cache-aside vs write-through

**Day 7: Case Study**
- [ ] Complete `08_Case_Studies/beginner/key_value_store.md`

---

## Phase 3: Advanced Topics (Week 5-6)

### Week 5: Strategies & Best Practices

**Day 1-2: Scaling & Data Modeling**
- [ ] Read `04_Strategies_and_Approaches/01_scaling_strategies.md`
- [ ] Read `04_Strategies_and_Approaches/02_data_modeling.md`

**Day 3-4: Availability & Performance**
- [ ] Read `04_Strategies_and_Approaches/04_availability_strategies.md`
- [ ] Read `04_Strategies_and_Approaches/05_performance_optimization.md`

**Day 5-6: Best Practices**
- [ ] Read `05_Best_Practices/01_api_best_practices.md`
- [ ] Read `05_Best_Practices/05_monitoring_observability.md`

**Day 7: Intermediate Case Study**
- [ ] Complete `08_Case_Studies/intermediate/twitter.md`

### Week 6: Trade-offs & Metrics

**Day 1-3: Trade-offs Deep Dive**
- [ ] Read all files in `06_Trade_Offs/`
- [ ] Practice articulating trade-offs for each decision

**Day 4-5: Key Metrics**
- [ ] Read `07_Key_Metrics_and_Calculations/01_latency_numbers.md`
- [ ] Read `07_Key_Metrics_and_Calculations/07_common_numbers.md`
- [ ] Memorize key numbers

**Day 6-7: Case Studies**
- [ ] Complete `08_Case_Studies/intermediate/uber.md`
- [ ] Complete `08_Case_Studies/intermediate/whatsapp.md`

---

## Phase 4: Interview Mastery (Week 7-8)

### Week 7: Interview Framework

**Day 1-2: Approach & Requirements**
- [ ] Read `09_Interview_Framework/01_approach.md`
- [ ] Read `09_Interview_Framework/02_requirements_gathering.md`

**Day 3-4: Communication & Diagrams**
- [ ] Read `09_Interview_Framework/04_diagram_techniques.md`
- [ ] Read `09_Interview_Framework/05_communication_tips.md`

**Day 5-6: Questions Prep**
- [ ] Read all files in `10_Interview_Questions/`
- [ ] Practice answering questions aloud

**Day 7: Advanced Case Study**
- [ ] Complete `08_Case_Studies/advanced/youtube.md`

### Week 8: Final Preparation

**Day 1-2: Quick Reference Review**
- [ ] Review `11_Quick_Reference/01_cheat_sheet.md`
- [ ] Review `11_Quick_Reference/02_formula_sheet.md`

**Day 3-5: Mock Interviews**
- [ ] Practice with mock problems from `12_Practice/01_mock_problems.md`
- [ ] Do timed practice from `12_Practice/02_timed_exercises.md`

**Day 6-7: Self Assessment & Gaps**
- [ ] Complete `12_Practice/03_self_assessment.md`
- [ ] Review any weak areas identified

---

## Quick Reference: Daily Practice

### 15-Minute Daily Habits

1. **Estimation Practice**: Do one back-of-envelope calculation
2. **Concept Review**: Re-read one concept from fundamentals
3. **Diagram Practice**: Draw one system component from memory

### Weekly Goals

- Complete 2-3 case studies
- Do at least one 45-minute mock interview
- Review and update progress tracker

---

## Adjusting the Timeline

### If You Have Less Time (2-4 weeks)

Focus on:
1. Core fundamentals (Week 1)
2. Key case studies (URL shortener, Twitter, YouTube)
3. Interview framework
4. Quick reference materials

### If You Have More Time (3+ months)

Add:
1. All advanced case studies
2. Deep dive into specific technologies
3. More mock interviews
4. Study group practice

---

## Success Metrics

You're ready when you can:

- [ ] Explain any fundamental concept in under 2 minutes
- [ ] Do capacity estimation quickly and accurately
- [ ] Draw a high-level system design in 10 minutes
- [ ] Articulate trade-offs for any design decision
- [ ] Answer follow-up questions confidently
- [ ] Complete a full design in 45 minutes
