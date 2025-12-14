# Phase 3: Scalability Patterns

This phase covers architectural patterns for building resilient, scalable systems. These patterns are critical for Staff+ level interviews where interviewers expect you to discuss system evolution, organizational boundaries, and operational excellence.

---

# Section 1: Service Architecture Evolution

## 1.1 Monolith → Modular Monolith → Microservices

### Concept Overview (What & Why)

**Monolith:**
- Single deployable unit
- Shared database
- All code in one repository
- Simple to develop and deploy initially

**Modular Monolith:**
- Single deployable unit
- Internal module boundaries (packages/namespaces)
- Modules communicate through defined interfaces
- Can be split later if needed

**Microservices:**
- Independent deployable services
- Each service owns its data
- Network communication (REST, gRPC)
- Independent technology choices

**Why This Matters:**
- Every large system starts as a monolith
- Premature microservices is a common mistake
- The evolution path matters more than the end state

### Key Design Principles

**When to Choose:**

| Stage | Architecture | Characteristics |
|-------|--------------|-----------------|
| Early startup | Monolith | <10 engineers, finding product-market fit |
| Growing startup | Modular Monolith | 10-50 engineers, clear domain boundaries |
| Scale-up | Microservices (selective) | 50+ engineers, independent team scaling |
| Enterprise | Full microservices | 100+ engineers, organizational boundaries |

**The Monolith Trap:**
- Too often: "We're building for scale, so microservices from day one"
- Reality: Microservices add complexity before they add value
- Better: Start monolith, extract services as needed

### Trade-offs & Decision Matrix

| Factor | Monolith | Modular Monolith | Microservices |
|--------|----------|------------------|---------------|
| Deployment | Simple | Simple | Complex |
| Debugging | Easy | Easy | Hard (distributed tracing) |
| Data consistency | ACID | ACID | Eventual |
| Team autonomy | Low | Medium | High |
| Operational overhead | Low | Low | High |
| Technology diversity | Single stack | Single stack | Polyglot |
| Scaling | All or nothing | All or nothing | Granular |

### Real-World Examples

**Successful Monolith:**
- Shopify ran as a monolith with 2,000+ developers
- Carefully modularized with strict module boundaries
- Only split services for specific scaling needs

**Microservices Done Right:**
- Amazon: Services aligned to teams (two-pizza rule)
- Netflix: Started monolith, migrated during cloud transition
- Uber: Thousands of services, but started with monolith

**Microservices Done Wrong:**
- Premature extraction before understanding boundaries
- Too fine-grained services (one per entity)
- Distributed monolith (services coupled, deploy together)

### Interview Perspective

**What interviewers look for:**
- Pragmatic thinking about architecture evolution
- Understanding of organizational factors
- Awareness of complexity costs

**Common traps:**
- ❌ "We need microservices for scalability" (modular monolith scales too)
- ❌ "Microservices from day one" (premature complexity)
- ❌ "One service per entity" (too granular)

**Strong signals:**
- ✅ "Start with modular monolith, extract when needed"
- ✅ "Microservices boundaries should align with team boundaries"
- ✅ "Each service owns its data; no shared databases"

**Follow-up questions:**
- "How do you decide when to extract a service?"
- "How do you handle a service calling 10 other services?"
- "What about shared libraries vs duplicate code?"

---

## 1.2 Service Decomposition Strategies

### Concept Overview (What & Why)

**Service Decomposition:** Breaking a system into services with clear boundaries.

**Bad Decomposition Signs:**
- Services that must deploy together
- Circular dependencies
- Shared mutable state
- Change in one service cascades to others

**Good Decomposition Signs:**
- Services can be developed independently
- Services have stable APIs
- Most changes are local to one service
- Teams can release independently

### Key Design Principles

**Decomposition Strategies:**

| Strategy | Description | Example |
|----------|-------------|---------|
| By Business Capability | Align to what business does | Orders, Payments, Shipping |
| By Subdomain (DDD) | Bounded contexts from Domain-Driven Design | Customer, Catalog, Fulfillment |
| By Team | One team, one or few services | Team owns the whole slice |
| By Data | Who owns this data? | User Service owns users table |

