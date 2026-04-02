# Webhook Embed Format Spec

## Discord Embed Limits vs MarketReport

| Constraint          | Limit  | Our usage (daily) | Our usage (flash) |
|---------------------|--------|--------------------|--------------------|
| Description         | 4096   | ~500 (TL;DR only)  | ~300               |
| Fields              | 25     | 5-9                | 2-3                |
| Field name          | 256    | ~40                | ~40                |
| Field value         | 1024   | ~200               | ~150               |
| Total embed chars   | 6000   | ~2500              | ~1000              |

Fits comfortably. The report `sections` array (long prose) does NOT go in the embed -- that is dashboard-only content. The webhook is a summary of the summary.

## Daily Digest Embed

```
color: #5B8DEF (blue sidebar)

+--------------------------------------------------+
| Daily Market Report -- 2026-03-31         [link]  |
|                                                    |
| ETH staking withdrawals hit 90-day high as        |
| Shanghai upgrade uncertainty drives repositioning. |
| Cross-source signals confirm Uniswap v4 audit     |
| passed with no critical findings. Macro tone       |
| cautious ahead of Friday's NFP print.             |
|                                                    |
| -- Key Events ----------------------------------- |
| > Uniswap v4 audit passed -- no critical          |
|   findings [Discord, Twitter]                      |
| > ETH staking withdrawals spike 3x avg            |
|   [Twitter, News]                                  |
| > Arbitrum sequencer congestion causing            |
|   elevated gas [Discord] [unconfirmed]             |
| > Fed speakers lean hawkish ahead of NFP           |
|   [News, RSS]                                      |
|                                                    |
| -- Sentiment ------------------------------------ |
| Ethereum    -0.4  bearish  >>>>........            |
| Uniswap     +0.6  bullish  ......>>>>>>            |
| Arbitrum    -0.2  cautious >>>>.........            |
|                                                    |
| -- Sources: 847 items from 4 source types ------  |
|                                                    |
| View full report                                   |
+--------------------------------------------------+
| podders | today at 09:00 WIB                       |
+--------------------------------------------------+
```

### Field mapping

```typescript
function formatDiscordEmbed(report: MarketReport, publicUrl?: string) {
  const fields = [];

  // Key events (1 field, bulleted, stays under 1024 chars)
  fields.push({
    name: 'Key Events',
    value: report.keyEvents
      .slice(0, 5)
      .map(e => `> ${e}`)
      .join('\n'),
    inline: false
  });

  // Entity sentiment (1 field, compact table)
  if (report.entitySentiment?.length) {
    fields.push({
      name: 'Sentiment',
      value: report.entitySentiment
        .slice(0, 6)
        .map(e => {
          const bar = sentimentBar(e.sentiment);
          const label = e.sentiment > 0.2 ? 'bullish'
            : e.sentiment < -0.2 ? 'bearish' : 'neutral';
          const trend = e.trend === 'new' ? ' *new*' : '';
          return `**${e.name}** ${e.sentiment > 0 ? '+' : ''}${e.sentiment.toFixed(1)} ${label}${trend}`;
        })
        .join('\n'),
      inline: false
    });
  }

  // Source coverage (1 field, single line)
  fields.push({
    name: 'Coverage',
    value: `${report.itemCount} items from ${report.sourceCount} sources`,
    inline: false
  });

  return {
    title: `Daily Market Report \u2014 ${report.date}`,
    description: report.tldr,
    fields,
    url: publicUrl ? `${publicUrl}/reports/${report.id}` : undefined,
    color: 0x5B8DEF,
    timestamp: report.generatedAt,
    footer: { text: 'podders' }
  };
}
```

## Flash Report Embed

```
color: #FF6B35 (orange sidebar)

+--------------------------------------------------+
| [FLASH] Euler Finance Exploit -- $12M drained     |
|                                                    |
| Euler Finance lending pool drained via flash loan  |
| exploit. On-chain data shows funds moving to       |
| Tornado Cash. Team has paused contracts.           |
|                                                    |
| -- What Happened -------------------------------- |
| > Euler lending pool drained ~$12M via             |
|   reentrancy [Discord, Twitter]                    |
| > Team confirmed pause, audit underway [Twitter]   |
|                                                    |
| View full report                                   |
+--------------------------------------------------+
| podders FLASH | today at 14:32 WIB                 |
+--------------------------------------------------+
```

Differences from daily: orange color (`0xFF6B35`), `[FLASH]` title prefix, no sentiment table (too early for consensus), no coverage stats, 1-2 events max, TL;DR is 2 sentences not 3.

## Market Pulse Embed

```
color: #4A4A5A (muted grey sidebar)

+--------------------------------------------------+
| Market Pulse — 15:00 WIB                          |
|                                                    |
| BTC steady at $91K. Arbitrum DAO vote ongoing,     |
| no clear outcome yet. Light chatter across         |
| Discord — nothing actionable in this window.       |
|                                                    |
+--------------------------------------------------+
| podders pulse | today at 15:00 WIB                 |
+--------------------------------------------------+
```

Differences from daily: muted grey color (`0x4A4A5A`), no `[FLASH]` prefix, no sentiment table, no coverage stats. Title format is `"Market Pulse — HH:MM WIB"`. Output length scales with activity — quiet periods produce 1-2 sentences, busy periods produce a fuller summary. Uses Sonnet. Runs every 3 hours.

## Mobile Readability Rules

1. **TL;DR under 280 chars.** Discord mobile truncates description around line 4-5 on most phones. Three short sentences fit.
2. **No inline fields.** `inline: false` on everything. Inline fields render side-by-side on desktop but stack badly on narrow screens.
3. **Key events as blockquotes** (`> text`), not separate fields. One field with `>` lines renders cleanly on mobile; 5 separate fields create excessive vertical spacing.
4. **Entity sentiment: names bold, no ASCII bar charts.** Monospace bars break on mobile proportional fonts. Use `**ETH** -0.4 bearish` plaintext instead.
5. **Max 3-4 fields total.** Each field header adds ~30px of dead space on mobile. Fewer fields = tighter layout.
6. **No images or thumbnails.** Adds load time on mobile data, provides no information density for a text report.

## Action Links

- **"View full report"**: the embed `url` property makes the title a hyperlink. Requires `PUBLIC_URL` to be set.
- **Reaction feedback**: not in MVP. Discord webhooks cannot read reactions without a bot token listening. Defer to v2.1 alongside the bot token migration. When implemented: thumbs-up/down on report quality feeds back into prompt tuning.

## Language

**English only** for webhook output. The pipeline already normalizes Indonesian input to English summaries at Stage 1. Bilingual embeds would double char usage for a marginal audience benefit. Users reading Indonesian sources already expect English-language market briefs.

## Security

- **Always set `allowed_mentions: { parse: [] }`** in the webhook POST body. This suppresses `@everyone`, `@here`, and user pings that could survive from scraped content through LLM summarization into the embed.
- **Truncate** embed description to 4096 chars, field values to 1024 chars. Discord silently drops oversized embeds.
- Strip Discord markdown abuse (mass `||spoiler||` tags, `>>> blockquote` flooding) from TL;DR before embedding.
