# Discord Gateway Subsystem

Reference spec for the Discord Gateway WebSocket integration in Podders v2.
Covers user-token connections, multi-token management, and message ingestion.

## 1. Gateway Connection Lifecycle

### Connection URL

Initial connect: `wss://gateway.discord.gg/?v=10&encoding=json`

After first session, use the `resume_gateway_url` from the READY event for reconnects.

### Opcodes

| Opcode | Name | Direction | Purpose |
|--------|------|-----------|---------|
| 0 | DISPATCH | Receive | Event payload (has `t` and `s` fields) |
| 1 | HEARTBEAT | Send/Receive | Keep-alive ping. Server may request one via opcode 1 |
| 2 | IDENTIFY | Send | Authenticate after connecting |
| 6 | RESUME | Send | Resume a dropped session |
| 7 | RECONNECT | Receive | Server requests client reconnect |
| 9 | INVALID SESSION | Receive | Session invalid. `d: true` = resumable, `d: false` = re-IDENTIFY |
| 10 | HELLO | Receive | Contains `heartbeat_interval` |
| 11 | HEARTBEAT ACK | Receive | Server acknowledges heartbeat |

### Sequence

```
Client                          Gateway
  |--- wss connect ----------------->|
  |<------------- opcode 10 HELLO ---|  (contains heartbeat_interval)
  |--- opcode 2 IDENTIFY ----------->|
  |<--- opcode 0 DISPATCH t=READY ---|  (session_id, resume_gateway_url, guilds)
  |--- opcode 1 HEARTBEAT --------->|   (every heartbeat_interval ms, jittered)
  |<--- opcode 11 HEARTBEAT ACK ----|
  |<--- opcode 0 DISPATCH t=* ------|  (MESSAGE_CREATE, GUILD_CREATE, etc.)
```

### IDENTIFY Payload (User Token)

```json
{
  "op": 2,
  "d": {
    "token": "user-token-here",
    "capabilities": 16381,
    "properties": {
      "os": "Linux",
      "browser": "Chrome",
      "device": "",
      "system_locale": "en-US",
      "browser_user_agent": "Mozilla/5.0 ... Chrome/124.0.0.0 ...",
      "browser_version": "124.0.0.0",
      "os_version": "",
      "referrer": "", "referring_domain": "",
      "referrer_current": "", "referring_domain_current": "",
      "release_channel": "stable",
      "client_build_number": 291963
    },
    "presence": { "status": "online", "since": 0, "activities": [], "afk": false },
    "compress": false,
    "client_state": {
      "guild_versions": {},
      "highest_last_message_id": "0",
      "read_state_version": 0,
      "user_guild_settings_version": -1,
      "user_settings_version": -1,
      "private_channels_version": "0",
      "api_code_version": 0
    }
  }
}
```

Key differences from bot IDENTIFY:
- No `intents` field. User tokens receive all events by default.
- `properties` mimics a real browser client (not `Bot` library info).
- `capabilities`: bitmask for client feature support. `16381` is a safe default.
- `presence` included inline (bots set presence via opcode 3).
- `client_state` required for server to send guild data correctly.
- `compress: false` -- we skip zlib transport compression.

### Heartbeat

On receiving HELLO (opcode 10):
1. Extract `d.heartbeat_interval` (typically 41250ms).
2. Send first heartbeat after `heartbeat_interval * Math.random()` (jitter).
3. Then send heartbeat every `heartbeat_interval` ms.
4. Each heartbeat: `{ "op": 1, "d": <last_sequence_number> }`.
5. If server sends opcode 1 (heartbeat request), respond immediately.
6. If no HEARTBEAT ACK (opcode 11) received before next heartbeat is due, the connection is zombied. Close and reconnect.

### Resume

When the connection drops (network error, opcode 7 RECONNECT, or close code 4000-4003):

1. Connect to `resume_gateway_url` (not the default gateway URL).
2. After receiving HELLO, send RESUME instead of IDENTIFY:

```json
{
  "op": 6,
  "d": {
    "token": "user-token-here",
    "session_id": "stored-session-id",
    "seq": 12345
  }
}
```

3. Server replays missed events, then sends a RESUMED dispatch.
4. If server responds with INVALID SESSION (`op: 9, d: false`), wait 1-5s random then do a fresh IDENTIFY on the default gateway URL.

