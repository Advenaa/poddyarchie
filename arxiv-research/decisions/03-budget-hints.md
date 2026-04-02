# Decision 3: Complexity-Adaptive Budget Hints + Prompt Caching

## Decision
Option D + complexity-adaptive — Add budget hints to Stage 1 prompts scaled by chunk density. Sparse chunks get "be concise, ~400 tokens." Dense chunks get no constraint. Also verify Pi AI passes through provider prompt caching.

## Why This Decision
- OckBench (ICLR 2026) proved LLMs over-generate 2-5x more tokens than needed on easy inputs
- Token-Budget-Aware Reasoning (2024) showed prompt-level budget hints reduce output tokens while maintaining accuracy
- Stage 1 cost is $0.15/day — batching saves $0.07/day max, not worth the complexity
- Budget hints are free — just a string in the prompt
- Prompt caching saves ~90% on repeated system prompts (Anthropic enables automatically)

## Research Papers
- Predicting LLM Output Length (ICLR 2026) — arxiv.org/abs/2602.11812 — Dynamic budget per complexity
- OckBench (ICLR 2026) — arxiv.org/abs/2511.05722 — LLMs over-generate 2-5x
- Token-Budget-Aware Reasoning (Jun 2025) — arxiv.org/abs/2412.18547 — Budget hints reduce tokens
- SelfBudgeter (May 2025) — arxiv.org/abs/2505.11274 — 61% token compression
- Prompt Caching (Jan 2026) — arxiv.org/abs/2601.06007 — 50-90% savings on repeated prefixes
- Compression Paradox (Mar 2026) — arxiv.org/abs/2603.23528 — Measure actual cost, don't assume

## Implementation

### Density Check Before LLM Call
```typescript
const isDense = chunk.estimatedEntities > 5
  || chunk.tokenCount > 4000
  || chunk.hasUrgencyKeywords;

const budgetHint = isDense
  ? '' // no constraint — let Haiku write freely
  : 'Target ~400 tokens for the summary field. Be concise.';
```

### Append to System Prompt
```typescript
const systemPrompt = STAGE1_SYSTEM_PROMPT + (budgetHint ? `\n\n${budgetHint}` : '');
```

### Prompt Caching
- Verify Pi AI passes through Anthropic's caching headers
- Our Stage 1 system prompt is identical across all ~50 daily calls
- After first call, subsequent calls pay ~90% less for the system prompt tokens
- If Pi doesn't support caching passthrough, file an issue or add cache-control headers manually

### Cost Impact
- ~70% of chunks are routine → output drops from ~1000 tokens to ~400 tokens
- Output tokens cost 5x input tokens on Haiku
- Estimated savings: ~30-40% on Stage 1 output token costs
- From ~$0.15/day to ~$0.10/day
- Prompt caching: additional ~$0.01-0.02/day savings on input tokens

## Upgrade Path
- **v2:** Dynamic maxTokens per chunk (not just budget hints). Set maxTokens: 1500 for sparse, 3000 for dense. Hard cap + soft hint together.
- **v3:** SelfBudgeter pattern — let Haiku estimate required budget in the first 50 tokens, then constrain itself. Requires prompt engineering, no fine-tuning.
