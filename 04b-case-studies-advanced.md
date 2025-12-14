# Phase 4b: Advanced Case Studies (Senior+ Level)

These case studies are expected at Staff and Principal levels. They involve more complex trade-offs, deeper technical discussions, and often touch on infrastructure concerns that junior engineers rarely encounter.

---

# Case Study 8: Distributed Cache (Redis-like)

## 1. Requirements Clarification

### Functional Requirements
- Key-value storage (GET, SET, DELETE)
- TTL support
- Atomic operations (INCR, LPUSH, etc.)
- Data structures (strings, lists, sets, hashes)
- Pub/Sub (optional)

### Non-Functional Requirements
- Sub-millisecond latency
- High throughput (100K+ ops/second per node)
- High availability
- Horizontal scaling
- Persistence options

### Clarifying Questions
- "Persistence requirement?" (Optional, configurable)
- "Cluster size?" (Start with 10 nodes, scale to 100+)
- "Consistency model?" (Eventual for replication, strong for single key)

## 2. Scale Estimation

```
Operations: 10 million ops/second (cluster)
Data size: 1 TB across cluster
Node capacity: 100K ops/second, 100 GB data
Nodes needed: ~100 for ops, ~10 for data → ops-bound

Latency: p99 < 1ms (within datacenter)
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Clients                                     │
│                    (Smart client with routing)                           │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────────────┐
         ▼                          ▼                          ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   Cache Node 1  │      │   Cache Node 2  │      │   Cache Node N  │
│   (Master)      │      │   (Master)      │      │   (Master)      │
│   Slots 0-5460  │      │   Slots 5461-   │      │   Slots 10923-  │
│                 │      │   10922         │      │   16383         │
└────────┬────────┘      └────────┬────────┘      └────────┬────────┘
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   Replica 1a    │      │   Replica 2a    │      │   Replica Na    │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

## 4. Deep Dive: Data Partitioning

### Hash Slot Approach (Redis Cluster)

```
Total hash slots: 16384
Key → CRC16(key) % 16384 → slot → node

Example:
- Node 1: slots 0-5460
- Node 2: slots 5461-10922
- Node 3: slots 10923-16383

Key "user:123" → CRC16("user:123") % 16384 = 8752 → Node 2
```

### Slot Migration (Resharding)

```
Adding a new node:
1. Allocate slots to new node (from existing)
2. Migrate keys in those slots
3. Update slot mapping
4. Clients refresh slot table

During migration:
- ASK redirect for migrating keys
- MOVED redirect for completed slots
```

### Hash Tags (Keep Related Keys Together)

```
Keys: user:{123}:profile, user:{123}:settings
Hash on: {123} (content within braces)
Both keys → same slot → same node

Use for: Multi-key operations, Lua scripts
```

## 5. Deep Dive: Replication and Failover

### Replication Model

```
Master-replica (async by default)
Each master has 1+ replicas
Replicas can serve reads (with staleness)

Write path:
Client → Master → ACK → Async replicate to replicas

Read path (if replica reads enabled):
Client → Master or Replica
```

### Failover Process

```
1. Replicas ping master every second
2. If no response for N seconds → master suspected down
3. Replicas vote (majority required)
4. Winning replica promotes to master
5. Other replicas replicate from new master
6. Cluster updates slot mapping
```

### Split Brain Prevention

```
Master isolated from majority:
- Can't see other masters
- Steps down, refuses writes
- Prevents two masters for same slot
```

## 6. Deep Dive: Persistence

### RDB (Snapshotting)

```
Periodic point-in-time snapshot
Fork process, write to disk
Pros: Compact, fast recovery
Cons: Data loss between snapshots
```

### AOF (Append-Only File)

```
Log every write operation
Options: always, everysec, never

