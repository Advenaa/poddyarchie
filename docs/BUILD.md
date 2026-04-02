# Build Plan — Podders v2

Complete implementation plan in 3 phases, 32 steps. Each step lists exact files, npm packages, verification commands, and dependencies.

---

## Overview

| Phase | Steps | What it delivers |
|-------|-------|------------------|
| **Phase 1** | 1-9 | Foundation: project compiles, DB has 17 tables, Fastify serves health endpoint, LLM + embedding wrappers ready |
| **Phase 2** | 2.1-2.13 | Pipeline: RSS→normalize→summarize→correlate→synthesize→deliver. System runs autonomously with all source types |
| **Phase 3** | 3.1-3.10 | Features: entity knowledge, embeddings, auth, dashboard, RAG chat, narrative clustering, ops infrastructure |

**Key milestones:**
- After Phase 1 Step 6: `curl localhost:3000/api/v1/health` returns `{"status":"ok"}`
- After Phase 2 Step 2.4: first Haiku call works
- After Phase 2 Step 2.8: first report appears in Discord
- After Phase 2 Step 2.9: system runs autonomously 24/7
- After Phase 3: full product with dashboard, chat, and monitoring

---

## Phase 1 — Foundation

### Step 1: Scaffolding

**Goal**: Project compiles with `npm run build`.

**Files to create**:
- `package.json` — `"type": "module"`, `"engines": { "node": ">=22" }`
- `tsconfig.json` — `strict: true`, `target: "ES2024"`, `module: "Node16"`, `moduleResolution: "Node16"`, `outDir: "dist"`, `rootDir: "src"`
- `.gitignore` — `node_modules/`, `dist/`, `.env`, `data/`, `*.log`, `dashboard/dist/`, `dashboard/node_modules/`
- `src/index.ts` — placeholder: `console.log('podders')`

**Scripts in package.json**:
```json
{
  "build": "tsc",
  "dev": "tsx watch src/index.ts -- run",
  "start": "node dist/index.js run",
  "test": "node --test",
  "migrate": "node dist/index.js migrate"
}
```

**Directories** (mkdir -p):
`src/db/`, `src/auth/`, `src/ingest/`, `src/normalize/`, `src/pre-summarize/`, `src/process/`, `src/knowledge/`, `src/deliver/`, `src/chat/`, `src/ops/`, `dashboard/src/`, `test/unit/`, `test/integration/`, `test/prompts/fixtures/`, `data/`

**npm packages**:
- Dev: `typescript`, `tsx`, `@types/node`

**Verify**:
```bash
npm run build && node dist/index.js   # prints 'podders'
```

**Depends on**: Nothing.

---

### Step 2: Config + Env Loading

**Goal**: Typed config object with all env vars validated.

**Files**: `src/config.ts`

**npm packages**: `dotenv`, `ulid`

**Config shape**:
```typescript
interface Config {
  anthropicApiKey: string;            // required, crash if missing
  geminiApiKey: string;               // required, crash if missing
  databaseUrl: string;                // required, must start with postgres(ql)://
  discordClientId: string | null;     // warn if missing (auth unavailable)
  discordClientSecret: string | null;
  adminUserIds: string[];             // comma-split
  discordTokens: string[];            // comma-split
  twitterApiKey: string | null;
  apiKey: string;                     // auto-gen 'pk_' + ulid() if missing
  sessionSecret: string;             // auto-gen crypto.randomBytes(32).toString('hex') if missing
  port: number;                       // default 3000
  dataDir: string;                    // default './data'
  publicUrl: string | null;
  alertWebhookUrl: string | null;
  models: {
    haiku: string;                    // default 'claude-haiku-4-5-20251001'
    sonnet: string;                   // default 'claude-sonnet-4-6-20250514'
  };
  secrets: string[];                  // assembled from above, for logger masking
}
```

Auto-generation: log WARNING with the generated value so user can save to `.env`.

**Verify**: `npm run build` compiles. Without `.env` → crashes with clear "DATABASE_URL is required" message.

**Depends on**: Step 1.

---

### Step 3: Logger

**Goal**: Pino logger with secret masking for all sensitive values.

**Files**: `src/logger.ts`

**npm packages**: `pino`, `pino-pretty` (dev)

**Implementation**:
- `createLogger(secrets: string[])` — pino with `redact` paths for known secret fields
- Custom serializer that replaces secret substrings with `[REDACTED]`
- Mask list: `DISCORD_TOKENS`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `TWITTERAPI_KEY`, `API_KEY`, `DISCORD_CLIENT_SECRET`, `SESSION_SECRET`, webhook URLs, `podders_session` cookie values
- Transport: `pino-pretty` in dev, raw JSON in production

**Verify**: Log a message containing a secret → output shows `[REDACTED]`.

**Depends on**: Step 2 (needs config.secrets).

---

### Step 4: Database

**Goal**: Postgres pool + migrations creating all 17 tables.

**Files**: `src/db/connection.ts`, `src/db/migrations.ts`, `src/db/queries.ts`

**npm packages**: `pg`, `@types/pg` (dev)

**`connection.ts`**: Pool from `DATABASE_URL`, `max: 10`, `idleTimeoutMillis: 30000`, `connectionTimeoutMillis: 5000`.

**`migrations.ts`**: Version-tracked migrations. Migration 1 creates all 17 tables:

