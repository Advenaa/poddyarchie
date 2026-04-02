# Financial Sentiment Analysis

Research on extracting and tracking market sentiment from text.

---

## [Event-Aware Sentiment Factors from LLM-Augmented Financial Tweets: A Transparent Framework for Interpretable Quant Trading](https://arxiv.org/abs/2508.07408)

- **Title & Authors:** Event-Aware Sentiment Factors from LLM-Augmented Financial Tweets -- Yueyi Wang, Qiyao Wei
- **Year:** 2025
- **Core method:** Uses an LLM to assign multi-label event categories (e.g., earnings, lawsuits, product launches) to high-sentiment-intensity tweets about specific companies. These labeled sentiment signals are then aligned with forward stock returns over 1-to-7-day horizons to evaluate their predictive power as alpha factors.
- **Key results/findings:** Certain event-sentiment label combinations consistently yield statistically significant negative alpha (Sharpe ratios as low as -0.38, information coefficients exceeding 0.05 at 95% confidence). The work confirms that social media sentiment is a valuable but noisy signal for financial forecasting and provides open-source reproducible code.
- **Relevance to Podders:** Directly applicable -- demonstrates how to transform social media text into structured, entity-level event-sentiment variables with measurable forward-looking signal over 1-7 day horizons, closely matching Podders' entity sentiment tracking and decay window.

---

## [A Multi-Level Sentiment Analysis Framework for Financial Texts](https://arxiv.org/abs/2504.02429)

- **Title & Authors:** A Multi-Level Sentiment Analysis Framework for Financial Texts -- Yiwei Liu, Junbo Wang, Lei Long, Xin Li, Ruiting Ma, Yuankai Wu, Xuebin Chen
- **Year:** 2025
- **Core method:** Proposes a three-tier sentiment architecture: firm-specific micro-level sentiment, industry-specific meso-level sentiment, and a duration-aware temporal smoothing layer that models the latency and persistence of textual impact. Built on both PLMs and LLMs, applied to 1.39M Chinese bond market texts (2013-2023).
- **Key results/findings:** Incorporating multi-level sentiment into credit spread forecasting reduced MAE by 3.25% and MAPE by 10.96%. Sentiment shifts closely correlated with major risk events and firm-specific crises. The duration-aware smoothing captures how textual signals decay over time.
- **Relevance to Podders:** Highly relevant -- the multi-level (entity/industry/macro) decomposition and duration-aware smoothing directly validates Podders' approach of entity-level sentiment with temporal decay. The framework's architecture could inform how to layer micro (token/project) and meso (sector) sentiment in the Indonesian/crypto context.

---

## [Adaptive Financial Sentiment Analysis for NIFTY 50 via Instruction-Tuned LLMs, RAG and Reinforcement Learning Approaches](https://arxiv.org/abs/2512.20082)

- **Title & Authors:** Adaptive Financial Sentiment Analysis for NIFTY 50 -- Chaithra, Kamesh Kadimisetty, Biju R Mohan
- **Year:** 2025
- **Core method:** Fine-tunes LLaMA 3.2 3B with instruction-based learning for sentiment classification, then augments predictions via a RAG pipeline that retrieves multi-source context using cosine-similarity embeddings. A feedback-driven module compares predicted sentiment against actual next-day stock returns to adjust source reliability, and a PPO-trained reinforcement learning agent optimizes source weighting policies over time.
- **Key results/findings:** The adaptive system significantly improved classification accuracy, F1-score, and market alignment over baselines on NIFTY 50 headlines (2024-2025). The RL-based feedback loop allows the model to self-correct by learning which information sources are more predictive of actual market moves.
- **Relevance to Podders:** The feedback loop concept (comparing sentiment predictions to actual price moves to adjust source reliability) is directly applicable to Podders' multi-source architecture. Could inform how to weight Discord vs. news vs. Twitter sources based on their historical predictive accuracy.

---

## [Bayesian Network Fusion of Large Language Models for Sentiment Analysis](https://arxiv.org/abs/2510.26484)

