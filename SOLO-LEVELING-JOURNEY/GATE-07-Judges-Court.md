# ⛩️ GATE 07 — Judge's Court (Evaluation & Observability)

**Rank Requirement:** B
**Theme:** Measuring LLM quality, building evaluation pipelines, observability in production
**Total Days:** 8 Dungeons + 1 Boss Battle
**Core Skills:** Evaluation metrics, eval datasets, LLM-as-judge, observability, A/B testing

---

## Dungeon 1 — Why Evaluation Matters

**Day 1 | Concept:** If you're not evaluating, you're guessing. LLMs are stochastic — you need systematic quality measurement.

**The Eval Imperative:**
- A model update can silently degrade quality
- Prompt changes have unpredictable effects
- Different use cases need different quality criteria
- Users notice degradation before metrics do

**Levels of Evaluation:**
```
1. Subjective: "Feels right" — dangerous but common
2. Heuristic: Exact match, F1, BLEU — cheap but limited
3. Model-based: LLM-as-judge — better but more expensive
4. Human: Gold standard but slow and expensive
5. Production: User feedback, A/B testing — ultimate truth
```

**🗡️ Dungeon Mastery:** Take 5 responses from an LLM. Rate them subjectively. Then create 5 heuristic eval criteria. Notice the gap between "feels right" and actual metrics.

**💻 Code — Eval Scorecard:**
```python
from dataclasses import dataclass, field
from typing import Callable

@dataclass
class EvalResult:
    name: str
    score: float    # 0.0 to 1.0
    passed: bool
    details: str = ""

@dataclass
class EvalScorecard:
    results: list[EvalResult] = field(default_factory=list)

    def add(self, result: EvalResult):
        self.results.append(result)

    def summary(self) -> dict:
        scores = [r.score for r in self.results]
        return {
            "total_evals": len(self.results),
            "passed": sum(1 for r in self.results if r.passed),
            "failed": sum(1 for r in self.results if not r.passed),
            "average_score": sum(scores) / len(scores) if scores else 0.0,
            "pass_rate": sum(1 for r in self.results if r.passed) / len(self.results) if self.results else 0.0,
        }

    def print(self):
        s = self.summary()
        print(f"Eval Scorecard: {s['passed']}/{s['total_evals']} passed ({s['pass_rate']:.0%})")
        for r in self.results:
            status = "✅" if r.passed else "❌"
            print(f"  {status} {r.name}: {r.score:.2f} — {r.details}")
```

**👹 Boss Lesson:** No eval system = flying blind. Even a bad eval system is better than none.

---

## Dungeon 2 — Classification Metrics

**Day 2 | Concept:** For classification tasks (sentiment, spam, intent), use precision, recall, F1, and confusion matrices.

**Metrics:**
```
Accuracy = (TP + TN) / (TP + TN + FP + FN)
Precision = TP / (TP + FP) — "When we say yes, how often are we right?"
Recall = TP / (TP + FN) — "Of all actual yeses, how many did we catch?"
F1 = 2 * (Precision * Recall) / (Precision + Recall)
```

**🗡️ Dungeon Mastery:** Generate 50 test classifications. Compute accuracy, precision, recall, and F1. Identify which metric matters most for a spam filter vs a medical diagnosis system.

**💻 Code — Classification Eval:**
```python
import numpy as np
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    confusion_matrix, classification_report,
)

class ClassificationEval:
    def __init__(self, y_true: list[str], y_pred: list[str]):
        self.y_true = y_true
        self.y_pred = y_pred
        self.labels = sorted(set(y_true + y_pred))

    def metrics(self) -> dict:
        return {
            "accuracy": accuracy_score(self.y_true, self.y_pred),
            "precision": precision_score(self.y_true, self.y_pred, average="weighted", zero_division=0),
            "recall": recall_score(self.y_true, self.y_pred, average="weighted", zero_division=0),
            "f1": f1_score(self.y_true, self.y_pred, average="weighted", zero_division=0),
        }

    def per_class_metrics(self) -> dict:
        return classification_report(self.y_true, self.y_pred, output_dict=True, zero_division=0)

    def confusion(self) -> np.ndarray:
        return confusion_matrix(self.y_true, self.y_pred)


# Example: Evaluating sentiment classification
y_true = ["positive", "negative", "neutral", "positive", "negative"]
y_pred = ["positive", "negative", "positive", "positive", "neutral"]

eval = ClassificationEval(y_true, y_pred)
print(eval.metrics())
```

