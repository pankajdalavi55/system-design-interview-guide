# Phase 5: Interview Execution Strategy

This phase covers the meta-skills that determine interview success. Technical knowledge is necessary but not sufficient—you need to demonstrate it effectively in a 45-60 minute conversation. This section explains how to structure your approach, manage time, and signal seniority.

---

# Section 1: How Interviewers Evaluate Candidates

## 1.1 The Evaluation Framework

### What Interviewers Look For

| Dimension | What It Means | How It's Assessed |
|-----------|---------------|-------------------|
| **Problem Solving** | Can you break down ambiguous problems? | Requirements gathering, approach |
| **Technical Depth** | Do you understand the underlying concepts? | Deep dives, trade-off discussions |
| **Technical Breadth** | Can you cover all system components? | Architecture completeness |
| **Communication** | Can you explain clearly? | Clarity, structure, diagrams |
| **Leadership/Seniority** | Do you demonstrate senior-level thinking? | Proactive trade-offs, questioning assumptions |

### Level Expectations

| Level | Expectation |
|-------|-------------|
| **Senior (L5)** | Drive the interview with occasional guidance. Cover all major components. Reasonable trade-offs. |
| **Staff (L6)** | Drive independently. Deep expertise in chosen areas. Challenge assumptions. Consider operational aspects. |
| **Principal (L7)** | Lead discussion like a peer. Anticipate interviewer's questions. Discuss organizational/cost implications. |

### Scoring Rubric (Typical)

| Score | Description |
|-------|-------------|
| 1 - Strong No Hire | Fundamental gaps, poor communication |
| 2 - Lean No Hire | Missing key areas, needed heavy guidance |
| 3 - Lean Hire | Covered basics, some gaps, acceptable trade-offs |
| 4 - Hire | Strong performance, good depth, clear communication |
| 5 - Strong Hire | Exceptional, taught interviewer something, obvious hire |

---

## 1.2 Common Failure Modes

### Failure Mode 1: Jumping to Solutions
**What happens:** Candidate immediately starts drawing databases and services without understanding the problem.

**Why it fails:** 
- May design the wrong system
- Signals junior thinking
- Misses important constraints

**How to avoid:** Spend 3-5 minutes on requirements before architecture.

---

### Failure Mode 2: Shallow Coverage
**What happens:** Candidate mentions many components but doesn't explain any deeply.

**Why it fails:**
- Sounds like reading a textbook
- Can't answer follow-ups
- Doesn't demonstrate real understanding

**How to avoid:** Cover breadth in 10 minutes, then go deep on 2-3 components.

---

### Failure Mode 3: Over-Engineering
**What happens:** Candidate designs for 1 billion users when the problem has 10,000.

**Why it fails:**
- Wastes time on irrelevant complexity
- Shows poor judgment
- Ignores actual constraints

**How to avoid:** Always ask about scale. Design for current + 10x growth.

---

### Failure Mode 4: Missing Trade-offs
**What happens:** Candidate makes decisions without explaining alternatives.

**Why it fails:**
- Real engineering is about trade-offs
- Sounds like memorization
- Misses seniority signal

**How to avoid:** For every decision, mention at least one alternative and why you didn't choose it.

---

### Failure Mode 5: Silent Thinking
**What happens:** Candidate goes quiet for extended periods.

**Why it fails:**
- Interviewer can't assess your thinking
- Feels awkward
- May seem stuck

**How to avoid:** Think out loud. "I'm considering X vs Y because..."

---

### Failure Mode 6: Not Driving the Discussion
**What happens:** Candidate waits for interviewer to ask what's next.

**Why it fails:**
- Signals junior behavior
- Wastes time
- Shows lack of ownership

**How to avoid:** Always have a next step. "Now I'll discuss the database schema, unless you'd like me to focus elsewhere."

---

## 1.3 Seniority Signals

### What Looks Junior

| Behavior | Why It's a Problem |
|----------|-------------------|
| "Let me add a cache here" (without justification) | No reasoning |
| "We'll use MongoDB" (without explaining why) | Tool namedropping |
| "The system should be scalable" (without specifics) | Buzzwords |
| Waiting for guidance on what to cover next | Lack of ownership |
| No mention of failure scenarios | Incomplete thinking |

### What Looks Senior

