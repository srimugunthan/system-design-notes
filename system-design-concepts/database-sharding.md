Sharding is the process of splitting a large database across multiple smaller databases, or shards, so that no single machine handles all the load [03:03:32].

- **Core Goal:** To keep queries fast and allow the system to scale to millions of patients or users [03:04:01].
- **Common Sharding Strategies:**
    - **Range-Based Sharding:** Data is split by ranges of IDs (e.g., ID 1 to 1 million goes to one shard). It's simple but can cause hotspots if most activity clusters in one range [04:04:06].
    - **Hash-Based Sharding:** A hash function is applied to an ID (like a patient ID) to route the record to a specific shard, often using consistent hashing to balance the load. However, range queries may now have to span multiple partitions [04:04:22].
    - **Geo-Sharding:** Data is split by region (e.g., East Coast patients on one shard). This works well when queries are always scoped to a single region but creates issues if data needs to be shared across regions [04:04:38].
- **Interview Risk/Trade-off Considerations:**
    - How to minimize cross-shard queries [04:04:52].
    - What to do if one shard grows too large and requires re-sharding [04:04:57].
    - How to avoid "hot keys" from overloading a single shard [05:02].

The video can be found here: http://www.youtube.com/watch?v=V4Zam6_oZKE

=====
