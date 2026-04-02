# LLM Wrapper Reference

All LLM calls in Podders go through `src/llm.ts`. Never import `@anthropic-ai/sdk` directly.

## `llm.call()` Function Signature

```typescript
interface LLMCallParams {
  model: string;                              // from config.models (never hardcode)
  system: string;                             // system prompt
  messages: { role: 'user' | 'assistant'; content: string }[];
  maxTokens: number;                          // max output tokens
  stage: 'summarize' | 'synthesize' | 'pre-summarize' | 'urgency-classify' | 'pulse' | 'chat';
}

interface LLMCallResult {
  content: string;                            // raw text from the model
  usage: { input_tokens: number; output_tokens: number };
}

async function call(params: LLMCallParams): Promise<LLMCallResult>
```

The wrapper handles retries, error taxonomy, nonce-based sanitization, token counting,
and cost logging. Call sites never deal with the SDK or retry logic directly.

## Model Configuration

Model IDs live in `config.ts`. Never hardcode model strings at call sites.

| Stage | Model | Default ID |
|-------|-------|------------|
| Stage 1 (summarize) | Haiku | `claude-haiku-4-5-20251001` |
| Stage 3 (synthesize) | Sonnet | `claude-sonnet-4-6-20250514` |
| Urgency classify | Haiku | `claude-haiku-4-5-20251001` |
| Market pulse | Sonnet | `claude-sonnet-4-6-20250514` |
| Chat (RAG) | Sonnet | `claude-sonnet-4-6-20250514` |

## Retry Decision Tree

```
llm.call() receives response
  |
  +-- HTTP 400, context length exceeded (L1)
  |     -> split chunk in half, retry both halves (see chunk-split below)
  |
  +-- HTTP 400, other (L2: malformed request)
  |     -> log error, skip batch, surface on /settings
  |     -> NO retry
  |
  +-- HTTP 401 (L3: invalid API key)
  |     -> log FATAL, halt ALL LLM jobs immediately
  |     -> NO retry — requires manual key update + restart
  |
  +-- HTTP 429 (L4: rate limited)
  |     -> read `retry-after` header
  |     -> queue retry after that delay
  |     -> no max — follows the header
  |
  +-- HTTP 529 (L5: overloaded)
  |     -> backoff 30s flat per attempt
  |     -> retry up to 3x
  |     -> if all 3 fail: log error, surface on /settings
  |
  +-- HTTP 500/502/503 (L6: server error)
  |     -> exponential backoff: 2s, 8s, 32s
  |     -> retry up to 3x
  |     -> if all 3 fail: log error, surface on /settings
  |
  +-- Valid HTTP, invalid JSON (L7)
  |     -> retry once with FRESH prompt (see retry safety below)
  |     -> if retry also fails: skip chunk, log error
  |
  +-- Empty content / refusal (L8)
        -> log error, skip batch
        -> surface "LLM refused to process batch" on /settings
        -> NO retry
```

## Chunk-Split Retry for L1

When a chunk exceeds the model's context window (HTTP 400 context length exceeded),
the wrapper splits it and retries both halves independently.

```
L1 triggered for chunk of N items
  |
  1. Sort items by index (preserve original order)
  2. midpoint = Math.floor(N / 2)
  3. chunkA = items[0..midpoint-1]
  4. chunkB = items[midpoint..N-1]
  5. Re-estimate tokens for each half:
       tokensA = chunkA.reduce((s, i) => s + Math.ceil(i.content.length / 4), 0)
       tokensB = chunkB.reduce((s, i) => s + Math.ceil(i.content.length / 4), 0)
  6. Call llm.call() for chunkA
  7. Call llm.call() for chunkB
  8. If either half STILL triggers L1: split again (recursive)
  9. Merge results: both halves produce independent summaries
     (two summary rows, not one merged summary)
```

Each half becomes its own summary row in the `summaries` table. No merging — Stage 3
handles synthesis across all summaries regardless of how they were chunked.

## Token Counting

**Pre-call estimation** (for chunking decisions):

```typescript
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}
```

Chunk budget: 6,000 estimated tokens (~24K chars) per Stage 1 chunk.

**Post-call actual** (from SDK response, for cost logging):

```typescript
const result = await anthropic.messages.create(/* ... */);
const inputTokens = result.usage.input_tokens;    // actual from API
const outputTokens = result.usage.output_tokens;   // actual from API
```

The wrapper always logs actual token counts from the SDK response, never estimates.

## Cost Calculation

### Price Table (per million tokens)

| Model | Input | Output |
|-------|-------|--------|
| Haiku (`claude-haiku-4-5-20251001`) | $1.00 | $5.00 |
| Sonnet (`claude-sonnet-4-6-20250514`) | $3.00 | $15.00 |

### Cost Formula

```typescript
const costUsd =
  (inputTokens / 1_000_000) * pricePerMillionInput +
  (outputTokens / 1_000_000) * pricePerMillionOutput;
```

### Usage Logging

