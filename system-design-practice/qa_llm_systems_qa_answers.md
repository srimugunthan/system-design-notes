# LLM Systems — Q&A Reference

## 1. Transformer Architecture

At its core, a transformer processes a sequence of tokens using **self-attention** instead of recurrence or convolution, allowing it to model relationships between all tokens in a sequence in parallel.

Key components:
- **Tokenization & Embeddings**: Input text is split into tokens, mapped to dense vectors, and combined with **positional encodings** (since attention has no inherent notion of order).
- **Self-Attention**: For each token, the model computes Query (Q), Key (K), and Value (V) vectors. Attention scores are computed as `softmax(QKᵀ/√d)V`, letting each token "attend to" every other token weighted by relevance — this is how the model captures context (e.g., resolving pronoun references or long-range dependencies).
- **Multi-Head Attention**: Multiple attention "heads" run in parallel, each learning to focus on different types of relationships (syntax, coreference, semantics), and their outputs are concatenated and projected.
- **Feed-Forward Network (FFN)**: After attention, each token's representation passes through a position-wise fully connected network (typically expand-then-contract, e.g., 4x hidden size) with a non-linearity (GELU/SwiGLU).
- **Residual Connections + Layer Normalization**: Each sub-layer (attention, FFN) is wrapped with a residual/skip connection and normalization, which stabilizes training of deep stacks.
- **Stacking**: These blocks (attention + FFN) are stacked N times (e.g., 32, 80+ layers) to build depth.
- **Decoder-only vs. Encoder-Decoder**: Modern LLMs (GPT, Llama, Gemini) are largely **decoder-only** with **causal (masked) self-attention** — each token can only attend to previous tokens, enabling autoregressive next-token generation. Encoder-decoder architectures (like the original Transformer, T5) use a bidirectional encoder plus a causal decoder, useful for translation/seq2seq tasks.
- **Output**: The final layer projects hidden states to vocabulary logits, and a softmax produces a probability distribution over the next token.

The key architectural insight is that self-attention gives **O(1) path length** between any two tokens (versus O(n) for RNNs), which is what allows transformers to model long-range dependencies effectively and train efficiently in parallel across a sequence.

---

## 2. How Do You Make Sure Your LLM System Doesn't Hallucinate

There's no single fix — hallucination mitigation is a layered strategy across the pipeline:

**Grounding the model in facts**
- **RAG (Retrieval-Augmented Generation)**: Ground responses in retrieved, verifiable source documents rather than relying purely on parametric knowledge.
- **Citations/attribution**: Require the model to cite the specific source passage for each claim, which both improves faithfulness and lets you verify programmatically.
- **Fine-tuning on high-quality, domain-verified data** reduces reliance on the model's possibly-outdated or noisy pretraining knowledge for domain-specific facts.

**Prompting and generation controls**
- Explicit instructions to say "I don't know" when the answer isn't in the provided context, rather than guessing.
- Lower temperature/top-p for factual tasks to reduce speculative generation.
- Chain-of-thought or structured reasoning prompts to reduce shortcut errors on multi-step questions.

**Verification layers**
- **Faithfulness/groundedness checks**: Post-hoc verify that each generated claim is entailed by the retrieved context (using NLI models or LLM-as-judge).
- **Self-consistency checks**: Sample multiple generations and check for agreement; large disagreement is a hallucination signal.
- **Fact-checking against a knowledge base** or external tool calls (e.g., calculator, search) for verifiable claims.

**System-level guardrails**
- Confidence thresholds that trigger fallback to "I'm not sure" or human escalation.
- Human-in-the-loop review for high-stakes outputs (e.g., financial/medical/legal domains).
- Continuous monitoring in production (see Part 2, Q3–5) to catch hallucination drift over time.

In practice, the most reliable systems combine RAG + groundedness scoring + confidence-based fallback rather than relying on any single technique.

---

## 3. How Do You Evaluate the LLM System

Evaluation typically spans multiple layers:

1. **Component-level evaluation**: For RAG systems, evaluate retrieval (precision/recall of retrieved chunks, context relevance) and generation (faithfulness, answer relevance) separately — see the RAG Triad below.
2. **Task-level/offline evaluation**: Run the system against a **golden dataset** of curated input/expected-output pairs, scoring correctness, factuality, tone, and format adherence. Public benchmarks (MMLU, TruthfulQA, HellaSwag) are useful for base model comparison but rarely map directly to your production task.
3. **LLM-as-a-Judge**: Use a strong LLM to score outputs against a rubric (helpfulness, correctness, safety) at scale, since human review doesn't scale.
4. **Human evaluation**: Sample-based expert or annotator review, especially for nuanced or high-stakes domains, and to calibrate/validate the LLM-judge itself.
5. **Online evaluation**: Live production signals — user thumbs up/down, edit rates, escalation rates, session abandonment, and latency/cost metrics.
6. **Regression testing**: Maintain a suite of test cases (including known past failure cases) that's re-run on every prompt/model/RAG-pipeline change to catch regressions before deployment.
7. **A/B testing / canary rollouts**: Compare a new version against the current baseline on live traffic before full rollout.

A mature evaluation setup treats this as continuous — golden-dataset regression tests in CI, LLM-judge scoring on a sample of live traffic, and human review loops feeding back into both fine-tuning data and the golden dataset.

---

## 4. Explain the DeepEval and RAGAS Metrics

**RAGAS** (RAG Assessment) is a framework focused specifically on evaluating RAG pipelines, using reference-free, LLM-based metrics:
- **Faithfulness**: Measures whether claims in the generated answer are supported by the retrieved context (checks for hallucination relative to the retrieved documents).
- **Answer Relevance**: Measures how well the generated answer actually addresses the user's question (penalizes incomplete or off-topic answers), often computed by generating synthetic questions from the answer and comparing embedding similarity to the original question.
- **Context Precision**: Measures whether the retrieved context chunks that are actually relevant are ranked highly (i.e., useful chunks aren't buried).
- **Context Recall**: Measures whether the retrieval step successfully retrieved all the necessary information needed to answer the question, typically compared against a ground-truth reference answer.
- Additional metrics include **Context Relevancy** and **Answer Correctness/Semantic Similarity** against a reference answer.

**DeepEval** is a broader open-source LLM evaluation framework (designed to feel like "Pytest for LLMs") that includes RAGAS-style metrics plus additional ones, generally implemented as LLM-as-judge or statistical metrics:
- **G-Eval**: A flexible, custom-rubric LLM-judge metric where you define your own evaluation criteria in natural language and DeepEval converts it into a chain-of-thought scoring prompt.
- **Faithfulness, Answer Relevancy, Contextual Precision/Recall/Relevancy**: Same conceptual metrics as RAGAS, reimplemented within DeepEval's framework.
- **Hallucination metric**: Directly scores whether an output contradicts a provided source document.
- **Bias and Toxicity metrics**: Screen outputs for problematic content.
- **Task-specific metrics**: Summarization quality, tool-correctness (for agents), and conversational metrics like knowledge retention across turns.
- Integrates into standard test suites (pytest-style assertions), CI/CD pipelines, and supports both unit-test-style evaluation and larger-scale batch evaluation with dashboards.

The practical difference: RAGAS is narrower and RAG-specific, with a simpler, well-established metric set; DeepEval is a more general-purpose testing framework that includes RAG metrics as one category among many (agents, chatbots, safety), and is built to integrate into a software test/CI workflow.

---

## 5. What Guardrails Are Needed in an LLM System

Guardrails generally fall into these categories:

- **Input guardrails**:
  - **Prompt injection / jailbreak detection**: Classifiers or pattern-matching to detect attempts to override system instructions.
  - **PII detection/redaction**: Scrub or block sensitive personal data before it reaches the model or logs.
  - **Topic/scope restriction**: Ensure queries fall within the system's intended domain (e.g., a banking assistant shouldn't answer unrelated medical questions).
  - **Input validation**: Length limits, encoding checks, rate limiting to prevent abuse.

