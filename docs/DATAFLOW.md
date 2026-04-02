# Data Flow Reference -- Podders v2

Visual navigation doc. ASCII diagrams showing every data flow in the system.

---

## 1. Main Pipeline Flow

```
 SOURCES                 INGEST              NORMALIZE (Stage 0, Haiku for Ind→EN only)
+----------+          +----------+          +-------------------+
| Discord  |--ws----->|          |          |                   |
| Twitter  |--poll--->| RawItem  |--------->| Truncate (20K ch) |
| RSS      |--poll--->| { id,    |          | Unicode NFC       |
| News     |--manual->|   source,|          | InstructDetector  |
|          |          |   content|          | Dedup (sha256)    |
+----------+          |   engag. |          | URL expand+dedup  |
                      |   ... }  |          | Spam filter       |
                      +----------+          | Language detect    |
                                            | Ind→EN translate* |
                                            +---+----------+----+
                                                |          |
                                           'filtered'   'ready'
                                           (kept for     (English content)
                                            debug)          |
                                                            v
                                            PRE-SUMMARIZE (Haiku)
                                            +-------------------+
                                            | RSS/news >4K ch?  |
                                            |   yes → Haiku     |
                                            |   compress ~300tok|
                                            |   no → pass thru  |
                                            +--------+----------+

 *Ind→EN translate: Haiku call only for Indonesian content (Decision 01).
  InstructDetector: prompt injection scan, cached per item (Decision 06).
                                                     |
                                                     v
 STAGE 1 (Haiku, per-source intervals)         +-------------------+
+----------------------------------------------| Batch claim       |
|  Claim: status ready -> processing           | (UPDATE...WHERE   |
|  Sources processed in parallel               |  status='ready')  |
|  Author cap (20% max per author)             +-------------------+
|  Topic-aware chunking (6K token budget)
|  Budget hints: sparse chunks get "~400 tok" hint (Decision 03)
|  Haiku call per chunk (JSON output)               |
|  Zod parse -> entity post-verify -> quality gate  |
|  Confidence < 5 + non-routine → escalate to       |
|    Sonnet (Decision 10)                            v
|                                          +-----------------+
|  Output per chunk:                       | summaries table |
|    ChunkSummary {                        | (body = JSON)   |
|      summary, urgency,                   +-----------------+
|      confidence (1-10),                       |
|      entities[], keyEvents[]                  |
|    }                                          |
+-----------------------------------------------+
      |                    |
      v                    v
 STAGE 2 (SQL)        FLASH CHECK (Decision 02)
+---------------+     +---------------------------+
| correlate.run |     | urgency='breaking'?        |
| entities in   |     | weighted trust sum >= 1.5   |
| 2+ sources    |     | (not just 2+ sources)       |
| within 24h    |     | High-trust src (1.0) can    |
+-------+-------+     |   fire alone; 3 low-trust   |
        |              |   (0.5 ea) also fires       |
        |              +-------------+---------------+
        |                            |yes
        |                            v
        |              STAGE 3 FLASH (Sonnet)
        |              scope: last 4h
        |              +---> MarketReport (type='flash')
        |                       |
        v                       v
 NARRATIVE CLUSTERING (Decision 05)
+--------------------------------------+
| k-means on summary embeddings        |
| Auto-tune k (3-10, silhouette score) |
| Haiku names each cluster (3-5 words) |
| Signal: new/emerging/strong/fading   |
| Cost: ~$0.01/day (3-5 Haiku calls)   |
+-------------------+------------------+
                    |
                    v
 STAGE 3 DAILY (Sonnet, at digest_time)    DELIVERY
+--------------------------------------+   +--------------------+
| Reads: summaries (prev calendar day) |   |                    |
|        correlated entities           |-->| webhook.deliver()  |
|        yesterday's TL;DR             |   | Discord embed POST |
|        narrative clusters (Decis 05) |   | 3 retries (2/8/32s)|
|        raw exemplars (7 engagement   |   |                    |
|          + 3 RSS/news reserved)      |   |                    |
| If >50 summaries: rank, take top 30  |   |                    |
| Quality gate: skip if no key events  |   | reports.delivery_  |
|                                      |   |   status =         |
| Output: MarketReport {               |   |   delivered|failed |
|   tldr, keyEvents[], sections[],     |   +--------------------+
|   entitySentiment[], newProjects[]   |            |
| }                                    |            v
+--------------------------------------+   +----------------+
        |                                  | Dashboard      |
        v                                  | / (latest)     |
 +---------------+                         | /reports (all) |
 | reports table |------------------------>| /feed (raw)    |
 | (body = JSON) |                         | /settings      |
                                           +----------------+
 +---------------+

 Pre-summarize runs as a separate step after normalize (not inside it).
 RSS/news items >4000 chars get a Haiku pre-summary. Equalizes signal density.
```

