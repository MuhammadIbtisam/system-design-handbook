# Twitter

## What is it?

A **real-time microblogging service** where users create short posts/messages called **tweets** (or news feeds). Users can **follow** other users. On the **home page**, each user sees a **timeline** (feed) of tweets from the people they follow. Users can **reply** to those tweets or news feeds.

---

## Clarifying Questions (Engineers Should Ask)

1. **Is it acceptable if users see stale data?**  
   (e.g., new tweet appears in feed after a few seconds vs. immediately)

2. **Should the home timeline be pull-based, push-based, or hybrid?**  
   (Affects latency, write load, and storage.)

3. **What is the peak of daily active users?**  
   **Answer for this system: 20 million DAU.**

4. **Is eventual consistency acceptable for news feeds / timelines?**  
   (Can the feed be a few seconds behind reality, or must it always be up-to-date?)

---

## Key Concepts (Notes)

### Pull-based vs push-based timeline

| Approach | What it means |
|----------|----------------|
| **Pull-based** | When a user opens their home timeline, the system **fetches** (pulls) the latest tweets from everyone they follow, then merges and sorts them. Feed is computed **on read**. |
| **Push-based** | When someone posts a tweet, the system **writes** (pushes) that tweet into the feed cache/storage of **every follower**. Feed is **pre-built on write**. When a user opens their timeline, we just read their pre-computed feed. |
| **Hybrid** | Combine both: e.g., push for active/heavy users, pull for cold or long-tail users; or push recent N tweets, pull older ones. |

### Consistency vs availability

| Term | Meaning |
|------|--------|
| **Consistency** | Every read sees the latest write. If user A posts a tweet, everyone who loads their feed sees it immediately (strong consistency). |
| **Availability** | The system always responds to requests (no downtime). Even if some nodes or regions fail, users can still read and write. |

**Trade-off (CAP):** In a distributed system you often can't have perfect consistency and perfect availability under failures. You balance:

- **Strong consistency** → risk of slower or failed responses when parts of the system are slow or down.
- **High availability** → system keeps serving, but users might briefly see **stale data** (eventual consistency).

### Eventual consistency (for feeds)

- **Eventual consistency** = Reads may not show the latest write right away; after some time (seconds/minutes), all replicas catch up and everyone sees the same data.
- For **news feeds / timelines**, eventual consistency is usually **acceptable**: a new tweet appearing in a follower's feed a few seconds late is often fine. That allows simpler, more available, and lower-latency designs (e.g., caches, async fan-out).

---


## Requirements

### Functional Requirements

1. **Create, update, or delete tweets** — Users can post new tweets and edit or delete their own tweets.
2. **Follow other users** — Users can follow (and unfollow) other users to control whose tweets appear in their timeline.
3. **See timelines** — Users can see tweets from the users they follow (home timeline).
4. **Reply to tweets** — Users can write replies to other users' tweets.

### Non-Functional Requirements

- **Scale** — Support **20 million+** daily active users.
- **Availability over consistency** — The system should prioritize **availability** (staying up and serving requests) over strong consistency; eventual consistency is acceptable.
- **Low latency** — Timelines must load with low latency (e.g., hundreds of milliseconds).

---

## Core Entities

| Entity | Description |
|--------|-------------|
| **User** | An account that can post tweets, follow others, and have a timeline. |
| **Tweet** (Post) | A single post/message created by a user. Has content, author, timestamp; can be a reply to another tweet. |
| **Follow** | The relationship “user A follows user B.” Stored as a directed edge (e.g. `follower_id` → `followee_id`). *Follower* = the user who follows; *followee* = the user being followed. |
| **Timeline** | The ordered list of tweets a user sees (e.g. home timeline = tweets from people they follow). Not necessarily a stored table—can be derived from Users + Follows + Tweets, or pre-computed/cached for performance. |

**Note:** Follower and followee are the two sides of one **Follow** relationship, not two separate entity types. The core persisted entities are **User**, **Tweet**, and **Follow**; **Timeline** is the main view/result the system serves.

---

## API

### 1. Create a tweet (post)

**`POST /tweets`**

Creates a new tweet for the authenticated user.

| Field       | Type   | Required | Description                    |
|------------|--------|----------|--------------------------------|
| `user_id`  | string | yes      | Author of the tweet.           |
| `content`  | string | yes      | Tweet text.                    |
| `reply_to` | string | no       | Tweet ID if this is a reply.   |

**Response:** The created tweet (e.g. `tweet_id`, `user_id`, `content`, `created_at`).

---

### 2. Follow (or unfollow) a user

**`POST /follow`** — Follow another user.

| Field         | Type   | Required | Description           |
|--------------|--------|----------|-----------------------|
| `follower_id`| string | yes      | User who is following.|
| `followee_id`| string | yes      | User being followed.  |

**`DELETE /follow`** — Unfollow a user.

