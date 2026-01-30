# URL Shortener (TinyURL)

## What is a URL Shortener?

A URL Shortener is a service that takes a long URL and turns it into a short, unique link. When someone visits that short link, they are redirected to the original long URL. Think of services like TinyURL or bit.ly—they make long links manageable for sharing on social media, in emails, or in ads.

### Core Responsibilities

- **Generate a unique short code** for each long URL you submit
- **Persist the mapping** between short URL and original URL so redirects always work
- **Redirect users** from the short URL to the long URL with low latency
- **Optionally track analytics** (clicks, geography, device, time)
- **Handle high traffic** and stay reliable under load

### Real-World Use Cases

- **Social media** — Fits within character limits (e.g., Twitter) and keeps feeds tidy
- **Marketing and analytics** — Track which campaigns and links drive traffic
- **Cleaner, shareable links** — Easier to read, remember, and type
- **Email and SMS campaigns** — Short links look better and are easier to click

---

## Clarifying Questions (Before You Design)

When you’re in an interview or scoping the product, it helps to align on these:

- **Can different users shorten the same long URL and get different short URLs?**  
  (e.g., one short link per user vs. one global short link per destination)

- **If the destination URL changes, should existing clicks redirect to the new URL immediately?**  
  (Affects whether you update in place or treat it as a new short link.)

- **Do we need audit history for changes?**  
  (e.g., who changed what and when—impacts schema and retention.)

- **If a user deletes their account, what happens to their URLs?**  
  (Keep working, redirect to a “deleted” page, or 404?)

- **Do we always use a 301 redirect, or are there cases for 302 or 307?**  
  (301 = permanent, cacheable; 302/307 = temporary—matters for SEO and caching.)

---

## Requirements

### Functional Requirements

- **Convert original URL to short URL** — Given a long URL, the system generates a short, redirectable link.
- **Return long URL from short URL** — When a user provides a short URL, the system returns (or redirects to) the original long URL.
- **Custom alias (optional)** — Users may specify their own short code instead of an auto-generated one.
- **Expiration (optional)** — Short URLs can optionally expire after a set period.

### Non-Functional Requirements

- **Uniqueness** — Short URLs must be unique; no two links share the same short code.
- **Low latency** — Redirects and lookups should be fast (typically cache-backed).
- **Availability over consistency** — Prefer staying up and serving reads over strong consistency (e.g., eventual consistency is acceptable when needed).
- **Scalable to 100M DAU** — The system must scale to support 100 million daily active users.

---

## Core Entities

- **Original URL** — The long, destination URL that users are redirected to.
- **Short URL** — The short link (domain + short code) that maps to an original URL.
- **User** — The person or system that creates and owns short URLs (when user accounts exist).

---

## API

### POST — Shorten a URL

Creates a short URL from a long one.

**Request body:**

```json
{
  "original_url": "https://example.com/very/long/path",
  "custom_alias": "optional",
  "expiration": "optional"
}
```

| Field           | Type   | Required | Description                                      |
|----------------|--------|----------|--------------------------------------------------|
| `original_url` | string | yes      | The long URL to shorten.                         |
| `custom_alias` | string | no       | Optional custom short code (e.g. `my-link`).     |
| `expiration`   | string | no       | Optional expiry (e.g. ISO date or duration).     |

**Response:**

```json
{
  "short_url": "https://short.example.com/abc123"
}
```

| Field        | Type   | Description                          |
|-------------|--------|--------------------------------------|
| `short_url` | string | The shortened URL the user can share. |

---

### GET `/{short_code}` — Redirect to original URL

Resolves a short code and redirects the caller to the original URL.

| Item        | Description                                        |
|-------------|----------------------------------------------------|
| **Path**    | `/{short_code}` (e.g. `/abc123`)                   |
| **Response**| `302 Found` — Redirect to the original URL         |


---

## High-Level Design

![High-Level Design](../../images/url-shortener/High-Level%20Design.png)

The diagram above shows the core flow of the URL shortener system:

### 1. URL Shortening Process