- **Output guardrails**:
  - **Toxicity/harmful content filters**: Block hate speech, self-harm content, harassment.
  - **Factuality/groundedness checks**: Verify claims against retrieved context before returning a response (especially for RAG systems).
  - **PII leakage filters**: Prevent the model from echoing back sensitive data it may have seen in context or training.
  - **Format/schema validation**: For structured outputs (JSON, function calls), validate against a schema before downstream use.
  - **Brand/tone/compliance checks**: Especially relevant in regulated industries (e.g., financial services) — ensure outputs don't constitute unauthorized advice or violate compliance language requirements.

- **Behavioral/system guardrails**:
  - **Tool-use guardrails**: For agentic systems, restrict which tools/actions can be invoked, require human approval for high-risk actions (payments, deletions), and sandbox execution environments.
  - **Rate limiting & cost controls**: Prevent runaway loops or excessive token usage (especially important for agent loops).
  - **Fallback/escalation logic**: Route to a human or a safe default response when confidence is low or a guardrail is triggered.

- **Monitoring/observability guardrails**:
  - Real-time logging and alerting on guardrail trigger rates, enabling detection of new attack patterns (see Part 2, Q5).

The general principle is defense-in-depth: no single guardrail is sufficient, so production systems layer input filtering, output filtering, tool/action restrictions, and continuous monitoring.

---

## When Would You Choose Fine-Tuning Over Prompt Engineering or RAG?

A general decision framework:

- **Prompt engineering** is the cheapest and fastest lever — try it first. It's sufficient when the base model already "knows" the task/domain and just needs better instructions, examples (few-shot), or output formatting.
- **RAG** is the right choice when the problem is a **knowledge gap** — the model needs access to information it wasn't trained on, that changes frequently, or that must be traceable/citable (e.g., internal company docs, recent events, per-customer data). RAG is generally cheaper to maintain than fine-tuning since updating the knowledge base doesn't require retraining.
- **Fine-tuning** is the right choice when the problem is a **behavior gap**, not a knowledge gap — for example:
  - You need the model to consistently follow a specific output format, tone, or style that prompting can't reliably enforce.
  - You need to teach a new skill or reasoning pattern not well elicited by prompting (e.g., a specialized classification schema, a proprietary structured output format, domain-specific jargon/reasoning).
  - You need to reduce latency/cost by baking a long, complex system prompt or set of few-shot examples into model weights instead of sending them on every call.
  - You need the model to reliably refuse or handle edge cases that prompting alone handles inconsistently.
  - You have **high-quality, sufficient labeled data** (typically hundreds to thousands of examples minimum) to actually move model behavior.

In practice, these aren't mutually exclusive — many production systems combine all three: a fine-tuned model for consistent behavior/format, RAG for up-to-date/proprietary knowledge, and prompt engineering to steer specific interactions. Fine-tuning has the highest cost (data curation, training compute, evaluation, ongoing maintenance as the base model or requirements change), so it's usually the last lever pulled, only after prompting and RAG prove insufficient.

---

## How Do You Create and Validate Datasets for Fine-Tuning?

**Creating the dataset**
- **Source data**: Curate from real production logs (with sensitive data scrubbed), synthetic generation (using a stronger LLM to generate examples, then filtering), or expert-authored examples for high-stakes tasks.
- **Format consistency**: Structure examples as instruction/input/output triples (or chat-format conversations) matching exactly how the model will be prompted at inference time.
- **Coverage**: Ensure the dataset spans the distribution of real use cases, including edge cases, ambiguous inputs, and desired refusal/fallback behaviors — not just "happy path" examples.
- **Diversity and de-duplication**: Avoid overrepresenting a narrow slice of scenarios or near-duplicate examples, which biases the model.
- **Label quality**: Use multiple annotators with clear rubrics for subjective tasks, measure inter-annotator agreement, and adjudicate disagreements.

