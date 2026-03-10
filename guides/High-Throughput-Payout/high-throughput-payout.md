# High-Throughput Payout System

## What is it?

A **high-throughput payout system** that processes **millions of payouts per day** — moving funds from a platform balance to recipients' bank accounts, cards, or wallets. Typical use cases: **gig economy** (driver/worker payouts), **marketplaces** (seller disbursements), **affiliate programs**, **payroll**, and **creator payouts**. The system must support **batch and real-time** payouts, **multiple payout methods** (ACH, wire, instant transfer, card, wallet), **idempotency**, **reconciliation** with bank files, **failure handling** (retries, partial failures), and **high availability** while guaranteeing **no double-payouts** and **no lost funds**.

---

## Clarifying Questions (Engineers Should Ask)

### Your questions (and why they matter)

**1. Who are the primary users?**

- **Platform operators** — Initiate payouts (batch or API), configure schedules, monitor status, handle exceptions.
- **Recipients (payees)** — Workers, sellers, affiliates; receive funds in bank/card/wallet.
- Clarifies auth (API keys, admin vs. automated), and whether recipients need self-service (request payout, view history).

**2. What payout methods do we need to support?**

| Method | Description | Latency | Cost | Use case |
|--------|-------------|---------|------|----------|
| **ACH (US)** | Batch bank transfer via NACHA file | 1–3 business days | Low | Standard payouts |
| **SEPA (EU)** | Batch bank transfer | 1–2 business days | Low | EU payouts |
| **Wire** | Same-day bank transfer | Same day | High | Large amounts |
| **Instant transfer** | Real-time (e.g. Visa Direct, RTP) | Seconds | Higher | Gig workers, urgent |
| **Push to card** | Load to debit card | Seconds–minutes | Medium | Unbanked, instant |
| **Wallet** | PayPal, Venmo, etc. | Minutes | Variable | Consumer preference |

- Drives integration complexity (NACHA file format, bank APIs, card networks) and queue/batch design.

**3. Batch vs. real-time payouts?**

- **Batch:** Platform accumulates payouts; submits daily (or scheduled) file to bank. Lower cost, predictable load. Example: seller disbursements every Tuesday.
- **Real-time:** Payout on demand (e.g. driver cashes out after shift). Higher throughput requirement, lower latency.
- Most systems support **both**; design must handle mixed workload.

**4. What scale (payouts per second/day)?**

- **10K/day** — Single region, simple queue.
- **100K/day** — Sharding, multiple workers, batch optimization.
- **1M+/day** — Partitioning by method/region, parallel batch generation, dedicated bank connections.
- Drives queue partitioning, worker count, and DB sharding.

**5. How should the system behave for money movement vs. reads? (CAP)**

| Operation | What we need | CAP / trade-off |
|-----------|--------------|------------------|
| **Create payout, debit balance, submit to bank** | No double-payout; no lost funds; balance correctness. | **Strong consistency.** Idempotency; ledger/transactional guarantees. |
| **List payouts, get status, dashboards** | Operators view history; eventual consistency OK. | **Availability** and low latency. Read replicas, cache. |

**6. What about duplicate requests (e.g. retry after timeout)?**

- **Idempotency:** Same request (same idempotency key) must not create two payouts. Essential for API and batch ingestion.

---

### Other questions worth asking in an interview

- **Reconciliation:** Do we need to match bank returns (rejects, NSF) back to payouts? — Affects status machine and exception handling.
- **Multi-currency:** Payouts in USD, EUR, etc.? — Affects FX, routing, and bank selection.
- **Compliance:** KYC, AML, sanctions screening? — Affects pre-payout validation pipeline.
- **Scheduling:** Cron-based batch windows? Cutoff times for same-day? — Affects job design.

---

## Requirements

### Functional Requirements

1. **Create payout** — Platform can create a single payout (API) or bulk payouts (batch upload). Payout has amount, currency, recipient (bank/card/wallet), and optional reference.
2. **Schedule payouts** — Support scheduled batch payouts (e.g. daily at 2 AM) and on-demand (real-time) requests.
3. **Payout execution** — Debit platform/merchant balance, submit to bank/card network; track status (pending, processing, completed, failed).
4. **Reconciliation** — Process bank return files (NACHA returns, SEPA rejections); update payout status; support manual resolution.
5. **List and retrieve** — Operators can list payouts (filter by status, date, recipient), get single payout, and view batch summary.

### Non-Functional Requirements

