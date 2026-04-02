# Error Handling Reference

Implementation reference for all error handling in Podders v2. Consolidates specs from ARCHITECTURE.md and PIPELINE.md.

## Error Taxonomy

### 1. LLM Errors (llm.ts)

| Code | Error | Trigger | Response | Retry | Logged As | User-Visible |
|------|-------|---------|----------|-------|-----------|--------------|
| L1 | Context length exceeded | 400, input exceeds model limit | Split chunk in half, retry both halves | Yes (split) | warn | No |
| L2 | Malformed request | 400, bad request body | Skip batch, alert | No | error | Yes — "LLM request error" on /settings |
| L3 | Invalid API key | 401 | Halt ALL LLM jobs immediately | No | fatal | Yes — "API key invalid" banner on /settings |
| L4 | Rate limited | 429 | Queue retry respecting `retry-after` header | Yes | warn | No (transient) |
| L5 | Overloaded | 529 | Backoff 30s, retry | Yes, 3x max | warn | No unless all 3 fail, then error on /settings |
| L6 | Server error | 500/502/503 | Exponential backoff: 2s, 8s, 32s | Yes, 3x | warn per attempt, error on final | No unless all 3 fail |
| L7 | Invalid JSON response | Valid HTTP, unparseable JSON | Retry once with correction prompt | Yes, 1x | warn | No unless retry also fails |
| L8 | Refusal | Empty content / refusal | Skip batch | No | error | Yes — "LLM refused to process batch" on /settings |

### 2. Scraper Errors (ingest/)

| Code | Error | Trigger | Response | Retry | Logged As | User-Visible |
|------|-------|---------|----------|-------|-----------|--------------|
| S1 | Network timeout | Connection/read timeout | Increment error_count, set backoff | Yes (via backoff schedule) | warn | Yes if disabled (error_count >= 5) |
| S2 | DNS resolution failure | Host unreachable | Same as S1 | Yes (via backoff) | warn | Same |
| S3 | Auth failure (Discord) | 401/403 or close code 4004 | Immediately disable token's sources | No | fatal | Yes — source shows "disabled" with reason |
| S4 | Rate limited (twitterapi.io) | 429 from twitterapi.io API | Backoff per standard schedule | Yes | warn | No unless disabled |
| S5 | Schema change (twitterapi.io) | Zod validation fails on response | Fail fast, skip batch, do not produce empty content | No | error | Yes — "Source schema changed" on /settings |
| S6 | Request timeout (twitterapi.io) | 15s request timeout exceeded | Return error, does not block cron mutex | No (next cron cycle) | error | Yes if repeated |
| S7 | Discord reconnect | Close code 4009 | Resume with session resume URL | Yes (automatic) | info | No |
| S8 | Discord network drop | WebSocket connection lost | Reconnect with exponential backoff, cap 60s | Yes | warn | No unless prolonged |
| S9 | RSS fetch failure | HTTP error or parse failure | Increment error_count, standard backoff | Yes | warn | Yes if disabled |
| S10 | SSRF blocked | URL resolves to private IP | Reject immediately | No | warn | Yes — "URL blocked (private IP)" |

### 3. Webhook Errors (deliver/webhook.ts)

| Code | Error | Trigger | Response | Retry | Logged As | User-Visible |
|------|-------|---------|----------|-------|-----------|--------------|
| W1 | Rate limited | HTTP 429 | Backoff: 2s, 8s, 32s | Yes, 3x | warn | Yes if all retries fail |
| W2 | Server error | HTTP 5xx | Backoff: 2s, 8s, 32s | Yes, 3x | warn per attempt | Yes if all retries fail |
| W3 | Network timeout | 10s timeout exceeded | Backoff: 2s, 8s, 32s | Yes, 3x | warn | Yes if all retries fail |
| W4 | Client error | HTTP 4xx (not 429) | Permanent failure, no retry | No | error | Yes — "Webhook delivery failed" on /settings |
| W5 | Invalid webhook URL | URL validation fails | Reject at config time | No | warn | Yes — validation error on save |

On final failure: `reports.delivery_status = 'failed'`, full context logged. Failed deliveries surface on /settings.

### 4. Database Errors (db/)

