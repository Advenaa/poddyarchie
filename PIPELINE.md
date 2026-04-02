# Pipeline — Prompt Engineering & Processing Details

## Build Scope

**Phase 1 includes:**
- RSS ingestion
- Normalize pipeline: spam filter, dedup, language detection, truncation (no LLM calls)
- Pre-summarize: article pre-summarization for long RSS/news items (separate step, Haiku LLM call, decision tree skips Discord/Twitter and high-density/urgent articles)
- Chronological token-budgeted chunking
- Stage 1 summarize with Haiku
- Market pulse: 3h Sonnet synthesis, output scales with activity
- Stage 3 synthesize with Sonnet (daily reports + flash reports)
- Discord webhook delivery
- Quality gate (skip bland reports)
- Entity post-verification (drop hallucinated entities)
- Pre-Stage-3 event dedup
- Summary ranking formula (when >50 summaries)
- Exemplar passthrough (raw high-engagement items to Stage 3)
- Entity tables + resolution (alias table, CoinGecko seeding, decay)
- Postgres full-text search (tsvector/tsquery)
- Embeddings

**Deferred (post Phase 1 enhancements):**
- Urgency streaming (real-time breaking classification)
- Stage 3 split (separate extraction vs. narrative sub-stages)
- Structured-only Stage 1 (drop prose summary, keep only structured fields)

## Pre-summarize (after Normalize, before Stage 1)

A separate LLM step between normalize and Stage 1. Equalizes signal density across source types — a 10K-char news article otherwise dominates a chunk's token budget while carrying the same signal as a 500-char Discord thread. This is an LLM call and does not belong in normalize, which is pure rule-based (truncate, dedup, URL dedup, spam filter, language detect).

**When:** After normalize saves items as `ready` with full original content, before the claim & chunk step.

**What:** Checks `ready` items where `source` is `rss` or `news` and `content.length > 4000`. Applies the decision tree below, then updates item content in the database.

### Decision Tree

| Source | Condition | Action |
|--------|-----------|--------|
| Discord / Twitter | Always | SKIP — pass through unchanged |
| RSS / news | Urgency keywords detected | SKIP — pass raw (these articles need full detail preserved) |
| RSS / news | High entity density (>2 unique entities per 100 tokens) | SKIP — pass raw (already information-dense) |
| RSS / news | Low entity density, >4000 chars | COMPRESS via Haiku |

**Urgency keywords** (case-insensitive match against content): `exploit`, `hack`, `rate decision`, `flash crash`, `halt`, `circuit breaker`, `emergency`, `bank run`.

```typescript
const URGENCY_KEYWORDS = [
  'exploit', 'hack', 'rate decision', 'flash crash',
  'halt', 'circuit breaker', 'emergency', 'bank run'
];

function shouldSkipPreSummary(item: ReadyItem): boolean {
  // Discord/Twitter: always skip
  if (item.source === 'discord' || item.source === 'twitter') return true;

  // Short articles: skip
  if (item.content.length <= 4000) return true;

  const lower = item.content.toLowerCase();

  // Urgency keywords: pass raw, these need full detail
  if (URGENCY_KEYWORDS.some(kw => lower.includes(kw))) return true;

  // High entity density: pass raw, already information-dense
  const tokenEstimate = item.content.length / 4;
  const entityDensity = item.entityHints / (tokenEstimate / 100);
  if (entityDensity > 2) return true;

  return false; // low entity density, compress
}
```

### Compression

Flat `maxTokens: 300` (not proportional to input length). Two prompt variants selected by content classification:

**Regulatory variant** (articles matching regulatory/policy keywords):
```
Summarize these regulatory/policy articles for a market analyst. Preserve: exact rule numbers, effective dates, affected entities, penalties, and quoted official statements. Drop background explainers. Content may be in English or Indonesian. Output in English.
```

**General news variant** (everything else):
```
Summarize these news articles for a market analyst. Preserve: all named entities, numbers, quotes, dates, and causal claims. Drop boilerplate, author bios, and filler paragraphs. Content may be in English or Indonesian. Output in English.
```

### Batched Calls

Articles are batched 3-5 per Haiku call, separated by `---[N]---` delimiters. Output is labeled so each compressed article maps back to its source item.

