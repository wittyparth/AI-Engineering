# Evaluation-First Philosophy

## The Mantra

> **If you can't measure it, you can't ship it.**
> **If you can't measure it, you can't improve it.**
> **If you can't measure it, you don't know if it broke.**

## Why Evaluation is THE Senior Skill

Every AI company interview asks about evaluation. Why? Because:
- Model outputs are probabilistic (not deterministic like traditional code)
- A prompt change that helps 90% of users might hurt 10%
- Model upgrades (GPT-4o → GPT-5) can regress on YOUR specific use case
- You can't observe quality just by looking at logs

## What to Evaluate at Each Layer

| Layer | What to Measure | How |
|-------|----------------|-----|
| LLM Output | Accuracy, faithfulness, relevance | LLM-as-judge, human eval, RAGAS |
| Retrieval | Precision, recall, MRR | Labeled test set |
| Agent | Task completion rate, tool call accuracy | Trajectory evaluation |
| System | Latency (P50, P95, P99), cost per query | Prometheus, CloudWatch |
| User | Satisfaction, retention, task success | A/B testing, analytics |

## The Evaluation Pipeline

```python
# EVERY phase in this course has an eval step
class EvalPipeline:
    def __init__(self, test_cases: list[TestCase]):
        self.test_cases = test_cases
    
    def run(self, system) -> EvalReport:
        results = []
        for case in self.test_cases:
            output = system.run(case.input)
            score = self.score(output, case.expected)
            results.append(Result(input=case.input, output=output, score=score))
        return EvalReport(
            accuracy=mean(r.score for r in results),
            failures=[r for r in results if r.score < THRESHOLD],
            latency_p95=percentile([r.latency for r in results], 95),
        )
```

## Cost of NOT Evaluating

| Scenario | Without Eval | With Eval |
|----------|-------------|-----------|
| Prompt change that breaks things | Users complain first | Regression test catches it |
| Model deprecation | Prod breaks, scramble to fix | Detected in staging |
| New feature | Ship and pray | A/B test validates |
| Cost regression | $10K surprise bill | Alert at $5K threshold |

## 🔴 Senior: Evaluation is Infrastructure, Not a Project

Juniors build an eval once and never update it. Seniors build eval pipelines that:
- Run on every PR (CI gate)
- Track metrics over time (dashboard)
- Alert on regression (PagerDuty/Slack)
- Get updated as new failure modes are discovered

## Drill

Take the system from your last AI project. Define:
1. 3 metrics you should have been tracking
2. 10 test cases that would have caught production bugs
3. An alert threshold for each metric
