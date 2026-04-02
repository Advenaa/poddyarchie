# Streaming & Real-Time Document Processing

Research on incremental summarization of continuous information streams.

---

## [Can Structural Cues Save LLMs? Evaluating Language Models in Massive Document Streams](https://arxiv.org/abs/2603.19250)
**Authors:** Yukyung Lee, Yebin Lim, Woojun Jung, Wonjun Choi, Susik Yoon
**Year:** 2026

**Core Method:** Introduces StreamBench, a benchmark of 605 events across 15,354 documents with three tasks: Topic Clustering, Temporal Question Answering, and Summarization. Tests LLMs with and without "structural cues" -- organized fact sheets that group key information by event -- to measure how well models handle interleaved concurrent events in a continuous document stream.

**Key Results:** Structural cues improve clustering by up to +4.37% and temporal QA by up to +9.63%, confirming that organizing facts by event helps models separate concurrent topics and locate relevant information. Temporal reasoning (correctly ordering and reasoning about when things happened) remains an inherent weakness of current LLMs even with structural support.

**Relevance to Podders:** Directly validates that Podders should maintain per-topic/per-event structured state when processing interleaved Discord/Twitter/RSS streams, rather than feeding raw mixed streams into an LLM. The clustering and temporal QA tasks mirror what a 3-hour market pulse needs to do: separate concurrent market events and answer "what happened when."

---

## [ReSum: Unlocking Long-Horizon Search Intelligence via Context Summarization](https://arxiv.org/abs/2509.13313)
**Authors:** Xixi Wu, Kuan Li, Yida Zhao, Liwen Zhang, Litu Ou, Huifeng Yin, Zhongwang Zhang, Xinmiao Yu, Dingchu Zhang, Yong Jiang, Pengjun Xie, Fei Huang, Minhao Cheng, Shuai Wang, Hong Cheng, Jingren Zhou
**Year:** 2025 (v1 Sep 2025, revised Mar 2026)

**Core Method:** A lightweight plug-and-play paradigm that periodically condenses accumulated interaction histories into compact summaries using an external summarization tool, enabling unbounded exploration without hitting context window limits. Proposes ReSum-GRPO, which adapts Group Relative Policy Optimization with "advantage broadcasting" to propagate rewards across segmented (summarized) trajectory segments for credit assignment over long horizons.

**Key Results:** ReSum achieves +4.5% over ReAct baselines in training-free settings, and ReSum-GRPO adds another +8.2% gain. A 30B model with ReSum and only 1K training samples matches leading open-source models, demonstrating that periodic summarization is both effective and cheap to implement.

**Relevance to Podders:** The periodic-summarization-as-external-tool pattern maps directly to Podders' 3-hour pulse cycle: accumulate messages, periodically compress into a running summary, then reason over the compressed state. The plug-and-play design means this can be bolted onto existing pipelines without retraining.

---

## [Density-aware Soft Context Compression with Semi-Dynamic Compression Ratio](https://arxiv.org/abs/2603.25926)
**Authors:** Yijiong Yu, Shuai Yuan, Jie Zheng, Huazheng Wang, Ji Pei
**Year:** 2026

**Core Method:** Introduces a Semi-Dynamic Context Compression framework that encodes long contexts into latent tokens with variable compression ratios based on information density. Uses a Discrete Ratio Selector that predicts how dense a text segment is and quantizes the compression ratio to a set of discrete levels, avoiding the instability of fully continuous dynamic ratios. Jointly trained with the compressor on synthetic data using summary lengths as proxy labels.

**Key Results:** The density-aware approach with mean pooling consistently outperforms static (uniform ratio) baselines, establishing a Pareto frontier across compression-vs-quality tradeoffs. The key insight is that discrete ratio buckets work much better than continuous ones -- models struggle with input-dependent continuous structural hyperparameters.

**Relevance to Podders:** Market messages vary wildly in density -- a single whale alert tweet carries more signal than 50 casual Discord messages. This paper validates that compression should be adaptive: spend more tokens on dense, signal-rich content and aggressively compress noise, which is exactly what Podders needs when building 3-hour summaries from heterogeneous sources.

---

## Summary

These three papers converge on a clear architectural direction for streaming document processing. StreamBench (2603.19250) demonstrates that LLMs processing interleaved multi-topic streams benefit significantly from structured, per-event state organization -- raw mixed streams degrade both clustering and temporal reasoning. ReSum (2509.13313) provides a practical pattern for handling unbounded streams by periodically compressing accumulated context into summaries, achieving strong results as a plug-and-play module with no retraining required. The density-aware compression work (2603.25926) adds a crucial refinement: not all content deserves equal compression, and adaptive ratios based on information density outperform uniform approaches. Together, they suggest Podders should (1) cluster incoming messages by topic/event before summarization, (2) use periodic incremental summarization to manage the growing context window, and (3) apply density-aware compression that preserves high-signal content (whale alerts, breaking news) while aggressively compressing noise (casual chat, reposts).
