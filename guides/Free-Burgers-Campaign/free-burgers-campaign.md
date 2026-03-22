# 6 Million Free Burgers Campaign

## Problem

Design a system where **6,000,000 (six million) free burgers** are to be claimed in a **marketing campaign lasting 10 minutes**, with **1 free burger per account**.

**Topics:** High-scale throughput, concurrency control, event monitoring  
**Difficulty:** Senior (high scale)  
**Est. Time:** ~45–60 min

---

## What is it?

A **flash campaign system** where millions of users compete to claim a limited supply of free rewards within a short, fixed time window. The core challenges:

1. **Extreme hotspot** — All 6M claims hit a **single logical resource** (burger stock). No natural partitioning by item type.
2. **High throughput** — 10,000 claims/sec average; peak can be 2–3× higher (20–30K RPS).
3. **Strict constraints** — Exactly 6M claims allowed; exactly 1 per account (no double-claiming).
4. **Burst traffic** — 10 minutes of intense load, then near-zero; auto-scaling must ramp quickly.
5. **Event monitoring** — Real-time dashboards, fraud detection, analytics.

---

## Clarifying Questions (Engineers Should Ask)

### Your questions (and why they matter)

**1. Consistency and correctness**

- Must we **never** exceed 6M claims? Never allow 1 account to claim twice?
- **Assume:** Strong consistency — no overselling; no double-claims.

**2. Response model**

- Is the claim flow synchronous (user waits for success/fail) or async (queue, poll later)?
- **Assume:** Synchronous — user gets immediate success or failure (better UX for a marketing campaign).

**3. Scale**

- Expected concurrent users? Geographic distribution?
- **Assume:** Peak 20–30K RPS; global (US, EU, APAC).

**4. Account definition**

- What counts as "one account"? User ID, email, device fingerprint?
- **Assume:** One account = one authenticated `user_id` (login required).

**5. Monitoring**

- Real-time dashboards? Fraud/abuse detection?
- **Assume:** Yes — live claim count, success/fail rates, bot detection, alerts.

---

### Other questions worth asking in an interview

- **Authentication:** Do we require login before claiming, or allow anonymous + later signup?
- **Delivery:** How is the free burger "delivered"? (Voucher code, app credit, loyalty points, in-store redemption?)
- **Campaign start:** Fixed start time (e.g. noon) or "first come" from announcement?
- **Rollback:** If fraud detected later, do we revoke claims?

---

## Requirements

### Functional Requirements

1. **Claim free burger** — Authenticated user requests 1 free burger.
2. **Prevent overselling** — Total claims must never exceed 6,000,000.
3. **Prevent double-claiming** — Each account can claim at most once.
4. **Synchronous response** — User receives success or failure immediately.
5. **Campaign status** — Users can check if campaign is active, and remaining count (approximate OK).
6. **Optional:** Admin can start/stop campaign, view real-time stats.

### Non-Functional Requirements

| NFR | Target |
|-----|--------|
| **Throughput** | 10,000+ claims/sec (avg); 20–30K RPS peak. |
| **Consistency** | No overselling; no double-claims. |
| **Latency** | Claim response < 500 ms (p99). |
| **Availability** | 99.9% during campaign window. |
| **Idempotency** | Retries (double-click, timeout) do not create duplicate claims. |
| **Observability** | Real-time dashboards, alerts, fraud detection. |

---

## Capacity Estimation

**Assumptions:**

- 6M claims in 10 minutes = 600 seconds  
- Average: 6,000,000 ÷ 600 = **10,000 claims/sec**  
- Peak (2–3×): **20,000–30,000 RPS**

**Storage (rough):**

- Claim record: ~200 bytes (user_id, campaign_id, timestamp, status)  
- 6M claims ≈ 1.2 GB  
- Account-claimed check: 6M user_ids × 8 bytes ≈ 48 MB (Redis set or DB index)

---

## Core Entities / Data Model

### Campaign

| Field | Type | Description |
|-------|------|-------------|
| `id` | VARCHAR | Primary key. |
| `name` | VARCHAR | e.g. "Free Burger March 2025". |
| `total_quantity` | INT | 6,000,000. |
| `claimed_count` | INT | Running total (or derived from claims). |
| `start_time` | TIMESTAMP | Campaign start. |
| `end_time` | TIMESTAMP | Campaign end (start + 10 min). |
| `status` | VARCHAR | `DRAFT`, `ACTIVE`, `ENDED`. |

### Claim

