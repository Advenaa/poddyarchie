# Podders v2

24/7 market intelligence for crypto and Indonesian markets. No code yet — this is the full architecture.

---

## What Is This

You follow 50+ channels across Discord, Twitter, Telegram, and news — in Indonesian and English. You can't read all of them. You skim 5-6, miss things, and find out about a narrative shift from a friend's WhatsApp three hours too late.

Podders watches everything, all the time. It reads Indonesian Discord alpha groups AND English CT AND macro news. It compresses thousands of messages into reports you can read in 30 seconds. If something breaks, your phone buzzes immediately.

The Indonesian language processing is the moat. No competitor does this. Indonesia has 20M+ crypto holders and zero tools that understand Bahasa content.

---

## How It Works — The Full Story

### Step 1: Messages Arrive

Podders holds connections to all your sources:
- **Discord** — WebSocket, messages stream in real-time
- **Twitter/X** — REST polling on each account's own schedule
- **RSS** — Feed polling on its own schedule
- **News** — Article extraction from URLs

Each source has its own poll interval. Alpha Discord: 15 minutes. General Twitter: 1 hour. Slow RSS: 4 hours.

Every message gets converted to the same format — author, content, timestamp, engagement, attachments. Doesn't matter where it came from. From here on, a message is a message.

### Step 2: Normalize (The Bouncer)

Every message hits five checks. No AI, just rules:

1. **Too long?** Over 20K characters gets cut. Only hits RSS/news articles — Discord has a 2K char limit.
2. **Seen before?** Hash the content, check the database. Duplicate? Drop it.
3. **Same URL from another source?** Expand shortened links (t.co, bit.ly), check if another source already posted it.
4. **Spam?** Rule-based. "gm", single emoji, airdrop copypasta. Kills 40-60% of Discord volume.
5. **Wrong language?** English and Indonesian only. Japanese, Korean, etc. get dropped.

Pass all five → saved as `ready`, sits in the database.
Fail any → saved as `filtered`, kept for debugging but never processed.

### Step 3: Pre-Summarize

Only touches RSS and news articles. Discord and Twitter skip this entirely — they're already short.

Decision tree:
- **Looks important** — urgency keywords like "exploit", "rate decision", "flash crash", or packed with entity names? Don't touch it. Pass the full article through.
- **Looks fluffy** — lots of filler, background paragraphs, author bios, boilerplate? Compress it with Haiku to ~300 tokens.

Two different compression prompts: one for regulatory news (keep exact rule numbers, dates, penalties), one for general news (keep entities, numbers, quotes, causal claims).

Articles get batched 3-5 per Haiku call to save cost. The first 200 tokens of the original are kept as a `content_anchor` for embedding quality.

### Step 4: Claim and Chunk

The scheduler ticks every 60 seconds, checking which sources are due for processing based on their individual poll intervals.

When sources are due, they get processed **in parallel** — not one at a time. For each source:

1. **Claim** — one SQL statement marks all `ready` items as `processing` with a batch ID. Locked. No other process can touch them.
2. **Load** — query items by batch ID, sort by timestamp.
3. **Chunk** — walk through items in order, pack into groups of 6,000 tokens each. A short tweet is ~50 tokens, a long post ~400. Keep packing until the budget is hit, then start a new chunk.

### Step 5: Haiku Summarizes

Each chunk gets sent to Claude Haiku. It reads the raw messages (which might be in Indonesian) and returns structured JSON in English:

- **summary** — 200-500 words of what was discussed
- **urgency** — `routine` (normal chatter), `elevated` (notable), or `breaking` (exploit, crash, regulatory action, rate decision)
- **entities** — up to 20, each with name, aliases, type (token/person/project/company/event), mention count, sentiment score (-1 to 1)
- **keyEvents** — up to 5 factual bullet points

Then three safety checks:
- **Validation** — is the JSON valid? Sentiment clamped, arrays within limits. If not, retry once.
- **Entity post-verification** — does "Wormhole" actually appear in the raw text? Hallucinated entities get dropped.
- **Quality gate** — zero entities, zero events, short summary? Marked as low quality, excluded from reports.

`maxTokens: 3000` — gives Haiku plenty of room. Typical output is 800-1200 tokens.

### Step 6: Flash Check

Did any chunk come back with `urgency: "breaking"`?

No → move on.
Yes → check if it's corroborated:
- **2+ different sources** said breaking → fire flash report
- **3+ chunks from same source** said breaking → fire flash report
- **1 chunk, 1 source** → log it, don't fire. Could be someone overreacting.

If it fires, Sonnet gets called immediately with just the last 4 hours of summaries. Produces a 2-sentence TL;DR. Delivered to Discord webhook with `[FLASH]` prefix, orange embed. 30-minute cooldown before another flash can fire.

### Step 7: Correlation

One SQL query, no AI. Finds entities mentioned in 2+ different sources in the last 24 hours.

"Wormhole appeared in Discord Server A, Server C, and Twitter — 3 sources, average sentiment -0.85."

Single-source mentions don't make the list. Cross-source confirmation is the signal.

### Step 8: Pulse (Every 3 Hours)

Sonnet writes a market pulse based on recent summaries. **Output scales with activity:**

- **Quiet** (1-3 summaries, all routine) → 2-3 sentences. "Nothing significant happened."
- **Moderate** (4-10 summaries, some elevated) → a paragraph with key events listed.
- **Chaos** (10+ summaries or any breaking) → full short report with sections and entity sentiment.

Each pulse carries the previous pulse's TL;DR as context. By the end of the day, it's a running narrative.

Delivered via Discord webhook with muted grey embed. Your phone buzzes, you read a few sentences, you know what's happening.

### Step 9: Daily Synthesis

