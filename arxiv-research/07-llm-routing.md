# LLM Routing & Model Selection

Research on when to use cheap vs expensive models.

---

## [Towards Efficient Multi-LLM Inference: Characterization and Analysis of LLM Routing and Hierarchical Techniques](https://arxiv.org/abs/2506.06579)

- **Title & Authors**: Towards Efficient Multi-LLM Inference: Characterization and Analysis of LLM Routing and Hierarchical Techniques -- Adarsh Prasad Behera, Jaya Prakash Champati, Roberto Morabito, Sasu Tarkoma, James Gross
- **Year**: 2025
- **Core method**: Survey of two complementary strategies for efficient multi-LLM inference: (i) routing, which uses a classifier or predictor to select the best model for a given query upfront, and (ii) cascading/hierarchical inference (HI), which starts with a cheap model and escalates to more expensive ones only when the response lacks confidence. Covers eight routing techniques (ZOOTER, FORC, Tryage, HybridLLM, OptLLM, MetaLLM, RouteLLM, Routoo) and six cascading systems (FrugalGPT, EcoAssistant, Cache & Distil, Automix, Efficient Hybrid Decoding, Uncertainty-Based Selection).
- **Key results/findings**: FrugalGPT achieved up to 98% cost savings by sequencing queries through cost-ranked LLMs. EcoAssistant surpassed GPT-4's success rate by 10% while reducing costs by over 50% by escalating from GPT-3.5-turbo to GPT-4. ROUTERBENCH showed hybrid routing systems can achieve 75% cost reductions while maintaining approximately 90% of GPT-4's quality. The paper proposes an Inference Efficiency Score (IES) combining quality, responsiveness, and cost.
- **Relevance to Podders**: Directly applicable -- Podders already uses a Haiku/Sonnet split, and this survey validates the cascading approach. A lightweight confidence check on Haiku outputs before escalating to Sonnet for synthesis tasks could cut costs significantly while maintaining quality on pulse reports and chat.

---

## [Integrating Large Language Models in Financial Investments and Market Analysis: A Survey](https://arxiv.org/abs/2507.01990)

- **Title & Authors**: Integrating Large Language Models in Financial Investments and Market Analysis: A Survey -- Sedigheh Mahdavi, Jiating (Kristin) Chen, Pradeep Kumar Joshi, Lina Huertas Guativa, Upmanyu Singh
- **Year**: 2025
- **Core method**: Comprehensive survey categorizing LLM applications in finance into four frameworks: (1) LLM-based Frameworks and Pipelines (MarketSenseAI, Ploutos, GPT-InvestAR), (2) Hybrid Integration combining quantitative models with LLMs, (3) Fine-Tuning and Adaptation including parameter-efficient methods and domain-specific models like FinLlama, and (4) Agent-Based Architectures with multi-agent collaborative systems (FINCON, TradingAgents). Covers stock selection, sentiment analysis, risk assessment, portfolio optimization, and trading.
- **Key results/findings**: MarketSenseAI achieved 72% cumulative returns over 15 months on S&P 100 with 10-30% excess alpha. GPT-4 consistently outperformed other models in accuracy, while Claude 3.5 Sonnet and GPT-4o showed more rational forecasting than GPT-3.5. Subjective reasoning was more effective in bull markets, while factual data led to better results in bear markets. Combining models with complementary strengths produced superior systems compared to single-model approaches.
- **Relevance to Podders**: Validates the multi-model architecture Podders uses. The finding that cheap models handle routine sentiment/summarization well while expensive models are needed for synthesis and reasoning maps directly to the Haiku-for-bulk-summarization, Sonnet-for-pulse-reports design. RAG and chain-of-thought patterns from this survey are relevant for the knowledge-building pipeline.

---

## Summary

Both papers strongly validate Podders' existing Haiku/Sonnet tiered architecture. The routing survey (Behera et al.) shows that cascading from cheap to expensive models can save 50-98% of costs while matching or exceeding single-expensive-model quality -- the key mechanism is a confidence or quality check on the cheap model's output before escalating. The finance survey (Mahdavi et al.) confirms that LLMs are effective for market sentiment analysis, news summarization, and investment signal generation, with multi-model and multi-agent systems outperforming single-model setups. For Podders, the actionable takeaway is to add a lightweight verification step on Haiku outputs (e.g., uncertainty scoring or self-consistency check) and only route to Sonnet when the task genuinely requires deeper reasoning -- rather than routing by task type alone.
