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
