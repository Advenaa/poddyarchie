# Podders v2 — Architecture

## What Is This

A 24/7 market intelligence engine. Scrapes social and news sources, builds knowledge over time, surfaces readable market conditions for crypto/DeFi and tradfi. Ships as a single process, single port, backed by Postgres.

**Known trade-off**: Discord ingestion uses user tokens via Gateway WebSocket, which violates Discord TOS ("self-botting"). Risk: token ban, account termination. Mitigate by using disposable accounts. Consider migrating to bot tokens with guild invites for production.

## Core Loop

```
Sources → Ingest → Normalize → Pre-summarize → Process → Knowledge → Surface
```

## 1. Sources

| Source    | Method              | Priority | Notes                              |
|-----------|--------------------|---------|------------------------------------|
| Discord   | Gateway WebSocket  | P0      | One connection per token, 5s IDENTIFY gap, MESSAGE_CREATE only |
| Twitter/X | twitterapi.io       | P0      | REST API, $0.15/1K tweets, per-source poll interval (default 2h). WebSocket available for v2.1 |
| News      | Article extraction | P1      | Readability-style content extraction from URLs |
| RSS       | Feed polling       | P1      | `rss-parser` npm, 15min poll, auto-extract full article via Readability |

### Source Configuration

Lives in database, not files. Configurable at runtime via `/settings` page.

```sql
CREATE TABLE sources (
  source TEXT NOT NULL,          -- discord | twitter | news | rss
  source_id TEXT NOT NULL,       -- channel id, handle, feed url
  label TEXT,                    -- human-readable name
  enabled BOOLEAN DEFAULT true,
  priority INTEGER DEFAULT 1,
  poll_interval INTEGER DEFAULT 7200,  -- per-source poll interval in seconds (default 2h)
  added_at INTEGER NOT NULL,
  PRIMARY KEY (source, source_id)
);
```

`.env` holds secrets only: `ANTHROPIC_API_KEY` (required if using Claude models, other provider keys as needed), `DATABASE_URL` (Postgres connection string, required), `DISCORD_TOKENS` (optional — needed for Discord sources), `TWITTERAPI_KEY` (optional — needed for Twitter sources), `API_KEY` (auto-generated if missing).

### Discord Gateway

- One WebSocket connection per token. Tokens from `DISCORD_TOKENS` env var (comma-separated).
- On startup: connect each token sequentially with 5s gap (IDENTIFY rate limit).
- Partition channels across tokens round-robin from `sources` table.
- Listen to `MESSAGE_CREATE` only. Ignore presence, typing, reactions. Capture attachment URLs from `message.attachments` array (Discord CDN URLs) into `items.attachments` as JSON array.
- On fatal close codes (4004, 4010-4014): disable that token's channels, log CRITICAL.
- On resumable codes (4000-4003, 4009): resume with stored session.
- On 4007 (invalid seq): fresh IDENTIFY (do not resume).
- On 4008 (rate limited): backoff 60s, then fresh IDENTIFY.
- On network drop: reconnect with exponential backoff capped at 60s.
- See [docs/DISCORD.md](./docs/DISCORD.md) for full close code reference.

### twitterapi.io Integration

- REST polling based on per-source `poll_interval` (default 2h). WebSocket streaming deferred to v2.1.
- Endpoints: `/twitter/tweet/advanced_search` (search queries), `/twitter/user/last_tweets` (tracked accounts).
- Auth: `x-api-key` header. 15s timeout per request, max 5 pages (100 tweets) per source per cycle.
- Each `source_id` in `sources` where `source='twitter'` is prefixed: `@handle` for accounts, plain text for search queries. Batch multiple accounts into one search query for cost efficiency.
- Data mapping: `text` → `content`, `author.userName` → `author`, `log10(likes + 2*retweets + 3*quotes) * 20` → `engagement`. Only Twitter uses this formula — Discord uses reaction count as `engagement`, RSS items have `engagement: 0` (no engagement signal available).
- **Validate response** with zod schema. If shape changes, fail fast with clear error.
- Dedup by tweet ID (`last_id` in `source_state`) + content hash in normalize layer.
- See [docs/TWITTER.md](./docs/TWITTER.md) for full spec.

### Scraper Resilience

Each scraper tracks state in `source_state`:

```sql
CREATE TABLE source_state (
  source TEXT NOT NULL,
  source_id TEXT NOT NULL,
  status TEXT DEFAULT 'active',     -- active | backoff | disabled
  last_fetched_at INTEGER,
  last_id TEXT,
  error_count INTEGER DEFAULT 0,
  last_error TEXT,
  next_retry_at INTEGER,
  PRIMARY KEY (source, source_id)
);
```

- **Initial window**: a brand-new source with no `source_state` entry uses `now - 2h` as the initial window start (one micro-batch window), not epoch 0. This avoids attempting to fetch all historical data on first run.
- **Backoff**: on error, `next_retry_at = now + min(2s * 2^error_count, 1h)`. Status → `backoff`.
- **Disable**: if `error_count >= 5` within 1 hour, status → `disabled`. Surfaces on `/settings`.
- **Token revocation**: Discord 401/403 → immediately disable that token's sources, log CRITICAL.
- **Recovery**: reset `error_count` on successful fetch. Health check cron (5min) resets `backoff` sources past `next_retry_at`.

## 2. Ingest

Every source adapter produces a unified format:

```typescript
interface RawItem {
  id: string                    // ulid
  source: 'discord' | 'twitter' | 'news' | 'rss'
  sourceId: string
  author: string
  content: string
  timestamp: Date
  url?: string
  engagement: number            // normalized 0-100 (0 = no data, 1-100 = scored)
  attachments?: string[]        // attachment URLs (e.g., Discord CDN image URLs)
  metadata: Record<string, unknown>
}
```

