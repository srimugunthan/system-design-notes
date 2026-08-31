# AWS Deployment: Global, Millions-of-Documents, High-Throughput, High-Availability

## Context

This is the AWS equivalent of [system-design-Earnings-report-RAG-gcp-deployment.md](./system-design-Earnings-report-RAG-gcp-deployment.md), mapping the same design — [system-design-Earnings-report-RAG.md](./system-design-Earnings-report-RAG.md), [system-design-Earnings-report-RAG-chat-interface.md](./system-design-Earnings-report-RAG-chat-interface.md), [system-design-Earnings-report-RAG-backend-scaling.md](./system-design-Earnings-report-RAG-backend-scaling.md) — onto AWS services, multi-region and active-active, same targets (5-10M documents, 2-5B chunks, sub-second TTFT, 1,000+ QPS, 99.9%+ availability).

Most decisions carry over unchanged in spirit — same non-functional requirements, same "replicate data vs. replicate compute" framing. What's genuinely different is **where AWS's service boundaries don't line up with GCP's**, and those gaps force different tradeoffs, not just different product names. Two matter enough to call out up front:

1. **No single global L7 load balancer.** GCP's Global External HTTPS LB does anycast entry, health-aware regional routing, *and* attaches CDN/WAF in one product. AWS splits this across Route 53, Global Accelerator, and CloudFront — composing three products where GCP uses one.
2. **No Spanner equivalent.** GCP's choice for the metadata/version store leaned entirely on Spanner's unique combination of horizontal scale + synchronous multi-region strong consistency. AWS has no service with that exact combination — the metadata store section below is the sharpest tradeoff in this document, not the vector store (which maps over cleanly).

## Non-Functional Requirements Recap

Unchanged from the GCP doc — reproduced for reference:

| Requirement | Target | Source |
|---|---|---|
| Global latency | Users on any continent see TTFT < 1-2s | Needs geo-local compute, not just geo-local routing |
| Throughput | 1,000+ QPS sustained, spiky at earnings season | chat-interface doc |
| Scale | 5-10M documents, 2-5B chunks, ~30-60TB vectors | backend-scaling doc |
| Availability | 99.9%+, survive a full region outage | Implies regional failover, not just intra-region redundancy |
| Consistency | Strong for version/supersession pointers, eventual for vector freshness | backend-scaling doc's two-speed model |

## AWS Service Mapping

| Component (from prior docs) | AWS Service | Why this one / how it differs from the GCP mapping |
|---|---|---|
| Global entry point | **Route 53** (latency-based routing) + **AWS Global Accelerator** (anycast IPs, L4, AWS-backbone routing) fronting regional **Application Load Balancers** | Composed, not single-product — see §1 |
| Static assets + raw filings | **CloudFront** in front of **S3** | Direct analog to Cloud CDN + Cloud Storage |
| Ingestion queue | **Amazon SQS** (or **Kinesis Data Streams** for higher-throughput ordered ingestion) in one canonical region | Regional resource, unlike Pub/Sub's cross-region reachability — reinforces keeping ingestion centralized in one region (see §6) |
| Ingestion workers | **AWS Fargate** (ECS/EKS tasks) | Direct analog to Cloud Run: scale-to-zero-ish, no server management, fits the bursty earnings-season profile |
| Chat Gateway (SSE termination) | **Amazon EKS** (regional clusters, one per active region) | Same reasoning as GKE choice — see §2 |
| Session store | **Amazon ElastiCache for Redis** (regional, one per region) | Direct analog to Memorystore; kept region-local, same reasoning as the GCP doc |
| Query/task queue (gateway→RAG workers) | **Amazon SQS** (regional) | Direct analog to the regional Pub/Sub topic / Cloud Tasks |
| RAG worker pool | **Amazon EKS** (regional, GPU/CPU node groups) | Autoscaled on SQS queue depth via KEDA or Karpenter, not just CPU |
| Vector store | **Amazon OpenSearch Service** (k-NN / vector engine, per region) *or* self-hosted (Milvus/Weaviate) on EKS | See §4 |
| Metadata store | **Amazon Aurora PostgreSQL Global Database** (primary + secondary regions) — with caveats | See §5 — the central tradeoff of this doc |
| Embedding generation | **Amazon Bedrock** (Titan/Cohere embedding models) *or* self-hosted (TEI/vLLM) on EKS GPU node groups | Same managed-vs-self-hosted crossover as the backend-scaling doc |
| LLM inference | **Amazon Bedrock** (on-demand or Provisioned Throughput) *or* self-hosted vLLM/TGI on EKS GPU node groups | Bedrock includes Claude directly; Provisioned Throughput gets you predictable batching/latency without running inference infra yourself |
| Rate limiting / WAF / DDoS | **AWS WAF** + **AWS Shield Advanced** | Attaches to CloudFront/Global Accelerator/ALB; same edge-rejection role as Cloud Armor |
| Secrets | **AWS Secrets Manager** | Standard |
| CI/CD | **CodePipeline + CodeBuild**, images in **Amazon ECR**, rollout via **CodeDeploy** or GitOps (Argo CD) on EKS | Direct analog to Cloud Build/Deploy/Artifact Registry |
| Observability | **CloudWatch** (Metrics, Logs, Container Insights) + **AWS X-Ray** (tracing) | Direct analog to Cloud Monitoring/Trace/Logging |
| Network isolation | **VPC** per region + **PrivateLink** (VPC endpoints) + **AWS Organizations SCPs** + **Resource Access Manager** for shared-VPC-style sharing | Analog to VPC Service Controls + Shared VPC, composed from more primitives |

