# Podders v2 — Dashboard Wireframes & Component Spec

Dark theme. Georgia headings, Helvetica Neue body, monospace eyebrows.
Background `#0a0a0f`, cards `#111118`, text `#e2e2e8`.

---

## 1. Report View (`/`)

Latest report. TL;DR visible immediately. Progressive disclosure below.

### Desktop (1200px)

```
+------------------------------------------------------------------------+
| [P] Podders                          [Reports]  [Settings]     [icon]  |
+------------------------------------------------------------------------+
|                                                                        |
|  DAILY REPORT                                         March 31, 2026   |
|  ----------------------------------------------------------------      |
|                                                                        |
|  "Bitcoin reclaimed $92K on renewed ETF inflows while Ethereum         |
|   lagged amid L2 fee compression debates. Macro risk-off mood          |
|   persists after soft NFP print."                                      |
|                                                                        |
|  ----------------------------------------------------------------      |
|                                                                        |
|  KEY EVENTS                                                            |
|  +-----------------------------+  +-----------------------------+      |
|  | [!] ETF inflows $420M       |  | [~] Arbitrum DAO vote       |      |
|  | Largest single-day since    |  | Proposal to redirect        |      |
|  | Jan. BTC +3.2% on news.    |  | sequencer fees passes.      |      |
|  | Sources: 3  Sentiment: +0.7|  | Sources: 2  Sentiment: 0.0  |      |
|  +-----------------------------+  +-----------------------------+      |
|  +-----------------------------+  +-----------------------------+      |
|  | [!] NFP miss 142K vs 185K  |  | [ ] Solana TPS new ATH      |      |
|  | Risk assets dipped then    |  | 12,400 sustained TPS on     |      |
|  | recovered. Fed cut odds up.|  | mainnet. MEV discourse hot.  |      |
|  | Sources: 4  Sentiment: -0.3|  | Sources: 2  Sentiment: +0.4 |      |
|  +-----------------------------+  +-----------------------------+      |
|                                                                        |
|  ENTITY SENTIMENT                                         [expand v]   |
|  Bitcoin     ████████████████░░░░  +0.72                               |
|  Ethereum    ██████████░░░░░░░░░░  +0.18                               |
|  Solana      █████████████░░░░░░░  +0.44                               |
|  Arbitrum    ██████████░░░░░░░░░░   0.00                               |
|                                                                        |
|  SECTIONS                                                              |
|  [v] DeFi & L2s ------------------------------------------------      |
|      Summary paragraph about L2 fee dynamics, Arbitrum vote,           |
|      Base TVL growth...                                                |
|  [>] Macro & Tradfi --------------------------------------------      |
|  [>] NFTs & Culture --------------------------------------------      |
|  [>] Alpha & Signals -------------------------------------------      |
|                                                                        |
|  SOURCE BREAKDOWN              12 Discord | 8 Twitter | 3 RSS         |
|                                                                        |
|  [< Previous day: Mar 30]                    [Flash reports today: 0]  |
+------------------------------------------------------------------------+
```

### Mobile (375px)

```
+-----------------------------------+
| [P] Podders          [=] menu     |
+-----------------------------------+
|                                   |
| DAILY REPORT          Mar 31      |
| --------------------------------- |
|                                   |
| "Bitcoin reclaimed $92K on        |
|  renewed ETF inflows while        |
|  Ethereum lagged amid L2 fee      |
|  compression debates."            |
|                                   |
| --------------------------------- |
|                                   |
| KEY EVENTS                        |
| +---------------------------------+
| | [!] ETF inflows $420M          ||
| | Largest single-day since Jan.  ||
| | Sources: 3  Sent: +0.7        ||
| +---------------------------------+
| +---------------------------------+
| | [~] Arbitrum DAO vote          ||
| | Proposal to redirect fees...   ||
| | Sources: 2  Sent: 0.0         ||
| +---------------------------------+
| +---------------------------------+
| | [!] NFP miss 142K vs 185K     ||
| | Risk assets dipped...          ||
| +---------------------------------+
|                                   |
| [>] Entity Sentiment (4)         |
| [>] DeFi & L2s                   |
| [>] Macro & Tradfi               |
| [>] NFTs & Culture               |
| [>] Alpha & Signals              |
|                                   |
| [< Mar 30]          [Flash: 0]   |
+-----------------------------------+
```