| Behavior | Why It's Impressive |
|----------|---------------------|
| "Before designing, let me clarify the read/write ratio..." | Systematic approach |
| "I chose PostgreSQL over DynamoDB because we need ACID for payments..." | Reasoned decisions |
| "Let me discuss what happens when the payment service fails..." | Failure-first thinking |
| "We could use eventual consistency here, but given the user experience requirement..." | Trade-off articulation |
| "At this scale, we might consider X, but let me start with the simpler Y..." | Right-sizing |

---

# Section 2: The 45-60 Minute Interview Flow

## 2.1 Time Allocation

### Recommended Time Split (45 minutes)

| Phase | Time | Activity |
|-------|------|----------|
| **Requirements** | 5 min | Clarify functional/non-functional requirements |
| **Scale Estimation** | 3 min | Back-of-envelope calculations |
| **High-Level Design** | 10 min | Draw major components and flows |
| **Deep Dives** | 20 min | 2-3 components in detail |
| **Wrap-up** | 5 min | Trade-offs, improvements, questions |
| **Buffer** | 2 min | For interviewer questions throughout |

### For 60-Minute Interviews

| Phase | Time |
|-------|------|
| Requirements | 5 min |
| Scale Estimation | 5 min |
| High-Level Design | 15 min |
| Deep Dives | 25 min |
| Wrap-up | 8 min |
| Buffer | 2 min |

---

## 2.2 Phase 1: Requirements Clarification (5 minutes)

### What to Clarify