**Domain-Driven Design (DDD) Concepts:**

**Bounded Context:**
- A boundary within which a term has specific meaning
- "Customer" in Sales vs "Customer" in Shipping = different models
- Each bounded context can be a service

**Aggregate:**
- Cluster of domain objects treated as a unit
- Transaction boundary
- Good candidate for service boundary

### Trade-offs & Decision Matrix

| Decomposition | Pros | Cons |
|---------------|------|------|
| By capability | Business-aligned, intuitive | May not match data ownership |
| By subdomain | Clear boundaries, DDD benefits | Requires domain expertise |
| By team | Organizational alignment | Services may be arbitrary |
| By data | Clear ownership | May not match use cases |

**Rules of Thumb:**
- Service should be ownable by one team (5-10 people)
- Service should be deployable independently
- Service should have one reason to change
- When in doubt, keep together (can split later)

### Interview Perspective

**Strong signals:**
- ✅ "Bounded context from DDD gives natural service boundaries"
- ✅ "Services aligned with teams and business capabilities"
- ✅ "Each service owns its data, no shared databases"

**Common trap:**
- ❌ Decomposing by technical layer (API service, DB service)

---

## 1.3 Database per Service

### Concept Overview (What & Why)

**Pattern:** Each microservice owns its data and database. No direct database sharing.

```
Order Service → Order DB
User Service → User DB
Payment Service → Payment DB

Order Service needs user data?
  → Call User Service API (not User DB)
```

**Why This Matters:**
- Loose coupling (can change schema without affecting others)
- Independent deployment
- Technology freedom (SQL vs NoSQL per service)
- Clear ownership

**Challenges:**
- No cross-service transactions
- Data consistency is eventual
- Queries across services are complex
- Data duplication may be needed

### Key Design Principles

**Handling Cross-Service Data:**

| Need | Solution |
|------|----------|
| Reference data | Cache local copy, refresh periodically |
| Real-time data | API call (sync) or subscribe to events (async) |
| Reports/analytics | Data warehouse (ETL from all services) |
| Cross-service queries | API composition or CQRS read model |

**Data Ownership Questions:**
- Who creates this data?
- Who is the source of truth?
- Who needs to query it?
- How fresh does it need to be?

### Trade-offs & Decision Matrix

| Challenge | Solution | Trade-off |
|-----------|----------|-----------|
| Cross-service joins | API calls + client-side join | Latency, complexity |
| Consistency | Saga pattern | Eventual, complex |
| Reporting | Data warehouse | Staleness, extra infrastructure |
| Duplicate data | Event-driven sync | Consistency window |

### Interview Perspective

**Strong signals:**
- ✅ "No shared databases; each service owns its data"
- ✅ "Cross-service data via APIs or events, not database"
- ✅ "Data warehouse for cross-cutting analytics"

**Follow-up questions:**
- "How do you handle a report that needs data from 5 services?"
- "What if Service A needs to guarantee consistency with Service B?"

---

### Section 1 Cheat Sheet

```
SERVICE ARCHITECTURE

EVOLUTION PATH:
Monolith → Modular Monolith → Microservices (if needed)

DON'T START WITH MICROSERVICES:
• <50 engineers: Modular monolith likely sufficient
• Microservices add complexity before value
• Distributed transactions, debugging, operations

DECOMPOSITION STRATEGIES:
• By business capability (Orders, Payments)
• By subdomain (DDD bounded contexts)
• By team ownership
• By data ownership

GOOD DECOMPOSITION SIGNS:
• Independent deployment
• Stable APIs
• Changes local to one service
• Team autonomy

DATABASE PER SERVICE:
• No shared databases
• Cross-service data via API/events
• Accept eventual consistency
• Data warehouse for analytics

WHEN TO EXTRACT A SERVICE:
• Different scaling requirements
• Different team ownership
• Different release cadence
• Clear bounded context
```

---

# Section 2: Distributed Transaction Patterns

## 2.1 Saga Pattern

### Concept Overview (What & Why)