**Validating the dataset**
- **Train/validation/test split**: Hold out a validation set (for hyperparameter tuning) and a separate test set (for final evaluation) that the model never sees during training.
- **Data quality audits**: Check for label noise, contradictory examples, PII leakage, and formatting errors before training — bad data is the most common cause of fine-tuning failures.
- **Distribution matching**: Verify the training distribution matches the expected production distribution (task types, difficulty, input length).
- **Bias/safety review**: Screen for harmful, biased, or policy-violating content in examples, since the model will directly learn from them.
- **Iterative evaluation loop**: After training, evaluate on the held-out test set plus a golden production-representative set, using both automated metrics and human review; use failure analysis to identify what additional data is needed, and iterate.

---

## How Do You Fix Catastrophic Forgetting in Fine-Tuning

Catastrophic forgetting occurs when fine-tuning on a narrow new task degrades the model's previously learned general capabilities. Mitigations:

- **Parameter-efficient fine-tuning (PEFT)**: Use techniques like **LoRA/QLoRA** or adapters that update a small number of additional parameters while freezing the base model weights — this inherently limits how much the original knowledge can be overwritten.
- **Lower learning rates and fewer epochs**: Aggressive learning rates or excessive training on a narrow dataset are the most common causes of forgetting; smaller learning rates and early stopping (monitoring validation loss on a broader held-out set) reduce this.
- **Replay/rehearsal**: Mix in a sample of general-purpose or original pretraining/instruction-tuning data alongside the new task-specific data, so the model continues to see (and retain) the original distribution during fine-tuning.
- **Regularization techniques**: Methods like Elastic Weight Consolidation (EWC) penalize large changes to parameters deemed important for previously learned tasks.
- **Multi-task / curriculum fine-tuning**: Fine-tune jointly on the new task plus a representative sample of the original tasks the model needs to retain, rather than fine-tuning exclusively on the new narrow dataset.
- **Evaluate broadly, not narrowly**: Track performance on general benchmarks (not just the new task) before and after fine-tuning, so forgetting is caught early rather than discovered in production.
- **Model merging**: In some setups, merging weights of the fine-tuned model with the original base model (weight averaging) can partially recover lost general capability while retaining task gains.

---

## What Does "Memory" Mean in Agentic Systems, and How Would You Design Memory for Production Agents?

"Memory" in agentic systems refers to information that persists **beyond a single LLM context window/inference call**, letting an agent maintain continuity, learn from past interactions, and act coherently across turns, sessions, or tasks. It's typically decomposed into layers:

- **Short-term/working memory**: The current conversation's context window — recent turns, intermediate reasoning, tool outputs — that's directly fed into each LLM call.
- **Episodic memory**: A record of past interactions/sessions (what happened, what was decided) that can be retrieved when relevant to a new task.
- **Semantic memory**: Distilled facts/knowledge learned over time (e.g., user preferences, domain facts), often stored as structured data or embeddings rather than raw transcripts.
- **Procedural memory**: Learned patterns of "how to do things" — successful tool-use sequences or strategies that worked before.

