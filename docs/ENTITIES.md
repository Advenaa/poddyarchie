# Entity Resolution

Reference for how Podders v2 resolves, tracks, and prunes entities across sources.

## Resolution Algorithm

Stage 1 (Haiku) extracts entities from each chunk as `{ name, aliases[], type }`. The resolution layer maps these to canonical rows in `entities` via `entity_aliases`.

### Alias Lookup-Then-Insert

```
for each extracted entity (name, aliases, type):
    canonical = lowercase(name)
    entity_id = SELECT entity_id FROM entity_aliases WHERE alias = canonical

    if entity_id is NULL:
        entity_id = new ULID
        INSERT ... ON CONFLICT DO NOTHING INTO entities (id, name, type, relevance, first_seen, last_seen)
            VALUES (entity_id, name, type, 0, now, now)
        -- If IGNORE fired (concurrent batch wrote same name+type), fetch the winner:
        entity_id = SELECT id FROM entities WHERE name = name AND type = type
        INSERT ... ON CONFLICT DO NOTHING INTO entity_aliases (alias, entity_id) VALUES (canonical, entity_id)

    for each alias in aliases:
        normalized = lowercase(strip_prefix('$', alias))
        INSERT ... ON CONFLICT DO NOTHING INTO entity_aliases (alias, entity_id) VALUES (normalized, entity_id)

    UPDATE entities SET last_seen = now, relevance = <decay formula> WHERE id = entity_id
    INSERT entity_mention row
```

All aliases are **lowercased** before storage. The `$` prefix (common in crypto tickers like `$ETH`) is stripped so that `$ETH`, `ETH`, and `eth` all resolve to the same alias key.

`INSERT ... ON CONFLICT DO NOTHING` is the concurrency pattern. Two Stage 1 batches processing simultaneously may both try to insert the same entity or alias. The first writer wins; the second silently skips. No locks, no retries, no conflicts.

## CoinGecko Seeding

The alias table is pre-populated from CoinGecko's public token list so that common tokens resolve immediately without waiting for LLM extraction.

### Fetch

```
GET https://api.coingecko.com/api/v3/coins/list
```

Returns `[{ id: "ethereum", symbol: "eth", name: "Ethereum" }, ...]`. No API key required for this endpoint.

### Seed Logic

For each coin in the response:

1. Create an entity with `name = coin.name`, `type = 'token'`, `relevance = 0`.
2. Insert aliases: `lowercase(coin.name)`, `lowercase(coin.symbol)`, `lowercase(coin.id)`.
3. Skip duplicates via `INSERT ... ON CONFLICT DO NOTHING` (symbol collisions are common -- multiple tokens share `"uni"`, `"sol"`, etc.). First writer wins.

### Refresh Cadence

Run on first startup if the `entities` table is empty. Re-seed weekly via the daily cron (check `last_coingecko_seed` in `app_config`). New tokens get inserted; existing aliases are untouched.

### Indonesian Entity Seeds

In addition to CoinGecko tokens, seed Indonesian regulatory bodies and exchanges on first startup:

| Entity | Type | Aliases |
|--------|------|---------|
| OJK | company | ojk, otoritas jasa keuangan |
| Bappebti | company | bappebti |
| Bank Indonesia | company | bank indonesia, bi (context: Indonesian finance) |
| Bursa Efek Indonesia | company | bei, idx, bursa efek indonesia |
| Indodax | company | indodax |
| Tokocrypto | company | tokocrypto, tko |
| Pintu | company | pintu |
| Indonesian Rupiah | token | idr, rupiah |

**"BI" disambiguation**: "BI" maps to Bank Indonesia by default. In crypto-heavy context, Haiku may return "BI" meaning Binance — the alias resolution prefers the existing mapping. If Haiku returns `{ name: "Binance", aliases: ["BI"] }`, the `INSERT ... ON CONFLICT DO NOTHING` on "bi" keeps the Bank Indonesia mapping. Binance resolves via its own aliases ("binance", "bnb"). Acceptable trade-off for Indonesian market focus.

## Entity Types

The `type` column is a text enum:

| Type | Use for | Examples |
|------|---------|----------|
| `token` | Fungible tokens, coins, stablecoins | Ethereum, Bitcoin, USDC |
| `person` | Named individuals | Vitalik Buterin, CZ |
| `project` | Protocols, DAOs, L2s, dApps | Uniswap, Aave, Arbitrum |
| `company` | Corporate entities, firms, exchanges | Coinbase, a16z, OpenZeppelin |
| `event` | Named events, conferences, exploits | Token2049, Euler exploit |

### Ambiguity Rules

- If unsure, default to `project` (instruction in Stage 1 prompt).
- An L1/L2 chain that also has a token: use `project` (Ethereum the network, not ETH the asset). The token alias still resolves to it.
- An exchange that is also a company (Coinbase): use `company`.
- A named hack or exploit: use `event`.

### UNIQUE(name, type) Constraint

The `entities` table has `UNIQUE(name, type)`, which means "Mercury" can exist as both a `token` and a `company`. Different canonical entities, different IDs, different alias sets. The LLM's `type` assignment determines which one a mention resolves to. If the LLM is inconsistent (calling Mercury a `token` in one chunk and `company` in another), they become two separate entities -- see the merge procedure below.

## Relevance Decay

Relevance tracks how "hot" an entity is. Updated on every mention and decayed daily.

### Formula

```
relevance = relevance * 0.95^days_since_update + log(1 + mentions) * source_weight
```

