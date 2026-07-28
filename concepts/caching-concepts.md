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


* **Cache Stampede (Thundering Herd Problem):** When a popular key expires or gets invalidated, hundreds of simultaneous incoming requests might experience a cache miss at the exact same millisecond—causing them all to hit the primary database at once and potentially crash it.