### Data Type Transformations

```
Source event     RawItem          items row        ChunkSummary       CorrelatedEntity    MarketReport
(ws/http)    --> (in-memory)  --> (Postgres)   --> (summaries.body)-> (in-memory)     --> (reports.body)
                 id: ulid         status: ready    summary: text      name: string        tldr: string
                 source: enum     content_hash     urgency: enum      sources: string[]   keyEvents[]
                 content: text    engagement: int  confidence: 1-10   avg_sentiment       sections[]
                 engagement: num  batch_id: null   entities: json[]   source_count        entitySentiment[]
                 metadata.translated?              keyEvents: str[]                       narratives[]
```

---

## 2. Cron Job Timing Diagram (24h)

Stage 1 no longer runs on a fixed 2h interval. Each source type has its own poll interval (e.g., Discord 30min, Twitter 2h, RSS 15min). When a source's interval elapses, it is processed. Multiple sources fire in parallel.

```
UTC   00    02    04    06    08    10    12    14    16    18    20    22    24
       |     |     |     |     |     |     |     |     |     |     |     |     |
S1     XX X  XX X  XX X  XX X  XX X  XX X  XX X  XX X  XX X  XX X  XX X  XX X  .
(var)  |||   |||   |||   |||   |||   |||   |||   |||   |||   |||   |||   |||
       ||+-- RSS (every 15min)        Sources processed in parallel.
       |+--- Discord (every 30min)    Each fires independently based on
       +---- Twitter (every 2h)       its own poll_interval.
             |
             +->S2 (after each batch) +->flash? (if breaking detected)

S3     .     .     .     .     .     .     .     .     .     .     .     .     .
daily  .     * (09:00 WIB = 02:00 UTC, configurable)
             |
             +->webhook.deliver()

Pulse  .  X  .  X  .  X  .  X  .  X  .  X  .  X  .  X  .  .  .  .  .  .  .  .
(3h)   ~~~(Sonnet, output scales with activity)~~~~~~~~~~~~~~~~~~~~~~~~~~
             +->webhook.deliver() (muted grey embed)

Decay  X     .     .     .     .     .     .     .     .     .     .     .     .
00:00  |
Reten  .X    .     .     .     .     .     .     .     .     .     .     .     .
00:05  |
Backup .X    .     .     .     .     .     .     .     .     .     .     .     .
00:10  |

Health X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X
(5min) ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Embed  XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
(15m)  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~(if enabled)

LEGEND: X = job fires    S1 = Stage 1    S2 = Stage 2 (direct call after S1)
        * = configurable time    -> = direct function call (not cron)
```

### Job Dependencies

```
Stage 1 (per-source interval) --> Stage 2 (direct) --if breaking--> Stage 3 flash (direct) --> Webhook
Stage 3 daily (cron) --> Webhook (direct)
Pulse (every 3h cron) --> Webhook (direct)  (Sonnet, output scales with activity)
Decay 00:00 --> Retention 00:05 --> Backup 00:10  (staggered 5min)
Scraper Health: every 5min, independent
Embed cron: every 15min, independent (no-op if GEMINI_API_KEY unset)
```

