# Database Queries Reference

Every SQL query the system needs, organized by module. All live in `src/db/queries.ts`.

## 1. Items

```typescript
function insertItem(item: RawItem, contentHash: string, status: string, originalLanguage?: string, translated?: boolean, filterReason?: string, contentAnchor?: string): void
```
```sql
INSERT INTO items (id, source, source_id, author, content, timestamp, url, engagement, content_hash, status, original_language, translated, attachments, filter_reason, content_anchor, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16)
```

```typescript
function claimBatch(batchId: string, source: string, sourceId: string, windowStart: number, windowEnd: number): number
```
```sql
UPDATE items SET batch_id = $1, status = 'processing'
WHERE status = 'ready' AND source = $2 AND source_id = $3
  AND timestamp BETWEEN $4 AND $5
```
Critical concurrency query. Atomic claim prevents double-processing across overlapping cron runs.

```typescript
function markProcessed(batchId: string): void
```
```sql
UPDATE items SET status = 'processed' WHERE batch_id = $1
```

```typescript
function resetCrashed(): number
```
```sql
UPDATE items SET status = 'ready', batch_id = NULL WHERE status = 'processing'
```
Runs on startup. Returns count of recovered items.

```typescript
function getItemsByBatch(batchId: string): Item[]
```
```sql
SELECT * FROM items WHERE batch_id = $1 ORDER BY timestamp ASC
```

```typescript
function getFilteredByInjection(limit: number, offset: number): Item[]
```
```sql
SELECT * FROM items
WHERE status = 'filtered' AND filter_reason = 'injection_detected'
ORDER BY created_at DESC
LIMIT $1 OFFSET $2
```
Dashboard review of items flagged by InstructDetector (Layer 2 prompt injection defense). Items are preserved for manual false-positive review.

## 2. Summaries

```typescript
function insertSummary(s: ChunkSummary): void
```
```sql
INSERT INTO summaries (id, source, source_id, window_start, window_end, body, sentiment, urgency, item_count, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
```

```typescript
function getSummariesByTimeWindow(start: number, end: number): Summary[]
```
```sql
SELECT * FROM summaries WHERE created_at >= $1 AND created_at <= $2
```
Uses `created_at`, not `window_start`/`window_end`. Avoids missing summaries whose micro-batch window spans the boundary.

```typescript
function getSummariesBySource(source: string, sourceId: string): Summary[]
```
```sql
SELECT * FROM summaries WHERE source = $1 AND source_id = $2 ORDER BY created_at DESC
```

## 3. Reports

```typescript
function insertReport(r: MarketReport): void
```
```sql
INSERT INTO reports (id, date, type, body, tldr, sentiment, delivery_status, created_at)
VALUES ($1, $2, $3, $4, $5, $6, 'pending', $7)
```

```typescript
function dailyReportExists(date: string): boolean
```
```sql
SELECT id FROM reports WHERE date = $1 AND type = 'daily'
```
Dedup check before insert. Flash reports skip this (multiples per day allowed).

```typescript
function getLatestReport(): Report | null
```
```sql
SELECT * FROM reports WHERE type = 'daily' ORDER BY created_at DESC LIMIT 1
```

```typescript
function getReportById(id: string): Report | null
```
```sql
SELECT * FROM reports WHERE id = $1
```

```typescript
function listReports(limit: number, offset: number, type?: string): Report[]
```
```sql
SELECT id, date, type, tldr, sentiment, delivery_status, created_at
FROM reports
WHERE ($1::text IS NULL OR type = $1)
ORDER BY created_at DESC
LIMIT $2 OFFSET $3
```

## 4. Sources

```typescript
function insertSource(source: string, sourceId: string, label: string | null, trustWeight?: number): void
```
```sql
INSERT INTO sources (source, source_id, label, enabled, priority, trust_weight, added_at)
VALUES ($1, $2, $3, true, 1, $4, $5)
```

