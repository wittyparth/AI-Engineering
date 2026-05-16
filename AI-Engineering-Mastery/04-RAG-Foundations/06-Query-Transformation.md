# Query Transformation

## The Problem

Users don't write optimal search queries. They ask questions that don't match the language in your documents.

## Query Rewriting

```python
def rewrite_query(original_query: str, llm_client) -> str:
    """Rewrite user query into an effective search query."""
    prompt = f"""Rewrite the following question into an effective search query
that would find relevant information in a technical documentation database.

Original: {original_query}
Search query:"""
    return llm_client.generate(prompt)

# Example:
# Original: "How do I fix the login error?"
# Rewritten: "login error troubleshooting authentication failure resolution"
```

## Query Expansion

```python
def expand_query(query: str, llm_client) -> list[str]:
    """Generate multiple query variations for better recall."""
    prompt = f"""Generate 3 different versions of this search query
to maximize relevant results. Each version should use different terminology.

Query: {query}
Versions:"""
    response = llm_client.generate(prompt)
    versions = parse_list(response)
    return [query] + versions

# Example:
# Original: "How to deploy FastAPI"
# Versions: ["FastAPI deployment guide", "deploying FastAPI applications", "FastAPI production deployment"]
```

## Query Decomposition

```python
def decompose_query(complex_query: str, llm_client) -> list[str]:
    """Break multi-part questions into individual sub-queries."""
    prompt = f"""Break this complex question into 2-3 simpler sub-questions
that can be answered independently.

Question: {complex_query}
Sub-questions:"""
    response = llm_client.generate(prompt)
    return parse_list(response)

# Example:
# "Compare the performance and cost of GPT-4o and Claude 4"
# → ["What is the performance of GPT-4o?", "What is the cost of GPT-4o?",
#    "What is the performance of Claude 4?", "What is the cost of Claude 4?"]
```

## HyDE (Hypothetical Document Embeddings)

```python
def hyde(query: str, llm_client, embedder):
    """Generate a hypothetical document that would answer the query,
    then embed that document instead of the query."""
    prompt = f"""Write a paragraph that answers the following question.
Use factual, technical language.

Question: {query}"""
    hypothetical_doc = llm_client.generate(prompt)
    return embedder.embed(hypothetical_doc)
```

## 🔴 Senior: When Each Strategy Works

| Strategy | Works When | Fails When |
|----------|-----------|------------|
| Rewriting | Query is conversational | Query is already optimized |
| Expansion | Single term queries | Query is very specific |
| Decomposition | Multi-part questions | Simple questions |
| HyDE | Query is abstract | Query is concrete |