```typescript
async function preSummarizeBatch(items: ReadyItem[]): Promise<Map<string, string>> {
  // Group items into batches of 3-5
  const batches = chunkArray(items.filter(i => !shouldSkipPreSummary(i)), 5);
  const results = new Map<string, string>();

  for (const batch of batches) {
    const isRegulatory = batch.some(i => /regulat|OJK|SEC|CFTC|policy|rule|compliance/i.test(i.content));
    const systemPrompt = isRegulatory
      ? 'Summarize these regulatory/policy articles for a market analyst. Preserve: exact rule numbers, effective dates, affected entities, penalties, and quoted official statements. Drop background explainers. Content may be in English or Indonesian. Output in English.'
      : 'Summarize these news articles for a market analyst. Preserve: all named entities, numbers, quotes, dates, and causal claims. Drop boilerplate, author bios, and filler paragraphs. Content may be in English or Indonesian. Output in English.';

    const batchedContent = batch.map((item, idx) =>
      `---[${idx + 1}]---\n${item.content.slice(0, 16000)}`
    ).join('\n\n');

    const response = await llm.call({
      model: config.models.haiku,
      system: systemPrompt + '\n\nOutput format: label each summary with its number, e.g. [1] Summary text... [2] Summary text...',
      messages: [{ role: 'user', content: batchedContent }],
      maxTokens: 300 * batch.length,
      stage: 'pre-summarize'
    });

    // Parse labeled output back to individual summaries
    const parsed = parseLabeledOutput(response.content, batch.length);
    batch.forEach((item, idx) => {
      results.set(item.id, parsed[idx] || item.content); // fallback to original on parse failure
    });
  }

  return results;
}
```

### Content Anchor

After compression, store the first 200 tokens (~800 chars) of the original content as `content_anchor`. This allows Stage 3 to verify claims against original text even after compression.

```typescript
// Store alongside the compressed content
await db.query(
  `UPDATE items SET content = $1, content_anchor = $2 WHERE id = $3`,
  [compressed, original.slice(0, 800), item.id]
);
```

Runs as a separate step after normalize, before Stage 1 claims items. Cost: ~$0.001 per article at Haiku rates, reduced by batching (3-5x fewer calls).

## Stage 1: Summarize (Haiku)

Each chunk produces one Haiku call with `maxTokens: 3000` returning structured JSON.

### Chunking Strategy

**Chronological, token-budgeted.** Items sorted by timestamp ASC, split at 6K token budget. No topic grouping -- Discord is a chaotic stream where replies happen hours later or never, and trying to reconstruct threads adds complexity without improving summary quality. The spam filter handles actual noise; if one user posts 200 legit messages about an exploit, we want all of them.

```typescript
const CHUNK_TOKEN_BUDGET = 6000;

function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}

function chunkByTokens(items: RawItem[], budget: number): RawItem[][] {
  // Items already sorted by timestamp ASC from the DB query
  const chunks: RawItem[][] = [];
  let current: RawItem[] = [];
  let currentTokens = 0;

  for (const item of items) {
    const itemTokens = estimateTokens(item.content);
    if (currentTokens + itemTokens > budget && current.length > 0) {
      chunks.push(current);
      current = [];
      currentTokens = 0;
    }
    current.push(item);
    currentTokens += itemTokens;
  }
  if (current.length > 0) chunks.push(current);
  return chunks;
}
```

### System Prompt