### Reconnect Logic

```
on_close(code):
  if code in [4004, 4010, 4014]:
    # Fatal -- do not reconnect
    disable_token()
    return
  if code in [4000, 4001, 4002, 4003, 4009]:
    # Resumable -- attempt session resume
    attempt_resume()
  if code in [4007, 4008]:
    # Not resumable -- fresh IDENTIFY required
  if resume_fails or code not resumable:
    fresh_identify(backoff)

backoff: min(2^attempt * 2s, 60s)
max_attempts: unlimited (but log WARN after 5 consecutive failures)
```

## 2. Multi-Token Management

### Token Pool

Tokens arrive from `DISCORD_TOKENS` env var, comma-separated. On startup:

```typescript
interface TokenState {
  token: string
  index: number
  status: 'idle' | 'connecting' | 'connected' | 'backoff' | 'disabled'
  ws: WebSocket | null
  sessionId: string | null
  resumeUrl: string | null
  lastSeq: number | null
  assignedChannels: Set<string>  // channel IDs
  errorCount: number
  nextRetryAt: number | null
}
```

### Startup Sequence

1. Load all tokens into the pool with status `idle`.
2. Query `sources` table for all enabled Discord channel IDs.
3. Partition channels across non-disabled tokens round-robin by token index.
4. Connect tokens sequentially with **5s gap** between each IDENTIFY.

### Channel Partitioning

Round-robin: `active[i % active.length].assignedChannels.add(channels[i])`.

Partitioning only matters for filtering. Every connection receives MESSAGE_CREATE for all guilds the user is in. We filter client-side by `channel_id` membership in `assignedChannels`. This prevents duplicate ingestion when multiple tokens share guild membership.

### Token Death and Reassignment

When a token is disabled (close code 4004, or error_count >= 5):

1. Set `token.status = 'disabled'`.
2. Collect its `assignedChannels`.
3. Re-run `partitionChannels()` across remaining active tokens. No reconnection needed -- the surviving connections already receive events for those channels (assuming shared guild membership).
4. Update `source_state` for reassigned channels.
5. Log CRITICAL with the disabled token index (never log the token itself).

If all tokens are disabled, log CRITICAL and set a flag surfaced on `/settings`. Discord ingestion halts.

## 3. MESSAGE_CREATE Event Handling

### Event Shape

Key fields from `d`: `id` (snowflake), `channel_id`, `guild_id`, `author` (`{id, username, bot}`), `content`, `timestamp` (ISO 8601), `embeds[]`, `attachments[]`, `referenced_message` (nullable), `type` (0=DEFAULT, 19=REPLY).

### Mapping to RawItem

