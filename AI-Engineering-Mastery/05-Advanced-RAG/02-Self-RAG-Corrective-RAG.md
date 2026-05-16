# Self-RAG & Corrective RAG

## Self-RAG

The RAG system evaluates its OWN retrieval and generation quality:

```python
class SelfRAG:
    def __init__(self, retriever, generator, evaluator):
        self.retriever = retriever
        self.generator = generator
        self.evaluator = evaluator  # LLM-as-judge

    async def answer(self, query: str) -> dict:
        # 1. Retrieve
        chunks = await self.retriever.retrieve(query)
        # 2. Generate draft
        draft = await self.generator.generate(query, chunks)
        # 3. Evaluate
        eval_result = await self.evaluator.evaluate(query, draft, chunks)
        # 4. Decide: accept, revise, or re-retrieve
        if eval_result["faithfulness"] < 0.7:
            return await self._revise(query, draft, chunks)
        elif eval_result["completeness"] < 0.6:
            return await self._retrieve_more(query)
        return draft
```

## Corrective RAG (CRAG)

When retrieval quality is low, correct it before generation:

```python
class CorrectiveRAG:
    def __init__(self, retriever, generator, relevance_evaluator):
        self.retriever = retriever
        self.generator = generator
        self.evaluator = relevance_evaluator

    async def answer(self, query: str) -> str:
        chunks = await self.retriever.retrieve(query, k=10)
        # Evaluate relevance of each chunk
        relevant = []
        irrelevant = []
        for chunk in chunks:
            score = await self.evaluator.relevance(query, chunk)
            if score > 0.5:
                relevant.append(chunk)
            else:
                irrelevant.append(chunk)

        if not relevant:
            # All retrieval failed → use web search or alternate source
            web_results = await self.web_search(query)
            return await self.generator.generate(query, web_results)

        if len(relevant) < 3:
            # Not enough good chunks → expand with knowledge graph
            expanded = await self.knowledge_graph_expand(relevant)
            relevant.extend(expanded)

        return await self.generator.generate(query, relevant)
```

## 🔴 Senior: When Self-RAG is Worth It

Self-RAG adds 30-50% latency and cost. Use it when:
- Queries are high-stakes (customer support, medical, legal)
- Retrieval quality is unpredictable
- The cost of a bad answer is high

Do NOT use it for: simple Q&A, well-structured knowledge bases, low-stakes applications.
