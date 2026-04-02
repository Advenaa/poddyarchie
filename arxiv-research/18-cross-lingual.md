# Cross-Lingual Summarization & Multilingual Processing

Research on processing non-English text and producing English outputs via LLMs.

---

## [Low-Resource Language Processing: An OCR-Driven Summarization and Translation Pipeline](https://arxiv.org/abs/2505.11177)

- **Title & Authors:** Low-Resource Language Processing: An OCR-Driven Summarization and Translation Pipeline -- Hrishit Madhavi, Jacob Cherian, Yuvraj Khamkar, Dhananjay Bhagat
- **Year:** 2025
- **Core method:** Presents an end-to-end pipeline that uses Tesseract OCR to extract text from image-based documents in languages like Hindi and Tamil, then feeds the extracted text through Gemini LLM APIs for cross-lingual translation, abstractive summarization, and re-translation into a target language. Additional modules perform sentiment analysis (TensorFlow), topic classification (Transformers), and date extraction (Regex) to enrich document comprehension.
- **Key results:** Demonstrates a working real-world application that chains OCR, LLM translation, and summarization to bridge language gaps in image-based media. The modular Gradio-based interface makes the pipeline accessible and shows that combining existing libraries and APIs can effectively close information access gaps across linguistic environments.
- **Relevance to Podders:** Validates the pipeline architecture of source-language extraction followed by LLM-based cross-lingual summarization -- directly analogous to Podders scraping Bahasa content and producing English summaries. The sentiment analysis and topic classification modules map to Podders' market signal detection needs.

---

## [Left Behind: Cross-Lingual Transfer as a Bridge for Low-Resource Languages in Large Language Models](https://arxiv.org/abs/2603.21036)

- **Title & Authors:** Left Behind: Cross-Lingual Transfer as a Bridge for Low-Resource Languages in Large Language Models -- Abdul-Salem Beibitkhan
- **Year:** 2026
- **Core method:** Benchmarks eight LLMs across five experimental conditions in English, Kazakh, and Mongolian using 50 hand-crafted questions spanning factual, reasoning, technical, and culturally grounded categories. Evaluates 2,000 responses on accuracy, fluency, and completeness, and tests cross-lingual transfer where models reason in English before translating back to the target language.
- **Key results:** Finds a consistent performance gap of 13.8-16.7 percentage points between English and low-resource language conditions, with models maintaining surface-level fluency while producing significantly less accurate content. Cross-lingual transfer yields selective gains for bilingual architectures (+2.2pp to +4.3pp) but provides no benefit to English-dominant models, showing that mitigation strategies are architecture-dependent rather than universal.
- **Relevance to Podders:** Directly quantifies the accuracy penalty when LLMs process low-resource languages. Since Bahasa Indonesia is mid-resource, Podders should benchmark its pipeline for accuracy degradation and consider a "reason in English" strategy, which this paper shows helps bilingual model architectures.

---

## [Language on Demand, Knowledge at Core: Composing LLMs with Encoder-Decoder Translation Models for Extensible Multilinguality](https://arxiv.org/abs/2603.17512)

- **Title & Authors:** Language on Demand, Knowledge at Core: Composing LLMs with Encoder-Decoder Translation Models for Extensible Multilinguality -- Mengyu Bu, Yang Feng
- **Year:** 2026
- **Core method:** Proposes XBridge, a compositional encoder-LLM-decoder architecture that offloads multilingual understanding and generation to external pretrained translation models (e.g., NLLB) while preserving the LLM as an English-centric core for general knowledge processing. Introduces lightweight cross-model mapping layers and an optimal transport-based alignment objective for fine-grained semantic consistency across model representations.
- **Key results:** Experiments on four LLMs across multilingual understanding, reasoning, summarization, and generation show XBridge outperforms strong baselines, especially on low-resource and previously unseen languages, without retraining the LLM. The approach enables extensible multilinguality by plugging in new translation models for new languages.
- **Relevance to Podders:** The XBridge architecture -- translate-in, reason in English, translate-out -- is a strong candidate pattern for Podders. It could use a dedicated ID-to-EN translation model on the input side and keep the summarization/reasoning LLM English-centric, avoiding quality loss from end-to-end multilingual prompting.