```typescript
function mapToRawItem(event: MessageCreate): RawItem | null {
  const d = event.d

  // Filter: only process messages from assigned channels
  if (!token.assignedChannels.has(d.channel_id)) return null

  // Filter: skip bot messages
  if (d.author.bot) return null

  // Filter: skip non-default message types (joins, pins, boosts, etc.)
  // Type 0 = DEFAULT, Type 19 = REPLY -- both contain user content
  if (d.type !== 0 && d.type !== 19) return null

  return {
    id: ulid(),
    source: 'discord',
    sourceId: d.channel_id,
    author: d.author.username,
    content: buildContent(d),
    timestamp: new Date(d.timestamp),
    url: `https://discord.com/channels/${d.guild_id}/${d.channel_id}/${d.id}`,
    engagement: 0,  // Discord has no engagement signal at creation time
    metadata: {
      guildId: d.guild_id,
      authorId: d.author.id,
      messageId: d.id,
      hasEmbeds: d.embeds.length > 0,
      hasAttachments: d.attachments.length > 0,
      isReply: d.type === 19,
      replyToId: d.referenced_message?.id ?? null,
      // Store Discord CDN URLs for image attachments (used by raw feed view)
      imageUrls: d.attachments
        .filter((a: any) => a.content_type?.startsWith('image/'))
        .map((a: any) => a.url),
    }
  }
}
```

### Content Construction

`buildContent(d)` concatenates with `\n`:
1. `d.content` (main text)
2. Reply context: `[replying to {author}: {content.slice(0, 200)}]` from `d.referenced_message`
3. Embed signal: `embed.description` (or `embed.title` if no description). Many alpha channels use bot embeds for structured data.
4. Attachment markers: `[attachment: {filename}]` -- filenames only, no binary fetch.

Image attachment URLs (Discord CDN) are stored in `metadata.imageUrls` for display in the raw feed dashboard view. These are not included in `content` sent to the LLM -- they are visual-only data surfaced to the user.

**Engagement**: hardcoded to 0. Discord does not include reaction counts in MESSAGE_CREATE. We skip MESSAGE_REACTION_ADD -- not worth the complexity for MVP.

## 4. Rate Limits

### IDENTIFY Rate Limit

- **Limit**: 1 IDENTIFY per 5 seconds per IP.
- **Enforcement**: server-side. Sending IDENTIFY too fast results in close code 4009.
- **Our handling**: sequential token connection with 5s sleep between each (see section 2).

### Gateway Send Rate Limit

- **Limit**: 120 events per 60 seconds per connection.
- **Counted events**: anything the client sends (HEARTBEAT, IDENTIFY, RESUME, status updates).
- **Breach penalty**: close code 4008 (rate limited).
- **Our exposure**: minimal. We only send heartbeats (~1.5/min) and the initial IDENTIFY. No status updates, no voice state, no request guild members. Typical send rate: 2-3 events/min.
- **Safety**: if we ever add features that send more (e.g., requesting guild members), implement a token bucket: 120 tokens, refill 2/sec, drain 1 per send.

### HTTP API Rate Limits (Not Used)

We do not call the Discord HTTP API. All data comes through the Gateway. No REST rate limits apply.

## 5. User Token Specifics

### User vs Bot: Key Differences

| Aspect | User Token | Bot Token |
|--------|-----------|-----------|
| IDENTIFY payload | `properties` (browser info), `capabilities`, `client_state`, `presence` | `intents` bitmask, `shard` array |
| Intents | Not applicable. Receives all events. | Must declare intents. Privileged intents (message content, presence, members) require approval. |
| Events received | Everything: MESSAGE_CREATE, GUILD_CREATE, TYPING_START, PRESENCE_UPDATE, CHANNEL_UPDATE, MESSAGE_UPDATE, RELATIONSHIP_*, SESSIONS_REPLACE, etc. | Only events matching declared intents. |
| Guild member list | Received lazily via GUILD_MEMBER_LIST_UPDATE (supplemental) | Requires REQUEST_GUILD_MEMBERS opcode |
| Token format | Starts with base64(user_id), then `.`, then HMAC. ~70 chars | `Bot ` prefix in HTTP, but raw in Gateway. Starts with base64(bot_id). |
| Rate limits | Same Gateway limits (120/60s send, 1 IDENTIFY/5s) | Same, but bots get higher concurrency in large guilds |
| Close codes | Same set, but 4014 (disallowed intents) does not apply | 4014 if privileged intents not approved |
| TOS status | Violates TOS (self-bot) | Compliant |

### Events We Process

| Event | Action |
|-------|--------|
| `READY` | Store `session_id`, `resume_gateway_url`, extract guild list |
| `RESUMED` | Log, continue processing |
| `GUILD_CREATE` | Cache guild/channel metadata for discovery endpoint |
| `MESSAGE_CREATE` | Map to RawItem, pass to normalize pipeline |

### Events We Ignore

Everything else: `TYPING_START`, `PRESENCE_UPDATE`, `MESSAGE_UPDATE`, `MESSAGE_DELETE`, `MESSAGE_REACTION_ADD`, `VOICE_STATE_UPDATE`, `CHANNEL_UPDATE`, `GUILD_MEMBER_LIST_UPDATE`, `SESSIONS_REPLACE`, `USER_SETTINGS_UPDATE`, etc. Switch on `event.t` in the dispatch handler, silently drop unmatched events.

## 6. Token Rotation on Failure

### Close Code Reference

| Code | Name | Resumable | Action |
|------|------|-----------|--------|
| 4000 | Unknown error | Yes | Resume, then fresh IDENTIFY |
| 4001 | Unknown opcode | Yes | Resume (likely our bug -- log ERROR) |
| 4002 | Decode error | Yes | Resume (likely our bug -- log ERROR) |
| 4003 | Not authenticated | Yes | Resume (race condition on connect) |
| 4004 | Authentication failed | **No** | **Disable token immediately**. Token is invalid or revoked. |
| 4005 | Already authenticated | Yes | Resume (sent IDENTIFY twice -- our bug) |
| 4007 | Invalid seq | No | Fresh IDENTIFY (cannot resume, seq was invalid) |
| 4008 | Rate limited | No | Fresh IDENTIFY after 60s backoff |
| 4009 | Session timed out | Yes | Resume with stored session |
| 4010 | Invalid shard | **No** | **Disable token**. Should not occur (we do not shard). |
| 4011 | Sharding required | **No** | **Disable token**. Account is in too many guilds (>2500). |
| 4012 | Invalid API version | **No** | **Disable token**. Our gateway version is outdated. Log CRITICAL. |
| 4013 | Invalid intents | **No** | Should not occur for user tokens. Disable + log. |
| 4014 | Disallowed intents | **No** | Should not occur for user tokens. Disable + log. |
| 1000 | Normal closure | No | Fresh IDENTIFY (server-initiated clean close) |
| 1001 | Going away | Yes | Resume (server restarting) |
| null | Network drop | Yes | Resume with exponential backoff |

### Disable vs Retry Decision

```
on_close(code):
  FATAL_CODES = [4004, 4010, 4011, 4012, 4013, 4014]
  NO_RESUME_CODES = [4007, 4008, 1000]

  if code in FATAL_CODES:
    token.status = 'disabled'
    token.errorCount = 999
    reassignChannels(token)
    log.critical({ tokenIndex: token.index, code }, 'Token permanently disabled')
    updateSourceState('discord', token.assignedChannels, 'disabled')
    return

  token.errorCount++

  if token.errorCount >= 5 within 1 hour:
    token.status = 'disabled'
    reassignChannels(token)
    log.critical({ tokenIndex: token.index }, 'Token disabled after repeated failures')
    return

  if code in NO_RESUME_CODES:
    freshIdentify(token, backoff)
  else:
    attemptResume(token)
