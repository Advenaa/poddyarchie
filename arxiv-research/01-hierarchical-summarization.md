# Hierarchical / MapReduce Summarization

Research on compressing massive text through multiple LLM stages.

---

## [LLM x MapReduce: Simplified Long-Sequence Processing using Large Language Models](https://arxiv.org/abs/2410.09342)
**Authors:** Zihan Zhou, Chong Li, Xinyi Chen, Shuo Wang, Yu Chao, Zhili Li, Haoyu Wang, Rongqiao An, Qi Shi, Zhixing Tan, Xu Han, Xiaodong Shi, Zhiyuan Liu, Maosong Sun
**Year:** 2024

**Core Method:** A training-free divide-and-conquer framework that splits long documents into chunks, has an LLM process each chunk (Map), then aggregates intermediate answers into a final output (Reduce). The key innovations are a structured information protocol to handle inter-chunk dependencies (where information needed to answer a question is split across chunks) and an in-context confidence calibration mechanism to resolve inter-chunk conflicts (where chunks produce contradictory answers).

**Key Results:** Outperforms representative open-source and commercial long-context LLMs on document understanding tasks, despite using standard-context models. The framework is model-agnostic and can be applied to several different LLMs without any fine-tuning.

**Relevance to Podders:** Directly validates Podders' chunk-then-synthesize architecture. The structured information protocol (passing structured metadata between Map and Reduce stages) and confidence calibration (resolving contradictions across source chunks) are both techniques Podders should adopt, especially when Discord/news messages from different sources conflict.

---

## [LLM x MapReduce-V2: Entropy-Driven Convolutional Test-Time Scaling for Generating Long-Form Articles from Extremely Long Resources](https://arxiv.org/abs/2504.05732)
**Authors:** Haoyu Wang, Yujia Fu, Zhu Zhang, Shuo Wang, Zirui Ren, Xiaorong Wang, Zhili Li, Chaoqun He, Bo An, Zhiyuan Liu, Maosong Sun
**Year:** 2025

**Core Method:** Extends V1 from QA to long-form generation (long-to-long). Inspired by convolutional neural networks, it uses stacked convolutional scaling layers that progressively expand local chunk understanding into higher-level global representations. An entropy-driven mechanism determines where additional compute should be allocated at test time, focusing processing on the most uncertain or information-dense regions.

**Key Results:** Substantially enhances LLMs' ability to process extremely long inputs and generate coherent, informative long-form articles (e.g., survey papers). Outperforms several representative baselines on long-to-long generation tasks. Also introduces SurveyEval, a benchmark for evaluating generated long-form articles.

**Relevance to Podders:** The convolutional multi-pass approach is highly relevant -- Podders could use iterative summarization layers where each pass integrates broader context, rather than a single Map-then-Reduce. The entropy-driven compute allocation suggests Podders should spend more synthesis budget on topics with higher information density or conflicting signals.

---

## [NexusSum: Hierarchical LLM Agents for Long-Form Narrative Summarization](https://arxiv.org/abs/2505.24575)
**Authors:** Hyuntak Kim, Byung-Hak Kim
**Year:** 2025 (Accepted to ACL 2025 Main Track)

**Core Method:** A multi-agent LLM framework for narrative summarization using a structured sequential pipeline without fine-tuning. Two key innovations: (1) Dialogue-to-Description Transformation that standardizes heterogeneous text formats (dialogue, description) into a unified format before summarization, and (2) Hierarchical Multi-LLM Summarization that optimizes chunk processing order and controls output length at each stage for accurate summaries.

**Key Results:** Achieves up to 30.0% improvement in BERTScore (F1) over baselines across books, movies, and TV scripts. Demonstrates that multi-agent LLM pipelines with proper preprocessing and length control significantly outperform single-pass approaches for long-form content.

**Relevance to Podders:** The dialogue-to-description preprocessing step is directly applicable -- Podders ingests heterogeneous sources (Discord chat, news articles, tweets) that should be normalized into a consistent format before the Map stage. The output length control at each hierarchical level is also important for managing token budgets across cheap/expensive LLM tiers.

---

## [ToM: Leveraging Tree-oriented MapReduce for Long-Context Reasoning in Large Language Models](https://arxiv.org/abs/2511.00489)
**Authors:** Jiani Guo, Zuchao Li, Jie Wu, Qianren Wang, Yun Li, Lefei Zhang, Hai Zhao, Yujiu Yang
**Year:** 2025 (EMNLP 2025 Main Conference)

