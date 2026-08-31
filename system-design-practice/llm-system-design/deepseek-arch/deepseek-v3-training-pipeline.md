Here's the full training pipeline, covering both DeepSeek-V3 (base/chat model) and DeepSeek-R1 (reasoning model), since R1 is built on top of V3 and the two pipelines are intertwined.

## Part 1: DeepSeek-V3 Pipeline

### Stage 1 — Pre-training
Pre-training was completed on 14.8T tokens at a cost of only 2.664M H800 GPU hours, producing the strongest open-source base model at the time. Key architectural/systems choices that made this efficient:
- **Architecture**: 671B total parameters, 37B activated per token (Mixture-of-Experts), plus Multi-head Latent Attention (MLA) for efficient KV-cache.
- **Auxiliary-loss-free load balancing** for MoE routing (avoids the usual auxiliary loss that hurts model quality).
- **Multi-Token Prediction (MTP)** objective — predicts multiple future tokens per step, improving sample efficiency and enabling speculative decoding at inference.
- **FP8 mixed-precision training** — one of the first large-scale successful demonstrations of FP8 training at this scale.
- **DualPipe** algorithm for pipeline parallelism to overlap computation and communication and reduce bubble time.

### Stage 2 — Context Length Extension
A two-stage context extension: first to 32K, then to 128K.

### Stage 3 — Post-training
Post-training (SFT + RL) required only about 0.1M GPU hours — roughly 3.6% of total training compute. Two sub-steps:
- **Supervised Fine-Tuning (SFT)** on curated instruction data.
- **Reinforcement Learning** for alignment/preference optimization, using a reward-model setup where DeepSeek-V3 itself was queried multiple times to judge preference pairs, averaging judgments and filtering for pairs with a clear score gap to reduce noise and positional bias.

### Stage 4 — Reasoning Distillation from R1
This is the interesting bit: DeepSeek-AI introduced a methodology to distill reasoning capability from the long-chain-of-thought DeepSeek-R1 model into DeepSeek-V3, incorporating R1's verification and reflection patterns while balancing accuracy against generation length. This is what makes V3's chat model meaningfully better at reasoning than a "pure" instruction-tuned model.

So V3's pipeline is actually circular with R1: V3-Base → R1 (via SFT+RL, below) → distill R1's reasoning back into V3 → final DeepSeek-V3 chat model.

---

## Part 2: DeepSeek-R1 Pipeline (built on V3-Base)

R1 has a genuinely distinctive multi-stage pipeline — this is the part worth understanding deeply.

### Stage 0 — DeepSeek-R1-Zero (proof of concept)
Pure RL directly on **DeepSeek-V3-Base**, no SFT at all — using **GRPO** (Group Relative Policy Optimization, a PPO variant that drops the critic network and instead normalizes reward across a group of sampled outputs). This showed reasoning behavior (long CoT, self-verification, "aha moments") could emerge from RL alone, but the outputs suffered from poor readability and language mixing.

### Stage 1 — Cold Start SFT
To fix R1-Zero's readability problems, a small amount of cold-start data was collected to fine-tune DeepSeek-V3-Base first, before any RL — giving the model a good stylistic prior (structured, readable CoT) to start from.

### Stage 2 — Reasoning-Oriented RL
After the cold-start fine-tune, reasoning-oriented RL (same GRPO-style process as R1-Zero) is applied, focused on math, coding, logic, and science. A language-consistency reward was added — computed as the proportion of target-language words in the chain-of-thought — specifically to fix language mixing observed during training.

### Stage 3 — Rejection Sampling + SFT (second SFT round)
Near convergence of that RL run, new SFT data is generated via rejection sampling on the RL checkpoint, combined with DeepSeek-V3's supervised data covering writing, factual QA, and self-cognition domains, and this combined dataset is used to retrain from DeepSeek-V3-Base again (not continuing from the RL checkpoint — a fresh SFT start). Concretely:
- **Reasoning data**: the intermediate RL model generated reasoning examples; high-quality samples were kept via rejection sampling, yielding ~600K reasoning samples.
- **Non-reasoning data**: ~200K samples covering creative writing, fact-based QA, role-play, and translation — largely reused/adapted from the DeepSeek-V3 SFT set, with V3 itself prompted to produce CoT for complex non-reasoning queries and skip CoT for trivial ones like "Hello."
- This gives roughly 800K samples total, and the model is fine-tuned for two epochs on this set.

### Stage 4 — Final RL for All Scenarios
A final RL fine-tuning stage using GRPO across diverse prompt distributions, including helpfulness, harmlessness, and safety training — not just reasoning-domain prompts. This is what produces the final DeepSeek-R1 checkpoint.

---

## Summary Flow

```
Raw web-scale corpus (14.8T tokens)
        │
        ▼
   Pre-training (FP8, MoE, MLA, MTP) ──► DeepSeek-V3-Base
        │
        ├────────────► [R1-Zero: pure GRPO RL] (proof of concept, side branch)
        │
        ▼
   Cold-start SFT (small curated CoT set)
        │
        ▼
   Reasoning-oriented RL (GRPO + language-consistency reward)
        │
        ▼
   Rejection sampling on RL checkpoint ──► ~600K reasoning + ~200K non-reasoning samples
        │
        ▼
   Re-SFT from DeepSeek-V3-Base on combined ~800K set (2 epochs)
        │
        ▼
   Final RL across all scenarios (reasoning + helpfulness + safety) ──► DeepSeek-R1
        │
        ▼
   Distill R1's reasoning traces back into DeepSeek-V3 ──► final DeepSeek-V3 chat model
```

Also worth noting: DeepSeek released **distilled dense models** (Qwen/Llama backbones fine-tuned via SFT directly on R1's 800K generated samples, no RL) — that's a separate, much cheaper pipeline for the smaller R1-Distill variants.

Given your work on RedTeamAgentLoop and RL-adjacent stuff, the GRPO mechanics (dropping the critic, group-normalized advantage) and the "why retrain from base rather than continue the RL checkpoint" design choice in Stage 3 are probably the two most transferable ideas here if you're thinking about applying similar multi-stage RL+distillation patterns to your own agentic pipelines.
