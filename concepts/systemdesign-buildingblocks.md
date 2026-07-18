### Building Blocks of a Scalable System

A typical scalable system logically includes:

- **Client Layer:** Web or mobile app sending requests [01:24].
- **CDN (Content Delivery Network):** Delivers static content (images, CSS) close to the user [01:17].
- **Load Balancer:** Distributes incoming traffic evenly across servers [01:33].
- **Application Servers:** The logic layer, usually restricted to microservices [01:37].
- **Cache Layer:** Fast memory to reduce database hits [01:40].
- **Database Layer:** Where all the data lives [01:43].
- **Message Queues:** To handle asynchronous jobs and traffic spikes [01:46].
- **Monitoring and Logging:** Essential because "what you can't measure you can't scale" [01:49].


### 3. Core Components in Detail

| **Component** | **Function** | **Key Concepts** |
| --- | --- | --- |
| **Load Balancer** | Splits traffic evenly across servers to prevent one server from dying of exhaustion [02:08]. | Uses algorithms like round-robin or least connections [02:15]. Can work at **Layer 4** (Transport) looking at IP/ports, or **Layer 7** (Application) inspecting the URL path [02:28]. |
| **Caching** | Acts as the system's short-term memory to store frequently fetched data, resulting in fast reads and happy users [02:57]. | Common strategies include **Write-Through**, **Write-Around**, and **Write-Behind** [03:13]. Tools used include **Redis** (in-memory data store with complex data types) and **Memcached** (simple, super-fast key-value cache) [03:33]. |
| **Databases** | Stores all system data. Replication improves availability and fault tolerance [04:22]. | **Sharding** is used to split a large dataset into smaller, faster chunks (e.g., A-M in one shard, N-Z in another) [04:40]. **Consistent Hashing** helps spread the load evenly across shards to avoid "hot spot" problems [04:59]. |
| **Asynchronous Processing** | Handles tasks that don't need to happen immediately (e.g., generating thumbnails after a photo upload) in the background [05:08]. | Sends background jobs to a **queue** using tools like AWS SQS or Kafka [05:27]. This keeps the main app snappy, scalable, and resilient [05:43]. |
| **Microservices** | Each service handles one specific job (e.g., authentication, payments) and is **stateless** (no session data stored locally) [06:08]. | Statelessness makes it easy to spin up or shut down instances for autoscaling [06:27]. |
| **Object Storage** | Used for storing large files like images, videos, or backups (e.g., Amazon S3) [06:57]. | Provides high durability and lets you access files from anywhere [07:13]. |
|  |  |  |
