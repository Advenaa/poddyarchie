# Decision 9: Selective LLM Entity Disambiguation with Batching + Context

## Decision
Option B (upgraded) — Rule-based alias lookup first. If ambiguous, check co-occurring entities as free context. If still ambiguous, batch all ambiguous entities into one Haiku call. Save results as new aliases for future lookups.

## Why This Decision
- Rule-based handles 95% of entities (ETH, Bitcoin, OJK) instantly
- Ambiguous names like "Wormhole", "Bridge", "Luna" mean different things in different contexts
- DeepEL (Nov 2025) showed co-occurring entities resolve ambiguity for free — "Wormhole" + "bridge" + "exploit" = DeFi protocol
- In-Context Clustering (Jun 2025) showed 150% higher accuracy with 5x fewer API calls by batching
- CE-RAG4EM (Feb 2026) validated blocking-based batching for cost-efficient entity matching
- Self-improving: each disambiguation result becomes a new alias, reducing future LLM calls

## Research Papers
- KG-Enhanced Disambiguation (May 2025, updated 2026) — arxiv.org/abs/2505.02737 — Use KG hierarchy to prune candidates before LLM
- DeepEL (Nov 2025) — arxiv.org/abs/2511.14181 — Self-validation via global context from co-occurring entities
- In-Context Clustering ER (Jun 2025) — arxiv.org/abs/2506.02509 — 150% accuracy, 5x fewer API calls via batching
- Adaptive Routing for Entity Linking (Oct 2025) — arxiv.org/abs/2510.20098 — Route easy→cheap model, hard→expensive model
- CE-RAG4EM (Feb 2026) — arxiv.org/abs/2602.05708 — Blocking-based batching for entity matching
- DistillER (Feb 2026) — arxiv.org/abs/2602.05452 — Distill LLM matching into small local model
- Entity Matching with LLMs (Oct 2023) — arxiv.org/abs/2310.11244 — LLMs beat traditional by 40-68% F1
- Selective LLM ER (Jan 2024) — arxiv.org/abs/2401.03426 — Only query LLM on uncertain pairs

## Implementation

### Step 1: Rule-Based Lookup (existing)
```typescript
async function resolveEntity(name: string, context: string[]): Promise<Entity> {
  const normalized = name.toLowerCase().trim();
  const alias = await lookupAlias(normalized);
  if (alias && alias.candidates.length === 1) {
    return alias.candidates[0]; // unambiguous match
  }
  // ... continue to Step 2
}
```

### Step 2: Context-Based Disambiguation (free)
```typescript
// Use co-occurring entities from the same chunk as context
if (alias && alias.candidates.length > 1) {
  const coEntities = context; // other entity names from same chunk
  for (const candidate of alias.candidates) {
    const candidateAliases = await getEntityAliases(candidate.id);
    const contextOverlap = coEntities.filter(e =>
      candidateAliases.some(a => a.relatedEntities?.includes(e))
    );
    if (contextOverlap.length >= 2) {
      return candidate; // context strongly suggests this candidate
    }
  }
}
// ... continue to Step 3
```

### Step 3: Batched LLM Disambiguation
```typescript
// Collect all ambiguous entities from this processing cycle
const ambiguousEntities: AmbiguousEntity[] = [];

// ... after all chunks processed, disambiguate in one call
if (ambiguousEntities.length > 0) {
  const prompt = ambiguousEntities.map((e, i) =>
    `${i+1}. "${e.name}" in context: "${e.surroundingText.slice(0, 200)}"
    Candidates: ${e.candidates.map(c => `${c.name} (${c.type})`).join(', ')}`
  ).join('\n\n');

  const response = await llm.call({
    model: config.models.haiku,
    system: 'For each ambiguous entity mention, select the correct candidate based on context. Output JSON array: [{"index": 1, "selected": "candidate name"}]',
    prompt,
    maxTokens: ambiguousEntities.length * 50
  });

  // Save results as new aliases
  for (const resolution of parsed) {
    const entity = ambiguousEntities[resolution.index - 1];
    await insertAlias(entity.name, entity.contextKey, resolution.selectedId);
  }
}
```

### Self-Improvement
Each disambiguation saves a context-aware alias:
- "Wormhole" + DeFi context → Wormhole bridge protocol
- "Wormhole" + gaming context → Wormhole game
- Next time "Wormhole" appears with DeFi keywords → rule-based resolves it, no LLM call

### Cost Impact
Day 1: ~5-10 ambiguous entities → 1-2 batched Haiku calls → ~$0.01/day
Month 2+: most ambiguities resolved via cached aliases → ~0-1 Haiku calls/day

## Upgrade Path
- **v2: DistillER pattern** — After 3 months of disambiguation data, distill into a small local classifier. Zero API cost for entity matching thereafter.
- **v3: KDR-Agent pattern** — Multi-agent disambiguation with Wikipedia/CoinGecko retrieval for unknown entities. Agent retrieves facts, disambiguates, self-assesses confidence.