**👹 Boss Lesson:** Accuracy lies. When classes are imbalanced (95% spam, 5% not), 95% accuracy is meaningless. Use precision, recall, F1.

---

## Dungeon 3 — Generation Metrics

**Day 3 | Concept:** For generation tasks (summarization, translation, Q&A), metrics like BLEU, ROUGE, BERTScore measure output quality against references.

**Metrics:**
| Metric | What it Measures | Range |
|--------|-----------------|-------|
| BLEU | N-gram overlap with reference | [0, 1] — punishes short output |
| ROUGE-L | Longest common subsequence | [0, 1] — good for summaries |
| METEOR | Synonym-aware matching | [0, 1] — better correlation |
| BERTScore | Semantic similarity via embeddings | [0, 1] — best correlation |
| Perplexity | How "surprised" is the model | Lower = better |

**🗡️ Dungeon Mastery:** Take a source text and 3 different summaries. Compute ROUGE-L and BERTScore for each. Which summary scores best? Does it match human preference?

**💻 Code — Generation Metrics:**
```python
from rouge_score import rouge_scorer
import bert_score

class GenerationEval:
    def __init__(self):
        self.rouge = rouge_scorer.RougeScorer(['rouge1', 'rouge2', 'rougeL'], use_stemmer=True)

    def rouge_scores(self, reference: str, candidate: str) -> dict:
        scores = self.rouge.score(reference, candidate)
        return {
            "rouge1": scores['rouge1'].fmeasure,
            "rouge2": scores['rouge2'].fmeasure,
            "rougeL": scores['rougeL'].fmeasure,
        }

    def bert_score(self, references: list[str], candidates: list[str]) -> dict:
        P, R, F1 = bert_score.score(candidates, references, lang="en", verbose=False)
        return {
            "precision": P.mean().item(),
            "recall": R.mean().item(),
            "f1": F1.mean().item(),
        }

    def exact_match(self, reference: str, candidate: str) -> bool:
        return reference.strip().lower() == candidate.strip().lower()

    def contains_answer(self, reference: str, candidate: str) -> bool:
        return reference.strip().lower() in candidate.strip().lower()


# Example
ref = "The quick brown fox jumps over the lazy dog"
cand = "A fast brown fox jumped over a sleepy dog"

eval = GenerationEval()
print(eval.rouge_scores(ref, cand))
```

**👹 Boss Lesson:** BLEU/ROUGE measure n-gram overlap, not semantics. A high BLEU doesn't mean a good answer. Always pair with semantic metrics like BERTScore.

---

## Dungeon 4 — LLM-as-Judge

**Day 4 | Concept:** Use an LLM to evaluate another LLM's output. This is the most practical eval method for open-ended tasks.

**Judge Types:**
| Type | How | Cost |
|------|-----|------|
| Simple Judge | "Rate 1-5 for correctness" | Cheap |
| Rubric Judge | Score against predefined criteria | Medium |
| Pairwise Judge | "Which response is better?" | Medium |
| Multi-dimension | Score across N axes (correctness, tone, conciseness) | Expensive |
| Reference-based | Compare against gold answer | Medium |

**🗡️ Dungeon Mastery:** Write 3 candidate responses for "Explain transformers in 3 sentences." Use GPT-4 as a judge with a rubric. Score each response. Do you agree with the judge?

**💻 Code — LLM Judge:**
```python
class LLMJudge:
    def __init__(self, judge_llm):
        self.judge = judge_llm

    def rate_correctness(self, question: str, answer: str, rubric: str = "") -> dict:
        prompt = f"""Evaluate this answer for correctness.

Question: {question}
Answer: {answer}
{rubric}

Rate 1-5 on:
- Correctness: Is the answer factually accurate?
- Completeness: Does it fully answer the question?
- Clarity: Is it well-structured and easy to understand?

Respond ONLY with JSON:
{{"correctness": 5, "completeness": 4, "clarity": 5, "reasoning": "brief"}}"""

        result = self.judge.generate(prompt)
        import json
        try:
            start = result.find("{")
            end = result.rfind("}") + 1
            return json.loads(result[start:end])
        except:
            return {"error": "Failed to parse judge response"}

    def pairwise(self, question: str, answer_a: str, answer_b: str) -> dict:
        prompt = f"""Compare these two answers.

Question: {question}

Answer A: {answer_a}

Answer B: {answer_b}

Which is better and why? Respond in JSON:
{{"winner": "A" or "B", "reasoning": "explanation", "a_score": 0-10, "b_score": 0-10}}"""

        result = self.judge.generate(prompt)
        import json
        try:
            start = result.find("{")
            end = result.rfind("}") + 1
            return json.loads(result[start:end])
        except:
            return {"error": "Parse failure"}

    def evaluate_multi_dimension(self, answer: str, criteria: list[str]) -> dict:
        scores = {}
        for criterion in criteria:
            prompt = f"""Score this answer on "{criterion}" from 1-5. Only return the number.
Answer: {answer}"""
            try:
                scores[criterion] = int(self.judge.generate(prompt).strip())
            except:
                scores[criterion] = 0
        return scores
```

