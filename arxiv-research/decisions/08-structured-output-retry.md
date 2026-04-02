# Decision 8: Feed Zod Error Paths Back in Retry Prompts

## Decision
Option B — When zod validation fails, extract specific error messages and include them in the retry prompt so Haiku knows exactly what to fix.

## Why This Decision
- PARSE (EMNLP 2025) showed 92% error reduction by feeding specific error paths back
- Current retry is generic "try again" — AI doesn't know what was wrong
- This is a zero-cost improvement — same retry call, just better prompt
- JSONSchemaBench (2025) confirmed no constrained decoding achieves 100% — validation + targeted retry is still necessary
- SO-Bench (Nov 2025) found over-constraining degrades content quality by 15-30% — validate after, don't constrain during

## Research Papers
- PARSE (Oct 2025, EMNLP) — arxiv.org/abs/2510.08623 — 92% error reduction with error path feedback
- Think Inside the JSON (Feb 2025) — arxiv.org/abs/2502.14905 — Schema-specific feedback beats generic retry
- JSONSchemaBench (Jan 2025) — arxiv.org/abs/2501.10868 — No framework achieves 100% compliance
- Schema RL (Feb 2026) — arxiv.org/abs/2502.18878 — Fine-grained validator signals improve adherence
- Failure-Aware LLM (Feb 2026) — arxiv.org/abs/2602.02896 — Different error types need different correction strategies
- Localizing Errors (Feb 2026) — arxiv.org/abs/2602.00276 — Pinpoint WHERE the error is, not just THAT there's an error
- SO-Bench (Nov 2025) — arxiv.org/abs/2511.21750 — Over-constraining degrades quality 15-30%
- JSON Whisperer (Oct 2025) — arxiv.org/abs/2510.04717 — Generate diff patches instead of full regeneration

## Implementation

### Current Retry (generic)
```typescript
// Bad — Haiku doesn't know what to fix
if (!zodResult.success) {
  const retryResponse = await llm.call({
    model: config.models.haiku,
    system: STAGE1_SYSTEM_PROMPT,
    prompt: chunk.content, // same prompt again
    maxTokens: 3000
  });
}
```

### New Retry (error-path feedback)
```typescript
if (!zodResult.success) {
  const errorMessages = zodResult.error.issues.map(issue =>
    `- ${issue.path.join('.')}: ${issue.message} (received: ${JSON.stringify(issue.received)})`
  ).join('\n');
  
  const retryResponse = await llm.call({
    model: config.models.haiku,
    system: STAGE1_SYSTEM_PROMPT,
    prompt: `Your previous output had these validation errors:
${errorMessages}

Fix ONLY these fields. Keep everything else the same.

Original input:
${chunk.content}`,
    maxTokens: 3000
  });
}
```

### Example Error Feedback
```
Your previous output had these validation errors:
- entities[12].sentiment: Expected number <= 1.0, received 5.0
- keyEvents: Expected array with max 5 items, received 8 items
- summary: Expected string with min length 50, received ""

Fix ONLY these fields. Keep everything else the same.
```

### Cost Impact
Zero. Same number of LLM calls. Just better prompt content on the retry call.

## Upgrade Path
- **v2: JSON Whisperer pattern** — Instead of asking Haiku to regenerate the full JSON, ask it to generate a diff patch: "change entities[12].sentiment from 5.0 to 0.8, remove keyEvents[5-7]." Smaller output, fewer side effects, cheaper.
- **v3: Error-type-specific strategies** — Structural errors (missing field, wrong type) → show schema again. Logical errors (sentiment out of range) → show the rule. Content errors (empty summary) → re-process with more context. Route each error type to its best fix strategy.
