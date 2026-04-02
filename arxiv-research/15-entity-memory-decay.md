# Entity Memory & Temporal Decay

Research on temporal knowledge management, relevance decay, and forgetting in AI systems.

---

## [Rethinking Memory in LLM-based Agents: Representations, Operations, and Emerging Topics](https://arxiv.org/abs/2505.00675)

**Authors:** Yiming Du, Wenyu Huang, Danna Zheng, Zhaowei Wang, Sebastien Montella, Mirella Lapata, Kam-Fai Wong, Jeff Z. Pan
**Year:** 2025

**Core method:** A comprehensive survey that categorizes agent memory into parametric (implicit in model weights) and contextual (explicit external data, structured/unstructured) forms. Defines six core atomic operations governing memory dynamics: Consolidation, Updating, Indexing, Forgetting, Retrieval, and Condensation. Maps these operations across memory types to identify four key research topics: long-term memory, long-context memory, parametric modification, and multi-source memory.

**Key results:** The taxonomy reveals that "Forgetting" is a first-class memory operation, not just data loss -- it is an intentional mechanism for managing memory capacity and relevance. The survey identifies that most current systems lack principled forgetting strategies, instead relying on simple TTL or capacity-based eviction. The paper also highlights that Condensation (summarizing/compressing memories) is underexplored but critical for long-running agents.

**Relevance to Podders:** Podders' 5%/day decay + 90-day pruning maps to the Forgetting operation in this taxonomy. The survey suggests Podders could benefit from adding Condensation -- instead of pruning old entities entirely, condense them into summary nodes that preserve historical context without full storage cost.

---

## [Zep: A Temporal Knowledge Graph Architecture for Agent Memory](https://arxiv.org/abs/2501.13956)

**Authors:** Preston Rasmussen, Pavlo Paliychuk, Travis Beauvais, Jack Ryan, Daniel Chalef
**Year:** 2025

**Core method:** Zep introduces Graphiti, a temporally-aware knowledge graph engine that dynamically synthesizes both unstructured conversational data and structured business data while maintaining historical relationships. Unlike static RAG, Graphiti tracks when facts were learned and when they were superseded, creating a temporal chain of knowledge states. Edges in the graph carry temporal metadata (valid_from, valid_to) enabling point-in-time queries.

**Key results:** Outperforms MemGPT on the Deep Memory Retrieval benchmark (94.8% vs 93.4%). On the more challenging LongMemEval benchmark (complex temporal reasoning), achieves up to 18.5% accuracy improvement while reducing response latency by 90% compared to baselines. Particularly strong on cross-session information synthesis and long-term context maintenance.

**Relevance to Podders:** Directly relevant architecture. Podders' entity system could adopt Graphiti's temporal edge model -- instead of a flat decay multiplier, track valid_from/valid_to on entity relationships so that when an entity "decays," its historical contributions are preserved and queryable. This would let Podders answer "what was the market sentiment around X token three weeks ago?" even after the entity has been deprioritized.

---

## [MAGMA: A Multi-Graph based Agentic Memory Architecture for AI Agents](https://arxiv.org/abs/2601.03236)

**Authors:** Dongming Jiang, Yi Li, Guanpeng Li, Bingzhe Li
**Year:** 2026

**Core method:** MAGMA represents each memory item across four orthogonal graph views: semantic, temporal, causal, and entity graphs. Rather than entangling all information in a single monolithic store, each graph captures a different relational dimension. Retrieval is formulated as policy-guided traversal over these views, where an agent selects which graph(s) to traverse based on query intent, enabling query-adaptive memory selection and structured context construction.

**Key results:** Consistently outperforms state-of-the-art agentic memory systems (including Zep/Graphiti) on LoCoMo and LongMemEval benchmarks for long-horizon reasoning tasks. The decoupled multi-graph design provides transparent reasoning paths and fine-grained control over retrieval, improving interpretability of why certain memories were retrieved.

**Relevance to Podders:** The multi-graph decomposition maps naturally to Podders' needs: a semantic graph for entity relationships, a temporal graph for decay/relevance windows, a causal graph for "X caused Y price movement" chains, and an entity graph for identity resolution. This architecture would let Podders separate "when was this relevant?" (temporal) from "what is this related to?" (semantic) queries, which are currently conflated in the single relevance score.

---

## Summary

The field is moving from flat memory stores toward structured, temporally-aware graph architectures:

1. **Forgetting as a first-class operation** (Rethinking Memory survey) -- Podders' 5%/day linear decay is a valid but simplistic forgetting strategy. The literature suggests more sophisticated approaches: event-driven decay (entities decay faster when contradicted by new information), condensation (compress old entities into summaries before pruning), and importance-weighted forgetting (high-impact entities decay slower).

2. **Temporal knowledge graphs** (Zep/Graphiti) -- Instead of a single relevance score that decays, track temporal validity windows on entity relationships. This preserves historical knowledge for retrospective analysis while still prioritizing recent information for real-time alerts. Zep's Graphiti engine is open-source and could be evaluated as a replacement for Podders' current entity storage.

3. **Multi-graph decomposition** (MAGMA) -- Separate temporal, semantic, causal, and entity dimensions into independent graph layers. This is the most architecturally ambitious option but addresses a real limitation in Podders: the current single relevance score conflates recency, importance, and semantic centrality.

**Immediate actionable takeaway for Podders:** Consider replacing the flat 5%/day decay with a two-tier system: (1) active entities with full detail and recency-weighted relevance, and (2) condensed historical entities that preserve key facts (peak sentiment, notable events, relationship graph position) without full storage cost. This mirrors the Condensation operation from the survey and the temporal edge model from Zep, and avoids losing valuable historical signal when entities cross the 90-day threshold.
