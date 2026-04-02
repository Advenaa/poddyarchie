# Testing Spec — Podders v2

Four test types: golden file, normalize pipeline, entity resolution, API endpoint. Plus one integration script.

## 1. Test File Structure

| Test file | Source file | What it covers |
|-----------|-------------|----------------|
| `test/unit/summarize.test.ts` | `src/process/summarize.ts` | Stage 1 Haiku output parsing, chunking, batch claiming |
| `test/unit/synthesize.test.ts` | `src/process/synthesize.ts` | Stage 3 Sonnet output parsing, daily/flash branching |
| `test/unit/correlate.test.ts` | `src/process/correlate.ts` | Stage 2 SQL cross-source grouping |
| `test/unit/dedup.test.ts` | `src/normalize/dedup.ts` | Content hash, URL match, URL expansion |
| `test/unit/filter.test.ts` | `src/normalize/filter.ts` | Spam rules (EN + ID), language detection |
| `test/unit/normalize.test.ts` | `src/normalize/index.ts` | Orchestrator: truncation, dedup, filter, language in sequence |
| `test/unit/entities.test.ts` | `src/knowledge/entities.ts` | Alias lookup, insert races, CoinGecko seed, ID seeds |
| `test/unit/decay.test.ts` | `src/knowledge/decay.ts` | Relevance decay math, pruning threshold |
| `test/unit/llm.test.ts` | `src/llm.ts` | Error taxonomy handling, retry logic, cost logging |
| `test/unit/webhook.test.ts` | `src/deliver/webhook.ts` | Retry backoff, status codes, embed formatting |
| `test/unit/server.test.ts` | `src/server.ts` | All 15 API endpoints |
| `test/unit/url-validator.test.ts` | `src/url-validator.ts` | SSRF protection, private IP rejection |
| `test/unit/config.test.ts` | `src/config.ts` | Env loading, defaults, missing required vars |
| `test/unit/scheduler.test.ts` | `src/scheduler.ts` | Mutex guard, crash recovery, cron wiring |
| `test/integration/pipeline.test.ts` | Full pipeline | Seed data through normalize to report delivery |
| `test/prompts/stage1-regression.ts` | Stage 1 prompt | Diff Haiku outputs against saved baselines |
| `test/prompts/stage3-regression.ts` | Stage 3 prompt | Diff Sonnet outputs against saved baselines |

Fixtures directory: `test/fixtures/`.

---

## 2. Golden File Tests

Golden files capture real LLM responses so CI never calls the API. Each fixture is a JSON file in `test/fixtures/` containing the raw LLM response string and the expected parsed output.

### Fixture format

```json
{
  "description": "What this fixture tests",
  "llmResponse": "{ raw JSON string as returned by the model }",
  "expectedParsed": { "the zod-validated output object" },
  "shouldPass": true
}
```

### Example fixtures

**`test/fixtures/stage1-discord-normal.json`** — Stage 1 (Haiku) normal response:

```json
{
  "description": "Stage 1: Discord crypto channel, 12 items, normal summary",
  "llmResponse": "{\"summary\":\"Discussion focused on ETH staking yields dropping post-Dencun and speculation about Solana ETF approval timeline. Two users shared links to the same Coindesk article.\",\"keyEvents\":[\"ETH staking APR dropped to 3.2%\",\"Solana ETF application filed by VanEck\"],\"urgency\":\"routine\",\"entities\":[{\"name\":\"Ethereum\",\"aliases\":[\"ETH\",\"$ETH\"],\"type\":\"token\",\"mentionCount\":5,\"sentiment\":0.1},{\"name\":\"Solana\",\"aliases\":[\"SOL\",\"$SOL\"],\"type\":\"token\",\"mentionCount\":3,\"sentiment\":0.2},{\"name\":\"VanEck\",\"aliases\":[],\"type\":\"company\",\"mentionCount\":2,\"sentiment\":0.1}]}",
  "expectedParsed": {
    "summary": "Discussion focused on ETH staking yields dropping post-Dencun and speculation about Solana ETF approval timeline. Two users shared links to the same Coindesk article.",
    "keyEvents": ["ETH staking APR dropped to 3.2%", "Solana ETF application filed by VanEck"],
    "urgency": "routine",
    "urgency": "routine",
    "entities": [
      { "name": "Ethereum", "aliases": ["ETH", "$ETH"], "type": "token" },
      { "name": "Solana", "aliases": ["SOL", "$SOL"], "type": "token" },
      { "name": "VanEck", "aliases": [], "type": "company" }
    ]
  },
  "shouldPass": true
}
```

