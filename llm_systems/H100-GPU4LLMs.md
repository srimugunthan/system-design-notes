# GPU advantage for LLMs: architecture, economics, and the road to trillion-parameter inference

NVIDIA's Hopper and Blackwell GPUs have become the de facto substrate of the large language model era because transformer training and serving are dominated by dense matrix multiplication at extreme FLOP counts, memory bandwidth is the scarce resource for inference, and only GPUs combine tensor-core math, HBM bandwidth, and a programmable fabric (NVLink + CUDA) that lets those operations scale from one chip to 72-chip racks. **Frontier LLM training now routinely consumes 10²⁵–10²⁶ FLOPs on 10,000–100,000-GPU clusters**, and CUDA's two-decade head start means every important systems innovation (FlashAttention, PagedAttention, speculative decoding, FP8 Transformer Engine, FP4 on Blackwell) still lands first on NVIDIA silicon. This document walks through the H100's architecture, the mathematical reasons LLMs need GPUs at all, the historical arc from rasterization to ChatGPT, the structure of NVIDIA's moat against AMD/Google/AWS/Cerebras/Groq, the inference-serving techniques that define production systems today, and a practical decision framework for choosing hardware in April 2026.

## 1. Inside the H100: Hopper's Streaming Multiprocessor, tensor cores, and NVLink fabric

The H100 ("GH100") is a **TSMC 4N die with 80 billion transistors on 814 mm²**, launched at GTC 2022 and still the dominant training/inference GPU in production through 2025. The full die contains 8 GPCs × 9 TPCs × 2 SMs = **144 SMs**, but no shipping SKU uses all of them: the **H100 SXM5 enables 132 SMs** (16,896 FP32 CUDA cores, 528 fourth-gen Tensor Cores), the **H100 PCIe uses 114 SMs** (14,592 CUDA cores, 456 Tensor Cores), and the **H100 NVL** exposes the full 6144-bit memory interface to deliver 94 GB HBM3 per die.

Each Hopper SM contains **128 FP32 cores, 64 FP64 cores, 64 INT32 cores, and 4 fourth-gen Tensor Cores**, with four warp schedulers issuing 32-thread warps. Per-SM resources include a **256 KB register file, a 256 KB unified L1/shared memory (up to 228 KB configurable as shared)**, and the new **Tensor Memory Accelerator (TMA)** for asynchronous bulk tensor copies between HBM and shared memory. New to Hopper are **thread-block clusters** (up to 16 blocks co-scheduled on a GPC with distributed shared memory), **async WGMMA** warpgroup matrix instructions, and **DPX instructions** for dynamic-programming kernels. The L2 cache is **50 MB**, shared across a partitioned crossbar.

### Fourth-gen tensor cores and the Transformer Engine

The fourth-gen tensor cores support **FP64, TF32, BF16, FP16, FP8 (E4M3 and E5M2), and INT8** inputs with FP32/FP16/INT32 accumulation, plus structured 2:4 sparsity for 2× effective throughput. FP8 is the defining innovation: **E4M3** (1 sign, 4 exponent, 3 mantissa) is used for forward-pass activations and weights, while **E5M2** (5 exponent, 2 mantissa) handles gradients where wider dynamic range matters. The **Transformer Engine** library dynamically manages FP8 scaling per tensor via **delayed scaling with an amax history** (default 1024-step rolling window), rather than a single global loss scale as FP16 AMP used. In HYBRID mode, TE runs forward in E4M3 and backward in E5M2, and can push attention end-to-end in FP8 via `fp8_dpa=True` to avoid cast overhead.

Peak math throughput on H100 SXM5 (dense, marketed datasheet numbers): **67 TFLOPS FP64 tensor, 989 TFLOPS TF32, 1,979 TFLOPS BF16/FP16, 3,958 TFLOPS FP8/INT8**, all doubling again under 2:4 sparsity. The PCIe variant delivers roughly 80% of these figures.

### HBM3, NVLink 4, NVSwitch, and scale-up topology

Memory is where the SXM/PCIe split matters most. **H100 SXM5 pairs 80 GB HBM3 across five stacks at 3.35 TB/s**; H100 PCIe uses HBM2e at **2.0 TB/s**; H100 NVL opens the full 6144-bit bus to deliver ~**3.9 TB/s per die and 188 GB per two-GPU card**; and the H200 upgrade retains the same GH100 compute die but swaps in **141 GB HBM3e at 4.8 TB/s**, delivering up to **1.5× H100 inference throughput** on Llama-2-70B in MLPerf v4.x purely from bandwidth.

