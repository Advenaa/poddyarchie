# Decision 11: Translate Indonesian Chat Queries Before Search

## Decision
Option B — Detect language of chat input. If Indonesian, translate to English with one Haiku call, then search English embeddings with English query.

## Why This Decision
- All summaries and embeddings are in English (after Decision 1 translates Indonesian source content)
- English query ↔ English embeddings = best cosine similarity
- Indonesian query ↔ English embeddings = degraded similarity
- Cross-lingual embeddings (Option C) would degrade English↔English quality (95% of queries) to slightly improve Indonesian↔English (5% of queries)
- Same pattern as Decision 1: translate to English first, do all work in English
- Our users are bilingual — they'll mostly type English, but should handle Indonesian gracefully
- Cost: maybe 1-2 Indonesian queries per day

## Research Papers
- Cross-Lingual IR (Oct 2025) — arxiv.org/abs/2510.00908 — Multilingual embeddings bridge language gaps
- Translation Strategies (Jul 2025) — arxiv.org/abs/2507.22923 — Optimal strategy varies by task, sometimes translate query is best
- Left Behind (Feb 2026) — arxiv.org/abs/2603.21036 — 14-17pp penalty on low-resource language processing

## Implementation

### In Chat Handler (before search)
```typescript
async function handleChat(userMessage: string, history: ChatMessage[]): Promise<string> {
  const lang = detectLanguage(userMessage); // franc, already in pipeline
  
  if (lang === 'ind') {
    userMessage = await llm.call({
      model: config.models.haiku,
      system: 'Translate Indonesian to English. Preserve entity names and technical terms. Output only the translation.',
      prompt: userMessage,
      maxTokens: 200
    }).then(r => r.text);
  }
  
  // Continue with normal chat flow (Decision 8: tool-based retrieval)
  // ...
}
```

### Cost Impact
- Maybe 1-2 Indonesian queries/day × one Haiku call = <$0.01/day
- Negligible

## Upgrade Path
- **v2:** If Indonesian chat volume grows significantly (>20 queries/day), switch to a dedicated translation model (NLLB-200) for lower latency and cost.
- **v3:** Cross-lingual embeddings — if we add more languages beyond Indonesian and English (e.g. Malay, Thai), cross-lingual search becomes more valuable than per-language translation.
