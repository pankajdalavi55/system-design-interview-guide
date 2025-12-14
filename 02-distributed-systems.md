# Phase 2: Distributed Systems Core

This phase covers the concepts that separate senior engineers from mid-level. Distributed systems introduce complexity that requires careful reasoning about failures, consistency, and coordination. These topics are where most candidates struggle, and where interviewers probe deeply.

---

# Section 1: Data Partitioning (Sharding)

## 1.1 Horizontal Sharding

### Concept Overview (What & Why)

**Horizontal Sharding (Partitioning):** Distributing rows of a table across multiple databases based on a shard key.

```
Users Table (10 billion rows)
    ↓ Shard by user_id
┌─────────────┬─────────────┬─────────────┐
│  Shard 1    │   Shard 2   │   Shard 3   │
│ users 1-3B  │ users 3-6B  │ users 6-10B │
└─────────────┴─────────────┴─────────────┘
```

**Why this matters:**
- Single database has limits (storage, connections, write throughput)
- Sharding provides horizontal scalability
- But adds significant complexity (cross-shard queries, transactions)

**When interviewers expect this:**
- Any design with billions of records
- Write-heavy systems exceeding single DB capacity
- When discussing database scaling strategy

### Key Design Principles

**Shard Key Selection (Critical):**

| Good Shard Key | Why |
|----------------|-----|
| user_id | Most queries are per-user, even distribution |
| tenant_id | Multi-tenant SaaS, complete isolation |
| order_id (hashed) | Even distribution for orders |