**Behavior notes:**
- Mobile: all sections collapsed by default, only TL;DR + key events visible.
- Desktop: Entity Sentiment expanded, topic sections collapsed.
- `[!]` = urgency breaking/high, `[~]` = medium, `[ ]` = normal.
- Previous day link hits `GET /api/v1/reports?type=daily&limit=1&offset=1`.
- Flash badge shown if `type === 'flash'` with amber accent border.

---

## 2. Report List (`/reports`)

### Desktop (1200px)

```
+------------------------------------------------------------------------+
| [P] Podders                          [Reports]  [Settings]     [icon]  |
+------------------------------------------------------------------------+
|                                                                        |
|  REPORTS                                                               |
|                                                                        |
|  Filter: [All v]  [Daily]  [Flash]  [Pulse]                            |
|                                                                        |
|  +------------------------------------------------------------------+  |
|  | Mar 31, 2026   [DAILY]                                           |  |
|  | Bitcoin reclaimed $92K on renewed ETF inflows while Ethereum...   |  |
|  +------------------------------------------------------------------+  |
|  | Mar 31, 2026   [FLASH]                                  02:14 AM |  |
|  | BREAKING: Bybit hot wallet drained — $140M in ETH moved to...    |  |
|  +------------------------------------------------------------------+  |
|  | Mar 30, 2026   [DAILY]                                           |  |
|  | Quiet session. BTC range-bound $88-89K. Solana DEX volume...     |  |
|  +------------------------------------------------------------------+  |
|  | Mar 28, 2026   [DAILY]                                           |  |
|  | Fed minutes hawkish. Risk assets sold off. BTC -4.1% to $87K... |  |
|  +------------------------------------------------------------------+  |
|                                                                        |
|  [Load more]                                                           |
|                                                                        |
+------------------------------------------------------------------------+
```

### Mobile (375px)

```
+-----------------------------------+
| [P] Podders          [=] menu     |
+-----------------------------------+
| REPORTS                           |
| [All] [Daily] [Flash] [Pulse]    |
|                                   |
| Mar 31         [DAILY]            |
| Bitcoin reclaimed $92K on         |
| renewed ETF inflows while...      |
| --------------------------------- |
| Mar 31         [FLASH]   02:14   |
| BREAKING: Bybit hot wallet       |
| drained -- $140M in ETH...       |
| --------------------------------- |
| Mar 30         [DAILY]            |
| Quiet session. BTC range-bound   |
| $88-89K...                        |
| --------------------------------- |
|                                   |
| [Load more]                       |
+-----------------------------------+
```

**Behavior notes:**
- Filter pills: `All | Daily | Flash | Pulse`. Maps to `?type=` query param.
- Pagination via `limit=20&offset=N`. "Load more" appends.
- Click row to navigate to `/reports/:id` (renders in ReportView layout).
- DAILY badge = muted blue. FLASH badge = amber. PULSE badge = muted grey.
- TL;DR truncated to 2 lines with ellipsis.

---

## 3. Settings (`/settings`)

Three tabs: Sources, Delivery, Pipeline Health.

### Desktop (1200px)

