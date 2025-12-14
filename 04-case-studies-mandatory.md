# Phase 4: System Design Case Studies (Mandatory Designs)

This phase provides complete end-to-end design walkthroughs for must-know system design problems. For each case study, we follow the interview structure: requirements, scale estimation, architecture, deep dives, bottlenecks, failure handling, and trade-offs.

---

# Case Study 1: URL Shortener (TinyURL)

## 1. Requirements Clarification

### Functional Requirements
- Shorten a long URL to a short URL
- Redirect short URL to original long URL
- Custom short URLs (optional)
- URL expiration (optional)
- Analytics (optional for MVP)

### Non-Functional Requirements
- Highly available (reads are critical)
- Low latency redirects (< 100ms)
- Short URLs should be unpredictable (not sequential)

### Clarifying Questions to Ask
- "What's the expected read/write ratio?" (Typically 100:1)
- "Should URLs expire?" (Yes, default 2 years)
- "Do we need analytics?" (Track later, not MVP)
- "Custom aliases allowed?" (Yes, optional)

## 2. Scale Estimation

### Traffic Estimates
```
New URLs created: 100 million/month
Read/Write ratio: 100:1
Reads: 10 billion/month

Writes: 100M / (30 * 24 * 3600) ≈ 40 writes/second
Reads: 10B / (30 * 24 * 3600) ≈ 4000 reads/second
Peak: 2-3x average ≈ 10,000 reads/second
```

### Storage Estimates
```
URL record: ~500 bytes (short URL, long URL, metadata)
100M URLs/month * 12 months * 5 years = 6 billion URLs
Storage: 6B * 500 bytes = 3 TB

Add replication (3x) = 9 TB
```

### Short URL Length
```
Characters: a-z, A-Z, 0-9 = 62 characters
6 characters: 62^6 = 56 billion combinations
7 characters: 62^7 = 3.5 trillion combinations

6 characters is sufficient for billions of URLs
```

## 3. High-Level Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Load      │────▶│  API        │
│             │     │   Balancer  │     │  Servers    │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                    ┌──────────────────────────┼──────────────────────────┐
                    │                          │                          │
                    ▼                          ▼                          ▼
            ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
            │   Cache     │           │   ID Gen    │           │   Database  │
            │   (Redis)   │           │   Service   │           │   (NoSQL)   │
            └─────────────┘           └─────────────┘           └─────────────┘
```

### API Design

```
POST /urls
{
  "long_url": "https://example.com/very/long/path",
  "custom_alias": "my-link",  // optional
  "expiration": "2025-12-31"  // optional
}
Response: {"short_url": "https://tiny.url/abc123"}

GET /{short_url}
Response: 301/302 Redirect to long_url

GET /urls/{short_url}/stats
Response: {"clicks": 1000, "created": "2024-01-01"}
```

## 4. Deep Dive: Short URL Generation

### Approach 1: Base62 Encoding of Counter

```
Counter: 1, 2, 3, ...
Base62(1) = "1"
Base62(1000000) = "4c92"
Base62(56800235584) = "zzzzzz"
```

**Pros:** Simple, no collisions
**Cons:** Predictable, need distributed counter

### Approach 2: MD5/SHA256 Hash + Truncate

```
hash = MD5(long_url)
short_url = hash[:6]

Collision? Try hash[:7], hash[1:7], etc.
```

**Pros:** No coordination needed
**Cons:** Collisions possible, need collision handling

### Approach 3: Pre-generated Keys (Recommended)

```
Key Generation Service:
- Pre-generate millions of unique keys
- Store in database (used/unused)
- On request, mark key as used and return

Advantages:
- No real-time generation overhead
- Guaranteed unique
- Fast: Just fetch from pool
```

**Implementation:**
```
Table: keys
| key    | used | used_at    |
|--------|------|------------|
| abc123 | true | 2024-01-15 |
| def456 | false| null       |

On URL creation:
1. Fetch unused key (atomic update)
2. Store mapping: short_url → long_url
3. Return short_url
```

## 5. Deep Dive: Database Design

### Schema

```sql
Table: urls
| short_url (PK) | long_url      | created_at | expires_at | user_id |
|----------------|---------------|------------|------------|---------|
| abc123         | https://...   | 2024-01-15 | 2026-01-15 | 123     |

