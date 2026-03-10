# Global Digital Rewards Distribution System

## What is it?

A **global digital rewards distribution platform** that enables brands, merchants, and platforms to **create, distribute, and track digital rewards** (points, gift cards, vouchers, coupons, crypto airdrops) to end users at scale across the world. The system handles **campaign creation**, **targeted distribution** (by region, segment, or trigger), **delivery** via multiple channels (email, in-app, push, SMS, QR), **redemption tracking**, and **fraud prevention** while meeting **GDPR**, regional compliance, and **multi-currency** requirements.

---

## Clarifying Questions (Engineers Should Ask)

### Your questions (and why they matter)

**1. Who are the primary users?**

- **Reward issuers (brands, merchants, platforms)** — Create campaigns, define reward types, target audiences, and monitor distribution and redemption.
- **End recipients (consumers)** — Receive rewards, redeem them, and view their reward history.
- Clarifies auth (API keys, OAuth for platforms), roles, and which flows we prioritize.

**2. What types of rewards do we need to support?**

- **Answer for this system:**
  - **Points** — Loyalty points (e.g. 100 points = $1); can be earned and redeemed.
  - **Gift cards** — Pre-loaded value (e.g. $25 Amazon); single-use or multi-use.
  - **Vouchers / promo codes** — Discount codes (e.g. 20% off); may have expiry and usage limits.
  - **Coupons** — Product-specific or cart-level discounts.
  - **Crypto / airdrops** — Token or NFT distribution (optional; adds wallet/blockchain complexity).

- Good question: it scopes MVP (e.g. points + vouchers only vs. full catalog) and integration complexity.

**3. How are rewards distributed? (Distribution triggers)**

| Trigger | Description | Example |
|---------|-------------|---------|
| **Campaign** | Issuer schedules a campaign; users in a segment receive rewards. | "New users in EU get 500 points." |
| **Achievement** | User completes an action (signup, purchase, referral). | "First purchase: $10 gift card." |
| **Event-driven** | External system triggers distribution via API. | "Order shipped: send voucher." |
| **Manual** | Issuer manually assigns reward to a user. | "Compensation for complaint." |

**4. What delivery channels do we support?**

- **Email** — Send reward code/link; must handle deliverability and localization.
- **In-app** — Push to user's app wallet; requires SDK or deep link.
- **Push notification** — Mobile push with reward summary.
- **SMS** — Short code or long code; regional compliance (e.g. TCPA, GDPR).
- **QR code** — Generate QR for in-store redemption.
- **Direct link** — Shareable URL (e.g. for referral rewards).

**5. How should the system behave for distribution vs. read operations? (CAP)**

| Operation | What we need | CAP / trade-off |
|-----------|--------------|------------------|
| **Create reward, distribute, redeem** | No double-issuance; no double-redemption; balance/entitlement correctness. | Prefer **consistency**. Idempotency for distribution; strong consistency for redemption. |
| **List campaigns, get reward status, analytics** | Issuers view dashboards; eventual consistency OK. | Prefer **availability** and **low latency**. Read replicas, cache. |

**6. What about duplicate distribution (e.g. retry after timeout)?**

- **Idempotency**: Same request (same idempotency key) must not create duplicate rewards for the same user. Essential for campaign and API-triggered distribution.

---

### Other questions worth asking in an interview