- **Throughput** — Process **100K–1M+ payouts per day**; peak bursts (e.g. end-of-day batch) without backlog buildup.
- **Consistency for money** — No double-payout; no lost funds; ledger/balance correctness. Idempotent APIs.
- **Latency** — API response (create payout) **< 500 ms**; real-time payouts reach bank **< 30 s** where supported.
- **Availability** — 99.9%+; graceful degradation if bank/partner is down (queue, retry).
- **Reconciliation** — Automated matching of bank returns to payouts; alerting on mismatches.
- **Auditability** — Immutable log of all payout attempts; support compliance and dispute resolution.

---

## Core Entities

| Entity | Description |
|--------|-------------|
| **Account** | Platform or merchant with a balance. Source of funds for payouts. |
| **Recipient** | Payee with linked payout method (bank account, card, wallet). Has verification status. |
| **Payout** | Single payout instruction: amount, currency, recipient, method, status. |
| **PayoutBatch** | Group of payouts submitted together (e.g. one NACHA file). Has batch-level status. |
| **Ledger** | Immutable record of balance changes (debit for payout, credit for funding). |
| **BankReturn** | Record of a failed/rejected payout from bank (e.g. NSF, invalid account). |
| **Event** | Immutable event for webhooks and audit (`payout.created`, `payout.completed`, `payout.failed`). |

---

## API

### 1. Create payout (single)

**`POST /v1/payouts`**

Creates a single payout. Idempotent with `Idempotency-Key`.

| Header / Body | Type | Required | Description |
|---------------|------|----------|-------------|
| `Idempotency-Key` | string | recommended | Prevents duplicate payouts. |
| `amount` | int | yes | Amount in smallest currency unit (e.g. cents). |
| `currency` | string | yes | e.g. `usd`. |
| `recipient_id` | string | yes | Recipient with linked bank/card. |
| `method` | string | no | Override: `ach`, `wire`, `instant`, `card`. Default from recipient. |
| `reference` | string | no | External reference (e.g. order ID). |
| `schedule_for` | string | no | ISO timestamp for future payout; omit for immediate. |

**Response:** Payout object (`id`, `amount`, `status`, `estimated_arrival`).

---

### 2. Create payouts (bulk)

**`POST /v1/payouts/bulk`**

Creates multiple payouts. Idempotent with `Idempotency-Key`; each row can have its own idempotency key for partial retry.

| Header / Body | Type | Required | Description |
|---------------|------|----------|-------------|
| `Idempotency-Key` | string | recommended | Batch-level key. |
| `payouts` | array | yes | `[{ "idempotency_key": "...", "amount": 1000, "recipient_id": "...", "reference": "..." }]`. |

**Response:** `{ "batch_id": "...", "payouts": [...], "accepted": 100, "rejected": 0 }`.

---

### 3. Get payout

**`GET /v1/payouts/{payout_id}`**

Returns a single payout with status and failure reason if applicable.

---

### 4. List payouts

**`GET /v1/payouts`**

| Query | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | no | `pending`, `processing`, `completed`, `failed`. |
| `batch_id` | string | no | Filter by batch. |
| `recipient_id` | string | no | Filter by recipient. |
| `created_after` | string | no | ISO timestamp. |
| `limit` | int | no | Default 20. |
| `cursor` | string | no | Pagination. |

**Response:** List of Payout objects and `next_cursor`.

---

### 5. Webhooks (outgoing)

Platform registers webhook endpoint. System sends **HTTP POST** for events (`payout.completed`, `payout.failed`, `payout.returned`). At-least-once delivery; idempotent handling by event ID.

---

## High-Level Design

### 1. Payout creation (write path)

- Client sends **POST /v1/payouts** (or bulk) with `Idempotency-Key`.
- **API Gateway** validates auth, checks idempotency (return cached response if seen).
- **Payout Service** validates (balance, recipient, amount limits), creates **Payout** record, writes **Ledger** (debit balance, credit "payout in flight"), and enqueues to **Payout Queue** (partitioned by method/region).
- Response returns Payout with status `pending`.

### 2. Payout execution (async workers)

- **Payout Workers** consume from queue. Each worker handles a partition (e.g. ACH US, SEPA EU, instant).
- **ACH/SEPA:** Workers aggregate payouts into **batch files** (NACHA, SEPA XML). At cutoff or batch size, submit file to **Bank Gateway** (SFTP, API). Create **PayoutBatch** record; mark payouts as `processing`.
- **Instant/Wire:** Workers call bank/card API per payout (or small batch). Update status to `completed` or `failed`.
- **Bank Gateway** returns confirmation or rejection; workers update Payout status, write Ledger (clear "in flight" on success; credit back on failure).

