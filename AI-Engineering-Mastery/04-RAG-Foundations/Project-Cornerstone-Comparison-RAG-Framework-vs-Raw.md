# Cornerstone Project: RAG Framework Comparison

**Time**: 2-3 days  **Difficulty**: Medium  **Portfolio**: Yes

## The Problem

You've built RAG from scratch. Now build the SAME RAG system using LangChain AND LlamaIndex. Compare all three on code quality, flexibility, and performance. By the end, you'll know EXACTLY when to use raw vs framework.

## Requirements

### Must Have
- [ ] Same corpus indexed in all 3 implementations (raw, LangChain, LlamaIndex)
- [ ] Same evaluation dataset (20+ test queries with ground truth)
- [ ] All 3 implementations scored on faithfulness, latency, cost
- [ ] Comparison report with specific tradeoffs called out
- [ ] Streaming responses in all 3

### Comparison Dimensions

| Dimension | Raw Python | LangChain | LlamaIndex |
|-----------|-----------|-----------|------------|
| Lines of code | ___ | ___ | ___ |
| Setup time | ___ | ___ | ___ |
| Faithfulness score | ___ | ___ | ___ |
| Latency p50 | ___ | ___ | ___ |
| Latency p95 | ___ | ___ | ___ |
| Ease of debugging | 1-5 | 1-5 | 1-5 |
| Flexibility | 1-5 | 1-5 | 1-5 |

### Report Structure

1. **Executive summary**: Which approach and why?
2. **Code comparison**: Key code snippets from each
3. **Metrics table**: All 3 side by side
4. **When to use what**: Decision criteria
5. **Personal recommendation**: What you'd use for your next project

## 🔴 Senior: The Real Answer

For a simple RAG: any of the 3 works. The choice depends on:
- **Your team's existing expertise** (most important factor)
- **How custom your retrieval needs to be** (raw for custom, frameworks for standard)
- **What other systems you need to integrate with** (ecosystem matters)
- **Debugging comfort** (raw is easier to trace, frameworks have better tooling)

There is no universal best answer. The best engineer knows all 3 and chooses appropriately.