| Field | Type | Description |
|-------|------|-------------|
| `id` | VARCHAR | Primary key (UUID). |
| `campaign_id` | VARCHAR | Foreign key. |
| `user_id` | VARCHAR | Claiming account. |
| `status` | VARCHAR | `SUCCESS`, `REJECTED_DUPLICATE`, `REJECTED_EXHAUSTED`. |
| `created_at` | TIMESTAMP | |

### User (existing)

| Field | Type | Description |
|-------|------|-------------|
| `id` | VARCHAR | Primary key. |
| `email` | VARCHAR | |

---

## API Design

### Claim Burger

**`POST /campaigns/{campaign_id}/claim`**

| Header / Body | Type | Required | Description |
|---------------|------|----------|-------------|
| `Idempotency-Key` | string | recommended | Prevents duplicate claims on retry. |
| `Authorization` | string | yes | Bearer token (user authentication). |
| `campaign_id` | path | yes | Campaign ID. |

**Response (200 Success):**

```json
{
  "claim_id": "clm_abc123",
  "status": "SUCCESS",
  "message": "You've claimed your free burger!"
}
```

**Response (409 Conflict — already claimed):**

```json
{
  "claim_id": "clm_xyz789",
  "status": "REJECTED_DUPLICATE",
  "message": "You have already claimed your free burger."
}
```

**Response (410 Gone — stock exhausted):**

```json
{
  "status": "REJECTED_EXHAUSTED",
  "message": "Sorry, all burgers have been claimed."
}
```

### Get Campaign Status

**`GET /campaigns/{campaign_id}`**

Returns campaign status, approximate remaining count (eventual consistency OK for display).

---

## High-Level Architecture

**Flow:**

