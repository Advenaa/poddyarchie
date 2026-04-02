# Podders v2 — Embeddings Architecture

Embeddings are a core capability, not a future optimization. They enable semantic
search, better dedup, narrative detection, and entity disambiguation — all things
the current pipeline does with heuristics or not at all.

---

## 1. What Gets Embedded

Four embedding targets, in priority order:

| Target | Text Input | When | Why |
|--------|-----------|------|-----|
| **Summaries** | `summaries.body` JSON stringified (the ChunkSummary text, not metadata) | After Stage 1 | Primary search surface. 12-30 per day, low volume, high value. |
| **Items** | `items.content` (raw post/article text, post-truncation) | During Stage 0 normalize | Semantic dedup, similar-item detection, topic clustering. High volume. |
| **Entities** | `entities.name + type + description` (description extracted from first meaningful mention) | On entity creation/update | Entity disambiguation ("Mercury" token vs "Mercury" company). Low volume. |
| **Reports** | `reports.tldr + key_events` (extracted from body JSON) | After Stage 3 | Report search ("what happened the week of the ETH ETF?"). 1-2 per day. |

**What does NOT get embedded**: raw metadata fields, author names, URLs,
engagement scores. These are filterable via SQL — no need to waste embedding
dimensions on structured data.

**Embedding text preparation**: before embedding, strip Discord formatting
(`**bold**`, `> quotes`, `<@mentions>`), collapse whitespace, truncate to
model's max input (512 tokens for most models). For summaries and reports,
use the English-language text only (summaries are already in English).

---

## 2. Embedding Model Recommendation

| Model | Dims | Max Tokens | Cost/1M tokens | Quality (MTEB avg) | Latency | Notes |
|-------|------|-----------|----------------|-------------------|---------|-------|
| Voyage AI voyage-3 | 1024 | 32K | $0.06 | ~68 | 50-100ms | Best quality-per-dollar. Batch API available. |
| OpenAI text-embedding-3-small | 1536 | 8K | $0.02 | ~62 | 30-60ms | Cheapest. Decent quality. Dimension reduction to 512 supported. |
| OpenAI text-embedding-3-large | 3072 | 8K | $0.13 | ~65 | 40-80ms | Overkill for this use case. |
| Google Gemini text-embedding-004 | 768 | 2K | $0.004 | ~66 | 40-80ms | Cheapest by far. Short context limit. |
| nomic-embed-text (local) | 768 | 8K | $0 | ~63 | 10-30ms* | No API cost. Needs ~2GB RAM. ONNX runtime. |
| all-MiniLM-L6-v2 (local) | 384 | 256 | $0 | ~57 | 5-15ms* | Tiny and fast. Quality is noticeably worse. |

*Local latency assumes CPU inference on a modest VPS.

**Recommendation: Google Gemini text-embedding-004 at 768 dimensions.**

Rationale:
- Free tier: 1,500 requests/day. At 5K items/day batched into groups of 100,
  that is ~50 requests/day for items + a handful for summaries/reports/entities.
  Comfortably within free tier.
- 768 dimensions is a good balance — higher quality than 512-dim reduced models,
  lower storage than 1536-dim OpenAI.
- Quality (MTEB ~66) is better than text-embedding-3-small (~62) and close to
  voyage-3 (~68).
- One new env var: `GEMINI_API_KEY`. SDK: `@google/generative-ai`.
- Local models (nomic, MiniLM) avoid API cost but add 2GB+ RAM and require ONNX
  runtime. Revisit if running on a GPU box.

**Fallback**: if Gemini free tier limits become a constraint at scale, switch to
OpenAI text-embedding-3-small ($0.02/1M tokens) or Voyage AI voyage-3
($0.06/1M tokens). The `model` column in the embeddings table enables clean
migration.

---

## 3. Storage

### Schema

```sql
CREATE TABLE embeddings (
  id TEXT PRIMARY KEY,             -- ULID
  target_type TEXT NOT NULL,       -- item | summary | entity | report
  target_id TEXT NOT NULL,         -- FK to items.id, summaries.id, etc.
  model TEXT NOT NULL,             -- 'text-embedding-004' (for migration tracking)
  dimensions INTEGER NOT NULL,     -- 768
  vector BYTEA NOT NULL,            -- Float32Array as raw bytes (768 * 4 = 3072 bytes)
  created_at INTEGER NOT NULL,
  UNIQUE(target_type, target_id)
);
CREATE INDEX idx_embeddings_target ON embeddings(target_type, target_id);
CREATE INDEX idx_embeddings_type ON embeddings(target_type, created_at);
```

