# System-design Questions 
*System Design
Building for scale
(Non functional requirements) - Endpoint protection, DB Proxy Sharding/Indexing, Load Balancing
Domain services design
Messaging, Caching, Polling/streaming/Sockets, Latency & Throughput, Networks and Protocols*
## Non-Functional Requirements

**Endpoint Protection**
- How would you protect a public API from abuse (rate limiting, throttling, API keys, WAF)? Walk through token bucket vs. leaky bucket vs. sliding window rate limiters.
- Where should rate limiting live — API gateway, service layer, or both? What are the tradeoffs?
- How do you prevent replay attacks and enforce idempotency on a payment/transaction endpoint?
- Design authentication/authorization for a multi-tenant API — OAuth2 vs. API keys vs. mTLS, and when each applies.
- How would you detect and mitigate a credential-stuffing or bot attack in real time?

**DB Proxy, Sharding & Indexing**
- When would you introduce a DB proxy (e.g., ProxySQL, PgBouncer, Vitess)? What problems does it solve vs. create?
- Design a sharding strategy for a table with 500M+ rows. Range-based vs. hash-based vs. directory-based sharding — tradeoffs?
- How do you handle re-sharding/rebalancing without downtime?
- What indexing strategy would you use for a query pattern that filters on (customer_id, transaction_date) but sometimes needs range scans?
- How do composite indexes interact with query planners, and when do they hurt write throughput?
- How would you design cross-shard joins or aggregations (e.g., "total fraud flags per region")?
- Read replicas vs. sharding — when do you reach for one over the other?

**Load Balancing**
- Compare L4 vs. L7 load balancing — when would you choose each?
- How does consistent hashing help with load balancing and cache distribution? Walk through a rebalancing scenario when a node is added/removed.
- Design a load balancing strategy across multi-region deployments with latency-aware routing.
- How do health checks and circuit breakers interact with load balancer behavior during a partial outage?
- How would you handle "hot key" or "hot shard" problems in a load-balanced, sharded system?

## Domain Services Design
- Design the service boundaries for a [fraud detection / KYC onboarding / claims processing] system — what are the bounded contexts and why?
- How do you decide between a monolith-first vs. microservices-first approach for a new domain?
- Design a service that needs strong consistency for writes but can tolerate eventual consistency for reads (e.g., account balance vs. transaction history).
- How would you version a domain service's API without breaking existing consumers?
- Design the data ownership model — which service owns "customer," and how do other services reference it without tight coupling?
- How do you handle distributed transactions across services (sagas vs. two-phase commit vs. eventual consistency with compensation)?
- Design a service for orchestrating a multi-step approval workflow (e.g., loan underwriting) — orchestration vs. choreography?

## Messaging
- Kafka vs. RabbitMQ vs. SQS — how would you choose for a given use case (e.g., fraud event pipeline vs. order processing)?
- Design an event-driven pipeline for real-time fraud scoring — what's the message schema, partitioning key, and consumer group design?
- How do you guarantee exactly-once processing (or handle at-least-once with idempotent consumers)?
- How would you design a dead-letter queue strategy and retry policy for failed message processing?
- Explain how you'd handle message ordering guarantees within a partition vs. across partitions.
- Design a system for event sourcing + CQRS for an audit-heavy domain (e.g., AML transaction history).
- How do you scale consumers when a single partition becomes a bottleneck?

## Caching
- Design a caching layer for a read-heavy service — cache-aside vs. write-through vs. write-behind, and when each applies.
- How do you handle cache invalidation in a distributed system (TTL vs. explicit invalidation vs. event-driven invalidation)?
- Design a solution to prevent cache stampede/thundering herd on a hot key.
- Redis vs. Memcached — what are the actual differentiators for your use case (data structures, persistence, clustering)?
- How would you design a multi-level cache (local in-process + distributed) and handle consistency between the two levels?
- What's your strategy for caching personalized/user-specific data at scale vs. shared/global data?

## Polling / Streaming / Sockets
- Compare short polling, long polling, WebSockets, and Server-Sent Events — map each to a use case.
- Design a real-time dashboard (e.g., live fraud alert feed) — would you use WebSockets, SSE, or gRPC streaming, and why?
- How do you scale WebSocket connections horizontally (sticky sessions, connection state, pub/sub fan-out)?
- Design a system to push real-time notifications to millions of mobile clients — push notifications vs. persistent socket connections vs. polling with backoff.
- How would you handle reconnection, backpressure, and message replay for a dropped WebSocket client?

## Latency & Throughput
- Walk through how you'd diagnose a service that has good throughput but poor p99 latency.
- Explain tail latency amplification — why does one slow dependency degrade overall system latency disproportionately?
- Design for a system requiring <100ms p99 latency — what architectural choices does this force (caching, colocating services, async vs. sync calls)?
- How do you trade off consistency, latency, and throughput in a globally distributed system (CAP/PACELC)?
- How would you design capacity planning for a system expecting 10x traffic growth in 12 months?
- Batch processing vs. stream processing — how do you decide, and how does that decision affect latency/throughput tradeoffs?

## Networks & Protocols
- Compare REST vs. gRPC vs. GraphQL — tradeoffs in latency, tooling, and use case fit.
- Explain what happens during a TCP handshake and why that matters for connection pooling design.
- How does HTTP/2 multiplexing change your service design compared to HTTP/1.1?
- When would you use UDP over TCP in a production system?
- Design service-to-service communication in a service mesh — how do sidecars (Envoy/Istio) handle retries, timeouts, and mTLS?
- How do DNS resolution and connection pooling interact to cause latency spikes during scale-up events?

---

**Scenario-style closers** (good for testing integration of multiple areas — these map well to your fraud/AML and ATM domain background):
- "Design a real-time transaction fraud-scoring system handling 50K TPS with <200ms latency, strong audit requirements, and multi-region failover." (forces messaging + caching + latency + sharding together)
- "Design the backend for a recycling-ATM cash management system that needs near-real-time cash-level polling across thousands of devices with intermittent connectivity." (forces polling/streaming + networks + DB design together)
