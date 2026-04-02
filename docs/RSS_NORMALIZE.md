# RSS Ingestion + Normalize Pipeline

Reference doc for the RSS feed ingestion adapter and the shared normalize pipeline that all sources pass through.

## RSS Ingestion

### Library

`rss-parser` npm package. Handles three feed formats transparently:

| Format | Detection | Notes |
|--------|-----------|-------|
| RSS 2.0 | `<rss version="2.0">` root element | Most common. Maps `<item>` children directly. |
| Atom | `<feed xmlns="http://www.w3.org/2005/Atom">` | Maps `<entry>` to items. `<content>` or `<summary>` for body. |
| JSON Feed | `version` field starting with `https://jsonfeed.org` | Parses `items[]` array. Rare in practice. |

`rss-parser` normalizes all three into a common `Item` shape with `title`, `contentSnippet`, `creator`, `link`, `isoDate`, and `categories`. No format-specific handling needed in the adapter.

### Polling Flow

- **Cron**: every 15 minutes via `scheduler.ts`.
- **Checkpoint**: `source_state.last_id` stores the GUID of the last-seen item per feed.
- **Fallback dedup**: if GUID is missing (some feeds omit `<guid>`), use `sha256(pubDate + title)` as synthetic ID.
- **Window**: on first poll (no `source_state` entry), fetch only items from `now - 2h` to avoid backfilling the entire feed history.
- **Per-feed isolation**: one feed error does not block other feeds. Each feed URL is an independent `source_id`.

```
cron fires (15min)
  for each enabled RSS source:
    fetch feed XML
    parse with rss-parser
    filter items newer than last checkpoint
    sort by pubDate ascending
    for each new item:
      map to RawItem
      call normalize(item)
    update source_state.last_id = newest item GUID
```

### RawItem Mapping

```typescript
const item: RawItem = {
  id: ulid(),
  source: 'rss',
  sourceId: feedUrl,
  author: parsed.creator ?? parsed['dc:creator'] ?? feedTitle,
  content: parsed.contentSnippet ?? parsed.title ?? '',
  timestamp: new Date(parsed.isoDate ?? Date.now()),
  url: parsed.link,
  engagement: 0,  // RSS has no engagement signal
  metadata: {
    feedTitle: feed.title,
    categories: parsed.categories ?? [],
    guid: parsed.guid ?? null
  }
};
```

Key decisions:
- `contentSnippet` over `content` because `content` is raw HTML. `contentSnippet` is the text-only version `rss-parser` extracts.
- `creator` falls back to the feed title when individual item authors are missing.
- `engagement: 0` always. RSS provides no reaction/like data.

### Article Extraction Handoff

If the mapped `content` is under 500 characters, the RSS snippet is likely truncated. Auto-fetch the full article:

```
if content.length < 500 AND item.link exists:
  validate URL via url-validator.ts (SSRF protection)
  fetch item.link (10s timeout)
  extract with Readability
  if extraction succeeds AND extracted.length > content.length:
    replace content with extracted text
```

This catches feeds that publish summaries only (common with Indonesian news sites). The SSRF validator rejects private IPs, non-HTTPS, and non-standard ports before any fetch.

### Indonesian Feed Specifics

**Encoding**: all feeds must be UTF-8. `rss-parser` handles the XML encoding declaration. Indonesian characters (e.g., accented vowels in loanwords) pass through without issue.

**Non-standard dates**: some Indonesian feeds use `DD/MM/YYYY HH:mm` or omit timezone. The adapter falls back to `new Date()` parsing and defaults to `Asia/Jakarta` (WIB, UTC+7) when no timezone offset is present.

**Recommended feeds**:

| Feed | URL | Notes |
|------|-----|-------|
| Detik Finance | `https://rss.detik.com/index.php/finance` | Broad IDX/macro coverage |
| Kontan | `https://www.kontan.co.id/rss` | Financial news, regulatory |
| Bisnis.com | `https://www.bisnis.com/rss` | Business/market focus |
| Blockchainmedia.id | `https://blockchainmedia.id/feed/` | Indonesian crypto news |

Do NOT add CoinDesk Indonesia -- the site is dead and the feed returns 404.

**Language**: these feeds publish in Bahasa Indonesia. They pass the normalize language filter (`franc` detects `ind`) and get summarized into English by the Haiku stage.

### Error Handling

| Condition | Behavior |
|-----------|----------|
| Malformed XML (parse failure) | Increment `error_count` in `source_state`, apply backoff (`2s * 2^error_count`, capped 1h). Log warning with feed URL and parse error. |
| Partial feed (valid XML, some items malformed) | Skip malformed items, process the rest. Log each skip. Do NOT increment `error_count` for partial success. |
| Missing required fields (`title` AND `contentSnippet` both null) | Drop item silently. A feed entry with no text content has no signal. |
| HTTP 404/410 (feed gone) | Increment `error_count`. After 5 failures in 1h, status -> `disabled`. Surfaces on `/settings`. |
| HTTP 429 (rate limited) | Backoff. Respect `Retry-After` header if present, otherwise standard exponential backoff. |
| Network timeout (>10s) | Treat as transient error. Standard backoff. |
| SSRF validation failure (Readability fetch) | Skip article extraction, keep the RSS snippet as-is. Log warning. |