1. `schema_version` — version tracking
2. `sources` — PK `(source, source_id)`, with `trust_weight`, `initial_trust_weight`, `poll_interval`
3. `source_state` — PK `(source, source_id)`, with `status`, `error_count`, `next_retry_at`
4. `source_rate_history` — PK `(source, source_id, hour_of_day, day_of_week)`
5. `items` — PK `id TEXT` (ULID), with `content_hash`, `content_anchor`, `original_language`, `translated`, `filter_reason`, `status`, `batch_id`. Generated `content_tsv` tsvector column + GIN index
6. `summaries` — PK `id TEXT` (ULID)
7. `entities` — PK `id TEXT` (ULID), with `status` (active/archived), `relevance`. UNIQUE(name, type)
8. `entity_aliases` — PK `(alias, context_key)`, FK `entity_id → entities`
9. `entity_mentions` — PK `id TEXT` (ULID), FK `entity_id → entities`, FK `summary_id → summaries`
10. `reports` — PK `id TEXT` (ULID), with `type`, `delivery_status`
11. `users` — PK `discord_id TEXT` (Discord snowflake, NOT ULID)
12. `sessions` — PK `id TEXT` (ULID), FK `discord_id → users ON DELETE CASCADE`
13. `app_config` — PK `key TEXT`. Seeds: `digest_time='09:00'`, `timezone='Asia/Jakarta'`
14. `llm_usage` — PK `id TEXT` (ULID), with `stage`, `model`, `input_tokens`, `output_tokens`, `cost_usd`
15. `health_events` — PK `id TEXT` (ULID), with `category`, `severity`, `metadata` (JSONB)
16. `narratives` — PK `id TEXT` (ULID), with `signal_strength`, `summary_ids` (TEXT[])
17. `embeddings` — PK `id TEXT` (ULID), with `target_type`, `target_id`, `vector` (BYTEA), `dimensions`. UNIQUE(target_type, target_id)

**`queries.ts`**: Skeleton with typed function signatures (getSources, insertItem, claimItems, insertSummary, insertReport, insertLlmUsage, getAppConfig, setAppConfig, insertEmbedding, insertHealthEvent). All use parameterized queries.

**Verify**:
```bash
createdb podders_test
DATABASE_URL=postgresql://localhost/podders_test node dist/index.js migrate
psql podders_test -c "\dt" | wc -l   # 17 tables
psql podders_test -c "\d embeddings" | grep vector   # BYTEA column
```

**Depends on**: Steps 1-3.

---

### Step 5: CLI Entry

**Goal**: Commander with `run` and `migrate` commands + graceful shutdown.

**Files**: `src/index.ts` (replace placeholder)

**npm packages**: `commander`

**Commands**:
- `podders run` — load config → create logger → create pool → run migrations → start server
- `podders migrate` — load config → create pool → run migrations → exit

**Graceful shutdown**: trap SIGTERM/SIGINT → set `shuttingDown = true` → close server → close pool → exit. Hard kill: 30s timeout.

**Verify**: `node dist/index.js --help` shows commands. `node dist/index.js migrate` runs migrations.

**Depends on**: Steps 1-4.

---

### Step 6: Fastify Server

**Goal**: Fastify with security headers, rate limiting, cookies, route skeletons.

**Files**: `src/server.ts`

**npm packages**: `fastify`, `@fastify/cookie`, `@fastify/rate-limit`, `@fastify/static`

**Implementation**:
- Register `@fastify/cookie` (signed with `SESSION_SECRET`)
- Register `@fastify/rate-limit` (100 req/s global)
- Security headers via `onSend` hook:
  - `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' https://cdn.discordapp.com`
  - `X-Frame-Options: DENY`
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: strict-origin-when-cross-origin`
- Health endpoint: `GET /api/v1/health` → `{"status":"ok"}`
- Skeleton routes returning `{"todo":true}` for reports, sources, config, search, feed, chat, users

**Verify**:
```bash
node dist/index.js run &
curl localhost:3000/api/v1/health           # {"status":"ok"}
curl -sI localhost:3000/api/v1/health | grep X-Frame   # DENY
```

**Depends on**: Steps 1-5.

---

### Step 7: LLM Wrapper

**Goal**: Single entry point for all LLM calls with full retry tree and prompt injection defense.

**Files**: `src/llm.ts`

**npm packages**: `@mariozechner/pi-ai`

**Exports**:
```typescript
export function createLLM(pool, logger, config) {
  return { call, estimateTokens, sanitizeForPrompt, wrapWithNonce, getBudgetHint };
}
```

**`call(params: LLMCallParams): Promise<LLMCallResult>`** — full retry decision tree:

| Level | Condition | Action |
|-------|-----------|--------|
| L1 | Context length exceeded | Throw `ContextLengthExceededError` (caller splits) |
| L2 | HTTP 400 other | Log + throw, no retry |
| L3 | HTTP 401 | Log FATAL, set `halted = true`, all future calls throw immediately |
| L4 | HTTP 429 | Respect `retry-after` header, wait, retry |
| L5 | HTTP 529 | 30s backoff, retry 3x |
| L6 | HTTP 500/502/503 | Exponential backoff 2s/8s/32s, retry 3x |
| L7 | Invalid JSON | Fresh prompt retry once (no failed output in context) |
| L8 | Empty/refusal | Log + throw, no retry |

**Stage enum**: `'summarize' | 'synthesize' | 'pre-summarize' | 'urgency-classify' | 'pulse' | 'chat' | 'translate' | 'escalate' | 'narrative-cluster' | 'entity-disambiguate'`

**Utilities**:
- `estimateTokens(text)` — `Math.ceil(text.length / 4)`
- `sanitizeForPrompt(content)` — escape `<>`, strip zero-width chars, NFKC normalize
- `wrapWithNonce(content)` — `<scraped_content_${nonce}>` wrapping with sanitization
- `getBudgetHint(chunk)` — returns conciseness instruction for sparse chunks

**Logging**: every successful call writes to `llm_usage` table (stage, model, tokens, cost).

**Verify**: `npm run build` compiles. Unit tests for `estimateTokens`, `sanitizeForPrompt`, `wrapWithNonce`.

**Depends on**: Steps 1-6.

---

### Step 8: Embeddings Wrapper

**Goal**: Gemini text-embedding-004 wrapper with retry, cost logging, BYTEA helpers.

**Files**: `src/embed.ts`

**npm packages**: `@google/generative-ai`

**Exports**:
```typescript
export function createEmbedder(config, pool, logger) {
  return { embed, embedBatch, prepareText, vectorToBytes, bytesToVector, isAvailable };
}
```

- `isAvailable()` — false if no GEMINI_API_KEY (all calls return null gracefully)
- `embed(text)` → `{ vector: Float32Array, dimensions: 768, model: 'text-embedding-004' }`
- `embedBatch(texts)` — batches of 100, Gemini batch endpoint
- `prepareText(text, type)` — strip Discord formatting, collapse whitespace, truncate to 512 tokens
- `vectorToBytes(Float32Array) → Buffer` / `bytesToVector(Buffer) → Float32Array`
- Retry: 429 → retry-after, 5xx → exponential backoff 2s/8s/32s
- Cost logging to `llm_usage` (stage: 'embedding', cost: 0 for free tier)

**Verify**: With GEMINI_API_KEY → embed "hello world" → 768-dim vector. `vectorToBytes` roundtrips.

**Depends on**: Steps 1-6.

---

### Step 9: URL Validator

**Goal**: SSRF protection for all outbound URL fetches.

**Files**: `src/url-validator.ts`

**npm packages**: None (Node built-ins: `dns`, `url`, `net`).

**Checks**:
1. Valid URL parse
2. Require `https://`
3. Reject non-standard ports (only 443)
4. DNS resolve → reject private/reserved IPs (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 169.254.0.0/16, ::1, fc00::/7, fe80::/10)
5. Reject `localhost`, `*.local`

