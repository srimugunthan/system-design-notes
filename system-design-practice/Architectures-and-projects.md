# 7 Architectural Patterns: Use Cases, Trade-offs, Project Ideas & GitHub References

---

## 1. Layered Architecture

- **What:** Organizing the application into horizontal tiers.
- **When to Use:** Small, simple applications with limited budgets.
- **When to Avoid:** High-scale applications, as layers can eventually become performance bottlenecks.

### Project: Credit Risk Scoring API

A straightforward REST API with presentation, service, and data layers for scoring loan applicants. Simple to explain, easy to demo, and lets you showcase clean separation of concerns without overengineering. Good entry point for DSChunkies content too — the architecture is visual and teachable.

### GitHub References

- **[kamilmazurek/layered-architecture-template](https://github.com/kamilmazurek/layered-architecture-template)** — Java/Spring Boot microservice template with clean Presentation → Business → Data separation, unit and integration tests included. Best-in-class for understanding the pattern's fundamentals.
- **[vj12354/spring-layered-architecture](https://github.com/vj12354/spring-layered-architecture)** — Spring Boot shopping checkout system; concise and readable, good for DSChunkies explainer material.
- **[benedya/nestjs-layered-architecture](https://github.com/benedya/nestjs-layered-architecture)** — NestJS/TypeScript with TypeORM; shows Domain/Application/Infrastructure separation plus Anticorruption pattern for module communication.

---

## 2. Microservices

- **What:** Decomposing an application into small, independent services.
- **When to Use:** Large-scale systems managed by multiple teams.
- **When to Avoid:** Small teams or simple apps; the operational overhead is often too high.

### Project: AML Detection Platform

Decompose AML into independent services: transaction ingestion, entity resolution, rule engine, GNN-based anomaly scorer, alert routing, and case management. Each service owns its data store. Fits naturally into the Shield-Fin / ContagionMapper ecosystem and mirrors how real financial institutions actually build this.

### GitHub References

- **[GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo)** — The gold standard reference: Google's "Online Boutique" with 10 polyglot microservices on Kubernetes + Istio + gRPC. Used internally at Google and widely cited in architecture literature.
- **[davidetaibi/Microservices_Project_List](https://github.com/davidetaibi/Microservices_Project_List)** — Curated academic list of real open-source projects that migrated to microservices; useful for pattern research and portfolio justification.
- **[aspnetrun/run-aspnetcore-microservices](https://github.com/aspnetrun/run-aspnetcore-microservices)** — .NET microservices with Docker, Kubernetes, RabbitMQ, CQRS, API Gateway, and Clean Architecture; closest to production-grade enterprise reference for financial systems.

---

## 3. Event-Driven Architecture

- **What:** Services that react to state changes in real-time.
- **When to Use:** Systems requiring high responsiveness and complex workflows.
- **When to Avoid:** Systems where strict transactional data consistency is the top priority (e.g., core banking).

### Project: Real-Time Fraud Detection Pipeline

Kafka-driven pipeline where transaction events trigger parallel enrichment (velocity checks, device fingerprinting, graph lookups) and feed into a streaming model. Emit fraud signals as events downstream to block/flag handlers. Directly extends existing fraud detection work and demonstrates the architecture's strength in latency-sensitive contexts.

### GitHub References

- **[ALgosZen/python-kafka-event-driven-architecture](https://github.com/ALgosZen/python-kafka-event-driven-architecture)** — Minimal Python + Kafka EDA using Docker; clean starting point for a fraud pipeline proof-of-concept.
- **[ava-orange-education/Ultimate-Event-Driven-Architecture-with-Python-and-Apache-Kafka](https://github.com/ava-orange-education/Ultimate-Event-Driven-Architecture-with-Python-and-Apache-Kafka)** — Book companion repo covering EDA fundamentals, Kafka consumers/producers, and real-world patterns in Python end-to-end.
- **[GingFreecss2/Spring-Kafka-Event-Driven-Architecture](https://github.com/GingFreecss2/Spring-Kafka-Event-Driven-Architecture)** — Spring Boot + Kafka event-driven demo with order processing, stock management, and notification services; good Java reference for the pattern.

---

## 4. Microkernel (Plug-in Architecture)

- **What:** A minimal core system where extra functionality is added via plug-ins.
- **When to Use:** Product-based apps that require highly customizable features.
- **When to Avoid:** Apps where the core logic itself changes on a regular basis.

### Project: RedTeamAgentLoop / RTtoolsuite

The core engine orchestrates attack loops; each attack technique (prompt injection, jailbreak, MITRE ATLAS tactic) is a plug-in. New red team modules slot in without touching the core. This is the canonical use case for microkernel — worth explicitly calling this out in the README and portfolio narrative.

### GitHub References

- **[ahhuisg/mkml](https://github.com/ahhuisg/mkml)** — Python microkernel architecture for ML libraries using metaclass-based plug-in registration. Directly relevant: shows how to build a core system with swappable StandardizationPlugin, MonitoringPlugin, DataSourcePlugin — mirrors the RedTeamAgentLoop attack-module design.
- **[moonrailgun/mini-star](https://github.com/moonrailgun/mini-star)** — Frontend micro-kernel framework for progressive plug-in migration; useful reference for the registry and IPC patterns that the microkernel relies on.
- **[HelenOS/helenos](https://github.com/HelenOS/helenos)** — Portable microkernel-based OS written from scratch; the canonical computer-science reference for the pattern's origin, worth linking in any architecture writeup.

---

## 5. Serverless

- **What:** Code runs only when triggered, with the cloud provider managing all resources.
- **When to Use:** Unpredictable traffic patterns or background tasks.
- **When to Avoid:** Long-running processes or high-performance computing due to "cold starts".

### Project: LLM Guardrail Trigger Functions

A set of Lambda/Cloud Functions that fire on API Gateway events to run input/output guardrails (PII detection, toxicity scoring, prompt injection classification) before and after LLM calls. Stateless, event-triggered, cheap to run — textbook serverless. Connects directly to Shield-Fin.

### GitHub References

- **[serverless/examples](https://github.com/serverless/examples)** — The definitive collection of serverless boilerplates across AWS Lambda, Azure Functions, and GCP Cloud Functions; covers Python, Node.js, Go. Use as a scaffold for the guardrail function project.
- **[aws-samples/lambda-refarch-fileprocessing](https://github.com/aws-samples/lambda-refarch-fileprocessing)** — AWS official serverless reference architecture for real-time parallel file processing using S3 Events → SNS → SQS → Lambda. Closest structural analogue to an input/output guardrail pipeline.
- **[aws-samples/lambda-refarch-mobilebackend](https://github.com/aws-samples/lambda-refarch-mobilebackend)** — AWS production-grade serverless backend reference: API Gateway → Lambda → DynamoDB with async processing via DynamoDB Streams; shows patterns for stateless event-triggered design.

---

## 6. Space-Based Architecture

- **What:** Distributing processing and storage across RAM to eliminate database bottlenecks.
- **When to Use:** Extreme concurrency, such as social media traffic.
- **When to Avoid:** Relational data that relies heavily on disk-based storage.

### Project: Transaction Velocity Cache

An in-memory grid (Hazelcast or Redis Cluster) that holds rolling transaction windows per customer, updated in real time, with no synchronous DB reads on the hot path. Fraud rules query RAM, not disk. Niche to build but powerful to present in interviews — shows understanding of where databases become the bottleneck.

### GitHub References

- **[hazelcast/hazelcast](https://github.com/hazelcast/hazelcast)** — The primary open-source Hazelcast IMDG repo (6.5k+ stars). Distributed maps, queues, stream processing, and Kafka integration; the go-to library for building a space-based velocity cache in Java.
- **[piomin/sample-hazelcast-spring-datagrid](https://github.com/piomin/sample-hazelcast-spring-datagrid)** — Spring Boot + Hazelcast in-memory data grid examples: 2nd-level JPA cache, hot cache with Striim, and Kubernetes deployment. Practical starting point for the transaction velocity cache project.
- **Note on Python:** For a Python-native implementation, use Redis Cluster (`redis-py`) as the in-memory grid. No single canonical "space-based architecture in Python" repo exists — the pattern is better demonstrated through the Hazelcast/Java ecosystem where the tooling is mature.

---

## 7. Hexagonal (Ports & Adapters)

- **What:** Isolating core logic from external tools (databases, APIs) using ports and adapters.
- **When to Use:** Systems needing high testability and long-term flexibility to "swap" components.
- **When to Avoid:** Simple CRUD applications where the abstraction adds unnecessary complexity.

### Project: AuditAgent / FinVision Core

Wrap LLM-based audit or document extraction logic in a hexagonal shell. The core domain logic (audit reasoning, extraction rules) talks only to ports. Adapters swap between OpenAI, Gemini, or a local model; between PostgreSQL and a vector store; between REST and CLI. Makes the system genuinely model-agnostic and testable without live API calls — a strong demonstration of production discipline.

### GitHub References

- **[szymon6927/hexagonal-architecture-python](https://github.com/szymon6927/hexagonal-architecture-python)** — Python/FastAPI gym management system demonstrating clean Domain → Application → Infrastructure layering with dependency injection. Best Python reference for the pattern; well-documented blog series accompanies it.
- **[BasicWolf/hexagonal-architecture-django](https://github.com/BasicWolf/hexagonal-architecture-django)** — Django + hexagonal architecture with four-part article series (2021–2025) covering ports/adapters, persistence, transactions, and lightweight integration tests. Useful if the AuditAgent stack uses Django ORM.
- **[tcmlabs/hexagonal-architecture-python-spark](https://github.com/tcmlabs/hexagonal-architecture-python-spark)** — Hexagonal architecture applied to a PySpark/Pandas data engineering project — niche but directly relevant for ML pipeline work where you want to swap between local Pandas and remote Spark execution.
- **[marcosvs98/hexagonal-architecture-with-python](https://github.com/marcosvs98/hexagonal-architecture-with-python)** — FastAPI + DDD (Aggregates, Bounded Contexts) with full ports-and-adapters implementation; the most feature-rich Python example for a microservice context.

---

---

## 8. Modular Monolith

- **What:** A single deployable unit structured into strongly-bounded internal modules, each owning its own domain logic and data. Modules communicate through well-defined interfaces, not shared database tables.
- **When to Use:** Medium-to-large applications with growing complexity that aren't yet ready for the operational overhead of microservices. Ideal stepping stone — modules can be extracted to independent services later if needed.
- **When to Avoid:** Systems that already have clear team and scaling boundaries justifying microservices, or trivially simple apps where modules add no value.

### Project: Fraud Investigation Platform (Modular Monolith)

A single deployable app with isolated modules for Transaction Ingestion, Rules Engine, Alert Management, Case Management, and Reporting. Each module has its own DB schema, communicates via in-process message bus, and can be extracted into a microservice if a specific module needs independent scaling. This is the right starting architecture for AML work before committing to microservices overhead.

### GitHub References

- **[kgrzybek/modular-monolith-with-ddd](https://github.com/kgrzybek/modular-monolith-with-ddd)** — The canonical reference. Full production-grade modular monolith with DDD, Event Sourcing, unit + integration tests, and a companion article series by Kamil Grzybek. Widely cited as the best example of the pattern.
- **[meysamhadeli/booking-modular-monolith](https://github.com/meysamhadeli/booking-modular-monolith)** — .NET 10 booking system with Vertical Slice Architecture, CQRS, EDA, gRPC, Outbox/Inbox pattern, and testcontainers. Very feature-complete and modern.
- **[mehdihadeli/food-delivery-modular-monolith](https://github.com/mehdihadeli/food-delivery-modular-monolith)** — .NET 8 food delivery app with each module running its own Composition Root (isolated DI container), treating modules as virtual microservices. Shows the cleanest approach to module autonomy within a monolith.

---

## 9. CQRS (Command Query Responsibility Segregation)

- **What:** Separating the write model (Commands that change state) from the read model (Queries that return data). The two sides can use different data stores, schemas, and scaling strategies.
- **When to Use:** Systems where read and write patterns differ significantly in volume, shape, or consistency requirements. Common in financial services, audit systems, and any domain with complex reporting on top of transactional writes.
- **When to Avoid:** Simple CRUD apps where a single model serves both reads and writes without tension. CQRS adds indirection that is costly when the problem doesn't justify it.

### Project: AuditAgent Read/Write Separation

Command side: LLM-driven audit actions write immutable audit events (findings, flags, decisions) to a command store. Query side: pre-projected read models expose audit summaries, risk dashboards, and trail views without touching the write store. Enables fast, independent scaling of reporting without locking the audit execution pipeline.

### GitHub References

- **[andreschaffer/event-sourcing-cqrs-examples](https://github.com/andreschaffer/event-sourcing-cqrs-examples)** — A minimalistic bank account system in Java demonstrating CQRS + Event Sourcing pragmatically (not dogmatically). Excellent for understanding how the two patterns compose, with good notes on event ordering and idempotency.
- **[aliseylaneh/Python-Eventsourcing-CQRS](https://github.com/aliseylaneh/Python-Eventsourcing-CQRS)** — Python/FastAPI/MongoDB microservice with CQRS + Event Sourcing. Domain aggregates, commands, events, and state reconstruction via event replay — the closest Python reference to the AuditAgent use case.
- **[cer/event-sourcing-examples](https://github.com/cer/event-sourcing-examples)** — Bank account transfer app demonstrating how CQRS + ES enables deployment flexibility: the same codebase can run as a monolith or as microservices with separate command and query services. Useful for portfolio narrative.
- **[heynickc/awesome-ddd](https://github.com/heynickc/awesome-ddd)** — Comprehensive curated list of DDD, CQRS, and Event Sourcing resources; the go-to reference index for the broader ecosystem.

---

## 10. Event Sourcing

- **What:** Persisting state as an ordered, immutable sequence of domain events rather than current state. The event log is the source of truth; current state is derived by replaying events. Snapshots can be used to avoid full replay.
- **When to Use:** Systems requiring full audit trails, temporal queries ("what was the state at T?"), event replay for debugging, or eventual consistency across services. Critical for financial services, fraud forensics, and compliance-heavy domains.
- **When to Avoid:** Systems with simple state that doesn't need history, or teams without the discipline to manage event schema versioning over time.

### Project: Transaction Audit Log (Event Sourced)

Every transaction action (submitted, enriched, flagged, reviewed, escalated, resolved) is stored as an immutable domain event. Current account state is a projection. Forensic investigators can replay the full event stream for any account. Regulators get a tamper-evident log. New ML models can be trained on the historical event stream without touching production state.

### GitHub References

- **[pyeventsourcing/eventsourcing](https://github.com/pyeventsourcing/eventsourcing)** — The most complete Python library for event sourcing. Supports aggregates, projections, CQRS, snapshotting, encryption (GDPR-compliant), and multiple persistence backends (PostgreSQL, DynamoDB, EventStoreDB). The go-to starting point for Python event-sourced systems.
- **[andreschaffer/event-sourcing-cqrs-examples](https://github.com/andreschaffer/event-sourcing-cqrs-examples)** — Same banking reference as CQRS section above; equally relevant here for its clean illustration of aggregate-level event sourcing with a real domain model.
- **[leandrocp/awesome-cqrs-event-sourcing](https://github.com/leandrocp/awesome-cqrs-event-sourcing)** — Curated list of event stores (KurrentDB/EventStoreDB, Marten, SQLStreamStore), frameworks, and articles. Essential reference index for evaluating persistence options.

---

## 11. Service Mesh / Sidecar Architecture

- **What:** A deployment pattern where cross-cutting infrastructure concerns (mTLS, observability, retries, circuit breaking, rate limiting, traffic routing) are delegated to a sidecar proxy co-located with each service, rather than being implemented in application code. A control plane manages all sidecars centrally.
- **When to Use:** Microservices at scale where operational concerns (security, tracing, load balancing) need to be enforced uniformly across many services without changing application code. Istio + Envoy is the dominant implementation.
- **When to Avoid:** Small microservices deployments where the operational overhead of a service mesh outweighs the benefit. Each sidecar consumes ~0.5 CPU cores and 50MB memory — this adds up fast.

### Project: Shield-Fin as a Sidecar Guardrail

Deploy LLM guardrail logic (PII detection, prompt injection classification, output filtering) as an Envoy-based sidecar that intercepts all traffic to and from LLM inference endpoints. The application code stays clean; the sidecar enforces guardrails, logs all LLM I/O for audit, and can be hot-swapped without redeployment. This is the production-grade evolution of Shield-Fin.

### GitHub References

- **[istio/istio](https://github.com/istio/istio)** — The primary open-source service mesh implementation (CNCF project, 36k+ stars). Envoy sidecars + Istiod control plane. The industry standard for Kubernetes service mesh; directly used in the GoogleCloudPlatform/microservices-demo reference above.
- **[envoyproxy/envoy](https://github.com/envoyproxy/envoy)** — The underlying sidecar proxy (C++, 25k+ stars). Understanding Envoy's filter chain is essential for building custom sidecar behaviour — relevant for implementing a guardrail filter in the Shield-Fin context.
- **[linkerd/linkerd2](https://github.com/linkerd/linkerd2)** — Lightweight Rust-based service mesh (CNCF graduated project). Simpler operational model than Istio; worth knowing as the alternative for teams where Istio's complexity is prohibitive.

---

## 12. Pipe and Filter

- **What:** Data flows through a sequence of independent processing stages (filters) connected by channels (pipes). Each filter does one transformation and passes the result downstream. Filters are stateless, composable, and independently testable.
- **When to Use:** Data processing pipelines, ETL workflows, ML feature engineering, log processing, stream transformations — anywhere the processing logic is naturally sequential and each stage is independently meaningful.
- **When to Avoid:** Systems requiring complex branching logic where a simple linear pipeline becomes unwieldy, or where the overhead of inter-stage data transfer is prohibitive for latency-critical paths.

### Project: ML Feature Engineering Pipeline for Fraud Detection

A modular pipeline: Raw Transaction → PII Scrubber → Velocity Feature Extractor → Graph Feature Extractor → Normalizer → Feature Store Writer. Each stage is an independent, testable filter. New features slot in without touching others. The same pipeline runs locally (Pandas) and at scale (Spark/Beam) by swapping the execution backend — a natural fit with Hexagonal Architecture's adapter pattern.

### GitHub References

- **[pyeventsourcing/eventsourcing](https://github.com/pyeventsourcing/eventsourcing)** — (see Event Sourcing section) — the notification/projection mechanism is itself a pipe-and-filter pattern over an event stream.
- **[Building-ML-Pipelines/building-machine-learning-pipelines](https://github.com/Building-ML-Pipelines/building-machine-learning-pipelines)** — O'Reilly book companion repo; TFX-based ML pipeline covering ingestion → validation → transformation → training → serving. The most complete real-world Python pipe-and-filter reference for ML workloads, using Apache Beam as the execution engine.
- **[pitzer42/pypes](https://github.com/pitzer42/pypes)** — Minimal Python implementation of Pipes and Filters using pure functions as filters and `yield`-based pipes. Good for understanding the pattern's mechanics before reaching for TFX or Beam.
- **Apache Beam / [apache/beam](https://github.com/apache/beam)** — The canonical production framework for pipe-and-filter data processing in Python. Unified model runs on Dataflow, Spark, or Flink. If you're building the fraud feature pipeline at scale, this is the execution engine.

---

## Recommended Build Sequence

If building these for a portfolio, a natural sequencing is:

1. **Layered** — quick win, easy to explain
2. **Modular Monolith** — the right starting point for most real systems; demonstrates discipline before reaching for microservices
3. **Hexagonal** — shows production depth and testability discipline; apply inside any module
4. **Pipe and Filter** — implement the fraud feature engineering pipeline; directly portfolio-relevant and fast to build
5. **CQRS + Event Sourcing** — add to the AuditAgent or fraud system; high-signal for fintech interviews
6. **Event-Driven** — demonstrates scale thinking; extend the fraud pipeline with Kafka
7. **Microkernel** — already underway via RedTeamAgentLoop; document it explicitly as microkernel
8. **Serverless** — extend Shield-Fin with guardrail trigger functions
9. **Service Mesh** — deploy Shield-Fin as a sidecar; capstone infrastructure pattern
10. **Microservices** — AML platform as the final capstone, when team/infra context justifies it
11. **Space-Based** — niche but high-signal for fraud/fintech interviews; build last or reference conceptually