```typescript
function updateSource(source: string, sourceId: string, fields: { enabled?: number; label?: string }): void
```
```sql
UPDATE sources SET enabled = COALESCE($1, enabled), label = COALESCE($2, label)
WHERE source = $3 AND source_id = $4
```

```typescript
function deleteSource(source: string, sourceId: string): void
```
```sql
DELETE FROM sources WHERE source = $1 AND source_id = $2
```
Keeps historical items. Does not cascade to `source_state` (orphaned rows are harmless, cleaned by retention).

```typescript
function listSourcesWithState(): SourceWithState[]
```
```sql
SELECT s.source, s.source_id, s.label, s.enabled, s.priority, s.added_at,
       ss.status AS state_status, ss.last_fetched_at, ss.last_id,
       ss.error_count, ss.last_error, ss.next_retry_at
FROM sources s
LEFT JOIN source_state ss ON s.source = ss.source AND s.source_id = ss.source_id
ORDER BY s.source, s.label
```

```typescript
function getSourceTrustWeight(source: string, sourceId: string): { trust_weight: number } | null
```
```sql
SELECT trust_weight FROM sources WHERE source = $1 AND source_id = $2
```

```typescript
function adjustSourceTrustWeight(source: string, sourceId: string, delta: number, floor: number, ceiling: number): void
```
```sql
UPDATE sources SET trust_weight = GREATEST($3, LEAST($4, trust_weight + $5))
WHERE source = $1 AND source_id = $2
```
Auto-adjustment after flash alert confirmation. `delta` is +0.05 (confirmed) or -0.03 (unconfirmed). `floor` and `ceiling` are ±0.2 from the initial manual weight.

## 5. Source State

```typescript
function upsertSourceState(source: string, sourceId: string, lastId: string | null, fetchedAt: number): void
```
```sql
INSERT INTO source_state (source, source_id, status, last_fetched_at, last_id, error_count)
VALUES ($1, $2, 'active', $3, $4, 0)
ON CONFLICT(source, source_id) DO UPDATE SET
  status = 'active', last_fetched_at = $3, last_id = COALESCE($4, source_state.last_id),
  error_count = 0, last_error = NULL, next_retry_at = NULL
```

```typescript
function backoffSource(source: string, sourceId: string, error: string, now: number): void
```
```sql
UPDATE source_state SET
  status = CASE WHEN error_count >= 4 THEN 'disabled' ELSE 'backoff' END,
  error_count = error_count + 1,
  last_error = $1,
  next_retry_at = $2 + LEAST(2000 * POWER(2, error_count), 3600000)
WHERE source = $3 AND source_id = $4
```
Disables at `error_count >= 5` (current value is 4, incremented to 5 by this update).

```typescript
function disableSource(source: string, sourceId: string, reason: string): void
```
```sql
UPDATE source_state SET status = 'disabled', last_error = $1
WHERE source = $2 AND source_id = $3
```
Used for immediate disable (e.g., Discord 401/403 token revocation).

```typescript
function resetRecoveredSources(now: number): number
```
```sql
UPDATE source_state SET status = 'active', error_count = 0, last_error = NULL, next_retry_at = NULL
WHERE status = 'backoff' AND next_retry_at <= $1
```
Health check cron (every 5min). Returns count of recovered sources.

## 6. Entities

```typescript
function lookupAlias(alias: string): string | null
```
```sql
SELECT entity_id FROM entity_aliases WHERE alias = $1
```

```typescript
function lookupAliasWithContext(alias: string, contextKey: string): string | null
```
```sql
SELECT entity_id FROM entity_aliases WHERE alias = $1 AND context_key = $2
```
Context-aware alias lookup. `contextKey` encodes the domain context (e.g., `'defi'`, `'gaming'`). Falls back to `lookupAlias` (no context) if no context-specific match exists.

