# Integration Flow — How Messages Become Reports

## The Full Journey of a Message

```
Discord msg → Ingest → Normalize* → Wait → Batch Claim → Chunk → Haiku → Summary + Entities
                                                                              ↓
Twitter tweet → Ingest → Normalize* → Wait → ─── same ───────────────────── Correlate (SQL)
                                       ↓                                      ↓
RSS entry → Ingest → Normalize* → Pre-summarize → Wait → ─── same ──── Synthesize (Sonnet)
                                       ↓                                      ↓
News article → Ingest → Normalize* → Pre-summarize → Wait → ─── same ── Report → Webhook

                                                               Every 3h ── Pulse (Sonnet) → Webhook

*Normalize includes: truncate → InstructDetector scan → dedup → spam → lang detect → Ind→EN translate (Haiku)
```

## Step by Step

### 1. A Discord message arrives

A WebSocket connection receives a `MESSAGE_CREATE` event. The handler in `discord.ts`:

```typescript
// Pseudocode
function onMessageCreate(event: GatewayEvent) {
  const item: RawItem = {
    id: ulid(),
    source: 'discord',
    sourceId: event.channel_id,
    author: event.author.username,
    content: event.content,
    timestamp: new Date(event.timestamp),
    engagement: event.reactions?.length ?? 0,  // Discord: reaction count. RSS: always 0. Twitter: log10 formula.
    url: undefined,
    metadata: {
      guildId: event.guild_id,
      messageId: event.id,
      // Image attachment URLs — stored for raw feed display
      imageUrls: event.attachments
        ?.filter(a => a.content_type?.startsWith('image/'))
        .map(a => a.url) ?? [],
    }
  };
  
  await normalize(item);  // immediate, async (DB lookups for dedup)
}
```

### 2. Normalize decides if it survives

Normalize is mostly rule-based. The only LLM call is Indonesian→English translation (Haiku).

```typescript
function normalize(item: RawItem) {
  // Truncate oversized content (>20K chars → 20K)
  if (item.content.length > 20000) {
    item.content = item.content.slice(0, 20000);
  }
  
  // Layer 1: Unicode normalization + zero-width strip (Decision 06)
  item.content = item.content.normalize('NFKC');
  item.content = item.content.replace(/[\u200B-\u200F\u2028-\u202F\uFEFF]/g, '');
  
  // Layer 2: InstructDetector scan — prompt injection defense (Decision 06)
  const hasInjection = await instructDetector.scan(item.content);
  if (hasInjection) {
    insertItem(item, '', 'filtered', 'injection_detected');
    return;
  }
  
  // Content hash for dedup (after sanitization, so hashes are stable)
  const hash = sha256(item.source + item.sourceId + item.content.slice(0, 200));
  
  // Check for duplicate
  const existing = await pool.query('SELECT id FROM items WHERE content_hash = $1', [hash]);
  if (existing.rows.length > 0) return; // drop silently
  
  // URL dedup (cross-source)
  if (item.url) {
    const expandedUrl = expandUrl(item.url);  // resolve t.co etc
    const urlDup = await pool.query('SELECT id FROM items WHERE url = $1', [expandedUrl]);
    if (urlDup.rows.length > 0) return;
    item.url = expandedUrl;
  }

  // Spam filter
  if (isSpam(item)) {
    insertItem(item, hash, 'filtered');
    return;
  }
  
  // Language detection
  const lang = franc(item.content);
  if (lang !== 'eng' && lang !== 'ind' && lang !== 'und') {
    insertItem(item, hash, 'filtered');
    return;
  }
  
  // Indonesian → English translation (Decision 01)
  // Translate before saving so all downstream processing works with English text.
  // 14-17pp accuracy loss avoided by translating before Stage 1 (XBridge architecture).
  if (lang === 'ind') {
    const translated = await llm.call({
      model: config.models.haiku,
      system: 'Translate the following Indonesian text to English. Preserve all entity names, numbers, and technical terms. Output only the translation.',
      prompt: item.content,
      maxTokens: Math.ceil(item.content.length / 4) * 1.5
    });
    item.content = translated.text;
    item.metadata.originalLanguage = 'ind';
    item.metadata.translated = true;
  }
  
  // Passed all checks — saved with full original content (translated if Indonesian)
  insertItem(item, hash, 'ready');
}
```