## Global Architecture

```
                    ┌──────────────┐        ┌──────────────────────────┐
                    │  Route 53     │        │  CloudFront (static UI    │
                    │  (latency-    │        │  assets + filings, S3      │
                    │  based DNS)   │        │  origin, WAF attached)     │
                    └──────┬────────┘        └──────────────────────────┘
                           ▼
                    ┌──────────────────────────────┐
                    │  AWS Global Accelerator        │
                    │  (anycast IPs, health-checked   │
                    │  regional endpoints, AWS Shield)│
                    └───────────────┬───────────────┘
            ┌──────────────────────┼──────────────────────┐
            ▼                       ▼                       ▼
  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
  │  us-east-1          │  │  eu-west-1          │  │  ap-southeast-1     │
  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │
  │  │ ALB → EKS:      │  │  │ ALB → EKS:      │  │  │ ALB → EKS:      │  │
  │  │ Chat Gateway    │  │  │ Chat Gateway    │  │  │ Chat Gateway    │  │
  │  └────────┬────────┘  │  │ └────────┬────────┘ │  │ └────────┬────────┘ │
  │  ┌────────▼────────┐  │  │ ┌────────▼────────┐ │  │ ┌────────▼────────┐ │
  │  │ ElastiCache Redis│  │  │ │ ElastiCache Redis│ │  │ │ ElastiCache Redis│ │
  │  │ (session, region-│  │  │ │ (session, region-│ │  │ │ (session, region-│ │
  │  │  local)          │  │  │ │  local)          │ │  │ │  local)          │ │
  │  └─────────────────┘  │  │ └─────────────────┘ │  │ └─────────────────┘ │
  │  ┌─────────────────┐  │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │
  │  │ SQS: query queue │  │  │ │ SQS: query queue │ │  │ │ SQS: query queue │ │
  │  │ (regional)       │  │  │ │ (regional)       │ │  │ │ (regional)       │ │
  │  └────────┬────────┘  │  │ └────────┬────────┘ │  │ └────────┬────────┘ │
  │  ┌────────▼────────┐  │  │ ┌────────▼────────┐ │  │ ┌────────▼────────┐ │
  │  │ EKS: RAG Workers │  │  │ │ EKS: RAG Workers │ │  │ │ EKS: RAG Workers │ │
  │  └────────┬────────┘  │  │ └────────┬────────┘ │  │ └────────┬────────┘ │
  │  ┌────────▼────────┐  │  │ ┌────────▼────────┐ │  │ ┌────────▼────────┐ │
  │  │ OpenSearch       │  │  │ │ OpenSearch       │ │  │ │ OpenSearch       │ │
  │  │ (regional vector │  │  │ │ (regional vector │ │  │ │ (regional vector │ │
  │  │ index, CCR-fed)  │  │  │ │ index, CCR-fed)  │ │  │ │ index, CCR-fed)  │ │
  │  │ + Bedrock (LLM)  │  │  │ │ + Bedrock (LLM)  │ │  │ │ + Bedrock (LLM)  │ │
  │  │ + Aurora reader   │  │  │ │ + Aurora reader   │ │  │ │ + Aurora reader   │ │
  │  └─────────────────┘  │  │ └─────────────────┘ │  │ └─────────────────┘ │
  └──────────┬──────────┘  └──────────┬──────────┘  └──────────┬──────────┘
             │                        │                        │
             └────────────────────────┼────────────────────────┘
                                       ▼
                    ┌───────────────────────────────────────────┐
                    │   PRIMARY REGION SHARED LAYER (us-east-1)   │
                    │   Aurora PostgreSQL Global Database          │
                    │     (write primary; secondary readers in     │
                    │      eu-west-1 / ap-southeast-1)              │
                    │   S3 (versioned, CRR to other regions)        │
                    │   SQS/Kinesis: ingestion queue (canonical)    │
                    └───────────────────┬───────────────────────────┘
                                         ▼
                    ┌───────────────────────────────────────────┐
                    │   INGESTION: Fargate workers (canonical      │
                    │   region), consume queue, write S3 + Aurora   │
                    │   primary, push index updates to each         │
                    │   region's OpenSearch domain (via CCR or       │
                    │   direct dual-write)                           │
                    └───────────────────────────────────────────┘
```