`engagement` is a first-class indexed column for "high-engagement items" queries. Per-source defaults: Twitter uses `log10(likes + 2*retweets + 3*quotes) * 20`, Discord uses reaction count, RSS always sets `engagement: 0` (no signal available).

## 3. Normalize (Stage 0 — No LLM)

Runs on every incoming item. Pure rule-based — no LLM calls.

1. **Truncate** — items exceeding 20,000 characters are truncated to that limit. Prevents single oversized items (e.g., full articles from RSS) from blowing the Stage 1 chunk budget.
2. **Dedup** — `sha256(source + sourceId + content.slice(0, 200))`. Plus exact URL match across sources after URL expansion. No SimHash for MVP — exact hash + URL dedup covers 90%.
3. **URL expansion** — resolve t.co, bit.ly redirects for cross-source dedup.
4. **Spam filter** — rule-based. English: bot accounts, sub-5-word posts, "gm/gn". Indonesian: "wm", "done min/sudah min", "gas!", "mantap", single-emoji, airdrop copypasta. Kills 40-60% of Discord volume.
5. **Language detection** — `franc` library. Process English (`eng`) and Indonesian (`ind`) — franc uses ISO 639-3 codes — exclude others. Indonesian is first-class.

Items pass → `status: 'ready'` with full original content. Filtered → `status: 'filtered'` (kept for debugging).

## 3.5. Pre-Summarize

Runs after normalize saves items as `ready`, before the claim & chunk step. Decides per-item whether to compress or pass raw based on source type and content characteristics.

**Skip (pass raw content to Stage 1):**
- Discord/Twitter items — already short, no compression needed.
- RSS/news with urgency keywords (`exploit`, `hack`, `rate decision`, `flash crash`, `halt`, `circuit breaker`) — full text carries critical detail.
- RSS/news with high entity density (>2 entities per 100 tokens, detected via regex for capitalized multi-word terms + `$`-prefixed tokens) — already signal-dense.

**Compress via Haiku:**
- RSS/news with low entity density — a 10K-char article otherwise dominates a chunk's token budget while carrying the same signal as a 500-char Discord message.
- Flat 300 token `maxTokens` cap.
- Two prompt variants: one for regulatory feeds, one for general news.
- Batched 3-5 articles per Haiku call for cost efficiency.
- Stores `content_anchor` (first 200 tokens of original) in the `items` table for embedding quality. The `content` column gets the compressed version.

**Cost:** ~$0.001 per article at Haiku rates.

## 4. Process (LLM Pipeline)

### Scheduler

Uses `node-cron` for a short-interval tick (every 60s) that checks which sources are due for processing. Two claim mechanisms depending on source type:

- **Twitter/RSS/News**: fixed `poll_interval`. Due when `now - last_fetched_at >= poll_interval`.
- **Discord**: Poisson-based adaptive claim. Messages arrive via WebSocket and sit as `ready`. Every 60s, the scheduler counts accumulated `ready` items and compares against the expected rate (lambda) from `source_rate_history`. Two thresholds:
  - **Process threshold**: actual > expected x 1.5 OR actual > 10 items — trigger processing.
  - **Alert threshold**: actual > expected x 3 OR Poisson survival probability < 0.01 — process immediately, flag as potential breaking event.

This catches breaking events ~12 minutes faster than fixed 15-min intervals. Lambda is learned from the last 7 days, keyed by hour-of-day and day-of-week.

Additionally, a 3-hour pulse cron (8x/day) triggers Sonnet market pulse synthesis. Event-driven jobs (Stage 2 after Stage 1, flash after breaking, delivery after daily/pulse) called directly at end of triggering function.

**Due-source query (Twitter/RSS/News)**: on each tick, select sources where `now - last_fetched_at >= poll_interval` (or `last_fetched_at IS NULL` for new sources). Process all due sources in parallel via `Promise.allSettled` — Postgres handles concurrent writes.

**Poisson check (Discord)**: on each tick, count `ready` items per Discord source. Look up lambda from `source_rate_history` for current hour and day-of-week. Compare against thresholds. Update `source_rate_history` after each processing cycle.

**Overlap guard**: each tick has a `running` boolean mutex. If a tick fires while the previous run is in-flight, skip and log warning. Batch-claiming SQL prevents double-processing of items regardless.

**Startup sequence**: run crash recovery (`UPDATE items SET status='ready' WHERE status='processing'`), then start cron.

**Rescheduling**: when `digest_time` or `timezone` changes via `PATCH /api/v1/config`, destroy and recreate the Stage 3 daily cron task with the new schedule. `node-cron` does not support in-place rescheduling.

### Call Chain (explicit)

The pipeline stages are connected via direct function calls, not an event bus:

```
scheduler (node-cron, every 60s tick)
  → getDueSources()                # SELECT sources WHERE now - last_fetched_at >= poll_interval
  → Promise.allSettled(            # Process all due sources in parallel
      dueSources.map(source =>
        ingest(source)             # Fetch + normalize for this source
      )
    )
  → preSummarize.run()             # Compress low-density RSS/news, skip Discord/Twitter/urgent/dense
  → summarize.runBatch()           # Stage 1: claim ready items, call Haiku, write summaries + entities
    → correlate.run()              # Stage 2: SQL cross-source query (called at end of runBatch)
    → if any chunk.urgency === 'breaking':
        synthesize.runFlash()      # Stage 3 flash: Sonnet, last 4h, webhook with [FLASH]

scheduler (node-cron, every 3h — 8x/day)
  → synthesize.runPulse()          # Stage 3 pulse: Sonnet, last 3h window (ground truth + drift flags injected)
    → webhook.deliver(report)      # Delivery: muted grey Discord embed (skip if nothing happened)

scheduler (node-cron, daily at digest_time)
  → synthesize.runDaily()          # Stage 3 daily: Sonnet, reads 8 pulse outputs + correlated entities + raw Stage 1 entity data
    → webhook.deliver(report)      # Delivery: POST to Discord webhook
```

