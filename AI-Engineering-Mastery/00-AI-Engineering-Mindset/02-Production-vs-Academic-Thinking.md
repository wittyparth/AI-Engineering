# Production vs Academic Thinking

## The Gap

| Dimension | Academic / Tutorial | Production |
|-----------|-------------------|------------|
| Dataset | Clean, curated | Noisy, messy, real user data |
| Metrics | Benchmark accuracy | User satisfaction, cost, latency, reliability |
| Failure mode | Model underperforms | Everything breaks at once |
| Evaluation | Holdout set | Continuous eval + drift detection |
| Focus | Model quality | System quality (model + infra + UX) |
| Timeline | One experiment | Running 24/7 for months |
| Scale | Single machine | 100K+ requests/day |

## Case Study: The 99% Trap

Academic mentality: "Our RAG system achieves 99% accuracy on the test set."

Production reality:
- 99% accuracy means 1 in 100 queries returns bad results
- At 10K queries/day, that's 100 angry users/day
- The 1% failure is NOT random — it's concentrated on the hardest queries (the ones that matter most)
- Users don't care about the 99% that worked; they remember the 1% that failed

## Production-Think Exercises

### Exercise 1: Before you write any code, ask:
```
1. What's the worst that can happen with this feature?
2. How will I know it's working in production?
3. What's my rollback plan?
4. What's the cost per execution?
5. How does this fail gracefully?
```

### Exercise 2: After you build something, ask:
```
1. Does this work at 1x load? 10x? 100x?
2. What happens when the LLM API is down?
3. What happens when input is malicious (prompt injection)?
4. What happens when input is gibberish?
5. How long until this needs maintenance?
```

## 🔴 Key Insight: The "Works on My Machine" Gap

Every AI engineer faces this. The difference between junior and senior is:
- Junior: ships to prod and discovers the gaps through outages
- Senior: anticipates the gaps and builds guardrails before shipping

## Reflection

Take your last AI project. How many of the production questions above did you NOT consider? List them. That's your growth area for this course.