**`test/fixtures/stage3-daily-report.json`** — Stage 3 (Sonnet) daily report:

```json
{
  "description": "Stage 3: Daily report from 28 summaries, 4 correlated entities",
  "llmResponse": "{\"date\":\"2026-03-31\",\"type\":\"daily\",\"tldr\":\"ETH staking yields compressed further while Solana ecosystem saw renewed developer activity. Bank Indonesia held rates steady, IDR stable against USD.\",\"keyEvents\":[\"ETH staking APR at 3.2%, lowest since merge\",\"VanEck filed Solana ETF application\",\"BI rate decision: hold at 6.25%\",\"Indodax daily volume up 40% WoW\"],\"sentiment\":-0.05,\"entitySentiment\":[{\"name\":\"Ethereum\",\"sentiment\":-0.2,\"trend\":\"declining\"},{\"name\":\"Solana\",\"sentiment\":0.4,\"trend\":\"rising\"},{\"name\":\"Bank Indonesia\",\"sentiment\":0.0,\"trend\":\"stable\"}],\"itemCount\":482,\"sourceCount\":8}",
  "expectedParsed": {
    "date": "2026-03-31",
    "type": "daily",
    "tldr": "ETH staking yields compressed further while Solana ecosystem saw renewed developer activity. Bank Indonesia held rates steady, IDR stable against USD.",
    "keyEvents": [
      "ETH staking APR at 3.2%, lowest since merge",
      "VanEck filed Solana ETF application",
      "BI rate decision: hold at 6.25%",
      "Indodax daily volume up 40% WoW"
    ],
    "sentiment": -0.05,
    "entitySentiment": [
      { "name": "Ethereum", "sentiment": -0.2, "trend": "declining" },
      { "name": "Solana", "sentiment": 0.4, "trend": "rising" },
      { "name": "Bank Indonesia", "sentiment": 0.0, "trend": "stable" }
    ],
    "itemCount": 482,
    "sourceCount": 8
  },
  "shouldPass": true
}
```

**`test/fixtures/stage1-malformed.json`** — Malformed LLM response:

```json
{
  "description": "Stage 1: model returns markdown-wrapped JSON with missing required fields",
  "llmResponse": "```json\n{\"summary\":\"Some discussion about tokens\",\"sentiment\":\"positive\"}\n```",
  "expectedParsed": null,
  "shouldPass": false,
  "expectedError": "keyEvents is required; sentiment must be number; urgency is required; entities is required"
}
```

### Test that validates parsing (`test/unit/summarize.test.ts` excerpt)

```typescript
import { readFileSync } from 'fs';
import { ChunkSummarySchema } from '../src/process/summarize.js';

const fixtures = [
  'stage1-discord-normal.json',
  'stage1-malformed.json',
  // ... all stage1-*.json files
];

for (const file of fixtures) {
  const fixture = JSON.parse(readFileSync(`test/fixtures/${file}`, 'utf-8'));

  test(`golden: ${fixture.description}`, () => {
    // Strip markdown fences if present (real pre-processing step)
    const cleaned = fixture.llmResponse.replace(/^```json\n?|\n?```$/g, '');

    let parsed;
    try {
      parsed = ChunkSummarySchema.safeParse(JSON.parse(cleaned));
    } catch {
      parsed = { success: false };
    }

    if (fixture.shouldPass) {
      expect(parsed.success).toBe(true);
      expect(parsed.data).toEqual(fixture.expectedParsed);
    } else {
      expect(parsed.success).toBe(false);
    }
  });
}
```

---

## 3. Normalize Pipeline Tests

All in `test/unit/normalize.test.ts`, `dedup.test.ts`, `filter.test.ts`. Uses an in-memory test DB (Postgres with isolated schema) seeded per test.

### Dedup (`dedup.test.ts`)

| # | Case | Input | Expected |
|---|------|-------|----------|
| 1 | Exact hash match | Two items: same source, sourceId, first 200 chars of content | Second item dropped |
| 2 | Different source, same content | Discord item and RSS item with identical content prefix | Both kept (hash includes source) |
| 3 | URL match, same URL | Discord item with `https://coindesk.com/article-1`, then Twitter item with same URL | Second dropped |
| 4 | URL match after expansion | Item with `https://t.co/abc123` that resolves to `https://coindesk.com/article-1`, second item already has expanded URL | Second dropped |
| 5 | URL match, different URL | Two items with different URLs | Both kept |
| 6 | No URL, no hash collision | Two distinct items, no URLs | Both kept |
| 7 | Hash boundary — 200 char cutoff | Two items identical in first 200 chars, differ at char 201 | Second dropped (by design) |
| 8 | Hash boundary — differ within 200 | Two items that differ at char 50 | Both kept |

