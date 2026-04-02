# Decision 1: Translate Only Indonesian Before Stage 1

## Decision
Option C — Language detection gates translation. English content skips, Indonesian content gets translated to English before Stage 1 (Haiku summarization).

## Why This Decision
- Papers show 14-17 percentage point accuracy loss when LLMs process low-resource languages (Bahasa Indonesia) directly vs English input
- XBridge architecture (Mar 2026) proved translate-in → reason in English → translate-out outperforms single-pass multilingual
- Translation Strategies paper (Jul 2025) showed selective pre-translation is optimal — don't translate everything, only what needs it
- Language detection already exists in normalize step (franc library), so the gate is free
- Cost: ~$0.02-0.04/day for 15-20 extra Haiku calls on Indonesian chunks

## Research Papers
- Left Behind (Feb 2026) — arxiv.org/abs/2603.21036 — 13.8-16.7pp gap on low-resource languages across 8 LLMs
- XBridge (Mar 2026) — arxiv.org/abs/2603.17512 — Separate translation from reasoning beats single-pass
- Translation Strategies (Jul 2025) — arxiv.org/abs/2507.22923 — Selective pre-translation is optimal
- OCR Pipeline (May 2025) — arxiv.org/abs/2505.11177 — Modular translate→summarize validated on Indian languages
- IndoBERT-Relevancy (Mar 2026) — arxiv.org/abs/2603.26095 — F1 0.948 on Indonesian text classification

## Implementation

### Where in Pipeline
Normalize step (Step 2), after language detection, before saving as `ready`.

### Code Pattern
```typescript
// In normalize pipeline, after language detection
if (detectedLanguage === 'ind') {
  const translated = await llm.call({
    model: config.models.haiku,
    system: 'Translate the following Indonesian text to English. Preserve all entity names, numbers, and technical terms. Output only the translation.',
    prompt: item.content,
    maxTokens: Math.ceil(item.content.length / 4) * 1.5 // ~1.5x input for translation
  });
  item.content = translated.text;
  item.metadata.originalLanguage = 'ind';
  item.metadata.translated = true;
}
```

### Schema Changes
None. Add optional metadata fields `originalLanguage` and `translated` to items for tracking.

### Cost Impact
- ~15-20 extra Haiku calls/day for Indonesian content
- ~$0.02-0.04/day ($0.60-1.20/month)
- Total system cost increase: ~3-5%

## Upgrade Path
- **v2:** Use a dedicated translation model (e.g. NLLB-200) instead of Haiku for translation. Cheaper, faster, potentially better for Indonesian specifically.
- **v3:** IndoBERT-Relevancy as a pre-filter — check if Indonesian content is even relevant to crypto/finance before translating. Skip irrelevant Indonesian messages entirely.