**NVLink 4.0** provides **18 links × 50 GB/s bidirectional = 900 GB/s aggregate per GPU**, seven times PCIe Gen 5 x16's 128 GB/s. Each third-gen **NVSwitch** chip (TSMC 4N, 64 NVLink4 ports) delivers 13.6 Tb/s of switching; an HGX H100 8-GPU baseboard integrates **4 NVSwitches** to provide **3.6 TB/s bisection within the node**. The DGX H100 SuperPOD extends this via external NVLink Switches to a **256-GPU NVLink domain with 57.6 TB/s all-to-all bisection** and roughly **1 exaFLOP of FP8 sparse compute**—a coherent shared-memory fabric at a scale competitors still cannot match in merchant silicon.

### MIG, confidential computing, and the SXM vs PCIe tradeoff

Hopper's **second-gen Multi-Instance GPU (MIG)** partitions an H100 into up to **seven isolated instances** by slicing SMs, L2, memory controllers, and NVDEC/NVJPG engines. Each instance has roughly **3× the compute and 2× the memory bandwidth** of an A100 MIG slice, and Hopper adds **hardware confidential computing**: a GPU-resident TEE with encrypted PCIe transfers and attestation, enabling multi-tenant secure AI.

| Attribute | H100 SXM5 | H100 PCIe | H100 NVL (per die) |
|---|---|---|---|
| SMs / CUDA cores | 132 / 16,896 | 114 / 14,592 | 132 / 16,896 |
| Memory | 80 GB HBM3 | 80 GB HBM2e | 94 GB HBM3 |
| Bandwidth | 3.35 TB/s | 2.0 TB/s | ~3.9 TB/s |
| NVLink | 900 GB/s + NVSwitch | 600 GB/s bridge | 600 GB/s bridge |
| FP8 dense TFLOPS | 3,958 | 3,026 | ~3,958 |
| TDP | 700 W | 350 W | ~400 W per die |
| Primary use | Large-scale training / DGX | Enterprise servers | LLM inference |

## 2. Why training runs on GPUs: the math of dense matmul at scale

A decoder-only transformer's training compute is almost entirely dense GEMM. The Kaplan/Chinchilla approximation **C ≈ 6ND** (N = non-embedding params, D = training tokens) captures this cleanly, and a per-layer FLOP breakdown shows **QKV and output projections plus the two FFN linears account for 72·L·D² FLOPs per token, while attention itself contributes only 12·L·T·D**. The ratio of attention FLOPs to matmul FLOPs is roughly **T/(8·D_model)**, so for typical hidden sizes (4k–8k) and sequence lengths (2k–32k), **more than 90%—often 99%—of training FLOPs are large dense matrix multiplies**, exactly the workload CUTLASS/cuBLAS and tensor cores optimize.

### Parallelism: DP × TP × PP × SP × EP

