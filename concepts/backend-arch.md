# Backend system architecture


## System Overview

<img width="1023" height="644" alt="image" src="https://github.com/user-attachments/assets/6e935e8e-a2cf-4e81-a579-ab14e1ee711f" />

This diagram represents a scalable, production-ready system designed to handle file-retrieval API requests (e.g., `GET /api/files/123`) from end users. It demonstrates how traffic moves from the client layer through edge security controls and into a secure Virtual Private Cloud (VPC).

---

## 1. Edge & Gateway Layer

* **User Traffic**: Represents client requests (scaled for ~10k Monthly Active Users).
* **API Gateway**: Acts as the central reverse proxy and entry point for incoming traffic.
* **CDN (Content Delivery Network)**: Caches static content geographically closer to users to minimize latency and offload traffic from backend servers.
* **Rate Limiting (Edge Cache)**:
* Tracks request counts per user ID using an edge key-value cache.
* If a user exceeds the threshold (e.g., exceeding 10 requests within a window), the Gateway immediately rejects the traffic with an HTTP status 
$$429\text{ Too Many Requests}$$


 to protect downstream services from abuse or DDoS attacks.



---

## 2. Virtual Private Cloud (VPC)

Valid requests pass into a private network containing microservices, databases, and message brokers.

### Microservices (Compute Layer)

* **Auth**: Validates user identity and permissions.
* **Files**: Core service handling metadata operations and business logic for files.
* **Realtime**: Manages persistent connections (e.g., WebSockets or SSE) for live updates.
* **Thumbnail**: Background processing service that generates preview thumbnails for uploaded media.

### Data & Caching Tier

* **In-Memory Cache (RAM)**: High-speed caching layer (e.g., Redis or Memcached).
* **Relational DB**: Stores structured data, relational mappings, and file metadata.
* **Object Storage**: Blob storage (e.g., AWS S3) used for storing raw binary files.

---

## 3. Read-Aside Caching Strategy (Inset Diagram)

When a file request (`file:123`) reaches the backend, it uses a standard **Cache-Aside** pattern:

1. **Check Cache**: Query the in-memory cache for the key `file:123`.
2. **Cache Hit (Yes)**: Immediately return the cached payload to the user without touching the database.
3. **Cache Miss (No)**:
* Fetch the data from the Relational DB or Object Storage.
* Write the fetched data into the Cache for subsequent requests.
* Return the file response to the user.



---

## 4. Asynchronous Messaging & Alerting

* **Broker**: An event streaming or queueing platform (such as Kafka or RabbitMQ) that decouples system events, job requests (e.g., thumbnail creation), and audit logging.
* **Slack Integration**: Asynchronous error or monitoring events from the message broker trigger webhook alerts sent directly to a Slack channel for real-time observability and incident management.