```typescript
function insertContextAlias(alias: string, entityId: string, contextKey: string): void
```
```sql
INSERT INTO entity_aliases (alias, entity_id, context_key) VALUES ($1, $2, $3)
ON CONFLICT DO NOTHING
```
Saves a disambiguation result as a context-aware alias. Future lookups with the same alias+context resolve without an LLM call.

```typescript
function getEntityAliases(entityId: string): { alias: string; context_key: string | null }[]
```
```sql
SELECT alias, context_key FROM entity_aliases WHERE entity_id = $1
```
Returns all aliases (with optional context keys) for an entity. Used during context-based disambiguation (Step 2) to check for co-occurring entity overlap.

```typescript
function insertEntity(id: string, name: string, type: string, now: number): void
```
```sql
INSERT INTO entities (id, name, type, relevance, first_seen, last_seen)
VALUES ($1, $2, $3, 0, $4, $5)
ON CONFLICT DO NOTHING
```

```typescript
function getEntityByNameType(name: string, type: string, status?: string): string | null
```
```sql
SELECT id FROM entities WHERE name = $1 AND type = $2
  AND ($3::text IS NULL OR status = $3)
```
Fallback after `INSERT ... ON CONFLICT DO NOTHING` fires (concurrent batch wrote same name+type). Optional `status` param filters by `'active'` or `'archived'`.

```typescript
function insertAlias(alias: string, entityId: string): void
```
```sql
INSERT INTO entity_aliases (alias, entity_id, context_key) VALUES ($1, $2, '')
ON CONFLICT DO NOTHING
```

```typescript
function upsertMention(id: string, entityId: string, source: string, summaryId: string, sentiment: number | null, count: number, now: number): void
```
```sql
INSERT INTO entity_mentions (id, entity_id, source, summary_id, sentiment, mention_count, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7)
```

```typescript
function updateEntityRelevance(entityId: string, mentions: number, sourceWeight: number, now: number): void
```
```sql
UPDATE entities SET
  relevance = relevance + LN(1 + $1) * $2,
  last_seen = $3
WHERE id = $4
```
Real-time boost on mention. Daily decay applied separately.

```typescript
function decayAllEntities(): void
```
```sql
UPDATE entities SET relevance = relevance * 0.95
WHERE status = 'active' AND relevance > 0.01
```
Only decays active entities. The `relevance > 0.01` guard skips entities already below the archive threshold.

```typescript
function archiveDeadEntities(threshold: number, cutoff: number): number
```
```sql
UPDATE entities SET status = 'archived'
WHERE relevance < $1 AND last_seen < $2 AND status = 'active'
```
Archives instead of deleting. `threshold` = 0.01, `cutoff` = now - 90 days. Aliases and mentions are preserved for reactivation.

```typescript
function reactivateEntity(entityId: string, relevance: number): void
```
```sql
UPDATE entities SET status = 'active', relevance = $2
WHERE id = $1 AND status = 'archived'
```
Promotes an archived entity back to active with a restart relevance (typically 0.5). Called when a previously archived entity is re-mentioned.

## 7. Correlation

```typescript
function getCorrelatedEntities(windowStart: number): CorrelatedEntity[]
```
```sql
SELECT e.name, e.type,
       COUNT(DISTINCT em.source) AS source_count,
       AVG(em.sentiment) AS avg_sentiment,
       string_agg(DISTINCT em.source, ',') AS sources
FROM entity_mentions em
JOIN entities e ON e.id = em.entity_id
WHERE em.created_at > $1 AND e.status = 'active'
GROUP BY e.id, e.name, e.type
HAVING COUNT(DISTINCT em.source) >= 2
ORDER BY source_count DESC, ABS(avg_sentiment) DESC
LIMIT 20
```
`windowStart` = now - 24h. Cross-source signal: entities in 2+ sources.

## 7.5. Narratives

