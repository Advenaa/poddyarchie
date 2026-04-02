# Decision 7: RAG Chat with Three Retrieval Tools

## Decision
Option D — Give Sonnet three retrieval tools: semantic_search, keyword_search, read_raw. Sonnet picks the right tool per query. We already have all three backends.

## Why This Decision
- A-RAG (Feb 2026) introduced hierarchical retrieval interfaces — model picks retrieval strategy per query
- CompactRAG (Feb 2026) proved complex questions need only 2 LLM calls: decompose + synthesize
- We already have the infrastructure: Gemini embeddings (semantic), Postgres entities table (keyword), items table (raw read)
- Option C (complexity-aware without tools) is smart but still hardcodes retrieval strategy. Option D lets Sonnet adapt per query
- AutothinkRAG (Mar 2026) showed automatic complexity detection works — Sonnet decides search depth naturally

## Research Papers
- A-RAG (Feb 2026) — arxiv.org/abs/2602.03442 — Hierarchical retrieval tools: keyword, semantic, chunk_read
- CompactRAG (Feb 2026) — arxiv.org/abs/2602.05728 — 2 LLM calls max for multi-hop questions
- AutothinkRAG (Mar 2026) — arxiv.org/abs/2603.05551 — Automatic complexity detection adjusts retrieval depth
- RT-RAG (Jan 2026) — arxiv.org/abs/2601.11255 — Reasoning trees for multi-hop questions
- VoiceAgentRAG (Mar 2026) — arxiv.org/abs/2603.02206 — Dual-agent with semantic cache for sub-ms lookup
- LiveVectorLake (Jan 2026) — arxiv.org/abs/2601.05270 — Temporal queries: "what's true now" vs "what was true then"
- Conversational QA Comparison (Feb 2026) — arxiv.org/abs/2602.09552 — Dialogue history matters more than retrieval method
- Secure RAG Survey (Mar 2026) — arxiv.org/abs/2603.21654 — Security for production RAG

## Implementation

### Tool Definitions
```typescript
const chatTools = [
  {
    name: 'semantic_search',
    description: 'Search summaries and reports by meaning. Use for questions about topics, themes, or "what happened with X?"',
    parameters: {
      query: { type: 'string', description: 'Natural language search query' },
      timeRange: { type: 'string', description: 'e.g. "7d", "30d", "2025-01-01 to 2025-01-31"' },
      limit: { type: 'number', default: 10 }
    }
  },
  {
    name: 'keyword_search',
    description: 'Search entities by name, get mention counts, sentiment scores, source breakdown. Use for "how is sentiment on X?" or "how many times was X mentioned?"',
    parameters: {
      entity: { type: 'string', description: 'Entity name' },
      timeRange: { type: 'string' },
    }
  },
  {
    name: 'read_raw',
    description: 'Read original source message by ID. Use to verify claims or get exact quotes.',
    parameters: {
      itemId: { type: 'string', description: 'Item ID from a summary citation' }
    }
  }
];
```

### Chat Handler
```typescript
async function handleChat(userMessage: string, history: ChatMessage[]): Promise<string> {
  // Translate if Indonesian (Decision 12)
  const lang = detectLanguage(userMessage);
  if (lang === 'ind') {
    userMessage = await translateToEnglish(userMessage);
  }
  
  const response = await llm.call({
    model: config.models.sonnet,
    system: CHAT_SYSTEM_PROMPT,
    messages: [...history, { role: 'user', content: userMessage }],
    tools: chatTools,
    maxTokens: 2000
  });
  
  // Handle tool calls iteratively
  while (response.toolCalls?.length) {
    const results = await executeToolCalls(response.toolCalls);
    response = await llm.call({
      model: config.models.sonnet,
      system: CHAT_SYSTEM_PROMPT,
      messages: [...history, { role: 'user', content: userMessage }, ...results],
      tools: chatTools,
      maxTokens: 2000
    });
  }
  
  return response.text;
}
```

### Example Queries
- "What happened with SOL?" → Sonnet calls semantic_search("SOL events", "7d")
- "How is sentiment on ETH?" → Sonnet calls keyword_search("ETH", "7d")
- "Compare SOL this week vs last month" → Sonnet calls keyword_search("SOL", "7d") AND keyword_search("SOL", "30d")
- "What is Podders?" → Sonnet answers directly, no tool call
- "Show me the original message about the exploit" → Sonnet calls read_raw(itemId)

### Cost Impact
Same as current chat cost (~$0.15-0.30/day). Tool calls don't add LLM cost — they're just function executions between Sonnet turns.

## Upgrade Path
- **v2:** Add semantic cache (VoiceAgentRAG pattern). Cache frequent queries and their results. "What happened with BTC?" asked 5 times/day → cache after first answer, serve from cache for 1 hour.
- **v2:** Add temporal query support (LiveVectorLake). "What was sentiment on ETH last January?" searches time-scoped embeddings, not just recency-ranked.
- **v3:** Add a `web_search` tool for questions outside our data. "What is restaking?" → Sonnet searches the web. Only if explicitly enabled by user.