```
Client → CDN / Edge → Load Balancer → API Gateway (rate limit, auth, idempotency)
                                                    ↓
                                          Claim Service
                                                    ↓
                    ┌──────────────────────────────┼──────────────────────────────┐
                    ↓                              ↓                              ↓
            Redis (claimed set)          Kafka (claim queue)              PostgreSQL (claims)
                    ↓                              ↓
            "Has user claimed?"           Workers (serialized)              Permanent record
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **API Gateway** | Rate limiting, auth, idempotency, routing. |
| **Claim Service** | Validate campaign, check Redis (claimed?), enqueue or reject. |
| **Redis** | O(1) "has user claimed?" check; optional counter cache. |
| **Kafka** | Serialize claim processing; partition by campaign. |
| **Workers** | Consume claims; atomic DB decrement; update Redis. |
| **PostgreSQL** | Source of truth for claims; atomic stock updates. |
| **Event Stream** | Real-time events for dashboards, alerts, analytics. |

---

## Deep Dive: Bad vs. Good vs. Best Approach

### 🔴 BAD APPROACH

**How it looks:**

- Single REST endpoint: `POST /claim`
- **Check-then-act** in application code:
  1. `SELECT remaining FROM campaign WHERE id = X`
  2. If `remaining > 0`, `UPDATE campaign SET remaining = remaining - 1`
  3. Insert claim record
- No idempotency; no per-account check (or check in app only, not atomic)
- Single DB row for `campaign` (remaining count)
- No partitioning; no queue; no rate limiting

**Why it fails:**

| Issue | Consequence |
|-------|-------------|
| **Race condition** | Two requests both read `remaining = 1`; both decrement → overselling. |
| **Double-claim** | User retries or double-clicks; no idempotency → 2 claims for 1 account. |
| **Hot row** | All 10K+ RPS hit one `campaign` row → lock contention, timeouts, cascading failures. |
| **No rate limiting** | Bots can drain stock before real users. |
| **DB overload** | Single DB becomes bottleneck; 10K+ writes/sec on one row. |

**Interview tip:** *"This approach is fundamentally broken due to non-atomic check-then-act and lack of concurrency control. We'd see overselling, duplicate claims, and system collapse under load."*

---

### 🟡 GOOD APPROACH

**How it looks:**

- **Atomic DB update** for stock (no check-then-act):
  ```sql
  UPDATE campaigns
  SET remaining_quantity = remaining_quantity - 1
  WHERE id = $campaign_id AND remaining_quantity >= 1
  RETURNING remaining_quantity;
  ```
- **Per-account check** in DB: `INSERT INTO claims ... ON CONFLICT (campaign_id, user_id) DO NOTHING` or unique constraint; check `affected_rows` before decrementing stock.
- **Idempotency key** — Store `(idempotency_key, user_id, campaign_id)`; return cached result on replay.
- **Stock partitioning** — Split 6M into N partitions (e.g. 100 rows of 60K each); randomly pick partition per request to reduce contention.
- **Rate limiting** — Per-user and per-IP limits (e.g. 5 req/min per user).
- **Redis cache** — Optional: cache "user has claimed" for fast rejection; DB is source of truth.

**Why it works:**

| Fix | Benefit |
|-----|---------|
| **Atomic update** | No overselling; DB guarantees serialization. |
| **Unique (campaign_id, user_id)** | Prevents double-claim at DB level. |
| **Partitioning** | Spreads load across N rows; reduces lock contention. |
| **Idempotency** | Safe retries. |
| **Rate limiting** | Slows bots. |

**Limitations:**

| Limitation | Impact |
|------------|--------|
| **DB still bottleneck** | Even with 100 partitions, 10K RPS = 100 writes/partition/sec. Doable but stressful. |
| **Order of checks** | Must check "already claimed" *before* attempting stock decrement to avoid wasting decrements on duplicates. |
| **No real-time streaming** | Dashboard would need to poll DB or use batch aggregation. |

**Interview tip:** *"This is a solid baseline. Atomic DB updates and partitioning get us correctness and reasonable scale. For 10K RPS it may work with enough DB capacity, but at 20–30K RPS we'd need something more."*

---

### 🟢 BEST APPROACH

**How it looks:**

1. **Redis-first duplicate check** — `SADD campaign:claimed:{campaign_id} {user_id}`. If `SADD` returns 0, user already claimed → reject immediately. If 1, proceed.
2. **Queue-based serialization** — Claim request → Kafka topic `campaign-claims`, partitioned by `campaign_id`. Single partition per campaign = strict ordering.
3. **Workers consume serially** — One consumer (or consumer group with one active) per campaign partition. Worker: atomic DB decrement; on success, record claim; on failure (exhausted), optionally `SREM` user from Redis (if we want to allow retry logic elsewhere — usually we keep them in set).
4. **Pre-sharded stock in DB** — 6M = 100 shards of 60K. Worker picks shard via round-robin or consistent hash; decrements that shard. Reduces per-row contention further.
5. **Synchronous response to user** — Option A: Request-reply pattern (Kafka request/reply, or temporary response topic). Option B: Optimistic — if Redis `SADD` succeeds, return "Claim received, processing..." and process async; user polls or gets webhook. **Best UX:** Use request-reply so user gets sync success/fail.
6. **Idempotency** — Before Redis `SADD`: check idempotency key in Redis/DB. If seen, return cached response.
7. **Rate limiting** — Per-user (e.g. 2 req/10s), per-IP, global. Token bucket in Redis.
8. **Event streaming** — Every claim (success/fail) → Kafka `campaign-events`. Real-time dashboards (Kafka Streams, Flink, or consuming into TimescaleDB/Pinot) for live metrics.
9. **Circuit breakers** — If Redis or Kafka is down, fail fast; return 503 with Retry-After.

**Why it's best:**

| Feature | Benefit |
|---------|---------|
| **Redis SADD for duplicate check** | O(1), in-memory; absorbs most duplicate/retry traffic before DB. |
| **Queue serialization** | Zero contention on stock; workers process at controlled rate. |
| **Pre-sharded stock** | Even with queue, parallel workers can handle multiple shards. |
| **Request-reply** | User gets synchronous success/fail without sacrificing correctness. |
| **Event stream** | Real-time monitoring, fraud detection, analytics. |
| **Graceful degradation** | Circuit breakers prevent cascade; clear error messages. |

**Flow (simplified):**

```
User → API Gateway (rate limit, auth)
     → Claim Service (idempotency check, Redis SADD)
     → If SADD new: produce to Kafka "claim" topic
     → Worker consumes, atomic DB decrement, insert claim
     → Response (sync via request-reply or async via webhook)
