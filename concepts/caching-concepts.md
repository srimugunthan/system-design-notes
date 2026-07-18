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