**👹 Boss Lesson:** LLM-as-judge is not perfect, but it's the best practical option. Use a STRONGER model as judge (e.g., GPT-4 judges GPT-3.5).

---

## Dungeon 5 — Eval Dataset Creation

**Day 5 | Concept:** An eval system is only as good as its dataset. Build systematic, representative, and evolving test sets.

**Dataset Types:**
| Type | Examples | Purpose |
|------|----------|---------|
| Golden | 50-100 hand-curated Q&A pairs | Core quality gate |
| Edge cases | Ambiguous, adversarial inputs | Stress testing |
| Regression | Past failures that must not recur | Prevent backsliding |
| Production | Sampled from real user queries | Realistic performance |

**Best Practices:**
- Cover all intents your system handles
- Include "impossible" questions (test refusal)
- Version your eval dataset
- Add new cases for every production bug

**💻 Code — Eval Dataset Manager:**
```python
import json
from datetime import datetime
from pathlib import Path

@dataclass
class EvalCase:
    id: str
    input: str
    expected_output: str
    category: str  # "qa", "summarization", "extraction", "refusal"
    tags: list[str] = field(default_factory=list)
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())

class EvalDataset:
    def __init__(self, path: str = "eval_dataset.json"):
        self.path = Path(path)
        self.cases: list[EvalCase] = self._load()

    def _load(self) -> list[EvalCase]:
        if self.path.exists():
            data = json.loads(self.path.read_text())
            return [EvalCase(**c) for c in data]
        return []

    def _save(self):
        self.path.write_text(json.dumps(
            [c.__dict__ for c in self.cases], indent=2))

    def add(self, case: EvalCase):
        self.cases.append(case)
        self._save()

    def add_from_failure(self, user_input: str, expected: str, category: str = "regression"):
        """Add a production failure as a regression test."""
        case_id = f"regression-{datetime.now().strftime('%Y%m%d-%H%M%S')}"
        self.add(EvalCase(
            id=case_id, input=user_input,
            expected_output=expected, category=category,
            tags=["regression"],
        ))

    def get_by_category(self, category: str) -> list[EvalCase]:
        return [c for c in self.cases if c.category == category]

    def get_by_tag(self, tag: str) -> list[EvalCase]:
        return [c for c in self.cases if tag in c.tags]

    def run_all(self, system_fn) -> dict:
        """Run all eval cases through system_fn."""
        results = {"passed": 0, "failed": 0, "cases": []}
        for case in self.cases:
            output = system_fn(case.input)
            passed = self._check(case.expected_output, output)
            results["cases"].append({"case": case.id, "passed": passed})
            if passed:
                results["passed"] += 1
            else:
                results["failed"] += 1
        return results

    def _check(self, expected: str, actual: str) -> bool:
        return expected.strip().lower() in actual.strip().lower()
```

**🗡️ Dungeon Mastery:** Create a mini eval dataset with 10 cases (5 normal Q&A, 3 edge cases, 2 refusal tests). Run it against your RAG system. Analyze failures.

**👹 Boss Lesson:** Your eval dataset is your system's constitution. Maintain it like code — versioned, reviewed, and always growing.

---

## Dungeon 6 — Observability & Monitoring

**Day 6 | Concept:** Eval happens offline. Observability happens in production, in real-time, on every request.

**What to Monitor:**
| Signal | What It Reveals | How |
|--------|-----------------|-----|
| Latency p50/p95/p99 | Slowdowns, regressions | Histogram over time |
| Token usage | Cost spikes | Sum per request |
| Error rate | System failures | % of requests with errors |
| User feedback | Quality perception | Thumbs up/down ratio |
| Drift detection | Model behavior changes | Compare recent vs baseline |
| Cache hit rate | Efficiency | % of requests served from cache |