**Verify**: Unit tests — `https://example.com` → valid, `http://example.com` → invalid, `https://localhost` → invalid, `https://10.0.0.1` → invalid.

**Depends on**: Step 1 (independent utility).

---

### Phase 1 Dependency Graph

```
Step 1 (Scaffolding)
  ├──→ Step 2 (Config) ──→ Step 3 (Logger) ──→ Step 4 (Database) ──→ Step 5 (CLI) ──→ Step 6 (Server)
  │                                                                                      ├──→ Step 7 (LLM)
  │                                                                                      └──→ Step 8 (Embeddings)
  └──→ Step 9 (URL Validator)  [independent]
```

### Phase 1 npm Dependencies

**Production**: `dotenv`, `ulid`, `pino`, `pg`, `commander`, `fastify`, `@fastify/cookie`, `@fastify/rate-limit`, `@fastify/static`, `@mariozechner/pi-ai`, `@google/generative-ai`

**Dev**: `typescript`, `tsx`, `@types/node`, `@types/pg`, `pino-pretty`

---

## Phase 2 — Core Pipeline

### Step 2.1: RSS Adapter

**Files**: `src/ingest/rss.ts`

**npm packages**: `rss-parser`, `@mozilla/readability`, `linkedom`

**Implementation**:
- `pollFeed(feedUrl)` — fetch XML, parse via rss-parser, filter newer than `source_state.last_id`
- Map to `RawItem`: `contentSnippet` over `content`, `engagement: 0`, `url: parsed.link`
- If `content.length < 500` and URL exists: extract article via Readability + linkedom
- Synthetic GUID fallback: `sha256(pubDate + title)`
- Per-feed isolation: one feed error doesn't block others
- Error handling: 5 failures in 1h → `status = 'disabled'`

**Verify**: Parse a real feed (e.g., CoinDesk) → correct `RawItem[]`. Article extraction fires for truncated snippets. SSRF rejects private IPs.

**Depends on**: Phase 1.

---

### Step 2.2: Normalize (All 6 Gates)

**Files**: `src/normalize/index.ts`, `src/normalize/spam.ts`, `src/normalize/instruct-detector.ts`, `src/normalize/url-expand.ts`

**npm packages**: `franc`

**The 6 gates** (items exit at first failure):

| # | Gate | On fail |
|---|------|---------|
| 1 | Truncate >20K chars | Continue (truncated) |
| 1.5 | NFKC normalize + strip zero-width + InstructDetector scan | `filtered` with `filter_reason: 'injection_detected'` |
| 2 | Content hash dedup: `sha256(source + sourceId + content.slice(0, 200))` | Drop silently |
| 3 | URL dedup: expand t.co/bit.ly (HTTP HEAD, 5 redirects max), check DB | Drop silently |
| 4 | Spam filter: 11 rules (3 English + 8 Indonesian patterns) | `filtered` |
| 5 | Language detect: franc → must be `eng`, `ind`, or `und` | `filtered` |
| 6 | Indonesian translation: Haiku call, stage: 'translate' | Keep original on error |

**InstructDetector**: regex-based heuristic scanner initially (pattern-match "ignore previous", "you are now", "system:", XML tag closings). Cached per content hash. Swappable interface for future ML classifier.

Pass all gates → INSERT as `ready`.

**Verify**: Golden file test suite covering all gates. `gm`, `gas!`, single emoji all filter. t.co expands correctly. Indonesian triggers translation.

**Depends on**: 2.1 (produces RawItem), Phase 1 (llm.ts for translation).

---

### Step 2.3: Pre-summarize

**Files**: `src/pre-summarize/index.ts`

**Decision tree**:

| Check | Condition | Action |
|-------|-----------|--------|
| Source type | Discord or Twitter | SKIP |
| Content length | ≤ 4000 chars | SKIP |
| Urgency keywords | `exploit`, `hack`, `rate decision`, `flash crash`, `halt`, `circuit breaker`, `emergency`, `bank run` | SKIP (pass raw) |
| Entity density | >2 entity hints per 100 tokens | SKIP (pass raw) |
| Default | Low-density RSS/news > 4000 chars | Compress via Haiku, `maxTokens: 300` per article |

**Before compression**: store first 200 tokens (~800 chars) in `items.content_anchor`.

**Batching**: 3-5 articles per Haiku call. Two prompt variants: regulatory vs general news.

**Verify**: Discord skips. Urgent articles skip. Dense articles skip. Long generic news compresses to ~300 tokens. `content_anchor` populated.

**Depends on**: 2.2 (items exist as `ready`).

---

### Step 2.4: Stage 1 — Summarize

**MILESTONE: First Haiku call works**

**Files**: `src/process/summarize.ts`, `src/process/schemas.ts`, `src/process/chunk.ts`