The item is now in the `items` table with `status: 'ready'` and full original content, waiting for pre-summarization (if applicable) and the next batch run.

### 2.5. Pre-summarize compresses long articles

A separate step after normalize, before batch claim. RSS/news items exceeding 4000 characters (~1000 tokens) get a lightweight Haiku pre-summary to equalize signal density across source types.

```typescript
async function preSummarizeReadyItems() {
  // Find ready items that need pre-summarization
  const { rows: items } = await pool.query(`
    SELECT id, content FROM items
    WHERE status = 'ready'
    AND source IN ('rss', 'news')
    AND LENGTH(content) > 4000
    AND pre_summarized = false
  `);

  for (const item of items) {
    const summary = await preSummarize(item.content); // Haiku, max 300 tokens output
    await pool.query(
      'UPDATE items SET content = $1, pre_summarized = true WHERE id = $2',
      [summary, item.id]
    );
  }
}
```

This is an LLM call (Haiku) and runs as its own pipeline step, not inside normalize. See PIPELINE.md for the `preSummarize` function details.

### 3. Per-source intervals trigger processing

Each source has its own `poll_interval` (e.g., Discord channels every 30min, Twitter accounts every 2h, RSS feeds every 15min). The scheduler fires each source independently based on its interval, rather than a single global cron. When a source's interval elapses, `summarize.runSource()` is called. Sources are processed in parallel via `Promise.allSettled`:

```typescript
async function runBatch() {
  // Mutex check
  if (running) { logger.warn('Stage 1 skipped: previous run still active'); return; }
  running = true;
  
  try {
    // Get all active sources due for processing
    const { rows: sources } = await pool.query(
      `SELECT source, source_id, poll_interval FROM sources
       WHERE enabled = true
       AND last_processed_at + poll_interval * interval '1 second' <= NOW()`
    );
    
    // Process sources in parallel
    await Promise.allSettled(
      sources.map(({ source, source_id }) => processSingleSource(source, source_id))
    );
    
    // After ALL sources processed, run correlation
    correlate.run();
    
  } finally {
    running = false;
  }
}
```

### 4. Process a single source — the chunking