| Bad Shard Key | Why |
|---------------|-----|
| created_date | Hot partition (today's shard overwhelmed) |
| country | Uneven (US shard much larger) |
| status | Low cardinality (few distinct values) |

**Sharding Strategies:**

| Strategy | Method | Pros | Cons |
|----------|--------|------|------|
| Range | user_id 1-1M → shard 1 | Range queries easy | Hot spots, uneven |
| Hash | hash(user_id) % N | Even distribution | No range queries |
| Directory | Lookup table | Flexible | Lookup is SPOF |
| Geo | Region → shard | Data locality | Cross-region queries hard |

### Trade-offs & Decision Matrix

| Challenge | Impact | Mitigation |
|-----------|--------|------------|
| Cross-shard queries | Slow, complex | Denormalize, co-locate related data |
| Cross-shard joins | Nearly impossible | Application-level joins |
| Cross-shard transactions | Very difficult | Saga pattern, eventual consistency |
| Rebalancing | Operational nightmare | Consistent hashing, virtual shards |
| Schema changes | Must apply to all shards | Automation, careful planning |

### Real-World Examples

**Instagram:**
- Shard by user_id
- Photos stored with owner
- Feed assembly: Query each followee's shard, merge

**Discord:**
- Shard by guild_id (server)
- All messages for a guild on same shard
- Enables efficient channel queries

**Slack:**
- Shard by workspace_id
- Each workspace is independent
- Cross-workspace features are limited

### Failure Scenarios & Edge Cases

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Shard unavailable | Data for those users inaccessible | Replication within shard |
| Hot shard | One shard overwhelmed | Better key, split hot shard |
| Shard key not in query | Must query all shards | Ensure key in common queries |
| Resharding needed | Major migration project | Virtual shards, consistent hashing |

### Interview Perspective

**What interviewers look for:**
- Shard key selection reasoning
- Understanding of trade-offs
- Awareness of operational complexity

**Common traps:**
- ❌ "Shard by timestamp" (today's shard is always hot)
- ❌ "We'll just join across shards" (nearly impossible)
- ❌ Ignoring the resharding problem

**Strong signals:**
- ✅ "Shard by user_id, co-locate user's related data"
- ✅ "Cross-shard queries are expensive; design to minimize"
- ✅ "Use consistent hashing for easier resharding"

**Follow-up questions:**
- "What if one user has 10x more data than average?"
- "How would you handle a query that needs all users?"
- "How do you add a new shard?"

---

## 1.2 Consistent Hashing

### Concept Overview (What & Why)

**Problem with Simple Hashing:**
```
server = hash(key) % N
```
If N changes (add/remove server), almost all keys remap → cache invalidation storm.

**Consistent Hashing Solution:**
- Arrange servers on a conceptual ring (0 to 2^32)
- Hash key to position on ring
- Route to first server clockwise from that position
- Adding/removing server only affects adjacent keys

```
        0°
    ┌───────┐
    │       │
 S1 ●       ● S2
    │       │
    └───────┘
       180°
       
Key K hashes to 45° → routes to S2 (first clockwise)
```

**Why this matters:**
- Adding server: Only keys between new server and predecessor move
- Removing server: Only those keys move to successor
- Result: Minimal disruption (1/N keys move instead of all)

### Key Design Principles

**Virtual Nodes (VNodes):**
- Problem: Few physical servers → uneven distribution
- Solution: Each server has multiple positions on ring
- Example: 3 servers × 100 virtual nodes = 300 points on ring
- Benefits: Even distribution, gradual failover

**Implementation:**
```
Ring positions: [S1-v1: 10, S1-v2: 85, S2-v1: 30, S2-v2: 150, ...]
Key "user:123" hashes to 45
Binary search for first position ≥ 45 → S1-v2 at 85
Route to S1
```

### Trade-offs & Decision Matrix

| Approach | Server Add/Remove Impact | Distribution | Complexity |
|----------|-------------------------|--------------|------------|
| Modulo hashing | ~100% keys move | Good | Low |
| Consistent hashing | ~1/N keys move | Good with vnodes | Medium |
| Jump consistent hash | ~1/N keys move | Excellent | Low |

### Real-World Examples

- **Cassandra:** Consistent hashing for partition distribution
- **DynamoDB:** Similar approach for partitioning
- **Memcached clients:** Key distribution across servers
- **CDN:** Request routing to edge servers

### Interview Perspective

**What interviewers look for:**
- Understanding of the problem it solves
- Knowledge of virtual nodes and why they're needed
- Practical application scenarios

**Strong signals:**
- ✅ "Consistent hashing minimizes key redistribution"
- ✅ "Virtual nodes for even distribution"
- ✅ "Used in cache layer for server routing"

**Follow-up questions:**
- "How many virtual nodes per server?"
- "How do you handle a hot key?"
- "What if a server goes down?"

---

## 1.3 Range-based vs Hash-based Partitioning

### Concept Overview

| Approach | Method | Key Ordering |
|----------|--------|--------------|
| Range-based | key 1-1M → shard 1 | Preserved |
| Hash-based | hash(key) % N → shard | Lost |

### Key Design Principles

**Range-based Partitioning:**
```
Shard 1: A-M
Shard 2: N-Z

Query: WHERE name LIKE 'John%'
→ Routes to Shard 1 only (efficient)
```

**Pros:**
- Range queries efficient (single shard)
- Sequential scans possible

**Cons:**
- Hot spots (popular ranges overloaded)
- Manual rebalancing often needed

**Hash-based Partitioning:**
```
Shard = hash(user_id) % 4

Query: WHERE user_id = 123
→ hash(123) % 4 = 3 → Shard 3 only (efficient)

Query: WHERE user_id BETWEEN 100 AND 200
→ Must query ALL shards (inefficient)
```

**Pros:**
- Even distribution
- No hot spots (assuming good hash)

**Cons:**
- No range queries
- Related data scattered

### Trade-offs & Decision Matrix

| Access Pattern | Recommended |
|----------------|-------------|
| Point queries only | Hash |
| Range queries needed | Range |
| Time-series (recent data hot) | Range with rotation |
| Even write distribution critical | Hash |
| Need to keep related data together | Range or composite key |

### Real-World Examples

**Range-based:**
- Time-series databases (partition by time range)
- Alphabetical sharding (by first letter)
- HBase row key design

**Hash-based:**
- Cassandra (hash of partition key)
- DynamoDB (hash of partition key)
- Redis Cluster

**Hybrid (Common):**
```
DynamoDB: hash(partition_key) + range(sort_key)
Cassandra: hash(partition_key) + clustering_columns
```

### Interview Perspective

**Strong signals:**
- ✅ "Hash for even distribution, range for range queries"
- ✅ "Hybrid approach: hash on partition key, range on sort key"
- ✅ "Time-series: range by time, but handle hot partition for 'now'"

---

### Section 1 Cheat Sheet

```
DATA PARTITIONING

SHARD KEY SELECTION:
✅ High cardinality
✅ Even distribution
✅ Matches query patterns
❌ Timestamps (hot partition)
❌ Low cardinality (country, status)

STRATEGIES:
Range: Key ordering preserved, hot spots possible
Hash: Even distribution, no range queries
Directory: Flexible, lookup overhead
Geo: Data locality, cross-region complex

CONSISTENT HASHING:
• Ring of hash values
• Key → first server clockwise
• Add/remove affects only neighbors
• Virtual nodes for even distribution

CROSS-SHARD OPERATIONS:
• Queries: Query all, merge (slow)
• Joins: Application level (very slow)
• Transactions: Saga pattern (complex)

RESHARDING:
• Consistent hashing reduces impact
• Virtual shards allow gradual migration
• Plan for 2x growth at minimum
```

---

# Section 2: Replication

## 2.1 Leader-Follower (Primary-Replica) Replication

### Concept Overview (What & Why)

```
Writes → [Leader] ──replication──→ [Follower 1]
                                 → [Follower 2]
                                 → [Follower 3]
              ↑
          Reads (optional)      Reads ←────────┘
```

**Why this matters:**
- Read scalability: Distribute reads across replicas
- High availability: Failover to replica if leader fails
- Data durability: Multiple copies of data

**When interviewers expect this:**
- Any database discussion at scale
- High availability requirements
- Read-heavy workloads

### Key Design Principles

**Replication Methods:**

| Method | Description | Durability | Latency |
|--------|-------------|------------|---------|
| Synchronous | Wait for replica ack before commit | Highest | Higher |
| Asynchronous | Commit immediately, replicate after | Lower | Lower |
| Semi-synchronous | Wait for at least one replica | Medium | Medium |

**Read Concerns:**
- Read from leader: Always consistent, higher load
- Read from replica: May be stale, distributes load

### Trade-offs & Decision Matrix

| Requirement | Choose |
|-------------|--------|
| No data loss | Synchronous replication |
| Low latency writes | Asynchronous |
| Balance | Semi-synchronous (1 sync, rest async) |
| Read scalability | Read from replicas (accept staleness) |
| Strong consistency | Read from leader only |

### Real-World Examples

**PostgreSQL:**
- Streaming replication (async by default)
- Synchronous option available
- pg_stat_replication to monitor lag

**MySQL:**
- Master-slave (now called source-replica)
- Semi-synchronous replication option
- Group replication for multi-leader

**Redis:**
- Async replication by default
- WAIT command for sync writes
- Sentinel for failover

### Failure Scenarios & Edge Cases

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Leader fails | Writes unavailable until failover | Automatic failover (Sentinel, Patroni) |
| Replication lag | Reads return stale data | Monitor lag, route to leader if critical |
| Split brain | Two leaders, data divergence | Quorum-based leader election |
| Network partition | Replicas fall behind | Detect and handle on recovery |

**Failover Process:**
1. Detect leader failure (heartbeat timeout)
2. Elect new leader (among replicas)
3. Redirect writes to new leader
4. Old leader rejoins as follower (after recovery)

### Interview Perspective

**Strong signals:**
- ✅ "Sync replication for durability, async for performance"
- ✅ "Read from replicas for scale, accept staleness"
- ✅ "Automatic failover with quorum to prevent split brain"

**Follow-up questions:**
- "What if the leader fails mid-transaction?"
- "How do you handle replication lag?"
- "How do you prevent split brain?"

---

## 2.2 Multi-Leader Replication

### Concept Overview (What & Why)

```
[Leader 1 (US)] ←──────→ [Leader 2 (EU)]
     ↓                        ↓
[Followers]              [Followers]
```

**Why Multi-Leader:**
- Geographic distribution (low latency per region)
- Datacenter redundancy
- Offline operation (mobile apps)

**Challenges:**
- Conflict resolution (same data modified in both)
- Increased complexity
- Eventual consistency only

### Key Design Principles

**Conflict Resolution Strategies:**

| Strategy | Description | Data Loss Risk |
|----------|-------------|----------------|
| Last-Write-Wins (LWW) | Latest timestamp wins | Higher (silent overwrites) |
| First-Write-Wins | Earliest timestamp wins | Higher (silent ignores) |
| Merge | Custom logic to combine | Lower (complex) |
| Conflict-free (CRDTs) | Data structures that auto-merge | None (limited use cases) |
| Manual | Flag conflict for user resolution | None (poor UX) |

### Trade-offs & Decision Matrix

| Scenario | Choose |
|----------|--------|
| Offices editing same document | Merge or CRDTs (Google Docs) |
| User profile updates | LWW usually fine |
| Financial data | Avoid multi-leader or use CRDTs for counters |
| Geographic distribution | Multi-leader with careful conflict handling |

### Real-World Examples

**CouchDB:** Multi-leader with conflict detection
**Cassandra:** Multi-datacenter with LWW
**Git:** Distributed with manual merge
**Google Docs:** CRDTs for real-time collaboration

### Interview Perspective

**Strong signals:**
- ✅ "Multi-leader for geo-distribution, but conflicts are hard"
- ✅ "LWW for simple cases, CRDTs for counters"
- ✅ "Prefer single-leader unless latency requires multi"

**Common trap:**
- ❌ "We'll use multi-leader" without addressing conflicts

---

## 2.3 Read Replicas and Replication Lag Handling

### Concept Overview (What & Why)

**Read Replicas:** Followers that serve read traffic, reducing load on leader.

**Replication Lag:** Delay between write on leader and visibility on replica.
- Async replication: Lag can be seconds to minutes
- Sync replication: No lag, but higher write latency

### Key Design Principles

**Handling Replication Lag:**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Read-your-writes | After write, read from leader (or wait) | User sees own updates |
| Monotonic reads | Stick to same replica within session | Avoid going "back in time" |
| Causal consistency | If read A before B, always see B after A | Related data stays consistent |
| Bounded staleness | Read from replica only if lag < threshold | Time-sensitive data |

**Implementation - Read Your Writes:**
```
After POST /comment:
  Set cookie: last_write_time = now()

On GET /comments:
  If last_write_time > (now - 5s):
    Route to leader
  Else:
    Route to replica
```

### Trade-offs & Decision Matrix

| Requirement | Approach |
|-------------|----------|
| User sees own writes | Read-your-writes pattern |
| Consistent within session | Monotonic reads (sticky sessions) |
| Fresh data critical | Read from leader |
| Can tolerate 5s staleness | Read from replica |
| Monitor freshness | Expose replication lag metrics |

### Real-World Examples

**Amazon RDS:**
- Read replicas in same or different regions
- Replica lag metric available
- Application chooses read endpoint

**Facebook:**
- Writes go to primary (US)
- Reads from regional replicas
- Cache layer masks most lag

### Interview Perspective

**Strong signals:**
- ✅ "Read-your-writes for user experience"
- ✅ "Monitor replication lag, route to leader if critical"
- ✅ "Accept staleness for most reads, not for own writes"

---

### Section 2 Cheat Sheet

```
REPLICATION

LEADER-FOLLOWER:
• Writes to leader, replicated to followers
• Sync: Higher durability, higher latency
• Async: Lower latency, potential data loss

MULTI-LEADER:
• Multiple regions accept writes
• Conflict resolution required
• Use for: Geo-distribution, offline support
• Avoid for: Financial, complex transactions

REPLICATION LAG:
• Async replication = lag is expected
• Read-your-writes: Route to leader after write
• Monotonic reads: Same replica per session
• Monitor lag: Alert if > threshold

FAILOVER:
1. Detect failure (heartbeat)
2. Elect new leader (quorum)
3. Update routing
4. Old leader rejoins as follower

SPLIT BRAIN PREVENTION:
• Quorum-based election (majority)
• Fencing: Shut down old leader
• STONITH: "Shoot The Other Node In The Head"
```

---

# Section 3: Consistency and Consensus

## 3.1 CAP Theorem (Practical View)

### Concept Overview (What & Why)

**CAP States:** In a distributed system experiencing a network partition, you can only have two of:
- **Consistency:** Every read returns the most recent write
- **Availability:** Every request receives a response
- **Partition Tolerance:** System operates despite network failures

**The Reality:**
- Network partitions WILL happen
- So you're really choosing between C and A during partitions
- Most of the time (no partition), you can have both

**PACELC Extension (More Accurate):**
```
if (Partition)
  choose Availability or Consistency
else
  choose Latency or Consistency
```

### Key Design Principles

**CP Systems (Choose Consistency):**
- During partition: Refuse requests if uncertain
- Examples: ZooKeeper, etcd, HBase, MongoDB (default)
- Use for: Financial transactions, leader election, inventory

**AP Systems (Choose Availability):**
- During partition: Serve potentially stale data
- Examples: Cassandra, DynamoDB (default), CouchDB
- Use for: Social feeds, caching, eventually-consistent data

### Trade-offs & Decision Matrix

| Data Type | Choose | Reasoning |
|-----------|--------|-----------|
| Bank balance | CP | Wrong balance = liability |
| User session | AP | Stale session > no session |
| Product catalog | AP | Stale product > no product |
| Inventory count | CP | Overselling = operational disaster |
| Like counts | AP | Approximate is fine |
| Leader election | CP | Must have exactly one leader |

### Interview Perspective

**What interviewers look for:**
- Understanding CAP applies during partitions
- Practical trade-off decisions
- Not claiming "CA system" exists

**Strong signals:**
- ✅ "CAP is about partition scenarios, not normal operation"
- ✅ "For payments: CP. For feed: AP with eventual consistency"
- ✅ "We'll design for partition tolerance, choose C or A per use case"

---

## 3.2 Quorum Reads/Writes

### Concept Overview (What & Why)

**Quorum:** Minimum number of nodes that must agree for an operation to succeed.

```
N = Total replicas
W = Write quorum (nodes that must ack write)
R = Read quorum (nodes that must respond to read)

Strong Consistency: W + R > N
```

**Examples:**
- N=3, W=2, R=2: Strong consistency (2+2=4 > 3)
- N=3, W=1, R=1: Eventual consistency (1+1=2 < 3)
- N=3, W=3, R=1: Always read latest (all nodes have latest write)

### Key Design Principles

**Why W + R > N Works:**
```
N = 3 replicas
W = 2 (write to at least 2)
R = 2 (read from at least 2)

Write to nodes A, B (quorum satisfied)
Read from nodes B, C (quorum satisfied)
Node B is in both sets → guaranteed to see latest write
```

**Common Configurations:**

| Config | W | R | Consistency | Availability |
|--------|---|---|-------------|--------------|
| Strong | Majority | Majority | Strong | Medium |
| Fast writes | 1 | N | Eventual | High write |
| Fast reads | N | 1 | Eventually strong | High read |

### Trade-offs & Decision Matrix

| Requirement | Configuration |
|-------------|---------------|
| Strong consistency | W + R > N (e.g., W=2, R=2, N=3) |
| Fast writes | W=1, R=N (risk: lost writes on failure) |
| Fast reads | W=N, R=1 (slow writes) |
| Balanced | W=R=Majority |

### Real-World Examples

**Cassandra:**
```
INSERT INTO users (id, name) VALUES (1, 'John')
  USING CONSISTENCY QUORUM;

SELECT * FROM users WHERE id = 1
  USING CONSISTENCY QUORUM;
```

**DynamoDB:**
```
Strong consistency read: 2x cost
Eventually consistent read: Cheaper, may be stale
```

### Interview Perspective

**Strong signals:**
- ✅ "W + R > N for strong consistency"
- ✅ "Quorum allows tunable consistency"
- ✅ "Trade-off: Higher quorum = stronger consistency, lower availability"

---

## 3.3 Consensus: Paxos / Raft (Conceptual)

### Concept Overview (What & Why)

**Problem:** How do distributed nodes agree on a value when nodes can fail and messages can be lost?

**Consensus Use Cases:**
- Leader election
- Distributed locking
- State machine replication
- Configuration management

**Why This Matters:**
- You won't implement Paxos/Raft
- But you should understand why ZooKeeper/etcd exist
- Consensus is expensive; use it sparingly

### Key Design Principles

**Raft (Easier to Understand):**

1. **Leader Election:**
   - Nodes start as followers
   - If no heartbeat from leader, become candidate
   - Request votes from other nodes
   - Majority votes → become leader

2. **Log Replication:**
   - Leader receives client requests
   - Appends to log, replicates to followers
   - Once majority acknowledges, entry is committed

3. **Safety:**
   - Only one leader per term
   - Leaders never overwrite committed entries

**Key Properties:**
- Tolerates (N-1)/2 failures (3 nodes tolerates 1, 5 tolerates 2)
- Requires majority (quorum) for progress
- Provides linearizable consistency

### Trade-offs & Decision Matrix

| Need | Use | Avoid |
|------|-----|-------|
| Leader election | ZooKeeper, etcd | Rolling your own |
| Distributed lock | ZooKeeper, etcd, Redis (simpler) | Database locks at scale |
| Config management | etcd, Consul | Polling files |
| State machine replication | Raft-based system | Ad-hoc replication |

### Real-World Examples

**ZooKeeper (ZAB - Paxos variant):**
- Kafka leader election
- HBase coordination
- Hadoop high availability

**etcd (Raft):**
- Kubernetes control plane
- Service discovery

**Consul (Raft):**
- Service mesh configuration
- Distributed locking

### Interview Perspective

**What interviewers expect:**
- High-level understanding (not implementation)
- Know when to use coordination services
- Understand it's expensive (use sparingly)

**Strong signals:**
- ✅ "Use ZooKeeper/etcd for leader election, not custom"
- ✅ "Consensus is expensive; only for coordination"
- ✅ "Raft requires majority; 3 nodes tolerates 1 failure"

**You DON'T need to:**
- Implement Paxos/Raft from scratch
- Explain the exact protocol steps
- Know the formal proofs

---

### Section 3 Cheat Sheet

```
CONSISTENCY & CONSENSUS

CAP THEOREM (PRACTICAL):
During partition, choose:
• CP: Refuse uncertain requests (banks)
• AP: Serve potentially stale data (social)

PACELC (Better Model):
if Partition → A or C
else → Latency or Consistency

QUORUM:
N = replicas, W = write ack, R = read from
Strong consistency: W + R > N
Common: W=R=majority (e.g., 2 of 3)

CONSENSUS (Paxos/Raft):
• Distributed agreement
• Leader election, locking
• Tolerates (N-1)/2 failures
• Use: ZooKeeper, etcd (don't build your own)

WHEN TO USE CONSENSUS:
• Leader election
• Distributed locking
• Critical coordination
• NOT for data storage (use databases)

RULE OF THUMB:
• Most data: Eventual consistency is fine
• Critical operations: Strong consistency
• Coordination: Use existing consensus systems
```

---

# Section 4: Message Queues & Event-Driven Architecture

## 4.1 Kafka vs RabbitMQ vs SQS

### Concept Overview (What & Why)

| System | Model | Ordering | Retention | Best For |
|--------|-------|----------|-----------|----------|
| Kafka | Log-based, consumer groups | Per partition | Configurable (days/forever) | High throughput, replay |
| RabbitMQ | Traditional queue | Per queue | Until consumed | Complex routing, low latency |
| SQS | Managed queue | Best-effort (FIFO option) | Up to 14 days | AWS native, simple |

### Key Design Principles

**Kafka:**
```
Producer → Topic (multiple partitions)
                     ↓
           Consumer Group
           ├── Consumer 1 (partition 0, 1)
           └── Consumer 2 (partition 2, 3)
```
- Messages persisted (replay possible)
- Ordering guaranteed per partition
- Consumer tracks offset (can replay)
- Best for: Event sourcing, streaming, audit logs

**RabbitMQ:**
```
Producer → Exchange → Queue → Consumer
                   ↘ Queue → Consumer
```
- Messages deleted after consumption
- Complex routing (fanout, topic, headers)
- Acknowledgment-based
- Best for: Task queues, RPC, complex routing

**SQS:**
```
Producer → Queue → Consumer
         (Standard or FIFO)
```
- Fully managed, scales automatically
- Standard: At-least-once, best-effort ordering
- FIFO: Exactly-once, strict ordering (lower throughput)
- Best for: AWS workloads, simple queue needs

### Trade-offs & Decision Matrix

| Requirement | Recommendation |
|-------------|----------------|
| High throughput (100K+ msg/s) | Kafka |
| Message replay needed | Kafka |
| Complex routing logic | RabbitMQ |
| Low latency per message | RabbitMQ |
| AWS native, managed | SQS |
| Strict ordering | Kafka (per partition) or SQS FIFO |
| Simple task queue | SQS or RabbitMQ |

### Real-World Examples

**Kafka:**
- LinkedIn (original creators): Activity streams
- Uber: Trip events
- Netflix: Data pipeline

**RabbitMQ:**
- Task distribution (Celery workers)
- Microservices communication
- Email/notification sending

**SQS:**
- AWS Lambda triggers
- Decoupling AWS services
- Background job processing

### Interview Perspective

**Strong signals:**
- ✅ "Kafka for event streaming and replay; RabbitMQ for task queues"
- ✅ "Kafka retains messages; RabbitMQ deletes after consumption"
- ✅ "SQS FIFO for strict ordering, standard for high throughput"

---

## 4.2 At-Least-Once vs Exactly-Once Delivery

### Concept Overview (What & Why)

| Guarantee | Description | Implementation |
|-----------|-------------|----------------|
| At-most-once | Message might be lost | Fire and forget |
| At-least-once | Message delivered, maybe duplicates | Retry until ack |
| Exactly-once | Each message processed exactly once | Hard to achieve |

**Why This Matters:**
- Network failures happen
- Most systems provide at-least-once
- Exactly-once requires careful design

### Key Design Principles

**At-Least-Once (Default for Most Systems):**
```
Producer sends → Broker acks → Success

If no ack (timeout):
  Producer retries → Potential duplicate
```

Consumer must be idempotent!

**Exactly-Once (How Kafka Achieves It):**
1. **Idempotent Producer:** Broker deduplicates using sequence number
2. **Transactional Writes:** Atomic batch writes across partitions
3. **Consumer:** Read committed only

**Making Consumers Idempotent:**
```
1. Dedupe by message ID:
   if message_id in processed_set:
     skip
   else:
     process and add to processed_set

2. Use database transactions:
   BEGIN
     INSERT INTO processed_messages (id) VALUES (msg_id)
     -- will fail if duplicate
     UPDATE business_table ...
   COMMIT
```

### Trade-offs & Decision Matrix

| Guarantee | Throughput | Complexity | Use Case |
|-----------|------------|------------|----------|
| At-most-once | Highest | Low | Metrics, logs (OK to lose) |
| At-least-once | High | Medium | Most applications |
| Exactly-once | Lower | High | Payments, critical operations |

### Interview Perspective

**Strong signals:**
- ✅ "At-least-once is standard; consumers must be idempotent"
- ✅ "Exactly-once is expensive; use only where critical"
- ✅ "Deduplicate using message ID or database constraint"

---

## 4.3 Event-Driven Architecture

### Concept Overview (What & Why)

**Event-Driven:** Services communicate by publishing and subscribing to events, rather than direct calls.

```
Traditional:
  Order Service → (HTTP) → Inventory Service
                → (HTTP) → Payment Service
                → (HTTP) → Notification Service

Event-Driven:
  Order Service → publishes OrderCreated event
  
  Inventory Service ← subscribes → OrderCreated
  Payment Service ← subscribes → OrderCreated
  Notification Service ← subscribes → OrderCreated
```

**Benefits:**
- Loose coupling (services don't know about each other)
- Scalability (add subscribers without changing publisher)
- Resilience (subscriber down? Events queue up)

**Challenges:**
- Eventual consistency (not immediate)
- Debugging harder (no linear call stack)
- Event schema evolution

### Key Design Principles

**Event Types:**

| Type | Description | Example |
|------|-------------|---------|
| Domain Event | Business fact that happened | OrderPlaced, PaymentReceived |
| Integration Event | For inter-service communication | Same as domain, but published |
| Command | Request to do something | ProcessPayment |

**Event Design:**
```json
{
  "eventId": "uuid",
  "eventType": "OrderPlaced",
  "timestamp": "2024-01-15T10:30:00Z",
  "aggregateId": "order-123",
  "data": {
    "orderId": "123",
    "customerId": "456",
    "total": 99.99
  }
}
```

**Event Schema Evolution:**
- Add optional fields (safe)
- Remove unused fields (safe if consumers ignore unknown)
- Rename fields (breaking!)
- Use schema registry (Avro, Protobuf)

### Trade-offs & Decision Matrix

| Aspect | Request-Response | Event-Driven |
|--------|-----------------|--------------|
| Coupling | Tight | Loose |
| Latency | Lower (sync) | Higher (async) |
| Consistency | Strong possible | Eventual |
| Debugging | Easier | Harder |
| Resilience | Lower (caller waits) | Higher (events queued) |

### Real-World Examples

**Event Sourcing (Advanced):**
- Store events as source of truth
- Rebuild state by replaying events
- Used by: Banking ledgers, audit systems

**CQRS (Command Query Responsibility Segregation):**
- Separate write and read models
- Events update read model asynchronously
- Used by: High-scale reads with different access patterns

### Interview Perspective

**Strong signals:**
- ✅ "Events for loose coupling between services"
- ✅ "Eventual consistency is acceptable here"
- ✅ "Event schema versioning is important"

**Common trap:**
- ❌ "Everything should be event-driven" (sync is often simpler)

---

## 4.4 Backpressure Handling

### Concept Overview (What & Why)

**Backpressure:** When a system is overwhelmed by incoming data faster than it can process.

```
Producer (1000 msg/s) → Queue → Consumer (100 msg/s)
                         ↓
                   Queue fills up!
```

**Why This Matters:**
- Unbounded queues → OOM
- Dropped messages → Data loss
- Slow consumers are common

### Key Design Principles

**Backpressure Strategies:**

| Strategy | Description | Trade-off |
|----------|-------------|-----------|
| Drop | Discard new messages | Data loss |
| Buffer | Queue messages (bounded) | Memory/disk usage |
| Throttle producer | Slow down producer | Latency |
| Scale consumers | Add more consumers | Cost |
| Sample | Process subset of messages | Approximate results |

**Implementation Patterns:**

**1. Bounded Queue:**
```
Queue with max size
When full: Block producer OR reject message
```

**2. Rate Limiting Producer:**
```
Token bucket on producer side
Limits how fast messages are sent
```

**3. Dynamic Consumer Scaling:**
```
Monitor queue depth
Depth > threshold → Add consumer
Depth < threshold → Remove consumer
```

**4. Circuit Breaker:**
```
If downstream overwhelmed:
  Stop sending new requests
  Wait for recovery
```

### Trade-offs & Decision Matrix

| Scenario | Strategy |
|----------|----------|
| Metrics/logs (loss OK) | Sample or drop |
| Important data | Buffer + scale consumers |
| Spike traffic | Buffer short-term, scale |
| Steady overload | Must throttle producer |

### Real-World Examples

**Kafka:**
- Consumers control their own pace
- Retention allows consumers to catch up
- Lag metric indicates backpressure

**RabbitMQ:**
- Memory and disk alarms
- Blocks producers when overwhelmed
- Prefetch limits for consumers

### Interview Perspective

**Strong signals:**
- ✅ "Bounded queues with overflow handling"
- ✅ "Monitor queue depth, auto-scale consumers"
- ✅ "Accept data loss for non-critical data"

---

### Section 4 Cheat Sheet

```
MESSAGE QUEUES & EVENTS

KAFKA vs RABBITMQ vs SQS:
Kafka: Log-based, replay, high throughput
RabbitMQ: Traditional queue, routing, low latency
SQS: Managed, AWS native, simple

DELIVERY GUARANTEES:
At-most-once: Might lose (fire-forget)
At-least-once: Duplicates possible (default)
Exactly-once: Expensive, rare

IDEMPOTENT CONSUMERS:
• Dedupe by message ID
• Database unique constraints
• Conditional updates (version checks)

EVENT-DRIVEN ARCHITECTURE:
• Services publish/subscribe to events
• Loose coupling, eventual consistency
• Event schema versioning important

BACKPRESSURE:
Problem: Producer > Consumer capacity
Solutions:
• Bounded queues
• Throttle producer
• Scale consumers
• Drop/sample (if acceptable)

KAFKA SPECIFICS:
• Ordering per partition
• Consumer groups for scaling
• Offset management for replay

EVENT DESIGN:
{
  eventId, eventType, timestamp,
  aggregateId, data: {...}
}
```

---

# Phase 2 Summary: Distributed Systems Maturity

These concepts form the foundation of distributed systems design. Mastery here separates senior engineers from those who just use distributed systems without understanding them.

**Key Takeaways:**

1. **Partitioning:** Shard key selection is critical; cross-shard operations are expensive
2. **Replication:** Understand the consistency/availability/latency trade-offs
3. **Consensus:** Use existing tools (ZooKeeper, etcd), don't build your own
4. **Messaging:** Choose the right tool, design for at-least-once, handle backpressure

**Interviewer's Perspective:**

| Level | Expectation |
|-------|-------------|
| Senior (L5) | Know all concepts, apply appropriately |
| Staff (L6) | Deep understanding, can explain trade-offs clearly |
| Principal (L7) | Can challenge assumptions, knows edge cases and failure modes |

**Common Failure Modes to Discuss:**
- Network partitions
- Replication lag
- Hot partitions
- Consumer lag
- Split brain

**Questions That Probe Deeply:**
- "What happens when the network partitions?"
- "How do you handle a slow consumer?"
- "What if the shard key changes?"
- "How do you debug an event that was lost?"