scheduler (node-cron, every 5min)
  → health.check()                 # 7 checks → inserts into health_events if threshold breached
                                   # Critical events → POST to ALERT_WEBHOOK_URL (red Discord embed)
                                   # Dedup: skip if same category+message unacknowledged within 30min

Each `→` is a direct function call. No message passing, no event emitters.

### Stage 1: Summarize (Haiku, micro-batch per source poll)

**Token-based chunking**, not item-count. Budget: 6,000 input tokens per chunk (~24K chars). Estimate via `content.length / 4`. If a source exceeds budget in one window, split into multiple chunks.

**Zero items**: if no `ready` items exist for a window, log `info` and skip. No LLM call, no empty summary created.

Each chunk → one Haiku call (`maxTokens: 3000`) returning structured JSON. Entity extraction happens here — no separate pass. Scraped content wrapped in `<scraped_content>` XML delimiters with explicit instruction to treat as untrusted data (prompt injection defense).

**Exemplar slots**: when assembling Stage 3 input from Stage 1 outputs, 10 exemplar slots are used. 7 slots ranked by engagement score. 3 slots reserved for RSS/news items ranked by entity density + urgency (since RSS engagement is always 0 and would never surface otherwise).

See [PIPELINE.md](./PIPELINE.md) for prompt details, chunking code, and validation.

**Idempotency via batch claiming:**

```
1. Claim: UPDATE items SET batch_id=?, status='processing' WHERE status='ready' AND ...
2. Call LLM
3. Commit: INSERT summary + UPDATE items SET status='processed' (in transaction)
4. Crash recovery on startup: UPDATE items SET status='ready' WHERE status='processing'
```

Three states: `ready` → `processing` → `processed`.

**Entity resolution** uses alias lookup-then-insert against `entity_aliases` table (lowercase normalized). CoinGecko token list seeds the alias table. `INSERT ... ON CONFLICT DO NOTHING` handles concurrent batch writes gracefully.

### Stage 2: Correlate (SQL, no LLM)

Cross-source signal detection — entities appearing in 2+ sources within the same window. See [PIPELINE.md](./PIPELINE.md) for the query.

### Stage 3: Synthesize (Sonnet, daily + flash + pulse)

Reads all summaries from past 24h + correlated signals → `MarketReport`. Yesterday's TL;DR passed as context for temporal anchoring.

If volume exceeds 50 summaries, rank by `item_count * avg_engagement` and take top 30.

**Causal chain instructions** (included in Stage 3 system prompt for daily, flash, and pulse):
- When multiple elevated or breaking events share entities or temporal proximity, explicitly state the causal chain or connection between them.
- Do not assume causation from entity co-occurrence alone — only connect events when the relationship is evident from the source data.
- Flag cross-entity ripple effects (e.g., bridge exploit → token sell-off → DeFi TVL drop) as a connected sequence, not isolated events.

**Zero summaries**: if no summaries exist for the 24h window, skip synthesis. Insert a "no activity" report with `tldr: "No market activity detected across configured sources."` so the dashboard always has something to show, or skip entirely and log. Decision: skip + log — don't produce empty reports.

**Flash reports**: when Stage 1 returns `urgency: 'breaking'`, trigger immediate Stage 3 scoped to last 4h. Delivers via same webhook with `[FLASH]` prefix. Expected: 2-3/month.

**Market pulse**: 8 Sonnet calls per day, every 3 hours. Each pulse carries the prior pulse's TL;DR as context. Output scales with activity:
- 1-3 summaries, all routine → 2-3 sentences (`maxTokens: 300`)
- 4-10 summaries, some elevated → paragraph + key events (`maxTokens: 800`)
- 10+ summaries or any breaking → full short report with sections (`maxTokens: 1500`)

Before each pulse call, inject ground truth (raw Stage 1 entity data for the current 3h window) and drift flags (prior pulse entity sentiments vs current Stage 1 data, threshold 0.4). Zero extra LLM calls, ~300 extra input tokens per pulse. See PIPELINE.md for `detectDrift` function.

Quality gate: skip delivery if nothing happened. Daily synthesis reads all 8 pulse outputs + full day's correlated entities + raw Stage 1 correlated entity data (not raw summaries). Delivered via Discord webhook with muted grey embed.

**Duplicate prevention**: for daily reports, check `SELECT id FROM reports WHERE date=? AND type='daily'` before inserting. If exists, skip and log. Flash and pulse reports allow multiples per day.

### LLM Error Handling

**Model IDs** must come from `config.ts`, never hardcoded in call sites. Defaults: Haiku = `claude-haiku-4-5-20251001`, Sonnet = `claude-sonnet-4-6-20250514`. Stage 1 (summarize) and pre-summarize use Haiku. Stage 3 (synthesize, pulse, flash) uses Sonnet.

`llm.ts` wraps all Anthropic API calls with typed error handling:

| Error | Action |
|-------|--------|
| 400 context length exceeded | Split chunk in half, retry both halves |
| 400 malformed request | Log + skip batch, alert |
| 401 invalid key | Log CRITICAL, halt all LLM jobs |
| 429 rate limit | Respect `retry-after` header, queue retry |
| 529 overloaded | Backoff 30s, retry up to 3x |
| 500/502/503 | Retry 3x with exponential backoff (2s/8s/32s) |
| Valid HTTP but invalid JSON | Retry once with correction prompt, then skip + log |
| Refusal (empty content) | Log as refusal, skip batch, surface on /settings |

## 5. Knowledge

Postgres via `pg` (node-postgres) with a connection pool. All queries are async. Postgres handles concurrent writes natively — no single-writer bottleneck.

### Schema