---

## Normalize Pipeline

Every source (Discord, Twitter, RSS, news) passes through the same normalize function. Pure rule-based — no LLM calls. The flow is a decision tree, not a flat list -- items exit at the first failing gate.

### Decision Tree

```
incoming RawItem
  |
  v
[1] TRUNCATE
  content > 20K chars? --> truncate to 20K
  |
  v
[2] CONTENT HASH DEDUP
  sha256(source + sourceId + content[0:200])
  hash exists in items table? --> DROP (silent)
  |
  v
[3] URL DEDUP (cross-source)
  item has URL?
    yes --> expand URL (resolve t.co, bit.ly)
            expanded URL exists in items table? --> DROP (silent)
    no  --> continue
  |
  v
[4] SPAM FILTER
  matches any spam rule? --> INSERT as 'filtered', RETURN
  |
  v
[5] LANGUAGE DETECT
  franc(content) returns lang code
  lang in {eng, ind, und}? --> continue
  else --> INSERT as 'filtered', RETURN
  |
  v
[6] PASS
  INSERT as 'ready' (with full original content)
```

Items that pass get `status: 'ready'` with full original content and wait for pre-summarization (if applicable) and the next Stage 1 batch. Items filtered at steps 4-5 get `status: 'filtered'` and are kept in the DB for debugging. Items deduped at steps 2-3 are dropped entirely (no DB row).

**Note:** Article pre-summarization is NOT part of normalize. It runs as a separate step after normalize — see the "Article Pre-Summarization" section below.

### Spam Filter Rule Registry

Rules are defined as an array of `SpamRule` objects. Each rule has a `name`, `test` function, and optional `sources` scope.

```typescript
interface SpamRule {
  name: string;
  sources?: ('discord' | 'twitter' | 'rss' | 'news')[];  // undefined = all sources
  test: (item: RawItem) => boolean;
}

const SPAM_RULES: SpamRule[] = [
  // --- English patterns ---
  { name: 'short-post',       test: i => i.content.split(/\s+/).length < 5 },
  { name: 'gm-gn',            test: i => /^(gm|gn|gm\/gn)\s*[!.]*$/i.test(i.content.trim()) },
  { name: 'bot-author',       test: i => /\bbot\b/i.test(i.author) },

  // --- Indonesian patterns ---
  { name: 'id-wm',            test: i => /^wm\s*$/i.test(i.content.trim()) },
  { name: 'id-done-min',      test: i => /^(done\s*min|sudah\s*min)/i.test(i.content.trim()) },
  { name: 'id-gas',           test: i => /^gas\s*!*$/i.test(i.content.trim()) },
  { name: 'id-mantap',        test: i => /^mantap\s*[!.]*$/i.test(i.content.trim()) },
  { name: 'single-emoji',     test: i => /^\p{Emoji}\s*$/u.test(i.content.trim()) },
  { name: 'airdrop-copypasta',test: i => /airdrop/i.test(i.content) && i.content.includes('0x') },
];
```

**Adding a new rule**: append to `SPAM_RULES` with a descriptive `name`. Optionally scope to specific sources. The `test` function receives the full `RawItem` and returns `true` to filter.

**Testing rules**: unit tests in `test/unit/normalize.test.ts` use golden files. Add test cases as `{ input: RawItem, expected: 'ready' | 'filtered' }` objects. Run `npm test` to verify.

Spam filtering kills 40-60% of Discord volume. RSS items rarely trigger spam rules but the `short-post` rule catches malformed feed entries.

### Language Detection

Uses the `franc` library which returns ISO 639-3 codes (three-letter).

| Code | Language | Action |
|------|----------|--------|
| `eng` | English | Accept |
| `ind` | Indonesian | Accept |
| `und` | Undetermined | Accept (pass through) |
| anything else | Other | Filter |

**`und` handling**: `franc` returns `und` for short texts (typically under ~50 characters) where there is not enough signal to classify. These pass through because short messages in Discord/Twitter are often English or Indonesian but too brief for detection. Filtering `und` would discard legitimate content.

### Article Pre-Summarization (separate step, after normalize)

RSS and news items exceeding 4000 characters (~1000 tokens) get a lightweight Haiku pre-summary before entering the chunking pipeline. This equalizes signal density -- a 10K-char article otherwise dominates a chunk's token budget while carrying the same signal as a 500-char Discord message.

**This is a separate pipeline step that runs after normalize**, not inside it. Normalize is pure rule-based (no LLM calls). Items are saved as `ready` with full original content by normalize, then pre-summarize runs on eligible `ready` items before Stage 1 claims them.

```
After normalize saves item as 'ready':

if (source is 'rss' or 'news') AND content.length > 4000:
  call Haiku with:
    system: "Summarize this article for a market analyst. Keep all facts,
             entities, numbers, and quotes. Drop boilerplate and filler.
             Output plain text, no JSON."
    input:  content (truncated to 16K chars)
    maxTokens: 300
  update item.content in DB with summary
```

Because pre-summarize runs after language detection (which happens in normalize), the language check runs on the original content. Indonesian articles are correctly detected as `ind` and pass through. The pre-summary output (always English from Haiku) replaces the content afterward.

Cost: ~$0.001 per article at Haiku rates.