- **Scale:** How many rewards per second? (e.g. 10K vs 1M/day) — Drives queue design, batch processing, and sharding.
- **Geographic scope:** Single region or global? — Affects data residency, latency, and compliance (GDPR, CCPA, regional gift card laws).
- **Redemption flow:** Where do users redeem? (Issuer's site, partner network, in-store?) — Affects integration and validation APIs.
- **Fraud:** Do we need to detect and prevent abuse (e.g. fake accounts, bot farms)? — Affects rate limits, device fingerprinting, and ML pipelines.
- **Multi-currency:** Do rewards have monetary value in multiple currencies? — Affects display and redemption logic.

---

## Requirements

### Functional Requirements

1. **Campaign management** — Issuer can create campaigns with reward type, amount, target segment (region, cohort, custom attributes), schedule, and delivery channel.
2. **Distribution** — System distributes rewards to users based on campaign rules or API triggers; supports batch and real-time distribution.
3. **Delivery** — Rewards are delivered via chosen channel (email, in-app, push, SMS, QR); user receives code/link/balance update.
4. **Redemption** — User can redeem reward (apply code, use points); system validates and records redemption; prevents double-redemption.
5. **Wallet / balance** — User can view their rewards (points balance, unredeemed vouchers, gift cards).
6. **Analytics** — Issuer can view distribution metrics (sent, delivered, opened, redeemed), redemption rates, and fraud signals.

### Non-Functional Requirements

- **Consistency for distribution and redemption** — No double-issuance; no double-redemption; idempotent distribution APIs.
- **Idempotency** — Distribution and redemption APIs accept **idempotency key**; duplicate requests return same result.
- **Availability** — High availability (e.g. 99.9%+); read path can tolerate eventual consistency.
- **Low latency** — Distribution API response in **under a few hundred ms**; read operations under ~100 ms where cached.
- **Global compliance** — **GDPR** (data minimization, right to deletion), **CCPA**, regional gift card laws (e.g. expiry rules), SMS consent (TCPA).
- **Fraud prevention** — Rate limits, device/user fingerprinting, anomaly detection for bulk abuse.
- **Localization** — Multi-language reward content; currency and date formatting by region.

---

## Core Entities

| Entity | Description |
|--------|-------------|
| **Issuer** | Brand/merchant/platform that creates and funds campaigns. Has API keys, balance (for funded rewards), and config (default currency, regions). |
| **Campaign** | A distribution rule set: reward type, amount, target segment, schedule, delivery channel. Has status (draft, active, paused, completed). |
| **Reward** | A single reward instance (e.g. one voucher code, one gift card). Has type, value, status (issued, delivered, redeemed, expired), and links to user and campaign. |
| **User** | End recipient. Identified by `user_id` (issuer-provided) and optionally email, phone, device_id. Has region, preferences. |
| **Segment** | Target audience definition (e.g. region=EU, cohort=new_users, custom attributes). Used by campaigns. |
| **Redemption** | Record of a reward being used. Has reward_id, redeemed_at, redemption_context (order_id, etc.). |
| **Event** | Immutable event for analytics and webhooks (e.g. `reward.distributed`, `reward.redeemed`). |

---

## API

### 1. Create campaign

**`POST /v1/campaigns`**

Creates a campaign. Idempotent with `Idempotency-Key`.

| Header / Body | Type | Required | Description |
|---------------|------|----------|-------------|
| `Idempotency-Key` | string | recommended | Prevents duplicate campaigns. |
| `name` | string | yes | Campaign name. |
| `reward_type` | string | yes | e.g. `points`, `gift_card`, `voucher`, `coupon`. |
| `reward_value` | object | yes | `{ "amount": 500, "currency": "usd" }` or `{ "points": 100 }`. |
| `segment` | object | yes | `{ "region": ["eu", "us"], "cohort": "new_users" }` or custom. |
| `schedule` | object | no | `{ "start_at": "...", "end_at": "..." }`. |
| `delivery_channel` | string | yes | e.g. `email`, `in_app`, `push`, `sms`. |
| `max_distributions` | int | no | Cap total rewards (e.g. first 10K users). |

**Response:** Campaign object (`id`, `name`, `status`, `created_at`).

---

### 2. Distribute reward (API-triggered)

**`POST /v1/rewards/distribute`**

Distributes a reward to a specific user. Idempotent with `Idempotency-Key`.

| Header / Body | Type | Required | Description |
|---------------|------|----------|-------------|
| `Idempotency-Key` | string | recommended | Prevents duplicate distribution to same user. |
| `user_id` | string | yes | Issuer's user identifier. |
| `campaign_id` | string | no | Link to campaign; omit for ad-hoc. |
| `reward_type` | string | yes | e.g. `voucher`, `gift_card`. |
| `reward_value` | object | yes | `{ "amount": 10, "currency": "usd" }`. |
| `delivery_channel` | string | yes | e.g. `email`, `in_app`. |
| `metadata` | object | no | Custom data (e.g. `order_id`). |

**Response:** Reward object (`id`, `status`, `delivery_status`, `expires_at`).

---

### 3. List user rewards (wallet)

**`GET /v1/users/{user_id}/rewards`**

Returns rewards for a user (points balance, unredeemed vouchers, etc.).

| Query | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | no | Filter: `active`, `redeemed`, `expired`. |
| `reward_type` | string | no | Filter by type. |
| `limit` | int | no | Max results. Default 20. |
| `cursor` | string | no | Pagination cursor. |

**Response:** List of Reward objects and `next_cursor`.

---

### 4. Redeem reward

**`POST /v1/rewards/{reward_id}/redeem`**

Redeems a reward. Idempotent with `Idempotency-Key`.

| Header / Body | Type | Required | Description |
|---------------|------|----------|-------------|
| `Idempotency-Key` | string | recommended | Prevents double-redemption. |
| `redemption_context` | object | no | e.g. `{ "order_id": "..." }` for audit. |

**Response:** Redemption object (`id`, `reward_id`, `redeemed_at`, `status`).

---

### 5. Validate reward (for checkout)

**`POST /v1/rewards/validate`**

Validates a reward code or ID for redemption at checkout. Used by issuer's or partner's backend.

| Body | Type | Required | Description |
|------|------|----------|-------------|
| `code` | string | no* | Voucher/promo code. *Required if no `reward_id`. |
| `reward_id` | string | no* | Reward ID. *Required if no `code`. |
| `order_amount` | int | no | For minimum-order validation. |
| `order_currency` | string | no | For currency match. |

**Response:** `{ "valid": true, "reward": {...}, "discount_amount": 10 }` or `{ "valid": false, "reason": "expired" }`.

---

### 6. Webhooks (outgoing)

Issuer registers a webhook endpoint. Platform sends **HTTP POST** for events (`reward.distributed`, `reward.redeemed`, `reward.expired`). At-least-once delivery; issuer should process idempotently by event ID.

---

## High-Level Design

The design covers the main flows as follows.

### 1. Campaign creation and execution

- Issuer creates campaign via **POST /v1/campaigns**.
- **Campaign Service** persists campaign, evaluates **Segment** (e.g. users in EU, new users) and produces a **distribution queue** (user_ids + campaign_id).
- **Distribution Workers** consume queue: for each user, call **Reward Service** to create reward, then **Delivery Service** to send via channel (email, in-app, etc.).
- Idempotency: (campaign_id, user_id) or (idempotency_key) prevents duplicate rewards per user.

### 2. API-triggered distribution

- External system calls **POST /v1/rewards/distribute** with `Idempotency-Key`, `user_id`, `reward_type`, `reward_value`, `delivery_channel`.
- **API Gateway** validates auth, checks idempotency (return cached response if seen).
- **Reward Service** creates reward record, generates code/link if needed, updates user balance (for points).
- **Delivery Service** sends via channel (email provider, push service, in-app notification).
- Event emitted for webhooks and analytics.

### 3. Redemption flow

- User redeems via **POST /v1/rewards/{id}/redeem** or issuer's checkout calls **POST /v1/rewards/validate** with code.
- **Reward Service** validates (not expired, not already redeemed, meets order constraints), creates **Redemption** record, marks reward as redeemed.
- Idempotency key prevents double-redemption.
- Event emitted for webhooks.

### 4. Read path (wallet, analytics)

- **List rewards / get campaign stats:** Read from **primary DB or read replicas**; cache hot data (e.g. user wallet). Pagination via cursor.
- **Analytics:** Events streamed to **data warehouse** (e.g. BigQuery, Snowflake) or **OLAP**; dashboards built on aggregated metrics.

### 5. Global considerations

- **Multi-region deployment:** Campaign and distribution workers in each region; user data routed by region for compliance (e.g. EU data in EU).
- **Delivery partners:** Email (SendGrid, SES), SMS (Twilio, regional providers), Push (FCM, APNs). Failover and retry per channel.
- **Fraud:** Rate limits per user/issuer; device fingerprinting; ML model for anomaly detection (bulk distribution to same device, etc.).

---

## Database Schema

### Issuers

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Issuer ID. |
| `name` | VARCHAR(255) | | |
| `default_currency` | VARCHAR(3) | | e.g. `usd`. |
| `balance` | BIGINT | | For funded rewards (e.g. gift card pool). |
| `created_at` | TIMESTAMP | | |

### Campaigns

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Campaign ID. |
| `issuer_id` | VARCHAR(255) | FOREIGN KEY | |
| `name` | VARCHAR(255) | | |
| `reward_type` | VARCHAR(50) | | e.g. `points`, `voucher`. |
| `reward_value` | JSONB | | `{ "amount": 10, "currency": "usd" }`. |
| `segment` | JSONB | | Target audience definition. |
| `schedule` | JSONB | | `{ "start_at", "end_at" }`. |
| `delivery_channel` | VARCHAR(50) | | |
| `max_distributions` | INT | | Cap. |
| `status` | VARCHAR(50) | | `draft`, `active`, `paused`, `completed`. |
| `idempotency_key` | VARCHAR(255) | UNIQUE | |
| `created_at` | TIMESTAMP | | |

**Indexes:** `INDEX (issuer_id, status)`, `INDEX (issuer_id, created_at)`.

### Users (issuer-scoped)

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Composite: issuer_id + user_id. |
| `issuer_id` | VARCHAR(255) | FOREIGN KEY | |
| `external_user_id` | VARCHAR(255) | | Issuer's user identifier. |
| `email` | VARCHAR(255) | | |
| `phone` | VARCHAR(50) | | |
| `region` | VARCHAR(10) | | e.g. `eu`, `us`. |
| `created_at` | TIMESTAMP | | |

**Indexes:** `UNIQUE (issuer_id, external_user_id)`, `INDEX (issuer_id, region)`.

### Rewards

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Reward ID. |
| `issuer_id` | VARCHAR(255) | FOREIGN KEY | |
| `user_id` | VARCHAR(255) | FOREIGN KEY | |
| `campaign_id` | VARCHAR(255) | FOREIGN KEY | Optional. |
| `reward_type` | VARCHAR(50) | | |
| `reward_value` | JSONB | | |
| `code` | VARCHAR(100) | UNIQUE | Voucher/promo code; nullable for points. |
| `status` | VARCHAR(50) | | `issued`, `delivered`, `redeemed`, `expired`. |
| `delivery_channel` | VARCHAR(50) | | |
| `expires_at` | TIMESTAMP | | |
| `idempotency_key` | VARCHAR(255) | UNIQUE | Per issuer scope. |
| `created_at` | TIMESTAMP | | |

**Indexes:** `INDEX (issuer_id, user_id, status)`, `INDEX (code)` for validation, `INDEX (issuer_id, idempotency_key)`.

### Redemptions

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | |
| `reward_id` | VARCHAR(255) | FOREIGN KEY | |
| `issuer_id` | VARCHAR(255) | FOREIGN KEY | |
| `redemption_context` | JSONB | | e.g. `{ "order_id": "..." }`. |
| `idempotency_key` | VARCHAR(255) | UNIQUE | |
| `redeemed_at` | TIMESTAMP | | |
| `created_at` | TIMESTAMP | | |

**Indexes:** `INDEX (reward_id)`, `INDEX (issuer_id, redeemed_at)`.

### Points balances (for points-type rewards)

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | BIGINT | PRIMARY KEY | Auto-increment. |
| `issuer_id` | VARCHAR(255) | | |
| `user_id` | VARCHAR(255) | | |
| `balance` | BIGINT | | Current points. |
| `updated_at` | TIMESTAMP | | |

**Indexes:** `UNIQUE (issuer_id, user_id)`.

### Events (for webhooks and analytics)

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Event ID. |
| `issuer_id` | VARCHAR(255) | | |
| `type` | VARCHAR(100) | | e.g. `reward.distributed`, `reward.redeemed`. |
| `payload` | JSONB | | |
| `created_at` | TIMESTAMP | | |

**Indexes:** `INDEX (issuer_id, type, created_at)`.

---

## Potential Deep Dives

### 1. Idempotency for distribution and redemption

**Problem:** Retries or duplicate API calls must not create duplicate rewards or double-redemptions.

**Approach:**

- Client sends **Idempotency-Key** on **POST /v1/rewards/distribute** and **POST /v1/rewards/{id}/redeem**.
- Server: before creating reward or redemption, **look up** (issuer_id, idempotency_key). If response stored, **return cached response**.
- For distribution: scope key to (issuer_id, user_id, campaign_id) or explicit key. Store response with TTL (e.g. 24 hours).
- For redemption: one key per redemption attempt; prevents double-redemption even with retries.

**Why it matters:** Prevents over-issuance and double-redemption; essential for trust and cost control.

---

### 2. Segment evaluation at scale

**Problem:** Campaign targets millions of users; we must efficiently produce the distribution queue without scanning full user table.

**Approach:**

- **Pre-computed segments:** Maintain segment membership tables (e.g. `segment_eu_new_users`) updated via CDC or batch jobs when users are created or attributes change.
- **Streaming evaluation:** For event-driven segments (e.g. "user signed up"), emit events to queue; workers evaluate segment rules and enqueue distribution.
- **Sharding:** Segment tables sharded by issuer_id; distribution queue partitioned by campaign_id or user_id range for parallel processing.
- **Rate limiting:** Throttle distribution per campaign to avoid overwhelming delivery channels.

**Why it matters:** Enables large campaigns (millions of users) without blocking or timeouts.

---

### 3. Multi-channel delivery and reliability

**Problem:** Email, SMS, push can fail (bounce, carrier rejection, device offline). We need reliable delivery and status tracking.

**Approach:**

- **Delivery queue:** Each reward has a delivery task in a queue (e.g. SQS, Kafka). Workers pull and call channel-specific adapters (SendGrid, Twilio, FCM).
- **Retries:** Exponential backoff; dead-letter queue for permanent failures. Track `delivery_status`: `pending`, `sent`, `delivered`, `failed`.
- **Webhooks from providers:** Email/SMS providers send delivery status (delivered, bounced); update reward and emit internal event.
- **Fallback:** Optional secondary channel (e.g. email if push fails).

**Why it matters:** Ensures users receive rewards; status enables analytics and support.

---

### 4. Global compliance and data residency

**Problem:** GDPR, CCPA, and regional laws require data minimization, right to deletion, and data residency (e.g. EU data in EU).

**Approach:**

- **User data routing:** Store user region; route reads/writes to region-specific DB or partition (e.g. EU users in EU cluster).
- **Right to deletion:** On delete request, anonymize or purge user data; cascade to rewards (anonymize user_id, retain aggregate stats for compliance).
- **Consent:** Track consent for SMS, email, push per user; only deliver to consented channels.
- **Gift card laws:** Some regions require no expiry or minimum validity; enforce in reward creation logic.

**Why it matters:** Legal compliance; avoids fines and enables global expansion.

---

### 5. Fraud prevention

**Problem:** Bad actors create fake accounts, use bots to claim rewards, or resell codes at scale.

**Approach:**

- **Rate limits:** Per user (e.g. max 5 rewards per day from same campaign), per device, per IP.
- **Device fingerprinting:** Collect device_id, IP; flag multiple accounts from same device.
- **Anomaly detection:** ML model on distribution patterns (e.g. bulk claims from same IP range, unusual velocity).
- **Code security:** Voucher codes should be unguessable (e.g. 16+ char random); no sequential patterns.
- **Redemption validation:** Validate at redemption (order amount, currency, single-use) to catch misuse.

**Why it matters:** Protects issuer cost; maintains reward program integrity.

---