Table: keys (if using key generation)
| key    | used | assigned_at |
|--------|------|-------------|
| abc123 | true | 2024-01-15  |
```

### Database Choice

**Option 1: NoSQL (Cassandra, DynamoDB)**
- Simple key-value lookup
- Horizontal scaling built-in
- Partition by short_url
- **Recommended for this use case**

**Option 2: SQL (PostgreSQL)**
- ACID for custom alias uniqueness
- Better for complex queries
- Shard by short_url if needed

## 6. Deep Dive: Caching

```
Read Flow:
1. Check cache (Redis) for short_url → long_url
2. Cache hit: Return long_url (fast path)
3. Cache miss: Query database, cache result, return

Cache Configuration:
- TTL: 24 hours (hot URLs stay cached)
- Eviction: LRU
- Cache size: Top 20% of URLs = 80% of traffic
```

### Cache Calculation
```
Hot URLs: 20% of 6B = 1.2B URLs
Each entry: 500 bytes
Cache size: 1.2B * 500 = 600 GB

Redis cluster with 600 GB across nodes
```

## 7. Bottlenecks & Solutions

| Bottleneck | Solution |
|------------|----------|
| Read throughput | Cache layer, read replicas |
| Write throughput | Key pre-generation, async writes |
| Database size | Purge expired URLs |
| Hot URLs | CDN, cache, rate limiting |

## 8. Failure Handling

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Database down | Creates fail | Queue writes, retry |
| Cache down | Latency spike | DB can handle, graceful degradation |
| Key service down | Can't create new URLs | Large key buffer, multiple instances |

## 9. Interview Perspective

**Common follow-ups:**
- "How do you handle custom aliases?" → Check uniqueness, reject if taken
- "How do you prevent abuse?" → Rate limiting per user/IP
- "How do you handle expired URLs?" → Background job to purge
- "What about analytics?" → Kafka → Analytics DB (separate path)

**Trade-offs to mention:**
- Pre-gen keys vs. hash: Pre-gen is faster, hash is simpler
- Cache size vs. cost: More cache = lower latency, higher cost
- 301 vs 302 redirect: 301 is cached (fewer hits), 302 always hits server (better analytics)

### Cheat Sheet

```
URL SHORTENER

SCALE:
40 writes/s, 4000 reads/s
3 TB storage over 5 years
6-character short URL = 56B combinations

KEY GENERATION:
Pre-generate keys (recommended)
Or: Base62(counter) with distributed counter
Or: Hash + collision handling

DATA MODEL:
short_url (PK) → long_url, metadata
NoSQL preferred (simple K-V)

CACHING:
Redis for hot URLs
80% hit rate target
LRU eviction, 24h TTL

REDIRECT:
301: Cacheable (fewer analytics)
302: Always hits server (better tracking)
```

---

# Case Study 2: Rate Limiter

## 1. Requirements Clarification

### Functional Requirements
- Limit requests per user/IP/API key
- Different limits for different APIs
- Different limits for different user tiers
- Return 429 when limit exceeded

### Non-Functional Requirements
- Low latency (< 10ms overhead)
- Highly available
- Distributed (works across multiple servers)
- Accurate (not approximate)

### Clarifying Questions
- "What's the time window?" (1 minute, sliding)
- "What identifiers?" (User ID, API key, IP)
- "What happens on limit?" (429 + Retry-After header)
- "Approximate OK?" (Slight over-limit OK, under-limit not OK)

## 2. Scale Estimation

```
Requests: 1 million/second
Rate limit check: Every request
Storage: 100M unique users * 100 bytes = 10 GB

Latency requirement: < 10ms per check
```

## 3. High-Level Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Load      │────▶│   API       │
│             │     │   Balancer  │     │   Gateway   │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                    Rate Limit Check
                                               │
                                               ▼
                                     ┌─────────────┐
                                     │   Redis     │
                                     │   Cluster   │
                                     └─────────────┘
```

## 4. Deep Dive: Rate Limiting Algorithms

### Token Bucket (Recommended)

```
bucket_size = 100 tokens
refill_rate = 10 tokens/second

Algorithm:
1. Check bucket for user
2. If tokens >= 1:
     Consume 1 token, allow request
3. Else:
     Reject with 429

Refill:
tokens += (time_now - last_refill) * refill_rate
tokens = min(tokens, bucket_size)
```

**Redis Implementation:**
```lua
-- Lua script for atomicity
local key = KEYS[1]
local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

-- Refill tokens
local elapsed = now - last_refill
tokens = math.min(capacity, tokens + (elapsed * rate))

-- Check and consume
if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return 1 -- allowed
else
    return 0 -- rejected
end
```

### Sliding Window Counter

```
Current window count + (Previous window count × overlap %)

Example at 15:01:30:
- Current minute (15:01): 40 requests
- Previous minute (15:00): 60 requests
- Overlap: 50% (30 seconds into current minute)

Weighted count = 40 + (60 × 0.5) = 70

If limit is 100, allow.
```