```

**Interview tip:** *"For 20–30K RPS and strict correctness, I'd use Redis for duplicate detection and a queue to serialize stock deduction. This gives us horizontal scalability, real-time events, and no overselling. The trade-off is added complexity (Kafka, workers, request-reply), but for this scale it's justified."*

---

## Comparison Table

| Aspect | Bad | Good | Best |
|--------|-----|------|------|
| **Stock safety** | ❌ Race, overselling | ✅ Atomic DB | ✅ Queue + atomic DB |
| **Duplicate claim** | ❌ No protection | ✅ DB unique + idempotency | ✅ Redis SADD + idempotency |
| **Throughput** | ❌ Single row meltdown | ⚠️ 10K RPS with partitioning | ✅ 20–30K+ via queue + sharding |
| **Latency** | ❌ Locks, timeouts | ⚠️ p99 can spike | ✅ Controlled via queue |
| **Bot protection** | ❌ None | ✅ Rate limiting | ✅ Rate limit + event-based fraud |
| **Monitoring** | ❌ Minimal | ⚠️ Polling | ✅ Real-time event stream |
| **Complexity** | Low | Medium | High |

---

## Preventing Overselling (Implementation Detail)

### Atomic decrement (single shard)

```sql
UPDATE campaign_shards
SET remaining_quantity = remaining_quantity - 1
WHERE campaign_id = $cid AND shard_id = $sid AND remaining_quantity >= 1
RETURNING remaining_quantity;
```

- Rows updated = 1 → success.  
- Rows updated = 0 → shard exhausted; try next shard or reject.

### Shard selection

With 100 shards (0–99), use `shard_id = hash(user_id) % 100` for deterministic per-user shard (reduces cross-shard logic), or round-robin for even spread.

---

## Idempotency

| Step | Action |
|------|--------|
| 1 | Client sends `Idempotency-Key: claim-{uuid}`. |
| 2 | Service checks Redis: `GET idempotency:{key}`. |
| 3 | If exists → return cached response. |
| 4 | If not → process claim; on completion, `SET idempotency:{key} {response}` with TTL 24h. |

---

## Event Monitoring & Real-Time Dashboards

**Key events:**

| Event | When | Use |
|-------|------|-----|
| `claim.submitted` | Request received | Throughput, rate |
| `claim.success` | Claim recorded | Success rate, total claimed |
| `claim.rejected_duplicate` | User already claimed | Fraud / retry analysis |
| `claim.rejected_exhausted` | Stock depleted | Campaign end detection |
| `claim.rejected_rate_limited` | Rate limit hit | Bot detection |

**Pipeline:**

```
Claim Service / Workers → Kafka (campaign-events) → Stream processor
                                                              ↓
                                              Real-time dashboard (Grafana, custom)
                                              Fraud detection (velocity, patterns)
                                              Analytics warehouse (batch)
```

**Metrics to track:**

- Claims/sec (rolling)
- Success vs. reject breakdown
- p50, p95, p99 latency
- Redis hit rate (claimed check)
- Kafka consumer lag
- DB connection pool utilization

---

## Scaling Strategy

| Layer | Strategy |
|-------|----------|
| **API Gateway** | Horizontal scaling; auto-scale on RPS. |
| **Claim Service** | Stateless; scale with traffic. |
| **Redis** | Cluster mode; `campaign:claimed` set can be sharded by campaign_id. |
| **Kafka** | Multiple partitions for `campaign-claims` only if multiple campaigns; single campaign = single partition for ordering. |
| **Workers** | One consumer per partition; scale partitions only for more campaigns. |
| **PostgreSQL** | Read replicas for dashboards; primary for writes. Shard claims table by campaign_id if needed. |

---

## Failure Handling

| Failure | Handling |
|---------|----------|
| **Redis down** | Fallback to DB for "claimed" check (slower); or fail open with 503. |
| **Kafka down** | Circuit breaker; return 503 Retry-After. |
| **DB down** | Retry with backoff; eventually fail claim. |
| **Worker crash** | Kafka consumer commits only after success; replay on restart. |
| **Campaign ended** | Reject immediately at API layer (check `end_time` before processing). |

---

## Security

- **Authentication** — Require valid JWT/session; reject unauthenticated requests.
- **Rate limiting** — Per-user (2–5 req/10s), per-IP (e.g. 10 req/10s), global burst limit.
- **Bot detection** — Velocity checks, device fingerprinting, CAPTCHA if needed.
- **Audit** — Log all claims for dispute resolution and fraud analysis.

---

## Wrap-Up (Strong Closing Statement)

*"The system ensures exactly 6M claims and 1 per account by combining Redis for fast duplicate detection, Kafka for serialized stock processing, and atomic DB updates. The bad approach fails due to race conditions and lack of partitioning. The good approach achieves correctness with atomic updates and partitioning but may struggle at peak load. The best approach adds queue-based serialization and real-time event streaming for high throughput, observability, and fraud detection. For a 10-minute flash campaign at 20–30K RPS, the best approach is justified by the scale and the need for real-time monitoring."*

---

## Why This Scores Well in Interviews

This design demonstrates:

- **Throughput** — Queue-based serialization, partitioning, Redis caching
- **Concurrency control** — Atomic updates, Redis SADD, idempotency
- **Trade-off reasoning** — Bad vs. Good vs. Best with clear justification
- **Event monitoring** — Real-time dashboards, fraud detection, analytics
- **Failure handling** — Circuit breakers, fallbacks, graceful degradation
- **Security** — Rate limiting, auth, bot detection