```typescript
async function processSingleSource(source: string, sourceId: string) {
  const batchId = ulid();
  const windowEnd = Date.now();
  const windowStart = getLastBatchEnd(source, sourceId); // from source_state, defaults to now - poll_interval for new sources
  
  // Step 1: Claim items (atomic)
  const claimed = await pool.query(`
    UPDATE items SET batch_id = $1, status = 'processing'
    WHERE status = 'ready' AND source = $2 AND source_id = $3
    AND timestamp BETWEEN $4 AND $5
  `, [batchId, source, sourceId, windowStart, windowEnd]);
  
  if (claimed.rowCount === 0) {
    logger.info({ source, sourceId }, 'No items to process');
    return;
  }
  
  // Step 2: Load claimed items
  const { rows: items } = await pool.query(
    'SELECT * FROM items WHERE batch_id = $1 ORDER BY timestamp ASC',
    [batchId]
  );
  
  // Step 3: Chunk by token budget
  const chunks = chunkByTokens(items, 6000);
  
  // Step 4: Process each chunk through Haiku (with confidence escalation)
  const ambiguousEntities: AmbiguousEntity[] = []; // collect for batched disambiguation
  
  for (const chunk of chunks) {
    let summary = await summarizeChunk(source, sourceId, chunk, windowStart, windowEnd);
    
    // Confidence-based escalation to Sonnet (Decision 10)
    // Low confidence + non-routine = Haiku is unsure about something important → escalate
    if (summary && summary.confidence < 5 && summary.urgency !== 'routine') {
      logger.info({ chunkId: chunk.id, confidence: summary.confidence, urgency: summary.urgency },
        'Escalating chunk to Sonnet');
      summary = await summarizeChunk(source, sourceId, chunk, windowStart, windowEnd, 'sonnet');
    }
    
    if (summary) {
      // Write summary to DB
      saveSummary(summary);
      
      // Three-tier entity disambiguation (Decision 09)
      // Tier 1: alias lookup (instant, handles ~95% of entities)
      // Tier 2: co-occurring entity context check (free, resolves ambiguous names)
      // Tier 3: batched Haiku call (deferred, for remaining ambiguity)
      for (const entity of summary.entities) {
        const resolved = await resolveEntity(entity.name, summary.entities.map(e => e.name));
        if (resolved) {
          upsertEntityMention(resolved.id, source, summary.id, entity);
        } else if (entity.candidates?.length > 1) {
          // Ambiguous — defer to batched LLM disambiguation
          ambiguousEntities.push({
            name: entity.name,
            surroundingText: chunk.rawText,
            candidates: entity.candidates,
          });
        }
      }
      
      // Check for flash trigger — weighted trust sum (Decision 02)
      // Old: fire if 2+ distinct sources report breaking
      // New: sum trust weights of sources reporting breaking, fire if >= 1.5
      if (summary.urgency === 'breaking') {
        const breakingChunks = await getRecentBreakingChunks('30 minutes');
        const weightedSum = breakingChunks.reduce((sum, c) => {
          const src = sources.find(s => s.id === c.sourceId);
          return sum + (src?.trustWeight ?? 0.5);
        }, 0);
        if (weightedSum >= 1.5) {
          await synthesize.runFlash();
        }
      }
    }
  }
  
  // Tier 3: Batched LLM disambiguation for remaining ambiguous entities (Decision 09)
  if (ambiguousEntities.length > 0) {
    const prompt = ambiguousEntities.map((e, i) =>
      `${i+1}. "${e.name}" in context: "${e.surroundingText.slice(0, 200)}"
      Candidates: ${e.candidates.map(c => `${c.name} (${c.type})`).join(', ')}`
    ).join('\n\n');
    
    const response = await llm.call({
      model: config.models.haiku,
      system: 'For each ambiguous entity, select the correct candidate. Output JSON: [{"index": 1, "selected": "name"}]',
      prompt,
      maxTokens: ambiguousEntities.length * 50,
    });
    // Save results as new aliases — self-improving, reduces future LLM calls
    for (const resolution of JSON.parse(response.text)) {
      const entity = ambiguousEntities[resolution.index - 1];
      await insertAlias(entity.name, entity.contextKey, resolution.selectedId);
    }
  }
  
  // Step 5: Mark items as processed (atomic)
  await pool.query(`UPDATE items SET status = 'processed' WHERE batch_id = $1`, [batchId]);
  
  // Update checkpoint
  updateSourceState(source, sourceId, windowEnd);
}
```

### 5. Inside `summarizeChunk` — the actual LLM call