```
+------------------------------------------------------------------------+
| [P] Podders                          [Reports]  [Settings]     [icon]  |
+------------------------------------------------------------------------+
|                                                                        |
|  SETTINGS                                                              |
|  [Sources]  [Delivery]  [Pipeline Health]                              |
|                                                                        |
+-- TAB: Sources ----------------------------------------------------+   |
|                                                                    |   |
|  + Add Source                                                      |   |
|                                                                    |   |
|  DISCORD (3)                                                       |   |
|  +--------------------------------------------------------------+  |   |
|  | #alpha-calls       Degen Hideout     [active]     [toggle] X |  |   |
|  | #market-chat       Degen Hideout     [active]     [toggle] X |  |   |
|  | #announcements     Solana FM         [backoff]    [toggle] X |  |   |
|  |                    Last error: 403 Forbidden, 2h ago         |  |   |
|  +--------------------------------------------------------------+  |   |
|                                                                    |   |
|  TWITTER (2)                                                       |   |
|  +--------------------------------------------------------------+  |   |
|  | @DefiIgnas                           [active]     [toggle] X |  |   |
|  | @MacroAlf                            [active]     [toggle] X |  |   |
|  +--------------------------------------------------------------+  |   |
|                                                                    |   |
|  RSS (1)                                                           |   |
|  +--------------------------------------------------------------+  |   |
|  | The Defiant         thedefiant.io    [disabled]   [toggle] X |  |   |
|  |                     5 errors in 1h — auto-disabled           |  |   |
|  +--------------------------------------------------------------+  |   |
+--------------------------------------------------------------------+   |
|                                                                        |
+------------------------------------------------------------------------+
```

### Add Source Flow — Discord Channel Picker

Triggered by `+ Add Source` > Discord. Calls `GET /api/v1/sources/discover/discord`.

```
+------------------------------------------+
|  Add Discord Channel                     |
|  ----------------------------------------|
|                                          |
|  Select server:                          |
|  +------------------------------------+  |
|  | Degen Hideout                   [v]|  |
|  +------------------------------------+  |
|  | Solana FM                          |  |
|  | Crypto Twitter Lounge              |  |
|  +------------------------------------+  |
|                                          |
|  Select channel:                         |
|  [ ] #general                            |
|  [x] #alpha-calls                        |
|  [ ] #market-chat                        |
|  [ ] #announcements                      |
|  [ ] #memes                              |
|                                          |
|  Label: [Alpha Calls - Degen Hideout  ]  |
|                                          |
|  [Test Connection]        [Add Channel]  |
|                                          |
|  -- or paste channel ID manually --      |
|  [                                    ]  |
+------------------------------------------+
```

### Add Source Flow — Twitter / RSS

```
+------------------------------------------+
|  Add Twitter Source                       |
|  ----------------------------------------|
|  Type: (@) Account  ( ) Search query     |
|  Handle: [@DefiIgnas                  ]  |
|  Label:  [DefiIgnas                   ]  |
|  [Test]                        [Add]     |
+------------------------------------------+

+------------------------------------------+
|  Add RSS Feed                            |
|  ----------------------------------------|
|  Feed URL: [https://thedefiant.io/feed]  |
|  Label:    [The Defiant               ]  |
|  [Validate Feed]               [Add]     |
+------------------------------------------+
```

### TAB: Delivery

```
+--------------------------------------------------------------------+
|                                                                    |
|  WEBHOOK                                                           |
|  URL: [https://discord.com/api/webhooks/1234.../abcd...       ]   |
|  [Test Webhook]                                       [Save]       |
|                                                                    |
|  DAILY DIGEST SCHEDULE                                             |
|  Time:     [09:00  v]                                              |
|  Timezone: [Asia/Jakarta      v]                                   |
|  Next delivery: Apr 1, 2026 at 09:00 WIB                          |
|                                                                    |
|  PUBLIC URL (for "View full report" links in webhook)              |
|  [https://podders.yourdomain.com                              ]    |
|  [Save]                                                            |
|                                                                    |
+--------------------------------------------------------------------+
```

### TAB: Pipeline Health

