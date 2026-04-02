# Multi-Agent Financial / Crypto Systems

Research on multiple AI agents collaborating on market analysis.

---

## [P1GPT: A Multi-Agent LLM Workflow Module for Multi-Modal Financial Information Analysis](https://arxiv.org/abs/2510.23032)

- **Authors:** Chen-Che Lu, Yun-Cheng Chou, Teng-Ruei Chen
- **Year:** 2025
- **Core method:** P1GPT is a layered multi-agent LLM framework that implements a structured reasoning pipeline -- not role-play simulation -- to fuse technical, fundamental, and news-based data modalities. Specialized agents communicate through coordinated workflows and perform integration-time synthesis to produce interpretable trading decisions. The key insight is that structured reasoning pipelines outperform loosely connected "team of analysts" role-play approaches.
- **Key results:** Backtesting on U.S. equities shows superior cumulative and risk-adjusted returns with low drawdowns compared to single-agent and role-simulation baselines. The system provides transparent causal rationales for each decision, addressing the explainability gap in financial AI.
- **Relevance to Podders:** Directly validates Podders' pipeline architecture (cheap model for data processing stages, expensive model for synthesis). The layered workflow design -- where each agent handles a specific data modality before synthesis -- maps closely to Podders' multi-stage processing approach.

---

## [LLM-Powered Multi-Agent System for Automated Crypto Portfolio Management](https://arxiv.org/abs/2501.00826)

- **Authors:** Yichen Luo, Yebo Feng, Jiahua Xu, Paolo Tasca, Yang Liu
- **Year:** 2025
- **Core method:** A multi-modal, multi-agent framework for crypto investment covering the top 30 cryptocurrencies by market cap. Uses an expert training module that fine-tunes agents on historical data and investment literature, plus a multi-agent investment module for real-time decisions. Features unique intrateam collaboration (confidence-weighted prediction adjustment within teams) and interteam collaboration (information sharing across teams).
- **Key results:** Evaluated on Nov 2023 - Sep 2024 data, the framework outperforms single-agent models and market benchmarks across classification accuracy, asset pricing, portfolio returns, and explainability. The confidence-based intrateam mechanism and cross-team information sharing are key contributors to improved accuracy.
- **Relevance to Podders:** Most directly relevant paper -- crypto-specific multi-agent system with multi-modal data fusion (on-chain, social, news, price). The intrateam/interteam collaboration pattern could inform how Podders structures its cheap-model processing teams (e.g., Discord scraping team, news team) before expensive-model synthesis.

---

## [TradingAgents: Multi-Agents LLM Financial Trading Framework](https://arxiv.org/abs/2412.20138)

- **Authors:** Yijia Xiao, Edward Sun, Di Luo, Wei Wang
- **Year:** 2024
- **Core method:** A trading framework modeled after real-world trading firms, with LLM agents in specialized roles: fundamental analysts, sentiment analysts, technical analysts, and traders with varied risk profiles. Includes Bull and Bear researcher agents that debate market conditions, a risk management team monitoring exposure, and traders who synthesize insights from debates and historical data. Open-source at github.com/TauricResearch/TradingAgents.
- **Key results:** Outperforms baseline models on cumulative returns, Sharpe ratio, and maximum drawdown. The adversarial Bull/Bear debate mechanism proves effective at surfacing balanced perspectives before trade decisions are made. Accepted as oral presentation at "Multi-Agent AI in the Real World" workshop.
- **Relevance to Podders:** The Bull/Bear debate pattern is a concrete mechanism Podders could adopt at the synthesis stage -- have the expensive model generate competing bull and bear cases for each condition/signal before making a final assessment. Open-source code available for reference.

---

## [When Agents Trade: Live Multi-Market Trading Benchmark for LLM Agents](https://arxiv.org/abs/2510.11695)

- **Authors:** Lingfei Qian, Xueqing Peng, Yan Wang, Vincent Jim Zhang, Huan He, Hanley Smith, Yi Han, Yueru He, Haohang Li, Yupeng Cao, Yangyang Yu, Alejandro Lopez-Lira, Peng Lu, Jian-Yun Nie, Guojun Xiong, Jimin Huang, Sophia Ananiadou
- **Year:** 2025
- **Core method:** Introduces Agent Market Arena (AMA), the first lifelong, real-time benchmark for evaluating LLM trading agents across crypto and stock markets. Tests four agent architectures (InvestorAgent, TradeAgent, HedgeFundAgent, DeepFundAgent with memory-based reasoning) across five LLM backbones (GPT-4o, GPT-4.1, Claude-3.5-haiku, Claude-sonnet-4, Gemini-2.0-flash). Uses verified trading data and expert-checked news.
- **Key results:** The critical finding is that agent framework/architecture matters far more than the underlying LLM model. Different agent architectures produce markedly distinct behavioral patterns (aggressive vs conservative), while swapping the LLM backbone contributes relatively little to outcome variation. This suggests that pipeline design is the primary lever for performance.
- **Relevance to Podders:** Validates the Podders design philosophy that pipeline architecture matters more than which specific LLM you use. Supports the cheap-model/expensive-model split -- if the framework dominates outcomes over model choice, then using cheaper models for data processing stages is well-justified. Also covers both crypto and stock markets, matching Podders' multi-market scope.

---

## Summary

These four papers collectively validate a multi-agent pipeline approach for financial/crypto intelligence systems. Three key takeaways for Podders:

1. **Pipeline architecture dominates model choice** (AMA benchmark). This directly supports Podders' strategy of using cheap models for data processing and reserving expensive models for synthesis -- the framework matters more than the model.

2. **Structured reasoning pipelines beat role-play simulation** (P1GPT). Rather than having agents "pretend" to be analysts, structured data-processing stages with explicit handoffs produce better results -- exactly what Podders' multi-stage pipeline does.

3. **Crypto-specific multi-modal fusion works** (Luo et al.). Combining on-chain data, social sentiment, news, and price data through specialized agent teams with confidence-weighted collaboration outperforms single-agent approaches, validating the multi-source scraping approach Podders is building.

4. **Bull/Bear debate as synthesis mechanism** (TradingAgents). Having adversarial agents argue opposing cases before a final decision improves signal quality -- a pattern worth adopting at Podders' expensive-model synthesis stage.