```typescript
function insertNarrative(id: string, name: string, date: string, memberCount: number, avgSentiment: number | null, signalStrength: string, summaryIds: string[], now: number): void
```
```sql
INSERT INTO narratives (id, name, date, member_count, avg_sentiment, signal_strength, summary_ids, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
```

```typescript
function getNarrativesByDate(date: string): Narrative[]
```
```sql
SELECT * FROM narratives WHERE date = $1 ORDER BY member_count DESC
```

```typescript
function getPreviousDayNarratives(date: string): Narrative[]
```
```sql
SELECT * FROM narratives WHERE date = ($1::date - INTERVAL '1 day')::date ORDER BY member_count DESC
```
Used for signal strength classification (growth comparison vs previous day).

## 8. App Config

```typescript
function getConfig(key: string): string | null
```
```sql
SELECT value FROM app_config WHERE key = $1
```

```typescript
function setConfig(key: string, value: string): void
```
```sql
INSERT INTO app_config (key, value) VALUES ($1, $2)
ON CONFLICT(key) DO UPDATE SET value = EXCLUDED.value
```

```typescript
function seedDefaults(defaults: Record<string, string>): void
```
```sql
INSERT INTO app_config (key, value) VALUES ($1, $2)
ON CONFLICT DO NOTHING
```
Called on startup. Seeds: `webhook_url`, `digest_time` ('09:00'), `timezone` ('Asia/Jakarta'), `api_key_hash`.

## 9. LLM Usage

```typescript
function insertLLMUsage(id: string, stage: string, model: string, inputTokens: number, outputTokens: number, costUsd: number, now: number): void
```
```sql
INSERT INTO llm_usage (id, stage, model, input_tokens, output_tokens, cost_usd, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7)
```
`stage` is one of: `'summarize'`, `'synthesize'`, `'pre-summarize'`, `'urgency-classify'`, `'pulse'`, `'chat'`, `'translate'`, `'escalate'`, `'narrative-cluster'`, `'entity-disambiguate'`.

```typescript
function getDailyCost(dayStart: number, dayEnd: number): { totalCost: number; byStage: { stage: string; cost: number }[] }
```
```sql
SELECT stage, SUM(cost_usd) AS cost, SUM(input_tokens) AS input_tokens, SUM(output_tokens) AS output_tokens
FROM llm_usage
WHERE created_at >= $1 AND created_at < $2
GROUP BY stage
```

## 10. Source Rate History

```typescript
function upsertSourceRateHistory(source: string, sourceId: string, hourOfDay: number, dayOfWeek: number, newRate: number): void
```
```sql
INSERT INTO source_rate_history (source, source_id, hour_of_day, day_of_week, avg_rate, sample_count, updated_at)
VALUES ($1, $2, $3, $4, $5, 1, $6)
ON CONFLICT (source, source_id, hour_of_day, day_of_week) DO UPDATE SET
  avg_rate = (source_rate_history.avg_rate * source_rate_history.sample_count + $5)
             / (source_rate_history.sample_count + 1),
  sample_count = source_rate_history.sample_count + 1,
  updated_at = $6
```
Rolling average. Called after each Discord processing cycle to update the baseline lambda for the Poisson claim gate. `newRate` = items processed / elapsed hours.

```typescript
function getSourceRateHistory(source: string, sourceId: string, hourOfDay: number, dayOfWeek: number): { avg_rate: number } | null
```
```sql
SELECT avg_rate FROM source_rate_history
WHERE source = $1 AND source_id = $2 AND hour_of_day = $3 AND day_of_week = $4
```
Returns the learned lambda for a given source at a specific hour-of-day and day-of-week. Used by the Poisson claim gate to compute expected message rate. Returns `null` if no history exists (falls back to a default lambda).

## 11. Embeddings

