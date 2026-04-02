# Decision 4: Two-Tier Entity Memory (Active/Archived)

## Decision
Option B — Entities demote to archived instead of being deleted. When re-mentioned, promote back to active with history intact.

## Why This Decision
- Every 2026 paper says "don't delete, demote" — flat decay + hard delete loses history
- Multi-Layered Memory Architectures (Mar 2026) describes exactly this: working → archived layers
- HALO (May 2025) shows different fact types have different half-lives — our flat 5%/day is crude but archived entities preserve history for re-evaluation
- Current design hard-deletes at 90 days. CZ disappears, then when re-mentioned, system treats him as brand new. All context lost.
- Full temporal graph (Option C) is 20x engineering effort for v1

## Research Papers
- Multi-Layered Memory Architectures (Mar 2026) — arxiv.org/abs/2603.29194 — Working → archived layers prevent entity drift
- HALO: Half-Life Fact Filtering (May 2025) — arxiv.org/abs/2505.07509 — Facts have different decay rates by type
- Agentic Memory (Jan 2026) — arxiv.org/abs/2601.01885 — Memory operations as tool calls, including discard
- Graph-based Agent Memory Survey (Feb 2026) — arxiv.org/abs/2602.05665 — Decay during ranking, not deletion
- MemoTime (Oct 2025) — arxiv.org/abs/2510.13614 — Re-weight by retrieval frequency + temporal freshness, +24% accuracy
- Zep/Graphiti (Jan 2025) — arxiv.org/abs/2501.13956 — Temporal edges with valid_from/valid_to
- MAGMA (Jan 2026) — arxiv.org/abs/2601.03236 — Multi-graph beats single-graph on long-horizon memory
- Entity State Tuning (Feb 2026) — arxiv.org/abs/2602.12389 — Track entity state evolution over time

## Implementation

### Schema Changes
```sql
ALTER TABLE entities ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
-- status: 'active' or 'archived'
-- No other schema changes needed
```

### Decay Logic Change
```sql
-- Old: DELETE after 90 days
DELETE FROM entities WHERE relevance < 0.01 AND last_seen_at < NOW() - INTERVAL '90 days';

-- New: ARCHIVE instead of delete
UPDATE entities SET status = 'archived'
WHERE relevance < 0.01
  AND last_seen_at < NOW() - INTERVAL '90 days'
  AND status = 'active';
```

### Reactivation Logic
```typescript
async function resolveEntity(name: string, type: string): Promise<Entity> {
  // Check active first
  let entity = await getEntityByNameType(name, type, 'active');
  if (entity) return entity;
  
  // Check archived
  entity = await getEntityByNameType(name, type, 'archived');
  if (entity) {
    // Promote back to active with history
    await updateEntity(entity.id, {
      status: 'active',
      relevance: 0.5, // restart at mid-level, not zero
    });
    return entity;
  }
  
  // Truly new entity
  return await createEntity(name, type);
}
```

### What's Preserved When Archived
- Entity name, type, aliases
- Last known sentiment score
- Last known relevance (before it decayed to threshold)
- Last seen timestamp
- Total historical mention count
- All alias mappings in entity_aliases table

### Query Changes
- Reports/correlation: `WHERE status = 'active'` — only active entities appear in reports
- Chat/search: no status filter — can find archived entities too
- Dashboard: show archived entities in a separate "inactive" section

### Cost Impact
Zero. One column change. Slightly more storage (entities aren't deleted), but at ~1000 entities max, this is negligible.

## Upgrade Path
- **v2: Half-life per entity type** — Identity entities (people, companies) decay slower than price-related entities (token prices, TVL). Add `entity_type_decay_rate` config. CZ decays at 2%/day, "BTC at $60K" decays at 10%/day.
- **v3: Full temporal graph** — Add `valid_from`/`valid_to` to entity facts. "CZ is CEO" valid 2017-2023. "Richard Teng is CEO" valid 2023-present. Add `entity_facts` table. Enables "CZ, former Binance CEO who stepped down after AML plea." Requires new schema, ingestion logic, timeline query builder. Build only if reports feel shallow after months of real data.
- **v3+: MAGMA-style multi-graph** — Separate semantic, temporal, causal, and entity relation graphs. Research-grade. Only if Podders becomes a platform serving multiple users with different analytical needs.