Recovery: Replay AOF
Pros: Minimal data loss
Cons: Larger files, slower recovery
```

### Hybrid

```
Use both: RDB for base, AOF for recent changes
Recovery: Load RDB, then replay AOF
Best of both worlds
```

## 7. Deep Dive: Memory Management

### Eviction Policies

| Policy | Description |
|--------|-------------|
| noeviction | Return error when memory full |
| allkeys-lru | Evict any key using LRU |
| volatile-lru | Evict keys with TTL using LRU |
| allkeys-random | Random eviction |
| volatile-ttl | Evict keys with shortest TTL |

### Memory Optimization

```
- Use Redis data structures (not serialized JSON)
- Short key names
- Use TTLs aggressively
- Consider compression for large values
```

## 8. Interview Perspective

**Common follow-ups:**
- "How to handle hot keys?" → Local caching, key splitting
- "How to ensure consistency?" → WAIT command for sync replication
- "Thundering herd?" → Probabilistic early refresh, locking

### Cheat Sheet

```
DISTRIBUTED CACHE (Redis-like)

PARTITIONING:
Hash slots: 16384 slots
Key → CRC16(key) % 16384 → slot → node
Hash tags: {tag} for co-location

REPLICATION:
Master-replica (async default)
Failover: Replica promotion (majority vote)
WAIT command for sync replication

PERSISTENCE:
RDB: Point-in-time snapshots
AOF: Append-only log (every write)
Hybrid: RDB + AOF for best durability

EVICTION:
LRU, LFU, TTL-based
Configure max memory
noeviction returns error

OPERATIONS:
100K+ ops/second per node
p99 < 1ms latency
Single-threaded (mostly) for simplicity

HOT KEY SOLUTIONS:
- Client-side caching
- Key splitting (key:1, key:2)
- Replicas for read distribution
```

---

# Case Study 9: Search Autocomplete

## 1. Requirements Clarification

### Functional Requirements
- Return suggestions as user types
- Rank by popularity/relevance
- Personalization (optional)
- Support for typos/fuzzy matching (optional)

### Non-Functional Requirements
- Ultra-low latency (< 50ms)
- High availability
- Fresh data (new trending terms within hours)
- Scale to millions of QPS

### Clarifying Questions
- "How many suggestions?" (5-10)
- "Personalized?" (Start with global, add personal later)
- "Fuzzy matching?" (Nice to have, not MVP)

## 2. Scale Estimation

```
Queries: 10 billion/day
Peak QPS: 200,000
Unique queries: 100 million

Per keystroke: ~5 API calls for average query
Actual autocomplete calls: 50 billion/day

Data:
- 100M queries * 50 bytes avg = 5 GB
- With metadata and indexes: ~50 GB
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              User Types                                  │
│                         (keystroke by keystroke)                         │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
               ┌─────────────────┐   ┌─────────────────┐
               │   CDN Cache     │   │   API Gateway   │
               │   (if possible) │   │                 │
               └─────────────────┘   └────────┬────────┘
                                              │
                                              ▼
                                   ┌─────────────────┐
                                   │  Autocomplete   │
                                   │  Service        │
                                   └────────┬────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    ▼                       ▼                       ▼
         ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
         │   Trie Servers  │     │   Cache Layer   │     │   Analytics     │
         │   (In-Memory)   │     │   (Redis)       │     │   (Popularity)  │
         └─────────────────┘     └─────────────────┘     └─────────────────┘
```

## 4. Deep Dive: Data Structures

### Trie (Prefix Tree)

```
        root
       / | \
      a  b  c
     /       \
    p         a
   /           \
  p             r
 /
l
e

Query "ap" → traverse to 'a' → 'p' → return all completions below
```

### Trie Node Structure

```python
class TrieNode:
    children: Dict[char, TrieNode]
    is_end: bool
    top_k: List[str]  # Pre-computed top-k completions from this node
    count: int
```

### Optimization: Pre-compute Top-K

```
At each node, store top-k completions
Query: Navigate to prefix → return node.top_k

Without: Traverse all children, rank, return top-k (slow)
With: O(len(prefix)) lookup (fast)
```

## 5. Deep Dive: Ranking

### Popularity-Based Ranking

```
Data collection:
- Query logs → Count frequency
- Time decay: Recent queries weighted higher
- Remove spam/inappropriate