```typescript
function upsertEmbedding(targetType: string, targetId: string, vector: Buffer, model: string): void
```
```sql
INSERT INTO embeddings (id, target_type, target_id, model, dimensions, vector, created_at)
VALUES ($1, $2, $3, $4, $5, $6, $7)
ON CONFLICT (target_type, target_id) DO UPDATE SET
  model = EXCLUDED.model,
  dimensions = EXCLUDED.dimensions,
  vector = EXCLUDED.vector,
  created_at = EXCLUDED.created_at
```
Idempotent insert. Re-embedding the same target (e.g., after model migration) overwrites the previous vector.

```typescript
function searchEmbeddings(vector: Buffer, targetType: string, limit: number): { target_id: string; target_type: string; similarity: number }[]
```
```sql
SELECT target_id, target_type, vector FROM embeddings
WHERE target_type = $1
ORDER BY created_at DESC
```
Returns all vectors for the given `target_type`. Cosine similarity is computed in JS (not SQL) against the query `vector`. Results are sorted by similarity descending and truncated to `limit` in application code. See [docs/EMBEDDINGS.md](./docs/EMBEDDINGS.md) for why BYTEA + JS cosine beats pgvector at our scale.

```typescript
function deleteEmbeddingsByModel(excludeModel: string): number
```
```sql
DELETE FROM embeddings WHERE model != $1
```
Model migration cleanup. After switching embedding models, delete all vectors from the old model so the batch re-embed job regenerates them. Returns count of deleted rows.

```typescript
function deleteEmbeddingsByRef(targetType: string, targetId: string): void
```
```sql
DELETE FROM embeddings WHERE target_type = $1 AND target_id = $2
```
Cascading delete. Called when the referenced record (item, summary, entity, report) is deleted.

## 11.5. RAG Chat Retrieval Tools

Queries backing the three retrieval tools available to Sonnet during chat (Decision 07).

**keyword_search** — entity mention lookup:
```typescript
function getEntityMentions(entityName: string, windowStart: number, windowEnd: number): EntityMention[]
```
```sql
SELECT em.*, e.name, e.type
FROM entity_mentions em
JOIN entities e ON e.id = em.entity_id
JOIN entity_aliases ea ON ea.entity_id = e.id
WHERE ea.alias = $1 AND em.created_at BETWEEN $2 AND $3
ORDER BY em.created_at DESC
```
`entityName` is lowercased before lookup. Returns mention counts, sentiment, and source breakdown for the matched entity.

**read_raw** — read original source message:
```typescript
function getItemById(itemId: string): Item | null
```
```sql
SELECT * FROM items WHERE id = $1
```
Used by the `read_raw` tool to retrieve the original unprocessed message for verification or quoting.

## 12. Retention

All run in the daily retention cron (after entity decay).

```typescript
function deleteOldItems(cutoff: number): number
```
```sql
DELETE FROM embeddings WHERE target_type = 'item'
  AND target_id IN (SELECT id FROM items WHERE status = 'processed' AND created_at < $1);
DELETE FROM items WHERE status = 'processed' AND created_at < $1
```
`cutoff` = now - 30 days. Summaries are the archival layer. Embeddings are cascade-deleted first — the `embeddings` table has no FK constraint, so the application must delete them explicitly before removing the referenced items. Both statements run in a single transaction.

```typescript
function deleteOldSummaries(cutoff: number): number
```
```sql
DELETE FROM embeddings WHERE target_type = 'summary'
  AND target_id IN (SELECT id FROM summaries WHERE created_at < $1);
DELETE FROM summaries WHERE created_at < $1
```
`cutoff` = now - 90 days. Embedding cascade same pattern as `deleteOldItems`.

```typescript
function deleteOldMentions(cutoff: number): number
```
```sql
DELETE FROM entity_mentions WHERE created_at < $1
```
`cutoff` = now - 90 days. Runs before entity pruning.

```typescript
function reindexSearch(): void
```
```sql
REINDEX INDEX items_content_tsvector_idx
```
Rebuilds the full-text search index after bulk deletes. Run last in the retention cron.