## 5. Distributed Considerations

### Race Conditions

**Problem:** Two servers check simultaneously, both allow, exceeding limit.

**Solution:** Redis atomic operations (Lua scripts)

```
All operations in single Lua script = atomic
No race condition possible
```

### Multi-Region

```
Option 1: Global Redis cluster
- Accurate but higher latency

Option 2: Local rate limiting + sync
- Lower latency
- Slightly over-limit possible

Option 3: Sticky routing
- Same user always hits same region
- Accurate but less flexible
```

## 6. Response Format

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000

{
  "error": "Rate limit exceeded",
  "retry_after_seconds": 30
}
```

## 7. Failure Handling

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Redis down | Can't check limits | Fail open (allow) or fail closed (deny) |
| Network partition | Inconsistent limits | Accept approximate limiting |
| Hot key (celebrity) | One user overwhelms | Shard by user ID |

**Fail Open vs Fail Closed:**
- **Fail Open:** If rate limiter is down, allow all requests (availability over protection)
- **Fail Closed:** If rate limiter is down, reject all requests (protection over availability)

For most APIs: **Fail open** (better to serve than block everyone)
For security-critical: **Fail closed** (better to block than allow abuse)

## 8. Interview Perspective

**Common follow-ups:**
- "What if Redis is slow?" → Local cache with short sync interval
- "How to handle burst?" → Token bucket allows burst up to bucket size
- "Different limits per tier?" → Lookup tier, apply corresponding limit

### Cheat Sheet

```
RATE LIMITER

ALGORITHMS:
Token Bucket: Allows burst (recommended)
Leaky Bucket: Smooth constant rate
Sliding Window: Accurate, more memory
Fixed Window: Simple, boundary issues

REDIS IMPLEMENTATION:
Lua script for atomicity
HMSET for bucket state
TTL to clean up old entries

RESPONSE:
429 Too Many Requests
Retry-After header
X-RateLimit-* headers

DISTRIBUTED:
Atomic operations prevent race
Accept ~5% over-limit for latency
Shard by user ID for hot users

FAILURE MODE:
Fail open: Allow when limiter down
Fail closed: Deny when limiter down
Choose based on use case
```

---

# Case Study 3: Notification System

## 1. Requirements Clarification

### Functional Requirements
- Send notifications via multiple channels (push, SMS, email)
- Support different notification types (transactional, marketing, alerts)
- User preferences (opt-in/out, quiet hours)
- Scheduling and batching
- Delivery tracking

### Non-Functional Requirements
- Reliable delivery (at-least-once)
- Scalable to millions of notifications/day
- Low latency for transactional notifications
- Cost-effective (SMS is expensive)

### Clarifying Questions
- "Priority levels?" (Critical: immediate, Normal: batched)
- "Retry policy?" (Retry failed deliveries, max 3 times)
- "Rate limiting per user?" (Max 10 notifications/hour)

## 2. Scale Estimation

```
Users: 100 million
Daily notifications: 500 million
Peak: 10,000 notifications/second

Storage:
- Templates: Minimal
- Delivery logs: 500M * 30 days * 200 bytes = 3 TB/month
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Notification Service                        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │     Priority Router    │
                    │   (Kafka / SQS)        │
                    └────────────┬───────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  Push Workers   │   │  SMS Workers    │   │  Email Workers  │
│                 │   │                 │   │                 │
└────────┬────────┘   └────────┬────────┘   └────────┬────────┘
         │                     │                     │
         ▼                     ▼                     ▼
   FCM/APNs             Twilio/AWS SNS          SendGrid/SES
```

## 4. Deep Dive: Components

### Notification Service API

```
POST /notifications
{
  "user_id": "123",
  "type": "order_shipped",
  "channels": ["push", "email"],
  "priority": "high",
  "payload": {
    "order_id": "456",
    "tracking_number": "ABC123"
  }
}
```

### Priority Router

```
High priority → Immediate queue (transactional)
Normal priority → Batch queue (digest possible)
Low priority → Scheduled queue (marketing, rate limited)
```

### Template System

```
Template: "order_shipped"
Title: "Your order {{order_id}} has shipped!"
Body: "Track at: {{tracking_url}}"

