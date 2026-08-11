# Syllabus: Prompt Tuning & Prompt Optimization

Given your background with QLoRA/RLHF fine-tuning and PyTorch, I'll pitch this at a practitioner level — moving quickly through prompt engineering basics into the parts with more technical meat (soft prompts, gradient-free optimization, DSPy-style compilers).

## Module 1: Foundations — Prompting as an Interface to Frozen Models
- In-context learning: why it works (induction heads, task vectors framing)
- Zero-shot vs few-shot vs chain-of-thought; when each helps
- Prompt sensitivity: order effects, formatting brittleness, verbalizer choice
- Key papers: GPT-3 (Brown et al., 2020), Chain-of-Thought (Wei et al., 2022), Calibrate Before Use (Zhao et al., 2021)
- **Exercise**: measure variance in accuracy on a classification task across 10 semantically-equivalent prompt paraphrases

## Module 2: Manual Prompt Engineering Techniques
- Few-shot selection and ordering strategies (similarity-based retrieval, diversity sampling)
- Structured prompting: ReAct, self-consistency, tree-of-thought
- Role/system prompt design, output format constraints (JSON mode, grammars)
- Instruction tuning's effect on prompt sensitivity (why instruct-models need less prompt engineering than base models)
- **Exercise**: build a self-consistency wrapper and measure accuracy/cost tradeoff vs single-shot CoT

## Module 3: Prompt Tuning (Soft Prompts) — the PEFT Lineage
- Prefix-Tuning (Li & Liang, 2021): learned continuous prefixes per layer
- Prompt Tuning (Lester et al., 2021): learned embeddings prepended to input only, scales with model size
- P-Tuning v2 (Liu et al., 2022): closing the gap with full fine-tuning across tasks/scales
- Contrast with LoRA/QLoRA (which you already know well) — parameter count, where the update lives (embedding space vs weight deltas), inference-time cost, compositionality (can you combine a soft prompt with a LoRA adapter?)
- Multi-task soft prompts and prompt transfer (SPoT)
- **Exercise**: implement prompt tuning from scratch in PyTorch — freeze a small model, train a virtual token sequence via backprop through the frozen forward pass, compare against a LoRA baseline on the same task

## Module 4: Discrete Prompt Optimization (Gradient-Free)
- Why soft prompts don't work for API-only models — need discrete, human-readable prompts
- AutoPrompt (Shin et al., 2020): gradient-guided discrete token search
- APE — Automatic Prompt Engineer (Zhou et al., 2022): LLM-generates-and-scores-prompts loop
- OPRO (Yang et al., 2023): using an LLM as the optimizer, natural language "trajectory" of past prompt+score pairs
- EvoPrompt: evolutionary algorithms (genetic/differential) over prompt population
- **Exercise**: implement a simple OPRO-style loop — meta-prompt an LLM with (prompt, score) history, have it propose the next candidate, run 10 iterations on a small benchmark

## Module 5: Compiled / Programmatic Prompting
- DSPy: signatures, modules, and teleprompters (BootstrapFewShot, MIPRO) — treating prompts as compiled artifacts rather than hand-written strings
- TextGrad: "backpropagation" through text via LLM-generated textual gradients
- Separating program logic from prompt wording — why this matters for maintainability in production systems
- **Exercise**: port one of your existing agentic components (e.g., an attack-strategy selector in RedTeamAgentLoop) into a DSPy signature + module, and let MIPRO optimize the few-shot examples/instructions against a held-out eval set

## Module 6: Evaluation & Optimization Loops
- Building eval sets: held-out test cases, LLM-as-judge, rubric-based scoring, pairwise preference
- Overfitting to the optimization set — train/val/test discipline for prompts, same as any ML pipeline
- Cost-aware optimization: number of optimizer calls vs quality gain (relevant given OPRO/APE can be expensive)
- Regression testing prompts in CI (useful for anything you productionize)
- **Exercise**: build a small eval harness (pytest-style) that scores prompt variants on accuracy, latency, and token cost simultaneously

## Module 7: Security & Robustness Angle (ties into your red-teaming work)
- Prompt injection and adversarial suffixes as the flip side of prompt optimization (same search techniques — GCG is literally AutoPrompt applied adversarially)
- Robust prompt design against jailbreaks: sandwich defenses, instruction hierarchies
- Automated red-teaming as prompt optimization against a safety objective instead of a task objective — direct overlap with RedTeamAgentLoop's MCTS-based mutation
- Key paper: GCG / Universal Adversarial Suffixes (Zou et al., 2023)
- **Exercise**: map your existing MCTS mutation operators onto the OPRO/EvoPrompt framing — is your attack strategy bank effectively a learned discrete-prompt-optimization population?

## Module 8: Applying This to Financial Services Use Cases
- Prompt tuning for domain adaptation without full fine-tuning (useful where you can't retrain due to compliance/latency constraints)
- Structured output reliability for extraction tasks (ties to FinVision)
- Guardrail prompt optimization (ties to Shield-Fin) — optimizing prompts against both task accuracy and refusal-rate objectives simultaneously
- Capstone: pick one of your portfolio projects (Shield-Fin or AuditAgent) and write up a prompt-optimization strategy section for its system-design.md — which technique from Modules 3–6 fits its constraints (API-only model? need for interpretable prompts? latency budget?) and why

