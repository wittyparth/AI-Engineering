# Langfuse / LangSmith

## Why Observability Platforms

Manual logging is fragile. Langfuse and LangSmith give you:
- Automatic trace capture
- Step-by-step LLM call visualization
- Token usage and cost tracking
- Evaluation integration
- Session-level view

## Langfuse Integration

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

langfuse = Langfuse()

@observe(name="rag_query")
async def rag_query(query: str):
    # Automatic: trace capture, timing, token counting
    chunks = await retrieve(query)

    # Manual: add any data to the trace
    langfuse_context.update_current_observation(
        input=query,
        metadata={"chunks_count": len(chunks)},
    )

    response = await generate(query, chunks)

    langfuse_context.update_current_observation(
        output=response,
        usage={"input": count_tokens(query), "output": count_tokens(response)},
    )

    # Score the response (0-1)
    langfuse.score(
        name="response_quality",
        value=0.9,
        comment="Good response with citations",
    )

    return response
```

## Custom Trace for RAG Pipeline

```python
class RAGTracer:
    """Add detailed tracing to every RAG step."""

    def __init__(self):
        self.langfuse = Langfuse()

    async def trace_retrieval(self, query: str, chunks: list):
        trace = self.langfuse.trace(
            name="retrieval",
            input=query,
            metadata={"chunk_count": len(chunks)},
        )
        for i, chunk in enumerate(chunks):
            trace.span(
                name=f"chunk_{i}",
                input=chunk.content[:200],
                metadata={"score": chunk.score, "source": chunk.metadata.get("source")},
            )
        return trace

    async def trace_generation(self, query: str, context: str, response: str):
        self.langfuse.trace(
            name="generation",
            input={"query": query, "context_length": len(context)},
            output=response,
            usage={"input": count_tokens(query + context), "output": count_tokens(response)},
        )
```

## LangSmith (LangChain Ecosystem)

```python
from langsmith import Client

client = Client()

# Create a dataset for testing
dataset = client.create_dataset("rag_eval", description="RAG quality test set")
client.create_examples(
    dataset_id=dataset.id,
    inputs=[{"question": "What is RAG?"}],
    outputs=[{"answer": "RAG stands for Retrieval-Augmented Generation..."}],
)

# Run evaluation
results = client.run_on_dataset(
    dataset_name="rag_eval",
    llm_or_chain_factory=my_rag_chain,
    evaluation=evaluators,
)
```

## 🔴 Senior: Langfuse vs LangSmith

| Feature | Langfuse | LangSmith |
|---------|----------|-----------|
| Open source | Yes | No (proprietary) |
| Self-host | Yes | No |
| Cost | Generous free tier | Pay-per-event |
| LangChain integration | Good | Best |
| Non-LangChain support | Better (decorators) | Requires LangChain |
| Evaluations | Built-in | Built-in |
| Datasets | Yes | Yes |
| Tracing granularity | Auto + manual | Auto with LC |

**Recommendation**: Use Langfuse for non-LangChain projects (most of this course). Use LangSmith if you're heavily invested in LangChain.