## 1. Global entry & routing: three products instead of one

GCP's single Global External HTTPS LB does anycast entry, L7 URL routing, health-aware regional failover, and CDN attachment together. AWS composes the same outcome from three services:

- **AWS Global Accelerator** provides the anycast-IP, AWS-backbone routing GCP's LB gives for free — it health-checks each region's ALB and shifts traffic away from unhealthy regions within seconds, without relying on DNS TTL expiry.
- **Route 53** (latency-based routing) is layered in front for cases Global Accelerator doesn't cover on its own (e.g., routing to services outside the accelerator, or as a fallback record) — for this architecture, Global Accelerator is the primary mechanism and Route 53 mostly points at it.
- **CloudFront** is kept as a *separate* path specifically for the CDN role (static assets + filings), rather than being the same product as the dynamic-traffic router — CloudFront optimizes for cacheable edge content, Global Accelerator optimizes for routing live dynamic traffic (the SSE chat stream) to the nearest healthy compute; conflating them would mean running dynamic API traffic through a caching layer that isn't doing anything useful for it.

**Tradeoff:** this is more moving parts to configure and reason about than GCP's single product — three services with three sets of health-check semantics instead of one. Accepted because each AWS service is purpose-built for its slice (edge caching vs. anycast dynamic routing) and composes cleanly; the operational cost is initial setup complexity, not ongoing overhead once configured.

## 2. Chat Gateway: EKS over Fargate/Lambda, same reasoning as GKE

Same argument as the GCP doc, mapped directly: Fargate (the Cloud Run analog) is the right default for the ingestion workers, but the chat gateway's long-lived SSE connections need:

- Pod-level connection draining control during deploys (finish in-flight streams before a pod is replaced) — native to Kubernetes lifecycle hooks on EKS, less direct to express on Fargate's task-replacement model.
- Autoscaling on **SQS queue depth**, not request count — achievable on EKS via KEDA (scaling on SQS queue metrics) with the same precision as the GCP doc's Cloud Monitoring custom-metric HPA.

**Tradeoff:** identical to the GCP doc's — EKS cluster/node-group operational overhead, accepted specifically for SSE connection control; Fargate remains the better default for the stateless, short-lived ingestion workers.

## 3. Session store: ElastiCache, region-local

Direct analog to the GCP doc's Memorystore decision — one ElastiCache for Redis cluster per region, no cross-region replication, same reasoning: session history is disposable state, and cross-region replication would tax every chat turn's write latency to protect a rare regional-failover edge case. Nothing AWS-specific changes this tradeoff.

## 4. Vector store: OpenSearch Service, per-region, replicated via CCR

This maps over cleanly from the GCP doc — the concepts and the conclusion are the same, only the mechanism for cross-region propagation differs.