```
+--------------------------------------------------------------------+
|                                                                    |
|  PIPELINE STATUS                                                   |
|  +-------------------------------+------------------------------+  |
|  | Stage 1 (Summarize)          | Last run: 14 min ago  [ok]   |  |
|  | Stage 2 (Correlate)          | Last run: 14 min ago  [ok]   |  |
|  | Stage 3 (Synthesize)         | Last run: 9h ago      [ok]   |  |
|  | Delivery                     | Last run: 9h ago      [ok]   |  |
|  +-------------------------------+------------------------------+  |
|                                                                    |
|  LLM COST                                                          |
|  Today:      $0.38                                                 |
|  This month: $11.42                                                |
|  Model breakdown: Haiku $0.05 | Sonnet $0.33                      |
|                                                                    |
|  SOURCE HEALTH                                                     |
|  +--------------------------------------------------------------+  |
|  | Source          | Status   | Last Fetch   | Errors           |  |
|  |-----------------|----------|--------------|------------------|  |
|  | discord/#alpha  | active   | 2 min ago    | 0                |  |
|  | twitter/@Ignas  | active   | 1h ago       | 0                |  |
|  | rss/thedefiant  | disabled | 3d ago       | 5                |  |
|  +--------------------------------------------------------------+  |
|                                                                    |
|  FAILED DELIVERIES                                                 |
|  +--------------------------------------------------------------+  |
|  | Mar 28 Daily    | 401 Unauthorized        | [Retry]          |  |
|  +--------------------------------------------------------------+  |
|  -- or: "No failed deliveries"                                     |
|                                                                    |
|  API KEY                                                           |
|  [Rotate API Key]  (current key ends in ...x7f2)                   |
|                                                                    |
+--------------------------------------------------------------------+
```

### Mobile: Settings

Tabs render as full-width pill bar. Each tab content stacks vertically.
Source rows become cards. Add-source modals go full-screen on mobile.

---

## 4. Raw Discord Feed (`/feed`)

Live view of unprocessed Discord messages with image attachments. Backed by `GET /api/v1/feed` which returns raw items from Discord sources with `status: 'ready'` or `'processed'`, most recent first.

### Desktop (1200px)

```
+------------------------------------------------------------------------+
| [P] Podders                    [Reports]  [Feed]  [Settings]   [icon]  |
+------------------------------------------------------------------------+
|                                                                        |
|  RAW FEED                                                              |
|  Filter: [All Channels v]                              [Live] [Paused] |
|                                                                        |
|  +------------------------------------------------------------------+  |
|  | #alpha-calls  Degen Hideout                          2 min ago   |  |
|  | @trader_mike: ETH looking strong above 3.2k, watching for       |  |
|  | breakout. Chart attached.                                        |  |
|  | +--------------------+                                           |  |
|  | | [chart image]      |                                           |  |
|  | | (Discord CDN)      |                                           |  |
|  | +--------------------+                                           |  |
|  +------------------------------------------------------------------+  |
|  +------------------------------------------------------------------+  |
|  | #market-chat  Degen Hideout                          5 min ago   |  |
|  | @degen_sarah: Anyone see the OJK announcement? Looks like new    |  |
|  | crypto regulations incoming for ID market.                       |  |
|  +------------------------------------------------------------------+  |
|  +------------------------------------------------------------------+  |
|  | #announcements  Solana FM                            12 min ago  |  |
|  | @solana_fm_bot: New validator set update — 1,847 active.         |  |
|  | +--------------------+  +--------------------+                   |  |
|  | | [infographic 1]    |  | [infographic 2]    |                   |  |
|  | +--------------------+  +--------------------+                   |  |
|  +------------------------------------------------------------------+  |
|                                                                        |
|  [Load more]                                                           |
+------------------------------------------------------------------------+
```

### Mobile (375px)

```
+-----------------------------------+
| [P] Podders          [=] menu     |
+-----------------------------------+
| RAW FEED              [Live]      |
| [All Channels v]                  |
|                                   |
| #alpha-calls           2 min ago  |
| @trader_mike: ETH looking         |
| strong above 3.2k, watching...    |
| +-------------------------------+ |
| | [chart image]                 | |
| +-------------------------------+ |
| --------------------------------- |
| #market-chat           5 min ago  |
| @degen_sarah: Anyone see the      |
| OJK announcement?...              |
| --------------------------------- |
|                                   |
| [Load more]                       |
+-----------------------------------+
```

**Behavior notes:**
- Channel filter dropdown populated from active Discord sources.
- Image attachments rendered inline from Discord CDN URLs stored in `metadata.imageUrls`.
- Images lazy-loaded, max 4 per message, click to expand.
- "Live" mode auto-polls every 10s for new messages. "Paused" stops polling.
- Pagination via `limit=50&offset=N`. "Load more" appends.
- Messages link back to Discord via the `url` field (opens in Discord app/web).
- No LLM processing shown here -- this is raw unprocessed content.

