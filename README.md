# System Design Interview Preparation Guide

## 📚 Complete Study Plan for Senior/Staff/Principal Engineers

**Target Audience:** Senior Backend Engineers (5-8+ years), Staff/Platform Engineers, Java/Spring Boot/Microservices/Cloud engineers preparing for system design interviews at FAANG+ companies.

**Estimated Study Time:** 8-10 weeks

---

## 📖 Document Structure

### [Phase 0: Foundations (Mandatory Prerequisites)](./00-foundations.md)
The fundamental concepts you MUST understand before any system design interview. These form the vocabulary and mental models that interviewers expect you to apply naturally.

- Latency vs Throughput
- Availability vs Consistency (CAP)
- Scalability vs Elasticity
- Stateful vs Stateless Services
- Horizontal vs Vertical Scaling
- Read-heavy vs Write-heavy Systems
- Strong vs Eventual Consistency
- Idempotency

### [Phase 1: Core Building Blocks](./01-core-building-blocks.md)
The essential components you'll use in every system design. Master these before moving to distributed systems.

- **Networking & API Design:** REST vs gRPC, versioning, pagination, rate limiting, auth
- **Load Balancing:** L4 vs L7, algorithms, sticky sessions
- **Databases:** SQL (indexing, transactions, sharding), NoSQL (types, partitioning)
- **Caching:** Strategies, eviction policies, cache patterns

### [Phase 2: Distributed Systems Core](./02-distributed-systems.md)
The concepts that separate senior engineers from mid-level. These are where most candidates struggle.

- Horizontal Sharding & Consistent Hashing
- Replication Strategies
- CAP Theorem (Practical View)
- Consensus (Paxos/Raft conceptual)
- Message Queues & Event-Driven Architecture

### [Phase 3: Scalability Patterns](./03-scalability-patterns.md)
Architectural patterns for building resilient, scalable systems. Critical for Staff+ interviews.

- Monolith → Microservices Evolution
- Saga Pattern
- Resilience Patterns (Circuit Breaker, Retries, Bulkheads)
- Observability (Metrics, Logging, Tracing)

### [Phase 4: System Design Case Studies](./04-case-studies-mandatory.md)
Complete end-to-end design walkthroughs with interviewer perspective.

**Mandatory Designs (Must Know):**
- URL Shortener
- Rate Limiter
- Notification System
- File Storage (S3-like)
- News Feed / Timeline
- Payment System
- Chat / Messaging System

### [Phase 4b: Senior+ Case Studies](./04b-case-studies-advanced.md)
Advanced designs expected at Staff/Principal level.

- Distributed Cache (Redis-like)
- Search Autocomplete
- Metrics / Monitoring System
- Feature Flag Service
- Identity & Access Management (IAM)

### [Phase 5: Interview Execution Strategy](./05-interview-strategy.md)
How to actually perform in the 45-60 minute interview. The meta-skills that determine success.

- How interviewers evaluate candidates
- Interview flow and time management
- Requirement clarification techniques
- Scale estimation methods
- Deep dive selection strategy
- Handling "what if" and "why not" questions

### [Phase 6: Advanced Topics (Staff+ Level)](./06-advanced-topics.md)
Topics for Staff/Principal level candidates. Optional but differentiating.

- Multi-region Deployments
- Active-Active vs Active-Passive
- Data Locality & Compliance
- Cost Optimization at Scale
- Internal Developer Platforms
- Control Plane vs Data Plane

---

## 🎯 Study Plan by Level

### Senior (L5) - 6 Weeks
| Week | Focus |
|------|-------|
| 1 | Phase 0: Foundations |
| 2 | Phase 1: Core Building Blocks |
| 3 | Phase 2: Distributed Systems |
| 4 | Phase 4: Mandatory Case Studies (1-4) |
| 5 | Phase 4: Mandatory Case Studies (5-7) |
| 6 | Phase 5: Interview Strategy + Mock Interviews |

### Staff (L6) - 8 Weeks
| Week | Focus |
|------|-------|
| 1 | Phase 0: Foundations (deep review) |
| 2 | Phase 1: Core Building Blocks |
| 3 | Phase 2: Distributed Systems |
| 4 | Phase 3: Scalability Patterns |
| 5 | Phase 4: Mandatory Case Studies |
| 6 | Phase 4b: Advanced Case Studies (1-3) |
| 7 | Phase 6: Advanced Topics |
| 8 | Phase 5: Interview Strategy + Mock Interviews |

### Principal (L7) - 10 Weeks
| Week | Focus |
|------|-------|
| 1-2 | Phase 0-2: Review with depth on trade-offs |
| 3-4 | Phase 3: Scalability Patterns + Organizational Impact |
| 5-6 | Phase 4 & 4b: All Case Studies |
| 7-8 | Phase 6: Advanced Topics (comprehensive) |
| 9 | Cross-cutting concerns, cost, compliance |
| 10 | Phase 5: Strategy + Multiple Mock Interviews |

---

## 📊 Quick Reference Cards

### Numbers Every Engineer Should Know
```
L1 Cache Reference:                    0.5 ns
L2 Cache Reference:                    7 ns
Main Memory Reference:                 100 ns
SSD Random Read:                       150 μs
HDD Seek:                              10 ms
Network Round Trip (same datacenter):  0.5 ms
Network Round Trip (cross-region):     150 ms

Read 1 MB sequentially from memory:    250 μs
Read 1 MB sequentially from SSD:       1 ms
Read 1 MB sequentially from HDD:       20 ms
Read 1 MB sequentially from network:   10 ms
```

### Scale Estimation Shortcuts
```
1 Million requests/day    ≈ 12 requests/second
1 Billion requests/day    ≈ 12,000 requests/second
1 Billion requests/month  ≈ 400 requests/second

1 KB * 1 Million = 1 GB
1 MB * 1 Million = 1 TB
1 GB * 1 Million = 1 PB

Typical web server: 1,000-10,000 concurrent connections
Typical database: 100-1,000 connections
```

### The 5-Minute Rule of Thumb
If data is accessed more frequently than every 5 minutes, cache it in memory.
If accessed less frequently, it can stay on disk.

---

## 🚨 Common Interview Pitfalls

1. **Jumping to solutions** without clarifying requirements
2. **Over-engineering** for scale you don't need
3. **Ignoring failure modes** (what happens when X fails?)
4. **Buzzword dropping** without depth
5. **Not doing math** for capacity estimation
6. **Single point of failure** in your design
7. **Ignoring cost** (especially at Staff+ level)
8. **Not discussing trade-offs** for your choices

---

## ✅ How to Use This Guide

1. **Read actively** - Don't just read; think about how you'd explain each concept
2. **Practice articulation** - Explain concepts out loud as if to an interviewer
3. **Draw diagrams** - Sketch architectures by hand
4. **Do the math** - Practice back-of-envelope calculations
5. **Mock interviews** - Apply the knowledge in timed conditions
6. **Review failures** - Understand what went wrong in past interviews

---

*This guide is designed to feel like mentorship from a real interviewer. Each section explains not just what to know, but why it matters and how to demonstrate seniority.*
