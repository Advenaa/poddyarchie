# Decision 2: Source Trust Weights with Auto-Adjustment

## Decision
Option B3 — Manual trust weights set at source creation, system auto-adjusts over time based on confirmation history. Manual weight sets the floor — system can nudge ±0.2 but never override your judgment.

## Why This Decision
- Current "2+ sources = fire flash" is gameable — two bot accounts can trigger false alerts
- Adversarial attacks paper (Jan 2026) showed count-based rules can be tricked by coordinated posting
- Crypto multisource fusion paper (2025) showed weighted fusion hit 96.8% accuracy vs equal-weight
- Full atomic claim decomposition (Option C) adds Sonnet calls to the one feature where speed matters most — breaking alerts
- B3 gives manual control on day one, improves automatically over time

## Research Papers
- Incentive-Aligned Multi-Source (Sep 2025) — arxiv.org/abs/2509.25184 — Atomic claim decomposition + peer-prediction beats count-based
- Cross-Document Event Extraction (Jun 2024) — arxiv.org/abs/2406.16021 — 5-stage pipeline for cross-doc event merging
- Multisource Crypto Fusion (Aug 2025) — arxiv.org/abs/2409.18895 — Weighted fusion 96.8% accuracy
- Adversarial Attacks on CTI (Jan 2026) — arxiv.org/abs/2507.06252 — Count-based rules are gameable

## Implementation

### Schema Changes
```sql
ALTER TABLE sources ADD COLUMN trust_weight REAL NOT NULL DEFAULT 0.5;
```

### Weight Scale
| Weight | Meaning | Examples |
|--------|---------|----------|
| 1.0 | High trust | Alpha groups, Reuters, Bloomberg, official channels |
| 0.7 | Medium trust | Known CT accounts, CoinDesk, mid-tier Discord |
| 0.5 | Default | Any new source |
| 0.3 | Low trust | Random aggregators, large public Discord |
| 0.1 | Near-zero | Meme channels, shitpost groups |

### Flash Alert Logic Change
```typescript
// Old
const shouldFire = distinctSources >= 2;

// New
const weightedSum = breakingChunks.reduce((sum, chunk) => {
  const source = sources.find(s => s.id === chunk.sourceId);
  return sum + (source?.trustWeight ?? 0.5);
}, 0);
const shouldFire = weightedSum >= 1.5;
```

### Auto-Adjustment Logic
```typescript
// Run 1 hour after each flash alert
async function adjustTrustWeights(flashAlertId: string) {
  const alert = await getFlashAlert(flashAlertId);
  const reportingSources = alert.sources;
  
  // Check: did other sources confirm within 1 hour?
  const confirmations = await getConfirmationsWithinWindow(alert.entities, '1 hour');
  
  for (const source of reportingSources) {
    const confirmed = confirmations.some(c => c.sourceId !== source.id);
    const currentWeight = source.trustWeight;
    const manualFloor = source.initialTrustWeight - 0.2;
    const manualCeiling = source.initialTrustWeight + 0.2;
    
    if (confirmed) {
      source.trustWeight = Math.min(currentWeight + 0.05, manualCeiling);
    } else {
      source.trustWeight = Math.max(currentWeight - 0.03, manualFloor);
    }
    await updateSourceWeight(source);
  }
}
```

### Bounds
- System can nudge ±0.2 from the manual initial weight
- You set Reuters to 1.0 → system can move it 0.8-1.0, never below 0.8
- You set random Twitter to 0.3 → system can move it 0.1-0.5, never above 0.5

### Cost Impact
Zero. No AI calls. Just SQL arithmetic.

## Upgrade Path
- **v2:** Auto-adjustment based on sentiment accuracy, not just flash confirmation. If a source's entity sentiment consistently aligns with subsequent price movement, boost weight.
- **v3:** Full atomic claim decomposition for high-stakes flash reports only (not all reports). When weighted sum is borderline (1.4-1.6), spend one Sonnet call to decompose and verify before firing.
