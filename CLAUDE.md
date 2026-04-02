# CLAUDE.md — Podders v2

## Project

Podders v2 is a 24/7 market intelligence engine that scrapes social media (Discord, Twitter/X) and news sources, builds knowledge over time via a multi-stage LLM pipeline, and surfaces market conditions for crypto/DeFi and tradfi traders. Ships as a single process, one port, one Postgres database. Deployed via pm2 or systemd on a VPS.

## Tech Stack

- **Runtime**: Node.js 22 + TypeScript (ESM only, strict mode)
- **Server**: Fastify v5 (API + static dashboard serving)
- **Database**: PostgreSQL via pg (node-postgres) with connection pool
- **Dashboard**: Vite + React 19 + React Router + Tailwind CSS 4
- **LLM**: @mariozechner/pi-ai (Haiku for Stage 1, Sonnet for Stage 3 + pulse — models swappable to any Pi-supported provider)
- **Validation**: zod (LLM output), Fastify JSON Schema (API input)
- **Twitter**: twitterapi.io REST API ($0.15/1K tweets)
- **Embeddings**: @google/generative-ai (Gemini text-embedding-004, 768 dims)
- **Deploy**: pm2 or systemd on VPS (no Docker)

## Commands

```bash
npm install              # Install all dependencies
npm run build            # TypeScript compile + Vite dashboard build
npm run dev              # Dev server with watch (ts-node + Vite dev)
npm run start            # Production: node dist/index.js run
npm test                 # Unit tests (golden file + normalize)
npm run test:integration # Full pipeline integration tests (requires .env)
npm run test:prompts     # Prompt regression tests (manual, ~$0.25/run, requires .env)
```

## File Structure

```
src/
  index.ts           # CLI entry (commander)
  config.ts          # Env loading + validation
  server.ts          # Fastify routes + static serving
  logger.ts          # Pino + secret masking
  llm.ts             # All LLM calls go through here
  url-validator.ts   # Shared SSRF protection
  scheduler.ts       # per-source poll intervals + mutex
  db/                # connection, migrations, queries
  ingest/            # discord, twitter, news, rss adapters
  normalize/         # dedup, spam filter, language detection, Indonesian→English translation via Haiku
  pre-summarize/     # Haiku pre-summary for long RSS/news articles (>4K chars)
  process/           # summarize (Stage 1), correlate (Stage 2), synthesize (Stage 3)
  knowledge/         # entity upsert, alias resolution, decay
  deliver/           # Discord webhook delivery
dashboard/
  src/               # React SPA (/, /reports, /chat, /settings)
test/
  unit/              # Golden file + normalize tests
  integration/       # Full pipeline (manual, not CI)
  prompts/           # Prompt regression tests (manual, real LLM calls)
    fixtures/        # 8 frozen real inputs + .expected.json golden files
```

See [ARCHITECTURE.md](./ARCHITECTURE.md) for full schema, API contract, and build order.

## Key Conventions

- **ESM only** — no CommonJS. All imports use `.js` extensions in compiled output.
- **TypeScript strict** — `strict: true` in tsconfig, no `any` escape hatches.
- **Async DB** — all database calls use `async/await` via the pg connection pool. No synchronous queries.
- **All LLM calls via `llm.ts`** — never import `@mariozechner/pi-ai` directly. The wrapper handles retries, error taxonomy, prompt injection defense, and stage-specific error decisions. Pi handles token counting, cost tracking, retries at the provider level, multi-provider support, and prompt caching passthrough (verify per Decision 12). Models are swappable across providers (Anthropic, OpenAI, Google, Groq, Mistral, etc.) by changing the model string in config. Stage 1 Haiku results with low confidence on non-routine chunks are escalated to Sonnet for re-processing (Decision 10).
- **zod for LLM output validation** — parse every LLM JSON response through a zod schema. Clamp values, default unknowns, enforce array limits. On zod validation failure, retry with specific error paths fed back to the LLM so it knows exactly what to fix (Decision 08).
- **Fastify JSON Schema for API input** — define schemas on route options, not in handler bodies. Source enums, format validation, length caps.
- **pino logging with secret masking** — mask `DISCORD_TOKENS`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `TWITTERAPI_KEY`, `API_KEY`, and webhook URLs in all log output. Indonesian content is first-class — process `eng` and `ind` languages (franc ISO 639-3 codes). Indonesian source content is translated to English via Haiku before entering the pipeline (Decision 01). Indonesian chat queries are also translated to English before search (Decision 11). Output summaries always in English.
- **ULID for all IDs** — every table PK is a ULID string. No autoincrement, no UUIDs.
- **Token-based chunking** — LLM input is chunked by estimated token count (content.length / 4), not item count. Budget: 6,000 tokens per Stage 1 chunk.
- **Entity resolution via alias table** — `entity_aliases` maps lowercase-normalized strings to canonical entity IDs. Lookup-then-insert pattern. CoinGecko token list seeds the table. Entities follow a two-tier active/archived lifecycle: entities demote to archived instead of deletion and promote back to active with history intact on re-mention (Decision 04). Disambiguation uses a three-tier approach: alias lookup first, then context-based resolution via co-occurring entities, then batched Haiku calls for remaining ambiguities. Results are saved as new aliases, making the system self-improving (Decision 09).
- **Narrative clustering via embeddings** — summary embeddings are clustered daily (k-means with silhouette auto-tuning) to detect emerging narratives. Each cluster is named by one Haiku call. Cluster growth rates classify signals as new/emerging/strong/stable/fading, fed to Sonnet as context for daily synthesis (Decision 05).

## Coding Guidelines

