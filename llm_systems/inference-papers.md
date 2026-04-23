Good, I have all I need. Here is the reading list, structured with proper annotation per paper.

---

## Foundational Reading List: Modern LLM Inference

This is structured in dependency order — each layer builds conceptually on the one before it. Reading them in sequence gives you a coherent mental model rather than a pile of disconnected techniques.

---

### Layer 0 — The Foundation You Must Start With

**[1] Vaswani et al. (2017) — Attention Is All You Need**
NeurIPS 2017. arXiv:1706.03762
https://arxiv.org/abs/1706.03762

Everything in LLM inference is downstream of this paper. The transformer architecture, the self-attention mechanism, the encoder-decoder structure — this is the original. You cannot read any inference optimization paper meaningfully without understanding what is being optimized. If you have read it before, re-read the attention computation section with inference cost in mind: the quadratic complexity in sequence length is the root cause of half the problems all subsequent papers try to solve.

---

### Layer 1 — The Bottleneck Papers

These three papers collectively define the problem space of modern LLM inference. Each identifies a different bottleneck — attention computation, memory management, and scheduling — and their solutions are now combined in every serious production serving system.

**[2] Dao et al. (2022) — FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness**
NeurIPS 2022. arXiv:2205.14135
https://arxiv.org/abs/2205.14135

The key insight is that standard attention is slow not because of compute but because of memory: it materializes the full N×N attention matrix in GPU HBM. FlashAttention is an IO-aware exact attention algorithm that uses tiling to reduce memory reads/writes between HBM and on-chip SRAM. The result is exact attention (no approximation) that is substantially faster. This paper reframes LLM inference as an IO-bound problem rather than a compute-bound one — a mental model shift that matters for everything that follows.

**[3] Yu et al. (2022) — Orca: A Distributed Serving System for Transformer-Based Generative Models**
OSDI 2022. https://www.usenix.org/conference/osdi22/presentation/yu

Orca proposes iteration-level scheduling, a mechanism that schedules execution at the granularity of a single iteration rather than a full request, so that requests finishing early can return immediately and new requests can join the batch without waiting for the current batch to complete. This is the paper that introduced continuous batching to LLM serving. The result was a 36.9x throughput improvement over NVIDIA FasterTransformer at equivalent latency — a surprisingly large gain from infrastructure changes alone, with no modification to the model.

**[4] Kwon et al. (2023) — Efficient Memory Management for Large Language Model Serving with PagedAttention**
SOSP 2023. arXiv:2309.06180
https://arxiv.org/abs/2309.06180

vLLM achieves near-zero waste in KV cache memory and flexible sharing of KV cache within and across requests by introducing PagedAttention — an attention algorithm inspired by virtual memory and paging in operating systems. This solves the problem Orca left open: Orca reserved memory for the maximum sequence length upfront, causing waste. PagedAttention allocates KV cache in fixed-size blocks on demand, eliminating fragmentation. This is the paper that makes the current serving paradigm work at scale.

---

### Layer 2 — Decoding Acceleration

These papers address the sequential token-generation bottleneck specifically — the hard floor on inter-token latency that batching and memory management alone cannot fix.

**[5] Leviathan et al. (2023) — Fast Inference from Transformers via Speculative Decoding**
ICML 2023. PMLR pp. 19274–19286
https://arxiv.org/abs/2211.17192

The theoretical foundation of speculative decoding. Establishes the lossless sampling criterion: if a draft model generates candidate tokens and a target model verifies them in parallel using a modified rejection sampling scheme, the output distribution is preserved exactly. Acceptance rate is proven to be the sufficient statistic for predicting speedup.

**[6] Chen et al. (2023) — Accelerating Large Language Model Decoding with Speculative Sampling**
arXiv:2302.01318
https://arxiv.org/abs/2302.01318

This paper presents speculative sampling and relies on the observation that the latency of parallel scoring of short continuations from a faster draft model is comparable to sampling a single token from the larger target model, combined with a modified rejection sampling scheme that preserves the target model's distribution. The companion paper to Leviathan et al., published concurrently by DeepMind. Stronger on empirical results and practical implementation details. Read both; they are complementary.

---

### Layer 3 — Quantization

Quantization is the other primary lever for reducing inference cost, orthogonal to the batching and decoding papers above. These two papers are the most practically relevant.

**[7] Dettmers et al. (2022) — LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale**
NeurIPS 2022. arXiv:2208.07339
https://arxiv.org/abs/2208.07339

Introduces mixed-precision decomposition to handle outlier activations that break naive int8 quantization. Practically important because it makes running large models on smaller GPUs feasible without significant accuracy loss. This is the paper behind the `load_in_8bit` flag you see everywhere.

**[8] Lin et al. (2023) — AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration**
arXiv:2306.00978
https://arxiv.org/abs/2306.00978

AWQ (Activation-aware Weight Quantization) identifies that a small fraction of weights are disproportionately important and should be protected during quantization. Produces better quality than GPTQ at 4-bit precision for most tasks. This is the technique behind the GPTQ/AWQ quantized model variants you will use in any GPU-constrained project including this one.

---

### Layer 4 — Distributed Serving and the Serving Stack

**[9] Moritz et al. (2018) — Ray: A Distributed Framework for Emerging AI Applications**
OSDI 2018. arXiv:1712.05889
https://arxiv.org/abs/1712.05889

Ray implements a unified interface that can express both task-parallel and actor-based computations under a single dynamic execution engine, with a distributed scheduler and fault-tolerant store to manage control state. Ray Serve is built directly on top of this foundation. Reading this paper gives you the systems model underlying Ray Serve's autoscaling, replica management, and request routing.

**[10] Shoeybi et al. (2019) — Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism**
arXiv:1909.08053
https://arxiv.org/abs/1909.08053

The foundational paper on tensor parallelism for large models. Relevant for inference because the same techniques — splitting attention heads and MLP layers across GPUs — apply directly to multi-GPU inference serving. If you ever serve a model that does not fit on a single GPU, this is the conceptual prerequisite.

---

### Survey Paper (Read Last, Use as a Map)

**[11] Zhou et al. (2024) — A Survey on Efficient Inference for Large Language Models**
arXiv:2404.14294
https://arxiv.org/abs/2404.14294

This survey identifies the primary causes of inefficient LLM inference as the large model size, the quadratic-complexity attention operation, and the autoregressive decoding approach, and organizes the literature into data-level, model-level, and system-level optimization with comparative experiments on representative methods. Read this after the papers above, not before. It is most useful as a navigation tool once you have enough context to evaluate what the survey is pointing at.

---

### Reading Order Recommendation

Start with paper 1 (Vaswani) if your transformer fundamentals are rusty, then read 2, 3, and 4 as a set — they are the core of the modern serving stack and each paper explicitly references its predecessors. Then read 5 and 6 together since they are parallel publications. Pick up 7 and 8 before your weekend project since you will be using quantized models. Papers 9 and 10 are background reading for the serving layer; you can defer them until after the project if time is short. Paper 11 is the capstone survey to read when you want a map of what else is out there.
