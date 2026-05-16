# HyDE & Multi-Query

## HyDE (Hypothetical Document Embeddings)

Instead of embedding the query, embed a hypothetical ideal document:

```python
def hyde_retrieve(query: str, llm, embedder, vector_db, k: int = 10):
    hypothetical_doc = llm.generate(
        f"Write a document that perfectly answers: {query}"
    )
    hyde_emb = embedder.embed(hypothetical_doc)
    return vector_db.search(hyde_emb, k=k)
```

**Why it works**: The query "What's the capital of France?" and "Tell me about Paris" embed differently even though they're about the same thing. A hypothetical document about "Paris is the capital of France..." embeds close to both.

**When to use**: Queries are short/ambiguous. Documents are long/descriptive.

**When NOT to use**: Queries are already precise. Documents are short.

## Multi-Query

```python
async def multi_query_retrieve(query: str, llm, embedder, vector_db, k: int = 10):
    variations = await llm.generate(
        f"Generate 5 different versions of this query: {query}"
    )
    queries = [query] + parse_variations(variations)

    # Run all queries in parallel
    tasks = [vector_db.search(embedder.embed(q), k=k) for q in queries]
    results = await asyncio.gather(*tasks)

    # Merge and deduplicate
    all_docs = {}
    for docs in results:
        for doc in docs:
            all_docs[doc.id] = all_docs.get(doc.id, 0) + doc.score

    return sorted(all_docs.items(), key=lambda x: x[1], reverse=True)[:k]
```

## Strategy Comparison

| Strategy | Recall Improvement | Latency | Cost |
|----------|-------------------|---------|------|
| Vanilla | baseline | 1x | 1x |
| HyDE | +5-10% | 1.5x | 1.5x (one extra LLM call) |
| Multi-Query | +10-20% | 1x parallel | 5x (5 embedding calls) |
| Both | +15-25% | 1.5x | 6x |

🔴 **Senior**: Use HyDE for free (it's just an extra LLM call with low temperature). Multi-query is expensive — only use when recall requirements are critical.

## 🔧 Framework Integration: LlamaIndex Workflows

After mastering these raw patterns, you can orchestrate them with LlamaIndex's Workflow engine:
- Define multi-hop retrieval as event-driven steps
- Chain HyDE → Multi-Query → Self-RAG in a single workflow
- Add conditional branching based on retrieval quality

See `08-Advanced-RAG-with-LlamaIndex.md` for the framework deep-dive.