### Spam filter (`filter.test.ts`)

**English patterns:**

| # | Case | Input content | Expected |
|---|------|---------------|----------|
| 1 | Bot account | author = "MEE6", content = "Level up!" | Filtered |
| 2 | Sub-5-word post | "nice one bro" | Filtered |
| 3 | GM/GN | "gm everyone" | Filtered |
| 4 | GM/GN mixed case | "GM" | Filtered |
| 5 | Borderline passes | "good morning here is my analysis of ETH" (>5 words, has substance) | Passes |

**Indonesian patterns:**

| # | Case | Input content | Expected |
|---|------|---------------|----------|
| 6 | "wm" | "wm" | Filtered |
| 7 | "done min" | "done min" | Filtered |
| 8 | "sudah min" | "sudah min" | Filtered |
| 9 | "gas!" | "gas!" | Filtered |
| 10 | "gas" no punctuation | "gas" | Filtered |
| 11 | "mantap" | "mantap" | Filtered |
| 12 | Single emoji | "🔥" | Filtered |
| 13 | Airdrop copypasta | "Join airdrop now! RT and follow @... tag 3 friends" | Filtered |
| 14 | Substantive ID | "Menurut analisis saya, ETH bisa tembus 5000 USD" | Passes |

### Language detection

| # | Case | Content (truncated) | franc result | Expected |
|---|------|---------------------|-------------|----------|
| 1 | English | "Ethereum staking yields dropped..." | `eng` | Passes |
| 2 | Indonesian | "Harga Bitcoin hari ini naik..." | `ind` | Passes |
| 3 | Japanese | "ビットコインの価格が..." | `jpn` | Filtered |
| 4 | Korean | "이더리움 스테이킹..." | `kor` | Filtered |
| 5 | Undetermined (short) | "ETH 5k?" | `und` | Passes (und allowed) |
| 6 | Mixed EN/ID | "ETH lagi bullish banget hari ini" | `ind` or `eng` | Passes (either accepted) |

### Truncation

| # | Case | Content length | Expected |
|---|------|---------------|----------|
| 1 | Under limit | 500 chars | Unchanged |
| 2 | At limit | 20,000 chars | Unchanged |
| 3 | Over limit | 25,000 chars | Truncated to 20,000 chars |
| 4 | Way over limit | 100,000 chars (full RSS article) | Truncated to 20,000 chars |

---

## 4. Entity Resolution Tests

All in `test/unit/entities.test.ts`. Each test starts with a fresh in-memory test DB (Postgres with isolated schema) with schema applied.

### Alias lookup

| # | Case | Setup | Action | Expected |
|---|------|-------|--------|----------|
| 1 | Hit — exact alias | Seed entity "Ethereum" with alias "eth" | Resolve `{ name: "ETH" }` | Returns existing Ethereum entity_id |
| 2 | Hit — dollar prefix | Seed entity "Ethereum" with alias "eth" | Resolve `{ name: "$ETH" }` | "$" stripped, matches "eth", returns existing entity_id |
| 3 | Hit — CoinGecko alias | Seed via CoinGecko: id="ethereum", symbol="eth", name="Ethereum" | Resolve `{ name: "Ether" }` with alias "ethereum" | Matches CoinGecko-seeded alias |
| 4 | Miss — new entity | Empty DB | Resolve `{ name: "NewProject", type: "project" }` | Creates new entity + alias "newproject" |
| 5 | Miss — adds aliases | Empty DB | Resolve `{ name: "Solana", aliases: ["SOL", "$SOL"] }` | Creates entity + aliases: "solana", "sol" |

### Concurrent batch writes

