- **Databases:** The backbone for storing and managing data efficiently.
    - **SQL (Relational):** Stores data in tables with a strict predefined schema, follows ACID properties, and is ideal for strong consistency (e.g., banking systems).
    - **NoSQL (Non-Relational):** Designed for high scalability and performance, doesn't require a fixed schema, and is optimized for large-scale distributed data.


### Database Optimization

- **Database Indexing:** A super-efficient lookup table that speeds up database ***read*** queries by allowing the database to quickly locate data without scanning the entire table.
- **Replication:** Creating copies of the database across multiple servers. A single **Primary Replica** handles *writes*, and multiple **Read Replicas** handle *reads*, improving performance and availability.
- **Sharding (Horizontal Partitioning):** Splitting a large database by rows into smaller, manageable pieces (**shards**) and distributing them across multiple servers to handle huge amounts of data and reduce database load.
- **Vertical Partitioning:** Splitting a database by columns (e.g., separating user profile details from login history) to improve query performance by scanning only relevant fields.
- **Caching (Cache-Aside Pattern):** Storing frequently accessed data in fast in-memory storage. The application checks the cache first (cache hit is instant); if not found, it retrieves data from the database, stores it in the cache, and returns it.
- **Denormalization:** Combining related data into a single table, even with some duplication, to reduce the need for slow ***join*** operations and improve read performance in read-heavy applications.
- **CAP Theorem:** States that a distributed system can only guarantee two out of three properties at any given time: **Consistency**, **Availability**, and **Partition Tolerance**. Since network failures are inevitable, you must choose between Consistency or Availability.
