# LLM Wrapper Reference

All LLM calls in Podders go through `src/llm.ts`. Never import `@mariozechner/pi-ai` directly.

Under the hood, `llm.ts` delegates to Pi's `complete()` function for all LLM calls. Pi handles multi-provider support, token counting, cost tracking per call, tool calling, and provider-level retries. `llm.ts` handles prompt injection defense (nonce XML wrapping), chunk-split retry on context overflow, and stage-specific error decisions.

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

Model IDs live in `config.ts` as Pi model identifiers. Never hardcode model strings at call sites. Models can be swapped to any Pi-supported provider (Anthropic, OpenAI, Google, Groq, Mistral, etc.) by changing the config string.

| Stage | Model | Default ID (Pi format) |
|-------|-------|------------|
| Stage 1 (summarize) | Haiku | `claude-haiku-4-5-20251001` |
| Stage 3 (synthesize) | Sonnet | `claude-sonnet-4-6-20250514` |
| Urgency classify | Haiku | `claude-haiku-4-5-20251001` |
| Market pulse | Sonnet | `claude-sonnet-4-6-20250514` |
| Chat (RAG) | Sonnet | `claude-sonnet-4-6-20250514` |

## Pi AI Provider Configuration

All LLM calls go through `@mariozechner/pi-ai` (Pi), which abstracts multi-provider support behind a unified interface.

### Model Identifiers

Models are configured in `config.ts` using Pi's `provider:model` format:

```
anthropic:claude-haiku-4-5-20251001
anthropic:claude-sonnet-4-6-20250514
openai:gpt-4o-mini
groq:llama-3.1-70b-versatile
```

The provider prefix tells Pi which API to call. The model suffix is the provider's native model ID. When no prefix is given, Pi defaults to Anthropic.

### Switching Providers

To switch providers:

1. Change the model identifier in `config.ts` (e.g. `anthropic:claude-haiku-4-5-20251001` to `openai:gpt-4o-mini`).
2. Add the provider's API key to `.env` (e.g. `OPENAI_API_KEY=sk-...`).
3. No code changes needed -- prompt format differences are handled by Pi internally. Pi translates system/user/assistant messages to each provider's native format.

### Cost Tracking Across Providers

Pi's `response.usage.cost.total` returns the USD cost for every call, regardless of provider. No manual price tables needed in Podders -- Pi stays current with each provider's published pricing.

### Tested / Recommended Providers

| Provider | Use Case | Model |
|----------|----------|-------|
| Anthropic | Stage 1 (summarize, classify) | Claude Haiku |
| Anthropic | Stage 3 (synthesis), pulse, chat | Claude Sonnet |

Other Pi-supported providers (OpenAI, Google, Groq, Mistral) work but have not been regression-tested against the prompt suite. If switching, run `npm run test:prompts` to verify output quality.

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

**Post-call actual** (from Pi response, for cost logging):

```typescript
const response = await complete(model, { /* ... */ });
const inputTokens = response.usage.input;     // actual from Pi
const outputTokens = response.usage.output;    // actual from Pi
const costUsd = response.usage.cost.total;     // Pi computes cost per call
```

The wrapper always logs actual token counts and cost from the Pi response, never estimates.

## Cost Calculation

Cost tracking is handled by Pi. Each `complete()` response includes `response.usage.cost.total` with the USD cost for that call, computed from the provider's published pricing. No manual price table needed in Podders — Pi stays up to date across all supported providers.

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
- `input_tokens` / `output_tokens` — from Pi `response.usage` object
- `cost_usd` — from Pi `response.usage.cost.total`
- `created_at` — epoch ms

Daily cost total is exposed via `GET /api/v1/status` (`llmCostToday` field).

### Expected Daily Cost (~5,000 items/day)

| Stage | Calls/day | Model | Cost | Breakdown |
|-------|-----------|-------|------|-----------|
| Summarize (Stage 1) | ~50 | Haiku | ~$0.15 | 50 × 6K input × $0.25/M + 50 × 1K output × $1.25/M |
| Pulse | 8 | Sonnet | ~$0.11 | 8 × 2K input × $3/M + 8 × 500 output × $15/M |
| Synthesize (daily) | 1 | Sonnet | ~$0.05 | |
| Synthesize (flash) | rare | Sonnet | ~$0.05 | |
| Chat (RAG) | ~10-20 | Sonnet | ~$0.15-0.30 | Depends on usage |
| Embeddings | ~500-800 | Gemini | $0 | Free tier |
| twitterapi.io | — | — | ~$0.30 | $9/month |
| **Total** | | | **~$0.81-0.96/day (~$25-29/month)** | |

> **Gemini free tier**: allows 1,500 requests/day. At 30 Discord channels + Twitter + RSS, expect ~500-800 embedding requests/day. Headroom exists but monitor usage — paid tier is required if exceeded.

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