| Code | Error | Trigger | Response | Retry | Logged As | User-Visible |
|------|-------|---------|----------|-------|-----------|--------------|
| D1 | Migration failure | Schema migration throws | Halt startup | No | fatal | Process won't start |
| D2 | Write contention | Transaction exceeds lock timeout | Retry with short delay | Yes, 1x | warn | No |
| D3 | Disk full | Postgres out of disk space | Halt writes, log | No | fatal | Yes — pipeline stops producing reports |
| D4 | Corruption | Integrity check fails | Log, attempt backup restore | No | fatal | Yes — requires manual intervention |
| D5 | FTS index stale | tsvector index out of sync after failed migration | Log, search degraded but pipeline continues | No | error | Search returns incomplete results |

### 5. Config Errors (config.ts)

| Code | Error | Trigger | Response | Retry | Logged As | User-Visible |
|------|-------|---------|----------|-------|-----------|--------------|
| C1 | Missing required env | `ANTHROPIC_API_KEY` absent | Halt startup with clear message | No | fatal | Process won't start |
| C2 | Invalid env format | Malformed token list, bad port | Halt startup | No | fatal | Process won't start |
| C3 | Invalid config update | Bad value via `PATCH /api/v1/config` | Reject with 400, validation error | No | info | Yes — form validation error |
| C4 | Invalid source config | Bad sourceId format for source type | Reject with 400 | No | info | Yes — form validation error |

## Circuit Breaker / Backoff Patterns

| Component | Backoff Formula | Cap | Disable Threshold | Reset Condition |
|-----------|----------------|-----|-------------------|-----------------|
| Scrapers (all) | `min(2s * 2^error_count, 1h)` | 1 hour | 5 errors within 1 hour -> disabled | Successful fetch resets error_count to 0 |
| Scraper health check | Runs every 5min | -- | -- | Resets `backoff` sources past `next_retry_at` |
| Discord reconnect | Exponential backoff | 60s | Close 4004 -> immediate disable | Successful reconnect |
| LLM server errors (500/502/503) | 2s, 8s, 32s (fixed schedule) | 3 attempts | No auto-disable; logs error | Next successful call |
| LLM overloaded (529) | 30s flat per attempt | 3 attempts | No auto-disable | Next successful call |
| LLM rate limit (429) | Respects `retry-after` header | No cap (follows header) | No disable | Header-driven |
| Webhook delivery | 2s, 8s, 32s (fixed schedule) | 3 attempts | Marks report as `failed` | Manual re-delivery or next report |
| Auth rate limiting | Per-IP via @fastify/rate-limit | 5 failures/min | Temporary block | Time-based reset |

### Source State Machine

```
active --[error]--> backoff --[5 errors/1h]--> disabled
  ^                   |                           |
  |                   v                           |
  +--[successful fetch]--+     [manual re-enable via /settings]
                                or recovery cron past next_retry_at
```

Stored in `source_state` table: `status` is `active | backoff | disabled`.

## Logging Levels

All logging via pino (structured JSON). Secrets masked: `DISCORD_TOKENS`, `ANTHROPIC_API_KEY`, `TWITTERAPI_KEY`, `API_KEY`, webhook URLs.

| Level | When to Use | Examples |
|-------|-------------|---------|
| **debug** | Internal state useful only during development | Token count per chunk: `{tokens: 5832, items: 14}`. Dedup hash match: `{hash: "abc123", action: "filtered"}`. Entity alias resolved: `{alias: "$ETH", entityId: "..."}` |
| **info** | Normal operations completing successfully | Stage 1 batch completed: `{stage: "summarize", chunks: 3, items: 42, costUsd: 0.004}`. Cron job started/skipped (zero items). Source added via /settings. Report delivered successfully. |
| **warn** | Recoverable problems, transient failures | LLM retry triggered: `{error: "529", attempt: 2, backoffMs: 30000}`. Scraper backoff: `{source: "discord", sourceId: "123", errorCount: 2}`. Cron overlap skipped. Webhook retry. |
| **error** | Failures that skip work or degrade output | LLM batch skipped (malformed request, refusal). All webhook retries exhausted. Scraper disabled (5 errors). twitterapi.io schema validation failed. FTS index stale. |
| **fatal** | System cannot function, requires intervention | Invalid API key (L3) — all LLM jobs halted. Discord token revoked (S3). Missing required env vars. DB migration failure. Disk full. DB corruption. |

## User-Visible Error States (/settings)

### Health Events