Rendered with user's locale and preferences
```

### User Preferences

```
Table: user_notification_settings
| user_id | channel | enabled | quiet_start | quiet_end |
|---------|---------|---------|-------------|-----------|
| 123     | push    | true    | 22:00       | 08:00     |
| 123     | sms     | false   | null        | null      |
| 123     | email   | true    | null        | null      |
```

## 5. Deep Dive: Reliability

### Delivery Tracking

```
Table: notification_log
| id      | user_id | channel | status    | sent_at    | delivered_at |
|---------|---------|---------|-----------|------------|--------------|
| abc123  | 123     | push    | delivered | 2024-01-15 | 2024-01-15   |
| def456  | 123     | email   | bounced   | 2024-01-15 | null         |
```

### Retry Strategy

```
Attempt 1: Immediate
Attempt 2: 5 minutes
Attempt 3: 30 minutes
Attempt 4: 2 hours
Max attempts: 4

Move to dead letter queue after max attempts
```

### Idempotency

```
Idempotency key: hash(user_id + type + payload + timestamp_hour)

Before sending:
1. Check if idempotency key exists in Redis
2. If exists, skip (already sent)
3. If not, send and record key (TTL 24h)
```

## 6. Channel-Specific Considerations

| Channel | Latency | Cost | Reliability | Notes |
|---------|---------|------|-------------|-------|
| Push | <1s | Free | Medium | Device must be reachable |
| SMS | <5s | $0.01/msg | High | Expensive, use sparingly |
| Email | Minutes | <$0.001/msg | High | Can be delayed, filtered |

### Push Notifications

```
Apple APNs: Device token, certificate-based auth
Google FCM: Device token, API key

Handle token refresh, invalid token cleanup
```

### SMS

```
Provider: Twilio, AWS SNS
10-digit number or short code
Carrier filtering for marketing
```

### Email

```
Provider: SendGrid, AWS SES
Handle bounces, unsubscribes
SPF, DKIM, DMARC for deliverability
```

## 7. Interview Perspective

**Common follow-ups:**
- "How to handle quiet hours?" → Check preferences, queue for later
- "What if push fails?" → Fall back to email
- "How to avoid spam?" → Per-user rate limiting, preference checks

### Cheat Sheet

```
NOTIFICATION SYSTEM

CHANNELS:
Push: Fast, free, device must be online
SMS: Reliable, expensive, use sparingly
Email: Cheap, can be delayed

ARCHITECTURE:
API → Priority Router → Channel Queues → Workers → Providers

RELIABILITY:
At-least-once delivery
Retry with exponential backoff
Dead letter queue for failures
Idempotency to prevent duplicates

USER EXPERIENCE:
Respect preferences
Enforce quiet hours
Per-user rate limiting
Batching for non-urgent

TRACKING:
Log all deliveries
Track open/click rates
Handle bounces and unsubscribes
```

---

# Case Study 4: File Storage (S3-like)

## 1. Requirements Clarification

### Functional Requirements
- Upload files (up to 5 GB)
- Download files
- Delete files
- List files in a bucket
- Metadata storage

### Non-Functional Requirements
- 99.99% availability
- 99.999999999% durability (11 nines)
- Low latency for metadata operations
- High throughput for data transfer

### Clarifying Questions
- "File size range?" (1 KB to 5 GB)
- "Access patterns?" (Write once, read many)
- "Consistency model?" (Strong for metadata, eventual for data)

## 2. Scale Estimation

```
Files: 1 billion
Average size: 1 MB
Total storage: 1 PB
Daily uploads: 10 million files

Write throughput: 10M / 86400 ≈ 120 files/second
Peak: 1000 files/second
Data write: 1000 * 1 MB = 1 GB/second
```

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                API Layer                                  │
│                     (Authentication, Routing)                             │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    ▼                                 ▼
         ┌─────────────────┐               ┌─────────────────┐
         │  Metadata       │               │  Data Service   │
         │  Service        │               │                 │
         └────────┬────────┘               └────────┬────────┘
                  │                                 │
                  ▼                                 ▼
         ┌─────────────────┐               ┌─────────────────┐
         │  Metadata DB    │               │  Data Nodes     │
         │  (Distributed)  │               │  (Blob Storage) │
         └─────────────────┘               └─────────────────┘
```

## 4. Deep Dive: Data Storage

### Data Node Architecture

```
File → Split into chunks (64 MB each)
     → Each chunk replicated 3x across data nodes
     → Metadata records chunk locations

5 GB file = 80 chunks
Each chunk on 3 different nodes (different racks)
```

### Write Path

```
1. Client initiates upload
2. Metadata service allocates chunk IDs and data nodes
3. Client uploads chunks directly to data nodes
4. Data nodes replicate to peers
5. Acknowledge to metadata service
6. Metadata service commits file record
```

### Read Path

```
1. Client requests file
2. Metadata service returns chunk locations
3. Client downloads chunks from data nodes (parallel)
4. Client assembles file
```

