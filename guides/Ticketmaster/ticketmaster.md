# Ticketmaster

## What is it?

A **ticket booking and sales system** (like Ticketmaster or Eventbrite) where users can **discover events** (concerts, sports, etc.), **view availability** (e.g. seats or general admission), **reserve or purchase tickets**, and receive confirmations. The system must handle **limited inventory**, avoid **overselling**, and support **high traffic** during popular onsales.

---

## Clarifying Questions (Engineers Should Ask)

### Your questions (and why they matter)

**1. Who is the primary user?**

- Clarifies whether we're designing for **end users** (customers buying tickets), **venue/organizer staff** (creating events, managing inventory), or **both**. Affects auth, roles, and which flows we prioritize.

**2. What are the major functionalities we need to cover?**

- **Answer for this system:**
  - User can **search** for events (e.g. by location, date, artist, category).
  - User can **view** events (details, venue, date, available seats/sections).
  - User can **book** a ticket for an event (select seats or quantity, reserve, pay, get confirmation).

- Good question: it scopes the MVP and avoids over-designing.

**3. How many daily active users?**

- Defines scale (e.g. 100K vs 10M DAU) and drives decisions on caching, DB sharding, and rate limits. Get a number or range from the interviewer.

**4. How should the system behave when booking a ticket vs when searching/viewing events? (CAP: consistency vs availability)**

| Operation | What we need | CAP / trade-off |
|-----------|--------------|------------------|
| **Booking a ticket** | We must **not oversell**. When a user books, inventory must be decremented correctly and no two users can get the same seat (or same last unit). Reads and writes must see **consistent** inventory. | Prefer **consistency** (strong or linearizable for inventory). We may accept slightly slower or failed bookings if we can't guarantee correctness, rather than double-selling. |
| **Search / view events** | Users see events and approximate availability. It's OK if the count is slightly stale (e.g. "a few left" vs exact number). | Prefer **availability** and **low latency**. We can serve from cache or read replicas; eventual consistency is acceptable. |

So: **booking = consistency-critical** (inventory, payments); **search/view = availability and latency** (eventual consistency OK). This is a good question because it forces the design to treat the two paths differently (e.g. strong consistency for reservation flow, cache/replicas for discovery).

---

### Other questions worth asking in an interview

- **Inventory model:** Do we have **reserved seats** (e.g. section, row, seat number) or **general admission** (capacity count only)? Affects schema and locking.
- **Peak traffic:** What happens during **flash sales** (e.g. 1M users for 10K tickets)? Do we need strict rate limiting?
- **Concurrency:** Can two users book the same seat at the same time? How do we prevent it (locks, optimistic locking, reserved inventory)?
- **Geographic scope:** Single region or global? Affects latency, data residency, and DB/cache placement.
---

## Requirements

### Functional Requirements


1. **View events** — User can view event details (e.g. venue, date, time, available seats or capacity).
2. **Search for events** — User can search for events (e.g. by location, date, artist, or category).
3. **Book an event / book a seat** — User can book a ticket (or reserve a seat) for an event.

### Non-Functional Requirements

- **Availability vs consistency (by operation):**
  - **Search and view events** — **High availability**. Users should always be able to browse and see events; we can tolerate slightly stale data (e.g. cache, read replicas).
  - **Book an event** — **High consistency**. We must not allow two bookings for the same seat; inventory must be correct and no double-selling.
- **Handle popular events** — The system must cope with **flash sales** and high demand (e.g. many users trying to book the same event at once).
- **Low latency** — Operations (e.g. search, view, book) should respond in **under 500 ms** where possible.
- **Read-heavy workload** — Expect roughly **100 reads per 1 write** (many users searching and viewing; fewer actually booking). Design for read scaling (caching, replicas) while keeping writes consistent.

---

## Core Entities

| Entity | Description |
|--------|-------------|
| **User** | A customer (or account) who searches for events, views details, and makes bookings. |
| **Event** | A single occurrence (e.g. concert, game) at a venue on a date/time. Has one or more performers and links to a venue. |
| **Performer** | Artist, team, or act associated with an event (e.g. band, sports team). |
| **Venue** | Place where the event is held (e.g. stadium, hall). Has capacity, sections, and/or seat layout. |
| **Ticket** | A sellable unit for an event (e.g. one seat or one general-admission slot). Tied to event and optionally to a seat/section. |
| **Booking** | A user’s reservation or purchase of one or more tickets for an event. Links user, event, and tickets; tracks status (e.g. held, confirmed, cancelled). |

---

## API