**Problem:** Microservices can't use distributed transactions (2PC) at scale.
**Solution:** Saga - a sequence of local transactions with compensating actions.

```
Order Saga:
1. Order Service: Create order (PENDING)
2. Payment Service: Charge customer
3. Inventory Service: Reserve items
4. Order Service: Confirm order (CONFIRMED)

If step 3 fails:
  Compensate step 2: Refund customer
  Compensate step 1: Cancel order
```

**Why This Matters:**
- Distributed transactions (2PC) don't scale
- Business processes span multiple services
- Need to handle partial failures

### Key Design Principles

**Two Saga Types:**

| Type | Coordination | Pros | Cons |
|------|--------------|------|------|
| Choreography | Events, no central coordinator | Loose coupling, simple | Hard to track, complex flows |
| Orchestration | Central saga orchestrator | Clear flow, easier to manage | Single point of coordination |

**Choreography Example:**
```
Order Service → publishes OrderCreated
                    ↓
Payment Service ← subscribes, charges, publishes PaymentCompleted
                    ↓
Inventory Service ← subscribes, reserves, publishes InventoryReserved
                    ↓
Order Service ← subscribes, confirms order
```

**Orchestration Example:**
```
Order Saga Orchestrator:
  1. Call Order Service: CreateOrder
  2. Call Payment Service: ChargeCustomer
  3. Call Inventory Service: ReserveItems
  4. Call Order Service: ConfirmOrder
  
  On failure: Execute compensating transactions
```

### Trade-offs & Decision Matrix

| Factor | Choreography | Orchestration |
|--------|--------------|---------------|
| Coupling | Looser | Tighter (to orchestrator) |
| Visibility | Harder to trace | Easy to see in orchestrator |
| Complexity | In events | In orchestrator |
| Testing | Harder | Easier |
| Best for | Simple flows, 2-3 steps | Complex flows, many steps |

### Real-World Examples

**E-commerce Order:**
1. Reserve inventory
2. Charge payment
3. Create shipment
4. Send confirmation

Compensations:
- Release inventory
- Refund payment
- Cancel shipment

**Travel Booking:**
1. Book flight
2. Book hotel
3. Book car

If hotel booking fails, compensate flight booking.

### Failure Scenarios & Edge Cases

| Scenario | Handling |
|----------|----------|
| Step fails | Execute compensating transactions backwards |
| Compensation fails | Retry compensation, alert if stuck |
| Saga stuck | Timeout + compensate or manual intervention |
| Idempotency | Each step must be idempotent (retries) |

### Interview Perspective

**Strong signals:**
- ✅ "Saga pattern for cross-service transactions"
- ✅ "Each step has a compensating action"
- ✅ "Orchestration for complex flows, choreography for simple"

**Follow-up questions:**
- "What if a compensation fails?"
- "How do you test a saga?"
- "How do you handle a saga that's stuck?"

---

### Section 2 Cheat Sheet

```
SAGA PATTERN

PROBLEM: No distributed transactions across services
SOLUTION: Local transactions + compensating actions

CHOREOGRAPHY:
• Event-driven
• No central coordinator
• Simple flows, loose coupling
• Harder to trace/debug

ORCHESTRATION:
• Central coordinator
• Explicit flow definition
• Complex flows, visibility
• Easier testing

COMPENSATION:
• Each step has reverse action
• Execute on failure
• Must handle compensation failures

IDEMPOTENCY:
• Every step must be idempotent
• Retries are expected
• Use unique transaction IDs

EXAMPLE (Order):
1. CreateOrder → compensation: CancelOrder
2. ChargePayment → compensation: Refund
3. ReserveInventory → compensation: Release
4. Ship → compensation: CancelShipment
```

---

# Section 3: Resilience Patterns

## 3.1 Circuit Breaker

### Concept Overview (What & Why)

**Problem:** When a downstream service is failing, callers keep trying, wasting resources and cascading failures.

**Solution:** Circuit breaker - stop calling a failing service temporarily.