### Consistency

```
Metadata: Strong consistency (Paxos/Raft-based store)
Data: Eventual consistency (replication async)

Read-after-write consistency:
- Immediately consistent for the uploader
- Eventually consistent for others
```

## 5. Deep Dive: Metadata Service

### Schema

```
Table: files
| file_id | bucket | key       | size     | created_at | chunks        |
|---------|--------|-----------|----------|------------|---------------|
| abc123  | my-bkt | photo.jpg | 1048576  | 2024-01-15 | [c1, c2, c3]  |

Table: chunks
| chunk_id | file_id | sequence | nodes          | checksum |
|----------|---------|----------|----------------|----------|
| c1       | abc123  | 0        | [n1, n2, n3]   | sha256   |
| c2       | abc123  | 1        | [n4, n5, n6]   | sha256   |
```

### Metadata Database Choice

- Distributed database with strong consistency
- Options: CockroachDB, Spanner, TiDB
- Or: Single-leader with replicas (simpler)

## 6. Deep Dive: Durability

### 11 Nines Durability

```
Probability of losing all 3 replicas of a chunk:
P(lose 1 node) = 0.1% per year
P(lose 3 specific nodes) = (0.001)^3 = 10^-9

With 1 billion chunks:
Expected data loss = 1 chunk in 1000 years
```

### Strategies

- Replicate across racks (power, network failures)
- Replicate across availability zones
- Periodic integrity checks (bit rot detection)
- Background repair of under-replicated chunks

## 7. Interview Perspective

**Common follow-ups:**
- "How to handle 5 GB upload?" → Multipart upload, resume capability
- "How to ensure durability?" → 3x replication, checksums, background repair
- "Hot data optimization?" → SSD for hot, HDD for cold, CDN for frequent access

### Cheat Sheet

```
FILE STORAGE (S3-like)

ARCHITECTURE:
Metadata service: File → chunks mapping
Data nodes: Store chunks, replicate

CHUNKING:
Large files split into 64 MB chunks
Each chunk replicated 3x
Parallel upload/download

DURABILITY:
3 replicas across racks/AZs
Checksums for integrity
Background repair

CONSISTENCY:
Metadata: Strong (distributed DB)
Data: Eventual (async replication)
Read-after-write for uploader

LARGE FILE UPLOAD:
Multipart upload
Each part independent
Resume on failure
```

---

# Case Study 5: News Feed / Timeline

## 1. Requirements Clarification

### Functional Requirements
- User can post updates
- User sees feed of posts from followed users
- Feed is personalized and sorted by relevance/time
- Support likes, comments, shares

### Non-Functional Requirements
- Low latency feed retrieval (< 200ms)
- High availability (feed is core feature)
- Near real-time updates
- Scale to billions of posts

### Clarifying Questions
- "Chronological or ranked feed?" (Ranked for engagement, chronological option)
- "How many users? How many follows?" (1B users, avg 200 follows)
- "Real-time updates?" (Near real-time, 1-2 second delay OK)

## 2. Scale Estimation

```
Users: 1 billion
DAU: 500 million
Posts per user per day: 2
Total posts/day: 1 billion

Feed reads per user per day: 10
Total feed reads: 5 billion/day
Feed reads/second: 60,000

Average follows: 200
Celebrity follows: 10 million (special case)
```

## 3. High-Level Architecture

### Two Approaches

**Approach 1: Pull Model (Fan-out on Read)**
```
When user opens feed:
1. Get list of followed users
2. Query recent posts from each
3. Merge and rank
4. Return top N

Pros: Simple, storage efficient
Cons: Slow for users following many people
```

**Approach 2: Push Model (Fan-out on Write)**
```
When user posts:
1. Get list of followers
2. Write post to each follower's feed cache

When user opens feed:
1. Read from pre-computed feed cache

Pros: Fast reads
Cons: Celebrity problem (10M writes per post)
```

**Approach 3: Hybrid (Recommended)**
```
Non-celebrities: Push (fan-out on write)
Celebrities: Pull (fan-out on read)

Threshold: > 10,000 followers = celebrity
```

## 4. Deep Dive: Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Write Path                                  │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
          ┌─────────────────┐            ┌─────────────────┐
          │   Post Service  │            │  Fan-out Service │
          │   (Posts DB)    │            │  (async)         │
          └─────────────────┘            └────────┬────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │  Feed Cache     │
                                         │  (Redis)        │
                                         └─────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                               Read Path                                  │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
 ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
 │  Feed Cache     │      │  Celebrity Posts │      │  Ranking        │
 │  (Redis)        │      │  (Pull)          │      │  Service        │
 └─────────────────┘      └─────────────────┘      └─────────────────┘
                                    │
                                    ▼
                          ┌─────────────────┐
                          │  Merge & Rank   │
                          └─────────────────┘