### 1. Get event by ID

**`GET /events/{event_id}`**

Returns the full details of a single event.

| Path / query | Type   | Required | Description        |
|--------------|--------|----------|--------------------|
| `event_id`   | string | yes      | Unique event ID.   |

**Response:** Event object (e.g. name, venue, date/time, performers, available seats or capacity, pricing).

---

### 2. Search events

**`GET /events/search`**

Returns a list of events matching the search criteria.

| Query        | Type   | Required | Description                          |
|--------------|--------|----------|--------------------------------------|
| `start_date` | string | no       | Filter events on or after this date. |
| `end_date`   | string | no       | Filter events on or before this date. |
| `venue_id`   | string | no       | Filter by venue.                     |
| `performer_id` | string | no     | Filter by performer.                  |
| `location`   | string | no       | City, region, or location.            |
| `limit`      | int    | no       | Max results to return (e.g. 20).     |
| `cursor`     | string | no       | Pagination cursor.                    |

**Response:** List of event summaries and optional `next_cursor` for pagination.

---

### 3. Make a booking

**`POST /bookings`**

Creates a booking (reserve or purchase tickets) for an event.

| Field       | Type   | Required | Description                          |
|-------------|--------|----------|--------------------------------------|
| `user_id`   | string | yes      | User making the booking.             |
| `event_id`  | string | yes      | Event to book.                       |
| `ticket_ids`| array  | no       | Specific seat/ticket IDs (reserved seats). |
| `quantity`  | int    | no       | Number of tickets (e.g. general admission). |

**Response:** Booking object (e.g. `booking_id`, status, tickets, confirmation details).

---

---

## High-Level Design

*(Add diagram here when ready, e.g. `![Ticketmaster HLD](../../images/ticketmaster/ticketmaster-hld.png)`)*

The design covers the three functional requirements as follows.

### 1. Get event by ID (view event)

- Client sends a request to **fetch an event by event ID**.
- **Event Service** handles the request and returns **full event details** in the response, including:
  - **Event** — name, start date, end date, and other event fields.
  - **Venue** — venue details (name, address, capacity, etc.).
  - **Performer** — performer details (name, type, etc.).
  - **Ticket** — ticket details (availability, price, sections/seats if applicable).
  - Any other fields related to the event and performers.

### 2. Search events

- Client sends a request with **search terms** (e.g. start date, end date, venue, location, or multiple factors).
- **Search Service** handles the request and returns **search results** (matching events).
- For now the search service is **simple and basic** — it fulfils the search functionality. We will **enhance and deep-dive into the search service** later (e.g. full-text search, filters, ranking).

### 3. Create a booking

- Client sends a request to **create a new booking** for an event (selecting or reserving tickets).
- **Booking Service** handles the request: it creates the booking, associates it with the event and the chosen tickets, and (when payment is in scope) coordinates with the **Payment Gateway**. The booking is persisted so we maintain consistency and avoid double-booking.

---

## Database Schema

### Users

| Field       | Type         | Constraints | Description        |
|-------------|--------------|-------------|--------------------|
| `id`        | UUID         | PRIMARY KEY | Unique user ID.    |
| `email`     | VARCHAR(255) | UNIQUE      | User email.        |
| `name`      | VARCHAR(255) |             | Display name.     |
| `created_at`| TIMESTAMP    |             | Registration time. |

### Venues

| Field       | Type         | Constraints | Description           |
|-------------|--------------|-------------|-----------------------|
| `id`        | UUID         | PRIMARY KEY | Unique venue ID.      |
| `name`      | VARCHAR(255) |             | Venue name.          |
| `address`   | TEXT         |             | Full address.        |
| `city`      | VARCHAR(100) |             | City (for search).    |
| `capacity`  | INT          |             | Max capacity.        |
| `created_at`| TIMESTAMP    |             |                       |

### Performers

| Field       | Type         | Constraints | Description          |
|-------------|--------------|-------------|----------------------|
| `id`        | UUID         | PRIMARY KEY | Unique performer ID. |
| `name`      | VARCHAR(255) |             | Performer name.      |
| `type`      | VARCHAR(50)  |             | e.g. band, artist.   |
| `created_at`| TIMESTAMP    |             |                      |

### Events

| Field        | Type      | Constraints    | Description                    |
|--------------|-----------|----------------|--------------------------------|
| `id`         | UUID      | PRIMARY KEY    | Unique event ID.               |
| `venue_id`   | UUID      | FOREIGN KEY    | Venue where event is held.     |
| `name`       | VARCHAR   |                | Event name.                    |
| `start_date` | TIMESTAMP |                | Event start (for search).      |
| `end_date`   | TIMESTAMP |                | Event end.                     |
| `created_at` | TIMESTAMP |                |                                |

