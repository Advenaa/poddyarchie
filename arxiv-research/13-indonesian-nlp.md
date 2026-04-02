# Indonesian NLP & Multilingual Sentiment

Research on processing Bahasa Indonesia text, especially social media with slang and code-mixing.

---

## [IndoBERT-Relevancy: A Context-Conditioned Relevancy Classifier for Indonesian Text](https://arxiv.org/abs/2603.26095)

- **Authors:** Muhammad Apriandito Arya Saputra, Andry Alamsyah, Dian Puteri Ramadhani, Thomhert Suprapto Siadari, Hanif Fakhrurroja
- **Year:** 2026
- **Core method:** Builds a context-conditioned relevancy classifier on top of IndoBERT Large (335M params), trained on 31,360 labeled topic-text pairs across 188 topics. Uses an iterative, failure-driven data construction process where targeted synthetic data patches specific model weaknesses -- demonstrating that no single data source suffices for robust relevancy classification.
- **Key results:** Achieves F1 of 0.948 and 96.5% accuracy on relevancy classification. Handles both formal and informal Indonesian text. Model is publicly available on HuggingFace.
- **Relevance to Podders:** Directly applicable as a pre-filter stage -- could classify whether scraped Discord/Twitter posts are relevant to a given market topic before feeding them into the summarization pipeline, reducing noise from off-topic chatter.

---

## [NusaX: Multilingual Parallel Sentiment Dataset for 10 Indonesian Local Languages](https://arxiv.org/abs/2205.15960)

- **Authors:** Genta Indra Winata, Alham Fikri Aji, Samuel Cahyawijaya, Rahmad Mahendra, Fajri Koto, Ade Romadhony, Kemal Kurniawan, David Moeljadi, Radityo Eko Prasojo, Pascale Fung, Timothy Baldwin, Jey Han Lau, Rico Sennrich, Sebastian Ruder
- **Year:** 2022 (EACL 2023)
- **Core method:** Creates the first parallel sentiment dataset spanning 10 low-resource Indonesian local languages (Acehnese, Balinese, Banjarese, Buginese, Javanese, Madurese, Minangkabau, Ngaju, Sundanese, Toba Batak) plus Indonesian and English. Provides datasets, a multi-task benchmark, and lexicons to enable cross-lingual NLP research.
- **Key results:** Establishes baselines for sentiment analysis and machine translation across these languages. Documents the significant challenges of working with endangered and low-resource languages in the Indonesian archipelago, including dialectal variation and limited annotator availability.
- **Relevance to Podders:** Indonesian social media users frequently mix regional languages (especially Javanese and Sundanese) into their posts. This dataset and benchmark can inform how Podders handles regional language fragments that appear in Discord/Twitter content.

---

## [Enhancing Multilingual Sentiment Analysis with Explainability for Sinhala, English, and Code-Mixed Content](https://arxiv.org/abs/2504.13545)

- **Authors:** Azmarah Rizvi, Navojith Thamindu, A.M.N.H. Adhikari, W.P.U. Senevirathna, Dharshana Kasthurirathna, Lakmini Abeywardhana
- **Year:** 2025
- **Core method:** Develops a hybrid aspect-based sentiment analysis framework for multilingual and code-mixed text (English, Sinhala, Singlish). Fine-tunes XLM-RoBERTa for low-resource/code-mixed text and BERT-base-uncased for English, integrating domain-specific lexicon correction. Uses SHAP and LIME for explainability of sentiment predictions.
- **Key results:** Achieves 92.3% accuracy (F1 0.89) on English and 88.4% on Sinhala/code-mixed content, outperforming standard transformer classifiers. The explainability layer (SHAP/LIME) reveals key sentiment drivers, improving trust and transparency in a banking domain application.
- **Relevance to Podders:** The architecture pattern of using XLM-RoBERTa for code-mixed text plus domain-specific lexicon correction is directly transferable to Indonesian-English code-mixing in crypto/market Discord channels. The explainability approach could help debug why sentiment scores skew on slang-heavy posts.

---

## [Comprehensive Study on Sentiment Analysis: From Rule-based to modern LLM based system](https://arxiv.org/abs/2409.09989)

- **Authors:** Shailja Gupta, Rajesh Ranjan, Surya Narayan Singh
- **Year:** 2024
- **Core method:** A comprehensive survey tracing the evolution of sentiment analysis from rule-based/lexicon methods through machine learning and deep learning to modern LLM-based approaches. Reviews the full taxonomy of techniques including aspect-based sentiment analysis, sarcasm detection, and multimodal sentiment analysis.
- **Key results:** Identifies key remaining challenges: handling bilingual/multilingual texts, detecting sarcasm and irony, addressing dataset biases, and domain adaptation. Notes that LLMs show strong zero-shot and few-shot sentiment capabilities but still struggle with nuanced cultural and linguistic contexts.
- **Relevance to Podders:** Provides a decision framework for choosing between fine-tuned models vs. LLM prompting for sentiment. For Podders, the survey's findings on bilingual text handling and sarcasm detection are particularly relevant given Indonesian social media's heavy English code-mixing and sarcastic tone.

---

## Summary

These four papers span the key challenges Podders faces when processing Indonesian social media content. IndoBERT-Relevancy (2026) offers a ready-made relevancy filter for Indonesian text that works on both formal and informal registers -- useful as a noise-reduction pre-filter. NusaX (2022) highlights the regional language diversity within Indonesia that will appear in Discord/Twitter scrapes, particularly Javanese and Sundanese fragments mixed into Bahasa posts. The Sinhala code-mixing paper (2025), while not Indonesian-specific, demonstrates a proven architecture pattern (XLM-RoBERTa + lexicon correction + explainability) that maps well to Indonesian-English code-mixed market content. Finally, the sentiment survey (2024) provides a broad decision framework, confirming that LLMs handle zero-shot multilingual sentiment reasonably but that fine-tuned models with domain lexicons still outperform on code-mixed and slang-heavy text -- suggesting Podders should use a hybrid approach of LLM summarization with specialized sentiment classifiers for accuracy-critical signals.
