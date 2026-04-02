# Scheduler Reference — Podders v2

All scheduling lives in `src/scheduler.ts`. Single file, no event bus, no message queue. `node-cron` for time-based triggers, direct function calls for chained stages.

---

## 1. Startup Sequence

On `node dist/index.js run`:

```
1. config.load()                         # Validate env vars
2. db.connect()                          # Open Postgres connection pool
3. migrations.run()                      # Check migration version, run pending
4. crashRecovery()                       # See below
5. discord.connect()                     # Sequential per token, 5s IDENTIFY gap
6. registerCronJobs()                    # Order matters — see Job Table
7. server.listen(PORT)                   # Fastify starts accepting requests
```

### Crash Recovery

Runs before any cron job is registered. Resets orphaned items that were mid-LLM-call when the process died:

```sql
UPDATE items SET status = 'ready', batch_id = NULL
WHERE status = 'processing';
```

This is safe because Stage 1 writes summaries + updates item status in a single transaction. If the process crashed, either both committed (items are `processed`) or neither did (items stay `processing` and get reset here). No partial state.

---

## 2. Job Table

Cron registration order matches this table top-to-bottom. Event-driven jobs (Stage 2, flash, delivery) are not cron — they are called directly at the end of their triggering function.

| Job | Type | Cron Expression | Calls | Depends On |
|-----|------|-----------------|-------|------------|
| Stage 1 micro-batch (Twitter/RSS/News) | per-source | Each source's `poll_interval` (e.g., Twitter 2h, RSS 15min) | `summarize.runBatch()` | Items with `status='ready'` for sources due for processing. Low-confidence non-routine chunks may escalate to Sonnet. |
| Stage 1 Poisson claim (Discord) | check every 60s | Adaptive: Poisson threshold against `source_rate_history` | `summarize.runBatch()` | Discord `ready` items exceeding expected rate x 1.5 OR > 10 items |
| Stage 3 daily | cron | `{mm} {hh} * * *` in configured TZ (default `0 9 * * *`) | `synthesize.runDaily()` | Summaries from previous calendar day |
| Market pulse | cron | `0 */3 * * *` UTC (every 3h) | `pulse.run()` | Summaries from last 3h window. Sonnet. Output scales with activity. |
| Entity decay | cron | `0 0 * * *` UTC | `decay.run()` | Active entities only. Archives (not deletes) entities below threshold. |
| Data retention | cron | `5 0 * * *` UTC (after decay) | `retention.run()` | Runs after decay so pruned entities are already decayed |
| Backup | cron | `10 0 * * *` UTC (after retention) | `db.backup()` (pg_dump) | Runs after retention so backup is clean |
| Scraper health | cron | `*/5 * * * *` (every 5min) | `sources.healthCheck()` | `source_state` rows with `status='backoff'` |
| Health check | cron | `*/5 * * * *` (every 5min) | `health.check()` | 7 checks: source silence, source disabled, LLM failures, missed pulse, missed daily, DB pool, cost spike. Inserts into `health_events`. Critical → `ALERT_WEBHOOK_URL`. Dedup: skip if same category+message unacknowledged within 30min. |
| Narrative clustering | direct call | Before Stage 3 daily synthesis | `narratives.detect()` | Summary embeddings from previous day |
| Stage 2 correlate | direct call | After each Stage 1 | `correlate.run()` | Stage 1 completion |
| Stage 3 flash | direct call | When Stage 1 returns `urgency: 'breaking'` | `synthesize.runFlash()` | Breaking chunk from Stage 1 |
| Delivery | direct call | After Stage 3 daily | `webhook.deliver(report)` | Stage 3 daily completion |
| Trust weight adjust | direct call | 1 hour after flash delivery | `trust.adjustWeights(flashId)` | Flash delivery completion |

The midnight UTC cluster (decay at `:00`, retention at `:05`, backup at `:10`) is staggered by 5 minutes to spread load.

---

## 3. Mutex Contract

Each cron job has a `running` boolean flag scoped to that job. Implementation:

```typescript
const jobState: Record<string, boolean> = {
  'stage1': false,
  'stage3-daily': false,
  'pulse': false,
  'entity-decay': false,
  'retention': false,
  'backup': false,
  'health-check': false,
  'health-monitor': false,
};

function withMutex(jobName: string, fn: () => Promise<void>): () => Promise<void> {
  return async () => {
    if (shuttingDown) return;
    if (jobState[jobName]) {
      log.warn({ job: jobName }, 'skipped: previous run still in progress');
      return;
    }
    jobState[jobName] = true;
    try {
      await fn();
    } catch (err) {
      log.error({ job: jobName, err }, 'job failed');
    } finally {
      jobState[jobName] = false;
    }
  };
}
```