**items**
```sql
CREATE TABLE items (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,
  source_id TEXT NOT NULL,
  author TEXT NOT NULL,
  content TEXT NOT NULL,
  timestamp INTEGER NOT NULL,
  url TEXT,
  engagement INTEGER DEFAULT 0,
  attachments TEXT,               -- JSON array of attachment URLs (e.g., Discord CDN image URLs)
  content_hash TEXT NOT NULL,
  content_anchor TEXT,            -- first 200 tokens of original content (set by pre-summarize for compressed items, used for embedding quality)
  status TEXT DEFAULT 'ready',    -- ready | filtered | processing | processed
  batch_id TEXT,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_items_claim ON items(status, source, timestamp);
CREATE INDEX idx_items_hash ON items(content_hash);
```

**items full-text search** (Postgres tsvector)
```sql
ALTER TABLE items ADD COLUMN content_tsv tsvector
  GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;
CREATE INDEX idx_items_fts ON items USING GIN (content_tsv);

-- Query example:
-- SELECT * FROM items WHERE content_tsv @@ plainto_tsquery('english', $1)
--   ORDER BY ts_rank(content_tsv, plainto_tsquery('english', $1)) DESC;
```

No triggers needed — Postgres generated columns keep `content_tsv` in sync automatically.

**summaries**
```sql
CREATE TABLE summaries (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,
  source_id TEXT NOT NULL,
  window_start INTEGER NOT NULL,
  window_end INTEGER NOT NULL,
  body TEXT NOT NULL,              -- JSON ChunkSummary
  sentiment REAL,                  -- derived: avg of per-entity sentiments in body
  urgency TEXT,                    -- from LLM output
  item_count INTEGER NOT NULL,
  created_at INTEGER NOT NULL
);
```

**entities**
```sql
CREATE TABLE entities (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  type TEXT NOT NULL,              -- token | person | project | company | event
  relevance REAL DEFAULT 0,
  first_seen INTEGER NOT NULL,
  last_seen INTEGER NOT NULL,
  UNIQUE(name, type)              -- allows "Mercury" as token AND company
);
```

**entity_aliases**
```sql
CREATE TABLE entity_aliases (
  alias TEXT PRIMARY KEY,          -- lowercase normalized
  entity_id TEXT NOT NULL REFERENCES entities(id)
);
```

**entity_mentions**
```sql
CREATE TABLE entity_mentions (
  id TEXT PRIMARY KEY,
  entity_id TEXT NOT NULL REFERENCES entities(id),
  source TEXT NOT NULL,
  summary_id TEXT REFERENCES summaries(id),
  sentiment REAL,
  mention_count INTEGER DEFAULT 1,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_mentions_entity_ts ON entity_mentions(created_at, entity_id);
CREATE INDEX idx_mentions_created_entity ON entity_mentions(created_at, entity_id);
```

**reports**
```sql
CREATE TABLE reports (
  id TEXT PRIMARY KEY,
  date TEXT NOT NULL,              -- YYYY-MM-DD
  type TEXT DEFAULT 'daily',       -- daily | flash | pulse
  body TEXT NOT NULL,              -- JSON MarketReport
  tldr TEXT,                       -- extracted for search/display
  sentiment REAL,                  -- extracted for queryability
  delivery_status TEXT DEFAULT 'pending',  -- pending | delivered | failed
  delivered_at INTEGER,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_reports_date ON reports(date);
-- Daily reports: one per day (enforced in code via INSERT ... ON CONFLICT DO NOTHING with date+type check)
-- Flash/pulse reports: multiple per day allowed
```

**app_config**
```sql
CREATE TABLE app_config (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- Seeds: webhook_url, digest_time ('09:00'), timezone ('Asia/Jakarta'), api_key_hash
```

**llm_usage**
```sql
CREATE TABLE llm_usage (
  id TEXT PRIMARY KEY,
  stage TEXT NOT NULL,
  model TEXT NOT NULL,
  input_tokens INTEGER NOT NULL,
  output_tokens INTEGER NOT NULL,
  cost_usd REAL NOT NULL,
  created_at INTEGER NOT NULL
);
```

**health_events**
```sql
CREATE TABLE health_events (
  id TEXT PRIMARY KEY,
  category TEXT NOT NULL,           -- source | llm | scheduler | db | pipeline | cost
  severity TEXT NOT NULL,           -- warn | critical
  message TEXT NOT NULL,
  metadata JSONB DEFAULT '{}',
  acknowledged BOOLEAN DEFAULT false,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_health_events_unacked ON health_events (acknowledged, created_at DESC) WHERE acknowledged = false;
```

**source_rate_history** (Poisson baseline for Discord claim gate)
```sql
CREATE TABLE source_rate_history (
  source TEXT NOT NULL,
  source_id TEXT NOT NULL,
  hour_of_day INTEGER NOT NULL,    -- 0-23
  day_of_week INTEGER NOT NULL,    -- 0-6
  avg_rate REAL NOT NULL,          -- messages per hour
  sample_count INTEGER DEFAULT 0,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (source, source_id, hour_of_day, day_of_week)
);
```

Tracks expected message rate (lambda) per source, per hour-of-day, per day-of-week. Learned from last 7 days. Used by the Poisson-based claim gate for Discord sources (see Scheduler section).

### Schema Versioning

A `schema_version` table tracks the current version. `migrations.ts` checks on startup, runs numbered migration functions in a transaction, bumps version. No external migration library needed.

```sql
CREATE TABLE IF NOT EXISTS schema_version (
  version INTEGER NOT NULL
);
```

### Entity Relevance Model

```
relevance = relevance * 0.95^days_since_update + log(1 + mentions) * source_weight
```

Source weights: Discord=1.0, Twitter=1.5, News=2.0 (rarer = higher signal). Applied daily via cron.

### Data Retention

Daily cron after entity decay:

- **Items**: delete where `status='processed' AND created_at < now - 30d`. Summaries are the archival layer.
- **Summaries**: keep 90 days.
- **Reports**: keep forever (~1KB each).
- **Entity mentions**: delete where `created_at < now - 90d`.
- **Entities**: prune where `relevance < 0.01 AND last_seen < now - 90d`. Cascade deletes aliases and mentions.
- At 5K items/day, 30-day retention caps `items` at ~150K rows.

