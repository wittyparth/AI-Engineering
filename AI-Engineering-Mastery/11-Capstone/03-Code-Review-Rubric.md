# Code Review Rubric

## General (All Projects)

| Criteria | Excellent (3) | Passing (2) | Failing (1) |
|----------|--------------|-------------|-------------|
| **Architecture** | Clean separation, single responsibility, easy to extend | Reasonable structure, some mixing of concerns | Everything in one file/module |
| **Error Handling** | Every error type handled, graceful degradation, informative messages | Main errors handled, some edge cases missed | Silent failures, uncaught exceptions |
| **Testing** | Unit + integration + eval tests, >80% coverage | Unit tests for critical paths | No tests |
| **Documentation** | README explains everything, runnable with one command | Basic README | No documentation |
| **Configuration** | Environment-based, typed settings, .env.example | Hardcoded defaults with env override | Hardcoded values |
| **Type Safety** | Full type hints, Pydantic models | Partial type hints | No type hints |
| **Performance** | Handles concurrent requests, async where appropriate | Sync but functional | Blocks on I/O |
| **Security** | Input validation, rate limiting, no secrets in code | Basic validation | No security measures |

## AI-Specific Criteria

| Criteria | Excellent | Passing | Failing |
|----------|-----------|---------|---------|
| **LLM Integration** | Multi-provider, fallback, retry | Single provider with retry | No fallback |
| **Prompt Engineering** | Versioned, tested, templates | Hardcoded prompts | Prompt in code |
| **RAG Quality** | Hybrid search, reranking, evals | Basic vector search | Keyword only |
| **Agents** | Stateful, tool validation, error recovery | Basic ReAct loop | Single tool call |
| **Observability** | Full tracing, metrics, dashboard | Logging only | Nothing |
| **Cost Awareness** | Per-query tracking, model routing | Tracked but not optimized | No cost tracking |
| **Evals** | Automated, regression detection | Manual eval | No evals |
| **Security** | Guardrails at every layer, audit log | Basic injection check | No security |

## Code Review Process

1. **Self-review**: Run through the rubric yourself
2. **Peer review**: Have someone else review your code
3. **Rubric scoring**: Score each criterion
4. **Improvement plan**: For any "Failing" scores, write a plan
5. **Final pass**: Re-score after improvements

## Minimum Passing Score

- No "Failing" scores
- At least 70% "Excellent" scores in AI-specific criteria
- System actually runs (not just code)