**Functional Requirements:**
- Core features (what must the system do?)
- Nice-to-have features (what's optional?)
- User types (who uses the system?)

**Non-Functional Requirements:**
- Scale (users, requests, data size)
- Latency (acceptable response times)
- Availability (uptime requirements)
- Consistency (strong or eventual?)

### Example Questions

```
"What's the expected scale? Users, daily active users, data volume?"
"What's the read/write ratio for this system?"
"What's the acceptable latency for the core operations?"
"Is strong consistency required, or is eventual consistency acceptable?"
"Are there any geographic requirements? Multi-region?"
"What's the expected growth rate?"
"Are there any cost constraints?"
```

### Example Dialog

**Interviewer:** "Design a messaging system."

**Candidate:** "Before I start, let me clarify some requirements. 
- For functional requirements: Are we talking about 1:1 messaging, group chats, or both? Do we need features like read receipts, typing indicators, or message reactions?
- For scale: How many daily active users? What's the average message volume per user?
- For latency: What's acceptable delivery latency? Sub-second within a region?
- For consistency: Is it okay if messages are occasionally delivered out of order, or do we need strict ordering within a conversation?"

**Interviewer:** "Let's focus on 1:1 and group messaging, 100M DAU, sub-second latency, and messages should be ordered within a conversation."

**Candidate:** "Great. And one more question: should we support message history and search, or just real-time messaging?"

---

## 2.3 Phase 2: Scale Estimation (3-5 minutes)

### How to Estimate

1. Start with user counts
2. Calculate operations per second
3. Estimate storage requirements
4. Consider peak vs. average

### Example Calculation

```
Messaging System:
- DAU: 100 million
- Messages per user per day: 50
- Total messages per day: 100M × 50 = 5 billion

Writes per second:
  5B messages / 86,400 seconds ≈ 60,000 messages/second
  Peak (2-3x): ~150,000 messages/second

Reads per second (assuming 3 reads per message on average):
  5B × 3 = 15B reads/day ≈ 170,000 reads/second

Storage:
  Average message size: 500 bytes (text + metadata)
  Daily storage: 5B × 500 bytes = 2.5 TB/day
  Yearly storage: ~1 PB
```

### Key Numbers to Remember

```
1 Million requests/day ≈ 12 requests/second
1 Billion requests/day ≈ 12,000 requests/second

Seconds in a day: 86,400 (round to 100,000 for estimation)
Seconds in a month: ~2.5 million
Seconds in a year: ~30 million

1 KB × 1 Million = 1 GB
1 MB × 1 Million = 1 TB
```

---

## 2.4 Phase 3: High-Level Design (10-15 minutes)

### Structure Your Design

1. **Start with the user flow**
   - What does the user do?
   - What path does the request take?

2. **Identify major components**
   - Client
   - API Gateway / Load Balancer
   - Application services
   - Data stores
   - Caches
   - Message queues (if async)

3. **Draw the diagram**
   - Left to right or top to bottom
   - Show data flows with arrows
   - Label each component

4. **Explain as you draw**
   - Don't draw silently
   - Explain each component's purpose

### Example Narrative

"Let me draw the high-level architecture. 

Starting from the left, we have the clients—mobile and web—connecting through a load balancer to our API Gateway. The gateway handles authentication, rate limiting, and routes requests to the appropriate services.

For messaging, we have a Chat Service that handles message sending and retrieval. This service writes to a Cassandra cluster for message storage, partitioned by conversation ID for efficient retrieval.

For real-time delivery, we have a separate WebSocket Gateway that maintains persistent connections with clients. When a message arrives, the Chat Service publishes to a message queue—let's say Kafka—and the WebSocket Gateway subscribes to deliver messages to connected users.

We also have a Presence Service backed by Redis to track online/offline status.

Should I go deeper into any of these components, or cover more of the high-level design first?"

---

## 2.5 Phase 4: Deep Dives (20-25 minutes)

### Selecting Deep Dives

**You should suggest areas based on:**
- What's most interesting/complex
- Where you have the most expertise
- What's core to the problem

**Interviewer may redirect:**
- "Let's discuss the database design in more detail"
- "How would you handle the failure case here?"
- "What about the real-time delivery component?"

### Deep Dive Structure

For each deep dive (7-10 minutes each):

1. **State the problem/component** (30 sec)
2. **Discuss options** (2 min)
3. **Explain your choice** (1 min)
4. **Go into details** (3-5 min)
5. **Discuss failure modes** (1-2 min)

### Example Deep Dive: Database Schema

"Let me deep dive into the data model for message storage.

**Options considered:**
- SQL database like PostgreSQL
- NoSQL like Cassandra or DynamoDB

**Choice:** Cassandra, because:
- Write-heavy workload (millions of messages/sec)
- Need to scale horizontally
- Query pattern is well-defined (get messages by conversation)

**Schema:**
```
Primary key: (conversation_id, sent_at, message_id)
```
This allows efficient range queries for loading conversation history.

**Partition key:** conversation_id ensures all messages for a conversation are on the same node.

**Clustering key:** sent_at, message_id ensures messages are sorted by time.

**Failure scenarios:**
- Hot partition: Very active group chat. Mitigation: Bucket by time (conversation_id + day).
- Cassandra node down: Replication factor of 3 ensures other replicas available."

---

## 2.6 Phase 5: Wrap-up (5-8 minutes)

### What to Cover

1. **Summarize key trade-offs**
   - "The main trade-off was X vs Y, and we chose X because..."

2. **Discuss improvements**
   - "If I had more time, I'd improve..."

3. **Handle "what if" questions**
   - Interviewer often asks curveballs here

4. **Ask clarifying questions**
   - "Was there any area you'd like me to explore more?"

### Common Wrap-up Questions

- "What would you change if the scale increased 100x?"
- "What are the bottlenecks in this design?"
- "How would you monitor this system?"
- "What's the hardest problem to solve here?"
- "How would you roll this out incrementally?"

---

# Section 3: Essential Interview Skills

## 3.1 How to Clarify Requirements

### The FIRED Framework

| Letter | Meaning | Questions |
|--------|---------|-----------|
| **F** | Functional | What features? Who uses it? |
| **I** | Interface | API? Mobile? Web? |
| **R** | Resilience | Availability? Consistency? Durability? |
| **E** | Estimation | Users? Traffic? Data? Growth? |
| **D** | Deployment | Single region? Multi-region? Cloud? |

### Don't Skip This Phase

Even if you think you know the problem:
- Shows systematic thinking
- Prevents wasted time on wrong design
- Gives you time to organize thoughts

---

## 3.2 How to Estimate Scale

### Step-by-Step Approach

1. **Anchor on users**
   - Total users, DAU, concurrent users

2. **Derive operations**
   - Actions per user per day
   - Convert to requests per second

3. **Calculate storage**
   - Size per record × count × retention

4. **Consider peak vs. average**
   - Peak is typically 2-3x average
   - Some events (Black Friday) are 10x

### When Unsure, State Assumptions

"I'm not sure of the exact number, so I'll assume 1 million DAU. If the interviewer has a different number in mind, please correct me."

---

## 3.3 How to Choose What to Deep Dive

### Prioritization Criteria

| Priority | What to Deep Dive |
|----------|-------------------|
| 1 | Core data model and storage |
| 2 | Most complex/interesting component |
| 3 | Scaling bottleneck |
| 4 | Failure handling |
| 5 | Your area of expertise |

### Signaling to Interviewer

"I think the most interesting challenges here are the real-time delivery and the message storage. I'd like to deep dive into the database design first. Does that work, or would you prefer a different area?"

---

## 3.4 How to Handle "What If" Questions

### Types of What-If Questions

| Type | Example | How to Handle |
|------|---------|---------------|
| Scale | "What if traffic increases 10x?" | Discuss horizontal scaling, sharding |
| Failure | "What if the database is unavailable?" | Discuss fallbacks, queuing, degradation |
| Feature | "What if we need feature X?" | Discuss how architecture would change |
| Cost | "This seems expensive. Alternatives?" | Discuss trade-offs, cheaper options |
| Constraint | "What if we need < 10ms latency?" | Discuss caching, edge deployment |

### Framework for Answering

1. **Acknowledge the challenge** (2 seconds)
2. **Explain the impact** (10 seconds)
3. **Propose solutions** (30-60 seconds)
4. **Discuss trade-offs** (15 seconds)

### Example

**Interviewer:** "What if the Redis cache goes down?"

**Candidate:** "Good question. If Redis goes down, we lose our cache layer, which means all requests hit the database directly. This would significantly increase database load and latency.

Several mitigations:
1. **Redis Cluster** with replication—if one node fails, replicas take over.
2. **Fallback to database** with graceful degradation—system works, just slower.
3. **Local caching** in application servers as a second tier.
4. **Circuit breaker** to prevent cascade failure.

For this design, I'd recommend Redis Cluster with automatic failover, plus local caching as a fallback."

---

## 3.5 How to Avoid Over-Engineering

### Signs of Over-Engineering

- Designing for 10x the stated scale
- Adding components "just in case"
- Complex solutions when simple ones work
- Microservices for a 5-person team

### How to Right-Size

1. **Design for current + 10x growth**
   - Not 100x, not 1000x
   - You can always iterate

2. **Start simple, note scaling paths**
   - "I'll start with a single database, but when we hit X, we'd add read replicas, and when we hit Y, we'd shard."

3. **Ask if unsure**
   - "Is this level of scale what you had in mind?"

---

## 3.6 Communication Best Practices

### Structure Your Thoughts

- Use frameworks (numbered points, categories)
- Summarize before diving deep
- Recap after complex explanations

### Think Out Loud

- "I'm considering two options here..."
- "The trade-off I'm thinking about is..."
- "Let me think about failure modes..."

### Draw Clearly

- Label every box and arrow
- Use consistent notation
- Erase and redraw if messy

### Check In

- "Does this make sense so far?"
- "Should I go deeper here or move on?"
- "Is there a specific aspect you'd like me to focus on?"

---

# Section 4: Quick Reference Guides

## 4.1 Phrases That Signal Seniority

### Opening Phrases

```
"Before designing, let me clarify the requirements..."
"Let me make sure I understand the problem space..."
"I want to understand the constraints before proposing solutions..."
```

### Trade-off Phrases

```
"There's a trade-off here between X and Y..."
"We could choose A for [benefit] or B for [different benefit]..."
"Given our requirement for Z, I'd lean toward A because..."
"If we had different constraints, I might reconsider..."
```

### Failure Handling Phrases

```
"Let me consider what happens when X fails..."
"We need to design for the failure case here..."
"To make this resilient, we should consider..."
"The failure mode here is..., and we handle it by..."
```

### Scaling Phrases

```
"At this scale, the bottleneck would be..."
"As we grow, we'd need to..."
"For now, X is sufficient, but at 10x scale, we'd need Y..."
"This design handles up to Z; beyond that, we'd need to reconsider..."
```

---

## 4.2 Common Interview Problems and Key Points

| Problem | Key Points to Cover |
|---------|-------------------|
| **URL Shortener** | Short URL generation, redirection, analytics, 301 vs 302 |
| **Rate Limiter** | Token bucket, distributed Redis, fail open vs closed |
| **Notification System** | Multi-channel, priority queues, delivery guarantees |
| **File Storage** | Chunking, replication, metadata service |
| **News Feed** | Fan-out on write vs read, celebrity problem |
| **Payment System** | Idempotency, ledger, reconciliation |
| **Chat System** | WebSocket, presence, message ordering |
| **Search Autocomplete** | Trie, ranking, freshness |
| **Distributed Cache** | Consistent hashing, persistence options |

---

## 4.3 Red Flags to Avoid

| Red Flag | How to Avoid |
|----------|--------------|
| "It's obvious" | Nothing is obvious; explain your reasoning |
| "I don't know" (and stopping) | "I don't know, but my approach would be..." |
| Long silences | Think out loud |
| Only happy path | Always discuss failures |
| Name-dropping without depth | Only mention what you can explain |
| Not managing time | Check time regularly |
| Waiting for guidance | Drive the discussion |
| Getting defensive | Accept feedback gracefully |

---

## 4.4 Interview Checklist

### Before the Interview

- [ ] Practiced 10+ system design problems
- [ ] Comfortable with back-of-envelope math
- [ ] Have a consistent diagram style
- [ ] Prepared for common follow-up questions
- [ ] Tested audio/video (for remote)
- [ ] Have paper and pen (or tablet) ready

### During Requirements Phase

- [ ] Asked about functional requirements
- [ ] Asked about scale (users, traffic, data)
- [ ] Asked about latency requirements
- [ ] Asked about consistency requirements
- [ ] Asked about any specific constraints

### During Design Phase

- [ ] Did back-of-envelope calculations
- [ ] Drew clear, labeled diagram
- [ ] Explained major data flows
- [ ] Covered all major components
- [ ] Discussed at least 2-3 trade-offs

### During Deep Dives

- [ ] Went deep on 2-3 components
- [ ] Discussed database design
- [ ] Addressed scaling strategies
- [ ] Covered failure scenarios
- [ ] Mentioned monitoring/observability

### Wrap-up

- [ ] Summarized key trade-offs
- [ ] Mentioned potential improvements
- [ ] Handled follow-up questions
- [ ] Asked for feedback (if time)

---

# Section 5: Practice Exercises

## 5.1 Mock Interview Self-Practice

### Setup
- Set a 45-minute timer
- Use paper or whiteboard
- Record yourself (optional but helpful)

### Practice Problems (in order of difficulty)

**Warm-up (15 minutes each):**
1. URL Shortener
2. Pastebin
3. Rate Limiter

**Core (45 minutes each):**
4. Notification System
5. Chat System
6. News Feed
7. Payment System

**Advanced (45-60 minutes each):**
8. Distributed Cache
9. Search Engine
10. Video Streaming

### Self-Evaluation

After each practice:
1. Did I clarify requirements? (5 points)
2. Did I estimate scale? (5 points)
3. Did I cover all major components? (10 points)
4. Did I go deep on 2-3 areas? (10 points)
5. Did I discuss trade-offs? (10 points)
6. Did I handle failures? (5 points)
7. Did I manage time well? (5 points)

**Score 35+:** Ready for interviews
**Score 25-35:** More practice needed
**Score <25:** Review fundamentals first

---

## 5.2 Improvement Strategies

### If You Struggle with Requirements
- Practice the FIRED framework
- Write down questions before each practice
- Review what you missed after practice

### If You Struggle with Scale Estimation
- Memorize the key numbers
- Practice 10 estimation problems
- Focus on being within order of magnitude

### If You Struggle with Depth
- Pick 3 components and study deeply
- Understand how Redis, Kafka, Cassandra actually work
- Read engineering blogs from companies

### If You Struggle with Trade-offs
- For every decision, force yourself to state an alternative
- Read about real-world architecture decisions
- Study post-mortems

### If You Struggle with Communication
- Record and watch yourself
- Practice with a friend or interview prep service
- Join a system design study group

---

# Phase 5 Summary

Interview execution is a skill that can be practiced and improved. The key points are:

1. **Structure your time** - Don't spend 30 minutes on requirements
2. **Drive the discussion** - Don't wait for guidance
3. **Think out loud** - Let the interviewer see your reasoning
4. **Cover trade-offs** - Every decision has alternatives
5. **Handle failures** - What happens when things break?
6. **Right-size your design** - Don't over-engineer

### The Seniority Equation

```
Technical Knowledge + Communication Skills + Engineering Judgment = Hire Decision
```

You need all three. Technical knowledge alone isn't enough. Practice the meta-skills as much as the technical content.

### Final Advice

- Practice with a timer
- Record yourself and review
- Get mock interview feedback
- Learn from each interview
- Stay calm and structured

**Remember:** The interviewer wants you to succeed. They're looking for a colleague, not trying to trick you. Show them how you think, not just what you know.