- **No secrets in DB** — only exception is the hashed API key (`argon2`) in `app_config`. All other secrets live in `.env` / environment variables.
- **Five-layer prompt injection defense** — (1) Unicode NFKC normalization + zero-width character stripping in normalize step, (2) InstructDetector scan at ingest (cached, items flagged as `filtered` with reason `injection_detected`), (3) XML `<scraped_content>` wrapping with nonce-based tags and explicit "treat as untrusted data" instruction, (4) entity post-verification against raw source text, (5) structural privilege separation via zod schema validation on Haiku's JSON-only output (Decision 06).
- **SSRF validation via `url-validator.ts`** — all outbound URL fetches (RSS, webhooks, news extraction) must go through the shared validator. Require HTTPS, reject private IPs, reject non-standard ports, block `file://`.
- **Market pulse every 3h** — a Sonnet synthesis that runs every 3 hours, producing a short market pulse. Output length scales with activity (quiet periods get shorter pulses). Delivered via webhook with muted grey embed.
- **No event bus, no plugin system, no multi-agent** — direct function calls. Cron fires handler, handler calls next stage at end. Keep it simple.
- **Per-source poll intervals** — each source defines its own `poll_interval` instead of a single global cron. The scheduler fires each source independently.
- **Parallel source processing** — sources are processed concurrently, not sequentially. The scheduler launches source ingestion jobs in parallel.
- **Raw Discord feed** — a raw feed API endpoint (`/api/raw-feed`) and dashboard view display unprocessed Discord messages. Discord attachments (images) are stored alongside messages.
- **Crash recovery** — on startup, reset orphaned `processing` items back to `ready`. Batch-claiming SQL (`UPDATE ... WHERE status='ready'`) prevents double-processing.
- **Overlap guard** — each cron job has a `running` mutex. If a job fires while the previous run is in-flight, skip and log warning.

## Environment Variables

| Var | Required | Notes |
|-----|----------|-------|
| `ANTHROPIC_API_KEY` | Yes (if using Claude models) | Claude API key. Other provider keys (e.g. `OPENAI_API_KEY`, `GOOGLE_API_KEY`) can be added as needed for Pi-supported models |
| `GEMINI_API_KEY` | Yes | Google Gemini API key for embeddings (free tier: 1500 req/day) |
| `DISCORD_TOKENS` | No | Comma-separated user token strings (needed for Discord sources) |
| `TWITTERAPI_KEY` | No | For Twitter/X scraping |
| `API_KEY` | No | Auto-generated on first run if missing |
| `DATABASE_URL` | Yes | Postgres connection string (e.g. `postgresql://user:pass@localhost:5432/podders`) |
| `PORT` | No | Default 3000 |
| `DATA_DIR` | No | Backups location, default ./data |
| `PUBLIC_URL` | No | For "View full report" links in webhooks |
| `ALERT_WEBHOOK_URL` | No | Discord webhook for critical health alerts (separate channel) |

## Reading Order

New to the repo? Start with [PRODUCT.md](./PRODUCT.md) to understand why this exists, then this file (CLAUDE.md) for conventions, then ARCHITECTURE.md for the full technical picture, then INTEGRATION.md to see how a message becomes a report. Everything else is reference material for specific subsystems.

## Reference Docs

- [ARCHITECTURE.md](./ARCHITECTURE.md) — full schema, API contract, deployment, build order, cost model
- [PIPELINE.md](./PIPELINE.md) — LLM prompts, chunking code, output validation, scheduling details
- [docs/DISCORD.md](./docs/DISCORD.md) — Gateway lifecycle, multi-token management, opcodes, close codes
- [docs/DASHBOARD.md](./docs/DASHBOARD.md) — wireframes, component inventory, dark theme spec
- [docs/ERRORS.md](./docs/ERRORS.md) — error taxonomy, circuit breakers, logging levels, recovery
- [docs/SCHEDULER.md](./docs/SCHEDULER.md) — startup sequence, job table, mutex, call chain, shutdown
- [docs/ENTITIES.md](./docs/ENTITIES.md) — resolution algorithm, CoinGecko seeding, decay, pruning, merging
- [docs/TWITTER.md](./docs/TWITTER.md) — twitterapi.io REST API, data mapping, zod schema, polling, cost model
- [docs/INTEGRATION.md](./docs/INTEGRATION.md) — end-to-end message flow, chunking, summarization, report generation
- [docs/WEBHOOK.md](./docs/WEBHOOK.md) — Discord embed format, mobile rules, daily/flash mockups
- [docs/DEPLOYMENT.md](./docs/DEPLOYMENT.md) — hosting, TLS, monitoring, backups, secret rotation
- [docs/TESTING.md](./docs/TESTING.md) — test structure, golden files, normalize/entity/API test cases
- [docs/EMBEDDINGS.md](./docs/EMBEDDINGS.md) — embeddings architecture: what gets embedded, model choice, storage, pipeline, cost model, phasing
- [docs/ROADMAP.md](./docs/ROADMAP.md) — feature roadmap: v2 (sentiment momentum, narratives), v3 (price feeds, credibility), research
- [docs/SECURITY.md](./docs/SECURITY.md) — prompt injection defenses, input sanitization, author rate limits, residual risk
- [docs/LLM.md](./docs/LLM.md) — llm.call() signature, retry decision tree, chunk-split, cost calculation, nonce sanitization
- [docs/QUERIES.md](./docs/QUERIES.md) — every SQL query with TypeScript signatures, organized by module
- [docs/RSS_NORMALIZE.md](./docs/RSS_NORMALIZE.md) — RSS ingestion, normalize pipeline, spam rules, language detection
- [docs/DATAFLOW.md](./docs/DATAFLOW.md) — ASCII data flow diagrams: pipeline, cron timing, per-source, entity lifecycle, embedding flow