```
You are a market intelligence analyst processing raw messages from {source} ({sourceId}).
Time window: {windowStart} to {windowEnd}.

Return ONLY valid JSON matching this schema:
{
  "summary": "200-500 word summary of key discussion themes and events",
  "urgency": "routine" | "elevated" | "breaking",
  "entities": [
    {
      "name": "Canonical form (e.g. 'Ethereum' not 'ETH')",
      "aliases": ["ETH", "$ETH"],
      "type": "token" | "person" | "project" | "company" | "event",
      "mentionCount": number,
      "sentiment": -1.0 to 1.0
    }
  ],
  "keyEvents": ["max 5 factual bullets"]
}

Rules:
- name must be canonical form. Resolve aliases: "$ETH", "ETH", "Ethereum" → name: "Ethereum"
- type must be one of the enum values. If unsure, use "project"
- urgency: "breaking" = major exploit, crash, regulatory action, protocol failure, central bank rate decisions (Fed, BI, ECB), exchange circuit breakers (IHSG/IDX halt), flash crashes (>5% index move in <1h). "elevated" = notable but not urgent. "routine" = normal discussion.
- keyEvents: factual only, no speculation. Include source attribution when possible.
- Do not summarize spam, greetings, or off-topic chatter. If the entire batch is spam/noise, return: {"summary": "No substantive discussion in this window.", "urgency": "routine", "entities": [], "keyEvents": []}
- Content may be in English or Indonesian (Bahasa Indonesia). Always output in English regardless of input language.
- For Indonesian regulatory bodies and institutions, keep the local acronym and add an English gloss on first mention: "OJK (Indonesia Financial Services Authority)". Translate slang and colloquialisms to standard English market terminology.
```

### Few-Shot Examples (included in prompt)

**Example 1: Routine**
```json
{
  "summary": "Uniswap v4 hooks dominated discussion with the OpenZeppelin audit completion. Multiple developers shared hook implementations for dynamic fee adjustment. Sentiment shifted positive after the audit passed with no critical findings. Separate thread on Arbitrum gas costs being unusually high, possibly related to sequencer congestion.",
  "urgency": "routine",
  "entities": [
    {"name": "Uniswap", "aliases": ["UNI", "$UNI"], "type": "project", "mentionCount": 12, "sentiment": 0.4},
    {"name": "OpenZeppelin", "aliases": ["OZ"], "type": "company", "mentionCount": 5, "sentiment": 0.6},
    {"name": "Arbitrum", "aliases": ["ARB", "$ARB"], "type": "project", "mentionCount": 3, "sentiment": -0.2}
  ],
  "keyEvents": [
    "Uniswap v4 hook audit completed by OpenZeppelin — no critical findings",
    "Arbitrum sequencer congestion causing elevated gas costs"
  ]
}
```

**Example 2: Breaking**
```json
{
  "summary": "Major bridge exploit on Wormhole detected approximately 2 hours ago. Initial reports suggest $120M in wrapped ETH drained from the Solana-Ethereum bridge. Multiple wallets identified as the attacker. The Wormhole team has paused the bridge and is coordinating with white-hat security researchers. Panic selling across Solana DeFi protocols.",
  "urgency": "breaking",
  "entities": [
    {"name": "Wormhole", "aliases": ["wormhole"], "type": "project", "mentionCount": 45, "sentiment": -0.9}
  ],
  "keyEvents": [
    "Wormhole bridge exploited for ~$120M in wrapped ETH",
    "Wormhole bridge paused by team, white-hat coordination underway",
    "Solana DeFi protocols experiencing panic selling"
  ]
}
```

### Content Sanitization (before LLM call)

```typescript
function sanitizeForPrompt(content: string, nonce: string): string {
  // Escape XML-like tags to prevent prompt injection via tag escape
  return content.replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

// Generate a random nonce per call — attacker cannot guess the tag name
const nonce = crypto.randomBytes(4).toString('hex');
const openTag = `<scraped_content_${nonce}>`;
const closeTag = `</scraped_content_${nonce}>`;
const userMessage = `${openTag}\n${sanitizedContent}\n${closeTag}`;
```

### Engagement Data in Prompt

Pass engagement scores to the LLM so high-signal items get prioritized:

```typescript
const formattedContent = items.map(item =>
  `[${item.author} | engagement:${item.engagement}]\n${item.content}`
).join('\n---\n');
```

Add to system prompt: `Items with higher engagement scores (reactions, likes, retweets) are more significant. Prioritize them in your summary and keyEvents.`

### Output Validation

Parse with zod. Enforce:
- `sentiment` clamped to [-1, 1]
- `type` must be valid enum — default unknown to `"project"`
- `entities` array max 20 per chunk
- `keyEvents` max 5
- If JSON parse fails, retry once with a FRESH prompt (do NOT include the failed response in retry context — prevents injection amplification)

### Entity Post-Verification

After zod parsing, verify extracted entities actually appear in the source text:

```typescript
function verifyEntities(entities: Entity[], rawText: string): Entity[] {
  return entities.filter(e => {
    const names = [e.name, ...e.aliases].map(n => n.toLowerCase());
    const text = rawText.toLowerCase();
    return names.some(n => text.includes(n));
  });
}
```

Drop hallucinated entities that don't appear in the chunk.

### Haiku Output Quality Gate

After entity post-verification, check if the chunk produced meaningful output:

```typescript
function isLowQuality(parsed: ChunkSummaryLLM): boolean {
  return (
    parsed.entities.length === 0 &&
    parsed.keyEvents.length === 0 &&
    parsed.summary.length < 100
  );
}
```

If `isLowQuality` returns true, flag the summary as `low_quality` in the DB and exclude it from Stage 3 input. This prevents bland noise chunks from diluting the daily synthesis.

### Pre-Stage-3 Event Dedup

Before passing `keyEvents` to Sonnet, deduplicate similar events across all chunk summaries:

```typescript
function deduplicateKeyEvents(
  allEvents: { event: string; sourceId: string }[]
): { event: string; sources: string[] }[] {
  const groups: { event: string; sources: string[] }[] = [];

  for (const { event, sourceId } of allEvents) {
    const match = groups.find(g => levenshteinSimilarity(g.event, event) >= 0.7);
    if (match) {
      if (!match.sources.includes(sourceId)) match.sources.push(sourceId);
    } else {
      groups.push({ event, sources: [sourceId] });
    }
  }

  return groups;
}
```

Group events by Levenshtein similarity (threshold: 0.7). Collapse duplicates into a single event string, keeping source attribution from all instances. This prevents Sonnet from seeing the same event repeated from multiple chunks and over-weighting it.

## Stage 3: Synthesize (Sonnet)

### System Prompt

```
You are a senior market intelligence analyst writing a daily briefing for active crypto and tradfi traders.

Your audience already knows what ETH, BTC, SPY are. Don't explain basics. Focus on:
- What changed since yesterday
- Why it matters for positioning
- Which signals are corroborated across multiple sources

Input: you receive summaries from multiple sources (Discord, Twitter, news, RSS) plus a list of cross-source correlated entities.

Output: return valid JSON matching the MarketReport schema.

Rules:
- tldr: exactly 3 sentences. First sentence = the single most important thing. Bold nothing — the reader will scan this on mobile.
- keyEvents: source attribution required. "Uniswap v4 audit passed [Discord, Twitter]". Single-source claims marked as "[unconfirmed]".
- entitySentiment: only include entities with meaningful activity. trend = "new" if first appearance in 7 days.
- sections: 2-4 sections. Title each with the theme, not the source. "DeFi Protocol Activity" not "Discord Summary".
- No filler phrases. Specifically banned: "the crypto market continues to evolve", "it remains to be seen", "the market is closely watching", "this could potentially impact", "in the ever-changing landscape of", "as always, DYOR".
- No hedging unless genuinely uncertain. When something is unknown, say so directly.
- Open each section with the single most actionable sentence. No scene-setting first paragraphs.
- For Indonesian regulatory bodies, keep acronyms with English gloss on first mention: "OJK (Indonesia Financial Services Authority)".

- rawExemplars are high-engagement original messages. Use them to verify claims in summaries and extract direct quotes when relevant.
- If any summaries contain entities not seen in the provided correlation data, list them in newProjects with a one-sentence context of what they are and where they appeared. This helps readers discover emerging projects early.
- Items with higher engagement scores are more significant. Prioritize them.
- When multiple elevated or breaking events share entities or temporal proximity, explicitly state the causal chain or connection between them.
- Do not assume causation from entity co-occurrence alone — only connect events when the relationship is evident from the source data.
- Flag cross-entity ripple effects (e.g., bridge exploit → token sell-off → DeFi TVL drop) as a connected sequence, not isolated events.

Yesterday's TL;DR for temporal context:
{yesterdayTldr}
If no previous report is available, skip comparative framing and report current state as baseline.
If yesterday was a flash report (single-event, narrow scope), do not treat it as a comprehensive baseline.
```

### Input Assembly