---

## 5. Chat (`/chat`)

RAG chat interface. User asks questions about their data, Sonnet answers grounded in summaries, reports, and items retrieved via semantic search.

### Desktop (1200px)

```
+------------------------------------------------------------------------+
| [P] Podders                    [Reports]  [Feed]  [Chat]  [Settings]   |
+------------------------------------------------------------------------+
|                                                                        |
|  CHAT                                                    [New Chat]    |
|                                                                        |
|  +------------------------------------------------------------------+  |
|  |                                                                  |  |
|  |  You                                                  10:32 AM   |  |
|  |  What happened with SOL this week?                               |  |
|  |                                                                  |  |
|  |  ----------------------------------------------------------------|  |
|  |                                                                  |  |
|  |  Podders                                              10:32 AM   |  |
|  |  Solana saw elevated activity this week. TPS hit a new ATH of    |  |
|  |  12,400 sustained on mainnet, driving bullish sentiment across   |  |
|  |  monitored channels. MEV discourse was hot in alpha groups.      |  |
|  |  Sentiment averaged +0.44 across 3 sources.                     |  |
|  |                                                                  |  |
|  |  Sources:                                                        |  |
|  |  [Daily Report Mar 31] [Summary #alpha-calls] [Summary #market]  |  |
|  |                                                                  |  |
|  |  ----------------------------------------------------------------|  |
|  |                                                                  |  |
|  |  You                                                  10:33 AM   |  |
|  |  How does that compare to last week?                             |  |
|  |                                                                  |  |
|  |  ----------------------------------------------------------------|  |
|  |                                                                  |  |
|  |  Podders                                              10:33 AM   |  |
|  |  Last week SOL sentiment was +0.18 with 2 source mentions.       |  |
|  |  This week's +0.44 across 3 sources represents a significant    |  |
|  |  uptick driven primarily by the TPS milestone.                   |  |
|  |                                                                  |  |
|  |  Sources:                                                        |  |
|  |  [Daily Report Mar 24] [Daily Report Mar 31]                     |  |
|  |                                                                  |  |
|  +------------------------------------------------------------------+  |
|                                                                        |
|  +------------------------------------------------------------------+  |
|  | [Ask about your data...                                 ] [Send] |  |
|  +------------------------------------------------------------------+  |
|                                                                        |
+------------------------------------------------------------------------+
```

### Mobile (375px)

```
+-----------------------------------+
| [P] Podders          [=] menu     |
+-----------------------------------+
| CHAT                 [New Chat]   |
|                                   |
| You                    10:32 AM   |
| What happened with SOL            |
| this week?                        |
| --------------------------------- |
| Podders                10:32 AM   |
| Solana saw elevated activity      |
| this week. TPS hit a new ATH...   |
|                                   |
| Sources:                          |
| [Daily Report Mar 31]             |
| [Summary #alpha-calls]            |
| --------------------------------- |
| You                    10:33 AM   |
| How does that compare to          |
| last week?                        |
| --------------------------------- |
| Podders                10:33 AM   |
| Last week SOL sentiment was       |
| +0.18 with 2 source mentions...  |
|                                   |
| Sources:                          |
| [Daily Report Mar 24]             |
| [Daily Report Mar 31]             |
| --------------------------------- |
|                                   |
| [Ask about your data...   ] [>]  |
+-----------------------------------+
```

**Behavior notes:**
- "New Chat" clears conversation history and starts a fresh `conversationId`.
- Each answer displays source citations as clickable pills. Clicking a source navigates to the relevant report (`/reports/:id`) or summary.
- Source citations come from the API response `sources[]` array — each has `type` (report/summary/item), `id`, and `snippet`.
- Conversation history maintained via `conversationId`. Server keeps last 5 messages in memory for follow-up context.
- Input field auto-focuses on page load. Enter key sends, Shift+Enter for newline.
- Loading state: animated dots ("...") in a Podders message bubble while waiting for response.
- Empty state: "Ask a question about your market data. Podders will search your summaries, reports, and items to find the answer."