| # | Case | Action | Expected |
|---|------|--------|----------|
| 6 | Two batches insert same entity | Batch A and B both resolve `{ name: "Ethereum", type: "token" }` simultaneously | `ON CONFLICT DO NOTHING` — first wins, second does SELECT to get winner's ID |
| 7 | Two batches insert same alias | Both try to insert alias "eth" for different entity_ids | First writer wins alias mapping; second's ON CONFLICT DO NOTHING is a no-op |
| 8 | Mention from losing batch | Batch B's entity insert was ignored, but it fetched the winner's ID | entity_mention row correctly references the winner's entity_id |

### CoinGecko seeding

| # | Case | Action | Expected |
|---|------|--------|----------|
| 9 | Seed creates entities | Feed mock CoinGecko list: `[{id:"bitcoin",symbol:"btc",name:"Bitcoin"}]` | Entity "Bitcoin" type "token" created; aliases: "bitcoin", "btc" |
| 10 | Symbol collision | Feed: `[{id:"solana",symbol:"sol",name:"Solana"},{id:"sol-protocol",symbol:"sol",name:"SOL Protocol"}]` | First "sol" alias wins; SOL Protocol gets entity but alias "sol" is not overwritten |
| 11 | Re-seed skips existing | Seed once, then re-seed with same data | No duplicate entities, no errors |

### Indonesian entity seeds

| # | Case | Action | Expected |
|---|------|--------|----------|
| 12 | OJK created | Run ID seed function | Entity "OJK" type "company", aliases: "ojk", "otoritas jasa keuangan" |
| 13 | Bank Indonesia + BI alias | Run ID seed function | Entity "Bank Indonesia", aliases include "bi" |
| 14 | Indodax, Tokocrypto, Pintu | Run ID seed function | All three created with correct aliases |

### BI disambiguation

| # | Case | Setup | Action | Expected |
|---|------|-------|--------|----------|
| 15 | BI defaults to Bank Indonesia | ID seeds applied | Lookup alias "bi" | Returns Bank Indonesia entity_id |
| 16 | Binance does not steal BI | ID seeds applied | Resolve `{ name: "Binance", aliases: ["BI", "BNB"] }` | "binance" and "bnb" aliases created; "bi" ON CONFLICT DO NOTHING is no-op; "bi" still maps to Bank Indonesia |
| 17 | Binance resolves via own alias | After test 16 | Lookup alias "binance" | Returns Binance entity_id (separate from Bank Indonesia) |

---

## 5. API Endpoint Tests

All in `test/unit/server.test.ts`. Uses `fastify.inject()` for in-process requests. Seeded in-memory DB. Valid API key set in `app_config` (argon2 hash).

For every endpoint: success case, auth failure (missing/wrong Bearer token → 401), and validation failure where applicable.

### Reports

| # | Endpoint | Case | Request | Expected |
|---|----------|------|---------|----------|
| 1 | `GET /api/v1/reports` | Success | `?limit=10&offset=0&type=daily` | 200, `{ reports: [...] }`, paginated |
| 2 | `GET /api/v1/reports` | Auth fail | No Authorization header | 401 |
| 3 | `GET /api/v1/reports` | Bad type param | `?type=invalid` | 400, validation error |
| 4 | `GET /api/v1/reports/latest` | Success, exists | DB has reports | 200, `{ report: MarketReport }` |
| 5 | `GET /api/v1/reports/latest` | Success, empty | Empty DB | 200, `{ report: null }` |
| 6 | `GET /api/v1/reports/latest` | Auth fail | Wrong Bearer token | 401 |
| 7 | `GET /api/v1/reports/:id` | Success | Valid report ID | 200, `{ report: MarketReport }` |
| 8 | `GET /api/v1/reports/:id` | Not found | Nonexistent ID | 404 |
| 9 | `GET /api/v1/reports/:id` | Auth fail | Missing header | 401 |

### Sources

