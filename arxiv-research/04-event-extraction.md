# Event Extraction & Detection

Research on extracting structured events from unstructured text and preventing hallucinations.

---

## [Event Extraction in Large Language Model](https://arxiv.org/abs/2512.19537)

- **Title & Authors:** Event Extraction in Large Language Model -- Bobo Li, Xudong Han, Jiang Liu, Yuzhe Ding, Liqiang Jing, Zhaoqi Zhang, Jinheng Li, Xinya Du, Fei Li, Meishan Zhang, Min Zhang, Aixin Sun, Philip S. Yu, Hao Fei
- **Year:** 2025
- **Core method:** A holistic survey arguing that event extraction (EE) should serve as a "cognitive scaffold" for LLM-based systems. Event schemas and slot constraints act as grounding interfaces for verification; event-centric structures provide controlled intermediate representations for stepwise reasoning; event links enable relation-aware retrieval via graph-based RAG; and event stores provide updatable episodic memory beyond the context window.
- **Key results/findings:** LLM prompting and generation can produce structured event outputs in zero-shot or few-shot settings, but pipelines still suffer from hallucinations under weak constraints, fragile temporal/causal linking over long contexts, and limited long-horizon knowledge management. The survey traces method evolution from rule-based and neural models to instruction-driven and generative frameworks, covering text and multimodal settings, cross-lingual, low-resource, and domain-specific scenarios.
- **Relevance to Podders:** Directly applicable -- the idea of using event schemas with slot constraints as grounding and verification interfaces maps to Podders' need for structured extraction (entity, sentiment, urgency) with hallucination prevention. The concept of event stores as updatable memory is relevant for Podders' knowledge graph accumulation across sources.

---

## [Towards Event Extraction with Massive Types: LLM-based Collaborative Annotation and Partitioning Extraction](https://arxiv.org/abs/2503.02628)

- **Title & Authors:** Towards Event Extraction with Massive Types: LLM-based Collaborative Annotation and Partitioning Extraction -- Wenxuan Liu, Zixuan Li, Long Bai, Yuxin Zuo, Daozhu Xu, Xiaolong Jin, Jiafeng Guo, Xueqi Cheng
- **Year:** 2025
- **Core method:** Proposes two innovations: (1) a collaborative annotation method where multiple LLMs refine trigger word annotations from distant supervision, carry out argument annotation, then vote to consolidate preferences, producing the EEMT dataset (200k+ samples, 3,465 event types, 6,297 role types). (2) LLM-PEE, a partitioning extraction method that recalls candidate event types and splits them into multiple partitions to fit within LLM context limits.
- **Key results/findings:** LLM-PEE outperforms SOTA by +5.4 F1 in event detection and +6.1 in argument extraction in supervised settings. In zero-shot settings, it achieves up to +12.9 improvement over mainstream LLMs. The multi-LLM voting annotation approach produces high-quality training data at scale without human annotators.
- **Relevance to Podders:** The partitioning strategy for handling massive event type schemas is directly useful -- Podders needs to classify social media messages into many market-relevant event types (token launches, exchange listings, security incidents, regulatory actions, etc.) and the partition-then-extract pattern could manage this at scale. The multi-LLM voting approach could improve annotation quality for Podders' training data.

---

## [Reducing Hallucinations in Summarization via Reinforcement Learning with Entity Hallucination Index](https://arxiv.org/abs/2507.22744)

- **Title & Authors:** Reducing Hallucinations in Summarization via Reinforcement Learning with Entity Hallucination Index -- Praveenkumar Katwe, Rakesh Chandra, Balabantaray Kali, Prasad Vittala
- **Year:** 2025
- **Core method:** Introduces a reward-driven fine-tuning framework that optimizes for Entity Hallucination Index (EHI), a metric quantifying the presence, correctness, and grounding of named entities in generated summaries. Baseline summaries are generated from a pre-trained LM, EHI scores are computed via automatic entity extraction and matching against source text, then reinforcement learning fine-tunes the model using EHI as a reward signal.
- **Key results/findings:** The approach consistently improves EHI across datasets (meeting transcripts), significantly reducing entity-level hallucinations without degrading fluency or informativeness. Importantly, no human-written factuality annotations are required, making the approach scalable. A reproducible Colab pipeline is released.
- **Relevance to Podders:** Highly relevant -- Podders extracts entities (tokens, projects, people, exchanges) from social media and needs to verify they actually appear in source text. The EHI metric (entity extraction + matching against source) could be adapted as a post-extraction validation step or used as an RL reward signal to fine-tune Podders' extraction models toward entity-faithful outputs.

---

## Summary

These three papers cover complementary aspects of reliable event extraction for LLM-based systems. The survey (Li et al.) establishes the architectural principle that event schemas and slot constraints should serve as grounding scaffolds to prevent hallucinations -- directly applicable to Podders' structured extraction pipeline. The collaborative annotation paper (Liu et al.) solves the practical problem of scaling event type coverage by using multi-LLM voting to build large annotated datasets and partitioning extraction to handle thousands of event types within LLM context limits. The EHI paper (Katwe et al.) provides a concrete, implementable metric and RL-based fine-tuning approach for reducing entity hallucinations in generated outputs, which maps directly to Podders' need to validate extracted entities against source social media messages. Together, they suggest a pipeline architecture: define event schemas with slot constraints for grounding, use partitioned extraction for broad event type coverage, and apply entity hallucination scoring as a post-extraction validation layer.