```
States:
CLOSED → Normal operation, calls go through
OPEN → Service failing, calls fail fast
HALF-OPEN → Testing if service recovered

CLOSED: Calls succeed
  ↓ (failure threshold exceeded)
OPEN: Calls fail immediately (don't try)
  ↓ (timeout expires)
HALF-OPEN: Allow one test call
  ↓ (success)
CLOSED (or back to OPEN if test fails)
```

### Key Design Principles

**Configuration:**
- **Failure threshold:** How many failures trigger open state (e.g., 5 failures in 10 seconds)
- **Open timeout:** How long to wait before testing (e.g., 30 seconds)
- **Half-open success threshold:** Successes needed to close (e.g., 3)

**What Counts as Failure:**
- HTTP 5xx responses
- Timeouts
- Connection failures
- NOT 4xx (client errors)

### Real-World Examples

**Netflix Hystrix (now Resilience4j):**
```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("paymentService");

Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> paymentService.charge(amount));

Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> "Fallback response");
```

**Fallback Strategies:**
- Return cached data
- Return default value
- Return degraded experience
- Fail gracefully with error message

### Interview Perspective

**Strong signals:**
- ✅ "Circuit breaker to prevent cascading failures"
- ✅ "Fail fast when downstream is unhealthy"
- ✅ "Fallback for graceful degradation"

---

## 3.2 Retries with Exponential Backoff

### Concept Overview (What & Why)

**Problem:** Transient failures happen; immediate retries may not help.

**Solution:** Retry with increasing delays.

```
Attempt 1: Immediate
Attempt 2: Wait 1 second
Attempt 3: Wait 2 seconds
Attempt 4: Wait 4 seconds
Attempt 5: Wait 8 seconds
Give up after max attempts
```

### Key Design Principles

**Exponential Backoff Formula:**
```
delay = base_delay * (2 ^ attempt) + random_jitter
```

**Jitter (Critical):**
- Without jitter: All clients retry at same time → thundering herd
- With jitter: Retries spread out over time

```
Attempt 1: 1s + random(0, 0.5s)
Attempt 2: 2s + random(0, 1s)
Attempt 3: 4s + random(0, 2s)
```

**What to Retry:**
- Transient errors (503, timeouts)
- NOT permanent errors (400, 404, 401)
- Idempotent operations only (or with idempotency key)

### Trade-offs & Decision Matrix

| Config | Impact |
|--------|--------|
| More retries | Higher success rate, longer latency |
| Shorter delays | Faster recovery, more load on failing service |
| Longer delays | Less load, slower recovery |
| No jitter | Thundering herd |

### Interview Perspective

**Strong signals:**
- ✅ "Exponential backoff with jitter"
- ✅ "Only retry transient errors"
- ✅ "Ensure idempotency for retries"

---

## 3.3 Bulkhead

### Concept Overview (What & Why)

**Problem:** One slow dependency uses up all resources, affecting all requests.

**Solution:** Isolate resources so one failure doesn't affect others.

**Analogy:** Ship compartments - if one floods, others stay dry.

```
Without Bulkhead:
Thread Pool: [A] [A] [A] [B] [B] [B] [C] [C] [C] [C]
If C is slow → all threads blocked → A and B fail too

With Bulkhead:
Pool A: [A] [A] [A]
Pool B: [B] [B] [B]
Pool C: [C] [C] [C] [C]
If C is slow → only Pool C affected
```

### Key Design Principles

**Bulkhead Types:**

| Type | Mechanism | Use Case |
|------|-----------|----------|
| Thread pool isolation | Separate thread pool per dependency | JVM-based services |
| Semaphore isolation | Limit concurrent calls | Lighter weight |
| Connection pool | Separate DB connection pools | Database access |

**Sizing Considerations:**
- Too small: Unnecessary throttling
- Too large: No protection
- Based on: Expected concurrency, dependency latency

### Interview Perspective

**Strong signals:**
- ✅ "Bulkhead to isolate failures"
- ✅ "Separate thread pools for critical dependencies"
- ✅ "Prevents one slow service from affecting others"

---

## 3.4 Timeouts

### Concept Overview (What & Why)

**Problem:** Without timeouts, slow services can hold resources indefinitely.

**Solution:** Set timeouts for all external calls.

