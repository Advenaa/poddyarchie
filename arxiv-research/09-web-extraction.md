# Web & Information Extraction

Research on getting structured data from messy web pages and documents.

---

## [ScrapeGraphAI-100k: A Large-Scale Dataset for LLM-Based Web Information Extraction](https://arxiv.org/abs/2602.15189)

- **Authors:** William Brach, Francesco Zuppichini, Marco Vinciguerra, Lorenzo Padoan
- **Year:** 2026

- **Core method:** Introduces ScrapeGraphAI-100k, a dataset of 93,695 real-world LLM web extraction events collected from ScrapeGraphAI telemetry (filtered from 9M events). Each instance includes Markdown content, a natural-language prompt, a JSON schema defining the desired output, the LLM response, and complexity/validation metadata. The dataset is deduplicated and balanced by schema to cover diverse domains and languages.

- **Key results:** A small 1.7B-parameter model fine-tuned on a subset of this data narrows the performance gap to 30B-parameter baselines, demonstrating that targeted fine-tuning on real extraction data can make small models viable for structured web extraction. The paper also characterizes failure modes as JSON schema complexity increases -- more nested/complex schemas lead to higher extraction error rates.

- **Relevance to Podders:** Directly applicable. The dataset and methodology (prompt + JSON schema -> structured output from Markdown/HTML) maps exactly to Podders' need to extract structured entities, events, and signals from RSS feed content and news articles. Could be used to fine-tune a small local model for Podders' extraction pipeline instead of relying on large API-based LLMs.

---

## [Exploring LLMs for Scientific Information Extraction Using The SciEx Framework](https://arxiv.org/abs/2512.10004)

- **Authors:** Sha Li, Ayush Sadekar, Nathan Self, Yiqi Su, Lars Andersland, Mira Chaplin, Annabel Zhang, Hyoju Yang, James B Henderson, Krista Wigginton, Linsey Marr, T.M. Murali, Naren Ramakrishnan
- **Year:** 2025 (v2: Jan 2026)

- **Core method:** SciEx is a modular pipeline that decouples PDF parsing, multi-modal retrieval, extraction, and aggregation into composable stages. This lets operators swap in different LLMs, prompting strategies, and reasoning mechanisms without re-architecting the system. It handles long-context documents and reconciles inconsistent fine-grained information across multiple sources into standardized schemas.

- **Key results:** Evaluated across three scientific domains, SciEx reveals practical strengths and limitations of LLM-based extraction: LLMs handle schema-driven extraction well but struggle with cross-document aggregation and fine-grained consistency. The modular design makes it easy to adapt when the target ontology changes, avoiding costly re-training.

- **Relevance to Podders:** The modular architecture pattern (parse -> retrieve -> extract -> aggregate) is a strong template for Podders' own pipeline. The finding that schema changes should not require re-architecture validates Podders' need for flexible extraction ontologies as new market signals or entity types are added. The cross-document aggregation challenge is directly relevant to entity alias resolution across multiple sources.

---

## [Retrieval-Augmented Generation Based Nurse Observation Extraction](https://arxiv.org/abs/2603.26046)

- **Authors:** Kyomin Hwang, Nojun Kwak
- **Year:** 2026

- **Core method:** Uses RAG to extract structured clinical observations from unstructured nurse dictation text. The pipeline retrieves relevant examples or schema definitions to ground the LLM's extraction, ensuring outputs conform to a predefined observation taxonomy rather than hallucinating categories.

- **Key results:** Achieves F1-score of 0.796 on the MEDIQA-SYNUR test dataset, demonstrating that RAG-grounded extraction can reliably pull structured fields from noisy, informal natural language without task-specific fine-tuning.

- **Relevance to Podders:** The RAG-for-extraction pattern is directly transferable: Podders could retrieve entity definitions, known aliases, or schema examples at extraction time to improve accuracy on messy social media posts and Discord messages. The approach of grounding extraction with retrieved context (rather than relying solely on the LLM's parametric knowledge) is especially relevant for entity alias resolution.

---

## Summary

These three papers cover complementary aspects of LLM-based information extraction. ScrapeGraphAI-100k (2026) provides a large real-world dataset showing that small models can be fine-tuned for schema-driven web extraction, directly relevant to Podders' RSS/news pipeline. SciEx (2025) offers a modular architecture pattern (parse-retrieve-extract-aggregate) that handles evolving schemas and cross-document consistency -- both key Podders requirements. The MEDIQA RAG paper (2026) demonstrates that retrieval-augmented extraction improves accuracy on noisy informal text without fine-tuning, a technique applicable to Podders' social media and Discord extraction where entity alias resolution matters. Together, they suggest Podders should adopt a modular pipeline with JSON-schema-driven extraction, optionally fine-tune a small model on domain data, and use RAG to ground extraction with known entity definitions and aliases.