| # | Endpoint | Case | Request | Expected |
|---|----------|------|---------|----------|
| 10 | `GET /api/v1/sources` | Success | Valid auth | 200, `{ sources: [...] }` with joined source_state |
| 11 | `GET /api/v1/sources` | Auth fail | Bad token | 401 |
| 12 | `POST /api/v1/sources` | Success — Discord | `{ source: "discord", sourceId: "123456789012345678", label: "alpha" }` | 201 |
| 13 | `POST /api/v1/sources` | Success — Twitter | `{ source: "twitter", sourceId: "@elonmusk", label: "Elon" }` | 201 |
| 14 | `POST /api/v1/sources` | Bad source enum | `{ source: "telegram", sourceId: "x" }` | 400 |
| 15 | `POST /api/v1/sources` | Bad Discord ID | `{ source: "discord", sourceId: "not-a-snowflake" }` | 400 |
| 16 | `POST /api/v1/sources` | Bad Twitter handle | `{ source: "twitter", sourceId: "no-at-sign" }` | 400 |
| 17 | `POST /api/v1/sources` | Auth fail | Missing header | 401 |
| 18 | `PATCH /api/v1/sources/:source/:sourceId` | Success | `{ enabled: false }` | 200 |
| 19 | `PATCH /api/v1/sources/:source/:sourceId` | Not found | Nonexistent source | 404 |
| 20 | `PATCH /api/v1/sources/:source/:sourceId` | Auth fail | Bad token | 401 |
| 21 | `DELETE /api/v1/sources/:source/:sourceId` | Success | Valid source | 200, source removed, items kept |
| 22 | `DELETE /api/v1/sources/:source/:sourceId` | Not found | Nonexistent | 404 |
| 23 | `DELETE /api/v1/sources/:source/:sourceId` | Auth fail | Missing | 401 |
| 24 | `POST /api/v1/sources/:s/:sid/test` | Success | Connectable source | 200, `{ ok: true }` |
| 25 | `POST /api/v1/sources/:s/:sid/test` | Fail | Unreachable source | 200, `{ ok: false, error: "..." }` |
| 26 | `POST /api/v1/sources/:s/:sid/test` | Auth fail | Bad token | 401 |
| 27 | `GET /api/v1/sources/discover/discord` | Success | Valid Discord tokens | 200, `{ guilds: [...] }` |
| 28 | `GET /api/v1/sources/discover/discord` | Auth fail | Missing | 401 |

### Config and health

| # | Endpoint | Case | Request | Expected |
|---|----------|------|---------|----------|
| 29 | `GET /api/v1/config` | Success | Valid auth | 200, `{ webhookUrl, digestTime, timezone, publicUrl }` |
| 30 | `GET /api/v1/config` | Auth fail | Bad token | 401 |
| 31 | `PATCH /api/v1/config` | Success | `{ digestTime: "10:00" }` | 200, cron rescheduled |
| 32 | `PATCH /api/v1/config` | Bad webhook URL | `{ webhookUrl: "http://evil.com/hook" }` | 400, must be Discord webhook URL |
| 33 | `PATCH /api/v1/config` | Bad timezone | `{ timezone: "Mars/Olympus" }` | 400 |
| 34 | `PATCH /api/v1/config` | Auth fail | Missing | 401 |
| 35 | `GET /api/v1/status` | Success | Valid auth | 200, `{ stages, llmCostToday, allSourcesDisabled }` |
| 36 | `GET /api/v1/status` | Auth fail | Bad token | 401 |
| 37 | `POST /api/v1/auth/rotate` | Success | Valid auth | 200, `{ apiKey: "new-key..." }`, old key invalidated |
| 38 | `POST /api/v1/auth/rotate` | Auth fail | Bad token | 401 |

### Search

| # | Endpoint | Case | Request | Expected |
|---|----------|------|---------|----------|
| 39 | `GET /api/v1/search` | Success | `?q=solana&days=7` | 200, `{ items: [...] }` via tsvector/tsquery |
| 40 | `GET /api/v1/search` | Missing query | `?days=7` (no q) | 400 |
| 41 | `GET /api/v1/search` | Auth fail | Bad token | 401 |

### Auth rate limiting

| # | Case | Action | Expected |
|---|------|--------|----------|
| 42 | Rate limit on failed auth | Send 6 requests with wrong key from same IP within 1 minute | 6th request returns 429 |

---

## 6. Integration Test Script

File: `test/integration/pipeline.test.ts`. Skipped in CI (`describe.skip` or env guard). Requires `.env` with a real `ANTHROPIC_API_KEY`.

### Pseudocode