Score = frequency * time_decay_factor
```

### Personalization (Advanced)

```
Combine:
- Global popularity: 70%
- User history: 20%
- User profile/context: 10%

Store per-user recent queries
Blend with global at query time
```

## 6. Deep Dive: Update Pipeline

### Data Flow

```
Query logs → Aggregation (hourly) → Trie Builder → Deploy to servers

Aggregation:
1. Collect all queries
2. Count, filter, rank
3. Build new trie
4. Deploy incrementally

Latency: New trending terms appear within 1-2 hours
```

### Incremental Updates

```
Option 1: Full rebuild
- Build new trie, swap out
- Simple, requires memory for two tries

Option 2: Delta updates
- Update counts in place
- Re-compute top-k periodically
- More complex, less memory
```

## 7. Deep Dive: Serving

### In-Memory Serving

```
Trie fits in memory: 50 GB (fits in large instances)
Replicate across multiple servers
No database lookup in hot path
```

### Caching

```
Cache prefix → results in Redis

Key: "ac:iph" (autocomplete for "iph")
Value: ["iphone 15", "iphone case", ...]
TTL: 1 hour

Cache hit rate: ~60% for popular prefixes
```

### Client-Side Optimization

```
Debounce: Wait 100ms after keystroke before calling
Client cache: Remember recent suggestions
Prefetch: After first character, fetch next likely chars
```

## 8. Interview Perspective

**Common follow-ups:**
- "How to handle typos?" → Edit distance, phonetic matching
- "How to remove inappropriate?" → Blocklist, ML classifier
- "How to personalize?" → Blend user history with global

### Cheat Sheet

```
SEARCH AUTOCOMPLETE

DATA STRUCTURE:
Trie (prefix tree)
Pre-compute top-k at each node
O(len(prefix)) lookup

RANKING:
Popularity (query frequency)
Time decay for freshness
Personalization (user history blend)

ARCHITECTURE:
In-memory trie servers
Redis cache for hot prefixes
CDN for static suggestions

UPDATE PIPELINE:
Query logs → Aggregate → Build trie → Deploy
Latency: 1-2 hours for new terms

CLIENT OPTIMIZATION:
Debounce (100ms)
Client-side caching
Prefetch likely next chars

LATENCY BUDGET:
< 50ms p99
In-memory serving essential
Cache frequently accessed prefixes
```

---

# Case Study 10: Metrics / Monitoring System

## 1. Requirements Clarification

### Functional Requirements
- Ingest metrics from thousands of services
- Store time-series data
- Query for dashboards and alerts
- Support aggregations (avg, sum, percentiles)
- Alerting on thresholds

### Non-Functional Requirements
- High write throughput (millions of data points/second)
- Reasonable query latency (< 1s for dashboards)
- Data retention (high-resolution: 7 days, downsampled: years)
- Highly available (monitoring must work when things break)

### Clarifying Questions
- "What's the cardinality?" (100K unique time series)
- "Resolution?" (1 second for real-time, 1 minute for dashboards)
- "Retention?" (7 days raw, 1 year downsampled)

## 2. Scale Estimation

```
Services: 10,000
Metrics per service: 100
Data points per metric per second: 1
Total: 1 million data points/second

Storage:
- 1M points/sec * 86400 sec/day * 7 days = 600 billion points
- Each point: 16 bytes (timestamp + value) + metadata
- ~10 TB for 7 days raw data
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Metric Sources                                   │
│              (Services, Hosts, Containers)                               │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────┐
                         │   Ingestion     │
                         │   (Kafka)       │
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
         ┌─────────────────┐         ┌─────────────────┐
         │   Write Path    │         │   Aggregation   │
         │   (TSDB)        │         │   (Streaming)   │
         └────────┬────────┘         └─────────────────┘
                  │
                  ▼
         ┌─────────────────┐
         │   Time-Series   │
         │   Storage       │
         └────────┬────────┘
                  │
         ┌────────┴────────┐
         ▼                 ▼