**What happens on overlap**: the new invocation is skipped entirely and a warning is logged. The previous run continues uninterrupted. This is a skip-not-queue strategy — if Stage 1 takes longer than a source's poll interval (unlikely but possible with LLM API degradation), one cycle is missed and the next picks up all accumulated `ready` items.

**Why this is safe**: batch-claiming SQL (`UPDATE items SET batch_id = $1, status = 'processing' WHERE status = 'ready'`) prevents double-processing regardless of mutex. The mutex is a performance guard (don't waste LLM calls), not a correctness guard.

Event-driven jobs (Stage 2, flash, delivery) do not need a mutex. They execute inline at the tail of their parent function and cannot overlap by construction.

---

## 3.5. Poisson-Based Claim Gate (Discord)

Discord messages arrive via WebSocket and sit as `ready`. Instead of a fixed `poll_interval` timer, the scheduler checks every 60 seconds:

1. Count accumulated `ready` items for each Discord source.
2. Compare against expected rate (lambda) for that source at this hour-of-day and day-of-week, learned from last 7 days via `source_rate_history`.
3. Two thresholds:
   - **Process threshold**: actual > expected x 1.5 OR actual > 10 items — trigger processing.
   - **Alert threshold**: actual > expected x 3 OR Poisson survival probability < 0.01 — process immediately, flag as potential breaking event.

### Poisson survival function

```javascript
function poissonSurvival(k, lambda) {
  let sum = 0;
  for (let i = 0; i < k; i++) {
    sum += Math.exp(-lambda) * Math.pow(lambda, i) / factorial(i);
  }
  return 1 - sum; // P(X >= k)
}
```

### Baseline tracking

Lambda is stored in `source_rate_history`, keyed by `(source, source_id, hour_of_day, day_of_week)`. Updated after each processing cycle with a rolling average over the last 7 days.

### What this changes

- **Discord sources**: Poisson-triggered claim (check every 60s), NOT fixed `poll_interval`. The claim gate is a decision point, not just a timer.
- **Twitter/RSS/News**: unchanged, still fixed `poll_interval`.
- Catches breaking events ~12 minutes faster than fixed 15-min intervals.

---

## 4. Call Chain

Two entry points, zero event emitters.

### Entry 1: Stage 1 (per-source intervals, parallel processing)

```
scheduler checks due sources (per poll_interval)
  └─ withMutex('stage1', async () => {
       const batches = await summarize.runBatch()
       //   ├─ queries sources due for processing (last_processed_at + poll_interval <= NOW())
       //   ├─ processes all due sources in parallel via Promise.allSettled
       //   ├─ per source: claims items, chunks by token budget (6,000 tokens per chunk)
       //   ├─ calls Haiku per chunk via llm.ts
       //   ├─ if confidence < 5 AND urgency != 'routine': escalates chunk to Sonnet
       //   ├─ writes summaries + entities + updates items in one tx
       //   └─ returns ChunkResult[] (includes urgency per chunk)

       await correlate.run()
       //   └─ SQL: entities appearing in 2+ sources within same window

       if (batches.some(b => b.urgency === 'breaking')) {
         const flashReport = await synthesize.runFlash()
         //   ├─ reads summaries from last 4h
         //   ├─ calls Sonnet via llm.ts
         //   ├─ writes report with type='flash'
         //   └─ await webhook.deliver(report, { prefix: '[FLASH]' })

         if (flashReport) {
           // Schedule trust weight adjustment 1 hour after flash delivery
           setTimeout(() => trust.adjustWeights(flashReport.id), 60 * 60 * 1000);
           //   ├─ checks which sources confirmed within 1 hour
           //   ├─ confirmed sources: trust_weight += 0.05 (capped at initial + 0.2)
           //   └─ unconfirmed sources: trust_weight -= 0.03 (floored at initial - 0.2)
         }
       }
     })
```

### Entry 2: Market Pulse (every 3h)

```
cron('0 */3 * * *', { timezone: 'UTC' })
  └─ withMutex('pulse', async () => {
       const report = await pulse.run()
       //   ├─ reads summaries from last 3h window
       //   ├─ calls Sonnet via llm.ts (output length scales with activity)
       //   ├─ quiet periods → shorter pulse, busy periods → fuller
       //   └─ writes report with type='pulse'

       if (report) {
         await webhook.deliver(report)
         //   └─ muted grey embed (0x4A4A5A), title "Market Pulse — HH:MM WIB"
       }
     })
```

### Entry 3: Stage 3 Daily Cron (at digest_time)

```
cron('{mm} {hh} * * *', { timezone })
  └─ withMutex('stage3-daily', async () => {
       const narratives = await narratives.detect()
       //   ├─ loads summary embeddings from previous calendar day
       //   ├─ runs k-means clustering with silhouette-based k selection
       //   ├─ names clusters with 3+ members via Haiku (~3-5 calls)
       //   ├─ classifies signal strength vs previous day (new/emerging/strong/stable/fading)
       //   └─ writes to narratives table

       const report = await synthesize.runDaily()
       //   ├─ reads summaries where window overlaps previous calendar day
       //   ├─ reads correlated entities for same window
       //   ├─ fetches yesterday's report.tldr for temporal anchoring
       //   ├─ if > 50 summaries, ranks by item_count * avg_engagement, takes top 30
       //   ├─ if 0 summaries, skips (logs info, no empty report)
       //   ├─ checks duplicate: SELECT id FROM reports WHERE date=$1 AND type='daily'
       //   ├─ calls Sonnet via llm.ts
       //   └─ writes report with type='daily'

       if (report) {
         await webhook.deliver(report)
         //   ├─ POST to configured Discord webhook URL
         //   ├─ includes "View full report" link if PUBLIC_URL is set
         //   ├─ 3 retries: 2s → 8s → 32s
         //   └─ updates reports.delivery_status
       }
     })
```

### Entry 4: Health Check (every 5min)

```
cron('*/5 * * * *', { timezone: 'UTC' })
  └─ withMutex('health-monitor', async () => {
       await health.check()
       //   ├─ source silence: no data for 3x poll_interval → warn
       //   ├─ source disabled: any source status='disabled' → critical
       //   ├─ LLM failures: 0 LLM calls when items ready for >1h → critical
       //   ├─ missed pulse: >4h since last pulse → warn
       //   ├─ missed daily: >26h since last daily → critical
       //   ├─ DB pool: idle connections = 0 for 3 consecutive checks → warn
       //   ├─ cost spike: hourly cost >3x the 7-day average → warn
       //   ├─ dedup: skip if same category+message unacknowledged within 30min
       //   └─ critical events → POST to ALERT_WEBHOOK_URL (red Discord embed)
     })
```

Also writes heartbeat file (`/tmp/podders-heartbeat`) every 60s from the main process loop. External watchdog (systemd or cron) restarts if file age >300s.

### Function Signatures

```typescript
// src/process/summarize.ts
export async function runBatch(): Promise<ChunkResult[]>

// src/process/correlate.ts
export async function run(): Promise<CorrelatedEntity[]>

// src/process/synthesize.ts
export async function runDaily(): Promise<MarketReport | null>
export async function runFlash(): Promise<MarketReport | null>

// src/process/narratives.ts
export async function detect(): Promise<Narrative[]>

// src/process/pulse.ts
export async function run(): Promise<MarketReport | null>

// src/knowledge/trust.ts
export async function adjustWeights(flashReportId: string): Promise<void>

// src/health.ts
export async function check(): Promise<void>

// src/deliver/webhook.ts
export async function deliver(
  report: MarketReport,
  opts?: { prefix?: string }
): Promise<void>
```

---

## 5. Rescheduling

`digest_time` and `timezone` are stored in `app_config`. When changed via `PATCH /api/v1/config`:

```typescript
// In the PATCH /api/v1/config route handler
if (body.digestTime || body.timezone) {
  await pool.query(`UPDATE app_config SET value = $1 WHERE key = $2`, [digestTime, 'digest_time']);
  await pool.query(`UPDATE app_config SET value = $1 WHERE key = $2`, [timezone, 'timezone']);

  scheduler.rescheduleDaily(digestTime, timezone);
}
```

```typescript
// src/scheduler.ts
let dailyCronTask: cron.ScheduledTask | null = null;

export function rescheduleDaily(digestTime: string, timezone: string): void {
  if (dailyCronTask) {
    dailyCronTask.stop();    // removes from node-cron internal list
    dailyCronTask = null;
  }

  const [hh, mm] = digestTime.split(':');
  dailyCronTask = cron.schedule(
    `${parseInt(mm)} ${parseInt(hh)} * * *`,
    withMutex('stage3-daily', runDailyAndDeliver),
    { timezone }
  );

  log.info({ digestTime, timezone }, 'rescheduled daily digest');
}
```

`node-cron` does not support modifying a running task's schedule. The only path is destroy + recreate. The `stop()` call is synchronous and immediate — if a daily run is currently in-flight (held by the mutex), the cron trigger is removed but the in-flight run completes normally.

---

## 6. Graceful Shutdown

Trap `SIGTERM` (Docker stop) and `SIGINT` (Ctrl+C):

```typescript
let shuttingDown = false;
const inflightLLM: Set<Promise<unknown>> = new Set();

export function trackLLMCall<T>(promise: Promise<T>): Promise<T> {
  inflightLLM.add(promise);
  promise.finally(() => inflightLLM.delete(promise));
  return promise;
}

async function shutdown(signal: string): Promise<void> {
  if (shuttingDown) return;         // prevent double-fire
  shuttingDown = true;
  log.info({ signal }, 'shutting down');

  // 1. Stop all cron tasks — no new jobs will fire
  cron.getTasks().forEach(task => task.stop());

  // 2. Wait for in-flight LLM calls (tracked via inflightLLM set)
  if (inflightLLM.size > 0) {
    log.info({ count: inflightLLM.size }, 'waiting for in-flight LLM calls');
    await Promise.allSettled([...inflightLLM]);
  }

  // 3. Close Discord gateway connections
  discord.closeAll();

  // 4. Close Fastify server (stop accepting HTTP)
  await server.close();

  // 5. Close DB connection pool last (in-flight jobs may still write)
  await pool.end();

  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

// Hard kill safety net — pm2/systemd sends SIGKILL after timeout
setTimeout(() => {
  log.error('forced exit after 30s timeout');
  process.exit(1);
}, 30_000).unref();
```

The `shuttingDown` flag is checked at the top of `withMutex`. Once set, no cron callback will begin a new run. In-flight runs complete because they already passed the check.

`llm.ts` calls `trackLLMCall()` on every Anthropic API request so the shutdown sequence knows what to wait for.

---

## 7. Error Propagation

### Stage 1 fails mid-batch

Stage 1 processes multiple chunks sequentially. Each chunk is independent:

- **Chunk N fails (LLM error)**: that chunk's items stay `processing`. Next successful startup resets them to `ready` via crash recovery. Other chunks in the same batch are unaffected.
- **Chunk N fails (DB write error)**: transaction rolls back. Items stay `processing`, same recovery path.
- **All chunks fail**: `summarize.runBatch()` throws. The `withMutex` catch logs the error. `correlate.run()` and flash check never execute.

**Stage 2 impact**: `correlate.run()` only fires if `runBatch()` completes without throwing. If Stage 1 partially succeeds (some chunks wrote summaries), Stage 2 does not run for that cycle. Those summaries are still in the DB and will be picked up by the next Stage 3 daily run.

### Stage 2 fails

`correlate.run()` is SQL-only, no LLM. Failure is a query error (schema issue, connection error). If it throws, the flash check still does not execute (sequential calls, no try/catch separation). Daily synthesis is unaffected — it reads correlations independently.

### Stage 3 daily fails

- **LLM error**: `synthesize.runDaily()` throws. `webhook.deliver()` never fires. Report is not written to DB. Next day's run covers a fresh midnight-to-midnight window — the failed day's summaries are still available but will age out of the 24h window.
- **No automatic retry**: a missed daily digest is missed. The summaries persist for 90 days, so manual re-run or next day's report captures the context via yesterday's TL;DR anchoring.

### Stage 3 flash fails

Same as daily — LLM error means no flash report written, no delivery attempted. Breaking events are not retried. The underlying summaries are preserved and will appear in the next daily digest.

### Delivery fails

`webhook.deliver()` retries 3x with exponential backoff (2s, 8s, 32s). If all retries fail:

- `reports.delivery_status` is set to `'failed'`
- The report body is preserved in DB — nothing is lost
- Failed deliveries surface on `/settings` dashboard
- No automatic re-delivery. User can trigger manually (future: retry button on settings page).

### Summary: failure isolation

| Failure Point | Stage 2 runs? | Flash runs? | Daily runs? | Delivery runs? |
|---------------|---------------|-------------|-------------|----------------|
| Stage 1 partial (some chunks) | No | No | Yes (next cycle) | N/A |
| Stage 1 total failure | No | No | Yes (next cycle, fewer summaries) | N/A |
| Stage 2 failure | N/A | No | Yes (without correlations for that cycle) | N/A |
| Stage 3 daily failure | N/A | N/A | N/A | No |
| Stage 3 flash failure | N/A | N/A | Yes (unaffected) | No |
| Delivery failure | N/A | N/A | N/A | Report saved, status='failed' |