---

## [How and Where to Translate? The Impact of Translation Strategies in Cross-lingual LLM Prompting](https://arxiv.org/abs/2507.22923)

- **Title & Authors:** How and Where to Translate? The Impact of Translation Strategies in Cross-lingual LLM Prompting -- Aman Gupta, Yingying Zhuang, Zhou Yu, Ziji Zhang, Anurag Beniwal
- **Year:** 2025 (Accepted at Prompt Optimization KDD '25)
- **Core method:** Systematically evaluates different prompt translation strategies for classification tasks with RAG-enhanced LLMs in multilingual systems. Compares pre-translation (creating a monolingual prompt before inference) versus cross-lingual prompting (direct inference with mixed-language prompts) across multiple language pairs and RAG configurations where knowledge bases are in English but queries are in other languages.
- **Key results:** Shows that an optimized prompting strategy can significantly improve knowledge sharing across languages and downstream classification performance. Advocates for broader utilization of multilingual resource sharing and cross-lingual prompt optimization for non-English languages, especially low-resource ones, rather than defaulting to full pre-translation or fully cross-lingual approaches.
- **Relevance to Podders:** Directly applicable to Podders' RAG pipeline where the knowledge base (market intelligence, historical summaries) is in English but source content arrives in Bahasa. The findings can guide whether to translate Indonesian source text before retrieval or use cross-lingual prompting, optimizing the prompt strategy for best classification and summarization quality.

---

## [Bridging Language Gaps: Advances in Cross-Lingual Information Retrieval with Multilingual LLMs](https://arxiv.org/abs/2510.00908)

- **Title & Authors:** Bridging Language Gaps: Advances in Cross-Lingual Information Retrieval with Multilingual LLMs -- Roksana Goworek, Olivia Macmillan-Scott, Eda B. Ozyigit
- **Year:** 2025
- **Core method:** A comprehensive survey covering the evolution of cross-lingual information retrieval (CLIR) from early translation-based methods to modern embedding-based approaches and multilingual LLMs. Structures the full CLIR pipeline -- query expansion, ranking, re-ranking, and answer generation -- and examines how cross-lingual embeddings and multilingual LLMs align representations across languages for retrieval.
- **Key results:** Identifies that the field has shifted from translation-augmented monolingual retrieval toward unified embedding spaces and generative techniques. Highlights persistent challenges including data imbalance across languages and linguistic variation, and outlines future directions for building retrieval systems that are robust, inclusive, and adaptable across languages.
- **Relevance to Podders:** Provides the theoretical and practical foundation for Podders' cross-lingual retrieval component -- searching Indonesian source documents with English queries or vice versa. The survey's coverage of embedding-based CLIR methods can inform how Podders indexes and retrieves Bahasa content for its English-output knowledge base.

---

## Summary

These five papers collectively map the landscape of cross-lingual NLP as it applies to Podders' core challenge: ingesting Indonesian (Bahasa) content and producing English market intelligence. The key takeaway is that a compositional architecture -- using dedicated translation models for language bridging while keeping the reasoning/summarization LLM English-centric (as in XBridge) -- consistently outperforms naive end-to-end multilingual prompting. There is a measurable 14-17pp accuracy penalty when LLMs process low-resource languages directly versus English, which argues against simply prompting a single LLM with raw Bahasa text. For Podders' RAG pipeline specifically, the prompt translation strategy matters: selectively translating source content versus using cross-lingual prompts should be benchmarked per task. Finally, cross-lingual embedding approaches offer the most promising path for Podders' retrieval layer, enabling English-query search over Indonesian document stores without full translation of every document.
