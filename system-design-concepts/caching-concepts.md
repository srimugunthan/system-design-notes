- **Caching** [02:24]
    - The technique of creating copies of data to be refetched faster.
    - Caching occurs at multiple levels to avoid expensive operations:
        - Browser disk cache [02:32]
        - Computer memory cache [02:40]
        - CPU cache (L1, L2, L3) [02:47]
        - 
### **1. Caching**

Caching involves storing frequently accessed data closer to the user or application to reduce latency and protect the database from heavy load [00:20].

- **Core Concept:** Every system has "hot data" (data accessed repeatedly), and caching speeds up access to this data without overwhelming the underlying database [00:09].
- **Caching Locations:**
    - In-memory on the application layer (e.g., Redis or Memcache) [00:45].
    - Client-side in the user's browser [00:50].
    - At the Content Delivery Network (CDN) edge, which makes platforms like YouTube and Netflix fast worldwide [00:55].
- **The Problem of Cache Invalidation:** The main challenge is ensuring the cached data does not go stale and that outdated information is not served [01:01:01].
- **Common Write Strategies:**
    - **Write-Through Caching:** Writes go to both the database and the cache simultaneously, ensuring consistency but making writes slightly slower [01:01:17].
    - **Write-Around Caching:** Writes go directly to the database, and the cache is updated only on a subsequent read. This avoids filling the cache with data that may never be read, but the first read will be slower [01:01:26].
    - **Write-Behind Caching:** Writes go to the cache first and are then asynchronously pushed to the database. This provides super-fast writes but risks losing data if the cache fails before syncing [01:01:40].

# Metadata caching pattern
This is **Phase 5 of System Design**: placing an in-memory cache in front of your database to eliminate disk read latency and protect your primary database from getting overwhelmed by repetitive read queries.

Here is a clear breakdown of why metadata caching is necessary, how Redis fits in, and how the **Cache-Aside Pattern** works step-by-step.

---

## 1. Why Metadata Caching?

In a system handling millions of users, reading data from a relational database disk (like PostgreSQL or MySQL) introduces two major problems:

1. **Disk Latency:** Reading from a hard disk or SSD takes milliseconds, whereas reading directly from RAM takes nanoseconds—roughly 1,000 times faster.
2. **Database Bottlenecks:** Most web applications have a heavy **read-to-write ratio** (e.g., 90% reads vs 10% writes). Reading user profile information, video metadata, or permissions on every single click will quickly max out the database CPU and connections.

By storing frequently accessed, small, non-changing data (**metadata**) in an in-memory key-value store like **Redis** or **Memcached**, your application serves most read requests straight out of RAM.

---

## 2. The Cache-Aside Pattern (Lazy Loading)

The **Cache-Aside Pattern** is the most widely used caching strategy because the application manages the relationship between the cache and the database directly.

Here is how the application handles a read request:

```
                  ┌──────────────────────┐
                  │ 1. Request Metadata  │
                  └──────────┬───────────┘
                             │
                             ▼
                    /─────────────────\
                   /   2. Cache Hit?   \
                  \     (Check RAM)   /
                   \─────────────────/
                     /             \
             YES    /               \  NO
                   /                 \
                  ▼                   ▼
    ┌──────────────────┐    ┌──────────────────┐
    │  Return Data     │    │  3. Read DB      │
    │  Instantly       │    └────────┬─────────┘
    └──────────────────┘             │
                                     ▼
                            ┌──────────────────┐
                            │  4. Write to     │
                            │     Cache (RAM)  │
                            └────────┬─────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │  5. Return Data  │
                            └──────────────────┘

```

### Step-by-Step Flow

1. **Check Cache First:** The application receives a request for metadata (e.g., `user_id_101_profile`). It queries **Redis** first.
2. **Cache Hit:** If the data exists in Redis, it is returned instantly to the user. The primary database is never touched.
3. **Cache Miss:** If the key is not in Redis, the application falls back to querying the **primary database**.
4. **Populate Cache:** The application takes the record returned from the database and writes it into Redis so it is ready for the *next* request.
5. **Return Data:** The data is sent back to the user.

---