---

## 6. Empty States

### No Reports (`/`)

```
+------------------------------------------------------------------------+
|                                                                        |
|            .-~~~-.                                                      |
|           /       \       Welcome to Podders                           |
|          |  o   o  |                                                   |
|           \  ___  /      Add your first source in Settings to start    |
|            '-----'       generating market reports.                    |
|                                                                        |
|                          [Go to Settings ->]                           |
|                                                                        |
+------------------------------------------------------------------------+
```

### No Reports (`/reports`)

```
+------------------------------------------------------------------------+
|                                                                        |
|  REPORTS                                                               |
|  [All] [Daily] [Flash]                                                 |
|                                                                        |
|          No reports yet.                                               |
|                                                                        |
|          Your first report will generate after sources are             |
|          configured and the daily digest runs.                         |
|                                                                        |
|          Next scheduled digest: Apr 1, 09:00 WIB                      |
|                                                                        |
+------------------------------------------------------------------------+
```

### No Sources (`/settings` Sources tab)

```
+--------------------------------------------------------------------+
|                                                                    |
|  Add your first source to start collecting market intelligence.    |
|                                                                    |
|  +------------------+  +------------------+  +------------------+  |
|  |   [Discord]      |  |   [Twitter]      |  |   [RSS]          |  |
|  |   Channels from  |  |   Track accounts |  |   News feeds     |  |
|  |   your servers   |  |   and searches   |  |   and blogs      |  |
|  |                  |  |                  |  |                  |  |
|  |   [+ Add]        |  |   [+ Add]        |  |   [+ Add]        |  |
|  +------------------+  +------------------+  +------------------+  |
|                                                                    |
+--------------------------------------------------------------------+
```

### All Sources Disabled

```
+--------------------------------------------------------------------+
|                                                                    |
|  [!] All sources are currently disabled.                           |
|                                                                    |
|  The pipeline cannot collect data. Re-enable at least one source   |
|  or check Pipeline Health for error details.                       |
|                                                                    |
|  DISCORD (2 disabled)                                              |
|  +--------------------------------------------------------------+  |
|  | #alpha-calls     [disabled]  5 errors    [Re-enable] [X]     |  |
|  | #market-chat     [disabled]  5 errors    [Re-enable] [X]     |  |
|  +--------------------------------------------------------------+  |
|                                                                    |
+--------------------------------------------------------------------+
```

---

## 7. Component Inventory

| Component | Description |
|-----------|-------------|
| `Header` | App name, nav links (Reports, Feed, Chat, Settings), hamburger on mobile. Fixed top. |
| `ReportCard` | Used in `/reports` list. Shows date, type badge, TL;DR preview (2-line truncated). Click navigates to full report. |
| `ReportHeader` | Date, type badge (DAILY/FLASH), source count. Used at top of ReportView. |
| `TldrBlock` | Large Georgia serif quote block. The hero element of every report. |
| `KeyEventCard` | Card with urgency icon, title, description, source count, sentiment mini-bar. 2-column grid on desktop, stacked on mobile. |
| `SentimentBar` | Horizontal bar from -1.0 to +1.0. Red left, green right, neutral center. Entity name + numeric value. |
| `SectionAccordion` | Collapsible section with chevron toggle. Title + item count. Body is markdown-rendered text. |
| `EntityBadge` | Pill with entity name + type icon (token/person/project/company/event). Used inline in report text. |
| `SourceRow` | Row in sources list: label, parent (guild/domain), status badge, toggle switch, delete button. Shows error detail when status is backoff/disabled. |
| `SourceCard` | Card variant of SourceRow for mobile and empty-state add flows. |
| `StatusBadge` | Colored pill: `active` = green, `backoff` = amber, `disabled` = red. |
| `TypeBadge` | Report type pill: `DAILY` = muted blue, `FLASH` = amber, `PULSE` = muted grey. |
| `EmptyState` | Centered illustration slot + heading + description + optional CTA button. Reused across all empty states. |
| `FilterPills` | Horizontal pill group for filtering. Active pill filled, others outlined. |
| `Modal` | Overlay dialog for add-source flows. Full-screen on mobile, centered 480px on desktop. |
| `ChannelPicker` | Discord-specific: server dropdown + channel checkbox list. Populated from discover endpoint. Falls back to manual ID input. |
| `ConfigField` | Label + input + save button pattern. Used in Delivery tab. Inline validation. |
| `PipelineStatusRow` | Stage name + last run time + status indicator (green dot / amber / red). |
| `CostSummary` | Today/month cost with model breakdown. Reads from `/api/v1/status`. |
| `FailedDeliveryRow` | Report date + error message + retry button. |
| `FeedItem` | Raw Discord message card: channel name, guild, author, content, timestamp, optional image gallery. Used in `/feed`. |
| `FeedImageGallery` | Grid of Discord CDN images from `metadata.imageUrls`. Max 4 visible, click to expand. Lazy-loaded. |
| `LiveToggle` | Toggle between "Live" (auto-poll) and "Paused" modes for the raw feed view. |
| `ChatPanel` | Full chat view: scrollable message list + input bar. Manages `conversationId` state and API calls to `POST /api/v1/chat`. |
| `ChatMessage` | Single message bubble. Renders user messages (right-aligned, muted surface) or Podders responses (left-aligned, surface card with source citations). Timestamp in text-secondary. |
| `SourceCitation` | Clickable pill showing source type icon + truncated snippet. Links to `/reports/:id` for reports, or displays summary/item detail. Accent blue text, surface-raised background. |
| `LoadMoreButton` | Pagination trigger. Shows "Load more" with count remaining. |

