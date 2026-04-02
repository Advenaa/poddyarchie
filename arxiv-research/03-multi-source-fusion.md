# Multi-Source Fusion & Cross-Document Event Extraction

Research on detecting and merging the same events across different sources.

---

## [Harvesting Events from Multiple Sources: Towards a Cross-Document Event Extraction Paradigm](https://arxiv.org/abs/2406.16021)

- **Authors:** Qiang Gao, Zixiang Meng, Bobo Li, Jun Zhou, Fei Li, Chong Teng, Donghong Ji
- **Year:** 2024 (ACL 2024 Findings)

**Core method:** Proposes cross-document event extraction (CDEE), a task that integrates event information scattered across multiple documents into unified structured records. The pipeline has 5 stages: per-document event extraction, coreference resolution, entity normalization, role normalization, and entity-role resolution. They also release CLES, a dataset of 20,059 documents with 37,688 mention-level events (>70% cross-document).

**Key results:** The end-to-end CDEE pipeline achieves roughly 72% F1, demonstrating the task is feasible but challenging. Single-document extraction misses substantial event arguments and introduces source bias, confirming the value of cross-document merging.

**Relevance to Podders:** Directly applicable -- Podders needs to detect the same event (e.g., a token listing, a hack, a regulatory action) mentioned across Discord, Twitter, RSS, and news, then merge partial details from each source into one canonical event record. The 5-stage pipeline (extract -> coreference -> normalize -> resolve) is a practical blueprint.

---

## [Incentive-Aligned Multi-Source LLM Summaries](https://arxiv.org/abs/2509.25184)

- **Authors:** Yanchen Jiang, Zhe Feng, Aranyak Mehta
- **Year:** 2025 (ICLR 2026)

**Core method:** Introduces Truthful Text Summarization (TTS), a framework for synthesizing multiple potentially conflicting sources while filtering unreliable ones. TTS decomposes a draft summary into atomic claims, elicits each source's stance on every claim, scores sources using a peer-prediction mechanism that rewards informative agreement, and filters low-quality sources before re-summarizing. Formal game-theoretic guarantees ensure that truthful reporting is the utility-maximizing strategy for sources.

**Key results:** TTS improves factual accuracy and robustness of multi-source summaries while preserving fluency. It aligns exposure with informative corroboration and disincentivizes adversarial manipulation of the summary pipeline.

**Relevance to Podders:** When Podders aggregates signals from Discord (often noisy/adversarial), Twitter, and news, it needs a principled way to score source reliability and filter manipulation. The atomic-claim decomposition + peer-prediction scoring pattern can serve as a trust layer: break a developing narrative into claims, check cross-source agreement, and down-weight unreliable or coordinated sources.

---

## [Tell me what I need to know: Exploring LLM-based (Personalized) Abstractive Multi-Source Meeting Summarization](https://arxiv.org/abs/2410.14545)

- **Authors:** Frederic Kirstein, Terry Ruas, Robert Kratel, Bela Gipp
- **Year:** 2024 (EMNLP 2024 Industry Track)

**Core method:** A three-stage LLM pipeline for multi-source summarization: (1) identify passages in the primary transcript that need additional context, (2) retrieve and inject relevant details from supplementary materials (slides, documents), and (3) generate a summary from the enriched transcript. Additionally introduces a personalization protocol that extracts participant roles/characteristics and tailors summaries per user.

**Key results:** The multi-source enrichment increases summary relevance by ~9% and produces more content-rich outputs. Personalization improves informativeness by ~10%. The paper also benchmarks performance-cost trade-offs across four LLM families, including edge-device models.

**Relevance to Podders:** The "identify gaps -> enrich from supplementary sources -> summarize" pattern maps well to Podders' workflow: a Discord message mentions an event with sparse detail, the system retrieves corroborating context from news/RSS, enriches the record, then generates a user-facing summary. The personalization protocol is also relevant for tailoring market intelligence outputs to different user profiles (trader vs. researcher).

---

## [A Multisource Fusion Framework for Cryptocurrency Price Movement Prediction](https://arxiv.org/abs/2409.18895)

- **Authors:** Saeed Mohammadi Dashtaki, Reza Mohammadi Dashtaki, Mehdi Hosseini Chagahi, Behzad Moshiri, Md. Jalil Piran
- **Year:** 2024

**Core method:** Fuses quantitative financial indicators (historical prices, technical indicators) with qualitative sentiment signals from X/Twitter. Sentiment is extracted using FinBERT (a finance-domain BERT), and the combined feature vectors are fed into a Bidirectional LSTM (BiLSTM) network for price movement classification.

**Key results:** On a large-scale Bitcoin dataset, the fused model achieves ~96.8% accuracy on price movement prediction, substantially outperforming single-source (price-only or sentiment-only) baselines. The results confirm that real-time social sentiment adds significant predictive value beyond traditional technical analysis.

**Relevance to Podders:** This is the closest domain match -- crypto market intelligence from social + quantitative data. The FinBERT sentiment extraction from Twitter and the late-fusion architecture (sentiment features + price features -> BiLSTM) validate the core Podders thesis: combining social chatter with structured market data yields materially better signal. The FinBERT + BiLSTM pattern could be adapted for Podders' condition detection pipeline.

---

## Summary

These four papers collectively validate the core architecture Podders needs: extracting structured events from multiple heterogeneous sources and merging them into unified records (Gao et al.), filtering unreliable or adversarial sources using cross-source agreement scoring (Jiang et al.), enriching sparse mentions with context from supplementary sources via a staged LLM pipeline (Kirstein et al.), and fusing social sentiment with quantitative signals for crypto-specific prediction (Dashtaki et al.). The key design patterns to carry forward are: (1) a multi-stage pipeline of extract-coreference-normalize-resolve for cross-document event deduplication, (2) atomic claim decomposition with peer-prediction scoring as a trust/reliability layer, (3) gap-detection + enrichment as a pre-summarization step, and (4) FinBERT-style domain-specific sentiment encoding fused with structured data via sequence models.
