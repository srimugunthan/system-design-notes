# Google Cloud Platform — Q&A Reference

## 1. Google Cloud AI Integration

**Core Functionality: Pre-built Vertex AI APIs vs. training/fine-tuning a custom model**

Pre-built Vertex AI APIs (Vision, Speech-to-Text, Translation, Natural Language, etc.) are fully managed, pre-trained models exposed as simple REST/gRPC endpoints. You send data and get predictions back — no ML expertise, training data, or infrastructure management required. They're optimized for common, general-purpose tasks (e.g., OCR, sentiment analysis, transcription) and priced per API call.

Training or fine-tuning a custom model on Vertex AI (via AutoML or custom training with your own code/containers) is for domain-specific problems where generic models underperform — e.g., classifying a proprietary product catalog, or fine-tuning a foundation model on internal documents. This requires you to supply labeled data, manage training jobs (compute, hyperparameters, pipelines), and take on responsibility for evaluation, versioning, and retraining. The tradeoff is higher effort and cost in exchange for accuracy tailored to your specific data distribution.

**Generative AI: How Gemini in Google Cloud assists developers and data teams**

Gemini in Google Cloud embeds a generative AI assistant directly into the tools people already use:
- **BigQuery**: Gemini can generate and explain SQL from natural language, suggest schema/pipeline optimizations, auto-generate data insights, and assist with data preparation and Python code in notebooks.
- **Application Integration**: Gemini can suggest integration flows, auto-generate mappings between fields of different systems, and help troubleshoot workflow configurations — reducing the manual work of stitching APIs together.
- **Cloud Code**: Gemini acts as an in-IDE coding assistant (in VS Code, IntelliJ, Cloud Shell) — code completion, chat-based debugging, generating Kubernetes manifests/Dockerfiles, and explaining unfamiliar code.

The common thread is contextual, in-product assistance that reduces the need to leave your workflow to look up syntax or documentation.

**Model Deployment: Role of Vertex AI Endpoints**

A Vertex AI Endpoint is the managed serving layer that hosts a trained model (custom or AutoML) and exposes it for **online/real-time predictions** via a REST/gRPC API. It handles:
- Provisioning and autoscaling compute (CPU/GPU) behind the endpoint
- Traffic splitting across multiple model versions (for A/B testing or canary rollouts)
- Request/response logging and monitoring
- Optionally, explainability outputs alongside predictions

Endpoints are distinct from **batch prediction jobs**, which run asynchronously over large datasets without needing a persistently running endpoint.

---

## 2. Cloud Run

**Architecture: Workloads and scaling model vs. Compute Engine**

Cloud Run is designed for **stateless, containerized workloads** that respond to requests or events — HTTP APIs, microservices, webhooks, background job processing triggered by events, and lightweight batch/task workloads (via Cloud Run Jobs).

Its serverless model differs fundamentally from Compute Engine VMs:
- **Scaling**: Cloud Run automatically scales the number of container instances up and down based on incoming request volume, and can **scale to zero** when there's no traffic — meaning you pay nothing when idle. Compute Engine VMs run continuously (or per your manual/managed-instance-group scaling policy) and you pay for provisioned capacity regardless of utilization.
- **Management**: Cloud Run abstracts away OS patching, VM provisioning, and capacity planning entirely. Compute Engine requires you to manage the OS, networking, and scaling policy yourself.
- **Billing granularity**: Cloud Run bills per request/CPU-second of actual usage; VMs bill per second of uptime regardless of whether they're doing work.

**Deployment Format**

You must package your application as a **container image** (typically built from a Dockerfile, conforming to the Container Runtime Contract — listening on the `PORT` environment variable). This image is pushed to Artifact Registry (or Container Registry) before Cloud Run deploys it.

**Trigger Mechanisms**

1. **HTTP(S) requests** — Cloud Run services expose a URL and respond directly to synchronous HTTP requests (REST APIs, webhooks, web apps).
2. **Event-driven execution via Eventarc** — Cloud Run services can be invoked asynchronously in response to events from over 90+ Google Cloud sources (Pub/Sub messages, Cloud Storage object changes, Firestore writes, Audit Logs, etc.), with Eventarc routing the event as an HTTP request to the service.

