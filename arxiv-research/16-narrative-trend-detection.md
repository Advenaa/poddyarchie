# Narrative & Trend Detection

Research on detecting emerging topics, narrative shifts, and trend clustering in text streams.

---

## [Narrative Shift Detection: A Hybrid Approach of Dynamic Topic Models and Large Language Models](https://arxiv.org/abs/2506.20269)
**Authors:** Kai-Robin Lange, Tobias Schmidt, Matthias Reccius, Henrik Mueller, Michael Roos, Carsten Jentsch
**Year:** 2025

**Core Method:** Combines dynamic topic models with LLMs to detect narrative shifts over time. A topic model with change point detection first identifies temporal shifts in a topic of interest, then filters representative documents for that shift and feeds them into an LLM that interprets whether a content shift or a deeper narrative shift occurred, using the Narrative Policy Framework.

**Key Results:** Tested on Wall Street Journal articles from 2009-2023. LLMs can efficiently extract and describe a narrative shift when one exists at a given point in time, but struggle to reliably distinguish between surface-level content shifts and genuine narrative shifts. The hybrid approach significantly reduces LLM cost by filtering the corpus down to only change-point-relevant documents.

**Relevance to Podders:** The two-stage architecture (cheap topic model for filtering, expensive LLM for interpretation) is directly applicable to Podders' need to monitor high-volume crypto social streams cost-effectively. The change point detection component could flag moments when narratives like "RWA rotation" emerge or shift.

---

## [BERTrend: Neural Topic Modeling for Emerging Trends Detection](https://arxiv.org/abs/2411.05930)
**Authors:** Allaa Boutaleb, Jerome Picault, Guillaume Grosjean
**Year:** 2024

**Core Method:** Uses neural topic modeling (BERTopic-based) in an online/streaming setting to detect and track emerging trends. Introduces a popularity metric that considers both document volume and update frequency over time, classifying topics into noise, weak signals, or strong signals. LLMs are used downstream to generate interpretable labels and summaries for detected trends.

**Key Results:** Demonstrated on two large real-world datasets, BERTrend accurately detects weak signals (emerging topics) while filtering noise. It supports both real-time monitoring and retrospective analysis of past events. The signal classification (noise/weak/strong) provides actionable triage for analysts rather than raw topic lists.

**Relevance to Podders:** The weak-signal vs. strong-signal classification is exactly what Podders needs to distinguish early-stage narratives (e.g., nascent "memecoin season" chatter) from established trends. The online/streaming architecture matches Podders' continuous ingestion from Discord, Twitter, and news feeds.

---

## [Online Density-Based Clustering for Real-Time Narrative Evolution Monitoring](https://arxiv.org/abs/2601.20680)
**Authors:** Ostap Vykhopen, Viktoria Skorik, Maksym Tereshchenko, Veronika Solopova
**Year:** 2026

**Core Method:** Investigates replacing batch HDBSCAN clustering with online density-based algorithms (DenStream and others) in a production narrative monitoring pipeline that processes multilingual social media streams. Evaluates cluster quality, computational efficiency, memory footprint, and integration with downstream LLM-based narrative extraction using sliding-window simulations.

**Key Results:** DenStream achieved the strongest overall performance, balancing temporal stability with narrative coherence. The study reveals concrete trade-offs between batch and streaming approaches: batch HDBSCAN produces better clusters but cannot scale to continuous streams, while online methods sacrifice some cluster quality for real-time adaptability and lower memory usage. Human validation confirmed semantic interpretability of online clusters.

**Relevance to Podders:** Directly addresses the production engineering challenge Podders faces: clustering a continuous stream of crypto social media posts into coherent narratives without expensive batch reprocessing. DenStream's streaming approach would allow Podders to detect narrative evolution (e.g., a DeFi narrative shifting to include RWA themes) in near real-time.

---

## [Large Language Model Enhanced Clustering for News Event Detection](https://arxiv.org/abs/2406.10552)
**Authors:** Adane Nega Tarekegn
**Year:** 2024

**Core Method:** Presents an event detection framework that uses LLMs at both ends of the clustering pipeline: pre-clustering (LLM-based keyword extraction and text embeddings) and post-clustering (LLM-generated event summaries and topic labels). Clustering is applied to news from the GDELT database. Introduces a Cluster Stability Assessment Index (CSAI) that uses multiple feature vectors to measure clustering robustness.

**Key Results:** LLM embeddings significantly outperformed traditional embeddings for event clustering quality, producing more robust clusters as measured by CSAI scores. Post-clustering LLM tasks (summarization, labeling) generated meaningful and interpretable event descriptions. The CSAI metric provides a novel way to validate clustering quality beyond standard metrics like silhouette score.

**Relevance to Podders:** Validates the approach of using LLM embeddings for clustering social/news content rather than cheaper alternatives. The CSAI metric could help Podders evaluate and tune its own narrative clustering quality over time, and the pre/post LLM augmentation pattern fits naturally into a pipeline that needs both good grouping and human-readable narrative labels.

---

## Summary

These four papers converge on a common architecture for narrative and trend detection: embed documents (increasingly using LLM or neural embeddings), cluster them (with growing emphasis on online/streaming methods over batch), detect temporal changes or emerging signals, and use LLMs to interpret and label the results. BERTrend's weak-signal classification and popularity metrics offer the most directly applicable framework for Podders' "emerging narrative" detection needs. The DenStream-based online clustering from Vykhopen et al. solves the production scalability problem of processing continuous social media streams. Lange et al.'s hybrid topic-model + LLM pipeline provides a cost-efficient pattern for large-scale monitoring, and Tarekegn's work confirms that LLM embeddings meaningfully improve clustering quality for event detection. For Podders, the recommended approach would combine online neural topic modeling (BERTrend-style signal classification) with streaming density-based clustering (DenStream) and LLM-powered interpretation, keeping LLM calls surgical and downstream to control costs.