**Timeout Types:**
- **Connection timeout:** Time to establish connection (typically 1-5 seconds)
- **Read timeout:** Time to receive response (varies by operation)
- **Total timeout:** End-to-end for the request

### Key Design Principles

**Timeout Guidelines:**

| Operation | Typical Timeout |
|-----------|----------------|
| Database query (simple) | 1-5 seconds |
| Database query (complex) | 30 seconds |
| Cache lookup | 100-500 ms |
| Internal service call | 1-5 seconds |
| External API call | 5-30 seconds |
| File upload | Minutes (depends on size) |

**Timeout Propagation:**
```
Client → API Gateway (5s) → Service A (3s) → Service B (1s)

Each layer should have shorter timeout than its caller.
Otherwise: Caller times out, but downstream continues processing.
```

**Deadline Propagation (gRPC):**
```
Client sets deadline: 5 seconds from now
Propagated to all downstream calls
Each service checks remaining time
```

### Interview Perspective

**Strong signals:**
- ✅ "Timeouts on all external calls"
- ✅ "Timeout budget: Each hop has less than caller"
- ✅ "Combine with circuit breaker for resilience"

---

### Section 3 Cheat Sheet

```
RESILIENCE PATTERNS

CIRCUIT BREAKER:
• States: Closed → Open → Half-Open
• Fail fast when service is down
• Fallback for graceful degradation

RETRY WITH BACKOFF:
delay = base * 2^attempt + jitter
• Jitter prevents thundering herd
• Only retry transient errors
• Require idempotency

BULKHEAD:
• Isolate resources per dependency
• Thread pools, semaphores, connection pools
• One failure doesn't affect others

TIMEOUTS:
• Set on ALL external calls
• Shorter than caller's timeout
• Connection timeout + read timeout

COMBINING PATTERNS:
Request → Timeout → Retry (with backoff) → Circuit Breaker → Fallback

TYPICAL CONFIGURATION:
• Timeout: 3s
• Retries: 3 with exponential backoff
• Circuit: Open after 5 failures
• Circuit: Half-open after 30s
```

---

# Section 4: Observability

## 4.1 The Three Pillars: Metrics, Logging, Tracing

### Concept Overview (What & Why)

**Observability:** Ability to understand system behavior from external outputs.

| Pillar | What | For |
|--------|------|-----|
| Metrics | Numerical measurements over time | Trends, alerts, dashboards |
| Logs | Discrete events with details | Debugging, auditing |
| Traces | Request flow across services | Understanding distributed calls |

**Why This Matters:**
- Microservices are hard to debug without observability
- Production issues need fast diagnosis
- Capacity planning requires metrics

### Key Design Principles

**Metrics (What to Measure):**

| Metric Type | Examples |
|-------------|----------|
| Counters | Request count, error count |
| Gauges | Current connections, queue depth |
| Histograms | Latency distribution, request sizes |
| Summaries | Similar to histograms (different trade-offs) |

**Logging Best Practices:**
- Structured logs (JSON, not plain text)
- Correlation ID across services
- Appropriate log levels (ERROR, WARN, INFO, DEBUG)
- Avoid PII in logs
- Centralized log aggregation (ELK, Splunk)

**Distributed Tracing:**
```
Request enters system:
  TraceID: abc123
  
Service A (SpanID: 1):
  → Calls Service B (SpanID: 2, ParentSpanID: 1)
      → Calls Database (SpanID: 3, ParentSpanID: 2)
  → Calls Service C (SpanID: 4, ParentSpanID: 1)
```

Tools: Jaeger, Zipkin, AWS X-Ray, Datadog APM

---

## 4.2 RED and USE Metrics

### Concept Overview (What & Why)

**RED Method (For Services):**
- **R**ate: Requests per second
- **E**rrors: Failed requests per second
- **D**uration: Latency distribution (p50, p95, p99)

**USE Method (For Resources):**
- **U**tilization: % of resource used (CPU %, memory %)
- **S**aturation: Queue depth (work waiting)
- **E**rrors: Error count

### Key Design Principles