```typescript
async function summarizeChunk(
  source: string,
  sourceId: string,
  items: RawItem[],
  windowStart: number,
  windowEnd: number
): Promise<ChunkSummary | null> {
  
  // Format items for the prompt
  const formattedContent = items.map(item => 
    `[${item.author} at ${new Date(item.timestamp).toISOString()}]\n${item.content}`
  ).join('\n---\n');
  
  // Budget hints — complexity-adaptive output control (Decision 03)
  // Sparse chunks get a conciseness hint; dense chunks write freely.
  const isDense = items.length > 15
    || estimateTokens(formattedContent) > 4000
    || items.some(i => /breaking|exploit|hack|emergency/i.test(i.content));
  const budgetHint = isDense
    ? '' // no constraint — let Haiku write freely
    : '\nTarget ~400 tokens for the summary field. Be concise.';
  
  // Build the prompt (see PIPELINE.md for full system prompt)
  const systemPrompt = buildStage1Prompt(source, sourceId, windowStart, windowEnd) + budgetHint;
  // Nonce-based XML wrapping (Layer 3 defense, see docs/LLM.md)
  const nonce = crypto.randomBytes(4).toString('hex');
  const userMessage = `<scraped_content_${nonce}>\n${sanitizeForPrompt(formattedContent)}\n</scraped_content_${nonce}>`;
  
  // Call LLM through the wrapper (handles retries, errors, cost logging)
  const response = await llm.call({
    model: config.models.haiku,  // from config.ts, default 'claude-haiku-4-5-20251001'
    system: systemPrompt,
    messages: [{ role: 'user', content: userMessage }],
    maxTokens: 3000,
    stage: 'summarize'
  });
  
  // Validate with zod — wrap JSON.parse in try-catch for injection defense (Decision 06)
  let jsonParsed: unknown;
  try {
    jsonParsed = JSON.parse(response.content);
  } catch (e) {
    if (e instanceof SyntaxError) {
      // JSON parse failure — retry with fresh prompt, no history (attack #6 defense)
      logger.warn({ source, sourceId }, 'Stage 1 JSON parse failed, retrying with fresh prompt');
      const freshResponse = await llm.call({
        model: config.models.haiku,
        system: systemPrompt,
        messages: [{ role: 'user', content: userMessage }],
        maxTokens: 3000,
        stage: 'summarize'
      });
      try {
        const freshParsed = ChunkSummarySchema.safeParse(JSON.parse(freshResponse.content));
        if (!freshParsed.success) {
          logger.error({ source, sourceId, error: freshParsed.error }, 'Stage 1 parse failed after fresh retry');
          return null;
        }
        return freshParsed.data;
      } catch {
        logger.error({ source, sourceId }, 'Stage 1 JSON parse failed twice, skipping chunk');
        return null;
      }
    }
    throw e;
  }

  const parsed = ChunkSummarySchema.safeParse(jsonParsed);
  
  if (!parsed.success) {
    // Retry with zod error-path feedback (Decision 08)
    // Feed specific error paths back so Haiku knows exactly what to fix.
    // PARSE (EMNLP 2025): 92% error reduction vs generic "try again" retry.
    // Security: only send original userMessage + error descriptions — never include
    // the failed assistant response in retry context (attack #6 defense).
    const errorMessages = parsed.error.issues.map(issue =>
      `- ${issue.path.join('.')}: ${issue.message} (received: ${JSON.stringify((issue as any).received)})`
    ).join('\n');
    
    const retryResponse = await llm.call({
      model: config.models.haiku,
      system: systemPrompt,
      messages: [
        { role: 'user', content: userMessage },
        { role: 'user', content: `Your previous output had these validation errors:\n${errorMessages}\n\nFix ONLY these fields. Keep everything else the same.` }
      ],
      maxTokens: 3000,
      stage: 'summarize'
    });
    
    try {
      const retryParsed = ChunkSummarySchema.safeParse(JSON.parse(retryResponse.content));
      if (!retryParsed.success) {
        logger.error({ source, sourceId, error: retryParsed.error }, 'Stage 1 parse failed after retry');
        return null;  // skip this chunk
      }
      return retryParsed.data;
    } catch {
      logger.error({ source, sourceId }, 'Stage 1 JSON parse failed on retry response');
      return null;
    }
  }
  
  return parsed.data;
}
```

### 6. Correlation — SQL, no LLM

After all sources are processed, `correlate.run()` fires:

```typescript
function run(): CorrelatedEntity[] {
  const windowStart = Date.now() - 24 * 60 * 60 * 1000; // 24h ago
  
  const { rows } = await pool.query(`
    SELECT e.name, e.type,
           COUNT(DISTINCT em.source) AS source_count,
           AVG(em.sentiment) AS avg_sentiment,
           string_agg(DISTINCT em.source, ',') AS sources
    FROM entity_mentions em
    JOIN entities e ON e.id = em.entity_id
    WHERE em.created_at > $1
    GROUP BY e.id, e.name, e.type
    HAVING COUNT(DISTINCT em.source) >= 2
    ORDER BY source_count DESC, ABS(AVG(em.sentiment)) DESC
    LIMIT 20
  `, [windowStart]);
  return rows;
}
```

Result: entities that appeared in 2+ sources. These become explicit context for Stage 3.

### 7. Synthesis — the daily report

