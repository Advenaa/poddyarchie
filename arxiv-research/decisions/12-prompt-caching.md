# Decision 12: Prompt Caching Verification

(Note: prompt caching was part of Decision 3, but deserves its own file for implementation details)

## Decision
Verify that Pi AI passes through provider prompt caching headers. Our Stage 1 system prompt is identical across ~50 daily Haiku calls — caching saves ~90% on those input tokens automatically.

## Why This Decision
- Anthropic enables prompt caching automatically for repeated prefixes
- Our Stage 1 system prompt (~2000 tokens) is the same for every call
- After first call, ~90% savings on system prompt tokens for calls 2-50
- Prompt Caching paper (Jan 2026) validated this across Anthropic, OpenAI, and Google
- Compression Paradox (Mar 2026) warned to measure actual savings — don't assume
- Zero code change if Pi AI passes through caching. If not, may need cache-control headers.

## Research Papers
- Prompt Caching Evaluation (Jan 2026) — arxiv.org/abs/2601.06007 — Anthropic/OpenAI/Google all offer caching
- Compression Paradox (Mar 2026) — arxiv.org/abs/2603.23528 — Measure actual cost, don't assume compression helps

## Implementation

### Verification Steps
```typescript
// 1. Make two identical Stage 1 calls
// 2. Check Pi AI response for caching indicators
const response1 = await llm.call({ system: STAGE1_PROMPT, prompt: chunk1 });
const response2 = await llm.call({ system: STAGE1_PROMPT, prompt: chunk2 });

// 3. Compare usage.input costs
console.log('Call 1 input cost:', response1.usage.cost);
console.log('Call 2 input cost:', response2.usage.cost);
// If call 2 is ~90% cheaper on input, caching is working
```

### If Pi AI Doesn't Support Caching
```typescript
// Option A: Add cache-control headers manually if the provider supports it
// Option B: Accept the cost — system prompt is ~2000 tokens × $0.25/M = $0.0005 per call
//           50 calls/day = $0.025/day. Caching saves ~$0.022/day. Not critical.
// Option C: File issue on Pi AI repo requesting cache passthrough
```

### Cost Impact
- With caching: ~$0.003/day for system prompt tokens (calls 2-50 cached)
- Without caching: ~$0.025/day for system prompt tokens
- Savings: ~$0.022/day ($0.66/month)
- Nice to have, not critical

## Upgrade Path
- Not applicable — caching is a provider feature, not an architecture decision. Either it works or it doesn't.