### Backup

Postgres backup via `pg_dump`. Daily after entity decay. Timestamped files in `backups/`, retain 7 days.

## 6. Surface

### Daily Digest (ships first)

Discord webhook delivery. The habit trigger — proves the pipeline end-to-end before any frontend exists.

**Webhook delivery spec:**
- 3 retries, exponential backoff: 2s → 8s → 32s
- Retry on: HTTP 429, 5xx, network timeout (10s)
- No retry on: 4xx (except 429) — permanent failure
- On failure: set `reports.delivery_status = 'failed'`, log full context
- Dashboard surfaces failed deliveries on `/settings`

### Dashboard

**Vite + React + React Router** SPA. Built to `dashboard/dist/`, served by Fastify via `@fastify/static`. One process, one port.

**SPA catch-all**: `server.ts` must register a wildcard route `GET /*` (after API routes) that serves `dashboard/dist/index.html` for any non-`/api/v1/*` path. Without this, browser refresh on `/reports` or `/settings` returns 404.

**No Zustand** — `useState` + `useEffect` + a thin `fetchApi` wrapper is sufficient. Auth token stored in `localStorage`. On 401 → clear token, show "paste your API key" prompt. No shared cross-page state needed for MVP.

**Tailwind CSS 4** — use `@theme` in CSS (v4 pattern), not `tailwind.config.ts` (v3 pattern). Define colors as CSS custom properties: `--color-surface: #111118`, reference as `bg-surface`. Always dark, no `dark:` prefixes.

**Separate package.json** — dashboard has its own `package.json` with react, react-dom, react-router, tailwindcss, vite, @vitejs/plugin-react. Root `postinstall` runs `cd dashboard && npm install`.

**MVP: 5 routes**

| Route | Content |
|-------|---------|
| `/` | Latest report — TL;DR → key events → sentiment |
| `/reports` | Date list, click to view any past report |
| `/feed/:sourceId` | Raw item feed for a source — messages with images as they arrive, chronological |
| `/chat` | RAG chat — ask questions about your data, answers grounded in summaries/reports/items |
| `/settings` | Source management, delivery config, pipeline health, failed deliveries |

**Landing experience**: User opens app → sees today's TL;DR in under 5 seconds. Progressive disclosure below. On mobile, sections are collapsed behind "Read more" — only TL;DR + key events visible by default.

**Empty states**:
- `/` (no reports yet): "Welcome to Podders. Add your first source in Settings to start generating reports." with CTA button to `/settings`.
- `/reports` (no reports): "No reports yet. Your first report will generate after sources are configured and the daily digest runs."
- `/settings` (no sources): Guided "Add your first source" prompt with per-type cards (Discord, Twitter, RSS, News).

**Mobile-responsive**: single column at 375px. TL;DR full-width, key events as vertical cards, sentiment as horizontal bars. Tailwind responsive prefixes. No table layouts on mobile.

### Auth

**API key** in `Authorization: Bearer <key>` header. Generated on first run. **Stored as argon2 hash** in `app_config` — raw key displayed once at generation, never again.

- `POST /api/v1/auth/rotate` — generate new key, invalidate old
- Rate limiting on failed auth: 5 failures/min per IP via `@fastify/rate-limit`

Multi-user escape hatch: config tables have `user_id TEXT DEFAULT 'default'` for non-destructive migration later.

### Input Validation

All API routes use **Fastify JSON Schema validation**:
- `source` constrained to `enum: ['discord', 'twitter', 'news', 'rss']`
- `sourceId` format validated per source type (Discord = numeric snowflake, Twitter = handle pattern, RSS = URL)
- `label` length capped
- `webhookUrl` validated: must be `https://discord.com/api/webhooks/` or `https://discordapp.com/api/webhooks/`
- RSS feed URLs: require `https://`, resolve DNS, reject private/reserved IP ranges (SSRF protection)

### Security Headers