---

## 8. Color & Theme

Dark theme. All colors specified as hex + Tailwind class.

### Palette

| Role | Hex | Tailwind | Usage |
|------|-----|----------|-------|
| Background | `#0a0a0f` | `bg-[#0a0a0f]` | Page background |
| Surface | `#111118` | `bg-[#111118]` | Cards, modals, dropdowns |
| Surface raised | `#1a1a24` | `bg-[#1a1a24]` | Hover states, active tabs |
| Border | `#2a2a3a` | `border-[#2a2a3a]` | Card borders, dividers |
| Border hover | `#3a3a4f` | `border-[#3a3a4f]` | Card hover border shift |
| Text primary | `#e2e2e8` | `text-[#e2e2e8]` | Body text, headings |
| Text secondary | `#8888a0` | `text-[#8888a0]` | Labels, timestamps, meta |
| Text muted | `#555566` | `text-[#555566]` | Disabled text, placeholders |
| Accent blue | `#6688cc` | `text-[#6688cc]` | Links, active nav, daily badge bg |
| Accent green | `#44bb77` | `text-[#44bb77]` | Positive sentiment, active status |
| Accent red | `#cc4455` | `text-[#cc4455]` | Negative sentiment, errors, disabled status |
| Accent amber | `#ccaa44` | `text-[#ccaa44]` | Flash badge, backoff status, warnings |
| Accent neutral | `#888899` | `text-[#888899]` | Neutral sentiment (0.0) |

### Typography

| Element | Font | Tailwind |
|---------|------|----------|
| Headings (h1-h3) | Georgia, serif | `font-serif` |
| Body text | Helvetica Neue, sans-serif | `font-sans` |
| Eyebrows / labels | SF Mono, monospace | `font-mono text-xs uppercase tracking-wider` |
| TL;DR quote | Georgia, 1.25rem | `font-serif text-xl leading-relaxed` |

### Card Hover

```css
transition: transform 160ms ease, box-shadow 160ms ease, border-color 160ms ease;
/* hover: */
transform: translateY(-1px);
box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
border-color: #3a3a4f;
```

Tailwind: `transition-all duration-[160ms] hover:-translate-y-px hover:shadow-lg hover:border-[#3a3a4f]`

### Sentiment Bar Colors

Gradient mapped to value:
- `-1.0` to `-0.3`: accent red
- `-0.3` to `+0.3`: accent neutral
- `+0.3` to `+1.0`: accent green

Bar fills from center (0) outward in the sentiment direction.