## 3. Key Considerations & Trade-Offs

While Cache-Aside is fast and resilient, it introduces two important architectural challenges:

* **Stale Data (Cache Invalidation):** If a user updates their profile in the database, the old version still exists in Redis. To prevent serving outdated data, you must either:
* Assign a **Time-To-Live (TTL)** to keys in Redis so they expire automatically after a set duration (e.g., 5 minutes).
* Explicitly delete/invalidate the cached key in Redis whenever a write/update query occurs in the database.

# CDN caching

This represents **Phase 6 of System Design**: moving heavy static content (like images, video files, CSS, and JS bundles) out of your central data center altogether and pushing it as close to the physical location of your users as possible.

While **Redis** (Phase 5) speeds up data reads *inside* your server data center, a **Content Delivery Network (CDN)** speeds up content delivery *across the physical globe*.

---

## 1. What is Edge Caching & Points of Presence (PoPs)?

Without a CDN, if your primary server or cloud storage bucket (e.g., AWS S3) is located in **Virginia, USA**, a user trying to stream a video or load an image from **Tokyo, Japan** has to send requests back and forth across underwater fiber-optic cables spanning thousands of miles.

```
WITHOUT CDN:
User (Tokyo) <═════════ 10,000 km (~200-300 ms round trip) ═════════> Origin Server (Virginia)

```

A **CDN** (like Cloudflare, CloudFront, or Fastly) solves this by using a global network of proxy servers called **Points of Presence (PoPs)** stationed in major cities worldwide.

```
WITH CDN:
User (Tokyo) <══ 10 km (2-5 ms) ══> CDN Edge Server (Tokyo) ──(Only on Cache Miss)──> Origin Server (Virginia)

```

* **Edge Servers:** CDN nodes deployed right at the "edge" of the internet, near consumer ISPs.
* **Origin Server:** Your main application backend / storage bucket where the master files live.

---

## 2. Dramatically Reduced Latency (The Math)

Network latency is limited by the speed of light in fiber optics. Every mile adds physical delay.

* **Direct Request (No CDN):** A 5MB video thumbnail loaded from Virginia to Tokyo might take **800ms – 1.5 seconds** due to multiple network hops, TCP handshakes, and round-trip physical distance.
* **Edge Request (With CDN):** The request travels a few miles to the local Tokyo Edge Server, returning the file in **10 – 30 milliseconds**.

For media-heavy apps (like YouTube, Netflix, or e-commerce sites), this reduction in latency directly impacts user retention—loading content almost instantaneously.

---

## 3. How Edge Caching Works (Origin Pull)

Edge caching uses a lazy-loading mechanism similar to the Cache-Aside pattern:

1. **User Requests Asset:** A user in London requests `[https://cdn.myapp.com/video_123.mp4](https://cdn.myapp.com/video_123.mp4)`.
2. **Edge Check:** The request routes to the nearest London CDN PoP.
3. **Cache Hit:** If the London server already has `video_123.mp4` cached, it streams it directly to the user instantly.
4. **Cache Miss (Origin Pull):** If the London server does *not* have it:
* The CDN server fetches `video_123.mp4` from your **Origin Server** in Virginia.
* It stores a copy on its local disk/RAM in London.
* It serves the file to the user.


5. **Subsequent Users:** The next 10,000 users in London requesting that same video file get served straight from the local London cache—your origin server never sees those requests.

---

## 4. Key Differences: Redis vs. CDN

| Feature | Redis Metadata Cache (Phase 5) | CDN Edge Cache (Phase 6) |
| --- | --- | --- |
| **Data Type** | Small dynamic data (User profiles, JSON, session tokens) | Large static assets (Videos, images, JS/CSS, PDFs) |
| **Location** | Inside your backend private network / VPC | Distributed globally at the internet edge |
| **Primary Goal** | Reduce primary database CPU/disk load | Reduce network distance latency and bandwidth costs |

* **Cache Stampede (Thundering Herd Problem):** When a popular key expires or gets invalidated, hundreds of simultaneous incoming requests might experience a cache miss at the exact same millisecond—causing them all to hit the primary database at once and potentially crash it.