**Indexes:** `INDEX (venue_id)`, `INDEX (start_date)`, `INDEX (start_date, end_date)` for search.

### Event_Performers (many-to-many)

| Field         | Type | Constraints    | Description     |
|---------------|------|----------------|-----------------|
| `event_id`    | UUID | FOREIGN KEY    | Event.          |
| `performer_id`| UUID | FOREIGN KEY    | Performer.      |

**Primary key:** `(event_id, performer_id)`.

### Tickets

| Field       | Type      | Constraints | Description                                      |
|-------------|-----------|-------------|--------------------------------------------------|
| `id`        | UUID      | PRIMARY KEY | Unique ticket ID.                                |
| `event_id`  | UUID      | FOREIGN KEY | Event this ticket belongs to.                     |
| `section`   | VARCHAR   |             | Section or area (optional; null for GA).         |
| `row`       | VARCHAR   |             | Row (optional).                                  |
| `seat`      | VARCHAR   |             | Seat number (optional).                          |
| `price`     | DECIMAL   |             | Ticket price.                                    |
| `status`    | VARCHAR   |             | e.g. `available`, `reserved`, `sold`.            |
| `created_at`| TIMESTAMP |             |                                                  |

**Indexes:** `INDEX (event_id)`, `INDEX (event_id, status)` for availability checks.

### Bookings

| Field        | Type      | Constraints | Description                         |
|--------------|-----------|-------------|-------------------------------------|
| `id`         | UUID      | PRIMARY KEY | Unique booking ID.                  |
| `user_id`    | UUID      | FOREIGN KEY | User who made the booking.          |
| `event_id`   | UUID      | FOREIGN KEY | Event booked.                       |
| `status`     | VARCHAR   |             | e.g. `pending`, `confirmed`, `cancelled`. |
| `created_at` | TIMESTAMP |             |                                     |

**Indexes:** `INDEX (user_id)`, `INDEX (event_id)`.

### Booking_Tickets

| Field        | Type | Constraints | Description                    |
|--------------|------|--------------|--------------------------------|
| `booking_id` | UUID | FOREIGN KEY  | Booking.                       |
| `ticket_id`  | UUID | FOREIGN KEY  | Ticket included in booking.   |

**Primary key:** `(booking_id, ticket_id)`. Ensures a ticket is linked to at most one booking (enforce in app or with unique constraint on `ticket_id` if one ticket → one booking).

---

## Potential Deep Dives

The diagram below shows the evolved architecture for the ticketing system: **Search Service** with **Elastic Search** (and Primary DB), **Event Service** with **Cache** and Primary DB, **Booking Service** with Primary DB and **Payment Gateway**, and a **Waiting Queue** so that when load is high the API Gateway can send booking requests to the queue and the Booking Service processes them from there.

![Ticketmaster Potential Deep Dives](../../images/ticketmaster/Ticketmaster%20Deep%20Dive.png)

---

### 1. High availability for viewing events (cache)

We need **high availability** for the “view event” flow. Put **mostly static event data** (event name, event details, venue, performers, etc.) in a **cache** (e.g. Redis).

**Flow (cache-aside):**

1. On **fetch event by ID**: check the cache first (e.g. key = `event:{event_id}`).
2. **Cache hit** — Return the event from cache; no database call.
3. **Cache miss** — Fetch from the database, **store the result in the cache**, then return it to the user.

This reduces load on the DB and keeps view-event highly available and fast. Cache TTL can be set so that rare updates (e.g. event details change) eventually propagate.

---

### 2. Search for events (indexes vs full-text vs Elasticsearch)

**Problem:** Users search by multiple factors (date, venue, location, keywords). We need efficient, flexible search.

| Approach | Pros | Cons |
|----------|------|------|
| **DB indexes only** | Simple (e.g. B-tree on date, venue_id). | **Not enough** for rich or text search; indexes help filters but not full-text or fuzzy search. |
| **PostgreSQL full-text search** | Use DB extensions (e.g. `tsvector`, full-text indexes). No extra system. | Uses **more space and resources**; **SELECT** queries by search term can be **slower**. Harder to scale and tune for complex relevance. |
| **Elasticsearch / Algolia (or similar)** | **Full-text search** with good performance and relevance. **Typo tolerance** and **semantic-like** behavior (e.g. Algolia): even with spelling mistakes or missing words, users still get relevant results. Dedicated search layer that scales separately. | Extra system to run and keep in sync with the primary DB. |

