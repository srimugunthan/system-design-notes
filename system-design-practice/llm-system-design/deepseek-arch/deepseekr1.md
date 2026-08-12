This tutorial by freeCodeCamp.org explores the theoretical architecture of DeepSeek R1, a breakthrough open-source reasoning model that rivals OpenAI's o1 series.

### **Core Methodology of DeepSeek R1**
DeepSeek R1 achieves its exceptional reasoning through post-training rather than pre-training from scratch. It uses **DeepSeek V3** as its base model [[03:05](http://www.youtube.com/watch?v=K34gBCjzni8&t=185)].

The process involves several key stages:
1.  **DeepSeek R1-Zero:** Created by applying **Reinforcement Learning (RL)** directly to the base model without any supervised fine-tuning. This allowed reasoning capabilities to emerge naturally, though it initially suffered from readability issues [[04:15](http://www.youtube.com/watch?v=K34gBCjzni8&t=255)].
2.  **Supervised Fine-Tuning (SFT):** To fix language mixing and formatting, researchers used a small "cold-start" dataset to fine-tune the model before further RL [[16:54](http://www.youtube.com/watch?v=K34gBCjzni8&t=1014)].
3.  **Data Augmentation & Distillation:** The large R1 model was used to generate reasoning data, which was then used to train smaller "distilled" models (based on Llama or Qwen) [[18:49](http://www.youtube.com/watch?v=K34gBCjzni8&t=1129)].

### **Group Relative Policy Optimization (GRPO)**
GRPO is the core RL algorithm used in R1. It improves upon traditional **Proximal Policy Optimization (PPO)** by removing the need for a separate "critic" or "value" model, which reduces computational overhead [[27:57](http://www.youtube.com/watch?v=K34gBCjzni8&t=1677)].


* **Group Computation:** Instead of evaluating a single output, the model generates a group of outputs for one question. The "advantage" of an output is calculated relative to the group's mean and standard deviation [[35:16](http://www.youtube.com/watch?v=K34gBCjzni8&t=2116)].
* **Rule-Based Rewards:** Unlike models that use humans to score answers, R1 uses deterministic rules (e.g., "does the code compile?" or "is the math answer correct?") to provide feedback [[07:34](http://www.youtube.com/watch?v=K34gBCjzni8&t=454)].

### **KL Divergence & Stability**
**KL (Kullback-Leibler) Divergence** is used as a penalty term in the RL loop to ensure the model doesn't diverge too far from its original base parameters [[52:45](http://www.youtube.com/watch?v=K34gBCjzni8&t=3165)].


The tutorial highlights an unbiased estimator for KL Divergence (referred to as **K3**), which uses a "control variate" to keep the variance low and the training stable [[01:02:26](http://www.youtube.com/watch?v=K34gBCjzni8&t=3746)]. This specific implementation is crucial for maintaining the model's core language abilities while it learns to reason.

### **Key Benchmark Results**
DeepSeek R1 performs on par with OpenAI’s o1 model across various reasoning benchmarks, including mathematics and coding [[22:55](http://www.youtube.com/watch?v=K34gBCjzni8&t=1375)]. It notably showed an **emergent behavior** where the model's response length increased over training steps as it learned that thinking longer led to better rewards [[01:13:50](http://www.youtube.com/watch?v=K34gBCjzni8&t=4430)].

**Watch the full tutorial here:** [DeepSeek R1 Theory Tutorial](https://www.youtube.com/watch?v=K34gBCjzni8)