| Field         | Type   | Required | Description           |
|--------------|--------|----------|-----------------------|
| `follower_id`| string | yes      | User who is following.|
| `followee_id`| string | yes      | User being followed.  |

**Response:** Success or error (e.g. 204 No Content on success).

---

### 3. Fetch home timeline

**`GET /timeline`**

Returns the home timeline for a user: all tweets from users they follow, ordered by time (newest first), with pagination.

| Query      | Type   | Required | Description                                      |
|------------|--------|----------|--------------------------------------------------|
| `user_id`  | string | yes      | User whose timeline to fetch.                    |
| `limit`    | int    | no       | Max number of tweets to return (e.g. 20).       |
| `cursor`   | string | no       | Pagination cursor for the next page (e.g. tweet ID or timestamp). |

**Response:** List of tweets (with author info, timestamp, etc.) and optional `next_cursor` for the next page.

---

## High-Level Design

![Twitter High-Level Design](../../images/twitter/Twitter%20HLD.png)

The diagram shows how clients, API Gateway, and backend services work together with separate stores for tweets, media, and the follow graph.

### Components

| Component | Role |
|-----------|------|
| **Clients** | Mobile and Web app; all requests enter through the API Gateway. |
| **API Gateway** | Single entry point: **routing** to the right service, **rate limiting**, **load balancing**, and **authentication**. |
| **Tweets server** | Create, update, delete tweets; stores tweet and media metadata in DB; uploads media to S3. |
| **Timeline server** | Builds and returns a user's home timeline using follow data and tweet metadata (and media from S3). |
| **User server** | Manages follower/following relationships; reads and writes the **Graph DB**. |
| **Tweets Primary DB** | Authoritative store for tweet and media metadata; syncs to replicas. |
| **Tweets REPLICA DBs** | Read replicas for tweet reads; reduce load on primary and improve read performance. |
| **Graph DB** | Stores who follows whom; used by User server and by Timeline server to know which users' tweets to fetch. |
| **AWS S3** | Stores actual media files (images, videos); Tweets server writes, Timeline server reads. |

### How the flow works

**1. Create, update, or delete a tweet**

- Request goes **Client → API Gateway** (auth, rate limit, route) **→ Tweets server**.
- Tweets server writes tweet and media **metadata** to **Tweets Primary DB**.
- If the tweet has media, Tweets server uploads files to **AWS S3**.
- Primary DB **syncs data to Tweets REPLICA DBs** so reads see the latest data.

**2. Follow or unfollow a user**

- Request goes **Client → API Gateway → User server**.
- User server updates the relationship (follower_id, followee_id) in the **Graph DB**.
- No tweet or timeline data is written here; only the social graph changes.

**3. Fetch home timeline**

- Request goes **Client → API Gateway → Timeline server**.
- Timeline server **fetches follow data** from **Graph DB** (list of users the current user follows).
- It then **fetches tweet and media metadata** for those users from **Tweets REPLICA DB** (or Primary).
- For tweets that reference media, it **fetches media** from **AWS S3**.
- Timeline server merges and orders the tweets (e.g. by time) and returns the timeline to the client.

---

## Potential Deep Dives

### Deep Dive: Scaling timeline reads (cache + fan-out on read vs fan-out on write)

We have **~20 million daily active users**. During **peak hours**, about **2–4 million users** may be active at once. If each user fetches their timeline **4–10 times** in that period, we get a very large number of **timeline read requests**. A single PostgreSQL database cannot handle this load. Below is a layered approach and why we may still need to change the read/write model.

#### 1. Read replicas

We already use **read replicas**: write to the primary, read from replicas. This spreads read traffic across multiple DB instances and helps a lot, but at 2–4M peak users with multiple timeline fetches each, **replicas alone are still not enough**.

#### 2. Connection pooling (e.g. pgBouncer)

Each application server opens DB connections. Without pooling, we can run out of connections (e.g. “too many connections”). **pgBouncer** (or similar) sits in front of PostgreSQL and **pools connections**: many app connections are multiplexed onto fewer DB connections. This avoids connection exhaustion but does **not** reduce the number of queries or the amount of work the DB does per request.

#### 3. Cache (e.g. Redis) — cache-aside for timelines

Add a **cache layer** (e.g. Redis) in front of the database:

- **On timeline request:** Check cache first (e.g. key = `timeline:{user_id}`).
- **Cache hit:** Return the timeline from cache; no DB call.
- **Cache miss:** Compute the timeline (query Graph DB for follow list, query Tweets DB for tweets, merge), **store the result in cache**, then return it.

This is **cache-aside** (or “lazy loading”). It greatly reduces DB load when cache hit rate is high. But:

- **Fan-out on read** = we still *compute* the timeline on every cache miss: fetch follow list, fetch tweets for all followed users, merge and sort. That is **expensive per request** and can cause DB spikes during peak or when many users get cache misses (e.g. after a restart or TTL expiry).