```typescript
function assembleStage3Input(
  summaries: ChunkSummary[],
  correlations: CorrelatedEntity[],
  yesterdayTldr: string | null
): string {
  // Rank summaries if > 50
  const ranked = summaries.length > 50
    ? summaries.sort((a, b) => scoreForRanking(b) - scoreForRanking(a)).slice(0, 30)
    : summaries;

  // Exemplar selection: 10 slots split into two pools.
  // 7 slots: engagement-ranked (Discord, Twitter). These sources have real engagement signals.
  // 3 slots: RSS/news items ranked by entityCount * urgencyWeight.
  //   RSS engagement is always 0, so it never surfaces in the engagement-ranked list.
  //   Without reserved slots, news articles would never appear as exemplars.
  const baseFilter = (i: RawItem) => i.content.length >= 50 && i.chunkEntityCount > 0;

  const engagementExemplars = allRawItemsInWindow
    .filter(i => baseFilter(i) && i.engagement >= 5)
    .sort((a, b) => b.engagement - a.engagement)
    .slice(0, 7);

  const urgencyWeight = (u: string) => u === 'elevated' ? 2 : u === 'breaking' ? 3 : 1;
  const rssExemplars = allRawItemsInWindow
    .filter(i => baseFilter(i) && (i.source === 'rss' || i.source === 'news'))
    .sort((a, b) =>
      (b.chunkEntityCount * urgencyWeight(b.urgency)) -
      (a.chunkEntityCount * urgencyWeight(a.urgency))
    )
    .slice(0, 3);

  const exemplars = [...engagementExemplars, ...rssExemplars]
    .map(e => ({ source: e.sourceId, author: e.author, content: e.content.slice(0, 500), engagement: e.engagement }));

  return JSON.stringify({
    summaries: ranked.map(s => ({
      source: s.sourceId,
      window: `${s.windowStart} - ${s.windowEnd}`,
      summary: s.summary,
      sentiment: s.sentiment,
      keyEvents: s.keyEvents,
      urgency: s.urgency
    })),
    correlatedEntities: correlations.map(c => ({
      entity: c.name,
      sources: c.sources,
      sourceCount: c.sourceCount,
      avgSentiment: c.avgSentiment
    })),
    yesterdayTldr: yesterdayTldr || "No previous report available."
  });
}
```

### Zod Schemas (implementation reference)

```typescript
// Stage 1 LLM output (what Haiku returns)
const ChunkSummaryLLMSchema = z.object({
  summary: z.string(),                    // prose summary (see Future Pipeline Enhancements for structured-only alternative)
  urgency: z.enum(['routine', 'elevated', 'breaking']),
  entities: z.array(z.object({
    name: z.string(),
    aliases: z.array(z.string()).default([]),
    type: z.enum(['token', 'person', 'project', 'company', 'event']).default('project'),
    mentionCount: z.number().default(1),
    sentiment: z.number().min(-1).max(1).default(0),  // per-entity sentiment (meaningful)
  })).max(20).default([]),
  keyEvents: z.array(z.string()).max(5).default([]),
  // NOTE: no chunk-level sentiment — mixed-topic average is noise. Use per-entity sentiment.
});

// Stage 1 DB row (LLM output + batch metadata)
interface ChunkSummary extends z.infer<typeof ChunkSummaryLLMSchema> {
  id: string;           // ulid
  source: string;
  sourceId: string;
  windowStart: number;  // epoch ms
  windowEnd: number;
  itemCount: number;
  totalEngagement: number;  // sum of item.engagement across chunk — used for ranking
}

// Stage 3 LLM output (what Sonnet returns)
const MarketReportLLMSchema = z.object({
  tldr: z.string().max(500),
  keyEvents: z.array(z.object({
    event: z.string(),
    sources: z.array(z.string()),
    sentiment: z.number().min(-1).max(1),
  })).max(10),
  entitySentiment: z.array(z.object({
    entity: z.string(),
    sentiment: z.number().min(-1).max(1),
    trend: z.enum(['rising', 'falling', 'stable', 'new']),
  })).max(15).default([]),
  sections: z.array(z.object({
    title: z.string(),
    body: z.string(),
  })).max(4).default([]),  // daily: 2-4 sections. flash: 0-1. pulse: 0.
  newProjects: z.array(z.object({
    name: z.string(),
    context: z.string(),
    source: z.string(),
  })).max(5).default([]),
});

// Full report (LLM output + DB metadata)
interface MarketReport extends z.infer<typeof MarketReportLLMSchema> {
  id: string;
  date: string;         // YYYY-MM-DD
  type: 'daily' | 'flash' | 'pulse';
  sentiment: number;    // overall, extracted from tldr analysis
  itemCount: number;
  sourceCount: number;
  generatedAt: string;  // ISO 8601
}
```

