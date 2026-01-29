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
