# Decision 5: Embedding-Based Narrative Clustering

## Decision
Option C — Cluster existing summary embeddings to detect emerging narratives. One Haiku call per cluster to name it. Track cluster growth for weak/strong signal classification.

## Why This Decision
- We already embed every summary with Gemini — vectors are sitting unused for this purpose
- Cost is ~$0.01/day (3-5 Haiku calls to name clusters)
- EvoTaxo (Mar 2026), k-LLMmeans (Feb 2026), DenStream (Jan 2026) all converge on: embed → cluster → LLM names → track over time
- Entity tracking tells you WHAT. Sentiment tells you HOW. Narrative clustering tells you WHY — the missing intelligence layer
- Option B (SQL co-occurrence) can't detect unknown themes. Option D (taxonomy) needs clusters first anyway

## Research Papers
- EvoTaxo (Mar 2026) — arxiv.org/abs/2603.19711 — Evolving taxonomy from social media streams
- k-LLMmeans (Feb 2026) — arxiv.org/abs/2502.09667 — Mini-batch k-means for streaming text with LLM embeddings
- Online Density-Based Clustering (Jan 2026) — arxiv.org/abs/2601.20680 — DenStream beats HDBSCAN for real-time narrative monitoring
- BEYONDWORDS (Mar 2025) — arxiv.org/abs/2503.01880 — Cluster tweets, Chain of Thought naming, secondary QA
- BERTrend (Nov 2024) — arxiv.org/abs/2411.05930 — Weak/strong signal classifier for emerging topics
- HERCULES (Jun 2025) — arxiv.org/abs/2506.19992 — Hierarchical recursive clustering + summarization

## Implementation

### When It Runs
Daily synthesis job, before Sonnet writes the report. Load recent summary embeddings, cluster, name clusters, feed to Sonnet as context.

### Clustering Logic
```typescript
import { kmeans } from 'ml-kmeans'; // or similar

async function detectNarratives(summaries: Summary[]): Promise<Narrative[]> {
  const vectors = summaries.map(s => s.embedding);
  
  // Auto-tune k using silhouette score
  let bestK = 3;
  let bestScore = -1;
  for (let k = 3; k <= Math.min(10, Math.floor(summaries.length / 3)); k++) {
    const result = kmeans(vectors, k);
    const score = silhouetteScore(vectors, result.clusters);
    if (score > bestScore) { bestScore = score; bestK = k; }
  }
  
  const clusters = kmeans(vectors, bestK);
  
  // Name each cluster with 3+ members
  const narratives: Narrative[] = [];
  for (const cluster of clusters.filter(c => c.members.length >= 3)) {
    const sampleSummaries = cluster.members.slice(0, 5).map(i => summaries[i].summary);
    const name = await llm.call({
      model: config.models.haiku,
      system: 'Given these related market summaries, name the theme in 3-5 words. Output only the name.',
      prompt: sampleSummaries.join('\n---\n'),
      maxTokens: 20
    });
    narratives.push({
      name: name.text.trim(),
      memberCount: cluster.members.length,
      avgSentiment: avg(cluster.members.map(i => summaries[i].avgSentiment)),
      summaryIds: cluster.members.map(i => summaries[i].id)
    });
  }
  return narratives;
}
```

### Signal Classification
```typescript
function classifySignal(narrative: Narrative, previousDay?: Narrative): SignalStrength {
  if (!previousDay) return 'new';
  const growth = narrative.memberCount / previousDay.memberCount;
  if (growth >= 3.0) return 'strong'; // 3x growth = strong signal
  if (growth >= 1.5) return 'emerging'; // 1.5x = emerging
  if (growth <= 0.5) return 'fading'; // halved = fading
  return 'stable';
}
```

### Daily Report Integration
Feed to Sonnet in daily synthesis prompt:
```
Emerging narratives detected:
- "RWA Rotation" (12 summaries, +300% vs yesterday, STRONG signal)
- "Memecoin Season Cooling" (5 summaries, -40% vs yesterday, FADING)
- "Indonesian Regulatory Shift" (3 summaries, NEW)
```

### Schema
```sql
CREATE TABLE narratives (
  id TEXT PRIMARY KEY, -- ULID
  name TEXT NOT NULL,
  date DATE NOT NULL,
  member_count INTEGER NOT NULL,
  avg_sentiment REAL,
  signal_strength TEXT NOT NULL, -- new, emerging, strong, stable, fading
  summary_ids TEXT[] NOT NULL,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_narratives_date ON narratives(date DESC);
```

### Cost Impact
3-5 Haiku calls/day × ~200 tokens each = ~$0.01/day. Negligible.

## Upgrade Path: C → D (Hierarchical Taxonomy)

When to upgrade: after 2-4 weeks of real data, if you see 20+ clusters forming natural hierarchies.

### What Changes
```sql
ALTER TABLE narratives ADD COLUMN parent_id TEXT REFERENCES narratives(id);
ALTER TABLE narratives ADD COLUMN depth INTEGER DEFAULT 0;
```

### Hierarchy Detection
```typescript
// One Haiku call per day to propose hierarchy
const hierarchyPrompt = `Given these narrative clusters, propose a 2-level hierarchy.
Clusters: ${narratives.map(n => n.name).join(', ')}
Output JSON: [{"parent": "Crypto", "children": ["RWA Rotation", "Memecoin Cooling"]}]`;
```

### EvoTaxo Monthly Evolution
- Every 30 days, re-evaluate taxonomy structure
- New clusters may create new parent categories
- Old parent categories with no children get pruned
- One Sonnet call per month to restructure: "Given last month's taxonomy and this month's clusters, update the hierarchy"
