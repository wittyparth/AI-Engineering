# Advanced RAG with LlamaIndex Workflows

## LlamaIndex Workflow Engine

LlamaIndex's workflow system is event-driven — steps react to events, emit events, and pass state.

```python
from llama_index.core.workflow import (
    Workflow, step, Event, StartEvent, StopEvent, Context
)

class RetrieveEvent(Event):
    query: str

class GenerateEvent(Event):
    chunks: list[str]

class RAGWorkflow(Workflow):
    @step
    async def retrieve(self, ctx: Context, ev: StartEvent) -> RetrieveEvent:
        query = ev.query
        retriever = self.get_index().as_retriever(similarity_top_k=5)
        chunks = retriever.retrieve(query)
        ctx.data["chunks"] = chunks
        return RetrieveEvent(query=query)

    @step
    async def rewrite_query(self, ctx: Context, ev: RetrieveEvent) -> RetrieveEvent:
        # Check if we need to rewrite
        if len(ctx.data["chunks"]) < 3:
            llm = OpenAI(model="gpt-4o-mini")
            new_query = llm.predict(
                f"Rewrite this query for better search: {ev.query}"
            )
            return RetrieveEvent(query=new_query)
        return GenerateEvent(chunks=ctx.data["chunks"])

    @step
    async def generate(self, ctx: Context, ev: GenerateEvent) -> StopEvent:
        llm = OpenAI(model="gpt-4o-mini")
        response = llm.predict(
            f"Context: {ev.chunks}\nQuery: {ctx.data.get('original_query')}"
        )
        return StopEvent(result=response)

# Run
w = RAGWorkflow()
result = await w.run(query="What is RAG?")
```

## Multi-Step Retrieval with LlamaIndex

```python
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core.response_synthesizers import TreeSummarize

class MultiHopRAG:
    """Answer multi-part questions by decomposing into sub-queries."""

    async def answer(self, question: str) -> str:
        # 1. Decompose question
        sub_questions = await self.decompose(question)

        # 2. Retrieve for each sub-question
        intermediate_results = []
        for sub_q in sub_questions:
            nodes = self.retriever.retrieve(sub_q)
            intermediate_results.append(nodes)

        # 3. Combine results
        combined = self._combine(intermediate_results)

        # 4. Synthesize final answer
        synthesizer = TreeSummarize(llm=self.llm)
        return await synthesizer.synthesize(question, combined)
```

## LlamaIndex vs LangChain for Advanced RAG

| Advanced Pattern | LlamaIndex | LangChain |
|-----------------|-----------|-----------|
| Multi-hop retrieval | Workflow steps | LangGraph DAG |
| Query decomposition | Built-in | Custom chain |
| Agentic RAG | RouterQueryEngine | LangGraph agent |
| Graph RAG | KnowledgeGraphIndex | Neo4j integration |
| Evaluation | LlamaIndex evals | LangSmith + RAGAS |
| Workflow state | Event-driven Context | StateGraph state |