### 3. Reconciliation (bank returns)

- Bank sends **return file** (e.g. NACHA return codes: NSF, invalid account). **Reconciliation Service** ingests file, matches to Payout by trace number, creates **BankReturn** record, updates Payout to `failed`, credits balance back in Ledger.
- Alerting for unmatched returns; manual resolution queue.

### 4. Read path

- **List/Get payouts:** Read from primary or replicas; cache hot data. Pagination via cursor.
- **Balance:** Derived from Ledger; can cache with short TTL.

### 5. Throughput optimizations

- **Queue partitioning:** By payout method and region; parallel workers per partition.
- **Batch aggregation:** Accumulate ACH/SEPA payouts; flush at size (e.g. 10K) or time (e.g. every 15 min).
- **DB sharding:** Payouts table sharded by `account_id` or `created_at` range.
- **Async all the way:** Creation returns quickly; execution is fully async.

---

## Database Schema

### Accounts

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Account ID. |
| `balance` | BIGINT | | Available balance (derived from Ledger in practice). |
| `currency` | VARCHAR(3) | | Default currency. |
| `created_at` | TIMESTAMP | | |

### Recipients

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Recipient ID. |
| `account_id` | VARCHAR(255) | FOREIGN KEY | |
| `type` | VARCHAR(50) | | `bank_account`, `card`, `wallet`. |
| `encrypted_details` | TEXT | | Bank account hash, last4, etc. (PCI-sensitive). |
| `method` | VARCHAR(50) | | `ach`, `wire`, `instant`, `card`. |
| `verified` | BOOLEAN | | KYC/verification status. |
| `created_at` | TIMESTAMP | | |

**Indexes:** `INDEX (account_id)`.

### Payouts

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Payout ID. |
| `account_id` | VARCHAR(255) | FOREIGN KEY | |
| `recipient_id` | VARCHAR(255) | FOREIGN KEY | |
| `batch_id` | VARCHAR(255) | FOREIGN KEY | Null for real-time; set when batched. |
| `amount` | BIGINT | | In smallest unit. |
| `currency` | VARCHAR(3) | | |
| `method` | VARCHAR(50) | | `ach`, `wire`, `instant`, `card`. |
| `status` | VARCHAR(50) | | `pending`, `processing`, `completed`, `failed`, `returned`. |
| `failure_code` | VARCHAR(50) | | e.g. `insufficient_funds`, `invalid_account`. |
| `trace_number` | VARCHAR(100) | | Bank trace/transaction ID. |
| `idempotency_key` | VARCHAR(255) | UNIQUE | Per account scope. |
| `reference` | VARCHAR(255) | | External reference. |
| `created_at` | TIMESTAMP | | |
| `completed_at` | TIMESTAMP | | |

**Indexes:** `INDEX (account_id, status, created_at)`, `INDEX (batch_id)`, `INDEX (account_id, idempotency_key)`, `INDEX (trace_number)` for reconciliation.

### PayoutBatches

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Batch ID. |
| `account_id` | VARCHAR(255) | FOREIGN KEY | |
| `method` | VARCHAR(50) | | `ach`, `sepa`. |
| `status` | VARCHAR(50) | | `pending`, `submitted`, `processed`, `partial_failure`. |
| `file_path` | VARCHAR(500) | | Stored batch file path. |
| `payout_count` | INT | | |
| `total_amount` | BIGINT | | |
| `created_at` | TIMESTAMP | | |
| `submitted_at` | TIMESTAMP | | |

**Indexes:** `INDEX (account_id, created_at)`.

### Ledger (double-entry)

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | BIGINT | PRIMARY KEY | Auto-increment. |
| `account_id` | VARCHAR(255) | | |
| `type` | VARCHAR(50) | | `payout`, `payout_reversal`, `funding`. |
| `reference_id` | VARCHAR(255) | | Payout ID. |
| `amount` | BIGINT | | Negative = debit, positive = credit. |
| `currency` | VARCHAR(3) | | |
| `created_at` | TIMESTAMP | | |

**Indexes:** `INDEX (account_id, created_at)`. Balance = SUM(amount) per account.

