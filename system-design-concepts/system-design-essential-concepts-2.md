
This comprehensive system design crash course breaks down core architectural principles, networking fundamentals, and database strategies necessary for scaling distributed applications.

Here is the reorganized transcript structured logically into key functional blocks.

---

## 1. Hardware Architecture (The Individual Node)

Before building large-scale systems, it is essential to understand the underlying constraints of a single machine.

* **Data Layout:** At the core, computers process bits ($0$ or $1$). $1\text{ Byte} = 8\text{ bits}$, which scales sequentially into kilobytes, megabytes, gigabytes, and terabytes.
* **Disk Storage:** Non-volatile storage (retains data without power) containing the OS, applications, and files.
* *Hard Disk Drives (HDDs):* Slower data retrieval speeds ($80\text{ to } 160\text{ MB/s}$).
* *Solid State Drives (SSDs):* Significantly faster retrieval speeds ($500\text{ to } 3,500\text{ MB/s}$).


* **Random Access Memory (RAM):** Volatile primary memory that holds data currently in use (runtime stacks, variables, data structures). It operates significantly faster than disk storage ($> 5,000\text{ MB/s}$).
* **CPU & Cache Layers:** The CPU fetches, decodes, and executes machine code instructions. To eliminate memory bottlenecks, it relies on ultra-fast, multi-tiered caches measured in megabytes:
* **L1 Cache:** Fastest access time (a few nanoseconds).
* **L2 & L3 Cache:** Intermediate safety nets checked sequentially before the CPU defaults to querying RAM.



---

## 2. Production Application Architecture

A robust high-level production footprint automates deployments, monitors health, and compartmentalizes infrastructure.

* **CI/CD Pipeline:** Automates testing and deployments from code repositories via platforms like Jenkins or GitHub Actions, moving code to production without manual intervention.
* **Traffic Management:** High volumes of user traffic are distributed evenly using load balancers and reverse proxies (e.g., Nginx) to maintain availability during traffic spikes.
* **Decoupled Storage:** Application servers do not run databases locally; they connect to dedicated external storage servers over the network.
* **Observability (Logging & Monitoring):**
* Logs are isolated on external services to prevent production disk saturation.
* Tools like `pm2` manage backend processes, while platforms like *Sentry* track real-time frontend exceptions.


* **Alerting & Debugging:**
* Anomalies trigger alerting services that pipe notifications directly into developer communication channels (e.g., Slack).
* *The Golden Rule of Debugging:* Issues are replicated and diagnosed exclusively in safe, isolated staging or test environments—never directly in production. Emergency hotfixes are rolled out as temporary patches.



---

## 3. System Design Core Pillars & Trade-offs

System design focuses on three fundamental operations: **moving data**, **storing data**, and **transforming data**. Evaluating architectural quality relies heavily on standard trade-offs.

### System Metrics

* **Availability:** Measured as a percentage of uptime.
* $99.9\%$ availability allows for $\sim 8.76$ hours of annual downtime.
* **Five Nines ($99.999\%$)** narrows allowed downtime down to just $\sim 5$ minutes per year.


* **SLO vs. SLA:**
* *Service Level Objectives (SLOs):* Internal performance goals (e.g., target a $300\text{ ms}$ response time $99.9\%$ of the time).
* *Service Level Agreements (SLAs):* Formal external contracts with customers specifying financial or legal penalties if service thresholds drop.


* **Resilience & Fault Tolerance:** Building redundancy through backup systems to eliminate single points of failure, or designing systems to degrade gracefully so core workflows survive partial outages.
* **Throughput vs. Latency:**
* *Throughput:* Amount of data or requests processed over time (Requests Per Second for servers, Queries Per Second for databases, Bytes Per Second for networks).
* *Latency:* Time taken to fulfill a single end-to-end request.
* *Trade-off:* Batching operations typically increases throughput but negatively impacts individual request latency.



### The CAP Theorem

In a distributed system, network partitions will inevitably occur. When a partition happens, you can only guarantee **two out of three** properties simultaneously:

1. **Consistency (C):** Every node returns the same, most recent data at the same time.
2. **Availability (A):** Every non-failing node returns a non-error response without a guarantee that it contains the most recent write.
3. **Partition Tolerance (P):** The system continues to operate despite arbitrary message loss or network disruptions.

> **Example:** A banking system prioritizes Consistency and Partition Tolerance (**CP**) to maintain financial accuracy, sacrificing immediate availability if certain transactions must wait for network resolution.

---

## 4. Networking Basics & Application Layer Protocols

Computers network by encapsulating data into packets governed by specialized rules across different protocol layers.

* **IP Addressing:** Devices are identified via 32-bit IPv4 addresses ($\sim 4 \text{ billion}$ limits) or modern 128-bit IPv6 addresses. Packets route via an IP header containing sender and receiver coordinates.
* **Transport Layer (TCP vs. UDP):**
* **TCP:** Connection-oriented, reliable protocol utilizing sequence numbers and a **three-way handshake**. Guarantees in-order, complete packet delivery.
* **UDP:** Connectionless, faster protocol without delivery guarantees. Preferred for time-sensitive applications (video calls, live streaming) where minimal packet loss is acceptable.