┌─────────────────┐ ┌─────────────────┐
│   Query Engine  │ │   Alerting      │
│   (Dashboards)  │ │   Engine        │
└─────────────────┘ └─────────────────┘
```

## 4. Deep Dive: Time-Series Storage

### Data Model

```
Metric: cpu_usage
Labels: {host: "server1", region: "us-east", service: "api"}
Timestamp: 1705312000
Value: 75.5

Time series = metric name + unique label combination
```

### Storage Engine Options

| Option | Pros | Cons |
|--------|------|------|
| InfluxDB | Purpose-built, easy | Clustering is commercial |
| Prometheus | Popular, pull-based | Single node limitation |
| TimescaleDB | SQL interface, PostgreSQL | Less specialized |
| ClickHouse | Fast analytics | Not purpose-built for metrics |
| Custom (LSM-tree) | Full control | Significant engineering |

### Write-Optimized Storage (LSM-Tree)

```
Writes:
1. Write to in-memory buffer (memtable)
2. When full, flush to disk as SSTable
3. Periodic compaction of SSTables

Why good for metrics:
- Sequential writes (fast)
- No random updates
- Append-only pattern
```

### Compression

```
Time-series specific compression:
- Delta-of-delta for timestamps
- XOR encoding for values
- 10x+ compression typical

Example:
Raw: [1705312000, 1705312001, 1705312002]
Delta: [1705312000, 1, 1]
Delta-of-delta: [1705312000, 1, 0]
```

## 5. Deep Dive: Query Engine

### Query Types

```
Instant query: Value at specific time
  SELECT cpu_usage{host="server1"} AT 1705312000

Range query: Values over time range
  SELECT avg(cpu_usage{host="server1"})[5m]

Aggregation: Across labels
  SELECT avg(cpu_usage) BY (region)
```

### Query Optimization

```
- Index by metric name and labels
- Pre-aggregate common queries
- Cache recent query results
- Limit time range and cardinality
```

## 6. Deep Dive: Downsampling

### Multi-Resolution Storage

```
Raw data: 1-second resolution, 7 days
5-minute rollups: 30 days
1-hour rollups: 1 year
1-day rollups: 5 years

Rollup: avg, min, max, count per bucket
```

### Rollup Pipeline

```
Background job:
1. Read raw data for time window
2. Compute aggregates (avg, min, max)
3. Write to rollup table
4. Delete old raw data
```

## 7. Deep Dive: Alerting

### Alert Evaluation

```
Alert rule:
  IF avg(cpu_usage{host="server1"})[5m] > 80
  FOR 5 minutes
  THEN notify on-call