Every `llm.call()` writes a row to `llm_usage` before returning:

```sql
INSERT INTO llm_usage (id, stage, model, input_tokens, output_tokens, cost_usd, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7);
```

Fields:
- `id` — ULID
- `stage` — one of `summarize`, `synthesize`, `urgency-classify`, `pulse`, `chat`
- `model` — the actual model ID used
- `input_tokens` / `output_tokens` — from SDK `usage` object
- `cost_usd` — computed via the price table above
- `created_at` — epoch ms

Daily cost total is exposed via `GET /api/v1/status` (`llmCostToday` field).

### Expected Daily Cost (~5,000 items/day)

| Stage | Calls/day | Model | Cost |
|-------|-----------|-------|------|
| Summarize | ~30 | Haiku | ~$0.05 |
| Synthesize (daily) | 1 | Sonnet | ~$0.30 |
| Synthesize (flash) | 0-3/mo | Sonnet | ~$0.01 |
| Urgency classify | 50-100 | Haiku | ~$0.01 |
| Pulse | 8 | Sonnet | ~$0.80 |
| Chat (RAG) | ~10-20 | Sonnet | ~$0.15-0.30 |
| **Total** | | | **~$1.32-1.47/day** |

## Retry Safety

**Rule**: on retry, always use a fresh prompt. Never include failed output in retry context.

Why: an attacker can craft scraped content that causes malformed JSON on the first
attempt. If the failed response is included as an assistant turn in the retry prompt
("here is what you produced, fix it"), the injected content now appears as
model-generated text, bypassing the untrusted-data framing.

```
JSON parse fails on LLM response:
  1. Log the failed response (debug level, for post-mortem)
  2. Build a NEW prompt from scratch: system + user message, no conversation history
  3. Call llm.call() with the fresh prompt
  4. If second attempt also fails: skip this chunk, log error
  5. Never retry more than once for L7
```

No conversation history. No "fix this output" pattern. No assistant-turn injection surface.

## Nonce-Based Sanitization

Every `llm.call()` for Stage 1 wraps scraped content in a randomized XML tag that
attackers cannot predict or close.

```typescript
// 1. Generate a unique nonce per call
const nonce = crypto.randomBytes(4).toString('hex');  // e.g. "a3f7c012"

// 2. Build unpredictable delimiters
const openTag = `<scraped_content_${nonce}>`;   // <scraped_content_a3f7c012>
const closeTag = `</scraped_content_${nonce}>`;  // </scraped_content_a3f7c012>

// 3. Sanitize content before wrapping
function sanitizeForPrompt(content: string): string {
  let s = content;
  s = s.replace(/</g, '&lt;').replace(/>/g, '&gt;');          // escape angle brackets
  // Strip zero-width chars: U+200B, U+200C, U+200D, U+FEFF, U+2060
  s = s.replace(/[\u200B\u200C\u200D\uFEFF\u2060]/g, '');
  // Strip RTL/LTR overrides: U+202A-U+202E, U+2066-U+2069
  s = s.replace(/[\u202A-\u202E\u2066-\u2069]/g, '');
  // NFKC normalize (collapse Unicode lookalikes)
  s = s.normalize('NFKC');
  return s;
}

// 4. Assemble user message
const userMessage = `${openTag}\n${sanitizeForPrompt(rawContent)}\n${closeTag}`;
```

The system prompt tells the model that everything inside the nonce-tagged delimiters
is untrusted user data. Because the nonce rotates per call, an attacker who embeds
`</scraped_content>` only produces escaped text — the real closing tag cannot be guessed.

## Zod Validation After LLM Response

Call sites parse with zod schemas (defined in PIPELINE.md), not the wrapper itself:

- **`ChunkSummaryLLMSchema`** (Stage 1): `summary`, `urgency`, `entities[]` (max 20),
  `keyEvents[]` (max 5). Sentiment clamped [-1, 1]. Unknown types default to `"project"`.
- **`MarketReportLLMSchema`** (Stage 3): `tldr` (max 500 chars), `keyEvents[]` (max 10),
  `entitySentiment[]` (max 15), `sections[]` (max 4), `newProjects[]` (max 5).

On zod parse failure, the L7 retry path fires (fresh prompt, one retry, then skip).

## Error Code Quick Reference

| Code | Condition | Retries | Action |
|------|-----------|---------|--------|
| L1 | Context length exceeded (400) | Split + retry halves | Auto-recovers |
| L2 | Malformed request (400) | None | Skip batch, alert |
| L3 | Invalid API key (401) | None | Halt all LLM jobs |
| L4 | Rate limited (429) | Follows `retry-after` | Auto-recovers |
| L5 | Overloaded (529) | 3x at 30s | Auto or degrade |
| L6 | Server error (5xx) | 3x at 2s/8s/32s | Auto or degrade |
| L7 | Invalid JSON output | 1x fresh prompt | Auto or skip |
| L8 | Refusal / empty | None | Skip batch, alert |
