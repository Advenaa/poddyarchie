# Twitter/X Integration — twitterapi.io

## Overview

Twitter data is fetched via [twitterapi.io](https://twitterapi.io), an unofficial Twitter API proxy. REST + WebSocket API, pay-per-use pricing, no Twitter developer account needed.

**Risk**: unofficial service — could break if Twitter cracks down. Degrade gracefully if unavailable.

## Authentication

Single API key in header:

```
x-api-key: YOUR_TWITTERAPI_KEY
```

Env var: `TWITTERAPI_KEY`. Get from twitterapi.io dashboard after signup.

## REST Endpoints

Base URL: `https://api.twitterapi.io`

### Advanced Search (for search queries)

```
GET /twitter/tweet/advanced_search
  ?query=from:vitalikbuterin OR "ethereum" since:2026-03-31_00:00:00_UTC
  &queryType=Latest
  &cursor=
```

Supports Twitter's native search syntax: `from:user`, `"exact phrase"`, `OR`, `since:`, `until:`.

### User Last Tweets (for tracked accounts)

```
GET /twitter/user/last_tweets
  ?userName=coaborchain
  &cursor=
  &includeReplies=false
```

20 tweets per page. Use `userId` over `userName` when available.

**Warning**: docs advise against high-frequency polling of individual users. For 10+ accounts, batch into a single search: `from:user1 OR from:user2 OR from:user3`.

## WebSocket (real-time monitoring)

twitterapi.io offers WebSocket/webhook streaming with filter rules for real-time tweet capture. This can power:
- **Live monitoring** of tracked accounts without polling
- **Breaking event detection** — faster than 2h polling cycles
- **Lower cost** — stream instead of repeated search calls

Integration approach:
1. Connect to twitterapi.io WebSocket endpoint
2. Set filter rules for tracked accounts/keywords
3. On incoming tweet, run through `normalize()` immediately
4. Items land in `items` table as `ready` — picked up by next Stage 1 batch

**For MVP**: start with REST polling (simpler). Add WebSocket in v2.1 when real-time matters.

## Response Format

```typescript
interface TwitterApiResponse {
  tweets: TwitterApiTweet[];
  has_next_page: boolean;
  next_cursor: string;
}

interface TwitterApiTweet {
  type: 'tweet';
  id: string;
  url: string;
  text: string;
  source: string;
  retweetCount: number;
  replyCount: number;
  likeCount: number;
  quoteCount: number;
  viewCount: number;
  bookmarkCount: number;
  createdAt: string;              // "Tue Dec 10 07:00:30 +0000 2024"
  lang: string;
  isReply: boolean;
  inReplyToId: string | null;
  conversationId: string;
  author: {
    type: 'user';
    userName: string;
    id: string;
    name: string;
    isBlueVerified: boolean;
    followers: number;
    description: string;
    location: string;
  };
  entities: {
    hashtags: Array<{ text: string }>;
    urls: Array<{ expanded_url: string }>;
    user_mentions: Array<{ screen_name: string }>;
  };
  quoted_tweet: TwitterApiTweet | null;
  retweeted_tweet: TwitterApiTweet | null;
}
```

## Zod Validation Schema

```typescript
import { z } from 'zod';

const TwitterTweetSchema = z.object({
  id: z.string(),
  text: z.string(),
  url: z.string().optional(),
  retweetCount: z.number().default(0),
  replyCount: z.number().default(0),
  likeCount: z.number().default(0),
  quoteCount: z.number().default(0),
  viewCount: z.number().default(0),
  createdAt: z.string(),
  lang: z.string().default('und'),
  isReply: z.boolean().default(false),
  author: z.object({
    userName: z.string(),
    id: z.string(),
    name: z.string(),
    followers: z.number().default(0),
    isBlueVerified: z.boolean().default(false),
  }),
  entities: z.object({
    hashtags: z.array(z.object({ text: z.string() })).default([]),
    urls: z.array(z.object({ expanded_url: z.string() })).default([]),
    user_mentions: z.array(z.object({ screen_name: z.string() })).default([]),
  }).default({}),
});

const TwitterApiResponseSchema = z.object({
  tweets: z.array(TwitterTweetSchema),
  has_next_page: z.boolean(),
  next_cursor: z.string(),
});
```

## Mapping to RawItem

```typescript
function mapTweetToRawItem(tweet: TwitterApiTweet, sourceId: string): RawItem {
  return {
    id: ulid(),
    source: 'twitter',
    sourceId,
    author: tweet.author.userName,
    content: tweet.text,
    timestamp: new Date(tweet.createdAt),
    url: tweet.url,
    engagement: normalizeEngagement(tweet),
    metadata: {
      tweetId: tweet.id,
      likeCount: tweet.likeCount,
      retweetCount: tweet.retweetCount,
      viewCount: tweet.viewCount,
      quoteCount: tweet.quoteCount,
      replyCount: tweet.replyCount,
      lang: tweet.lang,
      isVerified: tweet.author.isBlueVerified,
      followers: tweet.author.followers,
      hashtags: tweet.entities.hashtags.map(h => h.text),
      urls: tweet.entities.urls.map(u => u.expanded_url),
    }
  };
}
```

## Engagement Normalization

Log10-based scale, clamped 1-100:

```typescript
function normalizeEngagement(tweet: TwitterApiTweet): number {
  const raw = tweet.likeCount + tweet.retweetCount * 2 + tweet.quoteCount * 3;
  if (raw === 0) return 1;
  const score = Math.round(Math.log10(raw) * 20);
  return Math.max(1, Math.min(100, score));
}
```

| Raw engagement | Score | Example |
|---------------|-------|---------|
| 1-9 | 1-19 | Low interaction |
| 10-99 | 20-39 | Normal tweet |
| 100-999 | 40-59 | Popular tweet |
| 1K-10K | 60-79 | Viral |
| 10K+ | 80-100 | Mega viral |

## Source ID Convention

In the `sources` table:
- `@handle` → fetch via `/twitter/user/last_tweets`
- Plain text → search query via `/twitter/tweet/advanced_search`

For multiple accounts, prefer batching: `from:user1 OR from:user2 OR from:user3` (one API call, cheaper).

## Polling Model

Called when the source's `poll_interval` elapses (default 2h). Max 5 pages (100 tweets) per source per cycle.

Dedup by tweet ID: store `last_id` in `source_state`, filter out tweets with `BigInt(id) <= BigInt(last_id)`.

Content-hash dedup in normalize layer catches any remaining duplicates.

## Error Handling

| Error | Action |
|-------|--------|
| 401 Invalid API key | Log CRITICAL, disable all Twitter sources |
| 400 Bad request | Log, skip this source this batch |
| 429 Rate limit | Backoff via `source_state.next_retry_at` |
| Network timeout (15s) | Retry next cron cycle |
| Zod validation failure | Log with raw response, skip. Likely API schema change. |
| Empty response | Normal. Log info, update checkpoint. |

## Cost Model

Pay-per-use: **$0.15 per 1,000 tweets fetched**. Free $0.10 credit on signup.

| Setup | Tweets/day | Cost/day | Cost/month |
|-------|-----------|----------|------------|
| 10 accounts, 6 polls | ~1,200 | ~$0.18 | ~$5.40 |
| 10 accounts + 5 searches | ~2,000 | ~$0.30 | ~$9.00 |
| 20 accounts + 10 searches | ~5,000 | ~$0.75 | ~$22.50 |

## Rate Limits

- 200 QPS — never hit with 2h polling
- 20 tweets per page, max 5 pages per poll
- No per-15-minute windows