Source weights: `Discord = 1.0`, `Twitter = 1.5`, `News = 2.0`.

News is weighted highest because news mentions are rarest and highest signal. Discord is lowest because volume is highest and noise is greatest.

### Concrete Examples

| Scenario | Relevance | Interpretation |
|----------|-----------|----------------|
| Mentioned 50x across Discord+Twitter today | ~8.0 | Very active, will appear in correlations |
| Mentioned 5x in Discord yesterday, nothing today | ~2.6 | Moderately relevant, decaying |
| Not mentioned for 7 days, was at 2.0 | ~1.4 | Fading -- `2.0 * 0.95^7` |
| Not mentioned for 30 days, was at 2.0 | ~0.43 | Nearly dormant |
| Not mentioned for 90 days, was at 2.0 | ~0.02 | Approaching prune threshold |
| Relevance = 0.5 | | Entity was recently active but thinning out |
| Relevance = 0.01 | | Effectively dead -- prune candidate |

### Daily Cron

Runs at `00:00 UTC`:

```sql
UPDATE entities SET relevance = relevance * 0.95;
```

Applied to every entity, every day. Mention-driven boosts happen in real time during Stage 1 processing.

## Pruning

### Retention Rule

An entity is pruned when **both** conditions are true:

- `relevance < 0.01`
- `last_seen < now - 90 days`

The dual condition prevents pruning a low-relevance entity that was just re-discovered (new `last_seen`) or a stale entity that still has residual relevance from high historical activity.

### Cascade

Pruning an entity deletes its dependents:

1. Delete from `entity_aliases` where `entity_id` matches.
2. Delete from `entity_mentions` where `entity_id` matches.
3. Delete the `entities` row.

Run inside a single transaction. The mention retention cron (delete mentions older than 90 days) runs independently and handles most mention cleanup before entity pruning fires.

## Cross-Source Disambiguation

The same entity appears differently across sources:

| Source | Raw text | Normalized alias |
|--------|----------|------------------|
| Discord | `ETH` | `eth` |
| Twitter | `$ETH` | `eth` (strip `$`) |
| News | `Ethereum` | `ethereum` |

All three resolve to the same canonical entity because the alias table maps `eth` and `ethereum` to the same `entity_id`. CoinGecko seeding pre-populates both the symbol (`eth`) and name (`ethereum`) as aliases.

For non-seeded entities, the LLM's `aliases` array in Stage 1 output builds the mapping. When Haiku processes a Discord chunk and returns `{ name: "Uniswap", aliases: ["UNI", "$UNI"] }`, it inserts `uniswap`, `uni` as aliases. A later Twitter chunk referencing `$UNI` strips the `$`, finds `uni` in the alias table, and resolves to the same entity.

### Edge Cases

- **Name collision across types**: "Mercury" as a token and "Mercury" as a company are separate entities with separate alias sets. The alias `mercury` can only point to one. First writer wins. If the company is seeded first, Discord messages about the Mercury token saying just "Mercury" will misresolve. Mitigation: the LLM usually provides disambiguating aliases (`$MERC` for the token) that create distinct alias paths. When misresolution is detected, use the merge procedure or manually reassign the alias.

- **Ticker collision**: Multiple tokens share `sol`, `uni`, etc. CoinGecko seeding inserts the first match. Later tokens with the same symbol are skipped (`INSERT ... ON CONFLICT DO NOTHING`). Acceptable for MVP -- the dominant token for a given ticker is usually the one that matters.

- **Alias hijacking**: A new entity could claim an alias that belongs to an existing entity. `INSERT ... ON CONFLICT DO NOTHING` prevents this -- existing aliases are never overwritten. To reassign an alias, delete it first, then re-insert.

## Merge Procedure

When two entities are discovered to be duplicates (e.g., the LLM created both "Ethereum" as `token` and "Ethereum" as `project`), merge them manually or via a future admin endpoint.

### Steps

Given a **survivor** entity (keep) and a **duplicate** entity (delete):

```sql
BEGIN TRANSACTION;

-- 1. Move aliases from duplicate to survivor
UPDATE entity_aliases SET entity_id = :survivor_id WHERE entity_id = :duplicate_id;

-- 2. Reassign mentions from duplicate to survivor
UPDATE entity_mentions SET entity_id = :survivor_id WHERE entity_id = :duplicate_id;

-- 3. Update survivor metadata
UPDATE entities SET
    relevance = MAX(
        (SELECT relevance FROM entities WHERE id = :survivor_id),
        (SELECT relevance FROM entities WHERE id = :duplicate_id)
    ),
    first_seen = MIN(
        (SELECT first_seen FROM entities WHERE id = :survivor_id),
        (SELECT first_seen FROM entities WHERE id = :duplicate_id)
    ),
    last_seen = MAX(
        (SELECT last_seen FROM entities WHERE id = :survivor_id),
        (SELECT last_seen FROM entities WHERE id = :duplicate_id)
    )
WHERE id = :survivor_id;

-- 4. Delete the duplicate
DELETE FROM entities WHERE id = :duplicate_id;

COMMIT;
```

Choose the survivor by: prefer the entity with more mentions, or the one with the correct `type`. The survivor keeps its `name` and `type` -- update those manually if the duplicate had the better canonical form.

The alias `PRIMARY KEY` constraint means step 1 will fail if both entities share an alias (shouldn't happen, but guard with `ON CONFLICT DO NOTHING` if automating). Mention reassignment in step 2 is safe because `entity_mentions.id` is unique per row, not per entity.
