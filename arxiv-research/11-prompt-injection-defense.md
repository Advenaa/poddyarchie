# Prompt Injection Defense

Research on protecting LLM pipelines from malicious content in untrusted inputs.

---

## [When AI Meets the Web: Prompt Injection Risks in Third-Party AI Chatbot Plugins](https://arxiv.org/abs/2511.05797)

- **Authors:** Yigitcan Kaya, Anton Landerer, Stijn Pletinckx, Michelle Zimmermann, Christopher Kruegel, Giovanni Vigna
- **Year:** 2025 (IEEE S&P 2026)
- **Core method:** Large-scale empirical study of 17 third-party chatbot plugins across 10,000+ websites. Identifies two classes of vulnerability: (1) failure to enforce conversation history integrity, allowing forged system messages that boost attack success 3-8x, and (2) tools like web-scraping that mix trusted content (product descriptions) with untrusted third-party content (customer reviews) without distinction.
- **Key results:** 8 of 17 plugins (covering 8,000 websites) fail to enforce conversation history integrity. 15 plugins offer context-enrichment tools that introduce indirect prompt injection risk. ~13% of e-commerce sites already expose chatbots to untrusted third-party content. Many plugins adopt insecure defaults that undermine built-in LLM safeguards.
- **Relevance to Podders:** Directly validates the threat model Podders faces -- untrusted social media content (Discord messages, tweets) is analogous to customer reviews being mixed into LLM context. Reinforces the importance of clearly separating trusted system instructions from untrusted data at the architectural level.

---

## [The Landscape of Prompt Injection Threats in LLM Agents: From Taxonomy to Analysis](https://arxiv.org/abs/2602.10453)

- **Authors:** Peiran Wang, Xinfeng Li, Chong Xiang, Jinghuai Zhang, Ying Li, Lixia Zhang, Xiaofeng Wang, Yuan Tian
- **Year:** 2026
- **Core method:** Systematization of Knowledge (SoK) providing taxonomies for prompt injection attacks (heuristic vs. optimization payload generation) and defenses (text-level, model-level, execution-level interventions). Introduces AgentPI, a benchmark for evaluating defenses in context-dependent agent settings where agents must rely on runtime environmental observations.
- **Key results:** No single defense simultaneously achieves high trustworthiness, high utility, and low latency. Many defenses that appear effective on existing benchmarks work by suppressing contextual inputs, but fail in realistic agent settings where context-dependent reasoning is essential. Highlights a critical blind spot in current evaluation practices.
- **Relevance to Podders:** Podders is a context-dependent system -- the LLM must reason over scraped social content to extract intelligence. This paper warns that naive defenses (e.g., aggressively filtering inputs) may destroy the utility of summarization and condition-detection tasks. Defense selection must balance security against analytical capability.

---

## [Defending Against Prompt Injection with DataFilter](https://arxiv.org/abs/2510.19207)

- **Authors:** Yizhu Wang, Sizhe Chen, Raghad Alkhudair, Basel Alomair, David Wagner
- **Year:** 2025 (revised Feb 2026)
- **Core method:** A test-time, model-agnostic preprocessing defense that strips malicious instructions from data before it reaches the backend LLM. DataFilter is a separate model trained via supervised fine-tuning on simulated injections; it takes both the user's instruction and the untrusted data as input and selectively removes adversarial content while preserving benign information.
- **Key results:** Reduces prompt injection attack success rates to near zero across multiple benchmarks while maintaining LLM utility. Works as a plug-and-play layer in front of black-box commercial LLMs. Model and code are publicly released (HuggingFace: JoyYizhu/DataFilter).
- **Relevance to Podders:** Highly applicable -- DataFilter could be deployed as a preprocessing step before Discord/Twitter content enters the Podders LLM pipeline. Its model-agnostic, plug-and-play nature means it could sit between the scraper output and the XML-delimited prompt without requiring changes to the downstream LLM or prompt structure.

---

## [Zero-Shot Embedding Drift Detection (ZEDD): A Lightweight Defense Against Prompt Injections in LLMs](https://arxiv.org/abs/2601.12359)

