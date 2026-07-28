This is **Phase 7 of System Design**: setting up defensive guardrails to prevent your application, microservices, and databases from getting overwhelmed—whether by malicious attacks or poorly written scripts.

While **Load Balancers** distribute traffic (Phase 2) and **CDNs** cache heavy files (Phase 6), **Rate Limiting** acts as the system's quota manager. It enforces strict policies on how many requests a client can make in a given timeframe.

---

## 1. Why Rate Limiting?

Without rate limits, your application is exposed to severe resource exhaustion risks:

* **Denial of Service (DoS / DDoS):** A malicious actor or botnet sends millions of automated requests per second to crash your servers.
* **Brute-Force Attacks:** Attackers script thousands of password attempts per minute against your `/login` endpoint.
* **Resource Hogging / Scraping:** Web scrapers aggressively pull data from your site, hogging CPU and database connections meant for actual users.
* **Cascading Downstream Failures:** A sudden spike in requests bypasses the cache and overwhelms your primary database, causing the entire app to crash for everyone.

---

## 2. How Rate Limiting Works

Rate limiting tracks incoming requests against a specific identifier—typically an **IP Address** (for unauthenticated users) or an **API Key / User Token** (for logged-in users).

Because checking limits must happen in sub-milliseconds before processing the actual request, request counts are stored in **Redis** (fast in-memory RAM).

```
User / Bot Request ──> API Gateway / Rate Limiter ──> Check Redis Key (e.g., "ip:192.168.1.1")
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   [ Count <= Limit ]                 [ Count > Limit ]
            │                                 │
            ▼                                 ▼
   Forward Request to                 REJECT Request Instantly
   Microservice/App                   Return HTTP 429 "Too Many Requests"

```

### The HTTP 429 Response

When a client exceeds their quota, the rate limiter drops the request at the edge (usually at the **API Gateway**) and immediately responds with:

* **HTTP Status Code:** `429 Too Many Requests`
* **Header:** `Retry-After: 60` *(tells the client to wait 60 seconds before trying again)*

---

## 3. Common Rate Limiting Algorithms

How does Redis keep track of the count? Here are the most popular strategies:

### A. Token Bucket

* Imagine a bucket that holds a maximum of $N$ tokens.
* Tokens are added to the bucket at a constant rate (e.g., 10 tokens per second).
* Every incoming request consumes 1 token.
* If the bucket is empty, the request gets blocked (`429`).
* **Why it's popular:** Allows short, sudden bursts of traffic while still enforcing a steady average rate.

### B. Fixed Window Counter

* The timeline is divided into fixed time windows (e.g., 1-minute blocks: `12:00-12:01`, `12:01-12:02`).
* Each request increments a counter in Redis for the current window.
* **Drawback:** A surge of requests right at the edge of a window boundary (e.g., 100 requests at 12:00:59 and 100 at 12:01:01) can cause a spike of double the allowed traffic in a 2-second period.

### C. Sliding Window Log / Counter

* Tracks requests dynamically over a rolling time window (e.g., the last 60 seconds from the *exact current millisecond*).
* **Why it's great:** Smooths out the boundary spikes seen in Fixed Window, making enforcement completely fair.

---

## 4. Where is Rate Limiting Placed?

Rate limiting is usually implemented in one of two places:

1. **At the API Gateway / Reverse Proxy (Recommended):** Rejects bad traffic at the outer perimeter (VPC boundary) before it ever reaches your internal microservices.
2. **At the CDN Level (Cloudflare, AWS WAF):** Filters out massive DDoS floods before they even touch your cloud infrastructure.
