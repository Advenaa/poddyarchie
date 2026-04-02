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
  stage: 'summarize' | 'synthesize' | 'pre-summarize' | 'urgency-classify' | 'pulse' | 'chat' | 'translate' | 'escalate' | 'narrative-cluster' | 'entity-disambiguate';
  tools?: ToolDefinition[];                       // optional tool definitions (used by chat)
}

interface LLMCallResult {
  content: string;                            // raw text from the model
  usage: { input_tokens: number; output_tokens: number };
  toolCalls?: { name: string; arguments: Record<string, unknown> }[];  // populated when model invokes tools
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
| Chat (RAG) | Sonnet | `claude-sonnet-4-6-20250514` | Tool calling enabled (semantic_search, keyword_search, read_raw) |
| Translation (Indonesian→English) | Haiku | `claude-haiku-4-5-20251001` |
| Stage 1 escalation (low-confidence) | Sonnet | `claude-sonnet-4-6-20250514` |
| Narrative cluster naming | Haiku | `claude-haiku-4-5-20251001` |
| Entity disambiguation (batched) | Haiku | `claude-haiku-4-5-20251001` |

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
  +-- Valid HTTP, invalid JSON or zod validation failure (L7)
  |     -> JSON parse failure: retry once with FRESH prompt (see retry safety below)
  |     -> zod validation failure: retry once with error-path feedback (see zod section below)
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
- `stage` — one of `summarize`, `synthesize`, `pre-summarize`, `urgency-classify`, `pulse`, `chat`, `translate`, `escalate`, `narrative-cluster`, `entity-disambiguate`
- `model` — the actual model ID used
- `input_tokens` / `output_tokens` — from Pi `response.usage` object
- `cost_usd` — from Pi `response.usage.cost.total`
- `created_at` — epoch ms

Daily cost total is exposed via `GET /api/v1/status` (`llmCostToday` field).

### Expected Daily Cost (~5,000 items/day)

| Stage | Calls/day | Model | Cost | Breakdown |
|-------|-----------|-------|------|-----------|
| Summarize (Stage 1) | ~50 | Haiku | ~$0.10 | Budget hints reduce output ~30-40%. Prompt caching saves ~90% on repeated system prompt tokens.* |
| Pulse | 8 | Sonnet | ~$0.11 | 8 × 2K input × $3/M + 8 × 500 output × $15/M |
| Synthesize (daily) | 1 | Sonnet | ~$0.05 | |
| Synthesize (flash) | rare | Sonnet | ~$0.05 | |
| Chat (RAG) | ~10-20 | Sonnet | ~$0.15-0.30 | Depends on usage |
| Translation (Indonesian) | ~15-20 | Haiku | ~$0.02-0.04 | Only Indonesian content, gated by language detection |
| Stage 1 escalation | ~2-3 | Sonnet | ~$0.02-0.05 | Low-confidence + non-routine chunks re-processed by Sonnet |
| Narrative cluster naming | ~3-5 | Haiku | ~$0.01 | ~200 tokens per cluster name |
| Entity disambiguation | ~1-2 | Haiku | ~$0.01 | Batched; decreases over time as aliases are cached |
| Embeddings | ~500-800 | Gemini | $0 | Free tier |
| twitterapi.io | — | — | ~$0.30 | $9/month |
| **Total** | | | **~$0.82-1.06/day (~$25-32/month)** | |

> \* **Prompt caching**: Anthropic automatically caches repeated prompt prefixes. Our Stage 1 system prompt (~2000 tokens) is identical across all ~50 daily calls. After the first call, subsequent calls pay ~90% less for system prompt input tokens. Verify caching is active by comparing `response.usage.cost` between the first and second calls of a session.

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

**Exception — zod validation failure**: when the JSON parses successfully but fails zod schema validation, the retry prompt includes the specific zod error paths (not the failed output itself). This is safe because error paths are system-generated strings, not attacker-controlled content. See the Zod Validation section below for the error-path feedback pattern.

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

On zod parse failure, the L7 retry path fires with **error-path feedback**: extract specific zod error messages and include them in the retry prompt so the model knows exactly what to fix.

```typescript
if (!zodResult.success) {
  const errorMessages = zodResult.error.issues.map(issue =>
    `- ${issue.path.join('.')}: ${issue.message} (received: ${JSON.stringify(issue.received)})`
  ).join('\n');

  const retryResponse = await llm.call({
    model: config.models.haiku,
    system: STAGE1_SYSTEM_PROMPT,
    prompt: `Your previous output had these validation errors:\n${errorMessages}\n\nFix ONLY these fields. Keep everything else the same.\n\nOriginal input:\n${chunk.content}`,
    maxTokens: 3000
  });
}
```

This is safe because error-path strings are generated by zod, not from attacker content. The original failed output is never included in the retry prompt. If the retry also fails zod validation, skip the chunk and log the error.

## Prompt Caching

Anthropic automatically caches repeated prompt prefixes. Our Stage 1 system prompt (~2000 tokens) is identical across all ~50 daily Haiku calls. After the first call in a session, subsequent calls pay ~90% less for the cached system prompt tokens.

### How It Works

1. The first `llm.call()` with a given system prompt pays full price for all input tokens.
2. Anthropic caches the system prompt prefix on their side.
3. Calls 2-50 within the cache TTL (~5 minutes) pay ~10% of the original input cost for the cached portion.
4. Pi AI passes through provider caching headers transparently.

### Budget Hints (Decision 03)

Stage 1 appends complexity-adaptive budget hints to the system prompt for sparse chunks:

```typescript
const isDense = chunk.estimatedEntities > 5
  || chunk.tokenCount > 4000
  || chunk.hasUrgencyKeywords;

const budgetHint = isDense
  ? '' // no constraint — let Haiku write freely
  : 'Target ~400 tokens for the summary field. Be concise.';

const systemPrompt = STAGE1_SYSTEM_PROMPT + (budgetHint ? `\n\n${budgetHint}` : '');
```

~70% of chunks are routine/sparse. Budget hints reduce output from ~1000 to ~400 tokens on these chunks, saving ~30-40% on Stage 1 output token costs.

## Prompt Caching Verification

Verify caching is active by comparing costs between the first and second Stage 1 calls:

```typescript
const response1 = await llm.call({ system: STAGE1_PROMPT, prompt: chunk1 });
const response2 = await llm.call({ system: STAGE1_PROMPT, prompt: chunk2 });

console.log('Call 1 input cost:', response1.usage.cost);
console.log('Call 2 input cost:', response2.usage.cost);
// If call 2 is ~90% cheaper on input, caching is working
```

If Pi AI does not pass through caching headers, options are:
1. Add `cache-control` headers manually if the provider supports it.
2. Accept the cost — system prompt is ~2000 tokens x $0.25/M = $0.0005 per call. 50 calls/day = $0.025/day. Caching saves ~$0.022/day. Not critical.
3. File an issue on the Pi AI repo requesting cache passthrough.

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