---

## 3. Per-Source Data Flow

Each source type has a default poll interval. Sources are processed in parallel when intervals overlap.

```
               INGESTION METHOD         sourceId           POLL INTERVAL    CONTENT SIZE
Discord  ---- Gateway WebSocket ------> channel_id         30min (default)  short (msgs)
Twitter  ---- REST poll -------------> @handle / query     2h (default)     short (tweets)
RSS      ---- rss-parser ------------> feed_url            15min (default)  long (articles)
News     ---- manual URL submit ------> article_url        on-demand        long (articles)
               |
               v
           All produce RawItem -> normalize -> items table (status: 'ready')
           Pre-summarize: RSS/news items >4K chars get Haiku compression (separate step)
           Processing: sources fire independently, run in parallel via Promise.allSettled
```

### Source Differences

```
                  Engagement          Dedup                  Pre-summarize*  Chunking
+---------+---------------------------+------------------------+-----------+------------------+
| Discord | reaction count (0-100)    | sha256 hash            | no        | group by threadId|
| Twitter | log10(L+2*RT+3*Q)*20      | sha256 + last_id track | no        | token budget     |
| RSS     | always 0 (no signal)      | sha256 + URL match     | YES >4Kch | group by category|
| News    | always 0                  | sha256 + URL match     | YES >4Kch | token budget     |
+---------+---------------------------+------------------------+-----------+------------------+

* Pre-summarize runs as its own step after normalize, not inside normalize.

Shared normalize steps (all sources, Haiku for Ind→EN translation only):
  Truncate >20K chars | Unicode NFC + zero-width strip | InstructDetector scan (Decision 06)
  Dedup (sha256) | URL expand+dedup | Spam filter (en+id rules) | Language detect (eng/ind only)
  Ind→EN translate via Haiku if Indonesian (Decision 01)
```

---

## 4. Entity Lifecycle

### Early build (entities as JSON in summaries -- before entity tables)

```
Haiku -> ChunkSummary.entities[] -> stored in summaries.body as JSON
  Post-verify: drop entities not found in source text (anti-hallucination)
  Quality gate: 0 entities + 0 events + short summary = low_quality (excluded from Stage 3)
  New project detection: names not in previous summaries -> newProjects[] in MarketReport
```

### Full build (entity tables -- full resolution)

```
Haiku output             Alias Resolution              Entity Tables
+----------------+      +---------------------+       +------------------+
| name: Ethereum |      | lowercase("ETH")    |       | entities         |
| aliases: [ETH] |----->| lookup entity_aliases|------>| id, name, type,  |
| type: token    |      | found? -> entity_id  |       | relevance,       |
| mentions: 12   |      | not found? -> INSERT |       | first_seen,      |
+----------------+      |   new entity + alias |       | last_seen        |
                         +---------------------+       +------------------+
                                |                              |
                                v                              v
                         entity_aliases              entity_mentions
                         "eth" -> ULID_001           entity_id, source,
                         "ethereum" -> ULID_001      summary_id, sentiment
                         "$eth" -> strip $ -> "eth"  mention_count, created_at

SEEDING: CoinGecko tokens (symbol+name+id) + Indonesian entities (OJK, BI, BEI, etc.)

RELEVANCE = relevance * 0.95^days + log(1+mentions) * source_weight
  Weights: Discord=1.0  Twitter=1.5  News=2.0 (rarer = higher signal)
  Decay cron: daily 00:00 UTC (relevance *= 0.95)
  Boost: real-time during Stage 1 entity upsert

LIFECYCLE (Decision 04 — two-tier active/archived):
  Day 0 ~2.0 -> Day 7 ~1.4 -> Day 30 ~0.43 -> Day 90 <0.01 -> ARCHIVED (not deleted)

  Status transitions:
    ACTIVE ──(relevance < 0.01 AND last_seen > 90d)──> ARCHIVED
    ARCHIVED ──(re-mentioned in Stage 1)──> ACTIVE (relevance reset to 0.5)

  Archived entities preserve: name, type, aliases, last sentiment, mention count, timestamps
  Reports/correlation: WHERE status = 'active' (only active entities appear)
  Chat/search: no status filter (can find archived entities too)
  Dashboard: archived entities shown in separate "inactive" section

DISAMBIGUATION: UNIQUE(name, type) -- same name can be token AND company
  Alias collision: ON CONFLICT DO NOTHING (first writer wins)
```

