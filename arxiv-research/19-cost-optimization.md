# LLM Cost Optimization & Token Budgeting

Research on minimizing LLM inference costs while maintaining output quality.

---

## [Token-Budget-Aware LLM Reasoning](https://arxiv.org/abs/2412.18547)

**Authors:** Tingxu Han, Zhenting Wang, Chunrong Fang, Shiyu Zhao, Shiqing Ma, Zhenyu Chen
**Year:** 2024

**Core method:** Proposes a framework (TALE) that dynamically estimates the reasoning complexity of each problem and assigns a token budget accordingly, embedding that budget directly in the prompt. The model is instructed to complete its Chain-of-Thought reasoning within the allocated token count, compressing verbose reasoning steps for simpler problems while preserving full reasoning depth for harder ones.

**Key results:** Reduces CoT reasoning token costs significantly with only a slight accuracy reduction across math and commonsense benchmarks. Demonstrates that blindly applying a fixed token budget hurts performance, but complexity-adaptive budgets maintain quality while cutting costs.

**Relevance to Podders:** Directly applicable via prompt engineering -- Podders could classify incoming tasks (e.g., simple scrape-summarize vs. complex cross-source synthesis) and set per-call max_tokens or prompt-level budget hints for Haiku/Sonnet calls, reducing output tokens on routine tasks without degrading quality on hard ones.

---

## [SelfBudgeter: Adaptive Token Allocation for Efficient LLM Reasoning](https://arxiv.org/abs/2505.11274)

**Authors:** Zheng Li, Qingxiu Dong, Jingyuan Ma, Di Zhang, Kai Jia, Zhifang Sui
**Year:** 2025

**Core method:** Trains the model itself to self-estimate the required reasoning budget before generating the answer. Uses a budget-guided Group Relative Policy Optimization (GPRO) reinforcement learning step that rewards both accuracy and brevity, teaching the model to internally calibrate how many tokens it needs per query.

**Key results:** Achieves 61% average response length compression on math reasoning tasks while maintaining accuracy. Gives users the ability to set explicit token budgets upfront and lets the model predict generation length before starting, enabling early-stop decisions.

**Relevance to Podders:** The self-estimation concept maps to Podders' routing logic -- a lightweight pre-classifier (or even Haiku itself) could predict whether a task needs 200 or 2000 output tokens, then set max_tokens accordingly. The 61% compression figure suggests substantial savings on the ~50 daily Haiku calls.

---

## [BudgetThinker: Empowering Budget-aware LLM Reasoning with Control Tokens](https://arxiv.org/abs/2508.17196)

**Authors:** Hao Wen, Xinrui Wu, Yi Sun, Feifei Zhang, Liye Chen, Jie Wang, Yunxin Liu, Yunhao Liu, Ya-Qin Zhang, Yuanchun Li
**Year:** 2025

**Core method:** Inserts special control tokens periodically during inference to continuously inform the model of its remaining token budget. Combines a two-stage training pipeline: first Supervised Fine-Tuning (SFT) to teach budget awareness, then curriculum-based Reinforcement Learning with a length-aware reward function that optimizes for both accuracy and adherence to the specified budget.

**Key results:** Significantly outperforms baselines in maintaining accuracy across varying budget levels on mathematical benchmarks. Provides precise, fine-grained control over reasoning length -- the model learns to self-pace and wrap up reasoning as the budget depletes, rather than abruptly truncating.

**Relevance to Podders:** Most relevant as a design pattern rather than direct implementation, since Podders uses API-served models. The insight that budget-awareness should be continuous (not just an upfront max_tokens) suggests Podders could use streaming + token counting to issue early-stop signals or dynamically adjust follow-up prompts when a response is running long.

---

## [Faster LLM Inference using DBMS-Inspired Preemption and Cache Replacement Policies](https://arxiv.org/abs/2411.07447)

**Authors:** Kyoungmin Kim, Jiacheng Li, Kijae Hong, Anastasia Ailamaki
**Year:** 2024

**Core method:** Adapts classic database cost models and cache replacement policies to LLM inference serving. Builds resource cost models for concurrent inference requests and their KV-cache footprints in GPU memory, then applies DBMS-style preemption scheduling and eviction strategies to optimize which requests hold GPU memory and when intermediate results are swapped.

**Key results:** Substantially reduces GPU costs when serving multiple concurrent LLM requests. The DBMS-inspired scheduling outperforms default vLLM/FCFS strategies, particularly under high-concurrency workloads where KV-cache memory contention is the bottleneck.

**Relevance to Podders:** Less directly applicable since Podders calls hosted APIs (Anthropic) rather than self-hosting models. However, understanding these server-side dynamics helps explain why batching requests and avoiding burst patterns (sending all 50 Haiku calls simultaneously) can lead to better latency and potentially lower costs via prompt caching benefits.

---

## [Understanding Efficiency: Quantization, Batching, and Serving Strategies in LLM Energy Use](https://arxiv.org/abs/2601.22362)

**Authors:** Julien Delavande, Regis Pierrard, Sasha Luccioni
**Year:** 2026

**Core method:** Performs a detailed empirical study of LLM inference energy and latency on NVIDIA H100 GPUs, systematically varying quantization precision, batch size, and serving configuration (e.g., HuggingFace TGI). Analyzes how each system-level design choice affects energy consumption per token across the prefill (compute-bound) and decode (memory-bound) phases.

**Key results:** Lower-precision quantization only yields energy gains in compute-bound regimes (prefill), not memory-bound ones (decode). Batching improves energy efficiency most during decoding. Structured request timing ("arrival shaping") can reduce per-request energy by up to 100x compared to naive sequential serving. System-level orchestration matters more than model internals for efficiency.

**Relevance to Podders:** The "arrival shaping" finding is the key takeaway -- Podders' ~58 daily API calls should be batched/timed strategically rather than fired individually. Grouping Haiku summarization calls into scheduled batches (e.g., every 30 minutes rather than per-event) could improve Anthropic's server-side batching efficiency, and using the Batches API where available would translate this insight into real cost savings.

---

## Summary

These five papers converge on a central theme: LLM inference cost is highly compressible through intelligent budgeting and scheduling, without significant quality loss. For Podders' specific situation (~50 Haiku + 8 Sonnet calls/day on a $25-29/month budget), three actionable patterns emerge:

1. **Complexity-adaptive token budgets (Papers 1, 2, 3):** Classify each task's difficulty before calling the LLM and set max_tokens accordingly. Simple scrape summaries might need 200-300 output tokens; complex cross-source analysis might need 1500+. A lightweight pre-classifier or even prompt-level heuristics (input length, source count) can drive this. The literature shows 40-60% token reduction is achievable with minimal accuracy loss.

2. **Prompt-level budget hints (Paper 1):** Even without model fine-tuning, simply including a phrase like "Answer in under N tokens" or "Be concise, budget: N tokens" in the system prompt demonstrably reduces output length. This is zero-cost to implement and directly reduces per-call spend on output tokens.

3. **Request batching and timing (Papers 4, 5):** Rather than firing API calls as events arrive, accumulate tasks and dispatch them in batches. Anthropic's Batches API offers 50% cost reduction for non-urgent workloads. For Podders' use case, most scraping/summarization tasks are not latency-sensitive and can tolerate 5-30 minute delays, making batch dispatch an easy win.
