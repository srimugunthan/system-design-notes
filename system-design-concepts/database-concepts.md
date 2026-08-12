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

  --

  ### **Databases and Data Management**

- **SQL/Relational Databases (RDBMS)** [07:56]
    - Organizes data into rows and tables (e.g., MySQL, PostgreSQL).
    - They are usually **ACID compliant** [08:28], providing:
        - **Durability**: Data is stored on disk and persists after a machine restarts.
        - **Isolation**: Concurrent transactions don't interfere with each other.
        - **Atomicity**: Every transaction is all or nothing.
        - **Consistency**: Foreign key and other constraints are enforced.
- **NoSQL/Non-Relational Databases** [08:53]
    - Created because ACID's Consistency constraint makes relational databases harder to scale.
    - NoSQL drops this constraint and the idea of relations to improve scalability.
    - Examples include key-value stores, document stores, and graph databases [09:11].
- **Sharding (Database Partitioning)** [09:26]
    - The technique of horizontally breaking up a database and distributing the data across different machines.
    - A **Shard Key** (e.g., a person's ID) is used to decide which data goes on which machine [09:39].
- **Replication (Database)** [09:46]
    - Making copies of the database to scale read operations.
    - **Leader-Follower Replication** [09:54]: Writes go to the Leader, which propagates them to Followers; reads can go to either.
    - **Leader-Leader Replication** [10:09]: Every replica can be used for reads or writes, but this can result in inconsistent data.
- **CAP Theorem** [10:20]
    - States that, given a network partition, a database can only choose to favor either **Data Consistency** or **Data Availability** [10:27].