**Recommendation:** For production-grade search (full-text, typo tolerance, many filters), use **Elasticsearch** or **Algolia** (or similar). Sync event data from Primary DB into the search engine; Search Service queries the search engine instead of the DB. Keep PostgreSQL for transactional data; use the search layer for discovery and search.

---

### 3. Booking: consistency when millions are booking the same event

**Problem:** For a **very popular event**, many users try to book at once. We must **never** allow two users to book the same seat (strong consistency for inventory). We also need **high throughput** — we cannot make millions of users wait in line behind a single lock.

| Approach | How it works | Why it fails at scale |
|----------|----------------|------------------------|
| **Pessimistic locking** | First user acquires a lock on the seat (or row); others **wait** until that user’s transaction completes (book or abort). | OK for **hundreds or thousands** of requests per day. For **millions** of users, everyone waits on the same lock → **very low throughput**, bad latency, and poor UX. Not acceptable for flash sales. |

**Better approach: status-based hold (pending → booked or released)**

- Do **not** hold a long-lived lock that blocks everyone else.
- Use **ticket statuses** and a **short-term hold**:
  1. When a user **starts** a booking, move the ticket from **available** to **pending** (or “in progress”) in a **single, quick DB update** (e.g. conditional update: “set status = pending where id = ? and status = available”). If the update succeeds, the user “holds” that ticket for a limited time (e.g. 10–15 minutes).
  2. **User pays** (Payment Gateway). On **payment success** → move ticket to **booked** and create the booking record. Consistency is preserved because only the user who holds the ticket can confirm.
  3. If the user **does not complete payment** within the time window (or cancels), **release the hold**: move the ticket back to **available** (e.g. background job or TTL). Another user can then book it.

**Why this works:** No long-lived lock; many users can be in “pending” on *different* tickets at the same time. Only one user can hold a given ticket at a time (enforced by the conditional update). We get **high consistency** (no double-booking) and **high throughput** (no global wait). This is the standard pattern for ticket booking and inventory under high concurrency.

---

### 4. Handling extreme traffic: queue + WebSockets (real-time queue)

**Problem:** Even with **load balancing** and **horizontal scaling**, millions of users hitting the booking flow at the same time can still overwhelm the server, DB, and payment gateway. We need a way to **smooth and control** the rate of booking attempts while keeping users informed.

**Queue alone:** We can put incoming booking requests into a **queue** (e.g. Kafka, RabbitMQ, Redis Queue). The server processes from the queue at a controlled rate. But if the client just gets "request received" and then waits with no feedback, the user has no idea where they stand or when their turn will come — poor UX and more support load.

**Better: queue + WebSockets for real-time updates**

- When load is **high**, don't let every user hit the booking API at once. Instead:
  1. **User joins the queue** — e.g. "I want to book for this event." The server adds them to a **waiting queue** and opens a **WebSocket** (or long-lived connection) with that user.
  2. **User waits in the socket** — They stay connected. The server sends **real-time updates** over the WebSocket: e.g. "You're in line", "Your position: 5,432", "Estimated wait: 2 minutes", "It's your turn — you can now submit your booking."
  3. **When it's the user's turn** — The server tells them over the WebSocket that they can proceed. The client then **sends the actual booking request** (e.g. seat selection, payment). The server processes it and responds. So we **control how many users are "active" at once** — only those whose turn it is actually call the booking API.
  4. If the user abandons or disconnects, they can be removed from the queue so the next person can move up.

**Why this helps:**

| Benefit | Explanation |
|---------|-------------|
| **Controlled throughput** | Only N users at a time (or a steady rate) get to submit a booking. The rest wait in the queue. The server, DB, and payment gateway are not overloaded. |
| **Fair ordering** | Queue is typically FIFO (or priority), so order is clear and fair. |
| **Real-time UX** | Users see **live updates** (position, wait time, "your turn") instead of a spinning loader or a generic "high demand" error. No need to poll or rely only on push notifications after the fact. |
| **Scales to more traffic** | We can handle **more users** in total — they wait in the queue and get a turn instead of everyone failing or timing out. |

**Summary:** Use **load balancing** and **horizontal scaling** as a base. When traffic is extreme (e.g. millions for one onsale), add a **waiting queue** and keep users connected via **WebSockets** so they get **real-time queue updates**. When it's their turn, they submit the booking request. This keeps the system stable and gives users a clear, live experience instead of blind waiting or late push notifications.

---