* **Domain Name System (DNS):** Maps human-readable domain names to machine IP addresses via specialized records (e.g., `A` records map to IPv4, `AAAA` records map to IPv6). Globally overseen by ICANN.
* **Network Safeguards:**
* *Firewalls:* Control incoming and outgoing traffic.
* *Ports:* Specific endpoints assigned to run dedicated services (e.g., Port `80` for HTTP, Port `22` for SSH).



### Essential Application Protocols

* **HTTP:** A stateless, request-response protocol built over TCP/IP. It relies on standard methods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) and numerical status codes ($2xx$ Success, $3xx$ Redirection, $4xx$ Client Error, $5xx$ Server Error).
* **WebSockets:** Provides full-duplex, persistent two-way communication channels over a single connection, bypassing regular HTTP polling overhead. Crucial for real-time applications like chat rooms or stock tickers.
* **Mail Protocols:** *SMTP* handles outbound email transmission between servers. *IMAP* retrieves and syncs mail across multiple devices, while *POP3* downloads mail locally to a single client machine.
* **File & Remote Access:** *FTP* transfers bulky files during maintenance; *SSH* provides encrypted network channels for remote command execution.
* **IoT & Specialized Messengers:** *MQTT* is a lightweight messaging protocol designed for low-bandwidth, low-power IoT hardware. *AMQP* provides secure, robust message queueing for enterprise middleware (e.g., RabbitMQ).
* **RPC (Remote Procedure Call):** Abstracts network interactions, allowing a client application to execute functions on a remote server as if it were a local execution.

---

## 5. API Design Paradigms & Best Practices

API design maps out data transport formats and structures CRUD (Create, Read, Update, Delete) operations.

### API Architectural Styles

* **REST (Representational State Transfer):** A highly consumable, stateless design model utilizing standard HTTP methods. It primarily exchanges data using JSON. A primary drawback is **over-fetching** or **under-fetching** data from rigid endpoint structures.
* **GraphQL:** Allows clients to define the exact shape of the payload they need in a single query, preventing data mismatches. All queries route via `POST` methods, returning HTTP 200 status codes even when logical errors occur inside the response body.
* **gRPC (Google Remote Procedure Call):** Built on top of **HTTP/2**, offering advanced features like multiplexing. It utilizes **Protocol Buffers** to serialize highly compact structured data, maximizing bandwidth efficiency for microservice communication.

### Engineering Best Practices

* **Idempotency:** A well-designed `GET` request must be strictly idempotent; calling it repeatedly should return the identical resource state without changing any database data.
* **Backward Compatibility & Versioning:** Modifying active endpoints should not break older clients. Introduce explicit URI paths (e.g., `/api/v1/products` vs. `/api/v2/products`) or append new fields to GraphQL schemas without deleting legacy keys.
* **Security Controls:**
* *Rate Limiting:* Restricts the number of allowable requests from a single client within a specific timeframe to mitigate DDoS exploits.
* *CORS (Cross-Origin Resource Sharing):* Fine-grained configurations determining which third-party domains can securely read API responses.



---

## 6. Caching & Content Delivery Networks (CDNs)

Caching caches duplicate data in temporary storage to mitigate latency caused by geographical distances or expensive computing lookups.

* **Browser Caching:** Stores static assets (HTML, CSS, JS) directly on the client's local drive. Regulated by the server via `Cache-Control` response headers (e.g., `max-age=7200`).
* **Cache Evaluation:**
* *Cache Hit:* The requested resource is successfully found inside the cache.
* *Cache Miss:* The resource is missing, forcing a trip to the primary database or origin server.
* *Cache Ratio:* The percentage of requests fulfilled by the cache versus total inquiries. Inspectable via custom headers like `X-Cache`.


* **Server Caching & Write Strategies:** Storing frequent data lookups in memory (e.g., Redis, Memcached).
* *Write-Around:* Data writes bypass the cache entirely and save straight to the database. Good for lower write frequencies.
* *Write-Through:* Data writes simultaneously write to both the cache and database. Guarantees consistency but introduces minor latency.
* *Write-Back (Write-Behind):* Writes hit the fast cache layer first, syncing asynchronously to the persistent database later. Highest performance but carries a risk of data loss if a server crashes.


* **Eviction Policies:** When cache storage hits capacity, items must be pruned according to strict rules: Least Recently Used (**LRU**), First-In First-Out (**FIFO**), or Least Frequently Used (**LFU**).
* **Content Delivery Networks (CDNs):** A globally distributed edge proxy network optimized to cache and deliver static, bulky files (images, video streams, script bundles) closest to the user's geographical region.
* *Pull-Based CDN:* Automatically extracts and caches static components from the origin server upon a user's initial request. Low-maintenance.
* *Push-Based CDN:* The origin server actively pushes content updates down to the edge nodes. Best for massive files updated infrequently.



---

## 7. Proxies and Load Balancers

Proxies act as intermediate traffic managers standing between the client machine and the target server infrastructure.

### Forward vs. Reverse Proxies