**🗡️ Dungeon Mastery:** Add observability to a RAG pipeline. Log every request with latency, context used, and answer. Build a simple dashboard (or just print stats).

**💻 Code — Observability System:**
```python
from collections import defaultdict
from datetime import datetime, timedelta
import threading
import json

class MetricsCollector:
    def __init__(self, window_minutes: int = 60):
        self.window = timedelta(minutes=window_minutes)
        self.lock = threading.Lock()
        self.requests: list[dict] = []

    def record(self, request: dict):
        with self.lock:
            self.requests.append({
                **request,
                "timestamp": datetime.now().isoformat(),
            })
            self._trim()

    def _trim(self):
        cutoff = datetime.now() - self.window
        self.requests = [r for r in self.requests
                         if datetime.fromisoformat(r["timestamp"]) > cutoff]

    def latency_stats(self) -> dict:
        with self.lock:
            latencies = sorted([r.get("latency_ms", 0) for r in self.requests])
            if not latencies:
                return {}
            n = len(latencies)
            return {
                "p50": latencies[n // 2],
                "p95": latencies[int(n * 0.95)],
                "p99": latencies[int(n * 0.99)],
                "avg": sum(latencies) / n,
                "count": n,
            }

    def error_rate(self) -> float:
        with self.lock:
            if not self.requests:
                return 0.0
            errors = sum(1 for r in self.requests if r.get("error"))
            return errors / len(self.requests)

    def top_queries(self, n: int = 5) -> list[tuple[str, int]]:
        with self.lock:
            queries = defaultdict(int)
            for r in self.requests:
                queries[r.get("query", "")] += 1
            return sorted(queries.items(), key=lambda x: -x[1])[:n]

    def report(self) -> dict:
        return {
            "total_requests": len(self.requests),
            "latency": self.latency_stats(),
            "error_rate": self.error_rate(),
            "top_queries": self.top_queries(),
            "window_minutes": self.window.total_seconds() / 60,
        }


# Middleware pattern for easy integration:
class ObservabilityMiddleware:
    """Wrap any function with observability."""
    def __init__(self, fn, collector: MetricsCollector):
        self.fn = fn
        self.collector = collector

    def __call__(self, query: str, **kwargs):
        import time
        start = time.time()
        try:
            result = self.fn(query, **kwargs)
            self.collector.record({
                "query": query,
                "latency_ms": (time.time() - start) * 1000,
                "error": False,
                "tokens_used": result.get("tokens", 0),
            })
            return result
        except Exception as e:
            self.collector.record({
                "query": query,
                "latency_ms": (time.time() - start) * 1000,
                "error": True,
                "error_type": type(e).__name__,
            })
            raise
```

**👹 Boss Lesson:** If it's not monitored, it doesn't exist in production. Add observability before you deploy — not after the first user complaint.

---

## Dungeon 7 — A/B Testing & Experimentation

**Day 7 | Concept:** The only way to know if a change is actually better is to test it against the current system with real users.

**A/B Testing Framework:**
```
1. Hypothesis: "Using CoT in prompts increases answer completeness"
2. Variants: Control (no CoT) vs Treatment (CoT)
3. Metrics: Completeness score, user satisfaction, latency
4. Sample size: N users, randomly split 50/50
5. Run: Collect data for minimum 1 week
6. Analyze: Statistical significance, p < 0.05
```

**🗡️ Dungeon Mastery:** Design an A/B test: Compare 2 chunk sizes (256 vs 512). Define metrics, sample size, and success criteria. Mock the experiment and analyze results.