Automated checks run every 5 minutes and insert into `health_events` when thresholds are breached. Critical events also fire to `ALERT_WEBHOOK_URL` (red Discord embed). Warn events surface on `/settings` only. Events dedup by category+message within 30 minutes. Acknowledge events via `PATCH /api/v1/health/:id` or from the settings page. See `GET /api/v1/health` for unacknowledged events, source states, last pulse/daily timestamps, and 24h cost.

### Pipeline Health Section

Shows last successful run per stage (ingest, summarize, synthesize, delivery). Each stage displays:
- Green: last run < 2x expected interval
- Yellow: last run > 2x expected interval (stale)
- Red: last run failed or never completed

### Source Health Table

Each source row from `GET /api/v1/sources` (JOINs sources + source_state):
- **Status badge**: active (green), backoff (yellow), disabled (red)
- **Last error**: `source_state.last_error` text, shown on hover/expand
- **Error count**: current `error_count`
- **Actions**: "Re-enable" button for disabled sources (resets error_count, sets status to active). "Test" button hits `POST /sources/:source/:sourceId/test`.

### Failed Deliveries

Reports with `delivery_status = 'failed'` listed with date, type, and error reason. Action: "Retry delivery" button (re-runs webhook.deliver for that report).

### LLM Status

- API key valid/invalid indicator (based on last call result)
- Today's cost from `llm_usage` table
- If L3 (invalid key) fired: red banner "LLM pipeline halted — API key is invalid. Update ANTHROPIC_API_KEY and restart."

### Error Display Rules

- Transient errors (retries in progress) are NOT shown to the user
- Only surface errors that require attention or indicate degraded output
- Error messages are human-readable, not raw HTTP codes
- Each error card links to the relevant action (re-enable, retry, configure)

## Recovery Procedures

| Failure | Auto/Manual | Recovery Path |
|---------|-------------|---------------|
| **LLM context exceeded (L1)** | Auto | Chunk split + retry. No intervention needed. |
| **LLM batch skipped (L2, L8)** | Auto (partial) | Items remain `processing`. Crash recovery on next startup resets to `ready`. Items re-processed in next cycle. |
| **LLM key invalid (L3)** | Manual | Update `ANTHROPIC_API_KEY` env var, restart container. All LLM jobs halted until then. |
| **LLM rate/overload (L4, L5, L6)** | Auto | Built-in retry with backoff. If all retries fail, items stay `processing`, recovered on restart. |
| **LLM bad JSON (L7)** | Auto | Correction prompt retry. If still fails, batch skipped, items recovered on restart. |
| **Scraper in backoff (S1, S2, S4, S8, S9)** | Auto | Health check cron (5min) resets sources past `next_retry_at`. Successful fetch clears error_count. |
| **Scraper disabled (5+ errors)** | Manual | User clicks "Re-enable" on /settings. Investigate root cause (URL changed, service down). |
| **Discord token revoked (S3)** | Manual | Replace token in `DISCORD_TOKENS` env var, restart. Channels for that token disabled until then. |
| **twitterapi.io schema change (S5)** | Manual | Update zod schema in `twitter.ts` to match new twitterapi.io response format. Redeploy. |
| **Webhook permanent failure (W4)** | Manual | Check/update webhook URL on /settings. Retry failed deliveries after fixing. |
| **Webhook transient failure (W1-W3)** | Auto | 3 retries with backoff. If all fail, report marked `failed`, user retries from /settings. |
| **DB migration failure (D1)** | Manual | Fix migration code, redeploy. Process won't start until migration succeeds. |
| **DB disk full (D3)** | Manual | Free disk space. Check data retention cron is running. Reduce retention if needed. |
| **DB corruption (D4)** | Manual | Restore from daily backup in `backups/` directory. Worst case: lose data since last backup. |
| **Missing env vars (C1)** | Manual | Set required env vars, restart. Startup logs tell you exactly which var is missing. |
| **Orphaned processing items** | Auto | Startup crash recovery: `UPDATE items SET status='ready' WHERE status='processing'`. Automatic on every boot. |
| **Cron overlap** | Auto | Mutex skips overlapping run, logs warning. No data loss. Next cycle picks up. |

### Startup Sequence

1. Validate env vars (C1/C2 halt here)
2. Open DB, run migrations (D1 halts here)
3. Crash recovery: reset orphaned `processing` items to `ready`
4. Start cron scheduler
5. Connect Discord Gateway (sequential, 5s gap between tokens)
6. Start Fastify server

If any fatal error occurs in steps 1-2, the process exits with a clear error message. Steps 3-6 log but do not halt on non-fatal errors.