```
describe('full pipeline integration', () => {

  beforeAll(() => {
    // 1. Create fresh test DB (isolated Postgres schema)
    pool = createTestPool()
    await runMigrations(pool)

    // 2. Seed sources
    await pool.query(`
      INSERT INTO sources (source, source_id, label, enabled, priority, added_at)
      VALUES ('discord', '1111111111111111', 'test-channel', true, 1, ${now})
    `)
    await pool.query(`
      INSERT INTO sources (source, source_id, label, enabled, priority, added_at)
      VALUES ('twitter', '@testhandle', 'test-twitter', true, 1, ${now})
    `)

    // 3. Seed CoinGecko entities (mock, not live)
    seedEntities(pool, [
      { id: 'ethereum', symbol: 'eth', name: 'Ethereum' },
      { id: 'solana', symbol: 'sol', name: 'Solana' },
      { id: 'bitcoin', symbol: 'btc', name: 'Bitcoin' },
    ])
    seedIndonesianEntities(pool)

    // 4. Insert 50 raw items spanning both sources
    //    - 20 Discord messages (mix of EN + ID, some spam, some duplicates)
    //    - 15 Twitter tweets (EN, some with shared URLs)
    //    - 10 RSS entries (EN, one oversized at 30K chars)
    //    - 5 news articles (EN)
    for (item of testItems) {
      insertRawItem(pool, item)
    }

    // 5. Configure webhook (mock server)
    webhookServer = createMockWebhookServer(port=19876)
    await pool.query(`
      INSERT INTO app_config (key, value)
      VALUES ('webhook_url', 'http://localhost:19876/webhook')
    `)
    await pool.query(`
      INSERT INTO app_config (key, value)
      VALUES ('digest_time', '09:00')
    `)
    await pool.query(`
      INSERT INTO app_config (key, value)
      VALUES ('timezone', 'Asia/Jakarta')
    `)
  })

  test('normalize filters and deduplicates', async () => {
    // Run normalize on all 50 items
    for (item of await getAllReadyItems(pool)) {
      await normalize(item)
    }

    const { rows: [{ count: readyCount }] } = await pool.query("SELECT COUNT(*) FROM items WHERE status='ready'")
    const { rows: [{ count: filteredCount }] } = await pool.query("SELECT COUNT(*) FROM items WHERE status='filtered'")

    // Expect ~25-30 ready (spam, dupes, wrong language removed)
    assert(readyCount >= 20 && readyCount <= 35)
    assert(filteredCount >= 10)

    // Verify the oversized item was truncated
    const { rows: [oversized] } = await pool.query("SELECT content FROM items WHERE id = $1", [oversizedItemId])
    assert(oversized.content.length <= 20000)
  })

  test('stage 1 summarize produces summaries and entities', async () => {
    await summarize.runBatch()

    const { rows: summaries } = await pool.query("SELECT * FROM summaries")
    assert(summaries.length >= 2)  // at least one per active source

    for (s of summaries) {
      body = JSON.parse(s.body)
      assert(ChunkSummarySchema.safeParse(body).success)
      assert(body.entities.length >= 0)
      assert(['routine', 'elevated', 'breaking'].includes(body.urgency))
      assert(typeof body.sentiment === 'number')
      assert(body.sentiment >= -1 && body.sentiment <= 1)
    }

    // Entities were created
    const { rows: entities } = await pool.query("SELECT * FROM entities")
    assert(entities.length >= 3)

    // All items now processed
    const { rows: [{ count: processing }] } = await pool.query("SELECT COUNT(*) FROM items WHERE status='processing'")
    assert(processing === 0)
  })

  test('stage 2 correlate finds cross-source entities', async () => {
    correlated = await correlate.run()

    // At least one entity should appear in 2+ sources
    // (test data designed so "Ethereum" appears in both Discord and Twitter)
    ethCorrelated = correlated.find(e => e.name === 'Ethereum')
    assert(ethCorrelated !== undefined)
    assert(ethCorrelated.source_count >= 2)
  })

  test('stage 3 synthesize produces a valid report', async () => {
    await synthesize.runDaily()

    const { rows: [report] } = await pool.query("SELECT * FROM reports WHERE type='daily' LIMIT 1")
    assert(report !== null)

    body = JSON.parse(report.body)
    assert(typeof body.tldr === 'string' && body.tldr.length > 20)
    assert(Array.isArray(body.keyEvents) && body.keyEvents.length >= 1)
    assert(typeof body.sentiment === 'number')
    assert(body.date === todayDateString)

    // Delivery status set
    assert(['pending', 'delivered'].includes(report.delivery_status))
  })

  test('webhook delivery sends a valid Discord embed', async () => {
    // Wait briefly for async delivery
    await waitFor(() => webhookServer.requests.length > 0, 5000)

    req = webhookServer.requests[0]
    assert(req.method === 'POST')
    payload = JSON.parse(req.body)
    assert(Array.isArray(payload.embeds))
    assert(payload.embeds[0].title.includes('Daily Market Report'))
    assert(typeof payload.embeds[0].description === 'string')

    // Report marked as delivered
    const { rows: [report] } = await pool.query("SELECT delivery_status FROM reports LIMIT 1")
    assert(report.delivery_status === 'delivered')
  })

  afterAll(async () => {
    webhookServer.close()
    await pool.end()
  })
})
```

