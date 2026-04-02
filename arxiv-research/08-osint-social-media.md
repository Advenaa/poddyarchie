# OSINT & Social Media Intelligence

Research on monitoring public channels to extract structured knowledge.

---

## [COSINT-Agent: A Knowledge-Driven Multimodal Agent for Chinese Open Source Intelligence](https://arxiv.org/abs/2503.03215)

- **Authors:** Wentao Li, Congcong Wang, Xiaoxiao Cui, Zhi Liu, Wei Guo, Lizhen Cui
- **Year:** 2025 (withdrawn Jan 2026 for revision)
- **Core method:** Combines fine-tuned multimodal LLMs (COSINT-MLLM) with an Entity-Event-Scene Knowledge Graph (EES-KG) to process multimodal OSINT data. The EES-Match framework bridges the MLLM perception layer and the structured KG, enabling systematic extraction of entities, events, and scene context from images and text. Designed specifically for the Chinese-language OSINT domain.
- **Key results:** Outperforms baselines on entity recognition, EES generation, and context matching tasks. Demonstrates that coupling an MLLM with a structured knowledge graph yields more actionable intelligence than either component alone. Paper was withdrawn for further revision, so results should be considered preliminary.
- **Relevance to Podders:** The Entity-Event-Scene knowledge graph architecture is directly applicable to Podders' goal of structuring raw Discord/Twitter streams into entities (tokens, projects, people), events (launches, exploits, rugs), and scenes (market context). The KG-augmented approach could inform how Podders builds its own structured knowledge layer on top of scraped social data.

---

## [FABULA: Intelligence Report Generation Using Retrieval-Augmented Narrative Construction](https://arxiv.org/abs/2310.13848)

- **Authors:** Priyanka Ranade, Anupam Joshi
- **Year:** 2023
- **Core method:** Uses Retrieval-Augmented Generation (RAG) over an Event Plot Graph (EPG) -- a knowledge graph that stores structured event "plot points" (who, what, when, where, why). An analyst queries the EPG to retrieve relevant plot points, which are injected into LLM prompts to generate coherent intelligence reports following a narrative structure.
- **Key results:** Generated reports show high semantic relevance of included plot points, high coherency across the narrative, and low data redundancy. The EPG structure helps close information gaps that manual report writing typically suffers from, and supports fine-grained querying beyond what keyword search offers.
- **Relevance to Podders:** Directly applicable to Podders' report generation pipeline. The Event Plot Graph pattern (structured events feeding RAG prompts) maps well to generating daily/weekly market intelligence briefs from accumulated Discord and news events. The narrative construction approach could replace simple summarization with structured, analyst-grade reports.

---

## [Evaluation of LLM Chatbots for OSINT-based Cyber Threat Awareness](https://arxiv.org/abs/2401.15127)

- **Authors:** Samaneh Shafee, Alysson Bessani, Pedro M. Ferreira
- **Year:** 2024
- **Core method:** Benchmarks seven LLM chatbots (GPT-4, GPT4all, Dolly, Alpaca, Alpaca-LoRA, Falcon, Vicuna) on two OSINT tasks using Twitter data: binary classification (is this tweet cybersecurity-relevant?) and Named Entity Recognition (extracting specific threat entities like malware names, CVEs). Compares zero-shot LLM performance against purpose-trained ML models.
- **Key results:** GPT-4 achieved F1=0.94 and open-source GPT4all achieved F1=0.90 on binary classification, making LLMs competitive with specialized classifiers for relevance filtering. However, all chatbots performed poorly on NER -- extracting specific entities like IoCs, malware families, and vulnerability IDs still requires fine-tuned or specialized models.
- **Relevance to Podders:** Validates that LLMs can reliably classify social media posts as relevant/irrelevant (useful for Podders' ingestion filtering), but warns that structured entity extraction (token names, contract addresses, specific claims) may need dedicated NER models or fine-tuning rather than relying on general-purpose LLMs alone.

---

## [SoMe: A Realistic Benchmark for LLM-based Social Media Agents](https://arxiv.org/abs/2512.14720)

- **Authors:** Dizhan Xue, Jing Cui, Shengsheng Qian, Chuanrui Hu, Changsheng Xu
- **Year:** 2025 (AAAI 2026)
- **Core method:** Introduces a benchmark of 8 social media agent tasks (content comprehension, user behavior analysis, decision-making) with 9.1M posts, 6.5K user profiles, and 25.7K reports across multiple platforms. Agents are equipped with tool-use capabilities for accessing and analyzing social media data. Evaluates both closed-source and open-source LLMs as agentic social media processors.
- **Key results:** Both closed-source and open-source LLMs perform unsatisfactorily on realistic social media agent tasks, revealing significant gaps in content comprehension, cross-platform reasoning, and complex decision-making in noisy social environments. The benchmark identifies specific failure modes in handling real-world social media data at scale.
- **Relevance to Podders:** Provides a sobering reality check on LLM agent capabilities for social media analysis tasks similar to what Podders aims to do. The 8-task taxonomy (likely including sentiment analysis, misinformation detection, trend identification) could help Podders define its own evaluation framework and identify which sub-tasks need specialized solutions vs. general LLM agents.

---

## [False Alarms, Real Damage: Adversarial Attacks Using LLM-based Models on Text-based Cyber Threat Intelligence Systems](https://arxiv.org/abs/2507.06252)

- **Authors:** Samaneh Shafee, Alysson Bessani, Pedro M. Ferreira
- **Year:** 2025
- **Core method:** Investigates three types of adversarial attacks against automated CTI pipelines that ingest OSINT from social networks, forums, and blogs: evasion (crafting text that bypasses relevance classifiers), flooding (overwhelming the system with fake-relevant content), and poisoning (corrupting training data). Uses LLM-based adversarial text generation to create realistic fake cybersecurity text that fools ML classifiers in the pipeline.
- **Key results:** Demonstrates that LLM-generated adversarial text can effectively evade CTI classifiers, degrade system performance, and disrupt the entire intelligence pipeline. Evasion attacks are shown to be the foundational attack vector that enables both flooding and poisoning. The work highlights that any OSINT system ingesting public text is inherently vulnerable to adversarial manipulation.
- **Relevance to Podders:** Critical defensive concern for Podders. Since Podders ingests public Discord and Twitter data, it is vulnerable to the same attack vectors: adversarial actors could flood channels with misleading market signals, craft posts that evade relevance filters, or poison the knowledge base. Podders should consider adversarial robustness in its classifier design and implement multi-signal validation rather than trusting any single text classification layer.

---

## Summary

These five papers map out the current landscape for building LLM-powered OSINT systems over social media -- precisely the architecture Podders targets. The most actionable takeaways are: (1) Knowledge graphs (Entity-Event-Scene or Event Plot Graphs) significantly improve structured intelligence extraction beyond raw LLM summarization (COSINT-Agent, FABULA). (2) LLMs are strong at binary relevance classification but weak at fine-grained entity extraction from social media, suggesting Podders should use hybrid approaches with dedicated NER for tokens, addresses, and specific claims (LLM Chatbots paper). (3) Current LLM agents still struggle with realistic social media tasks at scale, so Podders should set realistic expectations and build evaluation frameworks early (SoMe). (4) Any system ingesting public social data is vulnerable to adversarial manipulation via evasion, flooding, and poisoning attacks, making multi-signal validation and adversarial robustness essential design considerations (Adversarial Attacks paper).