```

### Error Count Reset

Reset `token.errorCount = 0` after a successful READY or RESUMED event. This prevents transient network issues from accumulating toward the disable threshold.

## 7. Channel Discovery

### Purpose

The `GET /api/v1/sources/discover/discord` endpoint returns guilds and channels visible to the connected tokens. Used by the Settings UI channel picker.

### Data Source: GUILD_CREATE Events

On initial connection, after READY, Discord sends a GUILD_CREATE event for every guild the user account is a member of. These are "lazy guilds" for user tokens -- they include guild metadata and channel lists but not full member lists.

### Cache Structure

In-memory `Map<guildId, { id, name, icon, channels: Map<channelId, { id, name, type, parentId, position }> }>`. Maintained per-token, merged and deduped by guild ID for the API response.

### GUILD_CREATE Handler

Extract `data.id`, `data.properties.name`, `data.properties.icon`. Filter `data.channels` to text-like types only:

| Type | Name | Include |
|------|------|---------|
| 0 | GUILD_TEXT | Yes |
| 5 | GUILD_ANNOUNCEMENT | Yes |
| 15 | GUILD_FORUM | Yes |
| 2 | GUILD_VOICE | No |
| 4 | GUILD_CATEGORY | No |
| 13 | GUILD_STAGE_VOICE | No |

### Discovery API Response

```json
{
  "guilds": [
    {
      "id": "1111111111111111111",
      "name": "DeFi Alpha",
      "icon": "abc123",
      "channels": [
        { "id": "2222222222222222222", "name": "general", "type": 0 },
        { "id": "3333333333333333333", "name": "alpha-calls", "type": 0 },
        { "id": "4444444444444444444", "name": "announcements", "type": 5 }
      ]
    }
  ]
}
```

### Cache Staleness

The guild cache is populated on connection and updated on subsequent GUILD_CREATE/GUILD_DELETE events. It is not persisted to the database -- it lives in memory and rebuilds on restart. If all tokens are disconnected, the discovery endpoint returns an empty guild list.

### Note on User Token GUILD_CREATE Shape

User tokens receive a different GUILD_CREATE shape than bots. Key differences:
- Guild name and icon are nested under `properties` (not top-level `name`/`icon`).
- Channels array is top-level `channels` (same as bots).
- Member list is omitted (lazy guilds).
- `data.properties.name` not `data.name`.
