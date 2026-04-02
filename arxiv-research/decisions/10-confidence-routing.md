# Decision 10: Self-Reported Confidence Gate for LLM Routing

## Decision
Option B — Add `confidence` field (1-10) to Stage 1 Haiku output schema. If confidence < 5 AND urgency != "routine", escalate chunk to Sonnet for re-processing.

## Why This Decision
- Dynamic Model Routing Survey (Mar 2026) says static deployment is obsolete — cascading is the standard
- Self-reported confidence (Mar 2026) is cheapest and best-calibrated signal — just ask the model
- Quality Gate paper (Mar 2026) validated automated PROMOTE/HOLD decisions based on schema + evidence checks
- Inter-Cascade (2025/2026) proved that using strong model occasionally improves weak model's self-assessment
- 95% of chunks are fine with Haiku. The 5% that matter (breaking events, complex multi-entity situations) get Sonnet quality
- Cost: ~2-3 escalations/day × one Sonnet call = ~$0.02-0.05/day

## Research Papers
- Dynamic Model Routing Survey (Mar 2026) — arxiv.org/abs/2603.04445 — Definitive survey. Cascading is the standard.
- Calibrating LLM Confidence (Mar 2026) — arxiv.org/abs/2603.29559 — Self-reported confidence beats sampling
- Quality Gate for LLM Apps (Mar 2026) — arxiv.org/abs/2603.15676 — Automated PROMOTE/HOLD/ROLLBACK decisions
- Inter-Cascade (Jun 2025) — arxiv.org/abs/2506.11887 — Strong model improves weak model's calibration
- CARGO (Sep 2025) — arxiv.org/abs/2509.14899 — Lightweight confidence-aware routing framework
- Confidence Tokens (Oct 2024) — arxiv.org/abs/2410.13284 — Model emits confidence alongside output
- Unified Routing and Cascading (Oct 2024) — arxiv.org/abs/2410.10347 — Theoretically optimal cascade strategy
- Pyramid MoA (Feb 2026) — arxiv.org/abs/2602.19509 — Probabilistic cost-optimized inference
- Fine-Grained Confidence (Aug 2025) — arxiv.org/abs/2508.12040 — Per-token confidence estimation

## Implementation

### Schema Change — Add confidence to Stage 1 output
```typescript
const Stage1OutputSchema = z.object({
  summary: z.string().min(50).max(2000),
  urgency: z.enum(['routine', 'elevated', 'breaking']),
  confidence: z.number().min(1).max(10), // NEW
  entities: z.array(EntitySchema).max(20),
  keyEvents: z.array(z.string()).max(5),
});
```

### Prompt Addition
Add to Stage 1 system prompt:
```
Rate your confidence in this summary from 1-10:
- 10: Clear, unambiguous content. All entities identified. Sentiment obvious.
- 5: Some uncertainty. Mixed signals, unfamiliar entities, or conflicting information.
- 1: Very uncertain. Content is confusing, contradictory, or in an unfamiliar domain.
```

### Escalation Logic
```typescript
const stage1Result = await processWithHaiku(chunk);

const shouldEscalate = stage1Result.confidence < 5
  && stage1Result.urgency !== 'routine';

if (shouldEscalate) {
  logger.info(`Escalating chunk ${chunk.id} to Sonnet (confidence: ${stage1Result.confidence}, urgency: ${stage1Result.urgency})`);
  const sonnetResult = await processWithSonnet(chunk);
  return sonnetResult;
}

return stage1Result;
```

### Why confidence < 5 AND urgency != "routine"
- Low confidence + routine = boring chunk Haiku couldn't parse well. Doesn't matter.
- High confidence + breaking = Haiku is sure about something important. Trust it.
- Low confidence + elevated/breaking = Haiku is unsure about something important. This is the danger zone. Escalate.

### Cost Impact
- ~2-3 escalations/day × Sonnet call (~$0.01 each) = $0.02-0.05/day
- Total monthly: ~$0.60-1.50
- Quality improvement on the chunks that matter most (breaking events with ambiguity)

## Upgrade Path
- **v2: Feedback loop** — Track whether Sonnet's output differed materially from Haiku's. If Sonnet agrees with Haiku on 90% of escalations, Haiku's threshold might be too conservative. Auto-tune the confidence threshold based on disagreement rate.
- **v3: CARGO-style learned routing** — Train a lightweight classifier on our actual data to predict when Haiku will fail, instead of relying on self-reported confidence. More accurate but needs training data.