At 9 AM Jakarta time, Sonnet reads the 8 pulses from the day + correlated entities + top 10 raw messages as ground truth (7 by engagement, 3 reserved for RSS/news since RSS has no engagement score).

Produces the full `MarketReport`:
- **TL;DR** — 3 sentences. First sentence = the single most important thing.
- **Key Events** — up to 10, each citing which sources reported it. Single-source claims marked `[unconfirmed]`.
- **Entity Sentiment** — up to 15 entities with sentiment and trend (rising/falling/stable/new).
- **Sections** — 2-4 themed sections titled by topic ("DeFi Protocol Activity"), not by source.
- **New Projects** — entities not seen before, with context.

The prompt explicitly tells Sonnet to state causal connections between related events — "OJK eased capital rules in response to BI's rate hold" — not just list them as separate items.

Quality gate: if there are zero key events across all pulses, skip. No empty reports.

### Step 10: Delivery

Reports go to Discord as embeds:
- **Daily** → blue embed
- **Flash** → orange embed
- **Pulse** → grey embed

If Discord returns an error: retry at 2s, 8s, 32s. All retries fail → mark as `failed`, surface on the settings page.

Everything is also saved to the database and visible on the dashboard.

### Step 11: Chat

Dashboard has a chat interface. You type "what happened with SOL this week?" and it:

1. Embeds your question into a vector
2. Searches summaries and reports by meaning (semantic search)
3. Feeds the top 10 results to Sonnet as context
4. Sonnet answers from your data with source citations

Follow-up questions work. "Why is sentiment dropping?" → "Compare it to last month." Conversation history stays in memory.

It only answers from what the pipeline has collected. No web search, no speculation. If it doesn't have data, it says so.

---

## How It Builds Knowledge

### Entity Memory

Every time Haiku reads a batch, it extracts entities — tokens (ETH, SOL), people (CZ, Vitalik), projects (Uniswap), companies (Coinbase), events (exploits). These get stored with an alias system: `$ETH`, `ETH`, `eth`, `Ethereum` all point to the same thing. Pre-loaded with ~10K tokens from CoinGecko + Indonesian institutions (OJK, Bank Indonesia, Indodax).

### Relevance Decay

Every entity has a relevance score. Goes up when mentioned, **decays 5% per day** automatically. News mentions weigh more than Discord (rarer = higher signal). Not mentioned for 90 days → faded to near-zero → eventually pruned. The system naturally tracks what's trending vs what's dead.

### Cross-Source Correlation

"Wormhole" discussed in 3 Discord servers AND on Twitter AND in a news article → strong signal. One person mentioning it once → maybe noise. The system cross-references across all sources and languages.

### Embeddings

Every summary, report, and entity gets turned into a vector (768 numbers via Google Gemini). This enables semantic search — find things by meaning, not just keywords. "Bridge security incidents" finds results about Wormhole, Ronin, Nomad even if none of them say those exact words.

---

## What Gets Delivered

| Surface | When | What |
|---------|------|------|
| **Flash report** | Instant (on breaking events) | 2-sentence TL;DR, key event, orange Discord embed |
| **Pulse** | Every 3 hours | 2 sentences to full short report, scales with activity |
| **Daily digest** | 9 AM Jakarta | Full report: TL;DR, events, sentiment, sections, new projects |
| **Dashboard** | Anytime | All reports, raw Discord feed, search, chat |

---

## Tech Stack

| Layer | Choice |
|-------|--------|
| Runtime | Node.js 22 + TypeScript (ESM, strict) |
| Server | Fastify v5 |
| Database | PostgreSQL via pg (node-postgres) |
| Dashboard | Vite + React 19 + Tailwind CSS 4 |
| LLM | Claude — Haiku for Stage 1, Sonnet for synthesis/pulse/chat |
| Embeddings | Google Gemini text-embedding-004 (768 dims, free tier) |
| Twitter | twitterapi.io REST API |
| Deploy | pm2 or systemd on VPS |

Single process, one port. No Docker, no microservices.

---

## Cost

~$1.55/day (~$47/month) for the full system:
- Stage 1 Haiku calls: ~$0.15/day
- 8 Sonnet pulses: ~$0.90/day
- Daily synthesis: ~$0.05/day
- Flash reports: ~$0.05/day
- Chat: ~$0.15-0.30/day
- Embeddings: $0 (Gemini free tier)

---

## What's Next (After the Build)

**v2 features** (buildable on current stack):
- Sentiment momentum — is ETH getting more bearish or less?
- Narrative clustering — track themes like "RWA rotation" or "memecoin season"
- Event chains — link exploit → audit → governance vote over time
- Regional divergence — Indonesian community feels differently than English CT
- Cycle detection — FOMC dates, token unlocks on a calendar

**v3 features** (needs new data sources):
- Contrarian detection — sentiment bearish but price up
- Influencer tracking — who said it first
- Cross-market correlation — crypto sentiment vs VIX/DXY
- Macro regime detection — risk-on / risk-off

---

## Docs

| Doc | What |
|-----|------|
| [PRODUCT.md](./PRODUCT.md) | Why this exists, who it's for |
| [CLAUDE.md](./CLAUDE.md) | Tech stack, conventions, structure |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Full schema, API, build order |
| [PIPELINE.md](./PIPELINE.md) | LLM prompts, chunking, validation |
| [docs/INTEGRATION.md](./docs/INTEGRATION.md) | End-to-end message flow |
| [docs/DATAFLOW.md](./docs/DATAFLOW.md) | ASCII data flow diagrams |
| [docs/ROADMAP.md](./docs/ROADMAP.md) | Feature roadmap |
| [docs/](./docs/) | Everything else |