So even with replicas + pgBouncer + Redis, we can still hit limits or see latency spikes if we rely only on **fan-out on read**.

#### 4. Fan-out on read vs fan-out on write

| Approach | What it means | Pros | Cons |
|----------|----------------|------|------|
| **Fan-out on read** | When a user opens their timeline, we **fetch** tweets from everyone they follow and merge (pull model). Timeline is computed **on read**. | Simple writes (just store the tweet). No extra write load when someone has many followers. | Every timeline read (or cache miss) can be heavy: many queries, merge, sort. DB and cache can still be overwhelmed at scale. |
| **Fan-out on write** | When a user **posts** a tweet, we **push** that tweet into the timeline cache (or pre-built feed store) of **every follower**. Timeline read = just read a pre-built list. | Timeline reads are **cheap**: one cache/store read per user. DB is used mainly for writes and for building/refreshing feeds. Much better for high read traffic. | Heavy write amplification: one tweet from a user with 1M followers = 1M writes to timeline caches/stores. Celebrities and viral tweets are expensive. |

#### 5. How to solve the problem: hybrid or fan-out on write

- **Fan-out on read only:** Replicas + pgBouncer + Redis (cache-aside) help, but on cache miss we still do expensive fan-out on read, so we can still hit DB limits or latency issues at peak.
- **Fan-out on write:** Pre-build (or update) each user’s timeline when tweets are published. Reads become a single read from cache or a feed store. This **reduces read load on the DB dramatically** and fits systems where timeline reads dominate.
- **Hybrid** is common in practice:
  - **Fan-out on write** for “normal” users (e.g. followers below a threshold): their followers’ timelines are updated when they tweet.
  - **Fan-out on read** (or lazy computation) for users with **very large follower counts** (celebrities, bots): we don’t push to millions of timelines on one tweet; instead we merge their tweets when building the timeline.

So: **PostgreSQL alone is not enough**; we add **replicas**, **pgBouncer**, and **Redis cache**. To go further and avoid overload and latency spikes from fan-out on read, we introduce **fan-out on write** (or a **hybrid** of fan-out on write for most users and fan-out on read for the heaviest accounts).

---

### Deep Dive: Serving media (avoid server proxy; use direct URLs + CDN)

In the high-level design we had the Timeline server (or Tweets server) **fetch media from AWS S3** and then **return it to the client**. That approach is simple but has serious drawbacks. A better approach is to let the **client fetch media directly** and to put a **CDN** in front of storage so users worldwide get low latency.

#### Why the naive approach is bad

| Issue | What happens |
|-------|----------------|
| **Server as proxy** | Every image/video request goes: Client → Our server → S3 → Our server → Client. Our server handles every byte of media traffic. |
| **Wasted bandwidth and CPU** | Our servers download from S3 and upload to the client. We pay for double bandwidth and burn CPU and connections. |
| **Single point of failure** | Media traffic goes through our app; if our servers are overloaded or down, media breaks too. |
| **No global optimization** | All media is served from one place (e.g. one S3 region). Users far from that region see high latency. |

#### Better approach: direct URLs + CDN

**1. Don’t proxy media through the server**

- Store media in **S3** (or similar). In the API response (tweet object), return a **direct URL** to the media (e.g. a signed S3 URL or, better, a **CDN URL**).
- The **client** (browser/app) fetches the image or video **directly** from that URL. Our server is not in the path for the actual bytes.
- Result: no media traffic through our app servers, lower load and simpler scaling.

**2. Put a CDN in front of media**

- **Problem:** If the media URL points straight at S3 in one region, users in other regions still have to hit that region → **slower** and S3 gets all the traffic.
- **Solution:** Put a **CDN** (e.g. CloudFront, Cloudflare) in front of S3. The public media URL is a **CDN URL** (e.g. `https://cdn.example.com/media/xyz.jpg`).
- **Flow:** Client requests media → **CDN edge** (nearest to the user). On **cache miss**, the edge fetches from S3, caches it, and serves the client. On **cache hit**, the edge serves from cache; the request **never reaches S3 or our server**.
- **Benefits:** Lower latency for users in all regions, less load on S3, and less bandwidth cost than serving everything from a single origin.

#### Summary

| Approach | Flow | Issue |
|----------|------|--------|
| **Naive** | Client → Our server → S3 → Our server → Client | Server proxies all media; wasteful and doesn’t scale. |
| **Direct S3 URL** | Client → S3 (direct) | No server proxy, but single region → slow for distant users. |
| **Direct CDN URL** | Client → CDN edge (cache hit: edge; miss: edge → S3) | Best: no server in the path, fast globally, S3 offloaded. |

**Recommendation:** Store media in S3; expose **CDN URLs** in API responses; let the client load media directly from the CDN. Our servers only deal with metadata and redirects; media is served from the edge.

---

## Twitter system design — full diagram

![Twitter System Design](../../images/twitter/Twitter%20System%20Design.png)

---