### Why BYTEA, Not pgvector

**pgvector** (Postgres vector search extension) is compelling but adds operational complexity:
- Requires installing the `pgvector` extension on the Postgres server.
- Index types (IVFFlat, HNSW) require tuning and periodic rebuild.
- At our scale (150K item vectors + 3K summary vectors), brute-force cosine
  similarity over BYTEA columns is fast enough.

**BYTEA storage + JS cosine similarity**:
- Store vectors as raw `Float32Array` bytes in a `BYTEA` column. 768 dims * 4 bytes = 3,072 bytes per vector.
- Cosine similarity in JS is ~0.1ms per comparison. Scanning 150K vectors
  takes ~15 seconds — too slow for real-time search.
- **Solution**: load summary + report + entity vectors into memory on startup
  (~3K vectors * 3KB = 9MB). Search against these in <5ms.
  Item-level search uses a pre-filter (time range, source) to reduce the
  scan set, then cosine in JS over the filtered BYTEA rows.
- If item-level search gets too slow at scale, add pgvector as an
  optimization later. The BYTEA schema is forward-compatible.

### Volume Projections (30 days at 5K items/day)

| Target | Count | Size (vectors only) | Size (with overhead) |
|--------|-------|--------------------|--------------------|
| Items | 150,000 | 439 MB | ~525 MB |
| Summaries | 900 (30/day * 30) | 2.6 MB | ~3 MB |
| Entities | ~2,000 | 5.9 MB | ~7 MB |
| Reports | 30-35 | 102 KB | ~150 KB |
| **Total** | ~153,000 | **~448 MB** | **~535 MB** |

Item vectors dominate. The `embeddings` table will be the largest table in the
DB. Data retention cron should delete item embeddings alongside item records
(cascade on the 30-day TTL).

---

## 4. When to Embed

### Embedding Pipeline

```
Item arrives
  → Stage 0 (normalize)
    → if status = 'ready': queue for embedding
      → embed in micro-batches of 100 (Gemini batch endpoint)
      → write BYTEA to embeddings table
  → Stage 1 (summarize)
    → after Haiku returns: embed the summary text
    → write BYTEA to embeddings table

Entity created/updated
  → embed entity description
  → write/replace BYTEA

Stage 3 (synthesize)
  → after report generated: embed tldr + key_events
  → write BYTEA
```

### Batching Strategy

Do NOT embed one-at-a-time. The Gemini embeddings API accepts batch requests.

- **Items**: batch embed every 15 minutes via a dedicated cron job. Claim
  un-embedded items (`LEFT JOIN embeddings ... WHERE embeddings.id IS NULL`),
  batch into groups of 100, call API. At 5K items/day, that is ~21 items per
  15-min window — one API call per cycle.
- **Summaries/reports/entities**: embed immediately after creation. Volume is
  low enough (1-30 per cycle) to embed inline without batching.

### Failure Handling

- On API error (429, 5xx): retry with same backoff as `llm.ts` (2s/8s/32s).
- On persistent failure: mark items as `embedding_failed` in a status column
  (or a separate `embedding_status` column on the items table). Retry on next
  cycle. Do not block the LLM pipeline — embedding is async and non-blocking.
- On model change: re-embed everything. Add a `model` column to track which
  model generated each vector. Migration script: delete all embeddings where
  `model != current_model`, let the batch job re-embed.

---

## 5. Use Cases and Queries

### 5.1 Semantic Search

User query: "what did people say about Solana staking?"

```
1. Embed the query string using the same model
2. Load summary vectors (in-memory, ~900 vectors)
3. Cosine similarity against all summary vectors
4. Return top 10 matches above threshold (0.3)
5. JOIN back to summaries table for full text + metadata
6. Optional: also search item vectors with time filter for raw source hits
```

This replaces Postgres full-text search as the primary search mechanism. tsvector/tsquery stays as a fallback for exact keyword matches ("FOMC" -- you want exact match, not semantic).