- **Title & Authors:** Bayesian Network Fusion of Large Language Models for Sentiment Analysis -- Rasoul Amirzadeh, Dhananjay Thiruvady, Fatemeh Shiri
- **Year:** 2025
- **Core method:** Proposes the BNLF (Bayesian Network LLM Fusion) framework, which performs late fusion of sentiment predictions from three models (FinBERT, RoBERTa, BERTweet) by modeling their outputs as probabilistic nodes in a Bayesian network. This avoids fine-tuning any single model and instead learns the probabilistic dependencies between model predictions and ground truth.
- **Key results/findings:** BNLF achieves consistent ~6% accuracy gains over individual baseline LLMs across three distinct human-annotated financial corpora. The approach is robust to dataset variability and provides interpretable fusion through explicit probabilistic reasoning.
- **Relevance to Podders:** Offers a lightweight, interpretable way to combine multiple sentiment models (or multiple LLM passes) without retraining. Podders could use a similar Bayesian fusion approach to combine sentiment scores from different source types or different prompt strategies into a single entity-level score.

---

## [Designing Heterogeneous LLM Agents for Financial Sentiment Analysis](https://arxiv.org/abs/2401.05799)

- **Title & Authors:** Designing Heterogeneous LLM Agents for Financial Sentiment Analysis -- Frank Xing
- **Year:** 2024
- **Core method:** Proposes a multi-agent framework rooted in Minsky's theory of mind and emotions, where specialized LLM agents are instantiated based on prior knowledge of common FSA error types (e.g., sarcasm, implicit sentiment, domain jargon). Agents discuss and debate classifications, and a reasoning module aggregates their outputs. No fine-tuning is required -- the approach uses prompt engineering only.
- **Key results/findings:** The multi-agent discussion framework yields better accuracy than single-model approaches on standard FSA datasets, particularly when agent discussions are substantive. The framework demonstrates that zero-shot LLMs can rival fine-tuned models when strategically orchestrated.
- **Relevance to Podders:** The multi-agent debate pattern for handling ambiguous sentiment is relevant for Podders' processing of noisy crypto/social media text where sarcasm, hype, and domain jargon are prevalent. Could be implemented as a sentiment quality layer before scoring.

---

## [Sentiment-Aware Stock Price Prediction with Transformer and LLM-Generated Formulaic Alpha](https://arxiv.org/abs/2508.04975)

- **Title & Authors:** Sentiment-Aware Stock Price Prediction with Transformer and LLM-Generated Formulaic Alpha -- Qizhao Chen, Hiroaki Kawashima
- **Year:** 2025
- **Core method:** Uses an LLM to automatically generate formulaic alpha expressions (mathematical trading signals) from structured inputs including historical price/volume data, technical indicators, and sentiment scores of both target and related companies. These generated alpha features are then fed into downstream prediction models (Transformer, LSTM, TCN, SVR, Random Forest) rather than being used directly for trading.
- **Key results/findings:** LLM-generated alpha features significantly improve stock price predictive accuracy compared to manually crafted features across all tested models. The natural language reasoning accompanying each alpha expression enhances interpretability of the predictions. Accepted at Digital Finance journal.
- **Relevance to Podders:** Demonstrates how sentiment scores can be combined with price data to generate interpretable composite features. The idea of using sentiment of related companies (not just the target) maps well to Podders tracking entity-level sentiment across interconnected crypto projects and Indonesian market sectors.

---

## Summary

These six papers collectively validate and extend Podders' core design choices for entity-level financial sentiment tracking:

1. **Temporal decay is essential.** The Multi-Level framework (Liu et al.) explicitly models sentiment persistence with duration-aware smoothing, directly supporting Podders' 5%/day relevance decay. The Event-Aware paper (Wang & Wei) confirms sentiment signals are most informative over 1-7 day horizons.

2. **Multi-level decomposition works.** Separating sentiment into entity-specific (micro), sector/industry (meso), and broader market (macro) layers improves forecasting measurably. Podders should maintain this hierarchy across its crypto token and Indonesian market entity tracking.

3. **Model fusion beats single models.** Both the Bayesian Network Fusion (BNLF) and Heterogeneous Agents approaches show that combining multiple sentiment assessments -- whether from different models or different prompting strategies -- produces more robust scores. Podders could implement lightweight Bayesian fusion or multi-agent debate for its sentiment scoring pipeline.

4. **Feedback loops improve accuracy.** The Adaptive NIFTY 50 framework demonstrates that comparing sentiment predictions against actual market outcomes and adjusting source reliability via RL produces better market alignment. This is directly applicable to Podders' multi-source (Discord, news, Twitter) architecture.

5. **Sentiment as feature, not signal.** The Formulaic Alpha paper shows sentiment is most effective when treated as an input feature to downstream models rather than a standalone trading signal. Podders' design of surfacing conditions (rather than direct trading recommendations) aligns with this finding.