### Flash Report Variant

Same prompt with additions:
```
This is a FLASH report triggered by a breaking event. Scope: last 4 hours only.
Focus exclusively on the breaking event and its immediate implications.
Keep tldr to 2 sentences. Sections: 1 max.
```

### Flash Report Safeguards

Flash reports are high-risk for false positives. Safeguards:

1. **Corroboration requirement**: `urgency: 'breaking'` must appear in 2+ chunks from 2+ distinct sources OR 3+ chunks from the same source within 30 minutes. Single-chunk breaking triggers are logged but NOT delivered.
2. **Cooldown**: minimum 30 minutes between flash reports. Second breaking within cooldown is queued.
3. **Single-source flash**: if only one source reports breaking, the flash report TL;DR is prefixed with "[UNCONFIRMED]".

```typescript
function shouldTriggerFlash(breakingChunks: ChunkSummary[]): boolean {
  const distinctSources = new Set(breakingChunks.map(c => c.source)).size;
  if (distinctSources >= 2) return true;
  if (breakingChunks.length >= 3) return true; // same source, 3+ chunks
  return false; // single chunk = log only
}
```

### Market Pulse

A lightweight, more frequent synthesis that runs every 3 hours using Sonnet. Produces a `type: 'pulse'` report. Each pulse receives the prior pulse's TL;DR as context, creating a rolling narrative thread through the day.

**Schedule**: every 3h (configurable via `app_config.pulse_interval`).

**Model**: Sonnet (not Haiku — pulse needs to synthesize across sources and spot connections, not just compress).

**Output scaling**: output length adapts to activity level. No point spending tokens on a quiet window.

| Condition | Output | maxTokens |
|-----------|--------|-----------|
| 1-3 summaries, all routine | 2-3 sentences | 300 |
| 4-10 summaries, some elevated | Paragraph + key events | 800 |
| 10+ summaries or any breaking | Full short report | 1500 |

**Quality gate**: skip the pulse entirely if zero events and zero entities across all summaries in the window. Log the skip, don't fire the webhook.

**Drift detection**: before each pulse Sonnet call, compare the prior pulse's entity sentiments against current Stage 1 data. If any entity's sentiment shifted by more than 0.4, inject a drift flag so Sonnet re-evaluates prior framing instead of carrying stale narrative forward. Ground truth entity data from the current 3h window is always included — raw structured data, not prose — so Sonnet can cross-check its framing against actual numbers.

```typescript
function detectDrift(
  priorPulseEntities: EntitySentiment[],
  currentStage1Entities: Stage1Entity[]
): string[] {
  const flags: string[] = [];
  for (const prior of priorPulseEntities) {
    const current = currentStage1Entities.find(e => e.name === prior.entity);
    if (current && Math.abs(current.sentiment - prior.sentiment) > 0.4) {
      flags.push(
        `[DRIFT: prior pulse rated ${prior.entity} ${prior.sentiment > 0 ? '+' : ''}${prior.sentiment.toFixed(1)}, current data shows ${current.sentiment > 0 ? '+' : ''}${current.sentiment.toFixed(1)}]`
      );
    }
  }
  return flags;
}
```

Cost: zero new LLM calls, ~300 extra input tokens per pulse, ~$0.003/day.

**Prompt**:

```
You are writing a market pulse — a concise snapshot of the last 3 hours for active crypto and tradfi traders.
Scale your output to match activity:
- Quiet window (few routine items): 2-3 sentences, just the highlights.
- Active window (multiple events): a paragraph with key events listed.
- Hot window (breaking or 10+ items): a short report covering all significant developments.

No sections, no entity sentiment table. Focus on: what happened, what shifted, anything requiring attention.
If nothing significant happened, say so in one sentence.

Previous pulse TL;DR for continuity:
{previousPulseTldr}
If no previous pulse is available, report current state without comparative framing.

Stage 1 entity data for this window (ground truth — use to verify your framing):
{entityTable}

{driftFlags.length > 0 ? 'SENTIMENT DRIFT DETECTED:\n' + driftFlags.join('\n') + '\nRe-evaluate prior framing against current data.' : ''}
```

