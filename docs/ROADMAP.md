# Podders — Feature Roadmap

## How to Read This

Each feature is classified by **what it actually requires**:

- **Prompt change** — new/modified LLM prompt, no schema or data changes
- **New data** — needs a new data source, API integration, or external feed
- **New tables** — needs schema additions to the existing Postgres DB
- **New architecture** — requires fundamentally different infrastructure (streaming, distributed compute)
- **Research problem** — no known reliable solution; requires experimentation

Features are placed into phases based on dependencies, not ambition.

---

## Phase 1 — Full Build

Defined in ARCHITECTURE.md build order. Included here for completeness. Everything ships together.

- DB schema + migrations
- LLM wrapper (llm.ts)
- RSS ingestion + normalize pipeline
- Pre-summarize as separate step (decision tree: skip Discord/Twitter, skip urgent/dense RSS, compress low-density only)
- Stage 1 summarize (Haiku, micro-batch)
- Stage 3 synthesize (Sonnet, daily report + market pulse)
- Market pulse (Sonnet, every 3h, output scales with activity)
- Discord webhook delivery
- Scheduler with per-source poll intervals, mutex + crash recovery
- Parallel source processing (via Promise.allSettled, enabled by Postgres)
- Raw Discord feed with image attachments (live mirror on dashboard)
- Embeddings (summary + report embedding, per EMBEDDINGS.md)
- Discord Gateway ingestion (multi-token)
- Twitter/X via twitterapi.io
- News article extraction (Readability)
- Stage 2 cross-source correlation (SQL)
- Dashboard (report view, settings, empty states)
- Entity tables + resolution + CoinGecko seeding
- Entity relevance decay + pruning
- Postgres full-text search (tsvector/tsquery)
- LLM cost tracking
- Data retention + backup crons
- RAG chat (embed question via Gemini, retrieve from vector cache, Sonnet answers grounded in user's data)
- Discovery endpoints (Discord guilds, RSS auto-detect)
- Graceful shutdown + pm2/systemd deploy

---

## v2 Features (Buildable on Current Architecture)

These features work with the existing stack: Postgres, the current pipeline stages,
the current source adapters. They need new tables, new prompts, or new API
integrations — but nothing fundamentally different.

### 2.1 Sentiment Momentum
**What**: Track rate-of-change of sentiment per entity, not just point-in-time.
Flag "sentiment accelerating negative" or "reversal detected."

**Requires**: Prompt change + new table
- `entity_mentions` already stores per-mention sentiment. Add a materialized
  `entity_sentiment_daily` rollup table (entity_id, date, avg_sentiment,
  mention_count, momentum). Compute via daily cron after Stage 1.
- Momentum = weighted delta of last 3 days vs prior 7 days. Pure SQL.
- Surface in Stage 3 prompt as context: "ETH sentiment momentum: -0.3 (declining)."
- Dashboard: sparkline per entity on report view.

**Dependencies**: Phase 1 entity tables must exist.
**Feasibility**: Straightforward. This is a SQL aggregation + prompt injection. 1-2 days.

### 2.2 Narrative Clustering
**What**: Group summaries into themes ("RWA rotation", "L2 fee wars", "memecoin
season") and track them over days/weeks.

**Requires**: Prompt change + new tables
- Add a `narratives` table: id, label, description, first_seen, last_seen,
  status (active/fading/dead), mention_count.
- Add `summary_narratives` junction table linking summaries to narratives.
- In Stage 1 prompt: ask Haiku to tag each summary with 0-3 narrative labels
  from a provided list of active narratives, plus suggest new ones.
- Daily cron: decay inactive narratives, merge duplicates via LLM (batch
  dedup call — "are these the same narrative?").
- Stage 3 prompt: include active narratives with trajectory.

**Dependencies**: Phase 1 (entity tables for context). Stage 1 must be stable.
**Feasibility**: The LLM is good at thematic tagging when given a candidate list.
The hard part is narrative lifecycle — when does "L2 fee wars" stop being a
thing? Heuristic: no mentions in 5 days = fading, 14 days = dead. 3-5 days.

### 2.3 Event Chains
**What**: Link related events across time. Exploit at T=0 -> audit announcement
T+2d -> governance vote T+7d -> fork T+30d.

**Requires**: New tables + prompt changes
- `events` table: id, entity_id, event_type (exploit, audit, governance,
  launch, partnership, funding, hack, legal), description, timestamp,
  chain_id (nullable FK to self or a chain root).
- Stage 1 prompt addition: extract structured events with type classification.
- Linking logic: when a new event mentions an entity that has a recent event
  (<30 days), ask LLM "is this a follow-up to [previous event]?" Single
  yes/no call, Haiku-cheap.
- Surface in reports: "This is the 3rd event in the [Euler exploit] chain."

**Dependencies**: Phase 1 entity tables. Narrative clustering helps but not required.
**Feasibility**: Event extraction is reliable with structured prompts. Chain linking
is the tricky part — LLM yes/no classification works well here since it is
a bounded decision. 3-5 days.

### 2.4 Competitor Mapping
**What**: Know which projects compete (Aave vs Compound, Arbitrum vs Optimism)
and compare sentiment side-by-side.

**Requires**: New table + prompt change + optional manual seeding
- `entity_relationships` table: entity_id_a, entity_id_b, relationship_type
  (competes_with, built_on, invested_in, forked_from), confidence, source
  (llm_inferred | manual | coingecko).
- Seed common pairs manually (20-30 entries covers the obvious ones).
- Stage 1 prompt: "If this content compares or contrasts projects, extract
  the competitive relationship."
- Dashboard: side-by-side sentiment chart for competitor pairs.

**Dependencies**: Phase 1 entity tables + sentiment momentum (2.1) for useful comparison.
**Feasibility**: The relationship extraction is medium-reliable from LLM. Manual
seeding covers the important pairs. Sentiment comparison is just SQL once
the data exists. 2-3 days for core, ongoing curation.

### 2.5 Cycle Detection (Rule-Based)
**What**: Flag known recurring events — token unlocks, options expiry (monthly/
quarterly), FOMC meetings, ETH staking withdrawals, etc.

**Requires**: New table + external calendar data
- `calendar_events` table: id, name, recurrence_rule (cron-like), next_occurrence,
  entity_id (nullable), category (macro, unlock, expiry, governance).
- Seed with known schedules: FOMC dates, monthly options expiry, major token
  unlock schedules (from token unlock APIs or manual entry).
- Daily cron: check upcoming events in next 48h, inject into Stage 3 prompt
  as context.
- No LLM needed for detection — these are deterministic schedules.

**Dependencies**: Phase 1 (scheduler exists). Entity tables help for token-specific events.
**Feasibility**: High for known schedules. The value is in the curation of the
calendar, not the code. Token unlock data requires either manual entry or
an API like TokenUnlocks ($). 2 days for infra, ongoing data maintenance.

### 2.6 Pre/Post Event Analysis
**What**: After a known event (FOMC, token unlock, major hack), compare sentiment
and narrative before vs after.

**Requires**: Prompt change + calendar integration (2.5)
- When Stage 3 runs and a calendar event occurred in the past 24h, include
  a "pre/post" section: pull sentiment data from 48h before and 24h after.
- Pure SQL: `AVG(sentiment) WHERE created_at BETWEEN event_time - 48h AND event_time`
  vs `AVG(sentiment) WHERE created_at BETWEEN event_time AND event_time + 24h`.
- Stage 3 prompt: "FOMC announced rate hold yesterday. Pre-event sentiment
  was 0.3, post-event is 0.6. Analyze the shift."

**Dependencies**: Cycle detection (2.5) + sentiment momentum (2.1).
**Feasibility**: High. This is a prompt enrichment pattern — feed the LLM
structured context and let it narrate. 1-2 days.

### 2.7 Regional Divergence
**What**: Detect when ID-language community sentiment diverges from EN-language
community on the same token/topic.

**Requires**: Schema change + prompt change
- `entity_mentions` already stores source. Add `language TEXT` column.
- Stage 1 already detects language (franc). Pass language through to mention records.
- Query: per entity, compare `AVG(sentiment) WHERE language='eng'` vs
  `AVG(sentiment) WHERE language='ind'`.
- Flag divergence > 0.3 as notable. Include in Stage 3 prompt.
- Dashboard: split sentiment bar (EN vs ID) per entity.

**Dependencies**: Phase 1 entity tables. Requires meaningful ID-language source volume.
**Feasibility**: High technically. Limited by whether you have enough ID-language
sources to make the comparison meaningful. If you are monitoring 5+ ID
Discord channels, this works. 1-2 days.

---

## v3 Features (Requires Architectural Changes)

These need something the current stack cannot provide: price data feeds,
new external APIs, vector storage, or significant new infrastructure.

### 3.1 Contrarian Detection (Sentiment vs Price Divergence)
**What**: "Everyone is bearish on SOL but price is up 15% this week."
The classic contrarian signal.

**Requires**: New data source (price feeds) + new tables
- Need reliable price data: CoinGecko API (free tier: 30 calls/min), or
  CoinMarketCap, or a WebSocket feed from an exchange.
- `price_snapshots` table: entity_id, timestamp, price_usd, price_change_24h,
  price_change_7d, volume_24h, market_cap.
- Daily cron: fetch prices for all token-type entities with relevance > threshold.
- Divergence detection: compare sentiment trend (from 2.1) with price trend.
  Flag when they disagree by > 1 standard deviation.
- Inject into Stage 3: "Contrarian signal: SOL sentiment -0.4 but price +15% 7d."

**Dependencies**: Sentiment momentum (2.1), entity tables (Phase 1).
**Feasibility**: Technically simple — CoinGecko free tier is sufficient for daily
snapshots of top 100 tokens. The hard part is calibrating what counts as
"divergence" vs normal noise. Start with a generous threshold and tune.
Cost: CoinGecko free = $0, pro = $129/mo for historical data. 3-4 days.

### 3.2 Influencer Tracking + Credibility Scores
**What**: Track who said it first, who amplified, and build historical accuracy
scores per author.

**Requires**: New tables + new data + significant engineering
- `authors` table: id, platform, handle, display_name, follower_count,
  credibility_score, first_seen, total_calls, correct_calls.
- `author_calls` table: id, author_id, entity_id, claim (bullish/bearish/event),
  timestamp, resolved (bool), outcome (correct/incorrect/unresolved).
- To score credibility, you need to resolve claims against outcomes. This
  requires price data (3.1) and a claim extraction + resolution pipeline.
- "Who said it first": order items by timestamp per entity per narrative.
  Straightforward query once narratives (2.2) exist.
- "Who amplified": track retweet/quote chains. Twitter API provides this.
  Discord does not (no retweet equivalent).

**Dependencies**: Price feeds (3.1), narratives (2.2), entity tables (Phase 1).
**Feasibility**: "Who said it first" is easy — just a timestamp sort. Credibility
scoring is a **research problem** at the resolution step. How do you
determine if "@trader_x calling SOL to $200" was correct? You need to
define timeframes, thresholds, and handle ambiguous claims. The extraction
is LLM-solvable; the scoring logic is an engineering + design problem.
MVP of "first mover" tracking: 3-4 days. Full credibility: 2-3 weeks.

### 3.3 Cross-Market Correlation
**What**: Correlate crypto sentiment with tradfi risk appetite. "Crypto sentiment
is bullish but VIX is spiking — historically this divergence resolves badly."

**Requires**: New data sources + new tables
- Need tradfi data feeds: VIX, DXY, 10Y yield, S&P 500, gold.
- Options: Yahoo Finance API (free, unreliable), Alpha Vantage (free tier),
  FRED API (free, macro data), or a paid feed.
- `macro_snapshots` table: date, indicator (vix, dxy, us10y, spx, gold),
  value, change_1d, change_7d.
- Daily cron: fetch macro indicators, inject into Stage 3 prompt as context.
- Correlation analysis: compare crypto aggregate sentiment with macro regime.
  This is a prompt enrichment — let Sonnet narrate the relationship.

**Dependencies**: Sentiment momentum (2.1) for the crypto side.
**Feasibility**: Data acquisition is the bottleneck. FRED API covers yields and
some macro. Yahoo Finance is fragile but free. The LLM is good at
narrating macro context when fed structured data. The "correlation" itself
is better done as LLM reasoning than statistical — you do not have enough
data points for meaningful quantitative correlation. 3-5 days.

### 3.4 Macro Regime Detection
**What**: Classify current market as risk-on/risk-off/transition. "We are in a
risk-off regime: DXY up, yields up, equities flat, crypto down."

**Requires**: Same data as 3.3 + prompt engineering
- Once you have macro_snapshots (3.3), regime detection is a prompt problem.
- Feed Sonnet the last 30 days of macro indicators + crypto sentiment.
- Ask it to classify: risk-on / risk-off / transition / unclear.
- Track regime history in `macro_regimes` table: date, regime, confidence,
  rationale.
- Include in daily report header: "Current regime: risk-off (day 5)."

**Dependencies**: Cross-market correlation (3.3).
**Feasibility**: LLM-solvable with good prompt design. The risk is false
confidence — the model will always produce a classification even when the
market is genuinely ambiguous. Mitigate with a confidence score and
"unclear" as a valid output. 1-2 days on top of 3.3.

### 3.5 Entity Relationship Graph (Structured)
**What**: Not just "Solana was mentioned" but "Multicoin invested in Solana",
"Nansen acquired by..." — structured relationships with types.

**Requires**: Extended schema + prompt engineering + optional graph visualization
- Extend `entity_relationships` from 2.4: add temporal data (since, until),
  evidence (summary_id), confidence score.
- Relationship types: invested_in, acquired, founded, advises, competes_with,
  built_on, forked_from, partnered_with, regulated_by.
- Stage 1 prompt: extract relationships as structured JSON. This is
  within Haiku's capability for explicit statements ("a]6z announced
  investment in X"). Implicit relationships are unreliable.
- Visualization: force-directed graph on dashboard. Libraries like d3-force
  or vis-network. This is a frontend project, not an LLM problem.

**Dependencies**: Phase 1 entity tables, competitor mapping (2.4) as foundation.
**Feasibility**: Explicit relationship extraction works well. Implicit relationship
inference ("they were both at the same conference, probably connected") is
unreliable and not worth pursuing. Start with explicit only. Graph
visualization is a 3-5 day frontend project. Total: 5-7 days.

### 3.6 Alpha Decay Tracking
**What**: Measure how fast information propagates. Private Discord at T=0,
Crypto Twitter at T+2h, mainstream news at T+12h. Quantify the edge.

**Requires**: New tables + cross-source timestamp analysis
- For each entity-event pair, record first_seen per source tier:
  - Tier 1: private Discords (alpha groups)
  - Tier 2: Crypto Twitter (influencers)
  - Tier 3: news sites, mainstream CT
- `alpha_propagation` table: event_id, tier, source, first_mention_time.
- Detection: when an entity's mention_count spikes, look back at which
  source tier mentioned it first and compute time deltas.
- Requires source tier classification — user must tag which Discord channels
  are "alpha" vs "general" in source config. Add `tier` column to `sources`.

**Dependencies**: Event chains (2.3), multiple source types active (Phase 1).
**Feasibility**: The concept is sound and measurable. Challenges:
1. Requires enough sources across tiers to be meaningful (5+ per tier).
2. "Same event" matching across sources is the hard part — relies on entity
   resolution + event chain linking.
3. Small sample sizes make statistical claims unreliable.
Start with anecdotal tracking ("this story hit Discord 6h before CT") rather
than statistical claims. 3-5 days.

---

## Research (Unsolved / Experimental)

These are genuinely hard problems where no reliable solution exists. They
require experimentation, may not work, and should not be promised.

### R.1 Fake News / Manipulation Detection
**What**: Detect coordinated shilling, pump groups, bot networks, and
manufactured consensus.

**Why it is hard**:
- **Coordinated behavior detection** requires analyzing posting patterns
  across accounts: similar timing, similar phrasing, account age, follower
  graphs. This is a graph analysis problem, not an LLM problem.
- **Bot detection** on Twitter requires features the API does not expose
  (account creation date patterns, follower/following ratios at scale,
  posting frequency histograms). Twitter's own bot detection is unreliable.
- **Discord pump groups** post in private channels you likely do not have
  access to. By the time it hits public channels, the pump is underway.
- **LLM-based detection** ("does this look like a shill?") has high false
  positive rates. Genuine enthusiasm and paid shilling look identical in
  text form.

**What you can do now** (v2 scope):
- Flag unusually high mention velocity for low-relevance entities ("this
  token went from 0 to 50 mentions in 2 hours — suspicious").
- Flag copy-paste content (dedup layer already hashes content — track
  near-duplicate clusters via content_hash similarity).
- These are heuristics, not detection. Label them as "unusual activity"
  not "manipulation detected."

**What would actually work** (research):
- Social graph analysis (who follows whom, posting time correlation) —
  needs data you do not have at current API tier.
- Temporal pattern matching (coordinated posting within 30-second windows) —
  needs high-resolution timestamps and volume.
- Ground truth dataset of confirmed manipulations for model training — does
  not exist publicly.

**Honest assessment**: Do the heuristics in v2 (mention velocity anomaly,
content duplication). Do not claim manipulation detection. Real detection
requires social graph data and labeled training data you do not have.

### R.2 Influencer Credibility Scoring (Quantitative)
**What**: Assign a numerical accuracy score to influencers based on their
historical calls.

**Why it is hard**:
- **Claim extraction is ambiguous**. "SOL looking strong" — is that a call?
  What is the timeframe? What price constitutes "strong"?
- **Resolution requires price data + timeframe + threshold**. Did the call
  "work" if SOL went up 5% in a week but down 20% in a month?
- **Survivorship bias**: you only track people you are already monitoring.
  The influencer who was right once and never posted again scores 100%.
- **Context collapse**: a hedged take ("SOL could go either way but I lean
  bullish") is different from a conviction call ("all in SOL"). LLMs
  struggle to reliably distinguish confidence levels.

**What you can do now** (v2/v3):
- Track "first mover" — who mentioned an entity first. Factual, no
  judgment needed.
- Track mention frequency — who talks about what, how often. Descriptive.
- Manual tagging: let the user flag specific calls as "notable" and
  manually mark outcomes.

**What would actually work** (research):
- Structured claim extraction with explicit timeframes and targets, validated
  by the poster or a human. This is a product design problem, not a
  technical one — you would need influencers to make structured predictions.
- Academic research (e.g., Metaculus-style calibration tracking) exists but
  requires structured inputs, not free-text social media posts.

**Honest assessment**: First-mover tracking is valuable and buildable. Quantitative
credibility scores from free-text social media are unreliable. Do not ship
a number that implies false precision.

### R.3 Narrative Lifecycle Prediction
**What**: Predict when a narrative (e.g., "RWA rotation") is about to peak,
fade, or revive.

**Why it is hard**:
- Narrative lifecycles are driven by external catalysts (regulatory
  announcements, market events, influencer posts) that are inherently
  unpredictable.
- Historical pattern matching requires many completed narrative cycles to
  train on. You will not have this data for months/years.
- LLMs are good at describing current trajectory ("this narrative is
  accelerating") but poor at predicting inflection points.

**What you can do now**: Track narrative momentum (mention count trend). Label
as "emerging / active / peaking / fading / dead" based on heuristics.
Do not predict transitions.

### R.4 Cross-Language Semantic Matching
**What**: Detect that an Indonesian Discord message and an English tweet are
discussing the same event, without relying on entity name matching.

**Why it is hard**:
- Entity name matching (current approach) covers 80% of cases — token
  tickers and project names are language-independent.
- The remaining 20% requires semantic similarity across languages.
  Options: multilingual embeddings (e.g., multilingual-e5-large) or
  LLM translation. Both add latency and cost.
- Edge cases: Indonesian crypto slang does not translate well. "Cuan"
  (profit), "nyangkut" (stuck in a losing position) have no direct
  English equivalents.

**What you can do now**: Entity name matching covers most cases. Stage 1
summarizes both languages into English — cross-language correlation happens
naturally at the summary level via Stage 2.

**Honest assessment**: The current architecture handles this adequately through
entity matching + English-language summaries. Embedding-based semantic
matching is an optimization for edge cases, not a priority.

---

## Summary Matrix

| # | Feature | Phase | Type | Days | Dependencies |
|---|---------|-------|------|------|-------------|
| - | Core pipeline (RSS to report) | 1 | Planned | Done | - |
| - | Pre-summarize (decision tree) | 1 | Planned | Done | - |
| - | Market pulse (Sonnet, every 3h) | 1 | Planned | Done | - |
| - | Raw Discord feed (live mirror) | 1 | Planned | Done | - |
| - | Embeddings (summary + report) | 1 | Planned | Done | - |
| - | RAG chat (Sonnet + Gemini embed) | 1 | Planned | Done | Embeddings |
| 2.1 | Sentiment momentum | v2 | New table + prompt | 1-2 | Phase 1 |
| 2.2 | Narrative clustering | v2 | New tables + prompt | 3-5 | Phase 1 |
| 2.3 | Event chains | v2 | New tables + prompt | 3-5 | Phase 1 |
| 2.4 | Competitor mapping | v2 | New table + prompt + seed | 2-3 | Phase 1, 2.1 |
| 2.5 | Cycle detection | v2 | New table + calendar data | 2+ | Phase 1 |
| 2.6 | Pre/post event analysis | v2 | Prompt change | 1-2 | 2.5, 2.1 |
| 2.7 | Regional divergence | v2 | Schema change + prompt | 1-2 | Phase 1 |
| 3.1 | Contrarian detection | v3 | New data (price feeds) | 3-4 | 2.1 |
| 3.2 | Influencer tracking | v3 | New tables + data + eng | 3d-3w | 3.1, 2.2 |
| 3.3 | Cross-market correlation | v3 | New data (tradfi feeds) | 3-5 | 2.1 |
| 3.4 | Macro regime detection | v3 | Prompt on top of 3.3 | 1-2 | 3.3 |
| 3.5 | Entity relationship graph | v3 | Extended schema + frontend | 5-7 | Phase 1, 2.4 |
| 3.6 | Alpha decay tracking | v3 | New tables + source tiers | 3-5 | 2.3, Phase 1 |
| R.1 | Fake news / manipulation | Research | Graph analysis problem | ? | Social graph data |
| R.2 | Credibility scoring | Research | Claim resolution problem | ? | 3.1, 3.2 |
| R.3 | Narrative lifecycle prediction | Research | Prediction problem | ? | 2.2 + months of data |
| R.4 | Cross-language semantic match | Research | Embedding infrastructure | ? | Current approach is adequate |

## Recommended Build Order (Post Phase 1)

```
Phase 1 complete
    |
    v
2.1 Sentiment momentum -----> 2.7 Regional divergence
    |                               |
    v                               v
2.2 Narrative clustering       2.6 Pre/post event analysis
    |                               ^
    v                               |
2.3 Event chains               2.5 Cycle detection (can start early)
    |
    v
2.4 Competitor mapping
    |
    v
3.1 Price feeds ----------------> 3.3 Macro data
    |                               |
    v                               v
3.2 Influencer (first-mover)   3.4 Macro regime
    |
    v
3.5 Entity graph
    |
    v
3.6 Alpha decay
```

Start with 2.1 (sentiment momentum) — it unlocks the most downstream features
and has the best effort-to-value ratio. Then 2.2 (narratives) for the
qualitative layer. Everything else builds on those two.

## What NOT to Build

- **Real-time analytical dashboard** — SSE/WebSocket updates for the
  analytical pipeline (reports, sentiment charts) look impressive but the
  pipeline is batch-driven with per-source poll intervals. Real-time UI on
  batch analytics is theater. The raw Discord feed is a different story — it
  is near-real-time by nature and ships in Phase 1. But do not add streaming
  infrastructure to the analytical views until you have a real-time data
  source (exchange WebSocket).

- **Multi-agent orchestration** — the current direct-call pipeline is simple
  and debuggable. Adding agent delegation, tool use, or multi-step reasoning
  adds complexity without clear user value. The LLM's job is to summarize
  and synthesize, not to make autonomous decisions.

- **Plugin/skill system** — this is what killed v1. Do not repeat it.

- **Prediction markets / trading signals** — Podders surfaces conditions,
  it does not make calls. The moment you output "buy SOL," you need to be
  right. Surface the data, let users decide.