Evaluation:
- Run query every 30 seconds
- If condition true for duration, fire alert
- Deduplicate (don't re-alert for same issue)
```

### Alert State Machine

```
PENDING → FIRING → RESOLVED
         ↑         │
         └────────┘ (flapping prevention)
```

## 8. Interview Perspective

**Common follow-ups:**
- "High cardinality problem?" → Limit labels, drop high-cardinality
- "How to avoid alert storms?" → Grouping, inhibition, rate limiting
- "Cold storage?" → S3 for archived data, query when needed

### Cheat Sheet

```
METRICS / MONITORING

DATA MODEL:
metric_name{labels} timestamp value
Time series = name + unique label set

STORAGE:
LSM-tree based (write-optimized)
Compression: Delta-of-delta, XOR
Purpose-built TSDB or column store

INGESTION:
Kafka for buffering
Push or pull model
Batch writes for efficiency

DOWNSAMPLING:
Raw: 7 days
5-min rollups: 30 days
1-hour rollups: 1 year

ALERTING:
Periodic evaluation (30s)
FOR duration (prevent flapping)
Grouping and inhibition

QUERY:
Index by metric + labels
Pre-aggregate hot queries
Limit cardinality
```

---

# Case Study 11: Feature Flag Service

## 1. Requirements Clarification

### Functional Requirements
- Define feature flags (boolean, percentage, targeting)
- Evaluate flags for users
- Targeting rules (user attributes, segments)
- Override rules (specific users, testing)
- Audit log

### Non-Functional Requirements
- Ultra-low latency evaluation (< 10ms)
- High availability (critical for deployments)
- Eventual consistency OK (flag changes take effect in minutes)
- Handle millions of evaluations/second

### Clarifying Questions
- "Evaluation location?" (Server-side for control, client-side for speed)
- "Targeting complexity?" (User attributes, percentage rollouts)
- "Consistency requirement?" (Minutes delay acceptable)

## 2. Scale Estimation

```
Applications: 1,000
Flags per application: 100
Users: 100 million
Evaluations: 1 billion/day = 12,000/second

Flag definitions: 100K flags * 1 KB = 100 MB
Fits in memory on each app server
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Admin UI                                         │
│              (Create, Modify, Target Flags)                              │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────┐
                         │   Flag Service  │
                         │   (API + Store) │
                         └────────┬────────┘
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼                                                 ▼
┌─────────────────┐                             ┌─────────────────┐
│   Flags DB      │                             │   Flags Cache   │
│   (PostgreSQL)  │                             │   (Redis)       │
└─────────────────┘                             └────────┬────────┘
                                                         │
         ┌───────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Application SDKs                                 │
│              (Embedded, Local Evaluation)                                │
└─────────────────────────────────────────────────────────────────────────┘
```

## 4. Deep Dive: Flag Definition

### Flag Schema

```json
{
  "key": "new_checkout_flow",
  "type": "boolean",
  "default": false,
  "variations": [true, false],
  "rules": [
    {
      "attribute": "user_tier",
      "operator": "equals",
      "value": "premium",
      "variation": 0
    },
    {
      "attribute": "user_id",
      "operator": "in",
      "value": ["user_123", "user_456"],
      "variation": 0
    }
  ],
  "rollout": {
    "percentage": 10,
    "variation": 0
  },
  "off_variation": 1
}
```

### Rule Types

| Type | Description |
|------|-------------|
| Boolean | Simple on/off |
| Percentage rollout | X% get new experience |
| User targeting | Specific user IDs |
| Attribute targeting | Based on user properties |
| Segment targeting | Pre-defined user groups |

## 5. Deep Dive: Evaluation

### Evaluation Logic

```python
def evaluate(flag, user):
    # 1. Check if flag is enabled
    if not flag.enabled:
        return flag.off_variation
    
    # 2. Check user-specific overrides
    if user.id in flag.user_overrides:
        return flag.user_overrides[user.id]
    
    # 3. Evaluate targeting rules
    for rule in flag.rules:
        if matches(rule, user):
            return rule.variation
    
    # 4. Apply percentage rollout
    if flag.rollout:
        bucket = hash(user.id + flag.key) % 100
        if bucket < flag.rollout.percentage:
            return flag.rollout.variation
    
    # 5. Return default
    return flag.default_variation
```

### Consistent Bucketing

```
User always gets same experience (no flapping):
bucket = hash(user_id + flag_key) % 100

Same user + same flag = same bucket = same experience
Changing percentage from 10% to 20%:
- Users in 0-10% still get new experience
- Users in 10-20% now get new experience
- Users in 20-100% still get old experience
```

## 6. Deep Dive: SDK Architecture

### Two Modes

**Server-Side SDK:**
```
- Runs in backend service
- Fetches flags periodically (or streaming)
- Evaluates locally (fast)
- Never exposes rules to client
```

**Client-Side SDK:**
```
- Runs in browser/mobile
- Sends user context to backend
- Gets pre-evaluated variations
- No local rule evaluation (security)
```

### Flag Synchronization

```
Option 1: Polling
- SDK polls every 30 seconds
- Simple, slight delay

Option 2: Streaming (SSE/WebSocket)
- Real-time updates
- More complex, immediate

Option 3: Hybrid
- Stream for updates
- Poll as fallback
```

## 7. Interview Perspective

**Common follow-ups:**
- "How to handle SDK failures?" → Default values, cached flags
- "How to audit changes?" → Event log of all changes
- "How to test new rules?" → Override rules, targeting test users

### Cheat Sheet

```
FEATURE FLAG SERVICE

FLAG TYPES:
Boolean, percentage rollout, targeting, segments

EVALUATION:
1. Check enabled
2. User overrides
3. Targeting rules
4. Percentage rollout
5. Default value

CONSISTENT BUCKETING:
hash(user_id + flag_key) % 100
Same user = same experience
Gradual rollout maintains consistency

ARCHITECTURE:
Flag Service: CRUD + evaluation API
SDK: Local evaluation (fast)
Sync: Polling (30s) or streaming

RESILIENCE:
SDK caches flags locally
Default values on failure
Never block on flag evaluation

ADVANCED:
Experimentation (A/B tests)
Segments (user groups)
Scheduling (timed releases)
```

---

# Case Study 12: Identity & Access Management (IAM)

## 1. Requirements Clarification

### Functional Requirements
- User registration and authentication
- Role-based access control (RBAC)
- Permission management
- Multi-tenancy support
- SSO/Federation (SAML, OIDC)
- Audit logging

### Non-Functional Requirements
- High security (this is THE security system)
- Low latency for permission checks (< 10ms)
- High availability (system unusable without auth)
- Compliance (audit, encryption)

### Clarifying Questions
- "Scale?" (1M users, 10K roles, 100K permissions)
- "Multi-tenant?" (Yes, SaaS model)
- "Federation?" (Support SAML and OIDC)

## 2. Scale Estimation

```
Users: 1 million
Auth requests: 10 million/day = 120/second
Permission checks: 1 billion/day = 12,000/second

Permission storage:
- 100K permissions * 100 bytes = 10 MB
- Role-permission mappings: 10 MB
- User-role mappings: 100 MB
- Total: ~200 MB (fits in memory/cache)
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Applications                                     │
│              (Call IAM for auth and authz)                               │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────┐
                         │   API Gateway   │
                         │   (JWT Validation)
                         └────────┬────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Auth Service   │    │  Permission     │    │  User Service   │
│  (Login, SSO)   │    │  Service        │    │  (Profile)      │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                ▼
                     ┌─────────────────┐
                     │   IAM Database  │
                     │   (PostgreSQL)  │
                     └─────────────────┘
```

## 4. Deep Dive: Authentication

### Flow

```
1. User submits credentials
2. Validate against stored hash (bcrypt)
3. Generate tokens:
   - Access token (JWT, 15 min)
   - Refresh token (opaque, 7 days)
4. Return tokens

Subsequent requests:
1. Include access token in header
2. API Gateway validates JWT signature
3. If expired, use refresh token for new access token
```

### Token Structure

```json
// Access Token (JWT)
{
  "header": {"alg": "RS256", "typ": "JWT"},
  "payload": {
    "sub": "user_123",
    "tenant_id": "tenant_456",
    "roles": ["admin", "user"],
    "exp": 1705312900
  },
  "signature": "..."
}

// Refresh Token
Stored in DB, opaque to client
{
  "token_id": "rt_abc123",
  "user_id": "user_123",
  "expires_at": "2024-01-22",
  "revoked": false
}
```

## 5. Deep Dive: Authorization (RBAC)

### Data Model

```
Users ←→ Roles ←→ Permissions

Table: users
| user_id | tenant_id | email | password_hash |
|---------|-----------|-------|---------------|
| u_123   | t_456     | a@b.c | bcrypt_hash   |

Table: roles
| role_id | tenant_id | name    | description |
|---------|-----------|---------|-------------|
| r_001   | t_456     | admin   | Full access |
| r_002   | t_456     | viewer  | Read only   |

Table: user_roles
| user_id | role_id |
|---------|---------|
| u_123   | r_001   |

Table: role_permissions
| role_id | permission |
|---------|------------|
| r_001   | users:read |
| r_001   | users:write|
| r_002   | users:read |
```

### Permission Check

```python
def has_permission(user_id, permission, resource=None):
    # 1. Get user's roles
    roles = get_user_roles(user_id)
    
    # 2. Get permissions for roles
    permissions = get_role_permissions(roles)
    
    # 3. Check if permission is granted
    if permission in permissions:
        return True
    
    # 4. Check resource-specific permissions
    if resource:
        return check_resource_permission(user_id, permission, resource)
    
    return False
```

### Caching Permissions

```
Cache user's effective permissions:
Key: permissions:{user_id}
Value: Set of permissions
TTL: 5 minutes

On role change:
- Invalidate cache for affected users
- Or: Wait for TTL (eventual consistency)
```

## 6. Deep Dive: Multi-Tenancy

### Isolation Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Row-level | tenant_id column in all tables | Most SaaS apps |
| Schema-level | Separate schema per tenant | Medium isolation |
| Database-level | Separate DB per tenant | High isolation |

### Implementation

```sql
-- Row-level isolation
SELECT * FROM users WHERE tenant_id = ? AND user_id = ?;

-- Every query includes tenant_id
-- Application enforces tenant context
```

## 7. Deep Dive: Federation (SSO)

### SAML Flow

```
1. User accesses app
2. App redirects to IdP (Okta, ADFS)
3. User authenticates at IdP
4. IdP returns SAML assertion
5. App validates assertion, creates session
```

### OIDC Flow

```
1. User accesses app
2. App redirects to IdP with /authorize
3. User authenticates
4. IdP redirects back with code
5. App exchanges code for tokens
6. App validates ID token, creates session
```

## 8. Interview Perspective

**Common follow-ups:**
- "How to revoke tokens?" → Token blacklist or short expiry
- "How to handle permission changes?" → Invalidate cache, short TTL
- "How to scale permission checks?" → Cache effective permissions

### Cheat Sheet

```
IDENTITY & ACCESS MANAGEMENT

AUTHENTICATION:
Password: bcrypt hash, secure storage
Tokens: JWT (access) + opaque (refresh)
Access: 15 min, refresh: 7 days

AUTHORIZATION (RBAC):
Users → Roles → Permissions
Cache effective permissions
Check: user_id + permission + resource

MULTI-TENANCY:
Row-level isolation: tenant_id everywhere
All queries scoped to tenant
Enforce at application layer

FEDERATION:
SAML: Enterprise SSO
OIDC: Modern SSO, mobile friendly

TOKEN MANAGEMENT:
Short-lived access tokens
Refresh token rotation
Revocation via blacklist or short expiry

SECURITY:
Encrypt at rest
TLS everywhere
Audit all access
Rate limit auth endpoints
MFA for sensitive operations
```

---

# Phase 4b Summary

These advanced case studies demonstrate the depth expected at Staff+ levels:

1. **Distributed Cache:** Consistent hashing, persistence, cluster management
2. **Autocomplete:** Trie optimization, ranking, freshness
3. **Metrics:** Time-series storage, downsampling, alerting
4. **Feature Flags:** Evaluation logic, consistent bucketing, SDK design
5. **IAM:** RBAC, multi-tenancy, federation

**Key Differences from Basic Case Studies:**

| Aspect | Basic | Advanced |
|--------|-------|----------|
| Depth | Cover all components | Deep dive into 2-3 |
| Trade-offs | Obvious choices | Nuanced decisions |
| Failure modes | Basic handling | Comprehensive |
| Scale | Handle scale | Design for scale evolution |
| Operations | Mention | Detailed discussion |

**Staff+ Level Expectations:**

- Challenge requirements and assumptions
- Discuss operational concerns (deployment, monitoring)
- Consider cost implications
- Explain second-order effects
- Propose iterative evolution path
