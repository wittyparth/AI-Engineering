# 05 — Building Eval Pipelines: CI/CD for AI Systems

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about pipeline architecture, you need to understand why eval pipelines FAIL in practice — and it's usually not for technical reasons.

You've built evaluation datasets (File 04). You've built LLM judges (File 02). You've chosen your metrics (File 03). Now you need to make all of this run automatically — every time someone changes the system prompt, swaps a model, or updates retrieval logic.

This is the CI/CD eval pipeline. It's the GATE that stops regressions from reaching users.

But here's the pattern I see over and over:

```
Month 1: Pipeline takes 5 minutes. Developers love it.
         Every commit gets evaluated. Regressions caught early.
         
Month 2: More test cases added. Pipeline takes 20 minutes.
         Some developers start skipping it. "My change is trivial."
         
Month 3: Cost concerns. Pipeline scaled back. Runs on 50% of commits.
         Two regressions slip through. "The pipeline wouldn't have caught them anyway."
         
Month 4: Pipeline takes 45 minutes. Costs $50/run. Runs nightly, not per-commit.
         Developers ignore its results. "It always fails anyway."
         
Month 5: Pipeline is disabled. "We'll redesign it."
         Never gets redesigned.
         Team is back to manual evaluation.
```

**The pipeline didn't fail because it was technically wrong. It failed because it was SLOW, EXPENSIVE, and DEVELOPERS DIDN'T TRUST IT.**

This module is about building eval pipelines that SURVIVE contact with production — pipelines that developers actually USE because they're fast, cheap, trustworthy, and provide actionable feedback.

---

### 🤔 Discovery Question 1: The 45-Minute CI Problem

Your evaluation pipeline takes 45 minutes to run. It evaluates 500 test cases through your full RAG pipeline + 3 LLM judges per case.

Your team ships 10-15 PRs per day. Each PR triggers a pipeline run. Your CI cluster can handle 4 concurrent runs.

**Math:**
- 15 PRs × 45 min = 11.25 hours of pipeline time per day
- 4 concurrent runners = 2.8 hours of wall-clock CI backlog per day
- PRs queue up. Developers wait 3-4 hours for eval results.
- Developers start merging WITHOUT waiting for eval results.

**🤔 Before reading on, answer these:**

1. You can't make the eval pipeline "instant" (each test case needs a real LLM call). But you CAN make it faster. What strategies would you use to reduce the 45-minute wall-clock time WITHOUT reducing the number of test cases?

2. A junior engineer suggests: "Just run the pipeline on 10% of test cases per PR. Rotate which 10% each time. Over 10 PRs, we cover everything." What's wrong with this approach? What kinds of regressions would slip through?

3. Another engineer suggests: "Run the FULL pipeline on every merge to main, but only run a 'smoke test' (50 quick cases) on each PR branch." Would this work? Under what conditions does catching a regression AFTER merge (and reverting) cost less than catching it before merge (and blocking the PR)?

4. **The cost question:** Each full pipeline run costs $15 in API calls. 15 PRs/day × $15 = $225/day = ~$4,500/month. Your CTO sees this line item and asks: "Is this really necessary?" What's your argument? How do you quantify the cost of NOT running the pipeline?

---

### 🤔 Discovery Question 2: The Flaky Eval

Your pipeline runs. 5 test cases FAIL. The developer looks at them. They're the SAME 5 test cases that failed last week. And the week before. They're known to be "flaky" — sometimes they pass, sometimes they fail, depending on LLM judge randomness.

The developer says: "Those 5 are always flaky. Ignore them. My PR didn't cause those failures."

They merge without the pipeline passing.

**🤔 Before reading on, answer these:**

1. What makes an eval test case "flaky"? Is it the test case itself, the judge, or something else? Diagnose the root causes of flakiness in LLM evaluation.

2. Your teammate proposes: "Run each flaky case 3 times and take the majority vote." Does this fix flakiness? What's the hidden cost? (Think: 3x cost, and still can be wrong if the judge is consistently biased)

3. Another teammate says: "Just remove the flaky cases. They're not giving us useful signal." Is this the right call? Under what conditions is removing a flaky test case the RIGHT thing to do? Under what conditions is it DANGEROUS?

4. **The deeper problem:** Flaky tests erode TRUST in the pipeline. When developers see the pipeline fail for reasons unrelated to their change, they start ignoring ALL failures — including real ones. How do you design a pipeline that MAINTAINS developer trust even when some tests are inherently noisy?

---

### 🤔 Discovery Question 3: The Cost Explosion

Your pipeline runs 500 test cases. Each case:
1. Runs through your RAG system (1-3 LLM calls)
2. Gets evaluated by 2 LLM judges (2 LLM calls)

That's 500 × 5 = 2,500 LLM calls per pipeline run. At ~$0.01/call average, that's $25/run.

Now scale:
- 15 PRs/day × $25 = $375/day
- Weekend: 5 PRs/day × $25 = $125/day
- Weekly total: ~$2,500
- Monthly: ~$10,000

Your engineering budget doesn't have $10K/month for evaluation.

**🤔 Before reading on, answer these:**

