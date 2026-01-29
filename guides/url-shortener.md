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