**💻 Code — A/B Test Framework:**
```python
import random
import statistics
from datetime import datetime
from dataclasses import dataclass, field

@dataclass
class ABTestConfig:
    name: str
    control_name: str = "control"
    treatment_name: str = "treatment"
    split_ratio: float = 0.5  # 50% to treatment
    minimum_sample: int = 100

class ABTest:
    def __init__(self, config: ABTestConfig):
        self.config = config
        self.results = {"control": [], "treatment": []}

    def assign(self, user_id: str) -> str:
        """Assign user to variant deterministically."""
        if random.random() < self.config.split_ratio:
            return self.config.treatment_name
        return self.config.control_name

    def record(self, variant: str, metric_value: float):
        self.results[variant].append(metric_value)

    def analyze(self) -> dict:
        c = self.results["control"]
        t = self.results["treatment"]

        def stats(data):
            return {
                "mean": statistics.mean(data) if data else 0,
                "std": statistics.stdev(data) if len(data) > 1 else 0,
                "n": len(data),
            }

        c_stats = stats(c)
        t_stats = stats(t)

        # Simple uplift
        uplift = ((t_stats["mean"] - c_stats["mean"]) / c_stats["mean"] * 100
                  if c_stats["mean"] else 0)

        return {
            "experiment": self.config.name,
            "control": c_stats,
            "treatment": t_stats,
            "uplift_pct": uplift,
            "winner": "treatment" if uplift > 0 else "control",
            "adequate_sample": (
                c_stats["n"] >= self.config.minimum_sample // 2 and
                t_stats["n"] >= self.config.minimum_sample // 2
            ),
        }

# Example: A/B test chunk size
# config = ABTestConfig(name="Chunk size: 256 vs 512", minimum_sample=200)
# ab = ABTest(config)
# for user_query in production_queries:
#     variant = ab.assign(user_query["user_id"])
#     result = pipeline(query=user_query["text"], chunk_size=256 if variant=="control" else 512)
#     ab.record(variant, result["completeness_score"])
# print(ab.analyze())
```

**👹 Boss Lesson:** Intuition is wrong more often than you think. A/B test every change — prompts, chunk sizes, models, temperatures.

---

## Dungeon 8 — Continuous Evaluation Pipeline

**Day 8 | Concept:** Eval should be automated, continuous, and part of your CI/CD pipeline — not a manual monthly review.

**Pipeline Flow:**
```
PR → Run eval dataset → Score regression tests → Block if scores drop
Deploy → Monitor production metrics → Log failures → Add to eval dataset
Weekly → Report quality trends → Flag regressions
Monthly → Review full eval suite → Update golden test set
```

**🗡️ Dungeon Mastery:** Build a CI eval script. It should: load the eval dataset, run every case through the system, compute pass rate, fail the build if pass rate drops below 80%.

**💻 Code — Eval CI Pipeline:**
```python
import sys
import json
from pathlib import Path

class EvalPipeline:
    def __init__(self, dataset_path: str, threshold: float = 0.8):
        self.dataset = EvalDataset(dataset_path)
        self.threshold = threshold

    def run(self, system_fn) -> dict:
        results = self.dataset.run_all(system_fn)
        pass_rate = results["passed"] / len(results["cases"]) if results["cases"] else 1.0

        report = {
            "timestamp": datetime.now().isoformat(),
            "total_cases": len(results["cases"]),
            "passed": results["passed"],
            "failed": results["failed"],
            "pass_rate": pass_rate,
            "threshold": self.threshold,
            "status": "PASSED" if pass_rate >= self.threshold else "FAILED",
            "failures": [
                c for c in results["cases"] if not c["passed"]
            ],
        }

        report_path = Path("eval_report.json")
        report_path.write_text(json.dumps(report, indent=2))

        print(f"Eval: {report['passed']}/{report['total_cases']} passed "
              f"({pass_rate:.1%}, threshold={self.threshold:.0%})")
        print(f"Status: {report['status']}")

        if report["status"] == "FAILED":
            print(f"FAILURES:")
            for f in report["failures"]:
                print(f"  - {f['case']}")
            sys.exit(1)

        return report

# Usage in CI:
# pipeline = EvalPipeline("eval_dataset.json", threshold=0.85)
# pipeline.run(my_system)
```

**👹 Boss Lesson:** If evals aren't automated, they won't be run. If they're not run, they're useless.

---

## ⚔️ BOSS BATTLE: The Evaluation Platform

**Objective:** Build an evaluation system that:
1. Contains 20+ eval cases across 3+ categories
2. Runs automated eval on any LLM output
3. Computes 3+ different metrics (correctness, completeness, format compliance)
4. Uses LLM-as-judge with a rubric for at least one metric
5. Logs every eval run with timestamps and scores
6. Generates a human-readable report

**🗡️ Victory Conditions:**
- 20+ eval cases covering diverse scenarios
- Automated eval runs in under 30 seconds
- Metrics computed and logged
- LLM-as-judge produces consistent scores
- Report shows trends over multiple runs
- Regression tests prevent quality backsliding

**🏆 Rewards:**
- +500 XP
- B Rank → B+ Rank
- Skill Unlock: **Quality Sense** — Instantly identify weak spots in any LLM system
- Title: **The Judge**

---

*"What gets measured gets managed. What gets evaluated gets improved."*
