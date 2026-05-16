# Drill: Chunking Optimization

**Time**: 30 min  **Difficulty**: Medium

## Task

Compare 5 chunking strategies on YOUR data:

```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,
    TokenTextSplitter,
    MarkdownHeaderTextSplitter,
    PythonCodeTextSplitter,
)

strategies = {
    "recursive_250": RecursiveCharacterTextSplitter(chunk_size=250, chunk_overlap=25),
    "recursive_500": RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50),
    "recursive_1000": RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100),
    "token_500": TokenTextSplitter(chunk_size=500, chunk_overlap=50),
    "semantic": SemanticChunker(...),  # from earlier concept file
}

def evaluate_chunking(chunks: list[str], queries: list[str], relevant_chunks: list[set]):
    """Measure retrieval quality for each chunking strategy."""
    rag = RAGFromScratch(chunks)  # from previous drill
    recall_scores = []

    for query, relevant in zip(queries, relevant_chunks):
        retrieved = set(rag.retrieve(query, k=5))
        recall = len(retrieved & relevant) / len(relevant)
        recall_scores.append(recall)

    return {
        "num_chunks": len(chunks),
        "avg_chunk_length": np.mean([len(c) for c in chunks]),
        "recall@5": np.mean(recall_scores),
        "total_tokens": sum(len(c.split()) for c in chunks),
    }

# Run all strategies
for name, strategy in strategies.items():
    chunks = strategy.split_text(your_documents)
    results = evaluate_chunking(chunks, test_queries, ground_truth)
    print(f"{name}: {results}")
```

## Report

For each strategy:
- Number of chunks created
- Average chunk length
- Recall@5
- Context coherence (manual: does each chunk make sense alone?)
- Recommendation