Production frontier runs compose five parallelism axes. **Data parallelism** (DDP, ZeRO-1/2/3, FSDP) shards optimizer state, gradients, and parameters across replicas. **Tensor parallelism** (Megatron-LM, Shoeybi et al. 2019) splits individual matrices—column-parallel QKV followed by row-parallel output projection requires exactly one all-reduce per attention block and one per MLP block, confined ideally within a single NVLink node. **Pipeline parallelism** (GPipe, PipeDream 1F1B, interleaved 1F1B from Narayanan et al. SC'21) distributes layers across devices with micro-batch pipelining; the bubble shrinks as micro-batches grow. **Sequence/context parallelism** (Megatron SP, Ring Attention, DeepSpeed-Ulysses) splits the sequence dimension to cap activation memory on long contexts. **Expert parallelism** routes MoE tokens across devices. Narayanan et al. trained a **1T-parameter GPT on 3,072 A100s at 52% of peak (502 PFLOP/s aggregate)** using this PTD-P recipe.

### SIMT, the roofline, and arithmetic intensity

GPUs execute in **SIMT**: 32-thread warps share a program counter but carry independent registers. Each Hopper SM holds up to 64 resident warps and swaps between them on memory stalls at zero cost, a latency-hiding model that needs **massive oversubscription** to work—H100 keeps roughly **270,000 concurrent threads in flight**. This fits matmul perfectly: predictable strided loads, huge arithmetic intensity, and no branch divergence.

The **roofline ridge point** for H100 BF16 is **989 TFLOPS ÷ 3.35 TB/s ≈ 295 FLOPs/byte**. Square GEMMs at realistic batch sizes sit far above this and hit 80–90% of peak; **long-context attention has arithmetic intensity ≈ d ≈ 128 FLOPs/byte**, which is memory-bound—this is precisely why FlashAttention, which fuses attention into on-chip SRAM and avoids materializing the N×N score matrix, is essential. **Token-by-token decoding has AI ≈ 1 FLOP/byte**, making single-stream inference purely HBM-bandwidth-limited, the structural reason the field obsesses over KV cache compression and speculative decoding.

### Mixed precision, gradient accumulation, and activation recomputation

Modern pretraining uses **BF16** (8-bit exponent identical to FP32, so no loss scaling) with **FP32 master weights** and **FP8 via Transformer Engine** for a further 2× matmul throughput, with per-tensor scaling keeping accuracy within ~0.5% of BF16 perplexity. Blackwell extends this with **MXFP8/MXFP6/MXFP4** microscaling formats. Gradient accumulation lets practitioners hit the Chinchilla-optimal global batch (millions of tokens) without holding all activations in memory; **activation checkpointing** recomputes intermediate activations during backward at roughly +33% forward-FLOP cost, pushing effective training compute from 6ND toward ~8ND.

Optimizer-state memory explains why HBM capacity matters as much as bandwidth: **a 70B model in BF16 carries 140 GB of weights, 140 GB of gradients, 280 GB of FP32 master weights, and 560 GB of Adam moments—roughly 1.1 TB per replica before activations**. FSDP/ZeRO-3 shards this across data-parallel ranks, and the per-step cost of gathering and scattering optimizer state via NVLink/InfiniBand is why interconnect bandwidth is as much a training constraint as FLOPs. Real-world ground truth: **Llama 3.1 405B consumed 3.8 × 10²⁵ FLOPs on 16,384 H100s for ~54 days, totaling 30.8M GPU-hours** at roughly 42.7% MFU in FP8.

## 3. GPUs vs CPUs: why two orders of magnitude matter

A dual-socket server CPU in 2026 (top-bin Xeon Platinum 8592+ Emerald Rapids or AMD EPYC 9654/9754) offers roughly **60–128 cores with AVX-512 delivering ~1,000–2,000 aggregate FP32 SIMD lanes** at ~4–8 TFLOPS FP32. An **H100 SXM5 exposes 16,896 FP32 CUDA cores and 528 tensor cores**, delivering **1,979 BF16 TFLOPS dense and 3,958 FP8 TFLOPS dense**—roughly **500× higher matmul throughput** than even an Intel AMX-equipped Xeon, which tops out near 130 BF16 TOPS.

Memory bandwidth tells the same story: dual-socket EPYC 9654 with 12-channel DDR5-4800 provides **~920 GB/s aggregate**, while **H100 SXM5 HBM3 delivers 3.35 TB/s**—a 3.6× ratio that widens to **5× on H200 (4.8 TB/s)** and **~9× on B200 (8 TB/s)**. Cache hierarchies differ in character: a Xeon 8592+ has 320 MB of L3 and 2 MB of L2 per core optimized for single-thread latency, while H100 has only 50 MB of L2 but attaches it to 3.35 TB/s HBM and a 34 MB aggregate register file across 132 SMs.

Concrete LLM benchmarks make the gap unambiguous. **Llama-2-70B Q5 inference on a 96-core EPYC 9655P tops out around 44 tok/s; BF16 single-stream on CPU is 1–3 tok/s**. Four H100s with TensorRT-LLM sustain **60–100 tok/s per single-stream user and 1,000–2,000+ tok/s aggregate**, and 8×H200 MLPerf v4.1 submissions reach **~27,000 tok/s offline on Llama-2-70B**. Training GPT-3 class models on CPU clusters is economically impossible: GPT-3's 3.14 × 10²³ FLOPs ran on ~1,024 A100s for roughly a month; the same compute on even a 1,024-socket Xeon cluster would take **years** after accounting for the absence of tensor cores, lower memory bandwidth, and slower inter-node collectives. **CPUs bottleneck LLMs on four vectors at once**: matmul throughput, HBM-class bandwidth for weight streaming, the lack of a mainstream FP8 pipeline, and collective-communication latency.

## 4. From the Voodoo to the GB200: a 30-year arc

The modern GPU emerged from the 1996 **3dfx Voodoo Graphics** (rasterization only) and the 1999 **NVIDIA GeForce 256**, marketed as "the world's first GPU" for introducing hardware transform and lighting. Programmable shading arrived with **DirectX 8 (2000)** and matured with **DirectX 9 (2002)** and HLSL, enabling the GeForce FX and ATI's dominant Radeon 9700 Pro. Researchers began abusing pixel shaders for general compute around 2002, formalized as GPGPU on Mark Harris's site and by the Stanford **BrookGPU** project (Ian Buck et al., SIGGRAPH 2004)—Buck later joined NVIDIA and led CUDA.

**CUDA launched in November 2006** alongside the G80 (GeForce 8800 GTX), the first unified-shader GPU with 128 scalar stream processors. Jensen Huang's structural decision to ship CUDA on every GeForce seeded a massive developer base years before AI validated the bet. Early ML experiments followed quickly: **Raina, Madhavan, and Ng's ICML 2009 paper** trained deep belief networks on a GTX 280 at ~70× CPU speed, and **Cireșan's IDSIA group** won multiple 2011 image competitions with CNNs on GPUs. The decisive moment came on **September 30, 2012**, when Krizhevsky, Sutskever, and Hinton's **AlexNet** trained on two GTX 580s over 5–6 days won ILSVRC 2012 with **15.3% top-5 error vs 26.2% for the runner-up**—a discontinuity larger than the prior five years combined.

### The tensor core era and the transformer

The deep-learning boom (2013–2016) standardized on NVIDIA: **cuDNN** shipped in September 2014; **TensorFlow** open-sourced in November 2015; **PyTorch** in January 2017. **Pascal P100 (April 2016)** introduced HBM2 and NVLink 1.0, and Jensen personally delivered the first DGX-1 to OpenAI in August 2016. **Google's TPU v1**, publicly announced May 18, 2016, proved dedicated neural silicon could be 15–30× more efficient than contemporary GPUs on inference; TPU v2 (2017) added training with bfloat16, and TPU v3 (2018) scaled to 128 GB HBM per board.

NVIDIA's response was the **Volta V100 (May 2017)** with first-generation Tensor Cores (FP16 with FP32 accumulate, 125 TFLOPS), followed by **Ampere A100 (May 2020)** which added TF32, BF16, MIG, and 80 GB HBM2e at 2 TB/s. The transformer arrived between them: **Vaswani et al., "Attention Is All You Need" (arXiv June 12, 2017)**; **BERT (Oct 2018)**; **GPT-1 (June 2018)**; **GPT-2 (Feb 2019)**; **GPT-3 (May 2020) at 175B parameters and 3.14 × 10²³ FLOPs**. The **Kaplan scaling laws (January 2020)** and the corrective **Chinchilla paper (March 2022)**—which established the ~20 tokens-per-parameter compute-optimal rule—turned "more GPUs equals better model" into a quantitative investment thesis.

### The ChatGPT moment and the $5T company

**Hopper H100 arrived at GTC 2022** with FP8, TMA, and DPX instructions. Then **ChatGPT launched November 30, 2022**, and H100 lead times stretched to 36–52 weeks within six months. NVIDIA's data center revenue, which had quietly overtaken gaming in FY23 Q2, exploded to **$14.5B in FY24 Q3 (+279% YoY)** and **$115B for FY25**. Market cap crossed $1T on May 30, 2023; **$3T on June 5, 2024**; briefly peaked above $3.5T in late 2024 before DeepSeek-V3's January 2025 debut triggered the largest one-day loss in US history. By April 2026 NVIDIA trades near **$4.9T**, still the world's most valuable public company, with **Blackwell (GB200/B200) shipping through 2025** and **Rubin (288 GB HBM4, late 2026)** already committed by OpenAI, Microsoft, Meta, xAI, and Anthropic.

## 5. Why CUDA is hard to displace

NVIDIA holds **roughly 85–92% of the AI accelerator market** in 2025–2026 across measurement methodologies, and the question is not whether competitors have competitive silicon but why CUDA remains sticky despite nineteen years of attempts to route around it.

### The AMD challenge and why ROCm lags

AMD's **Instinct MI300X (Dec 2023, 192 GB HBM3, 5.3 TB/s, 1,307 BF16 TFLOPS)**, **MI325X (256 GB HBM3E, Q4 2024)**, and **MI350X/355X (288 GB HBM3E, 8 TB/s, up to 20.1 PFLOPS FP4, June 2025)** match or exceed NVIDIA on raw memory specs. The **MI400 "Helios"** previewed at CES 2026 ships in H2 2026. But **ROCm, launched 2016, is still closing software gaps a decade later**: PyTorch native support only stabilized mid-2024, Triton and FlashAttention ports trail CUDA by months, and production deployments at scale remain rare. Infinity Fabric delivers roughly 153 GB/s per GPU—an order of magnitude below NVLink—and AMD's scale-up story rests on the UALink consortium and UEC Ethernet rather than a merchant equivalent to NVL72. **MI300X sells at a 20–30% discount to H100, but TCO equalizes after accounting for software labor and lower utilization.**

### Hyperscaler silicon: Google, AWS, Meta

**Google TPU** is the most credible non-NVIDIA training stack. The v5p delivers 459 BF16 TFLOPS with 95 GB HBM across 8,960-chip pods, and **Ironwood (v7, GA November 2025)** introduces **native FP8 at 4,614 TFLOPS, 192 GB HBM3E, and 7.37 TB/s**—per-chip compute that slightly exceeds a single GB200. A 9,216-chip Ironwood pod runs on 10 MW with optical circuit switches, and Google's Jupiter fabric can federate ~43 pods. Anthropic committed to **1M Ironwood chips starting 2026**. TPU is GCP-only and requires XLA, which caps its impact on the broader enterprise market.

**AWS Trainium** follows a similar path. **Trainium2 (Dec 2024)** offers 96 GB per chip and 20.8 PFLOPS FP8, and **Trainium3 (re:Invent 2025)** is AWS's first 3nm silicon with 2.52 PFLOPS per chip and 144 GB HBM3E; the Trn3 UltraServer delivers **362 MXFP8 PFLOPS across 144 chips at 706 TB/s aggregate HBM bandwidth**. **Project Rainier**, activated October 2025, hosts ~500,000 Trainium2 chips for Anthropic—the largest non-NVIDIA AI cluster in operation. AWS claims 54% lower cost per token for GPT-class workloads.

### Intel, Graphcore, Cerebras, Groq, Tenstorrent

**Intel Gaudi3 (April 2024, 128 GB HBM2E, 1,835 FP8 TFLOPS)** failed to achieve meaningful adoption; Intel **cancelled Falcon Shores as a commercial product in January 2025**. **Graphcore was acquired by SoftBank in July 2024 for ~$500M**, below total funding. **Cerebras WSE-3** (46,225 mm², 44 GB on-chip SRAM, 21 PB/s aggregate bandwidth) holds inference speed records—**2,100 tok/s on Llama 3.2-70B, 969 tok/s on Llama 405B at 128K context**—but remains niche with a small deployed base; IPO targeted Q2 2026. **Groq LPU** achieves 300–800 tok/s on Llama 70B with deterministic latency via compiler-scheduled static execution over 230 MB of on-chip SRAM per chip, but Llama-70B requires ~300+ chips. In a notable signal, **NVIDIA invested ~$20B in Groq in November 2025**, a hedge against HBM supply constraints. **Tenstorrent** (Jim Keller) ships open-source RISC-V accelerators but at commercially marginal volumes.

### The five-layer moat

NVIDIA's defense runs deeper than CUDA libraries alone. First, **CUDA itself** carries 4.5M+ registered developers, 3,000+ accelerated applications, and the reference backend status in PyTorch, TF, and JAX. Every hardware feature (FP8 Transformer Engine on Hopper, FP4/NVFP4 on Blackwell, adaptive compression on Rubin) ships day-one with matching cuDNN/cuBLAS/CUTLASS support; competitors trail 12–24 months. Second, **NVLink/NVSwitch/NVL72**: the GB200 NVL72 rack packages 72 Blackwell GPUs and 36 Grace CPUs behind a coherent 130 TB/s NVLink fabric with 13.5 TB of HBM and **1.44 EFLOPS FP4**, and the **GB300 NVL72 (Blackwell Ultra, shipping H2 2025)** pushes this to 288 GB HBM3E per GPU and ~1.1 EFLOPS per rack. The upcoming **Vera Rubin NVL144 and NVL576 systems (2H 2026 / 2027)** scale to 4.6 PB/s HBM4E bandwidth and 15 EFLOPS FP4 per rack. Third, **Mellanox (acquired 2020 for $6.9B)** gives NVIDIA InfiniBand Quantum-X and Spectrum-X Ethernet—xAI Colossus runs Spectrum-X at 95% throughput versus 60% on standard Ethernet. Fourth, **OEM and supply-chain control**: DGX/HGX reference designs license to Dell, HPE, Supermicro, Lenovo, and Cisco, and NVIDIA consumes the majority of TSMC's advanced CoWoS packaging through 2027. Fifth, **flagship customer lock-in**: xAI Colossus (150K H100 + 50K H200 + 30K GB200 on a 250 MW site), Microsoft Fairwater superfactories, and Meta/OpenAI/Anthropic deployments establish NVIDIA as the default.

The structural answer for why the moat holds: **training workloads require kernel-level tuning tightly coupled to the programming model, switching costs are organizational rather than just technical, and every new research technique lands on CUDA first**. Erosion vectors exist—OpenXLA/MLIR/Triton reduce porting cost, inference migrates to custom silicon, and Chinese export controls force domestic ecosystems like Huawei's CANN stack around the Ascend 910C. Consensus analyst view is that **NVIDIA share drifts from ~85% toward 70–80% over 2026–2028** as custom silicon scales for inference, while absolute revenue keeps growing because total AI capex outpaces share loss.

## 6. The inference playbook: how modern serving systems extract 10× from the same silicon

LLM inference is dominated by memory bandwidth during decode (each token requires streaming the full weight set through HBM) and by compute during prefill (encoding the prompt). The production stack—vLLM, TensorRT-LLM, SGLang—composes roughly a dozen innovations from the last three years to push real-world serving efficiency an order of magnitude above a naive PyTorch implementation.

### FlashAttention: IO-aware exact attention

**FlashAttention (Dao et al., NeurIPS 2022)** reformulates attention as a single fused kernel that tiles Q/K/V into on-chip SRAM, uses an online-softmax trick to avoid materializing the N×N score matrix, and recomputes attention in the backward pass. It reduces HBM traffic from O(N²) to O(N²d²/M) and delivers 2–4× end-to-end speedup. **FlashAttention-2 (Dao, July 2023)** rewrites the kernel in CUTLASS 3.x with parallelism over the sequence dimension, reaching **230 TFLOPS on A100 (73% of peak)** and enabling 72% MFU end-to-end GPT training. **FlashAttention-3 (Shah et al., July 2024)** exploits Hopper-specific WGMMA and TMA with warp-specialized producer/consumer pipelines, hitting **740 TFLOPS BF16 (75% of H100 peak) and ~1.2 PFLOPS FP8**—plus 2.6× lower FP8 numerical error than per-tensor baselines. FA-4 targeting Blackwell is in development.

### PagedAttention and continuous batching

**PagedAttention (Kwon et al., SOSP 2023)**, the kernel behind **vLLM**, partitions the KV cache into fixed-size blocks (typically 16 tokens) stored non-contiguously with per-request block tables—an OS-virtual-memory analogy applied to GPU memory. Naive pre-allocation wastes 60–80% of KV memory; PagedAttention drops this below 4% and enables copy-on-write block sharing for parallel sampling. Combined with **continuous batching** (Orca, OSDI 2022), which schedules at iteration granularity so completed requests leave the batch and new ones join mid-generation, production systems achieve **2–4× throughput over Orca/FasterTransformer and up to 24× over HuggingFace Transformers**.

### Speculative decoding and self-speculation

**Speculative decoding (Leviathan et al., ICML 2023; Chen et al. DeepMind 2023)** uses a small draft model to propose k tokens which the target verifies in parallel; rejection sampling preserves the target distribution exactly, delivering 2–3× typical speedup. Self-speculation variants avoid the draft model entirely: **Medusa (Cai et al., 2024)** adds extra LM heads predicting future tokens with tree attention; **EAGLE-3 (NeurIPS 2025)** autoregresses on second-to-top-layer features and achieves **up to 6.5× speedup with ~4.5 accepted tokens per step**. **DeepSeek-V3's Multi-Token Prediction module** achieves 85–90% acceptance when reused for speculative decoding at inference. All production stacks (vLLM, TRT-LLM, SGLang) support at least n-gram, EAGLE, and Medusa speculation.

### KV cache: the memory-bandwidth battleground

KV cache per token for **Llama-2-70B MHA in FP16 is ~2.5 MB**, making 128K context infeasible without architectural help. **Grouped-Query Attention (Ainslie et al., EMNLP 2023)** groups Q heads to share KV heads and is now standard across Llama-2-70B, Llama-3, Mistral, and Qwen, dropping KV to ~320 KB/token at similar quality. **Multi-head Latent Attention (DeepSeek-V2/V3)** goes further: a low-rank joint K/V compression to a latent vector with decoupled RoPE achieves **93.3% KV reduction and 5.76× throughput vs a 67B MHA baseline**. KV cache quantization to FP8, INT8, or INT4 is production-standard in vLLM and TRT-LLM; **KVQuant (NeurIPS 2024)** pushes to 3-bit with <0.1 PPL degradation.

### Quantization across weights and activations

**GPTQ (Frantar et al., ICLR 2023)** applies inverse-Hessian column-wise error compensation for W4A16 post-training quantization in a few GPU-hours on 175B models. **AWQ (Lin et al., MLSys 2024 Best Paper)** scales the ~1% salient weight channels identified by activation magnitude, preserving accuracy better than GPTQ at small sizes. **SmoothQuant (Xiao et al., ICML 2023)** migrates activation outliers into weights for W8A8 INT8. **FP8 via Transformer Engine** is native on Hopper (H100, L40S, RTX 4090) at 2× FP16 throughput; **FP4/NVFP4 on Blackwell** adds another 2×, and Rubin is expected to add hardware-accelerated adaptive compression.

### Disaggregated prefill/decode, RadixAttention, and chunked prefill

**Splitwise (Patel et al., ISCA 2024)** and **DistServe (Zhong et al., OSDI 2024)** separate the compute-bound prefill phase from the memory-bandwidth-bound decode phase onto different GPU pools, transferring KV over InfiniBand. DistServe reports **up to 7.4× more requests or 12.6× tighter SLO** at >90% attainment; Moonshot's **Mooncake (FAST 2025)** productionizes the pattern for Kimi. **RadixAttention (SGLang, Zheng et al., NeurIPS 2024)** maintains a radix tree of KV blocks keyed by token sequences with LRU eviction and cache-aware scheduling, delivering **up to 5× throughput on shared-prefix workloads** and inspiring vLLM's automatic prefix cache. **Chunked prefill (SARATHI, Sarathi-Serve OSDI 2024; DeepSpeed-FastGen's Dynamic SplitFuse)** interleaves prefill chunks with decode tokens in the same forward pass, eliminating multi-second generation stalls and raising arithmetic intensity. **NVIDIA Dynamo (March 2025)** productionizes disaggregated inference across the Hopper/Blackwell fleet.

### Research frontier: sparsity, SSMs, and MoE inference

**StreamingLLM (Xiao et al., ICLR 2024)** retains initial "attention sink" tokens plus a sliding window to generate stably over **4M-token sequences with 22.2× speedup**. **Quest (Tang et al., ICML 2024)** uses query-aware page-level criticality to load only Top-K KV pages, achieving **7.03× self-attention and 2.23× end-to-end speedup**. H2O, SnapKV, PyramidInfer, and DuoAttention pursue related eviction strategies. On architectures, **Mamba (Gu & Dao, Dec 2023)** and **Mamba-2 (ICML 2024)** deliver selective state-space models with linear-time inference and 5× throughput at matched quality; **Jamba (AI21, 2024)** ships a hybrid Mamba+Transformer+MoE production model with 256K context on a single 80 GB GPU. RWKV v7 and RetNet pursue linear-attention variants. For MoE, **expert parallelism** with all-to-all dispatch/combine is the dominant production pattern; **SGLang's large-scale EP deployment on DeepSeek-V3** reports 52.3k input and 22.3k output tokens/s per 8×H100 node at **$0.20 per 1M output tokens**, matching DeepSeek's official API cost on open-source infrastructure.

## 7. Choosing a GPU: a decision framework for April 2026

The inference engineer's choice has broadened considerably as Hopper commoditized and Blackwell arrived. Pricing below is a snapshot and shifts weekly.

### The full lineup at a glance

| GPU | Memory | BW (TB/s) | BF16 TFLOPS | FP8 TFLOPS | NVLink | Cloud $/GPU-hr (typical) |
|---|---|---|---|---|---|---|
| B200 SXM | 192 GB HBM3e | 8.0 | ~2,250 | ~4,500 | 1.8 TB/s | $4.99–10 (spot $2.10) |
| GB200 (per GPU) | 192 GB HBM3e | 8.0 | ~2,500 | ~5,000 | 1.8 TB/s | $10.50–12 |
| B300 / Blackwell Ultra | 288 GB HBM3e | 8.0+ | ~3,375 | ~7,000 | 1.8 TB/s | $4.95–18 |
| H200 SXM | 141 GB HBM3e | 4.8 | 1,979 | 3,958 | 900 GB/s | $3.00–6.50 |
| H100 SXM5 | 80 GB HBM3 | 3.35 | 1,979 | 3,958 | 900 GB/s | $2.00–4.00 |
| H100 NVL | 94 GB HBM3 | 3.9 | 1,979 | 3,958 | bridge | $1.80–3.00 |
| H100 PCIe | 80 GB HBM3 | 2.0 | 1,513 | 3,026 | bridge | $1.80–2.49 |
| A100 80GB SXM | 80 GB HBM2e | 2.04 | 312 | — | 600 GB/s | $1.29–4.10 |
| L40S | 48 GB GDDR6 | 0.864 | 362 | 733 | none | $0.72–1.86 |
| L4 | 24 GB GDDR6 | 0.30 | 121 | 242 | none | $0.40–0.80 |
| A10G | 24 GB GDDR6 | 0.60 | 125 | — | none | $0.60–1.01 |
| T4 | 16 GB GDDR6 | 0.32 | 65 | — | none | $0.20–0.53 |
| RTX 5090 | 32 GB GDDR7 | 1.79 | 209 (dense) | 419 | none | $0.80–1.20 (consumer cloud) |
| RTX 4090 | 24 GB GDDR6X | 1.01 | 165 (dense) | 330 | none | $0.39–0.79 |
| RTX 3090 | 24 GB GDDR6X | 0.94 | 142 | — | 112 GB/s | $0.20–0.35 |

### Memory sizing rule of thumb

**Minimum VRAM ≈ (params × bytes/param) + KV cache + activations + ~15% overhead**, where bytes/param is 2 for FP16/BF16, 1 for FP8, and ~0.5 for INT4. Practical minimum single-user capacities (4K context): **7B fits 24 GB at FP16; 13B needs 40 GB; 70B FP16 needs 140 GB** (two 80 GB H100s or one H200); **70B FP8 fits one 80 GB**; **70B INT4 fits one 48 GB L40S**. At the frontier, **405B FP16 needs 810 GB**—eight H100s or six H200s minimum—while **405B FP8 needs ~410 GB** (four H200s or eight H100s). Production deployments must add 30–100% KV headroom for concurrency and long context.

### Cost per token in practice

For **Llama 3.1-70B FP8 inference**, self-hosted economics in April 2026 look roughly like: two H100 SXM at Lambda ($5.98/hr, ~2,700 agg tok/s) = **$0.62 per 1M output tokens**; a single H200 at Jarvis ($3.80/hr, ~1,100 tok/s) = **$0.96/M**; B200 on Spheron spot ($2.12/hr, ~1,800 tok/s) = **$0.33/M**, the current cost leader. Managed APIs benchmark at Groq $0.64/M, Together Turbo $0.88/M, Fireworks $0.90/M, and DeepInfra FP8 ~$0.35/M. For **Llama 3.1-405B**, eight H100s at AWS p5 cost ~$10.92 per 1M tokens, four H200s at Crusoe ~$5.08, and a GB200 NVL72 slice ~$5.83—but **Amazon Bedrock's managed 405B endpoint at $2.40/M blended beats all self-host options below ~70% utilization**. The consistent lesson: **managed APIs dominate below ~50% utilization; self-hosted B200/H100 on spot wins above ~70%**.

### When to use which parallelism

Pick no parallelism (TP=1) whenever the model fits on a single GPU—continuous batching in vLLM/SGLang recovers throughput without the all-reduce tax. **Use tensor parallelism (TP=2/4/8) only within an NVLink domain**; across PCIe the all-reduce cost dominates. Cross-node workloads should combine TP within each node with pipeline parallelism across nodes, accepting the bubble penalty for throughput. Ultra-long contexts (128K+) benefit from sequence/context parallelism. **MoE models (Mixtral, DeepSeek-V3, Llama 4 Maverick) want expert parallelism**, which is exactly what GB200 NVL72's 72-GPU NVLink domain was built for. The hard rule: **never tensor-parallelize across GPUs without NVLink-class interconnect for latency-sensitive inference**.

### The decision tree

For ≤7B models, an L4, A10G, or RTX 4090 (for dev work only) is sufficient. For 8–13B, L40S offers the best price/performance. For 34B, one A100 80 GB, H100, or RTX 5090 with INT4 suffices. For 70B: **latency-critical picks one H200 FP8**; **throughput batch picks two L40S FP8 or one B200 FP8**; **budget-constrained traffic under ~5M tokens/day goes to a managed API**. For 200B-class models, two to four H100s or two H200s in FP8. For 405B and above, self-host with eight H200s or use Amazon Bedrock at $2.40/M. For trillion-parameter MoE frontier, GB200 NVL72 or B300 rack-scale is the only reasonable choice.

### The consumer-GPU trap

NVIDIA's GeForce driver EULA (since 2018, still enforced in 2026) prohibits datacenter deployment of RTX 3090/4090/5090 except for blockchain. Hyperscalers and tier-1 neoclouds correctly refuse to offer them; marketplaces like Vast.ai and RunPod Community host them on prosumer workstations where production use risks EULA breach, lacks ECC, and draws inconsistent driver support. The legitimate alternatives using identical silicon are **L40S (AD102, 48 GB)**, **RTX 6000 Ada**, and the forthcoming **RTX 6000 Blackwell with 96 GB ECC**.

## Closing takeaways

The H100 and its descendants dominate LLM workloads for reasons that are simultaneously architectural, economic, and historical. Architecturally, **three numbers tell the story**—1,979 BF16 TFLOPS, 3.35 TB/s HBM bandwidth, and 900 GB/s NVLink—each an order of magnitude above the best non-tensor-core alternative. Mathematically, transformers are ≥90% GEMM plus a memory-bound attention kernel that FlashAttention fixes, and the arithmetic intensity of decoding (≈1 FLOP/byte) makes HBM bandwidth the ultimate constraint on single-stream latency. Economically, CUDA's nineteen-year compound-interest advantage survives every challenger because training demands kernel-level tuning, switching costs are organizational, and NVIDIA ships day-one software for every new hardware feature.

The next eighteen months will test the moat harder than any prior stretch. **Rubin arrives late 2026 with 288 GB HBM4 at ~13 TB/s**, pushing H200 into steep depreciation. **AWS Trainium3 and Google Ironwood are credible at inference scale**, and managed-API economics already beat self-hosting for most deployments under 50% utilization. **MLA-style KV compression, speculative decoding, FP4, and disaggregated serving** together have reduced production inference cost roughly 10× in two years—a rate that, if sustained, compounds faster than raw silicon improvement. For engineers building today, the correct defaults are: **H100 or H200 for self-hosted 70B-class inference, B200 spot for batch throughput, L40S for budget-constrained mid-range, and managed APIs until utilization passes 50%**. For the frontier, the answer is still the same NVIDIA rack NVL72—and increasingly, whoever can afford to buy one.