---

## 5. Report Lifecycle

```
  DAILY REPORT                                    FLASH REPORT
  ~~~~~~~~~~~~                                    ~~~~~~~~~~~~
  Inputs:                                         Trigger:
    summaries (prev calendar day)                   Stage 1 returns urgency:'breaking'
    correlated entities (24h)                       Corroboration: weighted trust sum >= 1.5 (Decision 02)
    narrative clusters (Decision 05)                  High-trust source (1.0) can fire alone
    yesterday's TL;DR (temporal anchor)               3 default sources (0.5 ea) also fires
    raw exemplars (7 engagement                       (single chunk below threshold = log only)
      + 3 RSS/news reserved)                      Scope: last 4h summaries only
  Guards:                                         Guards:
    quality gate (skip if 0 events+entities)        cooldown: 30min between flashes
    dedup (1 daily per date)                        no dedup (multiple per day OK)
         |                                              |
         v                                              v
    Sonnet synthesis --> MarketReport              Sonnet synthesis --> MarketReport
         |                type: 'daily'                 |                type: 'flash'
         v                                              v
         +-------------------+--------------------------+
                             |
                             v
  MARKET PULSE                                    
  ~~~~~~~~~~~~                                    
  Trigger: every 3h cron (Sonnet)                  
  Scope: summaries from last 3h window            
  Output scales with activity:                    
    quiet → shorter pulse, busy → fuller          
  Embed: muted grey (0x4A4A5A), titled            
    "Market Pulse — HH:MM WIB"                    
                             |
         +-------------------+
                             |
                             v
                      webhook.deliver()  (Discord embed POST)
                             |
              +--------------+--------------+
              v              v              v
         HTTP 2xx       429/5xx         4xx (not 429)
         'delivered'    retry 2/8/32s   'failed'
                        then 'failed'   surfaced on /settings
                             |
                             v
                      +---------------+
                      | Dashboard     |  / = latest TL;DR
                      | /reports      |  archive + detail view
                      | /feed         |  raw Discord feed + image attachments
                      | /settings     |  failed deliveries
                      +---------------+

Webhook embed: daily = title + TL;DR + events + sentiment + coverage + report link
               flash = [FLASH] prefix + 2-sentence TL;DR + 1 section max
               pulse = muted grey (0x4A4A5A) + "Market Pulse — HH:MM WIB" title

TRUST WEIGHT LIFECYCLE (Decision 02)
  Source created → manual trust_weight set (default 0.5)
  Scale: 1.0 high (Reuters) | 0.7 mid (CoinDesk) | 0.5 default | 0.3 low | 0.1 near-zero
  Auto-adjust: 1 hour after each flash alert
    confirmed by other sources → weight += 0.05 (capped at initial + 0.2)
    unconfirmed              → weight -= 0.03 (floored at initial - 0.2)
  Manual weight sets the floor/ceiling — system nudges ±0.2 but never overrides
```

---

## 6. Embedding Data Flow