| Approach | How it works | Tradeoff |
|---|---|---|
| **OpenSearch Service, one domain per region, kept in sync via Cross-Cluster Replication (CCR)** | Ingestion writes to the canonical region's OpenSearch domain; CCR follower indices in other regions replicate asynchronously | Each region serves queries against a **local index** — same latency win as the GCP doc's per-region Vector Search. Cost: index storage 3x, and CCR replication lag (typically seconds-to-low-minutes) |
| **Single-region OpenSearch domain, other regions query it remotely** | One index, all RAG workers query cross-region | No replication lag, no extra storage | Every non-local region eats a cross-region round-trip on every retrieval call — undoes the point of deploying globally, same conclusion as the GCP doc |

**Chosen: per-region replicated indexes via CCR**, for the same reason as the GCP doc: the latency win from local retrieval is the entire point of a global deployment, and the backend-scaling doc already established that vector freshness only needs eventual consistency — a CCR replication lag of seconds-to-minutes is well within that tolerance.

**Managed vs. self-hosted:** OpenSearch Service is the AWS-native managed choice (no cluster ops, k-NN plugin gives HNSW/IVF-style ANN search matching the backend-scaling doc's indexing discussion) and is the default here for the same reason Vertex AI Vector Search was the GCP default — self-hosting Milvus/Weaviate replicated across three regions on EKS multiplies an operational burden that was already significant single-region. Self-host only if OpenSearch's k-NN capabilities don't cover a specific requirement (e.g., a particular quantization scheme); if so, replicate the same way — one deployment per region fed by the canonical ingestion pipeline.

## 5. Metadata store: the central AWS-specific tradeoff — there is no Spanner

The GCP doc leaned on Spanner precisely because it uniquely offers horizontal scale *and* synchronous multi-region strong consistency in one system, which is what the backend-scaling doc's two-speed model demands for `superseded_by` version pointers. AWS has no service with that same combination. Three real options, each a genuine compromise relative to Spanner:

| Option | Consistency model | Tradeoff |
|---|---|---|
| **Aurora PostgreSQL Global Database** | Single write region; secondary regions get storage-level async replication, typically **< 1 second** lag; secondary reads are eventually consistent, not strongly consistent | Closest operational fit to the backend-scaling doc's relational partitioned-metadata model (it's Postgres-compatible) and requires no new database technology. But during that sub-second window, a secondary region *can* serve a just-superseded version — a real, if narrow, gap versus Spanner's guarantee |
| **DynamoDB Global Tables** | Multi-region, multi-writer, but conflict resolution is last-writer-wins and cross-region consistency is eventual (typically sub-second, no guarantee) | True multi-writer availability (any region can accept a write, useful if the "one canonical ingestion region" constraint is unwanted), but requires redesigning the metadata schema from relational/partitioned SQL into DynamoDB's key-value/document model — a bigger structural change than swapping the underlying database |
| **Self-hosted distributed SQL (CockroachDB or YugabyteDB) on EKS, multi-region** | True synchronous multi-region strong consistency, Spanner-equivalent | Exactly the ops burden managed services are meant to avoid — running and tuning a distributed consensus database across regions yourselves, the same category of complexity the backend-scaling doc flagged as a cost even *within* one region for the vector store |

**Recommendation: Aurora PostgreSQL Global Database as the default, with the sub-second consistency gap explicitly mitigated rather than ignored** — two mitigations, chosen together:
1. **Route version-supersession checks through the primary region** for the specific, low-frequency operation that must never be stale (checking whether a filing has been superseded) — a direct read against the Aurora *write primary* rather than a local secondary-region reader, accepting a cross-region round-trip on that one check in exchange for correctness. This mirrors the backend-scaling doc's own logic: version-pointer correctness matters enough to pay latency for, and this check is infrequent relative to the retrieval/generation path.
2. Accept eventual consistency for everything else metadata-related (company/quarter lookups, filing browsing) where the sub-second lag is immaterial — same as the GCP doc's reasoning, just with a narrower "strong" carve-out instead of blanket global strong consistency.

**When to reach for self-hosted CockroachDB/YugabyteDB instead:** if the version-pointer correctness requirement is absolute enough that even a cross-region read-to-primary isn't acceptable (e.g., regulatory requirement for zero-lag consistency), the self-hosted route is the honest answer — but it should be a deliberate call given the ops cost, not a default.

## 6. Ingestion pipeline: centralized in one region, same as the GCP doc's model

