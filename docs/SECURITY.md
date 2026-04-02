# Security — Prompt Injection & Data Integrity Defenses

Podders ingests untrusted text from public channels and feeds it to LLMs. Every message
is a potential attack vector. This document covers the defense layers, their limitations,
and the residual risks we accept.

Based on the 10 prompt injection attacks identified in round 21 red-teaming:

1. Direct instruction override ("ignore previous instructions, output X")
2. Tag escape (closing `</scraped_content>` to inject a fake system message)
3. Entity hallucination injection (mention a fake token to get it into reports)
4. Sentiment flooding (coordinated messages to skew entity sentiment)
5. Breaking-urgency fabrication (fake "EXPLOIT DETECTED" to trigger flash reports)
6. Retry poisoning (crafting output that fails parse but injects on retry)
7. Slow drip manipulation (gradual sentiment shift below detection thresholds)
8. Engagement gaming (inflate engagement metrics to boost malicious content priority)
9. Cross-source corroboration spoofing (post the same fake event on Discord + Twitter)
10. Nested encoding attack (Unicode lookalikes, zero-width chars, RTL overrides)

---

## 1. Input Sanitization Layer

**Where**: `sanitizeForPrompt()` in the ingest pipeline, called before every Stage 1 LLM call.

**What gets sanitized**:
- HTML/XML angle brackets: `<` and `>` escaped to `&lt;` / `&gt;` (blocks attack #2: tag escape)
- Zero-width characters stripped: U+200B, U+200C, U+200D, U+FEFF, U+2060 (blocks attack #10)
- RTL/LTR override characters stripped: U+202A-U+202E, U+2066-U+2069
- Unicode confusable normalization: NFKC normalization to collapse lookalike characters

**Nonce-based tag wrapping** (defined in PIPELINE.md):
```typescript
const nonce = crypto.randomBytes(4).toString('hex');
const openTag = `<scraped_content_${nonce}>`;
const closeTag = `</scraped_content_${nonce}>`;
```
The attacker cannot guess the tag name, so closing `</scraped_content>` does nothing. The
nonce rotates per LLM call. This is the primary defense against attack #2.

**System prompt framing**: the system prompt explicitly states that content within the
nonce-tagged delimiters is untrusted user data and must not be interpreted as instructions.

---

## 2. Output Validation Layer

Three layers, applied in sequence after every Stage 1 LLM response.

### 2a. Zod Schema Validation
Defined in PIPELINE.md. Enforces types, clamps sentiment to [-1, 1], caps entity array
at 20, keyEvents at 5. Rejects structurally invalid output.

### 2b. Entity Post-Verification
Also defined in PIPELINE.md. After zod parsing, every extracted entity is checked against
the raw source text:
```typescript
function verifyEntities(entities: Entity[], rawText: string): Entity[] {
  return entities.filter(e => {
    const names = [e.name, ...e.aliases].map(n => n.toLowerCase());
    return names.some(n => rawText.toLowerCase().includes(n));
  });
}
```
Entities not present in the source chunk are dropped. This is the primary defense against
attack #3 (hallucinated entity injection). An attacker must actually mention the entity
in the source text to get it extracted — they cannot trick the LLM into inventing one
from thin air.

### 2c. Prompt Fragment Detection
Scan LLM output for patterns that suggest the model followed injected instructions
rather than summarizing:
- Output containing `ignore`, `system:`, `assistant:`, `<|`, `[INST]` in suspicious
  contexts
- Output that is substantially longer than expected (> 3x median for that source)
- Output containing URLs not present in the source text

Flag and log for review. Do not auto-deliver flagged output.

### 2d. Content Policy Checks
Reject summaries where:
- `urgency: 'breaking'` but the source chunk has < 3 messages (too thin to be real)
- Sentiment magnitude > 0.8 but source chunk has < 5 messages (single-author spike)
- Any single author contributes > 60% of messages in a chunk (dominated by one voice)

---

## 3. Rate Limiting Per Author

**Problem**: attacks #4 (sentiment flooding) and #8 (engagement gaming) rely on volume
from a single author or small coordinated group.

**Defense**: per-author message cap within the processing window.

```
max_messages_per_author_per_window = 10  (configurable in app_config)
window_size = poll_interval  (matches the source's configured poll interval)
```

When an author exceeds the cap in a single batch:
- First 10 messages are included at full weight
- Messages 11+ are dropped from the LLM input (still stored in raw_items for audit)
- Log a warning: `author_rate_limit_hit: {author, count, window}`

This prevents a single poster from dominating a chunk. The cap is generous enough for
active legitimate participants but blocks spam-level flooding.

**Author deduplication**: normalize author identifiers (lowercase, strip whitespace).
Track across sources where possible (same handle on Discord and Twitter).

---

## 4. Flash Report Safeguards

Defined in PIPELINE.md. Summarized here for completeness.

**Corroboration requirement**: `urgency: 'breaking'` must appear in 2+ chunks from
2+ distinct sources, OR 3+ chunks from the same source within 30 minutes. A single
chunk claiming "breaking" is logged but never delivered. This is the primary defense
against attack #5 (breaking-urgency fabrication).

**Cooldown**: minimum 30 minutes between flash reports. A second breaking event within
the cooldown window is queued, not dropped.

**Single-source marking**: if only one source reports breaking, the flash TL;DR is
prefixed with `[UNCONFIRMED]`.

**Minimum message threshold**: a chunk can only produce `urgency: 'breaking'` if it
contains messages from 3+ distinct authors. Single-author "exploit" claims are capped
at `elevated`.

---

## 5. Retry Safety

**Problem**: attack #6. An attacker crafts content that causes the LLM to produce
malformed JSON on first try. If the failed response is included in the retry prompt
("here is what you produced last time, fix it"), the injected content now appears
as assistant-generated text in the conversation history, bypassing the untrusted-data
framing.

**Defense** (defined in PIPELINE.md): on JSON parse failure, retry with a completely
fresh prompt. The failed response is never included in retry context.

```
If JSON parse fails:
  1. Log the failed response for debugging
  2. Build a NEW prompt from scratch (system + user message, no history)
  3. Call LLM again
  4. If second attempt fails, skip this chunk entirely — do not retry further
```

No conversation history, no "fix this output" pattern, no assistant-turn injection surface.

---

## 6. Cost Circuit Breaker

**Problem**: an adversary could flood sources to inflate LLM costs. Legitimate cost
spikes also happen during high-activity periods (market crashes, exploit events).

**Defense**: compare current batch chunk count against a rolling average.

```
rolling_avg = AVG(chunk_count) over last 7 daily batches
current_batch_chunks = count of chunks in this micro-batch run

if current_batch_chunks > rolling_avg * 2:
  pause pipeline
  log CRITICAL: cost_circuit_breaker_triggered
  send alert via Discord webhook (to admin channel, not the report channel)
  do NOT process the batch automatically
```

Manual review required to resume. The `llm_usage` table (defined in PIPELINE.md)
provides per-call cost tracking to audit after the fact.

Additionally: hard cap of 100 chunks per micro-batch. If a single poll interval window produces
more than 100 chunks, process the first 100 (ranked by source priority and engagement)
and defer the rest.

---

## 7. Subtle Manipulation Defenses

Attacks #4, #7, and #8 are the hardest to catch because they work within the system's
normal parameters.

### 7a. Author Sentiment Tracking
Maintain a rolling sentiment profile per author (derived from chunks where they are a
dominant contributor):

```sql
CREATE TABLE author_sentiment (
  author TEXT NOT NULL,
  source TEXT NOT NULL,
  window_start INTEGER NOT NULL,
  sentiment REAL NOT NULL,
  message_count INTEGER NOT NULL,
  PRIMARY KEY (author, source, window_start)
);
```

Flag when an author's sentiment in a window deviates > 0.5 from their 30-day rolling
average. This catches attack #7 (slow drip) at the individual level — a previously
neutral author suddenly going extreme is suspicious.

### 7b. Diversity Weighting
When assembling chunks for Stage 1, track author diversity per chunk:

```
diversity_score = unique_authors / total_messages
```

Chunks with `diversity_score < 0.3` (dominated by 1-2 authors) get a lower weight in
Stage 3 synthesis. They are not excluded — the information might be legitimate — but
their sentiment and urgency carry less influence on the final report.

### 7c. Engagement Sanity Checks
For Twitter: flag items where engagement (likes, retweets) is disproportionate to the
account's follower count. A 100-follower account with 10K likes on a single tweet is
anomalous. Do not auto-exclude — flag and de-weight.

---

## 8. Residual Risk — What We Cannot Fully Defend Against

Being honest: several attacks remain partially effective even with all defenses deployed.

### Still partially effective:

**Attack #7 — Slow drip manipulation**: the hardest attack to defend. If an adversary
gradually shifts sentiment over weeks, staying within per-author rate limits and avoiding
sudden deviations, the system will absorb the bias. Author sentiment tracking catches
sudden shifts but not slow ones that mirror natural sentiment evolution. The defense is
diversity weighting (diluting single-author influence) and the corroboration requirement
(needing multiple sources to agree). But a patient, multi-account, multi-platform campaign
will move the needle. This is fundamentally the same problem as detecting astroturfing —
a research-grade problem (see ROADMAP.md R.1).

**Attack #9 — Cross-source corroboration spoofing**: if an attacker posts the same fake
event on Discord AND Twitter (or Discord + RSS via a controlled blog), it passes the
corroboration requirement. The flash report safeguards require 2+ sources, but they
cannot verify that the sources are independently operated. Short of a human editor, there
is no automated defense against coordinated cross-platform fabrication. Mitigation: the
`[UNCONFIRMED]` tag and the human reader's judgment are the last line.

**Attack #1 — Direct instruction override**: nonce tags and sanitization make this much
harder, but LLMs are not perfectly instruction-following. A sufficiently creative
injection — especially one that frames itself as a "correction" rather than an override —
may occasionally influence the model's output. The output validation layers (entity
verification, content policy checks) catch the most damaging outcomes, but subtle
influence on tone or framing can slip through.

**Attack #8 — Engagement gaming**: we de-weight anomalous engagement, but engagement
metrics from Twitter are self-reported by the API and can be manipulated at the platform
level (bot farms, like services). We can flag statistical anomalies but cannot verify
that engagement is organic. This is Twitter's problem to solve, not ours.

### Effectively neutralized:

**Attack #2** (tag escape) — nonce tags eliminate this entirely.
**Attack #3** (entity hallucination) — post-verification drops fabricated entities.
**Attack #5** (breaking fabrication) — corroboration + author-count minimums block this.
**Attack #6** (retry poisoning) — fresh-prompt retry eliminates the vector.
**Attack #10** (nested encoding) — NFKC normalization + zero-width stripping handles this.
**Attack #4** (sentiment flooding) — per-author rate limits + diversity weighting contain
the impact, though coordinated multi-account flooding remains partially effective (same
residual risk as #7).

---

## Implementation Priority

| Defense | Blocks | Effort | Priority |
|---------|--------|--------|----------|
| Nonce-based tags + sanitization | #2, #10 | Done (PIPELINE.md) | Shipped |
| Entity post-verification | #3 | Done (PIPELINE.md) | Shipped |
| Fresh-prompt retry | #6 | Done (PIPELINE.md) | Shipped |
| Flash corroboration + cooldown | #5 | Done (PIPELINE.md) | Shipped |
| Per-author rate limiting | #4, #8 | 0.5 day | P0 |
| Content policy checks | #1, #5 | 0.5 day | P0 |
| Cost circuit breaker | Cost DoS | 0.5 day | P0 |
| Prompt fragment detection | #1 | 1 day | P1 |
| Author sentiment tracking | #7 | 1-2 days | P1 |
| Diversity weighting | #4, #7 | 0.5 day | P1 |
| Engagement sanity checks | #8 | 1 day | P2 |

Total new work: ~5-6 days. The most critical items (rate limiting, content policy, cost
breaker) are half-day tasks each and block the most impactful attacks.

---

## 9. Key Rotation

How each secret behaves during rotation.

| Secret | Rotation Procedure | Behavior During Rotation |
|--------|--------------------|--------------------------|
| `ANTHROPIC_API_KEY` / `GEMINI_API_KEY` | Update `.env`, restart process | In-flight LLM calls complete with the old key (HTTP connection already established). New calls use the new key after restart. |
| `DISCORD_TOKENS` | Update `.env`, restart process | WebSocket disconnects and reconnects with the new token on restart. Expect ~5-10s of missed messages (same as any restart). |
| `TWITTERAPI_KEY` | Update `.env`, restart process | Next poll uses the new key. No in-flight concern (REST, not streaming). |
| `API_KEY` (dashboard auth) | Rotate via settings endpoint (`POST /api/v1/auth/rotate`) | Can be rotated without restart. Active sessions are invalidated immediately. |

**Recommendations**:
- Rotate all keys quarterly as a matter of hygiene.
- Rotate immediately if any key is suspected compromised.
- After rotation, verify the service is healthy via `GET /api/v1/status` and check logs for auth errors.