When a client (mobile or web application) wants to shorten a URL:

1. **Client** sends a `POST /url` request with the original URL to the **Primary Server**
2. **Primary Server** receives the request and validates the original URL
3. **IF URL is valid**: The server generates a unique short URL (or uses a custom alias if provided) and saves the mapping to the **DB**
4. **IF URL is invalid**: The server returns an error response
5. **Response**: The server returns the short URL (e.g., `https://short.example.com/abc123`) to the client

### 2. URL Redirection Process

When a user accesses a short URL:

1. **Client** sends a `GET /{short_code}` request to the **Primary Server**
2. **Primary Server** queries the **DB** to find the original URL mapped to the short code
3. **IF mapping exists**: The server returns a `302 Redirect` to the original URL
4. **Client** is redirected to the original destination

### Why 302 Redirect Instead of 301?

- **302 (Found/Temporary)** — Allows flexibility for analytics tracking, expiration handling, and URL updates. Browsers won't permanently cache the redirect.
- **301 (Moved Permanently)** — Would cause browsers to cache the redirect permanently, bypassing the server on future requests. This prevents analytics tracking and makes it impossible to update or expire URLs.

---

## Database Schema

### URLs Table

The primary table that stores the mapping between short codes and original URLs.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `short_code` | VARCHAR(10) | PRIMARY KEY | The unique short code (e.g., `abc123`) |
| `original_url` | TEXT | NOT NULL | The destination URL to redirect to |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | When the short URL was created |
| `updated_at` | TIMESTAMP | NULL | Last modification timestamp (for audit) |
| `expiration_time` | TIMESTAMP | NULL | When the short URL expires (nullable = never expires) |
| `created_by` | UUID | FOREIGN KEY → Users | The user who created this short URL |
| `is_active` | BOOLEAN | DEFAULT TRUE | Soft delete flag to disable without removing |

**Indexes:**
- `PRIMARY KEY (short_code)` — Fast lookups for redirects
- `INDEX (created_by)` — Query all URLs by a specific user
- `INDEX (expiration_time)` — Efficiently find and clean up expired URLs

### Clicks Table (Optional — for Analytics)

If you need detailed analytics beyond a simple counter, store each click as a separate record.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | BIGINT | PRIMARY KEY, AUTO_INCREMENT | Unique click identifier |
| `short_code` | VARCHAR(10) | FOREIGN KEY → URLs | The short URL that was clicked |
| `clicked_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | When the click occurred |
| `ip_address` | VARCHAR(45) | NULL | Client IP (for geo tracking) |
| `user_agent` | TEXT | NULL | Browser/device information |
| `referrer` | TEXT | NULL | Where the click came from |

**Indexes:**
- `INDEX (short_code, clicked_at)` — Analytics queries by URL and time range

### Users Table (If User Accounts Exist)

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Unique user identifier |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | User's email address |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Account creation time |

### Schema Diagram

```
┌─────────────────────────────────────────────────────────┐
│                        URLs                             │
├─────────────────────────────────────────────────────────┤
│ short_code (PK)    VARCHAR(10)                          │
│ original_url       TEXT                                 │
│ created_at         TIMESTAMP                            │
│ updated_at         TIMESTAMP                            │
│ expiration_time    TIMESTAMP                            │
│ created_by (FK)    UUID ─────────────┐                  │
│ is_active          BOOLEAN           │                  │
└─────────────────────────────────────────────────────────┘
           │                           │
           │ 1:N                       │
           ▼                           ▼
┌─────────────────────────┐    ┌─────────────────────────┐
│        Clicks           │    │         Users           │
├─────────────────────────┤    ├─────────────────────────┤
│ id (PK)      BIGINT     │    │ id (PK)      UUID       │
│ short_code   VARCHAR(10)│    │ email        VARCHAR    │
│ clicked_at   TIMESTAMP  │    │ created_at   TIMESTAMP  │
│ ip_address   VARCHAR(45)│    └─────────────────────────┘
│ user_agent   TEXT       │
│ referrer     TEXT       │
└─────────────────────────┘
```

---
