# Podders v2 — Architecture Documentation

24/7 market intelligence engine for crypto and Indonesian tradfi markets. Scrapes social media (Discord, Twitter/X) and news sources in Indonesian and English, builds knowledge via multi-stage LLM pipeline, surfaces market conditions through flash reports, 3-hour pulses, daily digests, and a dashboard with RAG chat.

## What This Repo Is

This is the architecture documentation for Podders v2. No code yet — just the complete design.

## Reading Order

1. **[PRODUCT.md](./PRODUCT.md)** — Why this exists, who it's for
2. **[CLAUDE.md](./CLAUDE.md)** — Tech stack, conventions, project structure
3. **[ARCHITECTURE.md](./ARCHITECTURE.md)** — Full schema, API contract, build order
4. **[PIPELINE.md](./PIPELINE.md)** — LLM prompts, chunking, validation, scheduling

## Reference Docs

| Doc | What It Covers |
|-----|---------------|
| [INTEGRATION.md](./docs/INTEGRATION.md) | End-to-end message flow — how a message becomes a report |
| [DATAFLOW.md](./docs/DATAFLOW.md) | ASCII data flow diagrams |
| [DISCORD.md](./docs/DISCORD.md) | Gateway lifecycle, multi-token management |
| [TWITTER.md](./docs/TWITTER.md) | twitterapi.io REST API, polling, cost model |
| [RSS_NORMALIZE.md](./docs/RSS_NORMALIZE.md) | RSS ingestion, normalize pipeline |
| [DASHBOARD.md](./docs/DASHBOARD.md) | Wireframes, component inventory, dark theme |
| [EMBEDDINGS.md](./docs/EMBEDDINGS.md) | Embedding architecture, model choice, storage |
| [ENTITIES.md](./docs/ENTITIES.md) | Entity resolution, CoinGecko seeding, decay |
| [SCHEDULER.md](./docs/SCHEDULER.md) | Startup sequence, job table, mutex |
| [WEBHOOK.md](./docs/WEBHOOK.md) | Discord embed format, daily/flash/pulse mockups |
| [DEPLOYMENT.md](./docs/DEPLOYMENT.md) | Hosting, TLS, monitoring, backups |
| [ERRORS.md](./docs/ERRORS.md) | Error taxonomy, circuit breakers, recovery |
| [LLM.md](./docs/LLM.md) | llm.call() signature, retry logic, cost model |
| [QUERIES.md](./docs/QUERIES.md) | Every SQL query with TypeScript signatures |
| [SECURITY.md](./docs/SECURITY.md) | Prompt injection defenses, input sanitization |
| [TESTING.md](./docs/TESTING.md) | Test structure, golden files, test cases |
| [ROADMAP.md](./docs/ROADMAP.md) | Feature roadmap: v2, v3, research |

## The Pipeline

```
Sources (Discord, Twitter, RSS, News)
  → Ingest (WebSocket / REST poll / feed poll)
  → Normalize (truncate, dedup, spam, language)
  → Pre-Summarize (compress low-density articles)
  → Stage 1: Summarize (Haiku per chunk → entities, events, urgency)
  → Stage 2: Correlate (SQL cross-source entity matching)
  → Pulse (Sonnet every 3h, scales with activity)
  → Daily Synthesis (Sonnet, full report from pulses)
  → Flash Reports (instant on breaking events, 2+ source corroboration)
  → Delivery (Discord webhook + dashboard + RAG chat)
```

## Tech Stack

- **Runtime:** Node.js 22 + TypeScript (ESM, strict)
- **Server:** Fastify v5
- **Database:** PostgreSQL via pg (node-postgres)
- **Dashboard:** Vite + React 19 + Tailwind CSS 4
- **LLM:** Claude (Haiku for Stage 1, Sonnet for synthesis/pulse/chat)
- **Embeddings:** Google Gemini text-embedding-004
- **Deploy:** pm2 or systemd on VPS
