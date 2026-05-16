# Retrieval-Augmented Generation

## The Core Pattern

```
Query → Retrieve relevant chunks → Build prompt with context → Generate answer
```

## Prompt Construction

```python
def build_rag_prompt(query: str, chunks: list[Chunk]) -> str:
    context = "\n\n".join([
        f"[Source: {c.metadata.get('source', 'unknown')}]\n{c.content}"
        for c in chunks
    ])

    return f"""You are a helpful assistant. Answer the question based ONLY on the provided context.
If the context doesn't contain enough information, say "I don't have enough information to answer this."

Context:
{context}

Question: {query}

Answer:"""
```

## From Scratch Implementation

```python
class RAGPipeline:
    def __init__(self, vector_db, embedder, llm_client, top_k: int = 5):
        self.db = vector_db
        self.embedder = embedder
        self.llm = llm_client
        self.top_k = top_k

    async def answer(self, query: str) -> str:
        # 1. Embed query
        query_emb = await self.embedder.embed(query)

        # 2. Retrieve relevant chunks
        chunks = await self.db.search(query_emb, k=self.top_k)

        # 3. Build prompt
        prompt = build_rag_prompt(query, chunks)

        # 4. Generate answer
        response = await self.llm.generate(prompt)

        return response
```

## 🔴 Senior: The Vanilla RAG Trap

Vanilla RAG (embed query → retrieve → generate) fails when:
- The query is complex (multi-part)
- The query uses different terminology than the documents
- Relevant info spans multiple chunks
- The retrieved chunks are irrelevant (irretrievable)

This is why Phase 5 (Advanced RAG) exists. But master the basics first.
