# Debug Challenge: RAG Troubleshooting

**Time**: 30 min  **Difficulty**: Medium

## Scenario

You're given a RAG system that "mostly works" but has these symptoms:

1. **Hallucination**: Answers contain information not in the retrieved context
2. **Irrelevant answers**: Retrieved chunks don't match the question
3. **Missing information**: The right chunk exists but isn't retrieved
4. **Slow responses**: Takes >10s per query
5. **Context overflow**: Prompt exceeds context window

## Your Task

For each symptom:
1. Diagnose the root cause (3+ possible causes per symptom)
2. Propose the fix
3. Say how you'd PREVENT it in the future

## Symptom Analysis Table

| Symptom | Possible Causes | Diagnostic | Fix |
|---------|----------------|------------|-----|
| Hallucination | Poor context, weak system prompt, high temperature | Check faithfulness score, inspect prompt | ... |
| Irrelevant retrieval | Bad embedding, poor chunking, wrong query | Check recall@k, inspect chunks | ... |
| Missing info | Chunk too small, wrong vocabulary, bad index | Manual relevance check | ... |
| Slow | Too many chunks, large prompts, no caching | Profile each stage | ... |
| Context overflow | Too many chunks, large system prompt | Token counting, dynamic limiting | ... |

## Deliverable

Write a troubleshooting guide (1-2 pages) with:
1. Decision tree for RAG debugging
2. Metrics to monitor (and alert thresholds)
3. Common fixes ranked by effort/impact
