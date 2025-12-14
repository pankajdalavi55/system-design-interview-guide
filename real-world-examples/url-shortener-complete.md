# Complete System Design: URL Shortener (Production-Ready)

> **Complexity Level:** Beginner to Intermediate  
> **Estimated Time:** 45-60 minutes in interview  
> **Real-World Examples:** bit.ly, TinyURL, goo.gl (deprecated)

---

## Table of Contents
1. [Problem Statement](#1-problem-statement)
2. [Requirements Clarification](#2-requirements-clarification)
3. [Scale Estimation](#3-scale-estimation)
4. [High-Level Design](#4-high-level-design)
5. [Deep Dive: Core Components](#5-deep-dive-core-components)
6. [Deep Dive: Database Design](#6-deep-dive-database-design)
7. [Deep Dive: Short URL Generation](#7-deep-dive-short-url-generation)
8. [Scaling Strategies](#8-scaling-strategies)
9. [Failure Scenarios & Mitigation](#9-failure-scenarios--mitigation)
10. [Monitoring & Observability](#10-monitoring--observability)
11. [Advanced Features](#11-advanced-features)
12. [Interview Q&A](#12-interview-qa)
13. [Production Checklist](#13-production-checklist)

---

## 1. Problem Statement

**Initial Question:**  
"Design a URL shortening service like bit.ly that converts long URLs into short URLs and redirects users when they access the short URL."

**Interviewer's Perspective:**  
This is a classic problem to assess:
- Basic system design skills
- Database design
- Scale estimation
- Caching strategies
- Understanding of HTTP redirects

---

## 2. Requirements Clarification

### Interview Dialog

**Candidate:** "Before I start designing, let me clarify the requirements. I'll break this into functional and non-functional requirements."

### 2.1 Functional Requirements

**Candidate:** "For functional requirements:
1. Can users create a short URL from a long URL?
2. When someone visits the short URL, should they be redirected to the original URL?
3. Do we need custom short URLs (vanity URLs), or can they be auto-generated?
4. Do we need analytics (click tracking, geographic data)?
5. Should short URLs expire, or are they permanent?
6. Do users need to be authenticated, or is it anonymous?"

**Interviewer:** "Let's keep it simple:
- Auto-generated short URLs (no custom URLs initially)
- Redirect to original URL
- Optional: basic click tracking
- URLs can have an expiration (default 5 years)
- No authentication required for creating URLs"

**Candidate:** "Got it. So the core features are:
1. ✅ Generate short URL from long URL (POST /api/shorten)
2. ✅ Redirect short URL to original URL (GET /{shortCode})
3. ✅ Basic click tracking (count only)
4. ✅ Optional expiration (default 5 years)"

### 2.2 Non-Functional Requirements

**Candidate:** "For non-functional requirements:
1. What's the expected scale? How many URLs are we shortening per day?
2. What's the read/write ratio? Are we more read-heavy or write-heavy?
3. What's the acceptable latency for redirects?
4. What's the availability requirement?
5. Do we need to support multiple regions?"

**Interviewer:** 
- Scale: 100 million URLs created per month
- Read-heavy: 100:1 read/write ratio (10 billion redirects/month)
- Latency: <100ms for redirects (p99)
- Availability: 99.9% (three nines)
- Single region initially, multi-region is a future consideration

**Candidate:** "Perfect. So to summarize:
- **Scale:** 100M new URLs/month, 10B redirects/month
- **Read:Write ratio:** 100:1 (read-heavy)
- **Latency:** <100ms for redirects
- **Availability:** 99.9%
- **Durability:** URLs should not be lost (data persistence)
- **Uniqueness:** Each long URL can have multiple short URLs, or same short URL? Let me confirm."

**Interviewer:** "Good question. Let's say the same long URL can be shortened multiple times, creating different short URLs each time."

---

## 3. Scale Estimation

### 3.1 Traffic Estimation

**Candidate:** "Let me do some back-of-envelope calculations:

**Writes (URL Creation):**
- URLs per month: 100 million
- URLs per day: 100M / 30 ≈ 3.3 million
- URLs per second: 3.3M / 86,400 ≈ **38 URLs/second**
- Peak (3x average): **~115 URLs/second**

**Reads (Redirects):**
- Redirects per month: 10 billion
- Redirects per day: 10B / 30 ≈ 330 million
- Redirects per second: 330M / 86,400 ≈ **3,800 redirects/second**
- Peak (3x): **~11,400 redirects/second**

So we're looking at **~38 writes/sec** and **~3,800 reads/sec** on average."

### 3.2 Storage Estimation

**Candidate:** "For storage:

**Per URL Storage:**
- Short code: 7 characters (we'll discuss encoding later)
- Long URL: average 500 bytes
- Metadata: created_at, expires_at, click_count → ~50 bytes
- Total per record: ~550 bytes → round to **1 KB per URL**

**Total Storage:**
- Month 1: 100M URLs × 1 KB = 100 GB
- Year 1: 1.2 billion URLs × 1 KB = 1.2 TB
- Year 5: 6 billion URLs × 1 KB = **6 TB**

With overhead and indexing, let's estimate **10 TB for 5 years**."

### 3.3 Bandwidth Estimation

**Candidate:** "For bandwidth:

**Incoming (URL Creation):**
- 38 URLs/sec × 1 KB = **38 KB/sec** ≈ 0.3 Mbps (negligible)

**Outgoing (Redirects):**
- 3,800 redirects/sec × 1 KB (HTTP redirect response) = **3.8 MB/sec** ≈ 30 Mbps

Pretty moderate bandwidth requirements."

### 3.4 Cache Estimation

**Candidate:** "Following the 80-20 rule, 20% of URLs generate 80% of traffic.

If we cache the top 20% of URLs:
- Active URLs in a day: Let's say 10 million unique URLs generate the 330M redirects
- 20% of 10M = 2 million URLs
- Cache size: 2M × 1 KB = **2 GB**

Very manageable for Redis or Memcached."

---

## 4. High-Level Design

### 4.1 Architecture Diagram

```
┌─────────────┐
│   Client    │
│ (Browser)   │
└──────┬──────┘
       │
       │ HTTPS
       │
       ▼
┌─────────────────────────────────────┐
│      Load Balancer (AWS ALB)        │
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
┌─────────────┐  ┌─────────────┐
│   API       │  │   API       │
│  Server 1   │  │  Server 2   │
│             │  │             │
│ (Node.js/   │  │ (Node.js/   │
│  Spring)    │  │  Spring)    │
└──────┬──────┘  └──────┬──────┘
       │                │
       └───────┬────────┘
               │
       ┌───────┴────────────────────┐
       │                            │
       ▼                            ▼
┌─────────────┐            ┌──────────────┐
│    Redis    │            │  PostgreSQL  │
│   (Cache)   │            │  (Primary)   │
│             │            │              │
│  - Hot URLs │            │  - All URLs  │
│  - TTL: 1hr │            │  - Metadata  │
└─────────────┘            └──────┬───────┘
                                  │
                                  │ Replication
                                  │
                           ┌──────▼───────┐
                           │ PostgreSQL   │
                           │ (Read        │
                           │  Replicas)   │
                           └──────────────┘
```

### 4.2 API Design

**Candidate:** "Let me define the core APIs:

**1. Create Short URL**
```http
POST /api/shorten
Content-Type: application/json

Request:
{
  "longUrl": "https://www.example.com/very/long/path?with=parameters",
  "expiresAt": "2030-01-01T00:00:00Z"  // Optional
}

Response (201 Created):
{
  "shortUrl": "https://short.ly/abc1234",
  "shortCode": "abc1234",
  "longUrl": "https://www.example.com/very/long/path?with=parameters",
  "createdAt": "2025-12-14T10:00:00Z",
  "expiresAt": "2030-12-14T10:00:00Z"
}
```

**2. Redirect**
```http
GET /{shortCode}

Response (301 Moved Permanently):
Location: https://www.example.com/very/long/path?with=parameters
```

**3. Get Analytics (Optional)**
```http
GET /api/analytics/{shortCode}

Response:
{
  "shortCode": "abc1234",
  "clicks": 12543,
  "createdAt": "2025-12-14T10:00:00Z"
}
```
"

### 4.3 Data Flow

**Candidate:** "Let me walk through the two main flows:

**Flow 1: Create Short URL**
1. Client sends POST /api/shorten with long URL
2. API server validates the long URL
3. Generate unique short code (7 characters)
4. Store mapping in PostgreSQL: {shortCode → longUrl, metadata}
5. Return short URL to client

**Flow 2: Redirect**
1. Client requests GET /abc1234
2. API server checks Redis cache for 'abc1234'
3. If cache hit → return 301 redirect immediately
4. If cache miss → query PostgreSQL
5. Store result in Redis (TTL: 1 hour)
6. Increment click counter (async)
7. Return 301 redirect to client

This handles the basic flow. Should I go deeper into any component?"

---

## 5. Deep Dive: Core Components

### 5.1 API Server

**Technology Choice:**

**Candidate:** "For the API server, I'd recommend:

**Option 1: Node.js + Express**
- Pros: Non-blocking I/O, great for I/O-heavy workloads, large ecosystem
- Cons: Single-threaded (but mitigated by clustering)

**Option 2: Java + Spring Boot**
- Pros: Mature, excellent for enterprise, strong typing
- Cons: Higher memory footprint, slower startup

**Option 3: Go**
- Pros: Fast, concurrent, low memory, great for microservices
- Cons: Smaller ecosystem, less familiar for many teams

**My choice:** Node.js for this use case because:
- I/O-heavy (database + cache lookups)
- Simple business logic
- Fast development
- Handles 10k+ requests/sec easily with clustering"

### 5.2 Load Balancer

**Candidate:** "For load balancing:

**AWS Application Load Balancer (ALB):**
- Layer 7 (HTTP/HTTPS)
- SSL termination
- Health checks
- Auto-scaling integration
- Sticky sessions not needed (stateless API)

**Configuration:**
- Round-robin algorithm (simple, effective for stateless)
- Health check: GET /health every 30 seconds
- Deregister unhealthy instances after 2 failed checks"

### 5.3 Why This Architecture?

**Candidate:** "This architecture prioritizes:
1. **Read performance:** Redis caching for hot URLs (80% hit rate expected)
2. **Availability:** Multiple API servers, database replication
3. **Scalability:** Stateless design, horizontal scaling
4. **Simplicity:** Avoid premature complexity

As we grow, we can:
- Add more API servers (horizontal scaling)
- Add read replicas for PostgreSQL
- Shard the database by short code range
- Add CDN for global distribution"

---

## 6. Deep Dive: Database Design

### 6.1 Schema Design

**Candidate:** "Let me design the database schema.

**Option 1: SQL (PostgreSQL)**

```sql
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0,
    
    INDEX idx_short_code (short_code),
    INDEX idx_created_at (created_at)
);
```

**Option 2: NoSQL (DynamoDB/Cassandra)**

```
Partition Key: short_code
Attributes: long_url, created_at, expires_at, click_count
```

**My choice:** PostgreSQL because:
1. ✅ Simple key-value lookups (our primary access pattern)
2. ✅ ACID guarantees (no duplicate short codes)
3. ✅ Mature, well-understood
4. ✅ Easy analytics queries
5. ✅ Read replicas for scaling reads

NoSQL would work too, but PostgreSQL is simpler for this scale."

### 6.2 Why This Schema?

**Candidate:** "Design decisions:

**`short_code` as VARCHAR(10):**
- Base62 encoding (a-z, A-Z, 0-9) = 62 characters
- 7 characters = 62^7 = 3.5 trillion combinations
- More than enough for 6 billion URLs over 5 years

**`long_url` as TEXT:**
- URLs can be up to 2,048 characters (some browsers)
- TEXT type handles variable length efficiently

**`click_count` as BIGINT:**
- Supports up to 9 quintillion clicks
- Viral URLs might get billions of clicks

**Indexes:**
- `idx_short_code`: Primary lookup (most common query)
- `idx_created_at`: For analytics, cleanup jobs"

### 6.3 Alternative: Separate Analytics Table

**Candidate:** "For better performance, we could separate analytics:

```sql
CREATE TABLE url_metadata (
    short_code VARCHAR(10) PRIMARY KEY,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP
);

CREATE TABLE url_analytics (
    short_code VARCHAR(10) REFERENCES url_metadata(short_code),
    click_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user_agent TEXT,
    ip_address INET,
    country VARCHAR(2)
);
```

**Trade-off:**
- Pro: No lock contention on click counter updates
- Pro: Rich analytics (per-click data)
- Con: Higher storage (10x increase)
- Con: More complex queries

For MVP, single table is fine. As we scale, migrate to separate tables."

---

## 7. Deep Dive: Short URL Generation

### 7.1 Encoding Strategy

**Candidate:** "This is the most interesting technical challenge. Let me discuss options.

**Option 1: Hash-based (MD5/SHA256)**

```python
import hashlib

def generate_short_code(long_url):
    hash_value = hashlib.md5(long_url.encode()).hexdigest()
    return base62_encode(int(hash_value[:8], 16))[:7]
```

**Pros:**
- Deterministic (same URL → same short code)
- Fast to compute

**Cons:**
- Collision risk (birthday paradox)
- Need collision detection and retry
- Can't guarantee uniqueness without DB check

**Option 2: Counter-based (Auto-increment)**

```python
def generate_short_code(counter):
    return base62_encode(counter)  # e.g., 12345 → "dnh"
```

**Pros:**
- Guaranteed unique
- Very fast
- Predictable length growth

**Cons:**
- Sequential (predictable, security concern)
- Requires distributed counter (single point of contention)
- Short codes reveal creation order

**Option 3: Random Generation**

```python
import random
import string

def generate_short_code():
    chars = string.ascii_letters + string.digits  # a-z, A-Z, 0-9
    return ''.join(random.choices(chars, k=7))
```

**Pros:**
- Simple
- Non-sequential
- No external dependencies

**Cons:**
- Collision risk (needs DB uniqueness check)
- Retries on collision

**Option 4: Snowflake ID (Twitter's approach)**

```
64-bit ID:
- 41 bits: timestamp (milliseconds since epoch)
- 10 bits: machine ID
- 12 bits: sequence number

Then base62 encode to 7-8 characters
```

**Pros:**
- Time-ordered (helps with DB indexing)
- Guaranteed unique across distributed system
- No central coordination

**Cons:**
- Requires clock synchronization
- Reveals timestamp (minor privacy concern)

**My recommendation: Random + Uniqueness Check**

```javascript
// Node.js implementation
const crypto = require('crypto');

function generateShortCode() {
    const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    let result = '';
    const randomBytes = crypto.randomBytes(7);
    
    for (let i = 0; i < 7; i++) {
        result += chars[randomBytes[i] % 62];
    }
    
    return result;
}

async function createShortUrl(longUrl) {
    let shortCode;
    let inserted = false;
    let attempts = 0;
    
    while (!inserted && attempts < 5) {
        shortCode = generateShortCode();
        
        try {
            await db.query(
                'INSERT INTO urls (short_code, long_url) VALUES ($1, $2)',
                [shortCode, longUrl]
            );
            inserted = true;
        } catch (err) {
            if (err.code === '23505') {  // Unique violation
                attempts++;
                continue;
            }
            throw err;
        }
    }
    
    if (!inserted) {
        throw new Error('Failed to generate unique short code');
    }
    
    return shortCode;
}
```

**Why this works:**
- 62^7 = 3.5 trillion combinations
- With 6 billion URLs, collision probability ≈ 0.0017% per attempt
- Average retries: ~1.00001 (effectively zero)
- Cryptographically random (secure)
- No distributed coordination needed"

### 7.2 Base62 Encoding Explanation

**Candidate:** "Base62 uses: a-z (26), A-Z (26), 0-9 (10) = 62 characters

```javascript
function base62Encode(num) {
    const chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    let result = '';
    
    while (num > 0) {
        result = chars[num % 62] + result;
        num = Math.floor(num / 62);
    }
    
    return result || '0';
}

// Examples:
base62Encode(0) → '0'
base62Encode(61) → 'Z'
base62Encode(62) → '10'
base62Encode(125) → '21'
base62Encode(12345) → '3D7'
```

**Length estimation:**
- 62^5 = 916 million (5 chars)
- 62^6 = 56 billion (6 chars)
- 62^7 = 3.5 trillion (7 chars)

For 6 billion URLs, 7 characters is perfect."

---

## 8. Scaling Strategies

### 8.1 Current Bottlenecks

**Candidate:** "At our scale (3,800 reads/sec, 38 writes/sec), bottlenecks are:

1. **Database Read Load**
   - Even with caching, 20% of reads (760/sec) hit the database
   - Single PostgreSQL can handle ~10k simple reads/sec
   - We're fine, but limited headroom

2. **Database Write Load**
   - 38 writes/sec is very light
   - PostgreSQL can handle 1k-5k writes/sec
   - Not a concern yet

3. **Cache**
   - Redis can handle 100k+ ops/sec
   - Not a bottleneck

4. **API Servers**
   - Each server handles ~1k req/sec
   - With 4 servers, we have plenty of headroom"

### 8.2 Scaling to 10x (38k reads/sec)

**Candidate:** "If traffic grows 10x:

**Step 1: Add Read Replicas (handles 10-50x growth)**
```
Primary (writes) → 380 writes/sec ✓
Read Replica 1 → 19k reads/sec ✓
Read Replica 2 → 19k reads/sec ✓
```

Application routes:
- Writes → Primary
- Reads → Round-robin across replicas

**Step 2: Increase Cache Size**
- From 2GB to 20GB (cache top 20% of 100M active URLs)
- Hit rate: 80% → 90% with larger cache
- Database reads: 38k × 10% = 3.8k reads/sec (easily handled)

**Step 3: Horizontal Scaling of API Servers**
- From 4 to 40 servers (auto-scaling)
- Each handles 1k req/sec → total capacity 40k req/sec"

### 8.3 Scaling to 100x (380k reads/sec)

**Candidate:** "At 100x scale, we need more aggressive strategies:

**Option 1: Database Sharding (by short_code range)**

```
Shard 1: short_codes 0000000 - 3ffffff (a-g)
Shard 2: short_codes 4000000 - 7ffffff (h-o)
Shard 3: short_codes 8000000 - bffffff (p-z, A-G)
Shard 4: short_codes c000000 - fffffff (H-Z, 0-9)
```

Routing logic:
```javascript
function getShardId(shortCode) {
    const firstChar = shortCode[0].toLowerCase();
    if (firstChar >= '0' && firstChar <= '9') return 4;
    if (firstChar >= 'a' && firstChar <= 'g') return 1;
    if (firstChar >= 'h' && firstChar <= 'o') return 2;
    return 3;
}
```

Each shard handles 95k reads/sec (within PostgreSQL capacity).

**Option 2: Multi-Region Deployment**

```
US Region:
  - Load Balancer
  - API Servers
  - Redis
  - PostgreSQL (replicated from primary)

EU Region:
  - Load Balancer
  - API Servers
  - Redis
  - PostgreSQL (replicated from primary)

Asia Region:
  - Similar setup
```

DNS routes users to nearest region. Writes go to primary region; reads are local.

**Option 3: CDN for Popular URLs**

Use CloudFront/Cloudflare to cache HTTP redirects:
```
GET /abc1234
→ CloudFront edge location
→ Cache hit: Return 301 redirect (0ms latency)
→ Cache miss: Origin server → cache for 1 hour
```

For viral URLs (millions of hits), CDN handles 99% of traffic."

### 8.4 Scaling Writes (if needed)

**Candidate:** "If we hit write bottlenecks (unlikely):

**Option 1: Write Buffering**
```
API Server → Kafka/RabbitMQ → Consumer → Database
```
Decouple write spikes from database.

**Option 2: Batch Inserts**
```javascript
// Instead of inserting one-by-one
await db.query('INSERT INTO urls VALUES ($1, $2)', [shortCode, longUrl]);

// Batch 100 at a time
await db.query('INSERT INTO urls VALUES ($1, $2), ($3, $4), ...', [...]);
```

**Option 3: Database Sharding**
Same as read scaling—distribute writes across shards."

---

## 9. Failure Scenarios & Mitigation

### 9.1 Database Failure

**Scenario:** Primary PostgreSQL database goes down.

**Impact:**
- Cannot create new short URLs (writes fail)
- Redirects still work (read replicas available)
- Analytics writes fail

**Mitigation:**

**Immediate (automated):**
1. Promote read replica to primary (automatic failover with Patroni/pgpool)
2. Update DNS/connection string to point to new primary
3. RTO: 30-60 seconds

**Write buffering (graceful degradation):**
```javascript
app.post('/api/shorten', async (req, res) => {
    try {
        const shortCode = await createShortUrl(req.body.longUrl);
        res.json({ shortCode });
    } catch (err) {
        if (err.code === 'DB_UNAVAILABLE') {
            // Buffer write to Kafka
            await kafka.publish('url-creation-queue', req.body);
            res.status(202).json({ message: 'Request queued, retry in 5 mins' });
        }
    }
});
```

**Long-term:**
- Multi-master replication (PostgreSQL + Patroni)
- Cross-region backup primary"

### 9.2 Redis Cache Failure

**Scenario:** Redis cluster goes down.

**Impact:**
- All reads hit the database (3,800 req/sec)
- Increased latency (50ms → 200ms)
- Database load spike

**Mitigation:**

**Immediate:**
1. Database read replicas absorb the load (can handle 10k req/sec)
2. Auto-scaling adds more API servers to handle increased DB query time
3. No downtime, just degraded performance

**Application-level:**
```javascript
async function getOriginalUrl(shortCode) {
    try {
        // Try cache first
        const cached = await redis.get(`url:${shortCode}`);
        if (cached) return cached;
    } catch (err) {
        // Redis down, skip gracefully
        logger.warn('Redis unavailable, falling back to DB');
    }
    
    // Fallback to database
    const result = await db.query('SELECT long_url FROM urls WHERE short_code = $1', [shortCode]);
    return result.rows[0]?.long_url;
}
```

**Recovery:**
- Restart Redis cluster
- Warm up cache with top URLs
- Gradual return to normal latency"

### 9.3 API Server Failure

**Scenario:** API server crashes.

**Impact:**
- Load balancer detects failure (health check)
- Redirects traffic to healthy servers
- No user impact (stateless design)

**Mitigation:**
- Health checks every 10 seconds
- Auto-scaling launches replacement instance
- RTO: 0 seconds (transparent)"

### 9.4 Short Code Collision

**Scenario:** Random generation produces a duplicate short code.

**Impact:**
- Database INSERT fails (unique constraint violation)
- Retry logic generates new code
- Slight delay (<10ms)

**Mitigation:**
```javascript
// Handled in generation logic (shown earlier)
// Max 5 retries before error
// Probability of 5 collisions: (0.0017%)^5 ≈ 0 (never happens in practice)
```

### 9.5 Expired URL Accessed

**Scenario:** User clicks a short URL that has expired.

**Impact:**
- Bad user experience (broken link)

**Mitigation:**

```javascript
async function redirect(shortCode) {
    const url = await getUrl(shortCode);
    
    if (!url) {
        return res.status(404).render('not-found');
    }
    
    if (url.expires_at && new Date(url.expires_at) < new Date()) {
        return res.status(410).render('expired', { 
            message: 'This short URL has expired' 
        });
    }
    
    // Async analytics update
    incrementClickCount(shortCode);
    
    return res.redirect(301, url.long_url);
}
```

**Cleanup Job:**
```sql
-- Daily cron job to delete expired URLs
DELETE FROM urls WHERE expires_at < NOW() - INTERVAL '30 days';
```

### 9.6 DDoS Attack

**Scenario:** Attacker floods the service with requests.

**Impact:**
- Increased costs
- Potential service degradation

**Mitigation:**

**Layer 1: CDN/DDoS Protection (Cloudflare)**
- 10 Tbps DDoS mitigation
- Automatic blocking of malicious IPs
- Rate limiting at edge

**Layer 2: API Rate Limiting**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,  // 100 requests per 15 min per IP
    message: 'Too many requests, please try again later'
});

app.use('/api/shorten', limiter);
```

**Layer 3: Authentication for Creation**
- Require API key for URL creation
- Authenticated users get higher limits"

---

## 10. Monitoring & Observability

### 10.1 Key Metrics to Track

**Candidate:** "For production, I'd monitor:

**Application Metrics (RED method):**
1. **Rate:** Requests per second (create, redirect)
2. **Errors:** Error rate (4xx, 5xx)
3. **Duration:** Latency (p50, p95, p99)

**Business Metrics:**
- URLs created per minute
- Redirects per minute
- Cache hit rate
- Top 10 most clicked short URLs

**Infrastructure Metrics (USE method):**
1. **Utilization:** CPU, memory, disk
2. **Saturation:** Queue depth, connection pool
3. **Errors:** Failed health checks, database errors

**Example Dashboard (Grafana):**
```
Row 1: Traffic
- [Graph] Requests/sec (create vs redirect)
- [Gauge] Current RPS
- [Graph] Error rate (%)

Row 2: Latency
- [Heatmap] Latency distribution (p50/p95/p99)
- [Graph] Database query time
- [Graph] Cache response time

Row 3: Cache
- [Gauge] Cache hit rate (%)
- [Graph] Cache evictions/sec
- [Graph] Cache memory usage

Row 4: Database
- [Graph] Connections (active/idle)
- [Graph] Query rate (read/write)
- [Graph] Replication lag
```
"

### 10.2 Alerting Rules

**Candidate:** "Critical alerts (page on-call):

```yaml
# High error rate
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
for: 2m
severity: critical
message: "Error rate above 5% for 2 minutes"

# High latency
alert: HighLatency
expr: histogram_quantile(0.99, http_request_duration_seconds) > 0.5
for: 5m
severity: critical
message: "p99 latency above 500ms"

# Database down
alert: DatabaseDown
expr: up{job="postgresql"} == 0
for: 1m
severity: critical
message: "PostgreSQL is down"

# Cache hit rate dropped
alert: LowCacheHitRate
expr: cache_hit_rate < 0.7
for: 10m
severity: warning
message: "Cache hit rate below 70%"
```
"

### 10.3 Logging Strategy

**Candidate:** "Structured logging with ELK stack (Elasticsearch, Logstash, Kibana):

```javascript
const logger = require('winston');

logger.info('URL created', {
    shortCode: 'abc1234',
    longUrl: 'https://example.com/...',
    ip: req.ip,
    userAgent: req.headers['user-agent'],
    duration: 45,  // ms
});

logger.info('Redirect', {
    shortCode: 'abc1234',
    cacheHit: true,
    duration: 12,  // ms
    ip: req.ip,
});

logger.error('Database error', {
    error: err.message,
    stack: err.stack,
    query: 'INSERT INTO urls...',
});
```

**Log retention:**
- INFO logs: 7 days
- ERROR logs: 30 days
- Archive to S3: 1 year"

### 10.4 Distributed Tracing

**Candidate:** "Use OpenTelemetry + Jaeger for request tracing:

```
Request ID: abc-123-def-456

Span 1: HTTP GET /abc1234 (120ms)
  Span 2: Redis GET url:abc1234 (5ms) [cache miss]
  Span 3: PostgreSQL query (80ms)
  Span 4: Redis SET url:abc1234 (3ms)
  Span 5: Async click counter update (15ms)
```

This helps debug slow requests."

---

## 11. Advanced Features

### 11.1 Custom Short URLs (Vanity URLs)

**Requirement:** Allow users to create custom short URLs (e.g., `short.ly/promo2024`)

**Implementation:**

```javascript
app.post('/api/shorten/custom', async (req, res) => {
    const { longUrl, customCode } = req.body;
    
    // Validation
    if (!isValidCustomCode(customCode)) {
        return res.status(400).json({ error: 'Invalid custom code' });
    }
    
    // Check availability
    const exists = await db.query('SELECT 1 FROM urls WHERE short_code = $1', [customCode]);
    if (exists.rows.length > 0) {
        return res.status(409).json({ error: 'Custom code already taken' });
    }
    
    // Reserve code
    await db.query('INSERT INTO urls (short_code, long_url) VALUES ($1, $2)', [customCode, longUrl]);
    
    res.json({ shortUrl: `https://short.ly/${customCode}` });
});

function isValidCustomCode(code) {
    return /^[a-zA-Z0-9_-]{4,20}$/.test(code);  // 4-20 alphanumeric chars
}
```

**Challenges:**
- Collision with auto-generated codes (reserve namespaces: auto-gen uses only 7 chars, custom allows 4-20)
- Abuse prevention (blacklist profanity, reserved words)

### 11.2 Analytics Dashboard

**Requirement:** Show click statistics per short URL

**Database Schema:**

```sql
CREATE TABLE url_clicks (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL,
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    user_agent TEXT,
    referer TEXT,
    country VARCHAR(2),
    
    INDEX idx_short_code_time (short_code, clicked_at)
);
```

**API:**

```http
GET /api/analytics/{shortCode}?days=30

Response:
{
  "shortCode": "abc1234",
  "totalClicks": 125403,
  "uniqueIPs": 45230,
  "clicksByDate": [
    { "date": "2025-12-01", "clicks": 4523 },
    { "date": "2025-12-02", "clicks": 5123 }
  ],
  "topCountries": [
    { "country": "US", "clicks": 50231 },
    { "country": "IN", "clicks": 30120 }
  ],
  "topReferers": [
    { "referer": "twitter.com", "clicks": 20450 },
    { "referer": "facebook.com", "clicks": 15230 }
  ]
}
```

**Scaling:**
- Use time-series database (InfluxDB, TimescaleDB) for analytics
- Pre-aggregate daily stats with cron job
- Separate analytics DB from operational DB

### 11.3 QR Code Generation

**Requirement:** Generate QR codes for short URLs

**Implementation:**

```javascript
const QRCode = require('qrcode');

app.get('/api/qr/:shortCode', async (req, res) => {
    const shortUrl = `https://short.ly/${req.params.shortCode}`;
    
    const qrCodeDataUrl = await QRCode.toDataURL(shortUrl, {
        width: 300,
        margin: 2,
        color: {
            dark: '#000000',
            light: '#FFFFFF'
        }
    });
    
    res.json({ qrCode: qrCodeDataUrl });
});
```

**Optimization:**
- Cache generated QR codes in Redis
- CDN for QR code images

### 11.4 Link Preview (Open Graph)

**Requirement:** Show preview when sharing on social media

**Implementation:**

```html
<!-- Dynamically generated based on short code -->
<meta property="og:title" content="Original Page Title" />
<meta property="og:description" content="Original Page Description" />
<meta property="og:image" content="https://example.com/image.jpg" />
<meta property="og:url" content="https://short.ly/abc1234" />
```

**Backend:**
```javascript
app.get('/:shortCode', async (req, res) => {
    const url = await getUrl(req.params.shortCode);
    
    // If user-agent is a social media bot, return HTML with OG tags
    if (isSocialBot(req.headers['user-agent'])) {
        const metadata = await fetchOGMetadata(url.long_url);
        return res.render('preview', { metadata, shortCode: req.params.shortCode });
    }
    
    // Otherwise, redirect
    return res.redirect(301, url.long_url);
});
```

### 11.5 Expiration and Auto-Delete

**Implementation:**

```sql
-- Cron job runs daily
DELETE FROM urls WHERE expires_at < NOW() - INTERVAL '30 days';
```

**Or use PostgreSQL partitioning:**
```sql
CREATE TABLE urls (
    -- ... columns
) PARTITION BY RANGE (created_at);

CREATE TABLE urls_2025_12 PARTITION OF urls
    FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');
    
-- Drop old partitions
DROP TABLE urls_2020_01;
```

---

## 12. Interview Q&A

### Q1: Why 301 vs 302 redirect?

**Answer:**
- **301 (Permanent):** Browser caches the redirect. Future requests don't hit our server (good for performance, bad for analytics).
- **302 (Temporary):** Browser doesn't cache. Every click hits our server (accurate analytics, slightly higher latency).

**My choice:** 
- **302 for analytics-critical URLs** (track every click)
- **301 for performance-critical URLs** (reduce server load)
- Make it configurable per URL

### Q2: How do you prevent abuse (spam, malicious URLs)?

**Answer:**

1. **Rate Limiting:** 100 URLs per IP per hour
2. **Blacklist Domains:** Block known spam/phishing sites
3. **URL Validation:** Check that URL is reachable (HEAD request)
4. **Captcha:** For anonymous users
5. **Authentication:** Require API key for high-volume users
6. **Content Scanning:** Integrate with Google Safe Browsing API
7. **Expiration:** Auto-expire URLs after 5 years

### Q3: How do you handle the same long URL being shortened multiple times?

**Answer:**

**Option 1: Always create new short code**
- Pros: Allows different expiration dates, separate analytics
- Cons: Wastes storage, more short codes

**Option 2: Return existing short code (deduplication)**
```javascript
async function createShortUrl(longUrl) {
    // Check if long URL already exists
    const existing = await db.query('SELECT short_code FROM urls WHERE long_url = $1', [longUrl]);
    if (existing.rows.length > 0) {
        return existing.rows[0].short_code;
    }
    
    // Create new
    const shortCode = generateShortCode();
    await db.query('INSERT INTO urls (short_code, long_url) VALUES ($1, $2)', [shortCode, longUrl]);
    return shortCode;
}
```

**My choice:** Option 1 for simplicity (users expect new URLs each time).

### Q4: How would you handle a viral URL (1 million clicks/min)?

**Answer:**

1. **CDN Caching:** CloudFront caches 301 redirect at edge (handles 99% of traffic)
2. **Redis Caching:** In-memory cache handles remaining 1%
3. **Database Read Replicas:** Fallback if cache misses
4. **Auto-Scaling:** API servers scale to handle spike

Result: System handles 1M clicks/min with <10ms latency globally.

### Q5: What if you need to update the long URL?

**Answer:**

**Current design:** No update API (URLs are immutable)

**If updates needed:**
```javascript
app.put('/api/update/:shortCode', async (req, res) => {
    const { longUrl } = req.body;
    
    // Invalidate cache
    await redis.del(`url:${req.params.shortCode}`);
    
    // Update database
    await db.query('UPDATE urls SET long_url = $1 WHERE short_code = $2', 
                   [longUrl, req.params.shortCode]);
    
    res.json({ success: true });
});
```

**Challenges:**
- Cache invalidation across distributed Redis
- CDN cache invalidation (CloudFront invalidation API)
- May confuse users (URL changes behavior)

**Alternative:** Create new short URL instead of updating.

---

## 13. Production Checklist

### 13.1 Pre-Launch Checklist

- [ ] **Load Testing:** Test with 10k req/sec using k6 or JMeter
- [ ] **Failure Testing:** Kill database, Redis, API servers—system recovers gracefully
- [ ] **Security Audit:** 
  - [ ] HTTPS only
  - [ ] SQL injection prevention (parameterized queries)
  - [ ] Rate limiting enabled
  - [ ] CORS configured
- [ ] **Monitoring:** Dashboards, alerts, on-call rotation
- [ ] **Backup Strategy:** Daily PostgreSQL backups to S3
- [ ] **Disaster Recovery:** Runbook for database failover
- [ ] **Documentation:** API docs, architecture diagrams, runbooks
- [ ] **Legal:** Terms of service, privacy policy (GDPR compliance)

### 13.2 Day-1 Operations

- [ ] Monitor error rate (<1%)
- [ ] Monitor latency (p99 <100ms)
- [ ] Check cache hit rate (>80%)
- [ ] Verify backups completed
- [ ] Review top errors in logs

### 13.3 Week-1 Optimization

- [ ] Analyze slow queries (use pg_stat_statements)
- [ ] Tune cache TTLs based on hit rate
- [ ] Add indexes for common queries
- [ ] Optimize auto-scaling thresholds

### 13.4 Month-1 Scaling

- [ ] Review capacity (database, cache, API servers)
- [ ] Plan for 3x growth
- [ ] Cost optimization (reserved instances, caching)
- [ ] User feedback and feature requests

---

## Summary: Key Takeaways

### Technical Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Database** | PostgreSQL | ACID, simple key-value, easy analytics |
| **Cache** | Redis | Fast, TTL support, distributed |
| **API** | Node.js + Express | I/O-heavy, simple logic, fast dev |
| **Load Balancer** | AWS ALB | Layer 7, health checks, SSL termination |
| **Short Code** | Random + uniqueness check | No coordination, collision-resistant |
| **Redirect** | 302 (or configurable) | Accurate analytics vs. performance |

### Scalability Path

1. **Current (3.8k reads/sec):** Single database + Redis cache
2. **10x (38k reads/sec):** Add read replicas, larger cache
3. **100x (380k reads/sec):** Shard database, multi-region, CDN

### Interview Performance Tips

1. ✅ Clarify requirements before designing (5 minutes)
2. ✅ Do back-of-envelope calculations (show your math)
3. ✅ Start with simple design, then discuss scaling
4. ✅ Deep dive into 2-3 components (short code generation, database schema)
5. ✅ Discuss failure scenarios (database down, cache miss)
6. ✅ Mention monitoring and alerting
7. ✅ Handle follow-up questions (301 vs 302, deduplication, abuse prevention)

---

**End of URL Shortener Complete Design**  
[← Back to Main Index](../README.md)