```
Requires GEMINI_API_KEY. If missing, all embedding functions are no-ops.
Model: Gemini text-embedding-004 (768 dims). Storage: BYTEA in embeddings table.

ITEM EMBEDDING (batch, every 15min cron)
+----------+     +-----------+     +-------------+     +----------------+
| items    |     | Claim un- |     | Gemini API  |     | embeddings     |
| status:  |---->| embedded  |---->| batch of    |---->| target: 'item' |
| 'ready'  |     | (LEFT JOIN|     | up to 100   |     | vector: BYTEA  |
+----------+     |  NULL)    |     | ~150 tok/ea |     | 3072 bytes/row |
                 +-----------+     +-------------+     +----------------+

SUMMARY EMBEDDING (inline, after Stage 1)
+------------+     +-------------+     +-------------------+
| Haiku      |     | Gemini API  |     | embeddings        |
| returns    |---->| single call |---->| target: 'summary' |
| ChunkSumm  |     | ~300 tokens |     | in-memory cache   |
+------------+     +-------------+     +-------------------+

ENTITY + REPORT EMBEDDING (inline, on create/update)
  Entity upsert  ---> Gemini (~30 tok)  ---> embeddings target:'entity' (in-memory cache)
  Stage 3 output ---> Gemini (~400 tok) ---> embeddings target:'report' (in-memory cache)

NARRATIVE CLUSTERING (daily, before Stage 3 — Decision 05)
+------------------+     +-------------------+     +------------------+
| Summary vectors  |     | k-means clustering|     | Haiku names each |
| from last 24h    |---->| k=3-10 (silhouette|---->| cluster (3-5     |
| (in-memory)      |     |   auto-tune)      |     | words, ~20 tok)  |
+------------------+     +-------------------+     +--------+---------+
                                                            |
                          +-------------------+             v
                          | narratives table  |<--- Signal: new/emerging/
                          | name, date,       |     strong/stable/fading
                          | signal_strength   |     (growth vs yesterday)
                          +-------------------+

QUERY FLOW (semantic search — Decision 11)
+-------------+     +-----------+     +-----------+     +------------------+     +---------+
| User query  |     | Language   |     | Embed the |     | Cosine similarity|     | Top 10  |
| "solana     |---->| detect     |---->| query     |---->| vs in-memory     |---->| results |
|  staking"   |     | (franc)    |     | string    |     | summary/report   |     | > 0.3   |
+-------------+     | Ind→EN?   |     +-----------+     | vectors (~3K)    |     | thresh  |
                    | translate  |                       +------------------+     +---------+
                    +-----------+
Indonesian queries translated to English via Haiku before embedding (Decision 11).
All embeddings are English — English query ↔ English vectors = best similarity.

SEMANTIC DEDUP (in normalize)
  New item -> embed -> cosine vs last 24h (same source type)
  > 0.92: filtered | 0.80-0.92: flagged as near-duplicate

STORAGE (30 days, 5K items/day): ~450MB items + ~9MB summaries/entities/reports
  Summary/report/entity vectors loaded into memory on startup (~9MB)
  Item embeddings deleted alongside items (30-day TTL)

COST: ~$0/day (free tier: 1500 req/day) at 5K items/day
```

---

## 7. RAG Chat Data Flow (Decision 07)

```
USER QUERY                        TOOL SELECTION (Sonnet)              BACKENDS
+-------------------+            +-------------------------+
| "What happened    |            | Sonnet picks tool(s)    |
|  with SOL?"       |---+        | per query:              |
+-------------------+   |        |                         |
                        |        | semantic_search --------+---> Gemini embeddings
  Indonesian query?     |        |   topic/theme questions |     cosine vs summaries/reports
  yes → Haiku translate |        |   "what happened with?" |     top 10 > 0.3 threshold
  (Decision 11)         |        |                         |
                        +------->| keyword_search ---------+---> Postgres entities table
                                 |   sentiment, mentions   |     entity_mentions join
                                 |   "how is sentiment?"   |     source breakdown
                                 |                         |
                                 | read_raw ---------------+---> items table
                                 |   verify claims, quotes |     full content + metadata
                                 +----------+--------------+
                                            |
                                            v
                                 +-------------------------+
                                 | Sonnet synthesizes      |
                                 | answer from tool results|
                                 | (iterative tool calls   |
                                 |  if multi-hop question) |
                                 +-------------------------+

  Cost: same as current chat (~$0.15-0.30/day). Tool calls are function executions, not LLM calls.
  Chat history: maintained per session. Dialogue context improves retrieval accuracy.
```