`server.ts` sets on all responses:
- `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'`
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`

Global rate limit: 100 req/s per IP across all `/api/v1/*` routes via `@fastify/rate-limit` (in addition to the 5/min auth failure limit).

### URL Validation (shared)

Used by RSS fetcher, webhook delivery, and news extractor:
- Require `https://` (or explicit allowlist for `http://`)
- Resolve DNS, reject private IPs (10.x, 172.16-31.x, 192.168.x, 127.x, 169.254.x, ::1)
- Reject non-standard ports
- Block `file://` scheme

### API Contract

All endpoints: `/api/v1/*`. All require `Authorization: Bearer <api_key>`.

**Reports:**

| Method | Path | Response |
|--------|------|----------|
| `GET` | `/api/v1/reports` | `{ reports: [{ id, date, type, tldr, createdAt }] }` — paginated `?limit=20&offset=0&type=daily\|flash\|pulse` |
| `GET` | `/api/v1/reports/latest` | `{ report: MarketReport }` or `{ report: null }` if no reports exist |
| `GET` | `/api/v1/reports/:id` | `{ report: MarketReport }` |

**Sources:**

| Method | Path | Response |
|--------|------|----------|
| `GET` | `/api/v1/sources` | `{ sources: [{ source, sourceId, label, enabled, pollInterval, lastFetchedAt, errorCount, lastError, status }] }` — JOINs `sources` + `source_state` |
| `POST` | `/api/v1/sources` | Body: `{ source, sourceId, label, pollInterval? }`. Per-type validation. |
| `PATCH` | `/api/v1/sources/:source/:sourceId` | Body: `{ enabled?, label?, pollInterval? }` |
| `DELETE` | `/api/v1/sources/:source/:sourceId` | Removes source config (keeps historical items) |
| `POST` | `/api/v1/sources/:source/:sourceId/test` | Test connection. Returns `{ ok: true }` or `{ ok: false, error: "..." }` |
| `GET` | `/api/v1/sources/discover/discord` | Returns `{ guilds: [{ id, name, icon, channels: [{ id, name }] }] }` from connected tokens |
| `POST` | `/api/v1/sources/discover/rss` | Body: `{ url }`. Fetches HTML, finds `<link rel="alternate">` RSS/Atom feeds. Returns `{ feeds: [{ url, title }] }` |

**Config & Health:**

| Method | Path | Response |
|--------|------|----------|
| `GET` | `/api/v1/config` | `{ webhookUrl, digestTime, timezone, publicUrl }` |
| `PATCH` | `/api/v1/config` | Body: partial of above |
| `GET` | `/api/v1/status` | `{ stages: { ingest, summarize, synthesize, delivery }, llmCostToday, allSourcesDisabled: bool }` |
| `GET` | `/api/v1/health` | `{ events: HealthEvent[], sourceStates: SourceState[], lastPulse: number, lastDaily: number, cost24h: number }` — unacknowledged events + system vitals |
| `PATCH` | `/api/v1/health/:id` | Acknowledge a health event |
| `POST` | `/api/v1/auth/rotate` | `{ apiKey: "new-key-displayed-once" }` |

**Search (after full-text search, build step 8.5):**

| Method | Path | Response |
|--------|------|----------|
| `GET` | `/api/v1/search?q=solana&days=7` | `{ items: [{ id, source, author, content, timestamp }] }` — Postgres `tsvector`/`tsquery` with optional time filter |

**Feed (raw item feed per source):**

| Method | Path | Response |
|--------|------|----------|
| `GET` | `/api/v1/feed/:sourceId` | `{ items: [{ id, source, author, content, timestamp, attachments, engagement }] }` — raw items filtered by source, paginated `?limit=50&offset=0` |

**Chat (RAG):**

| Method | Path | Response |
|--------|------|----------|
| `POST` | `/api/v1/chat` | Body: `{ message: string, conversationId?: string }`. Response: `{ answer: string, sources: [{ type: string, id: string, snippet: string }], conversationId: string }` |

How it works: user question is embedded via Gemini (same model as all embeddings), cosine similarity finds top 10 relevant summaries/reports/items from the in-memory vector cache, those results plus last 5 messages of conversation history are passed to Sonnet as context, Sonnet generates an answer grounded in the retrieved data. Conversation history stored in memory (cleared on restart). Sonnet system prompt instructs: answer ONLY from provided context, cite sources, say "I don't have data on that" if no relevant results, never speculate beyond the data. Each chat message = 1 Gemini embedding call + 1 Sonnet call.

**Vector retrieval details**: cosine similarity (standard for text embeddings). Retrieves top-10 results with no minimum similarity threshold -- Sonnet judges relevance from context rather than a hard cutoff. The vector cache is loaded into memory on startup from the `embeddings` table and rebuilt if the embedding model changes. Memory budget: ~50MB for 20K vectors x 768 dims x 4 bytes (Float32), well within VPS RAM. Conversation state is kept in-memory per session with no persistence across server restarts.

**Settings:**

| Method | Path | Response |
|--------|------|----------|
| `PATCH` | `/api/v1/settings` | Body: `{ webhookUrl?, digestTime?, alertWebhookUrl? }`. Update webhook URL, digest time, or alert webhook. |

**Deliveries:**

| Method | Path | Response |
|--------|------|----------|
| `GET` | `/api/v1/deliveries?status=failed` | `{ deliveries: [{ id, reportId, status, error, createdAt, retriedAt }] }` — list failed deliveries |
| `POST` | `/api/v1/deliveries/:id/retry` | `{ ok: true }` — retry a failed delivery |
| `DELETE` | `/api/v1/deliveries/:id` | `{ ok: true }` — clear a failed delivery from the list |

**Costs:**

| Method | Path | Response |
|--------|------|----------|
| `GET` | `/api/v1/costs?period=day\|week\|month` | `{ costs: [{ model, stage, calls, inputTokens, outputTokens, costUsd }], total: number }` — cost breakdown by model and stage |

**Source Admin:**

| Method | Path | Response |
|--------|------|----------|
| `POST` | `/api/v1/sources/:source/:sourceId/reset` | `{ ok: true }` — reset `error_count` to 0 and status to `active` |

26 endpoints total.

### Webhook Content

Every webhook message includes a **"View full report"** link: `{publicUrl}/reports/{id}`. Resolved in order: `PUBLIC_URL` env var → `publicUrl` in `app_config` → omit link if neither set. Flash reports use `[FLASH]` prefix in the webhook title.

### Source Configuration UX

Per-type "add source" flows, not a single generic form:
- **Discord**: channel picker populated from `GET /sources/discover/discord`. User selects guild → channel. Falls back to manual channel ID paste if discovery fails.
- **Twitter**: text field with prefix hint — `@handle` for accounts, plain text for search queries.
- **RSS**: URL field with `https://` validation.
- **News**: URL field for article extraction (manual submission).

## 7. Observability

Ships with MVP, not optional.

- **Structured logging** — pino, JSON output. Mask `DISCORD_TOKENS`, `ANTHROPIC_API_KEY`, `TWITTERAPI_KEY`, `API_KEY`, webhook URLs in all log output.
- **Scraper health** — `source_state` table, surfaced on `/settings`
- **LLM cost tracking** — `llm_usage` table, daily total on `/api/v1/status`
- **Pipeline status** — last successful run per stage, visible on `/settings`
- **Failed deliveries** — `reports.delivery_status`, visible on `/settings`
- **Health events** — `health_events` table, 7 automated checks every 5min, critical alerts to `ALERT_WEBHOOK_URL`
- **Process watchdog** — heartbeat file at `/tmp/podders-heartbeat` every 60s, systemd/cron restarts if >300s stale

## 8. Deployment

### Process Management

Deploy via **pm2** or **systemd** on a VPS. No Docker.

See [docs/DEPLOYMENT.md](./docs/DEPLOYMENT.md) for full pm2/systemd config and deploy procedure.

### Graceful Shutdown

Trap SIGTERM/SIGINT:
1. Set `shuttingDown = true` — stop accepting new cron triggers
2. Wait for in-flight LLM calls to complete (track with Promise)
3. Drain and close connection pool (`pool.end()`)
4. Exit. Hard kill timeout: 30s.

### Environment

| Var | Required | Description |
|-----|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | Claude API key |
| `GEMINI_API_KEY` | Yes | Google Gemini API key for embeddings (free tier: 1500 req/day) |
| `DISCORD_TOKENS` | No | Comma-separated user tokens (required for Discord sources) |
| `TWITTERAPI_KEY` | No | twitterapi.io API key for Twitter scraping |
| `DATABASE_URL` | Yes | Postgres connection string (e.g., `postgres://user:pass@localhost:5432/podders`) |
| `API_KEY` | No | Dashboard API key (auto-generated if missing) |
| `PORT` | No | Default 3000 |
| `DATA_DIR` | No | Backups location, default `./data` |
| `PUBLIC_URL` | No | Dashboard URL for webhook links (e.g., `https://podders.yourdomain.com`) |
| `ALERT_WEBHOOK_URL` | No | Discord webhook for critical health alerts (separate channel from delivery) |

## 9. Testing Strategy

1. **Golden file tests** — capture real LLM responses as fixtures. Test JSON parsing, schema validation, entity extraction into DB. No API calls in CI.
2. **Scraper mocks** — each source adapter gets a mock returning canned data. Test normalize pipeline against known inputs.
3. **Integration tests** — `test/integration/`, skipped in CI. Feed 50 real items through full pipeline, assert valid report output.
4. **Prompt regression** — when prompts change, run against 3 saved inputs, diff outputs. Human review. Stored in `test/prompts/`.

## Tech Stack

| Layer     | Choice                | Why                              |
|-----------|-----------------------|----------------------------------|
| Runtime   | Node.js / TypeScript  | ESM, already known               |
| Server    | Fastify v5            | Fast, typed, serves API + dashboard |
| Database  | Postgres (pg / node-postgres) | Async, connection pool, concurrent writes, full-text search via tsvector |
| Dashboard | Vite + React 19 + React Router | SPA, no SSR needed       |
| Styling   | Tailwind CSS 4        | Utility-first, dark theme, responsive |
| LLM       | @mariozechner/pi-ai   | Multi-provider, wrapped in llm.ts. Models swappable across providers |
| LLM validation | zod              | LLM output parsing + enforcement |
| API validation | Fastify JSON Schema | Route-level input validation    |
| Twitter   | twitterapi.io          | Pay-per-use, $0.15/1K tweets     |
| Auth      | argon2                | API key hashing                  |
| Discord   | ws                    | Gateway WebSocket connections    |
| RSS       | rss-parser              | RSS/Atom feed parsing              |
| News      | @mozilla/readability + linkedom | Article content extraction |
| Language  | franc                 | Language detection               |
| IDs       | ulid                  | Sortable unique IDs              |
| Scheduling| node-cron             | Time-based job triggers          |
| CLI       | commander             | CLI entry point                  |
| Embeddings | @google/generative-ai | Gemini text-embedding-004 (768 dims) |
| Deploy    | pm2 / systemd         | VPS process manager              |

## File Structure

```
podders/
├── src/
│   ├── index.ts              # CLI entry (commander)
│   ├── config.ts             # Env loading + validation
│   ├── server.ts             # Fastify: API routes + static dashboard + schema validation
│   ├── logger.ts             # Pino setup + secret masking
│   ├── url-validator.ts      # Shared SSRF protection
│   ├── db/
│   │   ├── connection.ts     # Postgres connection pool (pg.Pool)
│   │   ├── migrations.ts     # Schema creation + versioned migrations
│   │   └── queries.ts        # All queries (async)
│   ├── ingest/
│   │   ├── types.ts          # RawItem, SourceAdapter interface
│   │   ├── discord.ts        # Discord Gateway (multi-token, per-connection)
│   │   ├── twitter.ts        # twitterapi.io REST polling
│   │   ├── news.ts           # Article extractor (readability)
│   │   └── rss.ts            # RSS/Atom poller
│   ├── normalize/
│   │   ├── dedup.ts          # Content hash + URL dedup
│   │   ├── filter.ts         # Spam/noise rules
│   │   └── index.ts          # Normalize orchestrator
│   ├── process/
│   │   ├── summarize.ts      # Stage 1: Haiku micro-batch + entity extraction
│   │   ├── correlate.ts      # Stage 2: SQL cross-source signals
│   │   └── synthesize.ts     # Stage 3: Sonnet daily/flash/pulse reports
│   ├── knowledge/
│   │   ├── entities.ts       # Upsert entities + aliases + mentions
│   │   └── decay.ts          # Daily relevance decay
│   ├── deliver/
│   │   └── webhook.ts        # Discord webhook (3 retries, backoff)
│   ├── llm.ts                # Pi AI wrapper (error taxonomy, prompt injection defense, chunk-split retry)
│   └── scheduler.ts          # node-cron jobs + mutex + startup sequence
├── dashboard/
│   ├── src/
│   │   ├── main.tsx          # Entry point
│   │   ├── router.tsx        # React Router: /, /reports, /chat, /settings
│   │   ├── pages/
│   │   │   ├── ReportView.tsx    # Latest report (/) and report detail (/reports/:id)
│   │   │   ├── ReportList.tsx    # Report archive (/reports)
│   │   │   ├── Chat.tsx          # RAG chat — question/answer with source citations (/chat)
│   │   │   └── Settings.tsx      # Source management, config, health (/settings)
│   │   ├── components/       # Shared UI (empty states, report cards, sentiment bars)
│   │   └── lib/
│   │       ├── api.ts        # fetchApi wrapper + auth
│   │       └── types.ts      # Shared types
│   ├── index.html
│   ├── vite.config.ts
│   ├── app.css              # Tailwind v4 @theme config (colors, fonts)
│   └── tsconfig.json
├── test/
│   ├── unit/                 # Golden file + normalize tests
│   ├── integration/          # Full pipeline tests (manual)
│   └── prompts/              # Prompt regression inputs
├── data/                     # Backups (pg_dump)
├── .env
├── package.json
├── tsconfig.json
├── CLAUDE.md
├── ARCHITECTURE.md           # This file
├── PIPELINE.md               # Prompt engineering + pipeline details
└── docs/
    ├── DASHBOARD.md          # Wireframes, components, dark theme
    ├── DATAFLOW.md           # ASCII data flow diagrams
    ├── DEPLOYMENT.md         # Hosting, TLS, monitoring, backups, ops
    ├── DISCORD.md            # Gateway spec, multi-token, close codes
    ├── EMBEDDINGS.md         # Vector embeddings: model, storage, pipeline, cost
    ├── ENTITIES.md           # Resolution, CoinGecko seeding, decay, pruning
    ├── ERRORS.md             # Error taxonomy, circuit breakers, recovery
    ├── INTEGRATION.md        # End-to-end message→report flow with pseudocode
    ├── LLM.md               # llm.call() wrapper spec, retry tree, cost calc
    ├── QUERIES.md            # Every SQL query with TypeScript signatures
    ├── ROADMAP.md            # Feature roadmap: v2/v3/research
    ├── RSS_NORMALIZE.md      # RSS ingestion + normalize pipeline + spam rules
    ├── SCHEDULER.md          # Startup, job table, mutex, call chain, shutdown
    ├── SECURITY.md           # Prompt injection defenses, residual risk
    ├── TESTING.md            # Test structure, golden files, 42 API test cases
    ├── TWITTER.md            # twitterapi.io REST, data mapping, polling, cost
    └── WEBHOOK.md            # Discord embed format, mobile rules, mockups
```

## What's NOT in Scope (MVP)

- On-chain data / price feeds (v2.1)
- SSE / real-time updates (v2.1)
- Entity explorer (v2.1)
- Pretext engine (v2.1)
- Email / web push delivery (v2.1)
- OAuth / multi-user (v2.1)
- Redis / message queues
- Vector embeddings — **Phase 1 core capability**, see [docs/EMBEDDINGS.md](./docs/EMBEDDINGS.md)
- Plugin/skill system

## Cost Model

At 5,000 items/day post-normalization:

| Component | Calls/day | Cost |
|-----------|-----------|------|
| Stage 0 (code) | — | $0 |
| Stage 1 (Haiku, 12 batches) | ~30 | ~$0.05 |
| Stage 2 (SQL) | 12 | $0 |
| Stage 3 (Sonnet, daily + 8 pulse + rare flash) | 9-10 | ~$0.90 |
| Chat (Sonnet, user-driven) | ~10-20 | ~$0.15-0.30 |
| Twitter (twitterapi.io) | ~2K tweets | ~$0.30/day |
| **Total** | | **~$1.55/day** |

## Build Order

1. Database schema + migrations (core tables: items, summaries, reports, sources, source_state, source_rate_history, app_config, embeddings, health_events)
2. `llm.ts` wrapper (error taxonomy, retry, token counting)
3. `embed.ts` wrapper (Gemini text-embedding-004, retry, cost logging)
4. RSS ingestion (simplest source — `rss-parser` + normalize)
5. Normalize pipeline (dedup, filter, language detection, truncation)
6. Pre-summarize (Haiku compression for low-density RSS/news)
7. Stage 1 summarize (entities as JSON in summary body, no separate tables) + summary embedding (inline after Stage 1)
8. Stage 3 synthesize + report generation + report embedding (inline after Stage 3)
9. Market pulse (Sonnet, 3h cron, scaled output, Discord webhook with muted grey embed)
10. Discord webhook delivery
11. Scheduler (cron jobs + pulse cron + mutex + crash recovery)
12. In-memory vector cache for summaries + reports (load on startup)
13. Semantic search endpoint (`GET /api/v1/search?mode=semantic`)
14. RAG chat endpoint (`POST /api/v1/chat`) — embed question via Gemini, retrieve top 10 from vector cache, pass to Sonnet with conversation history, return grounded answer with source citations
15. Discord ingestion + source_state + Gateway multi-token + Poisson claim gate (`source_rate_history`)
16. Twitter/X via twitterapi.io
17. News article extraction (Readability)
18. Stage 2 correlate (SQL cross-source signals — needs multiple sources)
19. Dashboard (Vite scaffold + report view + settings + chat panel + search UI toggle)

Steps 15-17 are independent and can be built in parallel.

20. Entity tables (entities, entity_aliases, entity_mentions) + resolution from summary JSON
21. CoinGecko seeding + Indonesian entity seeds
22. Entity relevance decay + pruning cron
23. Item embedding (batch cron every 15 min) + entity embedding (inline on create/update)
24. Postgres full-text search (tsvector/tsquery) + search endpoint (keyword fallback)
25. `llm_usage` table + cost tracking
26. Data retention cron + backup cron (cascade embedding deletes with item retention)
27. Observability (cost dashboard, pipeline status, secret masking)
28. Health monitoring (`health_events` table, 7 checks every 5min, alert webhook, process heartbeat watchdog)
29. Discovery endpoints (Discord guilds, RSS auto-detect)
30. Graceful shutdown + Caddy TLS + pm2/systemd deploy

**Goal**: all sources ingesting, entity resolution working, dashboard live, semantic search and RAG chat operational. Everything ships together.