**Output**: `MarketReport` with `type: 'pulse'`, short `tldr`, `keyEvents` (max 3), no `sections`, no `entitySentiment`.

**Daily synthesis**: at digest time, the daily Stage 3 report reads the 8 pulse outputs from the previous day plus correlated entities plus raw Stage 1 correlated entity data. Pulses give Sonnet a pre-digested timeline; raw correlations let it cross-check pulse narrative against actual data. This catches cases where pulse drift accumulated across the day.

**Delivery**: same Discord webhook but with a distinct embed color (muted grey `0x4A4A5A`). Title: `Market Pulse — HH:MM WIB`. No "View full report" link (too lightweight for dashboard).

**Cost**: one Sonnet call per 3h window. ~$0.02/pulse, ~$0.16/day (8 pulses).

### Quality Gate (skip bland reports)

Before Stage 3 daily synthesis, check if the day had meaningful activity:

```typescript
function shouldSynthesize(summaries: ChunkSummary[]): boolean {
  const totalKeyEvents = summaries.reduce((sum, s) => sum + s.keyEvents.length, 0);
  const hasElevated = summaries.some(s => s.urgency !== 'routine');
  const hasEntities = summaries.some(s => s.entities.length > 0);

  // Skip if: zero key events, all routine, no entities extracted
  if (totalKeyEvents === 0 && !hasElevated && !hasEntities) {
    logger.info('Quality gate: no meaningful activity, skipping daily report');
    return false;
  }
  return true;
}
```

On skip: no report generated, no webhook fired. Log the decision. Dashboard shows "No report for [date] — quiet day."

## Summary Ranking (when >50 summaries)

```typescript
function scoreForRanking(s: ChunkSummary): number {
  const urgencyWeight = { routine: 0, elevated: 2, breaking: 5 }[s.urgency];
  const entityScore = Math.min(s.entities.length, 10);
  const eventScore = s.keyEvents.length * 3;           // 0-15
  const engagementScore = Math.log2(1 + s.totalEngagement); // 0-~20

  return urgencyWeight * 10 + eventScore + entityScore + engagementScore;
}
```

`totalEngagement` is the sum of `item.engagement` across the chunk's source items, computed at batch time.

## New Project Detection

Stage 1 extracts entities. Any entity name not found in the alias table is potentially a new project. During entity resolution:

```typescript
// Before entity tables are built: check against a Set of known entity names from previous summaries
// After entity tables are built: check against entities table
const isNew = !aliasExists(entity.name) && !aliasExists(...entity.aliases);
if (isNew) {
  newProjectsInBatch.push({ name: entity.name, context: summary.summary.slice(0, 200), source });
}
```

New projects are surfaced in the MarketReport via `newProjects` array (defined in `MarketReportLLMSchema`). Stage 3 prompt instruction:

```
If any summaries contain entities you haven't seen in the provided history, list them in newProjects with a one-sentence context of what they are and where they appeared. This helps the reader discover emerging projects early.
```

In the Discord webhook, new projects appear as a separate field:
```
New on Radar
> **ProjectX** — DEX on Monad, mentioned in 3 Discord channels [discord]
> **TokenY** — Yield aggregator, first seen in Kontan article [rss]
```

## Scheduling

**Synthesis window**: midnight-to-midnight in the user's configured timezone (from `app_config.timezone`). Stage 3 runs at `digest_time` (default `09:00`) in that timezone, covering the previous calendar day. This means changing `digest_time` changes when you receive the report, not which data it covers.