Semantic search is now one of three retrieval tools available to the RAG chat
feature (Decision 07). Sonnet selects the appropriate tool per query:
- `semantic_search` — embedding-based search against summaries/reports (this section)
- `keyword_search` — entity lookup via the entities table (mention counts, sentiment)
- `read_raw` — read original source messages by ID for verification/quotes

Sonnet picks the right tool (or combination) per query. Simple entity lookups use
keyword_search; thematic questions use semantic_search; verification uses read_raw.

**API endpoint**: extend `GET /api/v1/search` to accept `mode=semantic|keyword`
(default: semantic). Keyword mode uses tsvector/tsquery. Semantic mode uses embeddings.

### 5.2 Topic Clustering

Group items by semantic similarity before sending to Stage 1, instead of
grouping by source + time window.

```
1. Take all 'ready' items for current window
2. Compute pairwise cosine similarity (or use k-means on vectors)
3. Cluster into groups of semantically similar items
4. Send each cluster to Stage 1 as a chunk (instead of source-based chunks)
```

**Impact**: summaries become topical ("here is everything about the Eigen
airdrop") instead of source-based ("here is what Discord said in the last 2h").
This is a meaningful quality improvement for Stage 3 synthesis.

**Trade-off**: adds embedding latency before Stage 1 can run. Items must be
embedded before chunking. The 15-min embedding cron must run before the 2h
Stage 1 cron — which it will, naturally.

### 5.3 Entity Disambiguation

Problem: "Mercury" appears as both a token and a company. The alias table
handles exact matches but not fuzzy cases.

```
1. New entity candidate from Stage 1: "Merc" (unknown)
2. Embed "Merc"
3. Compare against all entity vectors (~2K)
4. If cosine similarity > 0.85 with "Mercury (token)": merge as alias
5. If similarity 0.6-0.85: flag for review (log, surface on /settings)
6. If similarity < 0.6: create new entity
```

This is better than the current lowercase-exact-match approach for handling
abbreviations, typos, and alternate spellings common in crypto social media.

### 5.4 Narrative Detection

Instead of relying on Stage 1 to tag narratives from a provided list (roadmap
item 2.2), detect narratives bottom-up from embedding clusters.

```
1. Take all summary vectors from the last 7 days
2. Cluster using DBSCAN or agglomerative clustering (in JS, no heavy deps)
3. Clusters that persist across 3+ days = candidate narrative
4. Use LLM to label the cluster: "What theme connects these summaries?"
5. Track cluster centroid over time — growing cluster = emerging narrative
```

**Advantage over prompt-based narratives**: does not require maintaining a
candidate list. New narratives emerge organically from the data.

### 5.5 Similar Item Detection (Semantic Dedup)

Current dedup: `sha256(source + sourceId + content.slice(0, 200))`. Catches
exact copies, misses paraphrases.

```
1. After embedding a new item, compare against items from last 24h (same source type)
2. If cosine similarity > 0.92: mark as duplicate (status = 'filtered')
3. If similarity 0.80-0.92: flag as "near-duplicate" in metadata (let Stage 1 decide)
```

**Volume concern**: comparing each new item against 5K daily items is 5K cosine
ops — ~0.5ms in JS. Acceptable. Do not compare across all 150K items; limit
to a 24h window.

---

## 6. Integration with Existing Pipeline

Embeddings **layer on top** of the existing architecture. No stage is removed
or replaced. The pipeline becomes:

```
Sources → Ingest → Normalize (Stage 0) → Embed (new) → Process → Knowledge → Surface
                                            |
                                            ├─ item vectors (async, batched)
                                            ├─ summary vectors (inline, after Stage 1)
                                            ├─ entity vectors (inline, on create/update)
                                            └─ report vectors (inline, after Stage 3)
```

### What Changes

| Component | Change | Breaking? |
|-----------|--------|-----------|
| `config.ts` | Add `GEMINI_API_KEY` env var (required — embeddings are a Phase 1 capability) | No |
| `db/migrations.ts` | Add `embeddings` table | No |
| `db/queries.ts` | Add embedding CRUD + similarity queries | No |
| New: `src/embed.ts` | Gemini embedding wrapper (like `llm.ts` — single call site, retry, cost logging) | No |
| New: `src/embed-pipeline.ts` | Batch embedding cron job + inline embedding helpers | No |
| `src/scheduler.ts` | Add 15-min embedding cron job | No |
| `src/process/summarize.ts` | After writing summary, call `embedSummary()` | No |
| `src/process/synthesize.ts` | After writing report, call `embedReport()` | No |
| `src/knowledge/entities.ts` | After entity create/update, call `embedEntity()` | No |
| `src/normalize/dedup.ts` | Add semantic dedup check | No |
| `src/server.ts` | Extend search endpoint with `mode=semantic` | No |
| `llm_usage` table | Log embedding API costs alongside LLM costs | No |

### What Does NOT Change

- The Stage 0 → 1 → 2 → 3 pipeline order and call chain.
- The item/summary/report schema (embeddings are a separate table).
- The scheduler's cron timing (embedding cron is additive).
- The dashboard (search UI gets a toggle, but report view is unchanged).

**Graceful degradation**: if `GEMINI_API_KEY` is not set, all embedding
functions become no-ops. The pipeline runs exactly as designed without
embeddings. Search falls back to Postgres full-text search (tsvector/tsquery) only.

---

## 7. Cost Model

Using Google Gemini text-embedding-004. **Free tier: 1,500 requests/day.**

Average tokens per target (estimated):
- Item: ~150 tokens (short social posts, truncated articles)
- Summary: ~300 tokens
- Entity: ~30 tokens
- Report: ~400 tokens

### Daily Request Budget at 5K Items/Day

| Target | Count/day | Requests/day (batched @ 100) | Cost/day |
|--------|-----------|------------------------------|---------|
| Items | 5,000 | ~50 | $0 (free tier) |
| Summaries | 30 | 30 (inline, 1 per call) | $0 |
| Entities | ~10 new/updated | 10 | $0 |
| Reports | 1-2 | 2 | $0 |
| Search queries | ~50 | 50 | $0 |
| **Total** | | **~142 req/day** | **$0/day** |

142 requests/day is well within the 1,500/day free tier limit. At 10K items/day
(~100 item batch requests + overhead), still under 250 req/day.

### If Free Tier Is Exceeded

Gemini paid tier: $0.004/1M tokens. At 5K items/day (765K tokens/day), that is
$0.003/day = ~$0.09/month. Negligible even if free tier runs out.

**Embeddings cost $0/month on free tier.** Cost is not a factor in any
embedding decision.

---

## 8. Phasing

### Core Embeddings (ships with build)

**Goal**: semantic search works from day one, embedding infrastructure exists.

Build:
1. `embed.ts` wrapper (Gemini SDK, retry, cost logging)
2. `embeddings` table + migration
3. Summary embedding (inline after Stage 1)
4. Report embedding (inline after Stage 3)
5. In-memory vector cache for summaries + reports (load on startup)
6. Semantic search endpoint (`GET /api/v1/search?mode=semantic`)
7. Search UI toggle (keyword vs semantic) on dashboard

**Why summaries first**: ~30 vectors/day, trivially fits in memory, immediately
useful for search. Proves the infrastructure without the volume challenge.

**Estimated effort**: 2-3 days.

### Full Semantic (ships with build)

Build:
1. Item embedding (batch cron every 15 min)
2. Entity embedding (inline on create/update)
3. Semantic dedup in normalize pipeline (cosine > 0.92 = duplicate)
4. Topic clustering for Stage 1 chunking (embedding-first grouping)
5. Entity disambiguation via embedding distance
6. Narrative detection via temporal clustering

Item embedding adds 150K vectors and ~525MB of storage. It needs the batch
pipeline, the pre-filter strategy for search, and careful integration with
the normalize layer. Build core embeddings first, then layer this on.

**Estimated effort**: 5-7 days.

### Migration Path

Core → full semantic requires no schema changes. The `embeddings` table handles
all target types from day one. The only additions are new callers (item
embedding, entity embedding) and new consumers (dedup, clustering).

If the embedding model changes (e.g., switch from Gemini to Voyage), the
`model` column in the `embeddings` table enables a clean migration: delete
vectors with the old model tag, let the batch job re-embed everything over
24-48 hours. No downtime.