**RED Metrics Example:**
```
Service: Order API
Rate: 1000 req/s
Errors: 5 req/s (0.5%)
Duration: p50=50ms, p95=200ms, p99=500ms
```

**USE Metrics Example:**
```
Resource: Database
Utilization: CPU 70%, Memory 85%
Saturation: Connection queue depth = 50
Errors: Connection timeout = 10/min
```

**When to Use:**
- RED: Request-driven services (APIs)
- USE: Infrastructure resources (CPU, memory, DB, queues)

---

## 4.3 SLIs, SLOs, and SLAs

### Concept Overview (What & Why)

| Term | Definition | Example |
|------|------------|---------|
| SLI (Indicator) | Quantitative measure of service | p99 latency, error rate |
| SLO (Objective) | Target value for an SLI | p99 latency < 200ms |
| SLA (Agreement) | Contract with consequences | 99.9% uptime or refund |

**Relationship:**
```
SLI: What you measure
SLO: What you aim for (internal target, tighter)
SLA: What you promise (external contract, looser)
```

### Key Design Principles

**Good SLIs:**
- User-centric (what users experience)
- Measurable and actionable
- Reflect actual quality

**Common SLIs:**
| Category | SLI |
|----------|-----|
| Availability | % of successful requests |
| Latency | p95 or p99 response time |
| Throughput | Requests per second at full load |
| Error rate | % of failed requests |
| Freshness | Age of data (for async systems) |

**Error Budget:**
```
SLO: 99.9% availability
Budget: 0.1% downtime per month = 43 minutes

If you've used 30 minutes this month:
  → 13 minutes remaining
  → Slow down risky changes
```

### Interview Perspective

**Strong signals:**
- ✅ "SLO of p99 < 200ms, SLA at p99 < 500ms (buffer)"
- ✅ "Error budget approach to balance velocity and reliability"
- ✅ "Alert on SLO burn rate, not just thresholds"

---

### Section 4 Cheat Sheet

```
OBSERVABILITY

THREE PILLARS:
Metrics: Numbers over time (Prometheus)
Logs: Events with details (ELK, Splunk)
Traces: Request flow (Jaeger, Zipkin)

RED (Services):
Rate: Requests per second
Errors: Error rate
Duration: Latency (p50, p95, p99)

USE (Resources):
Utilization: % used
Saturation: Queue depth
Errors: Error count

SLI/SLO/SLA:
SLI: What you measure
SLO: Internal target (stricter)
SLA: External promise (with consequences)

ERROR BUDGET:
99.9% SLO = 43 min downtime/month allowed
Track budget consumption
Slow down if burning too fast

LOGGING BEST PRACTICES:
• Structured (JSON)
• Correlation IDs
• No PII
• Appropriate levels

ALERTING:
• Alert on symptoms, not causes
• SLO burn rate alerts
• Avoid alert fatigue
```

---

# Phase 3 Summary: Building Resilient Systems

These patterns are what separate production-ready systems from prototypes. In Staff+ interviews, you're expected to discuss these naturally.

**Key Takeaways:**

1. **Architecture Evolution:** Start simple, extract services when needed
2. **Service Boundaries:** Align with teams and domains, not technical layers
3. **Distributed Transactions:** Saga pattern with compensating actions
4. **Resilience:** Circuit breakers, retries, bulkheads, timeouts
5. **Observability:** Metrics, logs, traces, and SLOs

**Interviewer Expectations:**

| Level | Expectation |
|-------|-------------|
| Senior (L5) | Know patterns, apply when prompted |
| Staff (L6) | Proactively bring up resilience patterns |
| Principal (L7) | Discuss organizational impact, build vs buy |

**Questions That Probe Depth:**
- "How would you handle partial failures?"
- "What's your alerting strategy?"
- "How do you know if the system is healthy?"
- "What happens when Service X goes down?"

**Red Flags:**
- No mention of observability
- No failure handling discussion
- "We'll use microservices" without justification
- No compensation strategy for distributed operations

**Green Flags:**
- "Circuit breaker to prevent cascade"
- "Saga with compensating transactions"
- "SLO-based alerting with error budget"
- "Start monolith, extract when team boundaries form"