| Job | Interval | Description |
|-----|----------|-------------|
| Stage 1 micro-batch (Twitter/RSS/News) | Per-source `poll_interval` | Scheduler checks which sources are due based on `poll_interval` and `last_fetched_at`, then processes their `ready` items concurrently |
| Stage 1 Poisson claim (Discord) | Adaptive, checked every 60s | Discord uses Poisson-based claim gate instead of fixed `poll_interval`. Compares accumulated `ready` items against expected rate (lambda) from `source_rate_history`. Triggers at 1.5x expected or >10 items. Flags potential breaking at 3x expected or Poisson P < 0.01. See [SCHEDULER.md](./docs/SCHEDULER.md) for thresholds and Poisson function. |
| Stage 2 correlate | After each Stage 1 | Run correlation query |
| Stage 3 daily | Configurable (default 09:00 in configured timezone) | Midnight-to-midnight synthesis |
| Stage 3 flash | Event-triggered | On `urgency: 'breaking'` from Stage 1 |
| Market pulse | Every 3h (configurable via `pulse_interval`) | Sonnet synthesis of recent summaries, output scales with activity |
| Digest delivery | After Stage 3 daily | Webhook the report |
| Entity decay | Daily 00:00 UTC | `relevance *= 0.95` for all entities |
| Scraper health check | Every 5min | Reset `backoff` sources past `next_retry_at` |

## Cost Tracking

Log every LLM call to `llm_usage`:

```sql
CREATE TABLE llm_usage (
  id TEXT PRIMARY KEY,
  stage TEXT NOT NULL,          -- summarize | synthesize | pre-summarize | urgency-classify | pulse
  model TEXT NOT NULL,
  input_tokens INTEGER NOT NULL,
  output_tokens INTEGER NOT NULL,
  cost_usd DOUBLE PRECISION NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Cost calculation uses published per-token pricing. Daily total available via `/api/v1/status`.

## Future Pipeline Enhancements

All items below are deferred from Phase 1. They are documented here for design continuity.

### Structured-Only Stage 1

Current design: Haiku returns prose `summary` + structured fields. Problem: Sonnet re-interprets Haiku's editorial spin rather than raw facts.

**Recommended change**: drop `summary` from Stage 1 output. Keep only: `entities`, `keyEvents`, `sentiment`, `urgency`. Stage 3 Sonnet writes ALL narrative directly from structured facts + raw keyEvents. Benefits:
- Better signal fidelity (no compression-through-narrative)
- Easier to test (structured output is verifiable)
- Sonnet gets facts, not opinions

Cost: same (Haiku output is smaller, but call count unchanged).

### Stage 3 Split

Problem: Stage 3 currently does both structured extraction (events, entities, sentiment, trends, new projects) and narrative generation (TL;DR, sections) in a single Sonnet call. This couples two distinct concerns and makes it harder to test, cache, or retry each independently.

**Recommended change**: split Stage 3 into two sub-stages:

- **Stage 3a — Structured extraction**: events, entities, sentiment, trends, new projects. Pure structured JSON output. Deterministic, testable, cacheable.
- **Stage 3b — Narrative generation**: TL;DR, sections. Takes 3a output as input. Prose output, harder to test but isolated from extraction logic.

Benefits:
- 3a output is independently useful (API consumers, dashboards, alerts) even if 3b fails or is slow
- 3b can be retried without re-running extraction
- Easier to A/B test narrative styles without affecting structured data
- 3a is a candidate for Haiku if extraction quality is sufficient (cost savings)

Cost: one additional Sonnet call per report (~$0.01-0.03). Offset by caching 3a across report types (daily, flash, pulse can share the same 3a output within a window).

### Urgency Streaming

Problem: per-source poll intervals mean breaking events wait up to one full poll cycle before entering the pipeline.

**Recommended change**: for channels tagged as "breaking-capable" in the `sources` table, add a lightweight urgency-only classifier on Discord MESSAGE_CREATE:

```typescript
// Only for sources with priority >= 2 (high-priority channels)
async function classifyUrgency(message: string): Promise<'routine' | 'breaking'> {
  // ~100 token input, ~10 token output. Cost: ~$0.00001 per call
  const result = await llm.call({
    model: config.models.haiku,
    system: 'Classify this message as "routine" or "breaking" (major exploit, crash, regulatory action). Return ONLY the word.',
    messages: [{ role: 'user', content: message.slice(0, 500) }],
    maxTokens: 10,
    stage: 'urgency-classify'
  });
  return result.content.trim() === 'breaking' ? 'breaking' : 'routine';
}
```

~50-100 calls/day for high-priority channels. Cost: ~$0.01/day. Catches exploits in seconds, triggers flash pipeline immediately.