---

## 3. Cloud Build (CI/CD)

**Core Purpose**

Cloud Build is Google Cloud's fully managed **CI/CD** service. It automates the process of fetching source code, running builds/tests, producing artifacts (container images, binaries, packages), and deploying them to target environments (Cloud Run, GKE, App Engine, etc.). It plays the role of the "pipeline engine" that turns a code commit into a deployed, running artifact — typically triggered automatically by a push to a source repository (GitHub, Cloud Source Repositories, Bitbucket).

**Build Configuration**

A build is defined in a **`cloudbuild.yaml`** (or `cloudbuild.json`) file, written in YAML (or JSON). This file specifies a sequence of **build steps**, each of which runs in its own container image, along with arguments, environment variables, and whether steps run sequentially or in parallel (using the `waitFor` field to control dependencies).

**Artifact Management**

After a successful pipeline run, compiled container images are typically pushed to **Artifact Registry** (Google's current recommended registry, superseding Container Registry), and non-container build artifacts (JARs, binaries, packages) are often stored in **Cloud Storage** buckets or language-specific Artifact Registry repositories (npm, Maven, Python, etc.).

---

## 4. Application Integration

**iPaaS Definition**

Google Cloud Application Integration is a fully managed **integration Platform-as-a-Service (iPaaS)**. It lets you connect SaaS applications (Salesforce, Jira, Workday, ServiceNow, etc.) with Google Cloud services and other systems using a visual, low-code/no-code designer rather than writing custom point-to-point glue code. It provides pre-built connectors for common SaaS/enterprise systems, handles authentication, retries, error handling, and transformation logic centrally, which reduces the maintenance burden of bespoke integration scripts.

**Visual Designer building blocks**

- **Triggers**: The entry point that starts a workflow (e.g., an API call, a scheduled time, a Pub/Sub message, or a Cloud Scheduler event).
- **Tasks**: Individual units of work within the workflow — calling a connector (e.g., "Create Salesforce Record"), calling an API, executing custom logic, sending an email, or branching logic (conditions/loops).
- **Data Mappings**: The configuration that maps fields/values from one system's data structure to another's (e.g., mapping a Salesforce "Opportunity" field to a BigQuery column), including any transformation functions applied along the way.

**Event-Driven Workflows**

Triggers listen for a specific business event — a new record created in Salesforce, a file landing in Cloud Storage, an API request, or a message published to Pub/Sub. When that event occurs, the trigger automatically fires and kicks off the integration's task sequence (transform data → call downstream systems → handle errors), without any manual intervention or polling required.

---

## 5. IAM Roles and Service Accounts

**Role Types**

- **Primitive/Basic roles** (Owner, Editor, Viewer): Broad, legacy roles that apply across an entire project with very coarse-grained permissions (e.g., Editor can modify almost any resource in the project). Generally discouraged for production use because they violate least privilege.
- **Predefined roles**: Curated by Google for specific services and job functions (e.g., `roles/bigquery.dataViewer`, `roles/storage.objectAdmin`), offering more precise permission sets than primitive roles while still being maintained and updated by Google.
- **Custom roles**: User-defined roles built from an exact list of permissions you select, used when predefined roles are either too broad or too narrow for a specific need. They give you full control but require you to maintain them as APIs/permissions evolve.

**Service Accounts**

A Service Account is a special type of identity intended for **non-human principals** — applications, VMs, or workloads — to authenticate and make authorized API calls to Google Cloud services. Unlike a standard user account, a service account:
- Isn't tied to a person or interactive login/password; it authenticates via **keys or short-lived tokens** (and ideally via Workload Identity/attached identity rather than downloaded keys).
- Is created and owned within a project, and can itself be granted IAM roles, or have other identities granted permission to "act as" it.
- Doesn't have an associated Google Workspace/Gmail-style session, MFA prompt, or interactive consent flow.

**Principle of Least Privilege**

Granting a Service Account only the specific, granular roles it needs — rather than project-level Owner/Editor — limits the "blast radius" if that account's credentials are ever leaked or the application is compromised. Since service accounts are often embedded in code, containers, or automated pipelines (making them a common attack target), over-privileging one effectively hands an attacker broad control over the project. Least privilege ensures a compromised service account can only touch the specific resources it was scoped to use.

---

## 6. BigQuery (Basic Understanding)

**Architecture: Decoupled compute and storage**

BigQuery separates the storage layer (Colossus, Google's distributed storage system) from the compute layer (Dremel-based query execution engine) so each can scale independently. This is beneficial because:
- **Scaling**: You can store petabytes of data cheaply without provisioning any compute, and query engines can scale up massively parallel processing only for the duration of a query, rather than requiring a persistently running, fixed-size cluster.
- **Cost**: You pay for storage (cheap, ongoing) separately from compute (billed per query/bytes scanned, or via flat-rate slot reservations), so idle storage doesn't incur compute costs, and burst query workloads don't require permanently reserved infrastructure.

**Data Model**

BigQuery is optimized for **OLAP (Online Analytical Processing)** workloads — large-scale analytical queries and aggregations over big datasets — rather than OLTP (high-frequency, low-latency transactional read/writes). It is queried using **standard SQL** (ANSI SQL-compliant, GoogleSQL dialect).

**Data Ingestion**

1. **Batch loading**: Loading data in bulk from files (CSV, JSON, Avro, Parquet, ORC) stored in Cloud Storage, or via scheduled/query-based load jobs — suited for periodic ETL/ELT pipelines.
2. **Streaming ingestion**: Inserting data row-by-row or in small batches in near real-time (via the Storage Write API or legacy streaming inserts), suited for use cases needing low-latency availability of fresh data for querying.

---

## 7. GKE and Pub/Sub (Good to Have)

### Google Kubernetes Engine (GKE)

**Containers at Scale: GKE vs. Cloud Run**

Choose GKE over Cloud Run when you need:
- Fine-grained control over networking, custom scheduling, and complex multi-container pod architectures (sidecars, init containers, DaemonSets)
- Stateful workloads requiring persistent volumes and complex storage orchestration
- Advanced traffic management, service mesh (Istio/Anthos Service Mesh), or custom autoscaling logic beyond simple HTTP concurrency
- Multi-cluster, hybrid, or on-prem/multi-cloud portability requirements
- Workloads with long-running, non-request-driven processes, specialized hardware (GPUs/TPUs) with custom scheduling, or very fine control over resource bin-packing across many services

Cloud Run remains the simpler choice for stateless request/event-driven services where you don't need this level of infrastructure control.

**GKE Modes: Standard vs. Autopilot**

- **GKE Standard**: You manage the underlying node pools yourself — choosing machine types, configuring autoscaling policies, and being responsible for node-level maintenance and capacity planning. Offers maximum flexibility and control.
- **GKE Autopilot**: Google manages the node infrastructure entirely — you only define pod specs (CPU/memory/requests), and GKE provisions and scales the underlying nodes automatically. Billing is per-pod resource usage rather than per-node, and it enforces best-practice security/configuration defaults out of the box, trading some flexibility for operational simplicity.

### Cloud Pub/Sub

**Messaging Pattern**

Cloud Pub/Sub implements the **publish-subscribe (pub/sub) messaging pattern**. Publishers send messages to a **topic** without any knowledge of who (if anyone) is consuming them. Subscribers create **subscriptions** to a topic to receive those messages. This decouples producers from consumers entirely — publishers don't need to know about subscribers' existence, count, or availability, and multiple subscribers can independently consume the same stream of messages at their own pace.

**Subscriptions: Pull vs. Push**

- **Pull subscription**: The subscriber application actively calls the Pub/Sub API to request and retrieve messages when it's ready to process them, then acknowledges them. This gives the subscriber control over the rate and timing of message consumption — well-suited for services that need to manage their own load/back-pressure.
- **Push subscription**: Pub/Sub itself initiates delivery by sending an HTTP POST request containing the message to a pre-configured endpoint (e.g., a Cloud Run service or webhook URL). This is useful for serverless/event-driven architectures where you want Pub/Sub to actively deliver events rather than having a service continuously poll for them.