### BankReturns

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | |
| `payout_id` | VARCHAR(255) | FOREIGN KEY | |
| `return_code` | VARCHAR(20) | | e.g. NACHA R01 (NSF). |
| `reason` | TEXT | | Human-readable. |
| `received_at` | TIMESTAMP | | |
| `processed_at` | TIMESTAMP | | |

**Indexes:** `INDEX (payout_id)`.

### Events

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | VARCHAR(255) | PRIMARY KEY | Event ID. |
| `type` | VARCHAR(100) | | e.g. `payout.completed`. |
| `payload` | JSONB | | |
| `created_at` | TIMESTAMP | | |

**Indexes:** `INDEX (type, created_at)`.

---

## Potential Deep Dives

### 1. Idempotency for payout APIs

**Problem:** Retries or duplicate API calls must not create two payouts for the same intent.

**Approach:**

- Client sends **Idempotency-Key** on **POST /v1/payouts** and **POST /v1/payouts/bulk**.
- Server: before creating payout, **look up** (account_id, idempotency_key). If response stored, **return cached response** (same payout ID, status).
- Store response with TTL (e.g. 24 hours). For bulk: support per-row idempotency keys so partial retries don't duplicate accepted rows.
- Key scope: per account; same key across accounts can be independent.

**Why it matters:** Prevents double-payout; critical for trust and cost.

---

### 2. Batch file generation at scale (NACHA/SEPA)

**Problem:** Generate NACHA (ACH) or SEPA files for 100K+ payouts without blocking workers or running out of memory.

**Approach:**

- **Streaming generation:** Don't load all payouts into memory. Query payouts in chunks (e.g. 1K) ordered by id; stream to file as you iterate.
- **Parallel batches:** Partition pending payouts by account or region; each worker generates its own batch file. Cap batch size (e.g. 50K entries per file) for bank limits.
- **File format:** NACHA has fixed-width fields; use a library or template. Validate before submit.
- **Deduplication:** Ensure each payout appears in exactly one batch; use DB transaction or lock when assigning `batch_id`.

**Why it matters:** Enables high-volume ACH/SEPA without OOM or single-worker bottleneck.

---

### 3. Queue design for throughput

**Problem:** Process 100K+ payouts per day with predictable latency and no backlog buildup.

**Approach:**

- **Partitioned queue:** Kafka or SQS with partitions by (method, region) or (account_id hash). Enables parallel consumers.
- **Consumer scaling:** One consumer per partition; scale partitions with load. For batch methods, consumers can aggregate before flush.
- **Priority:** Separate queues for instant (high priority) vs. batch (normal). Or use delay/priority in single queue.
- **Backpressure:** If bank is slow, slow down consumption (don't overwhelm downstream). Circuit breaker on repeated bank errors.
- **Dead-letter queue:** Failed payouts (after retries) go to DLQ for manual review; don't block main queue.

**Why it matters:** Throughput and latency SLAs depend on queue and worker design.

---

### 4. Reconciliation and failure handling

**Problem:** Bank returns rejections (NSF, invalid account, closed account). We must credit balance back and update status; support manual resolution.

**Approach:**

- **Return file ingestion:** Bank sends NACHA return file or SEPA pain.002. **Reconciliation worker** parses file, extracts return codes and trace numbers.
- **Matching:** Look up Payout by `trace_number`. If found: create BankReturn, update Payout to `failed`, write Ledger (credit balance), emit `payout.returned` event.
- **Unmatched returns:** Log and alert; manual process to match or create adjustment.
- **Partial batch failure:** If batch has mixed success/failure, update each Payout individually; batch status = `partial_failure`.
- **Retry policy:** Transient failures (timeout, 5xx) → retry with backoff. Permanent (invalid account) → fail immediately, no retry.

**Why it matters:** Ensures balance correctness and supports dispute resolution.

---

### 5. Balance and ledger consistency

**Problem:** Must never double-payout or lose funds. Balance must always reconcile with Ledger.

**Approach:**

- **Ledger as source of truth:** Every payout debits balance (or "payout in flight" sub-account); every completion or reversal credits appropriately. **Balance = SUM(ledger entries)** per account.
- **Transactional writes:** Create Payout + Ledger entry in same DB transaction. Enqueue to worker only after commit.
- **Idempotent workers:** When worker processes payout, check status first. If already `completed` or `failed`, skip (handles duplicate queue delivery).
- **Reconciliation jobs:** Periodic job sums Ledger per account, compares to cached balance; alerts on drift.

**Why it matters:** Financial correctness; audit and compliance depend on it.

---