**Core Method:** Instead of flat chunking, ToM constructs a DocTree by hierarchical semantic parsing that respects the document's inherent structure (headings, subheadings). It then performs bottom-up tree aggregation: in the Map step, rationales are generated at leaf/child nodes; in the Reduce step, sibling-node rationales are aggregated at parent nodes to resolve conflicts or reach consensus. This recursive tree structure replaces the flat Map-Reduce paradigm.

**Key Results:** Significantly outperforms both flat divide-and-conquer frameworks and RAG methods on long-context reasoning tasks using 70B+ parameter LLMs. Achieves better logical coherence by preserving document structure and resolving conflicts through hierarchical aggregation rather than flat merging.

**Relevance to Podders:** Suggests Podders should organize messages by topic/thread hierarchy rather than flat time-based chunks. Grouping Discord messages by channel/thread and news by topic before summarization, then aggregating summaries bottom-up through a topic tree, would preserve thematic coherence better than flat chunking.

---

## [SciZoom: A Large-scale Benchmark for Hierarchical Scientific Summarization across the LLM Era](https://arxiv.org/abs/2603.16131)
**Authors:** Han Jang, Junhyeok Lee, Kyu Sung Choi
**Year:** 2026 (Submitted to KDD 2026)

**Core Method:** Not a method paper but a benchmark: 44,946 ML papers from NeurIPS/ICLR/ICML/EMNLP (2020-2025) with three hierarchical summarization targets at different granularities -- Abstract (~150 words), Contributions (~50 words), and TL;DR (~20 words) -- achieving compression ratios up to 600:1. The dataset is stratified into Pre-LLM and Post-LLM eras to study how LLM-assisted writing has changed scientific prose.

**Key Results:** Linguistic analysis reveals that Post-LLM era papers show up to 10x increase in formulaic expressions and a 23% decline in hedging language, suggesting LLM-assisted writing produces more confident but homogenized prose. The benchmark enables evaluation of hierarchical summarization at multiple compression levels.

**Relevance to Podders:** The multi-granularity summarization hierarchy (TL;DR -> Contributions -> Abstract -> Full paper) maps directly to Podders' need for zoom levels: quick alerts, daily summaries, and deep-dive reports. Designing Podders' output tiers around tested compression ratios (e.g., 600:1 for TL;DR, 30:1 for summaries) gives concrete targets.

---

## [From Local to Global: A Graph RAG Approach to Query-Focused Summarization](https://arxiv.org/abs/2404.16130)
**Authors:** Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, Dasha Metropolitansky, Robert Osazuwa Ness, Jonathan Larson
**Year:** 2024

**Core Method:** GraphRAG builds a two-stage graph index: first, an LLM extracts entities and relationships from source documents to create a knowledge graph; then, community detection groups closely-related entities, and the LLM pre-generates summaries for each community. At query time, each community summary produces a partial response, and all partial responses are synthesized into a final answer. This enables answering "global" sensemaking questions (e.g., "what are the main themes?") that standard vector-similarity RAG cannot handle.

**Key Results:** On global sensemaking questions over datasets in the 1 million token range, GraphRAG substantially improves both comprehensiveness and diversity of generated answers compared to conventional RAG. The community-based summarization approach scales to large corpora while preserving thematic structure.

**Relevance to Podders:** GraphRAG's entity-graph + community-summary approach is the strongest fit for Podders' knowledge layer. Instead of just chunking messages temporally, Podders could extract entities (tokens, projects, people, events) and their relationships during the Map stage, build a knowledge graph, then use community summaries to answer cross-cutting market questions like "what are the emerging narratives around X?"

---

## Summary

The literature converges on several principles directly applicable to Podders' multi-stage market intelligence pipeline. First, flat sequential chunking is suboptimal -- both ToM and GraphRAG show that structure-aware decomposition (by topic hierarchy or entity graph) significantly outperforms naive splitting, which Podders should adopt by grouping messages by source/topic/thread rather than just time windows. Second, the critical challenge in any MapReduce LLM pipeline is handling inter-chunk dependencies and conflicts; LLM x MapReduce's structured information protocol and confidence calibration provide concrete techniques for Podders' Reduce stage when synthesizing contradictory market signals from different sources. Third, V2's convolutional multi-pass approach and entropy-driven compute allocation suggest that Podders' "cheap LLM Map / expensive LLM Reduce" split could be further optimized by adding intermediate aggregation layers and concentrating expensive model calls on high-uncertainty or high-density topics. Finally, NexusSum's preprocessing normalization and SciZoom's multi-granularity compression ratios give practical design targets: normalize heterogeneous inputs (Discord, news, tweets) before summarization, and design output tiers at specific compression levels (600:1 for alerts, 30:1 for reports) to match user needs.
