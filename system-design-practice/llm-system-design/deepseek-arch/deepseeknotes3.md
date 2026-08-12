This article by [Shakti Wadekar](https://pub.towardsai.net/deepseek-r1-model-architecture-853fefac7050) provides a detailed architectural breakdown of **DeepSeek-R1**, which is built upon the **DeepSeek-V3-Base** model. 

The architecture is designed to balance massive scale with inference efficiency through several key innovations:

### 1. Multi-Head Latent Attention (MLA)
MLA is the core efficiency driver. Unlike standard attention mechanisms that create heavy Key-Value (KV) caches, MLA uses **low-rank joint compression**. 
* **Compression:** It projects Q, K, and V into smaller dimensions (latent vectors) to reduce the memory footprint during inference.
* **Decoupled RoPE:** It uses a unique strategy to integrate **Rotary Positional Embeddings (RoPE)** by generating separate positional queries and keys, ensuring that positional info doesn't interfere with the matrix compression benefits.

### 2. Mixture-of-Experts (MoE)
DeepSeek-R1 utilizes a sparse MoE structure to scale parameters without a linear increase in compute costs.
* **Structure:** It consists of 61 transformer layers. The first three are "dense," while layers 4 through 61 use MoE.
* **Activation:** Out of **671B total parameters**, only **37B are activated** per token.
* **Routing:** It employs 1 shared expert and 8 routed experts per token, using a sigmoid-based gating mechanism with a bias term to ensure load balancing without auxiliary loss.

### 3. Multi-Token Prediction (MTP)
Used primarily during **training**, MTP helps the model pre-plan representations and increases training signals.
* **Sequential Chain:** Unlike Meta’s parallel approach, DeepSeek-V3/R1 predicts two tokens sequentially to maintain a complete causal chain.
* **Inference:** These modules are typically discarded during inference to keep the main model independent and efficient.

### 4. Context and Training
* **Context Length:** The model supports a **128K context window**, extended via the **YaRN** (Yet another RoPE extensioN) technique.
* **Architecture Evolution:** The design evolved through previous iterations like **DeepSeek-V2** and **DeepSeekMoE**, focusing on high performance with "economical" compute requirements.

Do you have questions about a specific part of this architecture, like how the MLA compression works mathematically?

https://pub.towardsai.net/deepseek-r1-model-architecture-853fefac7050 
--
--
This article by Zain ul Abideen provides a technical walkthrough for implementing the core components of [DeepSeek-V2](https://medium.com/@zaiinn440/coding-deepseek-v2-from-scratch-in-pytorch-06dd89917067) using PyTorch. DeepSeek-V2 is a massive Mixture-of-Experts (MoE) language model with **236B total parameters**, designed to reduce training costs and KV cache requirements significantly compared to traditional dense models.

The summary of the architectural innovations covered includes:

### 1. DeepSeekMoE (Feed-Forward Networks)
To improve expert specialization and reduce redundancy, the architecture employs two main strategies:
* **Fine-Grained Expert Segmentation:** Splits the FFN intermediate hidden dimension into smaller segments for more targeted knowledge acquisition.
* **Shared Expert Isolation:** Certain experts are always activated to capture "common knowledge," preventing multiple routed experts from converging on the same information.

### 2. Multi-head Latent Attention (MLA)
MLA is designed to boost inference efficiency by reducing the **KV cache** (up to **93.3%**):
* **Low-Rank Compression:** Jointly compresses Keys and Values into a latent vector, requiring less memory than standard Multi-Head Attention (MHA).
* **Decoupled RoPE:** Uses a unique strategy to make Rotary Position Embeddings compatible with low-rank compression by using additional multi-head queries and a shared key for positional info.

### 3. Implementation Details
The author provides PyTorch code snippets for:
* **`MoEGate`:** A module supporting both "greedy" and "group-limited greedy" routing logic.
* **`MoE`:** The top-level class combining routed and shared experts.
* **`MLA`:** The attention mechanism incorporating LoRA (Low-Rank Adaptation) projects for queries and keys/values.
  
https://medium.com/@zaiinn440/coding-deepseek-v2-from-scratch-in-pytorch-06dd89917067 
---

--
--
The article [Building DeepSeek R1 from Scratch Using Python](https://levelup.gitconnected.com/building-deepseek-r1-from-scratch-using-python-a967fbfdaac4) by Fareed Khan provides a comprehensive technical guide to replicating the DeepSeek R1 reasoning model architecture using a smaller base model, **Qwen2.5-0.5B-Instruct**.

The training process is broken down into two major phases:

### 1. The R1 Zero Phase (Pure Reinforcement Learning)
This stage focuses on developing reasoning capabilities from scratch without initial human examples.
* **Algorithm:** Uses **Group Relative Policy Optimization (GRPO)**, which eliminates the need for a separate "critic" model, saving significant computational resources.
* **Reward System:** The model is trained using five specific reward functions:
    * **Accuracy:** Verifies mathematical correctness using LaTeX parsing.
    * **Format:** Ensures the model uses `<think>` and `<answer>` tags correctly.
    * **Reasoning Steps:** Encourages structural logic (e.g., "Step 1", "Step 2").
    * **Cosine Scaling:** Balances conciseness for correct answers and persistence for wrong ones.
    * **Repetition Penalty:** Discourages the model from getting stuck in infinite loops.

---

### 2. The R1 Phase (Refinement & SFT)
To fix issues in R1 Zero—such as "language mixing" and messy reasoning—the author details a more structured pipeline:
* **Cold Start Data:** Small, high-quality datasets (like **Bespoke-Stratos-17k**) are used to provide the model with "long chain-of-thought" (CoT) examples.
* **Supervised Fine-Tuning (SFT):** The model is trained on these examples to establish a baseline for clear, readable reasoning.
* **Reasoning-Oriented RL:** A final round of RL that includes a **Language Consistency Reward** to ensure the model responds in the same language as the prompt.
* **Distillation:** The article concludes by explaining how the knowledge of a large model (like the original DeepSeek R1) can be transferred into smaller "student" models to make them more efficient for general use.
**Note:** The article mentions that the full pre-training scripts and complete code will be released on the author's [GitHub](https://medium.com/@zaiinn440/coding-deepseek-v2-from-scratch-in-pytorch-06dd89917067) soon.