SQS and Kinesis are regional resources — unlike Pub/Sub, there's no built-in cross-region reachability for a single queue/stream. In practice this changes little here, because the GCP doc *already* centralized ingestion in one canonical region and pushed updates outward (§4/§6 of that doc); the AWS version makes that same choice, just with a service that structurally enforces it rather than merely recommending it:

- **SQS (or Kinesis Data Streams** for strictly-ordered, higher-throughput ingestion**)** in the primary region buffers incoming filings — same durable, replay-able role as the GCP doc's Pub/Sub topic.
- **Fargate workers** in the primary region run load→parse→chunk→embed→version→store, writing to S3 (with Cross-Region Replication configured to the other regions) and to the Aurora write primary, then pushing/CCR-replicating vector updates to each region's OpenSearch domain.
- Embedding generation uses the same managed-vs-self-hosted crossover as before: Bedrock embedding models for simplicity, self-hosted TEI/vLLM on EKS GPU node groups once volume justifies the fixed infra cost.

## 7. High availability specifics

- **Global Accelerator failover**: unhealthy regional ALB targets are pulled from rotation within its health-check interval (configurable, as low as 10s), rerouting new connections to the next-nearest healthy region — the AWS analog to the GCP LB's automatic regional failover.
- **Multi-AZ within each region**: EKS node groups, ALBs, ElastiCache, and OpenSearch domains all span multiple AZs by default — a single AZ failure doesn't take out a region.
- **Aurora Global Database failover**: promoting a secondary region to the new write primary is a managed operation (typically well under a minute), but — unlike Spanner — it is not automatic by default; this needs to be wired to an operational runbook or automated via Route 53 health checks + a failover Lambda, which is real operational surface Spanner's design avoided entirely.
- **Region-sequenced rollouts**: CodeDeploy or Argo CD promotes changes region-by-region (canary in one region, verify CloudWatch SLO dashboards, then promote) — same discipline as the GCP doc's Cloud Deploy sequencing.

## Summary of Core Tradeoffs

| Decision | Chose | Traded away | Because |
|---|---|---|---|
| Global Accelerator + Route 53 + CloudFront (composed) | Purpose-built routing per traffic type (dynamic vs. cacheable) | More services to configure than GCP's single Global LB | Each AWS product is optimized for its slice; conflating dynamic SSE routing with CDN caching wouldn't help either |
| EKS for chat gateway (not Fargate) | Precise SSE connection lifecycle control, SQS-depth-based autoscaling via KEDA | More operational surface than a serverless platform | Same reasoning as GKE — long-lived streaming connections need that control |
| Region-local ElastiCache sessions | Low-latency session read/write every turn | Session loss on rare cross-region failover | Disposable state; unchanged from the GCP doc's reasoning |
| Per-region OpenSearch, CCR-replicated | Local, low-latency retrieval in every region | 3x index storage, seconds-to-minutes replication lag | Local retrieval latency is the point of going global; vector freshness only needs eventual consistency |
| Aurora Global Database + primary-region reads for version checks | Postgres-compatible, no new DB technology, correctness preserved for the one check that needs it | Sub-second staleness window on secondary-region reads for everything else; a manual/automated (not built-in) failover path | No AWS service matches Spanner's synchronous multi-region strong consistency; this is the closest fit without taking on a self-hosted distributed database |
| Ingestion centralized in one region (SQS/Kinesis + Fargate) | Simple, matches the GCP doc's canonical-region model | AWS messaging primitives don't offer Pub/Sub's cross-region reach, so this isn't optional the way it was a *choice* on GCP | Ingestion propagation was already async-tolerant; centralizing is the natural fit given SQS/Kinesis are inherently regional |

The throughline: **the "replicate data vs. replicate compute" framing from the GCP doc survives the move to AWS almost entirely intact — vector search and sessions replicate compute-adjacent state per-region for latency, metadata stays centralized-ish for correctness.** The one place AWS genuinely changes the shape of the decision is the metadata store: without a Spanner-equivalent, "strong consistency for version pointers" stops being a single clean architectural choice and becomes a deliberately narrow carve-out (route just the version check to the primary region) layered on top of an otherwise-eventually-consistent Aurora Global Database — a real compromise the GCP design didn't have to make, not just a renamed service.
