# RAG Chat & Conversational Retrieval

Research on retrieval-augmented generation for question answering over custom knowledge bases.

---

## [Retrieval-Augmented Generation: A Comprehensive Survey of Architectures, Enhancements, and Robustness Frontiers](https://arxiv.org/abs/2506.00054)

- **Authors:** Chaitanya Sharma
- **Year:** 2025
- **Core method:** Provides a broad taxonomy of RAG architectures, categorizing them into retriever-centric, generator-centric, hybrid, and robustness-oriented designs. Systematically analyzes enhancements across retrieval optimization, context filtering, decoding control, and efficiency improvements. Covers comparative performance on short-form and multi-hop QA tasks.
- **Key results/findings:** Identifies recurring trade-offs between retrieval precision and generation flexibility, efficiency and faithfulness, and modularity and coordination. Highlights open challenges including adaptive retrieval architectures, real-time retrieval integration, structured reasoning over multi-hop evidence, and privacy-preserving retrieval.
- **Relevance to Podders:** Useful as a reference taxonomy for evaluating where Podders' current "retrieve top-10 by similarity, feed to Sonnet" pipeline sits and what architectural upgrades (context filtering, adaptive retrieval, multi-hop reasoning) could improve answer quality.

---

## [Adaptive Retrieval-Augmented Generation for Conversational Systems](https://arxiv.org/abs/2407.21712)

- **Authors:** Xi Wang, Procheta Sen, Ruizhe Li, Emine Yilmaz
- **Year:** 2024
- **Core method:** Introduces RAGate, a gating model that predicts per-turn whether a conversational system actually needs RAG augmentation or can answer from its own knowledge. Uses conversation context and relevance signals to make a binary retrieval-or-not decision before each response.
- **Key results/findings:** RAGate effectively identifies when retrieval is beneficial, producing higher-quality responses and higher generation confidence compared to always-retrieve baselines. Also finds a strong correlation between generation confidence and the relevance of augmented knowledge.
- **Relevance to Podders:** Directly applicable -- Podders could add a lightweight gate before retrieval so that generic follow-up questions ("thanks", "can you explain more?") skip the vector search entirely, saving latency and reducing noise from irrelevant retrievals.

---

## [Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG](https://arxiv.org/abs/2501.09136)

- **Authors:** Aditi Singh, Abul Ehtesham, Saket Kumar, Tala Talaei Khoei, Athanasios V. Vasilakos
- **Year:** 2025 (v4: April 2026)
- **Core method:** Surveys the evolution from static RAG to Agentic RAG, where autonomous AI agents are embedded into the RAG pipeline. These agents use reflection, planning, tool use, and multi-agent collaboration to dynamically manage retrieval strategies and iteratively refine context. Proposes a taxonomy based on agent cardinality, control structure, autonomy, and knowledge representation.
- **Key results/findings:** Agentic RAG enables flexibility, scalability, and context-awareness that static RAG cannot achieve. Examines applications across healthcare, finance, education, and enterprise documents. Identifies open challenges in evaluation, coordination, memory management, efficiency, and governance.
- **Relevance to Podders:** Maps out the upgrade path from Podders' current single-shot retrieval to an agent-driven system that could decompose complex market questions ("compare SOL vs ETH sentiment this week across Discord and Twitter") into sub-queries, retrieve iteratively, and synthesize across multiple sources.

---

## [Toward Faithful Retrieval-Augmented Generation with Sparse Autoencoders](https://arxiv.org/abs/2512.08892)

- **Authors:** Guangzhi Xiong, Zhenghao He, Bohan Liu, Sanchit Sinha, Aidong Zhang
- **Year:** 2025 (ICLR 2026)
- **Core method:** Uses sparse autoencoders (SAEs) to disentangle LLM internal activations and identify features specifically triggered during RAG hallucinations. Introduces RAGLens, a lightweight hallucination detector built on information-based feature selection and additive feature modeling that flags unfaithful outputs using the LLM's own internal representations.
- **Key results/findings:** RAGLens achieves superior hallucination detection compared to existing methods (both trained detectors and LLM-as-judge approaches) while being lightweight and interpretable. Provides post-hoc mitigation of unfaithful RAG and reveals new insights into where hallucination signals concentrate within LLM layers.
- **Relevance to Podders:** Addresses a core risk in Podders' chat -- the model generating claims not grounded in the retrieved summaries. RAGLens or similar lightweight faithfulness checks could be added as a post-generation verification step to flag or suppress hallucinated market claims before they reach users.

---

## [A-RAG: Scaling Agentic Retrieval-Augmented Generation via Hierarchical Retrieval Interfaces](https://arxiv.org/abs/2602.03442)

- **Authors:** Mingxuan Du, Benfeng Xu, Chiwei Zhu, Shaohan Wang, Pengyu Wang, Xiaorui Wang, Zhendong Mao
- **Year:** 2026
- **Core method:** Exposes hierarchical retrieval interfaces (keyword search, semantic search, chunk read) directly to the LLM as callable tools, letting the model autonomously decide which retrieval strategy to use at each step. This replaces both single-shot retrieval and predefined multi-step workflows with fully model-driven retrieval decisions.
- **Key results/findings:** A-RAG consistently outperforms existing RAG approaches on open-domain QA benchmarks while using comparable or fewer retrieved tokens. Performance scales with both model size and test-time compute, demonstrating that better models naturally make better retrieval decisions when given the right interfaces.
- **Relevance to Podders:** Directly actionable architecture for Podders -- instead of always running semantic search for top-10 chunks, expose keyword search (for specific token names, dates), semantic search (for thematic queries), and chunk read (for drilling into a specific report) as tools the model can call adaptively per question.

---

## Summary

These five papers trace the evolution of RAG from static retrieve-and-read pipelines to adaptive, agent-driven systems. The key takeaways for Podders' chat interface are: (1) not every query needs retrieval -- a gating mechanism (RAGate) can skip unnecessary searches; (2) when retrieval is needed, exposing multiple retrieval interfaces (keyword, semantic, chunk-level) and letting the model choose (A-RAG) outperforms fixed top-k semantic search; (3) agentic patterns with planning and reflection enable complex multi-source market questions to be decomposed and answered iteratively; (4) faithfulness remains the critical unsolved problem, and lightweight internal-representation-based detectors (RAGLens) offer a practical path to catch hallucinated market claims before they reach users; (5) there are consistent trade-offs between retrieval precision and generation flexibility that system designers must navigate based on their domain requirements.