At the configured `digest_time` (default 09:00 in user's timezone), `synthesize.runDaily()` fires:

```typescript
async function runDaily() {
  const { timezone } = getConfig();
  
  // Midnight-to-midnight in configured timezone
  const { start, end } = getMidnightWindow(timezone);
  
  // Load all summaries created in window (use created_at, not window bounds,
  // to avoid missing summaries whose micro-batch window spans midnight)
  const { rows: summaries } = await pool.query(
    'SELECT * FROM summaries WHERE created_at >= $1 AND created_at <= $2',
    [start, end]
  );
  
  if (summaries.length === 0) {
    logger.info('No summaries for daily report, skipping');
    return;
  }
  
  // Get correlated signals
  const correlations = correlate.run();
  
  // Get yesterday's TL;DR for temporal anchoring
  const { rows: [yesterday] } = await pool.query(
    "SELECT tldr FROM reports WHERE type = 'daily' ORDER BY created_at DESC LIMIT 1"
  );
  
  // Rank if too many summaries
  const ranked = summaries.length > 50
    ? summaries.sort((a, b) => scoreForRanking(b) - scoreForRanking(a)).slice(0, 30)
    : summaries;
  
  // Select raw exemplars: 7 by engagement + 3 reserved for RSS/news
  // Ensures news/RSS voice is always represented even when Discord dominates engagement
  const exemplars = selectExemplars(ranked, { engagement: 7, rssNews: 3 });
  
  // Narrative clustering — detect emerging themes from summary embeddings (Decision 05)
  // Clusters existing embeddings via k-means, names each cluster with one Haiku call.
  // Provides the WHY layer: entities = WHAT, sentiment = HOW, narratives = WHY.
  const narratives = await detectNarratives(ranked);
  
  // Build input and call Sonnet
  // Stage 3 prompt includes explicit causal instruction:
  //   "State causal connections explicitly. If A caused B, say so.
  //    If correlation is unconfirmed, qualify with 'likely' or 'appears to'."
  // Daily synthesis includes raw Stage 1 correlated entity data alongside the 8 pulses,
  // so Sonnet can cross-check pulse narrative against actual data.
  const input = assembleStage3Input(ranked, correlations, yesterday?.tldr, narratives);
  const report = await callSonnet(input);  // see PIPELINE.md for prompt
  
  // Check for duplicate daily
  const { rows: [dup] } = await pool.query(
    "SELECT id FROM reports WHERE date = $1 AND type = 'daily'",
    [report.date]
  );
  if (dup) {
    logger.warn({ date: report.date }, 'Daily report already exists, skipping');
    return;
  }
  
  // Save report
  await pool.query(`
    INSERT INTO reports (id, date, type, body, tldr, sentiment, delivery_status, created_at)
    VALUES ($1, $2, 'daily', $3, $4, $5, 'pending', $6)
  `, [report.id, report.date, JSON.stringify(report), report.tldr, report.sentiment, Date.now()]);
  
  // Deliver
  await webhook.deliver(report);
}
```

### 8. Webhook delivery

```typescript
async function deliver(report: MarketReport) {
  const config = getConfig();
  if (!config.webhookUrl) {
    logger.warn('No webhook URL configured, skipping delivery');
    return;
  }
  
  // Format for Discord embed
  const embed = formatDiscordEmbed(report, config.publicUrl);
  
  // Retry loop: 3 attempts, backoff 2s → 8s → 32s
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const res = await fetch(config.webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ embeds: [embed] }),
        signal: AbortSignal.timeout(10000)
      });
      
      if (res.ok) {
        await pool.query("UPDATE reports SET delivery_status = 'delivered', delivered_at = $1 WHERE id = $2",
          [Date.now(), report.id]);
        return;
      }
      
      if (res.status === 429 || res.status >= 500) {
        await sleep(2000 * Math.pow(4, attempt));  // 2s, 8s, 32s
        continue;
      }
      
      // 4xx (not 429) — permanent failure
      throw new Error(`Webhook failed: ${res.status}`);
      
    } catch (err) {
      if (attempt === 2) {
        logger.error({ err, reportId: report.id }, 'Webhook delivery failed permanently');
        await pool.query("UPDATE reports SET delivery_status = 'failed' WHERE id = $1", [report.id]);
      }
    }
  }
}

// See docs/WEBHOOK.md for full format spec, mobile rules, and ASCII mockups
function formatDiscordEmbed(report: MarketReport, publicUrl?: string) {
  const isFlash = report.type === 'flash';
  const fields = [];

  // Key events — single field with blockquotes (mobile-friendly)
  fields.push({
    name: 'Key Events',
    value: report.keyEvents.slice(0, isFlash ? 2 : 5).map(e => `> ${e}`).join('\n'),
    inline: false
  });

  // Entity sentiment — daily only, compact text (no ASCII bars, breaks on mobile)
  if (!isFlash && report.entitySentiment?.length) {
    fields.push({
      name: 'Sentiment',
      value: report.entitySentiment.slice(0, 6).map(e => {
        const label = e.sentiment > 0.2 ? 'bullish' : e.sentiment < -0.2 ? 'bearish' : 'neutral';
        const trend = e.trend === 'new' ? ' *new*' : '';
        return `**${e.name}** ${e.sentiment > 0 ? '+' : ''}${e.sentiment.toFixed(1)} ${label}${trend}`;
      }).join('\n'),
      inline: false
    });
  }

  // Source coverage — daily only
  if (!isFlash) {
    fields.push({
      name: 'Coverage',
      value: `${report.itemCount} items from ${report.sourceCount} sources`,
      inline: false
    });
  }

  return {
    title: isFlash
      ? `[FLASH] ${report.keyEvents[0]?.slice(0, 60) || 'Market Alert'}`
      : `Daily Market Report \u2014 ${report.date}`,
    description: report.tldr,
    fields,
    url: publicUrl ? `${publicUrl}/reports/${report.id}` : undefined,
    color: isFlash ? 0xFF6B35 : 0x5B8DEF,
    timestamp: report.generatedAt,
    footer: { text: isFlash ? 'podders FLASH' : 'podders' }
  };
}
```

## Timing Example

A typical day for a setup with 5 Discord channels, 10 Twitter accounts, 20 RSS feeds. Each source has its own poll interval -- sources are processed in parallel when their intervals overlap:

```
00:00 UTC  — Entity decay cron runs (relevance *= 0.95)
00:05 UTC  — Data retention cron (delete old items, prune dead entities)
00:10 UTC  — Backup cron (pg_dump)

             Per-source processing runs throughout the day:
             - Discord channels: every 30min (high-frequency chatter)
             - Twitter accounts: every 2h (API cost management)
             - RSS feeds: every 15min (fast article pickup)

00:30 UTC  — Discord batch fires (5 channels processed in parallel)
             - Claims ~80 Discord msgs across all channels
             - Normalizes: ~40 survive (spam/dedup/language filtered)
             - Chunks into ~3 groups (token-budgeted)
             - 3 Haiku calls → 3 summaries + ~15 entity upserts
             - Correlation: finds 1 entity in 2+ sources

00:45 UTC  — RSS batch fires (20 feeds processed in parallel)
             - Claims ~8 new articles
             - Pre-summarize long articles, chunk, summarize

01:00 UTC  — Discord batch fires again (30min interval)
02:00 UTC  — Twitter batch fires (2h interval, 10 accounts in parallel)
             - Claims ~50 tweets across all accounts
             - Chunks + summarizes in parallel with any overlapping source jobs

09:00 WIB  — Stage 3 daily synthesis (user's timezone: Asia/Jakarta = UTC+7 = 02:00 UTC)
(02:00 UTC)  - Reads all summaries from midnight-midnight WIB
             - 1 Sonnet call → MarketReport
             - Webhook delivers to Discord
             - User reads TL;DR on their phone

12:00 WIB  — Market pulse fires (every 3h)
(05:00 UTC)  - Reads summaries from last 3h window
             - Computes drift flags (prior pulse entity sentiments vs current Stage 1 data)
             - Injects ground truth entity table + drift flags into prompt context
             - 1 Sonnet call → short pulse (output scales with activity)
             - Webhook delivers muted grey embed
             - Quiet periods get shorter pulses, busy periods get fuller ones

14:30 UTC  — [BREAKING] Stage 1 detects urgency:'breaking' in a Discord batch
             - Immediately triggers Stage 3 flash
             - Flash report delivered via webhook with [FLASH] prefix
```

## How Chunks Are Built

Given 200 items in a single poll interval window across one Discord channel:

```
Item 1:  "gm"                          →  filtered (spam)
Item 2:  "ETH looking bullish today"   →  ~6 tokens
Item 3:  "Has anyone seen the new..."  →  ~15 tokens
...
Item 87: "Just read the audit report"  →  ~8 tokens
─── chunk boundary (6000 token budget reached) ───
Item 88: "Interesting thread on..."    →  ~12 tokens
...
Item 142: "Summary of the call..."     →  ~20 tokens
─── chunk boundary ───
...remaining items form chunk 3
```

Each chunk gets its own Haiku call. The summaries are independent — they overlap in time but cover different items. Entity extraction from each chunk gets merged via the alias resolution table.

## Indonesian Content Handling

Indonesian-language messages are detected by `franc` in normalize and translated to English via a Haiku call before being saved as `ready` (Decision 01). This means all downstream processing (Stage 1, correlation, synthesis, embeddings) works with English text.

- Translation happens once at ingest — not repeated at each stage
- Original language tracked in `item.metadata.originalLanguage` and `item.metadata.translated`
- Entity canonical names use English form ("Bank Indonesia" not "BI", "Indonesian rupiah" not "IDR")
- This allows cross-source correlation between Indonesian Discord and English Twitter/news
- Chat queries in Indonesian are also translated before search (Decision 11, see section 10 below)
- Cost: ~$0.02-0.04/day for 15-20 extra Haiku calls on Indonesian chunks

### 9. RAG Chat -- how a question becomes an answer

Sonnet receives three retrieval tools and picks the right one per query (Decision 07):

```typescript
// Tool 1: semantic_search — cosine similarity vs summary/report embeddings
//   Use for: "What happened with SOL?", topic/theme questions
//   Backend: Gemini embeddings, in-memory cosine, top 10 > 0.3 threshold

// Tool 2: keyword_search — entity table lookup
//   Use for: "How is sentiment on ETH?", mention counts, source breakdown
//   Backend: Postgres entities + entity_mentions tables

// Tool 3: read_raw — fetch original source message by item ID
//   Use for: "Show me the original message about the exploit"
//   Backend: items table, returns full content + metadata

async function handleChat(userMessage: string, history: ChatMessage[]): Promise<string> {
  // Language detection + translation for Indonesian queries (Decision 11)
  const lang = detectLanguage(userMessage);
  if (lang === 'ind') {
    userMessage = await llm.call({
      model: config.models.haiku,
      system: 'Translate Indonesian to English. Preserve entity names and technical terms. Output only the translation.',
      prompt: userMessage,
      maxTokens: 200
    }).then(r => r.text);
  }
  
  const response = await llm.call({
    model: config.models.sonnet,
    system: CHAT_SYSTEM_PROMPT,
    messages: [...history, { role: 'user', content: userMessage }],
    tools: chatTools,  // semantic_search, keyword_search, read_raw
    maxTokens: 2000
  });
  
  // Handle tool calls iteratively — Sonnet may chain multiple tools
  // Tool results are framed as untrusted data with nonce-based XML tags (Layer 3 defense).
  // read_raw results get sanitizeForPrompt() since they contain original source content.
  while (response.toolCalls?.length) {
    const rawResults = await executeToolCalls(response.toolCalls);
    const framedResults = rawResults.map((result: ToolResult) => {
      const resultNonce = crypto.randomBytes(4).toString('hex');
      const content = result.toolName === 'read_raw'
        ? sanitizeForPrompt(result.content)
        : result.content;
      return {
        ...result,
        content: `<tool_result_${resultNonce}>\n${content}\n</tool_result_${resultNonce}>`,
      };
    });
    response = await llm.call({
      model: config.models.sonnet,
      system: CHAT_SYSTEM_PROMPT,
      messages: [...history, { role: 'user', content: userMessage }, ...framedResults],
      tools: chatTools,
      maxTokens: 2000
    });
  }
  
  return response.text;
}
```

### 10. Chat query translation -- Indonesian queries

Indonesian chat queries are detected and translated to English before search (Decision 11). All summaries and embeddings are in English (after Decision 01 translates Indonesian source content at ingest), so English query vs English embeddings gives the best cosine similarity.

```
Indonesian query → franc detects 'ind' → Haiku translates to English → search English embeddings
English query    → franc detects 'eng' → search English embeddings directly (no translation)
```

Cost: maybe 1-2 Indonesian queries per day, negligible.
