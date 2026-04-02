# LLM Structured Output & Schema Enforcement

Research on making LLMs produce valid, schema-conforming JSON.

---

## [PARSE: LLM Driven Schema Optimization for Reliable Entity Extraction](https://arxiv.org/abs/2510.08623)

**Authors:** Anubhav Shrimal, Aryan Jain, Soumyajit Chowdhury, Promod Yenigalla
**Year:** 2025 (EMNLP 2025 Industry Track)

**Core method:** PARSE treats JSON schemas not as static contracts but as natural language understanding contracts that LLMs can interpret and improve. It has two components: ARCHITECT autonomously optimizes JSON schemas to be more LLM-friendly while maintaining backward compatibility, and SCOPE implements reflection-based extraction with combined static and LLM-based guardrails for validation.

**Key results:** Up to 64.7% improvement in extraction accuracy on the SWDE benchmark, with combined framework improvements reaching 10% across models. Extraction errors are reduced by 92% within the first retry while maintaining practical latency. Evaluated on Schema-Guided Dialogue (SGD), SWDE, and internal retail conversation data.

**Relevance to Podders:** Directly applicable -- Podders extracts entities and sentiment from unstructured social/news text using zod schemas. PARSE's insight that optimizing the schema itself (not just the prompt) improves extraction quality suggests Podders could auto-refine its zod schemas for better LLM adherence rather than relying solely on retry logic.

---

## [JSONSchemaBench: A Rigorous Benchmark of Structured Outputs for Language Models](https://arxiv.org/abs/2501.10868)

**Authors:** Saibo Geng, Hudson Cooper, Michal Moskal, Samuel Jenkins, Julian Berman, Nathan Ranchin, Robert West, Eric Horvitz, Harsha Nori
**Year:** 2025

**Core method:** Introduces an evaluation framework assessing constrained decoding approaches across three dimensions: efficiency in generating constraint-compliant outputs, coverage of diverse constraint types, and quality of generated outputs. The benchmark comprises 10K real-world JSON schemas with varying complexity, paired with the official JSON Schema Test Suite.

**Key results:** Evaluates six state-of-the-art constrained decoding frameworks (Guidance, Outlines, Llamacpp, XGrammar, OpenAI, Gemini) and reveals significant gaps in how well current frameworks handle real-world schema complexity. Provides actionable insights into capabilities and limitations of constrained decoding for structured generation.

**Relevance to Podders:** Useful as a reference benchmark for evaluating how well Podders' current LLM provider (via zod-validated JSON) handles complex schemas. If Podders ever moves to self-hosted models, the constrained decoding frameworks benchmarked here (Outlines, XGrammar, etc.) could enforce schema compliance at the decoding level rather than post-hoc validation.

---

## [Think Inside the JSON: Reinforcement Strategy for Strict LLM Schema Adherence](https://arxiv.org/abs/2502.14905)

**Authors:** Bhavik Agarwal, Ishan Joshi, Viktoria Rojkova
**Year:** 2025

**Core method:** Uses reinforcement learning (building on DeepSeek R1's GRPO framework) to train a small 1.5B parameter model for structured reasoning about JSON schemas. The pipeline combines synthetic reasoning dataset construction (20K samples for RL, 10K for SFT) with custom reward functions that penalize schema violations. The model learns to "think" about the schema before generating output.

**Key results:** Despite training a model 450x smaller than DeepSeek R1 (671B), ThinkJSON demonstrates competitive schema adherence performance against R1, its distilled variants (Qwen-1.5B, Qwen-7B), and Gemini 2.0 Flash. Training is resource-efficient: ~20 hours on 8xH100 for GRPO and ~3 hours on 1xA100 for SFT.

**Relevance to Podders:** Demonstrates that even small models can achieve strong schema adherence through RL training. If Podders needs a cost-efficient local model for high-volume entity extraction (e.g., processing thousands of Discord messages), a ThinkJSON-style fine-tuned small model could replace expensive API calls while maintaining schema compliance.

---

## [Learning to Generate Structured Output with Schema Reinforcement Learning](https://arxiv.org/abs/2502.18878)

**Authors:** Yaxi Lu, Haolun Li, Xin Cong, Zhong Zhang, Yesai Wu, Yankai Lin, Zhiyuan Liu, Fangming Liu, Maosong Sun
**Year:** 2025

**Core method:** Proposes SchemaBench (40K JSON schemas) and a Fine-grained Schema Validator that provides granular error signals for RL training. Instead of binary valid/invalid rewards, the validator decomposes schema violations into specific error types (structure understanding, escaping, natural language description adherence), enabling more targeted reinforcement learning.

**Key results:** Finds that even the latest LLMs still struggle to generate valid JSON strings, particularly with escaping and complex nested structures. The fine-grained RL approach significantly improves both JSON generation validity and downstream task performance, outperforming coarse-grained reward approaches.

**Relevance to Podders:** The fine-grained error taxonomy (structure, escaping, description) maps well to the types of zod validation failures Podders likely encounters. Podders could implement a similar granular error feedback loop: instead of just retrying on zod failure, parse the specific validation error and include it in the retry prompt for targeted correction.

---

## Summary

The field is converging on three complementary approaches to schema adherence:

1. **Schema optimization** (PARSE) -- Make the schema itself more LLM-friendly rather than treating it as immutable. Podders could auto-generate schema descriptions and field-level examples in its zod definitions.

2. **Constrained decoding** (JSONSchemaBench) -- Enforce validity at the token level during generation. Relevant if Podders moves to self-hosted models; less applicable with API-based providers that already offer structured output modes.

3. **RL-based training** (ThinkJSON, Schema RL) -- Train models to internalize schema understanding through reinforcement learning with fine-grained validators. The key insight for Podders is that granular error feedback (not just "invalid") dramatically improves retry success rates -- this can be applied at the prompt level even without model training.

**Immediate actionable takeaway for Podders:** When zod validation fails, parse the specific zod error path and message, then include it verbatim in the retry prompt. This mimics the fine-grained feedback loop that RL papers show is far more effective than simple retry.