```

## 5. Deep Dive: Feed Storage

### Feed Cache Structure (Redis)

```
Key: feed:{user_id}
Value: List of post IDs (most recent first)
Size: 500-1000 post IDs per user

LPUSH feed:123 post_abc123
LTRIM feed:123 0 999  # Keep only 1000
LRANGE feed:123 0 49  # Get top 50
```

### Posts Storage

```
Table: posts
| post_id | user_id | content | media_ids | created_at |
|---------|---------|---------|-----------|------------|
| abc123  | 456     | "Hello" | [m1, m2]  | 2024-01-15 |

Table: media
| media_id | url          | type  |
|----------|--------------|-------|
| m1       | cdn.com/m1   | image |
```

### Social Graph Storage

```
Table: follows
| follower_id | followee_id | created_at |
|-------------|-------------|------------|
| 123         | 456         | 2024-01-01 |

Index on follower_id for "who do I follow"
Index on followee_id for "who follows me"
```

## 6. Deep Dive: Ranking

### Ranking Signals

```
relevance_score = 
    recency_weight * recency_score +
    affinity_weight * affinity_score +
    engagement_weight * engagement_score +
    content_weight * content_score

Where:
- recency: How new is the post
- affinity: How often user interacts with author
- engagement: Likes, comments, shares
- content: ML-predicted relevance
```

### Ranking Pipeline

```
1. Candidate generation: Get 500 posts from cache + celebrity pulls
2. Filtering: Remove seen, blocked, low quality
3. Ranking: ML model scores each post
4. Diversity injection: Avoid same author, same type
5. Return top 50
```

## 7. Interview Perspective

**Common follow-ups:**
- "Celebrity problem?" → Hybrid: Push for normal, pull for celebrities
- "How to handle new user?" → Cold start with popular content
- "Real-time updates?" → WebSocket with new post notifications

### Cheat Sheet

```
NEWS FEED

APPROACHES:
Pull (fan-out on read): Simple, slow reads
Push (fan-out on write): Fast reads, celebrity problem
Hybrid: Push for normal, pull for celebrities (>10K followers)

WRITE PATH:
Post → Posts DB → Fan-out Service → Feed Cache (Redis)

READ PATH:
Feed Cache + Celebrity Posts → Merge → Rank → Return

FEED CACHE:
Redis lists per user
Store post IDs (not full posts)
Keep last 1000 posts

RANKING:
Recency + Affinity + Engagement + Content
ML model for scoring
Diversity injection

CELEBRITY PROBLEM:
Threshold: 10K followers
Don't fan-out, pull at read time
Merge with pre-computed feed
```

---

# Case Study 6: Payment System

## 1. Requirements Clarification

### Functional Requirements
- Process payments (credit card, bank transfer)
- Refunds
- Payment history
- Multi-currency support

### Non-Functional Requirements
- **Exactly-once processing** (critical!)
- Strong consistency
- High availability
- Audit trail
- PCI compliance

### Clarifying Questions
- "Payment methods?" (Cards, ACH, wallets)
- "Volume?" (1 million transactions/day)
- "Idempotency requirement?" (Absolutely required)

## 2. Scale Estimation

```
Transactions/day: 1 million
Peak: 50 transactions/second
Average transaction: $50
Daily volume: $50 million

Storage:
- Transaction records: 1M * 365 * 5 years * 1 KB = 1.8 TB
- Must keep forever for compliance
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           API Gateway                                    │
│                    (Authentication, Rate Limiting)                       │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────┐
                         │  Payment Service │
                         │  (Idempotency)  │
                         └────────┬────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Payment        │    │  Ledger         │    │  Notification   │
│  Processor      │    │  Service        │    │  Service        │
│  (PSP)          │    │                 │    │                 │
└─────────────────┘    └────────┬────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │  Ledger DB      │
                       │  (Immutable)    │
                       └─────────────────┘
```

## 4. Deep Dive: Idempotency

### Why Critical

```
Scenario:
1. Client sends payment request
2. Server processes, charges card
3. Network error before response
4. Client retries
5. Without idempotency: Double charge!
```

### Implementation

```
POST /payments
Idempotency-Key: client-generated-uuid

Server logic:
1. Check if idempotency key exists in DB
2. If exists: Return stored result
3. If not: 
   a. Create pending record with key
   b. Process payment
   c. Update record with result
   d. Return result