* **Forward Proxy:** Sits directly in front of **clients** to shield their identities from the wider internet. Used within enterprise networks to monitor employee usage, filter malicious sites, anonymize browsing, or manage account restrictions.
* **Reverse Proxy:** Sits directly in front of a cluster of **backend web servers**, hiding their identity and implementation from external clients. It intercepts inbound requests to distribute traffic, compress payloads, handle caching, and manage SSL/TLS decryption (SSL offloading).

### Load Balancing Algorithms

Load balancers act as specialized reverse proxies that spread processing workloads across multiple backend servers to maximize throughput and eliminate bottlenecks.

* **Round Robin:** Routes incoming traffic sequentially down the list of available servers. Best when all target server specs are identical.
* **Least Connections:** Dynamically tracks and sends incoming requests to whichever server currently handles the fewest active connections. Ideal for long-running, uneven processing tasks.
* **Least Response Time:** Routes traffic to the server demonstrating the lowest response latency combined with minimal active connection loads.
* **IP Hashing:** Hashes the client's IP address to map them consistently to the exact same backend node, ensuring session persistence.
* **Weighted Algorithms (Weighted Round Robin/Least Connections):** Assigns static coefficients to specific machines based on their compute capacity (RAM, CPU). More capable servers absorb a proportionally higher share of requests.
* **Geographical Algorithms:** Automatically routes traffic to data center clusters geographically closest to the user's location to lower round-trip latency.
* **Consistent Hashing:** Arranges both nodes and keys in a virtual hash ring space. Minimizes the massive re-mapping of data or connection sessions when individual servers are dynamically added or dropped from the cluster pool.

### High Availability and Failover

Because a load balancer sits at the front entry point of infrastructure, it represents a dangerous **Single Point of Failure (SPOF)**.

To prevent systemic outages, engineers deploy redundant load balancers configured in active-passive pairs utilizing continuous health monitoring. If the active instance fails, automated health checks trigger a **failover**, switching traffic immediately to the standby node via automated DNS routing adjustments.

---

## 8. Database Architecture & Scaling Strategies

Choosing database backends requires an understanding of data structural layouts, structural integrity constraints, and horizontal distribution strategies.

### Database Categories

* **Relational (SQL) Databases:** Organized tabular files cleanly divided into rigid tables, columns, and foreign key relations (e.g., PostgreSQL, MySQL, SQLite). They guarantee structural data integrity by enforcing **ACID compliance**:
* **Atomicity:** All operations within a transaction succeed together, or the entire transaction rolls back completely (All-or-Nothing).
* **Consistency:** Every transaction transitions the database from one valid, rule-abiding state directly to another.
* **Isolation:** Concurrent transactions execute independently without bleeding data or conflicting mid-flight.
* **Durability:** Once a transaction commits, data locks into non-volatile storage and survives unexpected system crashes.


* **Non-Relational (NoSQL) Databases:** Schema-less, highly flexible structures that drop rigid consistency rules to maximize execution speed and horizontal throughput (e.g., MongoDB, Cassandra, Redis). They group into:
* *Key-Value Stores:* High-performance lookups (Redis).
* *Document Stores:* Storing hierarchical JSON objects (MongoDB).
* *Graph Databases:* Mapping complex interconnected networks of entities and edges (Neo4j).


* **In-Memory Databases:** Drop disk-bound reading entirely by holding datasets directly inside active RAM (e.g., Redis, Memcached). Delivers ultra-low latency; primarily used for caching and transient session tracking.

### Scaling Methodologies

When a database hits hardware performance thresholds, it scales using one of two methods:

* **Vertical Scaling (Scale-Up):** Adding raw physical power (more RAM, faster SSDs, or higher core CPUs) directly onto the existing database machine. This method hits strict physical engineering limits and introduces high costs quickly.
* **Horizontal Scaling (Scale-Out):** Distributing database workloads across multiple machines using two primary architectural approaches:

| Horizontal Strategy | Architectural Approach | Use Case |
| --- | --- | --- |
| **Data Replication** | Copies the identical dataset across multiple machines. Can be structured as *Master-Slave* (one master receives all writes and updates read-only slaves) or *Master-Master* (all nodes handle both read and write operations). | Maximizes **High Availability** and scales heavy read operations across nodes. |
| **Database Sharding** | Partitions distinct segments of a massive dataset into smaller, independent chunks called *shards*, distributing them across separate physical hardware nodes. | Eliminates storage bottlenecks on huge datasets. Implemented via **Range-Based**, **Directory-Based**, or **Geographical** partitioning. |

### Performance Optimization Techniques

Beyond scaling hardware, database performance can be optimized using specific application techniques:

1. **In-Memory Caching:** Intercepting repetitive query patterns before they touch the database layer by housing the results inside an ephemeral Redis layer.
2. **Indexing:** Creating dedicated data structures on heavily queried columns to change sequential table scans into fast index lookups, accelerating read speeds.
3. **Query Optimization:** Rewriting database queries to reduce slow table `JOIN` operations, or analyzing processing overhead via profiling tools like execution paths (`EXPLAIN PLAN`).
https://www.youtube.com/watch?v=F2FmTdLtb_4