**Designing memory for production agents:**
- **Tiered storage**: Keep the active context window small and precise; offload older/less relevant information to an external store (vector DB, key-value store, relational DB) and retrieve on demand via RAG-style similarity search or structured lookup.
- **Summarization/compression**: Periodically summarize long conversation histories into compact representations instead of keeping raw transcripts, to control context growth while preserving salient information.
- **Explicit write/read policies**: Define clearly what gets written to long-term memory (e.g., confirmed facts, user preferences) versus what's ephemeral (e.g., a single tool call's raw output), rather than persisting everything indiscriminately.
- **Retrieval relevance**: Use embeddings/metadata (recency, importance score, task relevance) to retrieve only the memory items relevant to the current task, avoiding context bloat and irrelevant distraction.
- **Versioning and correction**: Allow memory to be updated or invalidated (e.g., a user's preference changes) rather than only appending, to avoid the agent acting on stale facts.

### 1. How do you prevent memory from growing unbounded?

- **Summarization and compaction**: Regularly compress older memory into summaries, discarding raw detail once it's been distilled.
- **Importance/relevance scoring and pruning**: Score memory items on recency, frequency of access, and task relevance; evict or archive low-value items (similar to a cache eviction policy — e.g., LRU or relevance-weighted decay).
- **TTLs (time-to-live)**: Expire ephemeral memory (e.g., session-scoped facts) automatically after a defined period unless explicitly promoted to long-term storage.
- **Retrieval-based context construction**: Rather than growing the context window, store memory externally and retrieve only the top-k relevant items per query — the context window itself stays bounded regardless of how much total memory accumulates.
- **Hierarchical memory limits**: Cap the size of each memory tier (e.g., max N episodic entries, max size of semantic memory store) and enforce consolidation/summarization when limits are approached.

### 2. How do you avoid leaking sensitive information through memory?

- **PII detection and redaction at write time**: Scan content before it's persisted to memory and redact or tokenize sensitive fields (names, account numbers, health data) unless explicitly required and authorized for storage.
- **Access scoping**: Enforce memory isolation per user/tenant, ensuring one user's session or agent instance cannot retrieve another's memory (critical in multi-tenant systems).
- **Encryption at rest and in transit** for any persisted memory store, with strict IAM controls on who/what can query it.
- **Explicit consent and retention policies**: Only persist data the user has consented to store, and enforce deletion/right-to-be-forgotten workflows.
- **Guardrails on retrieval/output**: Even if sensitive data is stored, apply output-side filters to prevent the agent from surfacing it inappropriately (e.g., in a shared conversation, or to an unauthorized requester).
- **Audit logging**: Track what memory was written, read, and by which agent/session, to support compliance review and incident investigation.
- **Minimization principle**: Store only what's operationally necessary (e.g., a distilled preference, not a full raw transcript containing incidental sensitive details).

---


# Part 1: LLM Deployment

## 1. Deployment Architecture: Managed Cloud API vs. Self-Hosting

| Consideration | Managed Cloud API (Gemini, OpenAI) | Self-Hosted Open-Weight Model |
|---|---|---|
| **Operational overhead** | None — fully managed, no infra to maintain | High — you manage GPU provisioning, scaling, patching, model serving stack |
| **Cost model** | Pay-per-token, no upfront cost, but can be expensive at high volume | High upfront/fixed GPU cost, but can be cheaper at large sustained volume |
| **Latency control** | Limited — dependent on provider's infrastructure and rate limits | Full control — can co-locate with data, optimize serving stack |
| **Customization** | Limited to prompting/fine-tuning APIs offered by the provider | Full control — custom fine-tuning, quantization, architecture modifications |
| **Data privacy/compliance** | Data leaves your environment (subject to provider's data-handling terms) | Full data residency and control — important for regulated industries |
| **Model quality/capability** | Access to frontier, state-of-the-art models | Open-weight models often lag frontier proprietary models in capability |
| **Time to production** | Fast — integrate via API immediately | Slower — requires infra setup, optimization, and validation |
| **Scalability** | Provider handles scaling, but you're subject to their rate limits/quotas | You must engineer your own scaling, but have no external rate limits |

The general decision hinges on: **data sensitivity/compliance requirements**, **cost at your expected volume**, **latency/control requirements**, and **whether frontier model capability is necessary** versus a good-enough open-weight model.

## 2. Inference Optimization: Quantization and KV Caching

- **Quantization** (INT8/INT4): Reduces the numerical precision of model weights (and sometimes activations) from FP16/FP32 down to 8-bit or 4-bit integers. This directly shrinks the model's memory footprint (allowing larger models to fit on smaller/fewer GPUs) and speeds up compute (lower-precision arithmetic is faster on supporting hardware), at the cost of some accuracy degradation — modern quantization techniques (GPTQ, AWQ, bitsandbytes) minimize this loss significantly for INT8 and increasingly for INT4.
- **KV Caching**: During autoregressive generation, each new token's attention computation would otherwise require recomputing Key/Value projections for all previous tokens. KV caching stores these K/V tensors from prior steps so only the new token's K/V needs computing at each step — turning what would be O(n²) recomputation into incremental O(n) work per token. This dramatically reduces per-token latency during generation, at the cost of additional GPU memory to hold the growing cache (which is why techniques like paged attention / continuous batching exist to manage KV cache memory efficiently across concurrent requests).

Together, these are two of the most impactful levers for making LLM inference both cheaper (via quantization's memory savings, enabling higher batch sizes/throughput per GPU) and faster (via KV caching's reduced per-token compute).

## 3. Canary Deployment for New System Prompts / Fine-Tuned Models

A canary deployment routes a small percentage of live traffic to the new version while the majority continues on the stable version, gradually increasing traffic if metrics look healthy. This is particularly valuable for LLM changes because:
- **Non-deterministic, hard-to-predict impact**: Unlike a typical code deploy, a new system prompt or fine-tuned model can subtly shift behavior in ways that are difficult to fully catch in offline eval (tone, refusal rates, hallucination rates, edge-case handling).
- **Real user feedback signal**: Canary traffic exposes the new version to genuine, diverse production inputs (which are hard to fully replicate in test sets), surfacing regressions that offline golden-dataset evaluation might miss.
- **Limits blast radius**: If the new prompt/model version has a safety, quality, or cost regression, only a small fraction of users are affected before it's caught and rolled back — critical for LLM systems where a single bad output (e.g., a hallucinated financial figure) can cause real harm.
- **Statistical comparison**: Running both versions concurrently allows a direct A/B comparison of live metrics (thumbs-up rate, escalation rate, latency, cost-per-request) under identical real-world conditions, which is more reliable than sequential before/after comparisons.

Blue-green deployment (instant full cutover with instant rollback capability) is more appropriate when you're confident in correctness and mainly want zero-downtime infra switching; canary is preferred specifically when the *behavioral* quality of the new version is uncertain — which is almost always the case with prompt/model changes.

## 4. Context Window & Cost Control: Token Length Impact

- **Input tokens (prompt length)** directly affect:
  - **Time-to-First-Token (TTFT)**: Longer prompts take longer to process through the initial forward pass (prefill phase) before generation begins, since the model must compute attention over the entire input before producing the first output token.
  - **Cost**: Most providers charge per input token, so longer prompts (e.g., large RAG context, long conversation history) directly increase per-request cost, even before any output is generated.
  - **Compute/memory**: Longer inputs increase the KV cache size needed to hold the context, consuming more GPU memory and potentially reducing achievable batch size (and therefore throughput).

- **Output tokens (response length)** directly affect:
  - **Tokens-per-Second (TPS) / total generation latency**: Since generation is autoregressive (one token at a time), longer responses take proportionally longer to fully generate — total latency scales roughly linearly with output length.
  - **Cost**: Output tokens are typically priced higher than input tokens by providers (since generation is more compute-intensive per token than prefill), so verbose responses disproportionately increase cost.

**Practical implications**: Minimizing unnecessary context (e.g., retrieving only the most relevant RAG chunks rather than stuffing the whole knowledge base, summarizing long conversation history) reduces both TTFT and cost. Constraining output length (via `max_tokens`, prompting for conciseness, or structured output formats) reduces both total latency and cost. Systems optimizing for a good user experience typically track and control both dimensions separately, since they affect different parts of the latency/cost equation (TTFT vs. total completion time).

## 5. RAG & Agentic Infrastructure: Decoupling Retrieval and Generation

Retrieval (Vector DB) and generation (LLM) should be decoupled and monitored separately because:
- **Different failure modes**: A bad answer can stem from *poor retrieval* (relevant documents weren't found/ranked highly) or *poor generation* (the LLM ignored or misinterpreted good context, or hallucinated despite good retrieval). Without separating these, debugging becomes guesswork — you can't tell which component to fix.
- **Independent scaling and latency profiles**: Vector DB lookups and LLM inference have very different latency/throughput characteristics and scale independently (e.g., you might need to scale vector DB replicas for read throughput while GPU capacity is the bottleneck for generation) — monitoring them separately lets you identify which is the actual bottleneck.
- **Independent evaluation metrics**: Retrieval quality is measured with metrics like Context Precision/Recall (did we retrieve the right, sufficient information?), while generation quality is measured with Faithfulness/Answer Relevance (did the model use that information correctly?). Conflating them into a single end-to-end score obscures which part of the pipeline needs improvement (this is exactly why the RAG Triad splits these out — see Part 2, Q4).
- **Independent lifecycle/versioning**: The embedding model, chunking strategy, or index can be updated independently of the generation model/prompt — decoupled monitoring lets you attribute a quality change to the specific component that was modified.
- **Operational resilience**: Monitoring each component's health (Vector DB query latency/availability, LLM API latency/errors) separately enables faster root-cause diagnosis and targeted alerting/on-call routing when something breaks.

---

# Part 2: LLM Monitoring & Evaluation

## 1. Traditional ML Monitoring vs. LLM Monitoring

Traditional tabular ML monitoring relies on **well-defined ground truth and deterministic metrics**: precision/recall/F1/AUC against labeled outcomes, feature drift detection (statistical distribution shifts), and prediction drift — all computable automatically and objectively because the output space is structured (a class label, a numeric score).

LLM monitoring is fundamentally harder because:
- **Output is unstructured, open-ended text**: There's no single "ground truth" string to compare against for most tasks — correctness is often subjective, contextual, or has many valid phrasings.
- **Non-determinism**: The same input can produce different outputs across calls (especially with non-zero temperature), so monitoring must account for output variance, not just track a single value against a threshold.
- **Multi-dimensional quality**: A response can be factually correct but poorly formatted, or fluent but hallucinated, or relevant but unsafe — quality isn't a single scalar, requiring multiple parallel metrics (faithfulness, relevance, toxicity, tone).
- **Proxy metrics required**: Since automatic ground-truth scoring is often infeasible, LLM monitoring relies more heavily on **proxy signals** — LLM-as-judge scores, user feedback (thumbs up/down, edit distance), and heuristic classifiers (toxicity, PII detection) — rather than exact-match accuracy metrics.
- **Semantic drift instead of statistical drift**: Traditional feature drift (e.g., a numeric feature's distribution shifting) has a direct analog in embedding-space drift for LLM inputs/outputs, but is noisier and harder to interpret directly.

In short, LLM monitoring shifts from purely statistical, ground-truth-based metrics toward a blend of automated proxy scoring (LLM-judges, classifiers), human-in-the-loop sampling, and behavioral/business metrics (engagement, escalation rates).

## 2. Offline Evaluation vs. Online Monitoring

- **Offline evaluation** happens before deployment, against a **static, curated dataset** — golden Q&A pairs, benchmark suites (MMLU for knowledge, TruthfulQA for factuality/hallucination resistance, HellaSwag for commonsense reasoning), or a domain-specific regression test suite. It's controlled, repeatable, and used for comparing model/prompt versions in CI before shipping — but it can't capture the full diversity or drift of real-world usage.
- **Online monitoring** happens continuously in production against **live traffic** — tracking real user feedback (thumbs up/down, explicit corrections), operational metrics (latency, error rates, cost per request), and live quality probes (sampling a subset of production responses for automated hallucination/faithfulness scoring or human review). It catches issues offline evaluation can't anticipate — data drift, adversarial usage patterns, edge cases not represented in the golden set, and real-world latency/cost under actual load.

The two are complementary: offline evaluation is your pre-deployment gate (preventing known regressions from shipping), while online monitoring is your safety net for the unknown unknowns that only show up once real users interact with the system. Mature systems feed production failures discovered via online monitoring back into the offline golden dataset, closing the loop.

## 3. LLM-as-a-Judge

**How it works**: A (typically strong, frontier-tier) LLM is prompted with the original input, the response being evaluated, and a scoring rubric or reference answer, and asked to produce a score or verdict (e.g., "rate this response's helpfulness 1–5" or "is this response A or response B better?"). This scales evaluation far beyond what human review can achieve, since it can be run automatically over large volumes of production or test data.

**Main limitations**:
- **Positional bias**: When comparing two responses (A vs. B), the judge model tends to systematically favor whichever response appears first (or second) in the prompt, regardless of quality — mitigated by randomizing/swapping positions and averaging.
- **Verbosity bias**: Judges tend to rate longer, more elaborate responses as higher quality even when they're not more correct or useful — a well-known confound in LLM-judge setups.
- **Self-enhancement bias**: A judge model tends to rate outputs from its own model family more favorably than outputs from other models, which is a concern when using, say, GPT to judge GPT-generated content.
- **Rubric sensitivity**: Judge scores can vary significantly based on how the scoring prompt/rubric is worded, making results less stable/reproducible than they first appear.
- **Cost and latency**: Running an LLM call to judge every other LLM call scales cost and adds latency, especially at high evaluation volume.
- **Judge accuracy ceiling**: The judge itself can make errors or share blind spots with the model being evaluated (e.g., both may fail to catch the same subtle factual error), so LLM-judge scores should be periodically validated/calibrated against human judgment rather than trusted blindly.

## 4. The RAG Triad Metrics

1. **Faithfulness / Groundedness**: Measures whether the claims made in the generated answer are actually supported by (entailed by) the retrieved context — i.e., is the model hallucinating or fabricating information not present in the source documents it was given? This isolates *generation* quality relative to the provided context.
2. **Answer Relevance**: Measures whether the generated answer actually addresses the user's original question — a response can be perfectly faithful to the retrieved context yet still fail to answer what was asked (e.g., off-topic or incomplete). This isolates whether the *generation* step correctly used the context to serve the user's intent.
3. **Context Relevance**: Measures whether the retrieved context/documents are actually relevant to the user's question in the first place — if retrieval surfaces irrelevant or noisy chunks, even a perfect generation step can't produce a good answer. This isolates *retrieval* quality independent of generation.

Together, these three metrics let you pinpoint exactly where a RAG failure originates: bad retrieval (low Context Relevance), the model ignoring good context (low Faithfulness), or the model answering a different question than what was asked (low Answer Relevance) — rather than having only a single opaque end-to-end quality score.

## 5. Security & Guardrail Observability

Production metrics/indicators to monitor for adversarial behavior:
- **Guardrail trigger rate**: Frequency of input/output filters firing (prompt injection detectors, jailbreak classifiers, toxicity filters, PII detectors) over time — spikes indicate either a new attack pattern or a change in user population/behavior.
- **Anomalous prompt patterns**: Statistical or embedding-based outlier detection on incoming prompts (unusually long prompts, unusual token distributions, known jailbreak phrase patterns like "ignore previous instructions").
- **Refusal rate tracking**: Sudden increases or decreases in the model's refusal rate can indicate either a wave of adversarial probing or a guardrail miscalibration causing over/under-blocking.
- **PII/sensitive-data detection hits**: Rate of PII detected in either inputs or outputs, especially any detected PII that appears in *outputs* it shouldn't have surfaced (a leakage signal).
- **Output-input divergence / instruction-override detection**: Signals that the model's behavior deviated from its system prompt/intended persona (e.g., the assistant starts responding in an unexpected format or tone), which can indicate a successful prompt injection.
- **Tool/action call anomalies** (for agentic systems): Unexpected or high-risk tool invocations, unusual sequences of tool calls, or attempts to invoke tools outside the expected scope for the given task.
- **Rate/volume anomalies**: Spikes in request volume from a single user/session/IP that could indicate automated adversarial probing or scraping.
- **Latency/cost outliers**: Adversarial inputs designed to maximize token usage (e.g., prompt-based denial-of-wallet attacks) often show up as unusual cost or latency spikes per request.
- **Feedback loop from red-teaming**: Continuously incorporate newly discovered jailbreak/injection patterns (from internal red-teaming or external disclosures) into both the detection classifiers and the monitored signal set, since adversarial techniques evolve faster than static rule sets.

Effective guardrail observability treats these as a real-time dashboard with alerting thresholds (similar to a security operations center), not just a periodic offline audit — since adversarial behavior needs fast detection and response.