```

### Database Schema

```sql
Table: payments
| id      | idempotency_key | amount | currency | status    | created_at |
|---------|-----------------|--------|----------|-----------|------------|
| pay_123 | idem_abc        | 100.00 | USD      | succeeded | 2024-01-15 |

Table: idempotency_records
| idempotency_key | payment_id | response     | expires_at |
|-----------------|------------|--------------|------------|
| idem_abc        | pay_123    | {json...}    | 2024-01-16 |

UNIQUE constraint on idempotency_key
```

## 5. Deep Dive: Double-Entry Ledger

### Concept

Every transaction has two entries:
```
Debit: Decrease one account
Credit: Increase another account

Sum of all entries = 0 (always balanced)
```

### Example

```
Payment of $100:

Entry 1: DEBIT  Customer account    -$100
Entry 2: CREDIT Merchant account    +$100

Refund:
Entry 3: CREDIT Customer account    +$100
Entry 4: DEBIT  Merchant account    -$100
```

### Schema

```sql
Table: ledger_entries
| entry_id | transaction_id | account_id | amount  | type   | created_at |
|----------|----------------|------------|---------|--------|------------|
| e1       | tx_123         | cust_456   | -100.00 | DEBIT  | 2024-01-15 |
| e2       | tx_123         | merch_789  | +100.00 | CREDIT | 2024-01-15 |

Constraint: SUM(amount) for transaction_id = 0
```

## 6. Deep Dive: Payment State Machine

```
┌─────────┐     ┌───────────┐     ┌───────────┐
│ PENDING │────▶│ PROCESSING│────▶│ SUCCEEDED │
└─────────┘     └─────┬─────┘     └───────────┘
                      │
                      ▼
               ┌───────────┐
               │  FAILED   │
               └───────────┘

SUCCEEDED → REFUNDING → REFUNDED
```

### State Transitions

```
Only valid transitions allowed
Idempotent: If already in target state, return success
Audited: All transitions logged
```

## 7. Interview Perspective

**Common follow-ups:**
- "How to handle network failure to PSP?" → Retry with idempotency key to PSP
- "How to reconcile?" → Daily reconciliation with PSP records
- "Multi-currency?" → Store in original currency, convert on display

### Cheat Sheet

```
PAYMENT SYSTEM

CRITICAL REQUIREMENTS:
Idempotency: No double charges
Consistency: Strong, ACID transactions
Auditability: Immutable ledger

IDEMPOTENCY:
Client sends Idempotency-Key
Server stores and returns cached result
Key expires after 24h

DOUBLE-ENTRY LEDGER:
Every transaction: Debit + Credit
Sum always zero (balanced)
Immutable (no updates, only appends)

STATE MACHINE:
PENDING → PROCESSING → SUCCEEDED/FAILED
SUCCEEDED → REFUNDING → REFUNDED

RECONCILIATION:
Daily comparison with PSP records
Flag mismatches for investigation
Automated resolution where possible

SECURITY:
PCI-DSS compliance
Tokenize card numbers
Encrypt sensitive data
Audit all access
```

---

# Case Study 7: Chat / Messaging System

## 1. Requirements Clarification

### Functional Requirements
- 1:1 messaging
- Group messaging (up to 500 members)
- Online/offline status
- Read receipts
- Message history

### Non-Functional Requirements
- Real-time delivery (< 100ms within region)
- Reliable delivery (messages never lost)
- Ordered messages (within conversation)
- Scale to millions of concurrent users

### Clarifying Questions
- "Text only or media?" (Text + images for MVP)
- "End-to-end encryption?" (Yes for 1:1)
- "Message retention?" (Forever, or configurable)

## 2. Scale Estimation

```
Users: 500 million
DAU: 100 million
Concurrent: 10 million

Messages/day: 50 billion
Messages/second: 600,000

Storage:
- Average message: 200 bytes
- 50B messages * 200 bytes = 10 TB/day
```

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Clients                                     │
│                     (WebSocket connection)                               │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────┐
                         │  WebSocket      │
                         │  Gateway        │
                         └────────┬────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Chat Service   │    │  Presence       │    │  Notification   │
│                 │    │  Service        │    │  Service        │
└────────┬────────┘    └─────────────────┘    └─────────────────┘
         │
         ▼
┌─────────────────┐    ┌─────────────────┐
│  Message Store  │    │  User Session   │
│  (Cassandra)    │    │  (Redis)        │
└─────────────────┘    └─────────────────┘
```

## 4. Deep Dive: WebSocket Management

### Connection Flow

```
1. Client connects via WebSocket
2. Authenticate (JWT token)
3. Register connection in session store
4. Subscribe to user's message channel
5. Keep alive with ping/pong
```