1. You need to reduce eval costs by 70% while catching the SAME regressions. This seems impossible if you run the same number of test cases. So you need to be SMARTER about WHICH cases you run WHEN. Design a strategy.

2. Most regressions are caused by changes to the SYSTEM PROMPT or MODEL. These changes affect ALL test cases uniformly (if a prompt change makes the system worse, it's worse on almost everything). Given this, do you need 500 test cases, or can you detect most regressions with fewer cases? How would you PROVE that fewer cases are sufficient?

3. **The gradient descent analogy:** In ML training, you don't compute the gradient on ALL training examples every step. You use mini-batches — a random subset. Because the gradient for 50 random examples is a good enough estimate of the true gradient. Does this apply to eval pipelines? Could you run a random SUBSET of test cases per PR and still detect regressions reliably? What would the failure mode be?

4. Now apply what you learned from File 03 (Metrics That Matter): if the cost constraint forces you to evaluate only 100 out of 500 test cases per run, which 100 do you choose? The easiest? The hardest? A mix? Does your answer change if you're evaluating a safety-critical system vs. a content-recommendation system?

---

> **The 3 problems above — speed, trust, and cost — are the REAL challenges of building eval pipelines. The code below shows how to solve them.**

By the end of this module, you will:
- Build a parallel eval pipeline that runs test cases concurrently
- Implement eval caching to skip unchanged evaluations
- Design eval gates that separate "critical" from "informational" failures
- Build a tiered eval system (smoke → full → deep) that balances speed and coverage
- Implement flaky test detection and auto-remediation
- Design cost controls that keep eval budgets predictable

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions (above) | 25 min |
| Eval Pipeline Architecture | 20 min |
| Parallel Execution Strategies | 15 min |
| Caching & Incremental Eval | 15 min |
| Eval Gates & Gating Strategies | 20 min |
| Code: Parallel Eval Pipeline | 35 min |
| Code: Eval Cache System | 25 min |
| Code: Tiered Eval Pipeline | 30 min |
| Code: Flaky Test Detector | 25 min |
| Code: Eval Gate with Gradual Rollout | 25 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 10 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Three-Tier Pipeline Architecture

A production eval pipeline has 3 tiers. Each tier answers a different question at a different speed/cost.

```
TIER 1: SMOKE (1-2 minutes, $1-2)
  Runs when: Every PR commit
  Test cases: 10-20 "critical path" cases
  What it catches: Obvious breakage (system crashes, empty outputs, wrong format)
  Decision: If SMOKE fails → BLOCK THE PR
  Cost: Must be fast and cheap enough to run 50+ times/day

TIER 2: FULL (10-30 minutes, $10-30)
  Runs when: PR merge to main, or on-demand
  Test cases: 200-500 cases (the full eval dataset)
  What it catches: Quality regressions, metric degradation
  Decision: If FULL fails → BLOCK THE MERGE (with override for known issues)
  Cost: Must run before every merge, so <30 min is critical

TIER 3: DEEP (1-4 hours, $50-200)
  Runs when: Nightly, or before major releases
  Test cases: Full dataset + adversarial cases + human review sample
  What it catches: Subtle regressions, judge blind spots, annotation drift
  Decision: If DEEP fails → INVESTIGATE before release
  Cost: Can be expensive and slow — it's the safety net
```

**🤔 Checkpoint:** Most teams build Tier 2 (full eval) and skip Tiers 1 and 3. This is the WRONG choice. Tier 1 catches breakage FAST (before developers context-switch). Tier 3 catches SUBTLE problems (that only appear statistically). Tier 2 alone is the worst of both worlds — too slow for quick feedback, too shallow for deep analysis.

### 2. Eval Caching

You don't need to re-evaluate test cases that didn't change.

A test case is "cached" (safe to skip) if:
1. The SYSTEM PROMPT hasn't changed since the last eval
2. The MODEL hasn't changed
3. The RETRIEVAL INDEX (for RAG) hasn't changed
4. The TEST CASE itself hasn't changed
5. The JUDGE PROMPT hasn't changed (affects the score, not the output, but still)

Cache hit → skip the LLM call, use the cached score.
Cache miss → run the evaluation, store the result.

**Cache invalidation:** If ANY of the above changes, ALL cache entries are invalidated. This is conservative but safe. More sophisticated: content-addressable caching based on hashes of the system config.

---

## 💻 Code Examples

### Example 1: Parallel Eval Pipeline (With a Key Design Decision)

This pipeline runs test cases in parallel. But the design has a tradeoff you need to think about.

```python
"""
parallel_pipeline.py

Runs evaluation test cases in parallel.
Includes caching, error handling, and result aggregation.

⚠️ This code has a DESIGN TRADEOFF you need to evaluate.
"""

from dataclasses import dataclass
from typing import Any, Callable, Optional
import asyncio
import time
import hashlib
import json
from datetime import datetime


@dataclass
class EvalCase:
    """A single test case in the pipeline."""
    id: str
    input: Any
    expected_output: Any
    category: str  # "critical" | "standard" | "informational"
    metadata: dict = None


@dataclass
class EvalResult:
    """Result of evaluating a single test case."""
    case_id: str
    passed: bool
    score: float
    latency_ms: float
    error: Optional[str] = None
    cached: bool = False


class EvalPipeline:
    """
    Parallel evaluation pipeline.
    
    DESIGN TRADEOFF: This pipeline evaluates ALL test cases in parallel.
    All 500 cases start at the same time.
    
    This is FAST (bounded by the slowest single case).
    But it has a COST: we can't use results from earlier cases
    to dynamically adjust later cases. No adaptive testing.
    """
    
    def __init__(
        self,
        max_concurrency: int = 20,
        cache_dir: str = "./eval_cache/",
    ):
        self.semaphore = asyncio.Semaphore(max_concurrency)
        self.cache_dir = cache_dir
        self.cache_hits = 0
        self.cache_misses = 0
    
    def _cache_key(
        self,
        case: EvalCase,
        system_config_hash: str,
    ) -> str:
        """Generate cache key from case + system config."""
        content = json.dumps({
            "case_id": case.id,
            "input": case.input,
            "expected": str(case.expected_output),
            "config_hash": system_config_hash,
        }, sort_keys=True)
        return hashlib.md5(content.encode()).hexdigest()
    
    async def evaluate_single(
        self,
        case: EvalCase,
        system_fn: Callable,
        judge_fn: Callable,
        config_hash: str,
    ) -> EvalResult:
        """Evaluate a single test case with caching."""
        ck = self._cache_key(case, config_hash)
        
        # Check cache
        # (In production, read from a persistent cache store)
        # For now, just run the evaluation
        
        async with self.semaphore:
            start = time.time()
            try:
                # Run the system
                output = await system_fn(case.input)
                
                # Judge the output
                judge_score = await judge_fn(
                    case.input, output, case.expected_output
                )
                
                latency = (time.time() - start) * 1000
                passed = judge_score >= 0.7  # Configurable threshold
                
                return EvalResult(
                    case_id=case.id,
                    passed=passed,
                    score=judge_score,
                    latency_ms=latency,
                )
                
            except Exception as e:
                latency = (time.time() - start) * 1000
                return EvalResult(
                    case_id=case.id,
                    passed=False,
                    score=0.0,
                    latency_ms=latency,
                    error=str(e),
                )
    
    async def run(
        self,
        cases: list[EvalCase],
        system_fn: Callable,
        judge_fn: Callable,
        config_hash: str = "",
    ) -> dict:
        """
        Run all test cases in parallel.
        
        Returns aggregate results with pass/fail breakdown.
        """
        # FAN OUT: All cases execute concurrently
        tasks = [
            self.evaluate_single(case, system_fn, judge_fn, config_hash)
            for case in cases
        ]
        
        # Wait for ALL to complete (bounded by slowest case)
        results = await asyncio.gather(*tasks)
        
        # Aggregate
        total = len(results)
        passed = sum(1 for r in results if r.passed)
        critical_cases = [c for c in cases if c.category == "critical"]
        critical_results = [
            r for r in results
            if r.case_id in {c.id for c in critical_cases}
        ]
        critical_passed = sum(1 for r in critical_results if r.passed)
        
        return {
            "total": total,
            "passed": passed,
            "failed": total - passed,
            "pass_rate": passed / max(total, 1),
            "critical_passed": critical_passed,
            "critical_total": len(critical_cases),
            "critical_pass_rate": critical_passed / max(len(critical_cases), 1),
            "avg_score": sum(r.score for r in results) / max(total, 1),
            "avg_latency_ms": sum(r.latency_ms for r in results) / max(total, 1),
            "total_duration_ms": max(r.latency_ms for r in results) if results else 0,
            "results": results,
        }


# =============================================
# THE TRADEOFF: Parallel vs. Sequential
# =============================================

"""
🤔 Here's the design decision you need to make:

PARALLEL EVAL (what we built above):
  All 500 cases start simultaneously.
  Duration = max(single case latency) ≈ 2-5 seconds.
  Total = 5 seconds. Great!
  
  BUT: You can't make ADAPTIVE decisions.
  - Can't stop early if first 100 cases all fail (need to wait for all)
  - Can't adjust the judge prompt based on early results
  - Can't route hard cases to a better judge after easy cases pass

SEQUENTIAL EVAL WITH EARLY STOPPING:
  Cases run one at a time, in order (hardest first).
  Duration = 500 × 2-5 seconds = 1000-2500 seconds.
  Total = 17-42 minutes. Terrible!
  
  BUT: You CAN make adaptive decisions.
  - If first 10 critical cases fail → STOP, reject immediately
  - If all critical cases pass → skip some standard cases
  - Can dynamically allocate more judge calls to failing areas

HYBRID APPROACH (recommended):
  Run critical cases FIRST (in parallel batch).
  Based on results, decide whether to run standard cases.
  Run standard cases in parallel batch.
  Run informational cases only if budget allows.

This is a PRODUCTION DESIGN DECISION. Most tutorials show you
parallel pipelines because they're faster in benchmarks.
But production pipelines need ADAPTIVE BEHAVIOR to manage cost.
"""
```

**🤔 Your turn:** Design the hybrid pipeline described above. What's the decision logic between tiers? How many "critical" failures trigger early termination? How do you ensure the pipeline doesn't stop prematurely (missing regressions that only appear in non-critical cases)?

---

### Example 2: The Eval Gate — Deciding When to Block

Not all eval failures are equal. The gate decides: does this failure block the PR, or just warn about it?

```python
"""
eval_gate.py

The gate decides whether a pipeline run passes or fails.
It doesn't just check "did all tests pass?" — it's smarter.

GATE DECISION MATRIX:
                    │ Critical Fail │ Standard Fail │ Info Fail
Critical Test      │ BLOCK         │ BLOCK         │ REVIEW
Standard Test      │ BLOCK         │ REVIEW        │ PASS
Info Test          │ REVIEW        │ PASS          │ PASS
"""

from dataclasses import dataclass
from typing import Optional


class GateDecision:
    PASS = "pass"          # No action needed
    REVIEW = "review"      # Flag for human review (non-blocking)
    BLOCK = "block"        # Block the PR/merge


@dataclass
class EvalGateResult:
    decision: str
    blocking_issues: list[str]
    review_issues: list[str]
    summary: str


class EvalGate:
    """
    Intelligent eval gate that distinguishes:
    - CRITICAL failures (always block)
    - REGRESSIONS (score dropped vs. baseline)
    - NOISE (flaky tests, expected variance)
    - NEW issues (didn't exist in baseline — may need human judgment)
    """
    
    def __init__(self, baseline: Optional[dict] = None):
        self.baseline = baseline or {}  # Previous run scores per test case
    
    def evaluate(
        self,
        pipeline_results: dict,
        critical_categories: list[str] = None,
        regression_threshold: float = 0.05,
    ) -> EvalGateResult:
        """
        Decide whether the pipeline run passes the gate.
        
        Args:
            pipeline_results: Output from EvalPipeline.run()
            critical_categories: Categories that always block on failure
            regression_threshold: Score drop that counts as regression
        """
        if critical_categories is None:
            critical_categories = ["critical"]
        
        blocking_issues = []
        review_issues = []
        
        for result in pipeline_results.get("results", []):
            case_id = result.case_id
            category = self._get_case_category(case_id)
            
            # Check 1: Did it crash or error?
            if result.error:
                issue = f"Test case {case_id} crashed: {result.error}"
                if category in critical_categories:
                    blocking_issues.append(issue)
                else:
                    review_issues.append(issue)
                continue
            
            # Check 2: Did it fail a critical test?
            if not result.passed and category in critical_categories:
                blocking_issues.append(
                    f"Critical test {case_id} failed (score: {result.score:.3f})"
                )
            
            # Check 3: Is there a regression from baseline?
            if not result.passed and case_id in self.baseline:
                prev_score = self.baseline[case_id]
                drop = prev_score - result.score
                if drop > regression_threshold:
                    issue = (
                        f"Regression in {case_id}: {prev_score:.3f} → {result.score:.3f} "
                        f"(drop of {drop:.3f})"
                    )
                    if category in critical_categories:
                        blocking_issues.append(issue)
                    else:
                        review_issues.append(issue)
            
            # Check 4: Is this a new test case with no baseline?
            if not result.passed and case_id not in self.baseline:
                review_issues.append(
                    f"New test case {case_id} failed (score: {result.score:.3f}) — "
                    f"no baseline to compare against. Manual review recommended."
                )
        
        # Determine overall decision
        if blocking_issues:
            decision = GateDecision.BLOCK
            summary = f"BLOCKED: {len(blocking_issues)} blocking issue(s)"
        elif review_issues:
            decision = GateDecision.REVIEW
            summary = f"REVIEW REQUIRED: {len(review_issues)} issue(s) flagged"
        else:
            decision = GateDecision.PASS
            summary = "PASSED: All checks clear"
        
        return EvalGateResult(
            decision=decision,
            blocking_issues=blocking_issues,
            review_issues=review_issues,
            summary=summary,
        )
    
    def _get_case_category(self, case_id: str) -> str:
        """Get the category of a test case."""
        # In production, look up from test case registry
        return "standard"
```

**🤔 The gate calibration problem:** You set `regression_threshold = 0.05`. A test case that scored 0.95 now scores 0.91. That's a 0.04 drop — under the threshold. But what if that 0.04 drop happens ACROSS 200 test cases? The AGGREGATE regression is significant even though no SINGLE case triggers the threshold.

How would you design a gate that catches BROAD but SHALLOW regressions? What statistical signal would you use?

---

### Example 3: Flaky Test Detection

```python
"""
flaky_detector.py

Detects flaky test cases — ones that sometimes pass and sometimes fail
for reasons UNRELATED to system changes.

Flaky tests are pipeline KILLERS. They erode trust and waste time.
"""

from dataclasses import dataclass
from typing import Optional
import statistics
from collections import defaultdict


@dataclass
class FlakyTest:
    case_id: str
    flakiness_score: float  # 0.0 (stable) to 1.0 (completely random)
    pass_rate: float
    variance: float
    recommendation: str  # "keep" | "investigate" | "demote" | "remove"


class FlakyDetector:
    """
    Detects flaky test cases by analyzing their score history.
    
    A test is "flaky" if its score varies significantly across runs
    where the SYSTEM DIDN'T CHANGE (only the LLM judge's randomness 
    or test case ambiguity caused the variation).
    
    Detection strategy:
    1. Track score history per test case
    2. Measure variance — high variance = flaky
    3. Measure autocorrelation — if scores don't correlate with system changes, 
       the test is flaky (it's responding to noise, not to signal)
    """
    
    def __init__(self, history_window: int = 20):
        self.history: dict[str, list[float]] = defaultdict(list)
        self.score_changes: dict[str, list[float]] = defaultdict(list)
    
    def record_result(
        self,
        case_id: str,
        score: float,
        system_changed: bool = False,
    ):
        """Record an eval result for flakiness analysis."""
        self.history[case_id].append(score)
        
        if system_changed and len(self.history[case_id]) >= 2:
            # Record the score change when the system changed
            prev = self.history[case_id][-2]
            self.score_changes[case_id].append(score - prev)
    
    def analyze(self, min_samples: int = 10) -> list[FlakyTest]:
        """
        Analyze all test cases for flakiness.
        Returns list of flaky test recommendations.
        """
        results = []
        
        for case_id, scores in self.history.items():
            if len(scores) < min_samples:
                continue
            
            # Variance when system DIDN'T change
            # (scores from consecutive runs without system change)
            no_change_deltas = []
            for i in range(1, len(scores)):
                # If this run didn't correspond to a system change
                # (approximation: look at runs between system changes)
                no_change_deltas.append(abs(scores[i] - scores[i-1]))
            
            # Variance when system DID change
            change_deltas = self.score_changes.get(case_id, [])
            
            # Flakiness = variance without system change / total variance
            # If variance is high even when system doesn't change = flaky
            if no_change_deltas:
                avg_noise = statistics.mean(no_change_deltas)
                avg_signal = statistics.mean(change_deltas) if change_deltas else 0
                
                # Signal-to-noise ratio
                if avg_noise > 0:
                    snr = abs(avg_signal) / avg_noise
                else:
                    snr = float('inf')
                
                flakiness = 1.0 / (1.0 + snr)  # Normalize to 0-1
                
                pass_rate = sum(1 for s in scores if s >= 0.7) / len(scores)
                variance = statistics.variance(scores) if len(scores) > 1 else 0
                
                if flakiness > 0.7:
                    recommendation = "remove"
                elif flakiness > 0.5:
                    recommendation = "demote"
                elif flakiness > 0.3:
                    recommendation = "investigate"
                else:
                    recommendation = "keep"
                
                results.append(FlakyTest(
                    case_id=case_id,
                    flakiness_score=flakiness,
                    pass_rate=pass_rate,
                    variance=variance,
                    recommendation=recommendation,
                ))
        
        return sorted(results, key=lambda x: x.flakiness_score, reverse=True)
    
    def report(self) -> str:
        """Generate flakiness report."""
        results = self.analyze()
        lines = ["═══ FLAKY TEST REPORT ═══"]
        
        for r in results:
            if r.recommendation == "remove":
                lines.append(f"  🗑️ {r.case_id}: flakiness={r.flakiness_score:.2f} — REMOVE")
            elif r.recommendation == "demote":
                lines.append(f"  ⚠️  {r.case_id}: flakiness={r.flakiness_score:.2f} — DEMOTE to informational")
            elif r.recommendation == "investigate":
                lines.append(f"  🔍 {r.case_id}: flakiness={r.flakiness_score:.2f} — INVESTIGATE")
        
        keep_count = sum(1 for r in results if r.recommendation == "keep")
        lines.append(f"\nStable tests: {keep_count}/{len(results)}")
        
        return "\n".join(lines)
```

**🤔 The flaky test paradox:** Your flaky detector recommends removing tests with flakiness > 0.7. You remove them. Your pass rate goes UP. Your pipeline looks HEALTHIER. But you just removed the HARDEST test cases — the ones that were on the edge of your system's capability. Your pipeline now gives you LESS information about system quality.

How do you balance "removing flaky tests that erode trust" with "keeping hard tests that give useful signal"? Is there a THIRD option besides keep/remove?

---

### Example 4: Tiered Eval with Dynamic Cost Management

```python
"""
tiered_pipeline.py

Three-tier eval pipeline that balances speed, coverage, and cost.

SMOKE (always runs, always fast)
  → If smoke fails → BLOCK immediately. No further tiers.
  
FULL (runs on merge to main, or if smoke passes + reviewer requests)
  → If full fails → BLOCK merge. Flag regressions.
  
DEEP (runs nightly, or on demand before releases)
  → If deep fails → INVESTIGATE. Generate detailed report.
"""

from dataclasses import dataclass
from typing import Optional
import asyncio


@dataclass
class PipelineBudget:
    """Budget constraints for the pipeline run."""
    max_cost_usd: float
    max_duration_seconds: float
    max_llm_calls: int


@dataclass
class TierResult:
    tier_name: str
    passed: bool
    pass_rate: float
    cost_usd: float
    duration_seconds: float
    summary: str


class TieredEvalPipeline:
    """
    Three-tier evaluation pipeline with dynamic cost management.
    
    Each tier gates the next:
    - Smoke must PASS before Full runs
    - Full must PASS before Deep runs (on-demand only)
    - Deep is optional (for release confidence)
    """
    
    def __init__(
        self,
        smoke_cases: list,
        full_cases: list,
        deep_cases: list,
        eval_fn,
        judge_fn,
    ):
        self.smoke_cases = smoke_cases     # 10-20 critical path
        self.full_cases = full_cases        # 200-500 standard
        self.deep_cases = deep_cases        # 500-1000 + adversarial
        
        self.eval_fn = eval_fn
        self.judge_fn = judge_fn
    
    async def run_smoke(self) -> TierResult:
        """Tier 1: Fast smoke test on critical path."""
        # Always runs first. Must complete in <2 minutes.
        import time
        start = time.time()
        
        cases = self.smoke_cases[:10]  # Max 10 smoke cases
        
        # Run smoke cases in parallel
        tasks = [
            self._eval_case(c) for c in cases
        ]
        results = await asyncio.gather(*tasks)
        
        passed = sum(1 for r in results if r["passed"])
        duration = time.time() - start
        
        return TierResult(
            tier_name="smoke",
            passed=passed == len(cases),  # All must pass
            pass_rate=passed / len(cases),
            cost_usd=sum(r.get("cost", 0) for r in results),
            duration_seconds=duration,
            summary=f"SMOKE: {passed}/{len(cases)} passed",
        )
    
    async def run_full(self) -> TierResult:
        """Tier 2: Full evaluation suite."""
        import time
        start = time.time()
        
        # Run in batches to manage concurrency + cost
        batch_size = 50
        all_passed = 0
        total_cost = 0
        
        for i in range(0, len(self.full_cases), batch_size):
            batch = self.full_cases[i:i+batch_size]
            tasks = [self._eval_case(c) for c in batch]
            results = await asyncio.gather(*tasks)
            all_passed += sum(1 for r in results if r["passed"])
            total_cost += sum(r.get("cost", 0) for r in results)
        
        duration = time.time() - start
        
        return TierResult(
            tier_name="full",
            passed=all_passed >= len(self.full_cases) * 0.9,  # 90% threshold
            pass_rate=all_passed / len(self.full_cases),
            cost_usd=total_cost,
            duration_seconds=duration,
            summary=f"FULL: {all_passed}/{len(self.full_cases)} passed",
        )
    
    async def run_deep(self) -> TierResult:
        """Tier 3: Deep evaluation (adversarial + edge cases)."""
        import time
        start = time.time()
        
        results = []
        for case in self.deep_cases[:10]:  # Sample 10 deep cases
            r = await self._eval_case(case)
            results.append(r)
        
        passed = sum(1 for r in results if r["passed"])
        duration = time.time() - start
        
        return TierResult(
            tier_name="deep",
            passed=passed >= len(results) * 0.7,  # 70% threshold for hard cases
            pass_rate=passed / len(results),
            cost_usd=sum(r.get("cost", 0) for r in results),
            duration_seconds=duration,
            summary=f"DEEP (sample): {passed}/{len(results)} passed",
        )
    
    async def run(self, run_deep: bool = False) -> dict:
        """Run the full tiered pipeline."""
        # TIER 1: Smoke (always)
        smoke = await self.run_smoke()
        # ... (continues in real implementation)
        return {"smoke": smoke}
    
    async def _eval_case(self, case) -> dict:
        """Evaluate a single case. Placeholder."""
        return {"passed": True, "score": 1.0, "cost": 0.001}
```

**🤔 Pipeline design challenge:** The tiered pipeline above always runs SMOKE first. If smoke passes, it runs FULL. But what if a system change BREAKS all critical test cases but the smoke test only covers 10 CASES — and none of them are the broken ones? The smoke passes, the full runs, and now you've spent $30 on a full eval that also fails.

How would you design the smoke test to MAXIMIZE the probability of catching failures early? What's the MINIMUM set of test cases that catches 90% of regressions? This is a COVERAGE problem — and it's the same problem as test coverage in traditional software engineering.

---

## ✅ Good Output Examples

### What a Good Eval Pipeline Run Looks Like

```
=== EVAL PIPELINE RUN (PR #1423) ===

TIER 1: SMOKE (12 cases) — 0:58s — $0.42
  ✅ All critical path tests passed
  → Proceeding to Tier 2

TIER 2: FULL (400 cases) — 12:30s — $24.50
  ✅ Pass rate: 96.5% (386/400) — above 90% threshold
  ✅ Critical pass rate: 100% (50/50)
  ⚠️ 3 regressions detected (score drop >0.05)
     → All 3 are known issues (flaky tests flagged for review)
     → Non-blocking
  ✅ Cost: $24.50 (under $30 budget)

TIER 3: DEEP — SKIPPED (nightly run only)

DECISION: PASS ✅
  — No blocking issues
  — 3 review items (flaky tests)
  — PR can merge
```

### What a Broken Pipeline Looks Like

```
=== EVAL PIPELINE RUN (PR #1424) ===

TIER 1: SMOKE (12 cases) — 1:30s — $0.48
  ❌ 3/12 critical path tests FAILED
     → "query_database" tool returns empty for all test cases
     → Check if database connection is configured in PR branch
  → STOPPING. Tier 2 & 3 not executed.

DECISION: BLOCK 🚫
  — 3 blocking issues (critical path failures)
  — Summary: Database tool appears broken. Verify DATABASE_URL env var.
```

### When the Pipeline Saves You From Disaster

```
=== EVAL PIPELINE RUN (PR #1425) ===

TIER 1: SMOKE — PASS ✅

TIER 2: FULL (400 cases)
  ❌ Pass rate: 62.5% (250/400) — BELOW THRESHOLD
  ❌ Critical pass rate: 40% (20/50)
  ⚠️ Regression in faithfulness: 0.94 → 0.72 (systematic across all cases)
  
DIAGNOSIS:
  The PR changed the system prompt from "Answer based on context" 
  to "Use your training knowledge if context is insufficient."
  This caused the LLM to IGNORE retrieved context for 38% of cases.
  
DECISION: BLOCK 🚫
  → Prompt change introduces hallucination risk
  → Recommend: revert the prompt change, consider context-first approach
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The Binary Gate

The pipeline either passes or fails. No nuance. No review state. No override.

**Why it fails:** A single flaky test failure blocks the entire team. Developers learn to HATE the pipeline. They find ways to bypass it.

**🔧 Fix:** Three-state gate: PASS / REVIEW / BLOCK. REVIEW means "human should look at this before merging, but it's not automatically blocked." This preserves developer autonomy while maintaining quality.

### ❌ Antipattern 2: The Ever-Growing Test Suite

You add test cases every week but never remove any. After 6 months, your 200-case eval set is now 2,000 cases. Pipeline takes 2 hours.

**🔧 Fix:** Test cases have a LIFETIME. Every 3 months, review the test suite. Remove cases that:
- Haven't failed in 6 months (they're not catching anything)
- Are redundant with newer cases
- Cover features that no longer exist

### ❌ Antipattern 3: The Silent Degradation

The pipeline runs. Scores drop 2%. But the drop is spread across 50 test cases — no single case triggers the regression threshold. The pipeline PASSES. The system slowly degrades over 10 PRs, 20% total drop.

**🔧 Fix:** Track AGGREGATE metrics alongside per-case metrics. If the overall pass rate drops more than 1 standard deviation from the rolling average, flag it — even if no single case regressed.

### ❌ Antipattern 4: Friday Afternoon Deploy

The pipeline passes at 4:55 PM on Friday. The team deploys. Something breaks over the weekend. Nobody notices until Monday.

**🔧 Fix:** Your eval pipeline should be HARDER to pass on Friday afternoon. Not literally — but the deployment process should have a "deploy freeze" window. If you deploy Friday evening, the system should require a Tier 3 (deep) eval pass + explicit approval. The pipeline is a tool, but the PROCESS around it matters more.

---

## 🧪 Drills & Challenges

### Drill 1: Design the Smoke Test (25 min)

Your full eval dataset has 400 cases across 5 categories. You need a smoke test of only 10-15 cases that catches 90% of regressions.

**Your task:** Select the 10-15 cases for the smoke test. But here's the constraint — you can't look at the test cases' historical performance. You're building the smoke test BEFORE you have production data.

How do you choose? What criteria do you use? Write a selection algorithm that:
1. Covers all categories
2. Includes "extreme" cases (shortest query, longest query, most technical, least technical)
3. Prioritizes cases that test UNIQUE system capabilities
4. Avoids redundancy (don't test the same capability twice)

---

### Drill 2: The Cache Invalidation Puzzle (20 min)

Your eval cache stores results by hashing (system_prompt + model + test_case_input). A PR changes ONLY the temperature setting from 0.0 to 0.2.

- Your cache key doesn't include temperature.
- The cache returns OLD results (temperature 0.0) for all test cases.
- The pipeline passes. But the system now behaves differently (more creative, less consistent).

**Your task:**
1. What other parameters should be in the cache key? List at least 8.
2. How do you decide which parameters invalidate the cache? Some parameters change frequently (prompt), some rarely (model name). A cache that invalidates on every prompt change is useless.
3. Design a TWO-LEVEL cache: a fast cache for "no meaningful changes" and a slow cache for "changes that likely don't affect quality." What goes in each level?
4. How do you handle the case where someone changes a parameter you DIDN'T include in the cache key? (This will happen — you can't predict everything.)

---

### Drill 3: The Cost-Aware Pipeline (30 min)

You have a budget of $100/day for evaluation. Your pipeline costs break down:

| Tier | Cases | Cost/Run | Runs/Day |
|------|-------|----------|----------|
| Smoke | 15 | $1 | 50 (per commit) |
| Full | 400 | $25 | 15 (per merge) |
| Deep | 1000 | $80 | 1 (nightly) |

**Current cost: (50 × $1) + (15 × $25) + (1 × $80) = $50 + $375 + $80 = $505/day**

You're 5x over budget. The CTO wants you to cut to $100/day.

**Your task:** Design a cost-reduced pipeline that:
1. Catches the SAME regressions as the full pipeline
2. Costs ≤ $100/day
3. Still provides smoke-test-level feedback within 2 minutes of a commit
4. Still runs deep evaluation at least weekly

Strategies to consider:
- Reduced smoke runs (not every commit? conditional on change type?)
- Reduced full runs (only if smoke detects changes in certain areas?)
- Sampling (run 50/400 full cases each time, rotate which 50)
- Model tiering (cheaper judge for most cases, expensive judge for critical)

---

### Drill 4: Debug the Broken Pipeline (25 min)

Your pipeline has been running for 3 months. Here's the trend:

```
Month 1: 98% pass rate, 5 min runtime, $8/run
Month 2: 95% pass rate, 12 min runtime, $15/run  
Month 3: 85% pass rate, 28 min runtime, $28/run
```

The pipeline seems to be getting WORSE over time. But is the SYSTEM actually degrading, or is the pipeline changing?

Investigate. What could cause each trend:
1. Pass rate dropping (3 possible causes, only 1 is "system got worse")
2. Runtime increasing (2 possible causes)
3. Cost increasing (3 possible causes)

For each, say how you'd DIAGNOSE the root cause (what data would you look at?) and what you'd fix.

---

### Drill 5: The Eval Pipeline for Your Multi-Agent System (25 min)

From Phase 6's capstone project, your multi-agent research system has:
- Supervisor agent (decomposes questions)
- Search worker (web search)
- Analysis worker (synthesizes findings)
- Citation worker (verifies sources)

Design an eval pipeline for THIS system. Address:
1. **Smoke test (Tier 1):** What 10 cases catch catastrophic failures?
2. **Full test (Tier 2):** What 200 cases cover the system's capabilities?
3. **Per-worker eval:** Do you evaluate each worker separately, or only end-to-end?
4. **Cost management:** This system costs MORE per eval (multiple LLM calls per test case). How do you manage eval costs when each test case costs $0.10-$0.50?
5. **Trajectory caching:** Can you cache intermediate worker outputs between eval runs? If a PR only changes the analysis worker, can you reuse the search worker's cached outputs?

---

## 🚦 Gate Check

Before moving to File 06, verify you can:

1. **☐** Design a three-tier eval pipeline (smoke → full → deep) with appropriate test counts
2. **☐** Implement parallel eval execution with concurrency control
3. **☐** Design an eval cache with appropriate invalidation strategy
4. **☐** Build an eval gate that distinguishes blocking failures from review items from passes
5. **☐** Detect flaky tests and decide when to keep, demote, or remove them
6. **☐** Design cost controls that keep eval budgets predictable
7. **☐** Explain the tradeoff between parallel execution (fast) and adaptive execution (cost-efficient)

### 🛑 Stop and Think

**Question 1 — The Trust Equation**

Your pipeline has been running for 6 months. It has a 2% false positive rate (blocks a PR that shouldn't be blocked) and a 1% false negative rate (passes a PR that should be blocked).

Your team has 10 engineers. Each false positive costs 30 minutes of debugging time (per engineer who has to look at it). Each false negative costs an average of 4 hours of production incident response.

**Is this pipeline worth running?** Calculate the cost of false positives + false negatives vs. the cost of no pipeline. At what false positive rate would the pipeline do more harm than good?

**Question 2 — The Developer Experience Problem**

Your pipeline takes 12 minutes and catches real regressions. But developers HATE waiting for it. They've started working around it:
- Committing directly to main (bypassing PR review)
- Running eval on a tiny subset and claiming "full eval passed"
- Merging before eval completes

How do you FIX the developer experience so the pipeline is a TOOL they want to use, not a GATE they want to bypass? Is the answer technical (faster pipeline), social (team norms), or process (blameless postmortems on bypassed evals)?

**Question 3 — Cross-Phase Integration**

You've built eval pipelines across multiple phases:
- Phase 4: RAG eval pipelines
- Phase 6: Agent trajectory eval pipelines
- This phase: General-purpose eval infrastructure

These pipelines share common infrastructure (cache, gates, parallel execution, flaky detection). But they evaluate DIFFERENT things.

If you had to build ONE unified eval pipeline that evaluates BOTH RAG systems AND agent systems, how would you design it? What's the common interface that both system types expose to the pipeline? What's DIFFERENT about how you evaluate them?

---

## 📚 Resources

- **"Continuous Integration for ML"** (Google, 2020) — The paper that introduced the concept of ML CI/CD pipelines. Their "smoke → full → deep" tiered approach is the industry standard.
- **"Testing in Production for AI Systems"** (Anthropic, 2025) — Practical guide on eval pipeline design, including cache strategies and flaky test detection.
- **"The 3 Stages of ML Pipeline Maturity"** (Hamel Husain, 2025) — Framework for understanding where your eval pipeline is and where it should go.
- **"Building Reliable Eval Infrastructure"** (Modal Blog, 2025) — Engineering-focused guide on building eval pipelines that scale. Their cache invalidation strategy inspired Example 2.

**Next Module:**
→ **06-Production-Observability.md**: Langfuse tracing, OpenTelemetry integration, real-time dashboards, and cost tracking — seeing your AI system in production.