**Implementation**:
1. **Batch claim**: SQL marks `ready` → `processing` with batch ID
2. **Token-budgeted chunking**: 6,000 tokens per chunk, items sorted by timestamp
3. **Haiku call**: nonce-based XML wrapping, few-shot examples, engagement data
4. **Budget hints**: sparse chunks get "Target ~400 tokens" instruction
5. **Zod validation with two-path retry**:
   - JSON parse failure → fresh prompt retry (no failed output, attack #6 defense)
   - Zod failure → error-path feedback with specific field errors (92% error reduction)
6. **Entity post-verification**: drop entities not found in raw text (case-insensitive)
7. **Confidence escalation**: if `confidence < 5 AND urgency !== 'routine'` → re-run with Sonnet (stage: 'escalate', maxTokens: 3000)

**Zod schema** (`ChunkSummaryLLMSchema`):
- `summary: z.string()` (min 50 chars)
- `urgency: z.enum(['routine', 'elevated', 'breaking'])`
- `confidence: z.number().min(1).max(10)`
- `entities: z.array().max(20)` — name, aliases, type, mentionCount, sentiment (clamped -1 to 1)
- `keyEvents: z.array(z.string()).max(5)`

**Verify**: Feed 10 real RSS items through full pipeline. Haiku returns valid JSON. Entity post-verification drops fake entities. Confidence escalation triggers on crafted input.

**Depends on**: 2.2, 2.3, Phase 1 (llm.ts, entity tables).

---

### Step 2.5: Stage 2 — Correlate

**Files**: `src/process/correlate.ts`, `src/knowledge/trust.ts`

**Implementation**:
1. **Correlation query**: SQL — entities in 2+ distinct sources within 24h
2. **Flash trigger**: compute `weightedSum = sum(trust_weight)` across breaking sources. Fire if `>= 1.5` (NOT simple "2+ sources")
3. **Trust weight auto-adjustment** (1h post-flash):
   - Confirmed sources: `+0.05` (capped at `initial + 0.2`)
   - Unconfirmed: `-0.03` (floored at `initial - 0.2`)

**Verify**: Flash fires at weighted sum 1.5, not at 1.4. Trust adjustment stays within ±0.2 bounds.

**Depends on**: 2.4 (summaries + entity mentions in DB).

---

### Step 2.6: Stage 3 — Synthesize

**Files**: `src/process/synthesize.ts`, `src/process/exemplars.ts`, `src/process/dedup-events.ts`

**npm packages**: `fastest-levenshtein`

**Daily synthesis** (`runDaily()`):
- Load summaries from midnight-to-midnight (configured TZ). Skip if 0. Rank top 30 if >50.
- **Exemplar selection**: 10 slots — 7 engagement-ranked + 3 RSS/news (entity density × urgency weight)
- Deduplicate `keyEvents` across summaries (Levenshtein ≥ 0.7)
- Get narrative context from `detectNarratives()` (step 2.13)
- Get correlated entities from `correlate.run()`
- Sonnet call with causal chain instructions + yesterday's TL;DR
- Duplicate prevention: check if daily for this date already exists

**Flash synthesis** (`runFlash()`):
- Triggered by Stage 1 when weighted trust sum ≥ 1.5
- Read 4h of summaries, Sonnet call, deliver with orange embed (`0xFF6B35`)
- Schedule trust weight adjustment: `setTimeout(1h)`

**Verify**: Daily report with TL;DR, key events, entity sentiment. Exemplars include 3 RSS/news. Flash delivers with orange embed. Empty day skips.

**Depends on**: 2.4, 2.5, 2.13 (narrative clustering).

---

### Step 2.7: Pulse

**Files**: `src/process/pulse.ts`

**Implementation**:
- Cron: `0 */3 * * *` UTC (8x/day)
- **Activity-scaled output**:
  - 1-3 summaries, all routine → `maxTokens: 300` (2-3 sentences)
  - 4-10, some elevated → `maxTokens: 800` (paragraph + events)
  - 10+ or any breaking → `maxTokens: 1500` (full short report)
- **Drift detection**: flag entities where `|prior - current sentiment| > 0.4`
- Quality gate: skip if 0 summaries
- Context chaining: each pulse carries prior pulse's TL;DR
- Grey embed (`0x4A4A5A`), title `"Market Pulse — HH:MM WIB"`

**Verify**: Quiet → short. Active → full. Drift flag fires on 0.5 swing. Zero-activity skips.

**Depends on**: 2.4 (summaries), 2.8 (delivery).

---

### Step 2.8: Delivery

**MILESTONE: First report appears in Discord**

**Files**: `src/deliver/webhook.ts`

**Implementation**:
- **Embed formatting**: daily=blue (`0x5B8DEF`), flash=orange (`0xFF6B35`), pulse=grey (`0x4A4A5A`)
- **Mobile rules**: TL;DR under 280 chars, `inline: false`, max 3-4 fields
- `allowed_mentions: { parse: [] }` always (suppress @everyone)
- Truncate: description 4096, field values 1024
- **Retry**: 2s/8s/32s exponential, 3 attempts. No retry on 4xx (except 429)
- On failure: `delivery_status = 'failed'`
- "View full report" link if `PUBLIC_URL` set

**Verify**: Embed fits Discord limits (6000 total). Retry on 429. Failed delivery tracked. Colors correct.

**Depends on**: Phase 1 (config for webhook URL).

---

### Step 2.9: Scheduler

**MILESTONE: System runs autonomously**

**Files**: `src/scheduler.ts`, `src/health.ts`

**npm packages**: `node-cron`

**Startup sequence** (order matters):
1. `config.load()` → 2. `db.connect()` → 3. `migrations.run()` → 4. `crashRecovery()` (reset orphaned `processing` → `ready`) → 5. `discord.connect()` → 6. `registerCronJobs()` → 7. `server.listen()`

**Job table**:

| Job | Schedule | Mutex key |
|-----|----------|-----------|
| Source polling tick | every 60s (check which sources are due) | `source-poll` |
| Discord Poisson claim | every 60s (threshold: actual > 1.5λ or > 10) | `source-poll` |
| Market pulse | `0 */3 * * *` UTC | `pulse` |
| Daily synthesis | configurable time + TZ | `stage3-daily` |
| Entity decay | `0 0 * * *` UTC | `entity-decay` |
| Data retention | `5 0 * * *` UTC | `retention` |
| Backup | `10 0 * * *` UTC | `backup` |
| Health monitor | `*/5 * * * *` | `health-check` |
| Scraper health | `*/5 * * * *` | `scraper-health` |

**Poisson-based Discord claim gate**:
- Look up lambda from `source_rate_history` (hour_of_day × day_of_week, 7-day rolling)
- Process threshold: `actual > lambda * 1.5 OR actual > 10`
- Alert threshold: `actual > lambda * 3 OR poissonSurvival < 0.01`

**Mutex**: `Record<string, boolean>`. Skip-not-queue. Check `shuttingDown`.

**Health monitor** (7 checks):
1. Source silence (3x `poll_interval` → warn)
2. Source disabled → critical
3. LLM failures (0 calls while items ready >1h → critical)
4. Missed pulse (>4h → warn)
5. Missed daily (>26h → critical)
6. DB pool exhaustion → warn
7. Cost spike (>3x 7-day avg → warn)

Dedup: skip if same category+message unacknowledged within 30min. Critical → POST to `ALERT_WEBHOOK_URL`.

**Graceful shutdown**: `shuttingDown = true` → stop crons → wait for in-flight LLM → close Discord WS → close Fastify → close pool → exit. Hard kill 30s.

**Verify**: Crash recovery resets orphaned items. Mutex prevents overlap. Health alerts fire on critical. Graceful shutdown waits for in-flight calls.

**Depends on**: All of 2.1-2.8.

---

### Step 2.10: Discord Adapter

**Files**: `src/ingest/discord.ts`

**npm packages**: `ws`, `@types/ws` (dev)

**This is the most complex adapter:**

**Gateway lifecycle**: Connect → opcode 10 HELLO → opcode 2 IDENTIFY (browser-mimicking properties) → READY (cache session_id, resume_gateway_url) → heartbeat loop → MESSAGE_CREATE handler

**Multi-token management**:
- Sequential IDENTIFY with **5s gap** between tokens (rate limit: 1/5s/IP)
- Channel partitioning: round-robin across active tokens
- Token death on fatal close codes (4004, 4010-4014): disable permanently, reassign channels

**Reconnect**:
- Resumable (4000-4003, 4009, 1001, network drop): RESUME on resume_gateway_url
- Non-resumable (4007, 4008, 1000): fresh IDENTIFY with backoff `min(2^attempt * 2s, 60s)`
- INVALID SESSION (opcode 9, `d: false`): wait 1-5s random, fresh IDENTIFY

**MESSAGE_CREATE**: filter assigned channels + skip bots + skip non-DEFAULT/REPLY. Content = text + reply context (200 chars) + embed descriptions. Images from Discord CDN in `metadata.imageUrls`. Engagement hardcoded to 0.

**Verify**: Connect, receive READY, receive MESSAGE_CREATE. Resume works after close 4000. Token death reassigns channels. Bot messages filtered.

**Depends on**: 2.2 (normalize), 2.9 (scheduler for Poisson).

---

### Step 2.11: Twitter Adapter

**Files**: `src/ingest/twitter.ts`

**Implementation**:
- twitterapi.io REST: `x-api-key` header, 15s timeout
- `@handle` → `/twitter/user/last_tweets`. 10+ accounts → batch search: `from:user1 OR from:user2`
- Engagement: `Math.round(Math.log10(likeCount + retweetCount * 2 + quoteCount * 3) * 20)`, clamped 1-100
- Dedup by tweet ID (`BigInt` comparison with `source_state.last_id`)
- Zod validation of API response. On failure: skip, log raw response
- 401 → disable all Twitter sources (CRITICAL). 429 → backoff.

**Verify**: Poll real account → correct RawItem[]. Engagement formula: 100 likes + 50 RTs + 10 quotes ≈ 47. Dedup skips seen tweets.

**Depends on**: 2.2 (normalize).

---

### Step 2.12: News Adapter

**Files**: `src/ingest/news.ts`

**Implementation**: URL-based article extraction via Readability + linkedom. Validate URL with `url-validator.ts`. `engagement: 0`. One-shot (individual URLs, not feeds).

**Verify**: Extract real article → clean text. SSRF rejects private IPs. Graceful fallback on extraction failure.

**Depends on**: 2.2 (normalize).

---

### Step 2.13: Narrative Clustering

**Files**: `src/process/narratives.ts`

**npm packages**: `ml-kmeans`

**Implementation**:
1. Load summary embeddings from previous calendar day (from vector cache)
2. If < 9 summaries, skip
3. Auto-tune k (3 to min(10, floor(count/3))) using silhouette score
4. Filter clusters with 3+ members
5. Name each cluster via Haiku (`maxTokens: 20`, stage: 'narrative-cluster')
6. **Signal strength** by growth rate vs prior day:
   - Match days by centroid cosine similarity
   - `≥ 3.0x` → strong, `≥ 1.5x` → emerging, `≤ 0.5x` → fading, else → stable, no match → new
7. Insert into `narratives` table
8. Return for injection into Stage 3 Sonnet prompt

**Verify**: 30 embeddings → 3-10 clusters named. 3x growth = strong. New cluster = new. Clusters < 3 members excluded.

**Depends on**: Phase 1 (embed.ts), 2.4 (summaries with embeddings).

---

### Phase 2 Dependency Graph

```
2.1 RSS ──→ 2.2 Normalize ──→ 2.3 Pre-summarize ──→ 2.4 Stage 1 ──→ 2.5 Correlate
                │                                         │               │
                ├──→ 2.10 Discord ──────────────────────→ │          2.13 Narratives
                ├──→ 2.11 Twitter ──────────────────────→ │               │
                └──→ 2.12 News ─────────────────────────→ │               │
                                                          └───→ 2.6 Stage 3 ←──┘
                                                                │
                                                          2.7 Pulse ──→ 2.8 Delivery
                                                                              │
                                                                         2.9 Scheduler
```

Steps 2.10, 2.11, 2.12 are parallelizable. 2.13 can start after 2.4.

### Phase 2 npm Dependencies

`rss-parser`, `@mozilla/readability`, `linkedom`, `franc`, `fastest-levenshtein`, `ws`, `@types/ws` (dev), `ml-kmeans`, `node-cron`

---

## Phase 3 — Features

### Parallel Tracks

```
Track A: 3.1 (entities) ─────────────────────────────────────────────────
Track B: 3.2 (embeddings) ──→ 3.6 (chat backend) ──→ 3.7 (chat page)
Track C: 3.3 (auth) ──→ 3.4 (dashboard scaffold) ──→ 3.5 (pages)
Track D: 3.8 (narratives, after 3.2) ────────────────────────────────────
Track E: 3.9 (ops) ──────────────────────────── anytime
Track F: 3.10 (testing) ─────────────────────── anytime
```

---

### Step 3.1: Entity Knowledge (Track A)

**Files**: `src/knowledge/entities.ts`, `src/knowledge/decay.ts`, `src/knowledge/seed.ts`

**All 3 disambiguation tiers**:
- **Tier 1**: Rule-based alias lookup in `entity_aliases`. Handles ~95% of cases.
- **Tier 2**: Context-based — find co-occurring entities via `entity_mentions` JOIN on `summary_id`. Free, no LLM.
- **Tier 3**: Batched Haiku call for remaining ambiguous entities. Save results as context-aware aliases (self-improving).

**CoinGecko seeding**: `GET /api/v3/coins/list` → create entities + aliases. Re-seed weekly.

**Indonesian seeds**: OJK, Bappebti, Bank Indonesia, BEI/IDX, Indodax, Tokocrypto, Pintu, IDR/Rupiah.

**Decay**: daily `relevance *= 0.95`. Archive below 0.01 after 90 days (UPDATE, not DELETE). Reactivate on re-mention with `relevance = 0.5`.

**Source weights for relevance**: Discord=1.0, Twitter=1.5, News=2.0.

**Integration**: wire `resolveEntities()` into `summarize.ts` after Stage 1. Register decay in scheduler at `0 0 * * *` UTC.

**Verify**: Alias lookup resolves ETH/$ETH/ethereum. CoinGecko seed works. BI → Bank Indonesia (not Binance). Decay: 2.0 → 1.9. Archival after 90d. Reactivation at 0.5.

**Depends on**: Phase 2 (summaries exist). Independent of other Phase 3 steps.

---

### Step 3.2: Embeddings Integration (Track B)

**Files**: `src/embed-pipeline.ts`, `src/vector-cache.ts`

**Batch item embeddings**: 15-min cron, groups of 100, Gemini batch endpoint.

**Vector cache**: load summaries/reports/entities into `Map<string, Float32Array>` on startup (~9MB for 3K vectors). `cosineSimilarity()` in JS. Update methods for new embeddings.

**Semantic search**: extend `GET /api/v1/search?mode=semantic` — embed query, search cache, top-10 by cosine. Indonesian queries translated before embedding. Item search with pre-filter (time range + source) for large collections.

**Integration**: register 15-min cron in scheduler. Wire entity embedding into `entities.ts`.

**Verify**: Batch embeds 200 items in 2 API calls. Cache loads on startup. Semantic search for "Solana staking" returns relevant results. Pre-filter by time range works.

**Depends on**: Phase 2 (summaries/items exist). Independent of 3.1 and 3.3.

---

### Step 3.3: Auth (Track C)

**Files**: `src/auth/discord-oauth.ts`, `src/auth/sessions.ts`, `src/auth/middleware.ts`

**OAuth2 flow**:
- `GET /api/v1/auth/discord` → generate 32-byte state in httpOnly cookie (10min maxAge, **CSRF defense**) → redirect to Discord
- Callback: validate state, exchange code for token, fetch user profile, discard token
- Role resolution: `ADMIN_USER_IDS` env var first (always admin), then DB lookup
- Invite-only: unknown Discord IDs → 403

**Sessions**: DB-backed, ULID IDs, 30-day expiry with sliding refresh (>24h old → update). Max 5 per user. Cleanup in retention cron.

**Cookie**: `podders_session`, `httpOnly: true`, `secure: true`, `sameSite: 'lax'`, signed.

**Middleware**: `requireAuth` (check cookie OR Bearer API key → set `request.user`). `requireAdmin` (role check).

**User management** (admin-only): `GET/POST/PATCH/DELETE /api/v1/users`.

**Auth logging**: log all login attempts, session lifecycle. Mask `podders_session` in Pino.

**Verify**: Full OAuth flow. State parameter CSRF blocks tampered cookies. Unknown user → 403. Blocked user → 403. Max 5 sessions enforced. API key → admin role. Rate limit on failed auth.

**Depends on**: Phase 1 (server, db). Independent of 3.1 and 3.2.

---

### Step 3.4: Dashboard Scaffold (Track C, after 3.3)

**Files**: `dashboard/package.json`, `dashboard/index.html`, `dashboard/vite.config.ts`, `dashboard/tsconfig.json`, `dashboard/app.css`, `dashboard/src/main.tsx`, `dashboard/src/router.tsx`, `dashboard/src/lib/api.ts`, `dashboard/src/lib/types.ts`, `dashboard/src/components/AuthProvider.tsx`, `dashboard/src/components/ProtectedRoute.tsx`, `dashboard/src/components/Header.tsx`, `dashboard/src/components/EmptyState.tsx`, `dashboard/src/components/StatusBadge.tsx`, `dashboard/src/components/TypeBadge.tsx`

**npm packages** (dashboard/): `react` (19), `react-dom`, `react-router` (7), `tailwindcss` (4), `vite`, `@vitejs/plugin-react`, `typescript`

**Dark theme** (Tailwind v4 `@theme`):
- `--color-background: #0a0a0f`, `--color-surface: #111118`, `--color-surface-raised: #1a1a24`
- `--color-text-primary: #e2e2e8`, `--color-text-secondary: #8888a0`

**Auth context**: calls `GET /api/v1/auth/me` on mount. Provides `user/loading/logout`.

**Vite proxy**: `/api/v1` → `http://localhost:3000` in dev.

**SPA catch-all** in `server.ts`: serve `dashboard/dist/index.html`.

**Verify**: `cd dashboard && npm run dev` renders dark header. Auth redirects unauthenticated users to login.

**Depends on**: 3.3 (auth middleware).

---

### Step 3.5: Dashboard Pages (Track C, after 3.4)

**Files**: `dashboard/src/pages/Login.tsx`, `ReportView.tsx`, `ReportList.tsx`, `Settings.tsx`, `Feed.tsx` + ~20 components (TldrBlock, KeyEventCard, SentimentBar, SectionAccordion, ReportCard, SourceRow, FilterPills, Modal, ChannelPicker, ConfigField, PipelineStatusRow, CostSummary, FailedDeliveryRow, FeedItem, FeedImageGallery, LiveToggle, UserRow, UserAvatar, RoleBadge, LoadMoreButton)

**Login**: "Sign in with Discord" button. Error states: `?error=unauthorized`, `?error=blocked`.

**ReportView** (/): hero TL;DR, key events grid, entity sentiment bars, collapsible sections.

**ReportList** (/reports): type filter pills (All/Daily/Flash/Pulse), paginated, click → detail.

**Settings** (/settings): 4 tabs — Sources, Delivery, Pipeline Health, Users (admin only). Viewers get read-only, no Users tab.

**Feed** (/feed): raw Discord messages with images, channel filter, live/paused toggle (10s poll).

**Verify**: All pages render. Source CRUD works. Role-based visibility correct. Mobile responsive at 375px.

**Depends on**: 3.4 (scaffold).

---

### Step 3.6: RAG Chat Backend (Track B, after 3.2)

**Files**: `src/chat/handler.ts`, `src/chat/tools.ts`, `src/chat/conversations.ts`

**Three retrieval tools** (Sonnet picks strategy):
- `semantic_search` — embed query → cosine search in vector cache → top-10 summaries/reports
- `keyword_search` — lookup entity in aliases → mention counts + sentiment from entity_mentions
- `read_raw` — fetch item by ID from `items` table

**Security**:
- All tool results wrapped in `<tool_result_${nonce}>` tags
- `read_raw` results pass through `sanitizeForPrompt()` (original source content)
- `semantic_search` and `keyword_search` results wrapped but NOT sanitized (already pipeline-processed)

**Indonesian queries**: detect with franc, translate via Haiku before embedding.

**Conversation history**: in-memory Map, last 5 messages per conversationId. Cleaned up after 1h idle. Not persisted across restarts.

**Verify**: "what happened with SOL?" returns grounded answer with citations. `read_raw` sanitized. Indonesian queries translated. Follow-ups work. Unknown topics → "I don't have data."

**Depends on**: 3.1 (entity tables for keyword_search), 3.2 (vector cache for semantic_search).

---

### Step 3.7: Dashboard Chat Page (after 3.5 + 3.6)

**Files**: `dashboard/src/pages/Chat.tsx`, `dashboard/src/components/ChatPanel.tsx`, `ChatMessage.tsx`, `SourceCitation.tsx`, `ToolUsageIndicator.tsx`

**Tool usage indicators**: while loading show "Searched summaries...", "Looked up ETH entity...", "Read source message..." — collapse to source citation pills after response.

**Source citations**: clickable pills → navigate to `/reports/:id` or show snippet.

"New Chat" clears conversation. Enter sends, Shift+Enter newline.

**Verify**: Send question → loading dots → answer with citations. Tool indicators appear then collapse. New Chat works. Mobile stacks correctly.

**Depends on**: 3.5 (dashboard pages), 3.6 (chat backend).

---

### Step 3.8: Narrative Clustering (Track D, after 3.2)

**Files**: `src/knowledge/narratives.ts`

Same implementation as Phase 2 Step 2.13 — if not already built there, build here. K-means on summary embeddings, silhouette auto-tuning, Haiku naming, growth-rate signal strength (3x→strong, 1.5x→emerging, 0.5x→fading, no match→new).

**Integration**: wire `detectNarratives()` call into `synthesize.runDaily()` before Sonnet call.

**Depends on**: 3.2 (vector cache with summary embeddings).

---

### Step 3.9: Operational Infrastructure (Track E — anytime)

**Files**: `src/ops/health.ts`, `src/ops/retention.ts`, `src/ops/backup.ts`, `src/ops/watchdog.ts`

**Health monitor** (every 5min): 7 checks (source silence, source disabled, LLM failures, missed pulse, missed daily, DB pool, cost spike). Dedup within 30min. Critical → POST to alert webhook.

**Retention** (daily 00:05 UTC): delete processed items >30d, summaries >90d, mentions >90d, cascade orphaned embeddings, cleanup expired sessions. Update vector cache after deletes.

**Backup** (daily 00:10 UTC): `pg_dump` → gzip → `DATA_DIR/podders_YYYYMMDD.sql.gz`. Retain 7 days.

**Watchdog**: write `/tmp/podders-heartbeat` every 60s. External monitor (systemd) restarts if stale >300s.

**Verify**: Simulate source silence → warn event. Simulate disabled source → critical + webhook POST. Old items purged. Backup file created. Heartbeat file updates.

**Depends on**: Phase 2 (scheduler). Independent of other Phase 3 steps.

---

### Step 3.10: Testing (Track F — anytime)

**Files**: `test/unit/entities.test.ts`, `test/unit/decay.test.ts`, `test/unit/auth.test.ts`, `test/fixtures/*.json`

**Entity tests** (17 cases from TESTING.md): alias lookup, concurrent writes, CoinGecko seed, Indonesian seeds, BI disambiguation.

**Decay tests**: math (2.0 × 0.95 = 1.9), archival threshold, dual condition (relevance AND last_seen), reactivation at 0.5.

**Auth tests**: valid session → 200, expired → 401, valid API key → admin, invalid → 401, viewer → 403 on admin routes, rate limit, state parameter CSRF, unknown user → redirect.

All tests mock external APIs.

**Verify**: `npm test` passes all files.

**Depends on**: Corresponding features built. Independent ordering.

---

### Phase 3 npm Dependencies

**Backend**: `ml-kmeans` (if not from Phase 2)

**Dashboard**: `react` (19), `react-dom`, `react-router` (7), `tailwindcss` (4), `vite`, `@vitejs/plugin-react`, `typescript`

---

## Full Cron Job Schedule (end state)

| Job | Schedule | Handler | Mutex |
|-----|----------|---------|-------|
| Source poll tick | every 60s | `scheduler.tick()` | `source-poll` |
| Item embedding batch | `*/15 * * * *` | `embedPipeline.batchEmbedItems()` | `embed-items` |
| Market pulse | `0 */3 * * *` UTC | `pulse.run()` | `pulse` |
| Daily digest | configurable time + TZ | `synthesize.runDaily()` | `stage3-daily` |
| Entity decay | `0 0 * * *` UTC | `decay.run()` | `entity-decay` |
| Data retention | `5 0 * * *` UTC | `retention.run()` | `retention` |
| Backup | `10 0 * * *` UTC | `backup.run()` | `backup` |
| Health monitor | `*/5 * * * *` | `health.check()` | `health-check` |
| Scraper health | `*/5 * * * *` | `sources.healthCheck()` | `scraper-health` |

---

## Complete File Manifest

### Phase 1 (src/)
```
src/index.ts                    # CLI entry (commander)
src/config.ts                   # Env loading + validation
src/server.ts                   # Fastify routes + static serving
src/logger.ts                   # Pino + secret masking
src/llm.ts                      # All LLM calls (Pi AI wrapper)
src/embed.ts                    # Gemini embeddings wrapper
src/url-validator.ts            # SSRF protection
src/db/connection.ts            # pg pool
src/db/migrations.ts            # Schema versioning + all tables
src/db/queries.ts               # Typed SQL functions
```

### Phase 2 (src/)
```
src/ingest/rss.ts               # RSS feed polling
src/ingest/discord.ts           # Discord Gateway WebSocket
src/ingest/twitter.ts           # twitterapi.io REST
src/ingest/news.ts              # Readability extraction
src/normalize/index.ts          # 6-gate normalize pipeline
src/normalize/spam.ts           # Spam rules
src/normalize/instruct-detector.ts  # Injection detection
src/normalize/url-expand.ts     # t.co/bit.ly expansion
src/pre-summarize/index.ts      # Long article compression
src/process/summarize.ts        # Stage 1: Haiku summarization
src/process/schemas.ts          # Zod schemas
src/process/chunk.ts            # Token-budgeted chunking
src/process/correlate.ts        # Stage 2: cross-source correlation
src/process/synthesize.ts       # Stage 3: daily + flash reports
src/process/exemplars.ts        # Exemplar selection (7+3)
src/process/dedup-events.ts     # keyEvent dedup (Levenshtein)
src/process/pulse.ts            # 3h market pulse
src/process/narratives.ts       # k-means narrative clustering
src/deliver/webhook.ts          # Discord webhook delivery
src/knowledge/entities.ts       # Entity resolution (3 tiers)
src/knowledge/aliases.ts        # Alias lookup + insert
src/knowledge/trust.ts          # Trust weight adjustment
src/scheduler.ts                # Cron + mutex + crash recovery
src/health.ts                   # 7-check health monitor
```

### Phase 3 (src/)
```
src/auth/discord-oauth.ts       # OAuth2 flow
src/auth/sessions.ts            # Session CRUD
src/auth/middleware.ts           # requireAuth + requireAdmin
src/knowledge/decay.ts          # Relevance decay + archival
src/knowledge/seed.ts           # CoinGecko + Indonesian seeds
src/embed-pipeline.ts           # Batch item embedding cron
src/vector-cache.ts             # In-memory vector cache
src/chat/handler.ts             # RAG chat request handler
src/chat/tools.ts               # 3 retrieval tool definitions
src/chat/conversations.ts       # In-memory conversation store
src/ops/health.ts               # Health monitoring cron
src/ops/retention.ts            # Data retention + cleanup
src/ops/backup.ts               # pg_dump backup
src/ops/watchdog.ts             # Heartbeat file writer
```

### Dashboard (dashboard/)
```
dashboard/package.json
dashboard/index.html
dashboard/vite.config.ts
dashboard/tsconfig.json
dashboard/app.css
dashboard/src/main.tsx
dashboard/src/router.tsx
dashboard/src/lib/api.ts
dashboard/src/lib/types.ts
dashboard/src/components/AuthProvider.tsx
dashboard/src/components/ProtectedRoute.tsx
dashboard/src/components/Header.tsx
dashboard/src/components/EmptyState.tsx
dashboard/src/components/StatusBadge.tsx
dashboard/src/components/TypeBadge.tsx
dashboard/src/pages/Login.tsx
dashboard/src/pages/ReportView.tsx
dashboard/src/pages/ReportList.tsx
dashboard/src/pages/Settings.tsx
dashboard/src/pages/Feed.tsx
dashboard/src/pages/Chat.tsx
dashboard/src/components/ChatPanel.tsx
dashboard/src/components/ChatMessage.tsx
dashboard/src/components/SourceCitation.tsx
dashboard/src/components/ToolUsageIndicator.tsx
dashboard/src/components/TldrBlock.tsx
dashboard/src/components/KeyEventCard.tsx
dashboard/src/components/SentimentBar.tsx
dashboard/src/components/SectionAccordion.tsx
dashboard/src/components/ReportCard.tsx
dashboard/src/components/SourceRow.tsx
dashboard/src/components/FilterPills.tsx
dashboard/src/components/Modal.tsx
dashboard/src/components/ChannelPicker.tsx
dashboard/src/components/ConfigField.tsx
dashboard/src/components/PipelineStatusRow.tsx
dashboard/src/components/CostSummary.tsx
dashboard/src/components/FailedDeliveryRow.tsx
dashboard/src/components/FeedItem.tsx
dashboard/src/components/FeedImageGallery.tsx
dashboard/src/components/LiveToggle.tsx
dashboard/src/components/UserRow.tsx
dashboard/src/components/UserAvatar.tsx
dashboard/src/components/RoleBadge.tsx
dashboard/src/components/LoadMoreButton.tsx
```

### Tests
```
test/unit/llm.test.ts
test/unit/url-validator.test.ts
test/unit/entities.test.ts
test/unit/decay.test.ts
test/unit/auth.test.ts
test/fixtures/stage1-entity-extraction.json
test/fixtures/stage1-indonesian-entities.json
```