### Session Store (Redis)

```
Key: session:{user_id}
Value: {
  "server_id": "ws-server-1",
  "connected_at": 1705312000,
  "device_id": "device_abc"
}

Multiple devices: session:{user_id}:{device_id}
```

### Message Routing

```
User A sends to User B:
1. A → WebSocket Gateway → Chat Service
2. Chat Service stores message
3. Lookup B's session in Redis
4. If online: Route to B's WebSocket Gateway → B
5. If offline: Queue for push notification
```

## 5. Deep Dive: Message Storage

### Schema (Cassandra)

```sql
-- Optimized for: Get messages in conversation
Table: messages_by_conversation
PRIMARY KEY ((conversation_id), created_at, message_id)

| conversation_id | created_at | message_id | sender_id | content | status |
|-----------------|------------|------------|-----------|---------|--------|
| conv_123        | 1705312000 | msg_abc    | user_456  | "Hello" | SENT   |

-- Optimized for: User's recent conversations
Table: conversations_by_user
PRIMARY KEY ((user_id), updated_at)

| user_id | updated_at | conversation_id | last_message |
|---------|------------|-----------------|--------------|
| 123     | 1705312000 | conv_456        | "Hello"      |
```

### Message Ordering

```
Ordering challenge:
- User A sends M1, M2
- Network delay: M2 arrives before M1

Solution:
- Client assigns sequence number
- Server orders by (client_seq, server_timestamp)
- Display in client order
```

## 6. Deep Dive: Group Messaging

### Fan-out

```
Group with 500 members:
- User posts message
- Fan-out to 500 members

Options:
1. Write 500 times (one per member) → Storage heavy
2. Members pull from group → Query heavy
3. Online members: Push. Offline: Mark for pull → Hybrid
```

### Group Metadata

```
Table: groups
| group_id | name    | owner_id | created_at |
|----------|---------|----------|------------|
| grp_123  | "Team"  | user_456 | 2024-01-01 |

Table: group_members
| group_id | user_id | joined_at  | role  |
|----------|---------|------------|-------|
| grp_123  | 456     | 2024-01-01 | ADMIN |
| grp_123  | 789     | 2024-01-02 | MEMBER|
```

## 7. Deep Dive: Presence (Online Status)

### Heartbeat Approach

```
Client sends heartbeat every 10 seconds
Server updates last_seen in Redis (TTL 30 seconds)

User is online if: last_seen > now - 30s
```

### Fan-out of Presence

```
Problem: User A has 1000 contacts. Updating all is expensive.

Solution:
- Only push presence to active contacts
- Active = Recently messaged or viewing contact list
- Others: Poll on demand
```

## 8. Interview Perspective

**Common follow-ups:**
- "How to handle message sync across devices?" → Server assigns sequence, client syncs from last known
- "Group message delivery guarantee?" → At-least-once with client dedup
- "End-to-end encryption?" → Key exchange via server, messages encrypted client-side

### Cheat Sheet

```
CHAT SYSTEM

CORE COMPONENTS:
WebSocket Gateway: Persistent connections
Chat Service: Message routing/storage
Presence Service: Online/offline status
Session Store: Connection mapping (Redis)

MESSAGE FLOW:
A → WS Gateway → Chat Service → Store
Chat Service → Lookup B's session → Route
If offline → Push notification

STORAGE:
Messages: Cassandra (partition by conversation)
Sessions: Redis (fast lookup)
Groups: PostgreSQL (metadata)

ORDERING:
Client sequence + server timestamp
Display in client order

PRESENCE:
Heartbeat every 10s
TTL 30s in Redis
Fan-out to active contacts only

GROUP MESSAGING:
Hybrid: Push to online, pull for offline
Cap group size (500) or special handling
```

---

# Phase 4 Summary

These seven case studies cover the most commonly asked system design problems. For each, you should be able to:

1. **Clarify requirements** (functional, non-functional, scale)
2. **Estimate scale** (QPS, storage, bandwidth)
3. **Draw high-level architecture** (2-3 minutes)
4. **Deep dive** on 2-3 components (interviewer may guide)
5. **Discuss trade-offs** (why this approach vs. alternatives)
6. **Handle failure scenarios** (what if X fails?)

**Interviewer Expectations by Level:**

| Level | Expectation |
|-------|-------------|
| Senior (L5) | Cover all basics, reasonable depth |
| Staff (L6) | Deep expertise in 2-3 areas, trade-off articulation |
| Principal (L7) | Challenge assumptions, discuss at scale, operational concerns |