- **Authors:** Anirudh Sekar, Mrinal Agarwal, Rachel Sharma, Akitsugu Tanaka, Jasmine Zhang, Arjun Damerla, Kevin Zhu
- **Year:** 2026 (NeurIPS 2025 Lock-LLM Workshop)
- **Core method:** Detects prompt injection by measuring semantic drift in embedding space between benign and suspect inputs using cosine similarity. Operates zero-shot -- no access to model internals, no prior knowledge of attack types, no task-specific retraining required. Uses adversarial-clean prompt pairs to quantify embedding drift as a detection signal.
- **Key results:** Achieves >93% accuracy detecting prompt injections across Llama 3, Qwen 2, and Mistral architectures with <3% false positive rate. Embedding drift is shown to be a robust and transferable signal that outperforms traditional detection methods in both accuracy and operational efficiency.
- **Relevance to Podders:** Could serve as a lightweight pre-screening layer for incoming social media content. Before injecting Discord/Twitter text into the LLM context, ZEDD could flag suspicious messages with minimal latency overhead. The zero-shot property is valuable since Podders processes diverse, unpredictable content formats.

---

## [A Multi-Agent LLM Defense Pipeline Against Prompt Injection Attacks](https://arxiv.org/abs/2509.14285)

- **Authors:** S M Asif Hossain, Ruksat Khan Shayoni, Mohd Ruhul Ameen, Akif Islam, M. F. Mridha, Jungpil Shin
- **Year:** 2025 (IEEE WIECON-ECE 2025)
- **Core method:** Uses multiple specialized LLM agents in coordinated pipelines to detect and neutralize prompt injection in real-time. Evaluates two architectures: a sequential chain-of-agents pipeline and a hierarchical coordinator-based system, where each agent handles a specific aspect of injection detection (e.g., pattern matching, semantic analysis, intent verification).
- **Key results:** Achieved 100% mitigation (ASR reduced to 0%) across 400 attack instances in 8 categories on ChatGLM and Llama2, compared to baseline ASR of 20-30%. Maintains system functionality for legitimate queries. Effective against direct overrides, code execution attempts, data exfiltration, and obfuscation.
- **Relevance to Podders:** The multi-agent approach is architecturally interesting but likely too expensive for Podders' high-throughput social media processing (multiple LLM calls per input). Could be worth considering for high-stakes outputs (e.g., trade signal generation) but impractical for bulk Discord message processing.

---

## [Know Thy Enemy: Securing LLMs Against Prompt Injection via Diverse Data Synthesis and Instruction-Level Chain-of-Thought Learning](https://arxiv.org/abs/2601.04666)

- **Authors:** Zhiyuan Chang, Mingyang Li, Yuekai Huang, Ziyou Jiang, Xiaojun Jia, Qian Xiong, Junjie Wang, Zhaoyang Li, Qing Wang
- **Year:** 2026
- **Core method:** InstruCoT -- a model enhancement method that synthesizes diverse training data covering multiple injection vectors and positions, then fine-tunes LLMs with instruction-level chain-of-thought reasoning to identify and reject malicious instructions regardless of source or context position. The CoT process helps the model reason about whether each instruction in context is legitimate or injected.
- **Key results:** Significantly outperforms baselines across three dimensions: Behavior Deviation, Privacy Leakage, and Harmful Output, tested on four LLMs. Maintains utility performance without degradation, meaning the model can still perform its intended tasks while rejecting injections.
- **Relevance to Podders:** If Podders moves to self-hosted models, InstruCoT-style fine-tuning could harden the summarization/analysis LLM against injections embedded in social media content. Not applicable to black-box API usage (requires fine-tuning access), but relevant for future architecture decisions.

---

## Summary

These six papers span the prompt injection defense landscape from taxonomy/analysis to concrete defenses. For Podders' specific threat model -- untrusted social media content processed through LLMs wrapped in XML delimiters -- the most immediately actionable approaches are:

1. **DataFilter** (preprocessing layer that strips injections before they reach the LLM) is the strongest candidate for near-term deployment. It is model-agnostic, plug-and-play, and publicly available. It could sit between Podders' scrapers and the LLM prompt construction step.

2. **ZEDD** (embedding drift detection) offers a lightweight, zero-shot detection layer that could flag suspicious content before or alongside LLM processing, useful as a secondary signal or for audit logging.

3. **The AgentPI benchmark findings** from the SoK paper are a critical strategic warning: many defenses that look good on benchmarks actually degrade utility for context-dependent tasks like Podders' market intelligence extraction. Any defense Podders adopts must be evaluated against real summarization/condition-detection tasks, not just injection detection benchmarks.

4. **InstruCoT fine-tuning** becomes relevant if Podders eventually self-hosts models, enabling injection resistance to be baked into the model itself.

5. The **multi-agent pipeline** approach, while effective, carries too much latency and cost overhead for bulk social media processing but could protect high-value downstream steps.