### Running

```bash
# Unit tests (CI-safe, no API keys needed)
npm test

# Integration (requires .env with ANTHROPIC_API_KEY)
npm run test:integration
```

The integration test takes 30-60 seconds due to real LLM calls. It is designed to fail loudly if any pipeline stage produces invalid output, ensuring the full chain from raw item to delivered webhook embed works end to end.

---

## 7. Prompt Regression Tests

Separate from golden file tests (which validate parsing of frozen LLM output). Prompt regression tests send real inputs to the LLM and assert on output **properties**, not exact text. They catch regressions when prompts are edited.

### Structure

- Fixtures in `test/prompts/fixtures/` — 8 frozen real inputs, each a JSON file with raw scraped content.
- Golden files assert on **properties**: entity count range, urgency classification, presence of specific entity names, sentiment sign, fact anchors. Never exact text comparison.
- Each fixture runs 3 times to account for non-determinism. A property must hold in all 3 runs to pass.
- `npm run test:prompts` — manual, not CI. Costs ~$0.25 per run.

### Fixtures

| Fixture | Purpose |
|---------|---------|
| `discord-en-routine-15.json` | Baseline: Discord EN, routine, 15 messages |
| `discord-id-routine-slang.json` | Indonesian handling: Discord ID, routine, mixed slang |
| `discord-en-breaking-exploit.json` | Urgency = breaking: Discord EN, breaking exploit |
| `twitter-en-quiet-20.json` | Minimal output: Twitter EN, 20 tweets, quiet day |
| `rss-regulatory-ojk.json` | Pre-summarize regulatory: OJK ruling, 6000 chars |
| `rss-general-news.json` | Pre-summarize general: news article, 8000 chars |
| `mixed-spam-heavy.json` | Signal extraction: mixed chunk, 60% noise |
| `stage3-input-12-summaries.json` | Daily synthesis: 12 summaries + correlations |

### Golden File Approach

Each fixture has a companion `.expected.json` defining property assertions:

```json
{
  "entities": {
    "minCount": 2,
    "maxCount": 8,
    "mustInclude": ["Ethereum", "Solana"],
    "mustExcludeTypes": []
  },
  "urgency": "routine",
  "sentiment": { "sign": "positive" },
  "factAnchors": [
    "staking",
    "3.2%"
  ]
}
```

The test runner checks:
- **Entity count** falls within `[minCount, maxCount]`
- **Required entities** appear in output (by canonical name or alias)
- **Urgency** matches labeled value exactly
- **Sentiment sign** matches (positive/negative/neutral band)
- **Fact anchors** — specific strings that must appear somewhere in the summary or keyEvents

### Regression Metrics

| Metric | Definition | Threshold |
|--------|-----------|-----------|
| Entity recall | found / expected entities (3+ mentions in fixture) | Must stay >80% |
| Urgency accuracy | output urgency matches labeled fixture | Must match exactly |
| Key fact preservation | pre-summarize output contains defined anchor facts | All anchors present |

A prompt change is a **regression** if entity recall drops below 80% or urgency misclassifies on any fixture.

### CI Plan

| Test type | When | LLM calls |
|-----------|------|-----------|
| Structural tests (zod parsing, chunking, decision tree) | Every push | None |
| Prompt regression tests (`npm run test:prompts`) | Manual, before merging prompt changes | Yes (~$0.25/run) |

Prompt tests are never automated in CI — they cost money and are non-deterministic. Run them locally before any PR that touches system prompts or few-shot examples.
