# 01 — Evals-First Mindset: Design Evaluation Before Implementation

## 🎯 Purpose & Goals

### 🛑 STOP. Before reading about eval-first design, consider this story.

**The $50,000 Deployment**

A senior engineer at a mid-size SaaS company spends 3 weeks building an AI-powered customer support triage system. It classifies incoming support tickets into categories, estimates severity, and routes to the right team. The demo is impressive — 90% accuracy on the 50 test cases the engineer manually curated.

The system deploys on a Monday. By Wednesday, the support team is in revolt. The system is routing "billing complaint" tickets to "feature request" with alarming frequency. A major customer's urgent outage ticket was classified as "low priority documentation question" and sat for 6 hours.

The engineer spends 2 weeks debugging. The problem: the test set was 50 tickets the engineer wrote himself, all perfectly matching the documentation. Production tickets are messy — typos, mixed signals, frustrated customers venting before stating their actual problem.

**The real cost:** 3 weeks of engineering time (building) + 2 weeks (debugging) + 1 week (fixing) + lost customer trust + 6 hours of delayed response to a major customer.

**The eval-first question:** What if the engineer had spent the FIRST 3 days building an evaluation framework and curating a realistic test set? Would the accuracy problem have been caught before deployment? Would the system have launched with a known "fragile on noisy input" warning?

---

### 🤔 Your First Challenge

Think about the LAST AI system you built (or would build). Answer honestly:

1. **When did you design your evaluation?** Before implementation, during, or after?
2. **How representative was your test data?** Did it match production distribution, or was it what was easiest to collect?
3. **What was the gap between your eval scores and production performance?** Did you ever measure it?
4. **How much time did you spend on evaluation infrastructure vs. implementation?**

**These questions reveal your eval maturity.** Most teams spend <5% of their AI engineering time on evaluation infrastructure. Phase 7 shifts that to 30-40%.

---

### 🤔 Why "Eval Later" Always Fails

The temptation to build first, evaluate later is powerful:
- Building is visible progress; evaluating feels like overhead
- "I'll add evals once the system works" sounds reasonable
- Demo pressure favors building over testing
- Evaluation feels like someone else's job (QA, product)

But eval-later has a deadly pattern:

```
Week 1: Build something cool. Demo it. Stakeholders are impressed.
Week 2: Add more features. Demo again. More impressed.
Week 3: "Let's add some evaluation" — but the system is already complex.
         Eval results are mediocre. Hard to tell if it's the eval or the system.
Week 4: "Let's just fix the obvious issues and ship."
Week 5: Deployed. Production performance is worse than expected.
Week 6: "We should have built evaluation first."
```

**The pattern is universal.** Every AI team I've seen that didn't start with evaluation ended up here. The fix is structural, not motivational — you need to embed evaluation into your workflow so deeply that it's HARDER to skip than to do.

---

By the end of this module, you will:

1. **Understand the eval-first workflow** and why it produces better systems
2. **Recognize eval debt** — the hidden cost of skipping evaluation infrastructure
3. **Design an eval-first process** for any AI project
4. **Build an eval specification template** that you use before writing implementation code
5. **Quantify the cost of poor evaluation** — making the business case for eval infrastructure
6. **Implement the "eval contract" pattern** — a formal spec that both eval and implementation must satisfy

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The Eval-First Workflow | 25 min |
| Eval Debt: The Hidden Cost | 20 min |
| The Eval Contract Pattern | 30 min |
| Code: Eval Specification Template | 30 min |
| Code: Automated Eval Scaffolding | 30 min |
| Code: Eval Debt Tracker | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 10 min |
| **Total** | **~4.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Eval-First Workflow

Eval-first means: **before you write a single line of system code, you define how you'll measure success.**

The workflow has 5 phases:

```
PHASE 1: DEFINE           PHASE 2: DATASET        PHASE 3: METRICS
                                                          
  What does success           Collect/curate         Define HOW you'll
  look like? Define           representative         measure success.
  it concretely.              test examples.         Implement metric
  "95% of queries get        "100 real support      functions BEFORE
  the correct answer          tickets from the       writing system code.
  within 3 seconds."          last 3 months."
                                                          
          │                         │                         │
          ▼                         ▼                         ▼
                                                          
  PHASE 4: BASELINE         PHASE 5: BUILD
                                                          
  Establish baseline         Build the system.
  performance with a         Compare against
  simple/cheap approach.     baseline at every
  "If we just use GPT-       step. Ship ONLY when
  4o-mini with no RAG,       eval passes your
  what accuracy do we        bar — not when
  get?"                      the demo looks good.
```

**Key insight:** The evaluation should be running BEFORE you have anything to evaluate. You start with a "null system" (empty responses, random guesses, or a simple baseline) and watch the scores improve as you build.

This is exactly how TDD works in software engineering:
- TDD: Write a failing test → Write code to make it pass → Refactor → Rinse repeat
- Eval-first: Write eval with baseline score → Improve system → Check score improves → Rinse repeat

### 2. The Eval Contract

An eval contract is a formal specification that BOTH the evaluation system and the implementation system must satisfy.

```
EVAL CONTRACT
══════════════

System: Customer Support Triage

Input Contract:
  - Type: Support ticket text
  - Max length: 5000 characters
  - Languages: English only
  - Fields: {text, metadata(customer_tier, product_category)}

Output Contract:
  - category: billing | technical | feature_request | account | other
  - severity: critical | high | medium | low
  - confidence: float (0.0-1.0)
  - response_time_ms: int

Quality Contract:
  - Accuracy ≥ 0.90 (macro avg across categories)
  - Critical severity recall ≥ 0.95 (must not miss urgent tickets)
  - Response time ≤ 2000ms (p95)
  - No category confusion rate > 5% between billing ↔ feature_request

Eval Dataset:
  - 500 tickets from last 3 months of production data
  - Stratified by category (real distribution: ~25% each)
  - 50 edge cases (ambiguous tickets, typos, mixed intents)
  - Human-annotated by 2 senior support agents (Kappa ≥ 0.85)

Baseline:
  - GPT-4o-mini zero-shot classification: 0.72 accuracy
  - Our system must beat: 0.85 accuracy (minimal bar)
```

**Why this matters:** The eval contract makes evaluation NON-NEGOTIABLE. You can't ship a system that doesn't satisfy its contract. The contract is the source of truth — not the demo, not stakeholder intuition, not a blog post benchmark.

### 3. Eval Debt

Eval debt is the accumulation of deferred evaluation infrastructure. Like tech debt, it compounds:

| Eval Debt Type | Symptom | Interest Payment |
|---------------|---------|-----------------|
| **No eval dataset** | "We tested 20 cases manually" | Missed regressions, unknown failure modes |
| **Low-quality dataset** | "The test set doesn't match production" | High eval scores, poor production performance |
| **No automated pipeline** | "We run eval manually before release" | Inconsistent testing, skipped eval under pressure |
| **No regression tracking** | "I think the system got worse but I'm not sure" | Can't debug regressions, no improvement signal |
| **No production feedback** | "We don't know how the system is actually performing" | Blind deployment, user trust erosion |
| **No eval versioning** | "I updated the test set last week, I think" | Can't compare results across time, results are meaningless |

#### The Eval Debt Snowball

```
Day 1: Skip writing eval contract ("I'll start coding first")
Day 3: Skip curating eval dataset ("I'll collect examples as I go")
Day 5: Skip implementing metrics ("I can eyeball it")
Day 10: Demo looks good. Stakeholders want to ship.
Day 12: Someone asks "how do we know it works?" Panic.
Day 13-20: Rush to build evaluation. It's cobbled together.
            Eval results are bad because the dataset is bad.
            Hard to tell if system is bad or eval is bad.
Day 21: Ship anyway because pressure > evidence.
Day 30: Production performance is poor. Users complain.
Day 31: "We should have built evals first."
```

**The cost of eval debt is ALWAYS higher than the cost of building evals first.** But it's deferred — you feel the pain later, not now. This is why eval-first requires discipline.

---

## 💻 Code Examples

### Example 1: The Eval Specification Template

This is the FIRST file you create for any AI project. Before writing any implementation code, fill out this template.

```python
"""
eval_spec.py

Eval Specification for: [System Name]
Created: [Date]
Owner: [Name]

THIS FILE IS WRITTEN BEFORE ANY IMPLEMENTATION CODE.
It defines how success is measured. Implementation code
must satisfy this spec — not the other way around.
"""

from dataclasses import dataclass, field
from typing import Callable, Any
from datetime import datetime


@dataclass
class InputSpec:
    """What inputs does the system accept?"""
    type: str  # "text" | "structured" | "multimodal"
    max_length: int = 5000
    fields: dict[str, str] = field(default_factory=dict)
    languages: list[str] = field(default_factory=lambda: ["en"])
    notes: str = ""


@dataclass
class OutputSpec:
    """What outputs does the system produce?"""
    type: str  # "text" | "classification" | "structured" | "action"
    fields: dict[str, str] = field(default_factory=dict)
    constraints: list[str] = field(default_factory=list)
    notes: str = ""


@dataclass
class MetricTarget:
    """A single quality target with minimum acceptable value."""
    name: str
    type: str  # "accuracy" | "recall" | "latency" | "cost" | "custom"
    min_value: float
    max_value: float = 1.0
    is_critical: bool = False  # Critical metrics BLOCK deployment
    notes: str = ""


@dataclass
class EvalDatasetSpec:
    """Description of the evaluation dataset needed."""
    min_size: int
    source: str  # "production" | "synthetic" | "human_annotated" | "mixed"
    stratification: list[str] = field(default_factory=list)
    edge_case_count: int = 0
    annotation_guidelines: str = ""
    inter_annotator_agreement: float = 0.0  # Minimum Cohen's Kappa


@dataclass
class Baseline:
    """Performance of a simple/cheap approach."""
    approach: str  # "gpt-4o-mini-zeroshot" | "random" | "rule-based"
    scores: dict[str, float]  # metric_name → score
    cost_per_query: float = 0.0
    notes: str = ""


@dataclass
class EvalContract:
    """
    The complete eval contract for this system.
    Written BEFORE implementation begins.
    """
    system_name: str
    version: str = "0.1.0"
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())
    
    input_spec: InputSpec = field(default_factory=InputSpec)
    output_spec: OutputSpec = field(default_factory=OutputSpec)
    metrics: list[MetricTarget] = field(default_factory=list)
    dataset_spec: EvalDatasetSpec = field(default_factory=EvalDatasetSpec)
    baseline: Baseline = field(default_factory=Baseline)
    
    def validate(self) -> list[str]:
        """Check that the contract is internally consistent."""
        issues = []
        
        if not self.metrics:
            issues.append("No metrics defined!")
        for m in self.metrics:
            if m.min_value > m.max_value:
                issues.append(f"Metric '{m.name}': min > max ({m.min_value} > {m.max_value})")
        
        if not self.dataset_spec.min_size:
            issues.append("Dataset spec missing minimum size")
        
        if not self.baseline.approach:
            issues.append("No baseline defined")
        
        return issues
    
    def summary(self) -> str:
        """Human-readable summary of the contract."""
        lines = [
            f"Eval Contract: {self.system_name} v{self.version}",
            f"Created: {self.created_at}",
            "",
            "INPUT:",
            f"  Type: {self.input_spec.type}",
            f"  Max length: {self.input_spec.max_length} chars",
            f"  Languages: {', '.join(self.input_spec.languages)}",
            "",
            "OUTPUT:",
            f"  Type: {self.output_spec.type}",
            f"  Fields: {', '.join(self.output_spec.fields.keys())}",
            "",
            "QUALITY TARGETS:",
        ]
        for m in self.metrics:
            critical = " [CRITICAL]" if m.is_critical else ""
            lines.append(f"  {m.name}: {m.min_value}-{m.max_value}{critical}")
        
        lines.extend([
            "",
            "DATASET:",
            f"  Min size: {self.dataset_spec.min_size}",
            f"  Source: {self.dataset_spec.source}",
            f"  Stratification: {', '.join(self.dataset_spec.stratification)}",
            "",
            "BASELINE:",
            f"  Approach: {self.baseline.approach}",
            f"  Best score: {max(self.baseline.scores.values()) if self.baseline.scores else 'N/A'}",
        ])
        
        return "\n".join(lines)


# =============================================
# EXAMPLE: Customer Support Triage Eval Contract
# =============================================

triage_contract = EvalContract(
    system_name="Customer Support Triage",
    input_spec=InputSpec(
        type="text",
        max_length=5000,
        fields={"text": "str", "customer_tier": "str", "product_category": "str"},
        languages=["en"],
    ),
    output_spec=OutputSpec(
        type="classification",
        fields={
            "category": "billing | technical | feature_request | account | other",
            "severity": "critical | high | medium | low",
            "confidence": "float",
        },
        constraints=[
            "Exactly one category per ticket",
            "Confidence must be 0.0-1.0",
        ],
    ),
    metrics=[
        MetricTarget(name="accuracy", type="accuracy", min_value=0.90, is_critical=True),
        MetricTarget(name="critical_recall", type="recall", min_value=0.95, is_critical=True),
        MetricTarget(name="p95_latency_ms", type="latency", min_value=0, max_value=2000, is_critical=True),
        MetricTarget(name="billing_feature_confusion", type="custom", min_value=0, max_value=0.05),
        MetricTarget(name="cost_per_query", type="cost", min_value=0, max_value=0.05),
    ],
    dataset_spec=EvalDatasetSpec(
        min_size=500,
        source="production",
        stratification=["category", "severity"],
        edge_case_count=50,
        annotation_guidelines="See ANNOTATION_GUIDELINES.md",
        inter_annotator_agreement=0.85,
    ),
    baseline=Baseline(
        approach="gpt-4o-mini-zeroshot",
        scores={"accuracy": 0.72, "critical_recall": 0.68},
        cost_per_query=0.002,
    ),
)
```

**The eval contract is your FIRST deliverable.** Before writing a single line of the triage system, you create and review this document with stakeholders. This surfaces disagreements early — when they're cheap to fix — rather than at deployment review.

---

### Example 2: Eval Scaffolding — Running Evaluation Before Implementation

Once you have the eval contract, build the eval harness that runs even when there's nothing to evaluate yet.

```python
"""
eval_harness.py

Eval harness that runs from Day 1. 
Starts with a null system and tracks scores as implementation improves.
"""

from dataclasses import dataclass
from typing import Any, Callable, Optional
import json
import time
import statistics


@dataclass
class EvalResult:
    """Result of evaluating a single test case."""
    test_id: str
    input_data: Any
    expected_output: Any
    actual_output: Any
    metrics: dict[str, float]
    latency_ms: float
    passed: bool
    error: Optional[str] = None


@dataclass
class EvalSuiteResult:
    """Aggregate results across all test cases."""
    system_name: str
    total_cases: int
    passed_cases: int
    metrics: dict[str, float]  # metric_name → average value
    metric_targets: dict[str, tuple[float, float]]  # metric_name → (actual, target)
    critical_failures: list[str]
    timestamp: str
    duration_seconds: float
    
    @property
    def pass_rate(self) -> float:
        return self.passed_cases / max(self.total_cases, 1)
    
    def summary(self) -> str:
        lines = [
            f"═══ Eval: {self.system_name} ═══",
            f"Pass rate: {self.passed_cases}/{self.total_cases} ({self.pass_rate:.1%})",
            f"Duration: {self.duration_seconds:.1f}s",
            "",
            "METRICS:",
        ]
        for name, value in sorted(self.metrics.items()):
            target = self.metric_targets.get(name)
            if target:
                actual, (min_val, max_val) = value, target
                status = "✅" if min_val <= actual <= max_val else "❌"
                lines.append(f"  {status} {name}: {actual:.4f} (target: {min_val}-{max_val})")
            else:
                lines.append(f"  {name}: {value:.4f}")
        
        if self.critical_failures:
            lines.extend(["", "CRITICAL FAILURES:"])
            for f in self.critical_failures:
                lines.append(f"  ❌ {f}")
        
        return "\n".join(lines)


class EvalHarness:
    """
    Runs evaluation for any AI system.
    
    Phase 1: Runs with a NULL system (returns empty strings)
             — establishes floor baseline
    Phase 2: Runs with baseline system (e.g., GPT-4o-mini zero-shot)
             — establishes minimum acceptable bar
    Phase 3: Runs with your implementation
             — measures improvement over baseline
    """
    
    def __init__(self, contract: 'EvalContract', test_cases: list[dict]):
        self.contract = contract
        self.test_cases = test_cases
        self.results: list[EvalResult] = []
    
    def run(self, system_fn: Callable) -> EvalSuiteResult:
        """
        Run all test cases through the system function.
        
        Args:
            system_fn: Callable that takes input and returns output.
                       On Day 1, this might return empty strings.
                       Later, it's your actual implementation.
        """
        start_time = time.time()
        self.results = []
        
        for i, case in enumerate(self.test_cases):
            result = self._evaluate_single(case, system_fn)
            self.results.append(result)
        
        duration = time.time() - start_time
        
        # Aggregate metrics
        all_metrics = {}
        for result in self.results:
            for name, value in result.metrics.items():
                if name not in all_metrics:
                    all_metrics[name] = []
                all_metrics[name].append(value)
        
        avg_metrics = {
            name: statistics.mean(values)
            for name, values in all_metrics.items()
        }
        
        # Check metric targets
        metric_targets = {}
        for target in self.contract.metrics:
            actual = avg_metrics.get(target.name, 0.0)
            metric_targets[target.name] = (actual, (target.min_value, target.max_value))
        
        # Find critical failures
        critical_failures = []
        for target in self.contract.metrics:
            if not target.is_critical:
                continue
            actual = avg_metrics.get(target.name, 0.0)
            if actual < target.min_value or actual > target.max_value:
                critical_failures.append(
                    f"{target.name}: {actual:.4f} outside [{target.min_value}, {target.max_value}]"
                )
        
        return EvalSuiteResult(
            system_name=self.contract.system_name,
            total_cases=len(self.test_cases),
            passed_cases=sum(1 for r in self.results if r.passed),
            metrics=avg_metrics,
            metric_targets=metric_targets,
            critical_failures=critical_failures,
            timestamp=time.strftime("%Y-%m-%dT%H:%M:%S"),
            duration_seconds=duration,
        )
    
    def _evaluate_single(self, case: dict, system_fn: Callable) -> EvalResult:
        """Evaluate a single test case."""
        test_id = case.get("id", "unknown")
        input_data = case.get("input", "")
        expected = case.get("expected", {})
        
        try:
            start = time.time()
            actual = system_fn(input_data)
            latency = (time.time() - start) * 1000
            
            # Compute metrics (simplified — real implementation is per-system)
            metrics = self._compute_metrics(expected, actual)
            
            # Check if all critical metrics pass
            passed = all(
                target.min_value <= metrics.get(target.name, 0) <= target.max_value
                for target in self.contract.metrics
                if target.is_critical
            )
            
            return EvalResult(
                test_id=test_id,
                input_data=input_data,
                expected_output=expected,
                actual_output=actual,
                metrics=metrics,
                latency_ms=latency,
                passed=passed,
            )
            
        except Exception as e:
            return EvalResult(
                test_id=test_id,
                input_data=input_data,
                expected_output=expected,
                actual_output=None,
                metrics={},
                latency_ms=0,
                passed=False,
                error=str(e),
            )
    
    def _compute_metrics(self, expected: Any, actual: Any) -> dict[str, float]:
        """Compute metrics for a single prediction."""
        # This is system-specific.
        # In a real implementation, you'd have metric functions
        # registered per metric type.
        metrics = {}
        
        if isinstance(expected, dict) and isinstance(actual, dict):
            # Simple exact match for structured outputs
            correct = sum(1 for k, v in expected.items()
                         if actual.get(k) == v)
            total = len(expected)
            metrics["accuracy"] = correct / max(total, 1)
        
        return metrics


# =============================================
# DAY 1 USAGE: Run eval with NULL system
# =============================================

if __name__ == "__main__":
    # Load eval contract (from Example 1)
    contract = triage_contract  # Our eval contract from earlier
    
    # Load test cases (even if it's just 10 simple ones on Day 1)
    test_cases = [
        {
            "id": "001",
            "input": {"text": "My account was charged twice for the same invoice"},
            "expected": {"category": "billing", "severity": "high"},
        },
        {
            "id": "002",
            "input": {"text": "The API keeps returning 500 errors on POST /orders"},
            "expected": {"category": "technical", "severity": "high"},
        },
        # ... more cases as you curate them
    ]
    
    harness = EvalHarness(contract, test_cases)
    
    # Phase 1: Null system (returns empty/None)
    def null_system(input_data):
        return {"category": None, "severity": None, "confidence": 0.0}
    
    result = harness.run(null_system)
    print(result.summary())
    # Output: Pass rate: 0/2 (0.0%) — Baseline established
    
    # Phase 2: Baseline system (random guessing)
    import random
    def random_baseline(input_data):
        categories = ["billing", "technical", "feature_request", "account", "other"]
        severities = ["critical", "high", "medium", "low"]
        return {
            "category": random.choice(categories),
            "severity": random.choice(severities),
            "confidence": random.random(),
        }
    
    result = harness.run(random_baseline)
    print(result.summary())
    # Output: Pass rate: ~0/2 (0%) — Random is still terrible
    
    # Phase 3: Simple baseline (cheap LLM)
    # (You'd implement this once you have the eval harness working)
    def gpt_mini_baseline(input_data):
        # This won't work until you implement it, but the
        # eval harness is already running!
        return {"category": "unknown", "severity": "unknown", "confidence": 0.0}
    
    result = harness.run(gpt_mini_baseline)
    print(result.summary())
    # Day 1 output: Everything fails. But that's expected!
    # You now have a MEASUREMENT SYSTEM in place before writing real code.
```

**The key insight:** On Day 1, the eval harness produces terrible scores. This is expected and valuable. It establishes:
- The floor (null system: 0%)
- The naive baseline (random: ~5% for 5 categories)
- The cheap baseline (GPT-4o-mini: 72%)
- Your target (custom system: 90%+)

Every improvement you make is measured against this progression. You always know if you're making progress.

---

### Example 3: The Eval Debt Tracker

```python
"""
eval_debt.py

Tracks eval debt items and their estimated cost.
Updated in every sprint retrospective.
"""

from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional


class EvalDebtSeverity:
    LOW = "low"          # Annoying but not blocking
    MEDIUM = "medium"    # Could cause missed regressions
    HIGH = "high"        # Known blind spot in evaluation
    CRITICAL = "critical"  # You are flying blind


@dataclass
class EvalDebtItem:
    description: str
    severity: str
    estimated_interest: str  # What happens if we delay
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())
    resolved_at: Optional[str] = None
    interest_paid: list[str] = field(default_factory=list)  # Actual incidents caused
    
    def resolve(self):
        self.resolved_at = datetime.now().isoformat()


class EvalDebtTracker:
    """Tracks evaluation debt across projects."""
    
    def __init__(self):
        self.items: list[EvalDebtItem] = []
    
    def add_item(self, item: EvalDebtItem):
        self.items.append(item)
    
    def unresolved(self, severity: Optional[str] = None) -> list[EvalDebtItem]:
        items = [i for i in self.items if i.resolved_at is None]
        if severity:
            items = [i for i in items if i.severity == severity]
        return items
    
    def report(self) -> str:
        """Generate eval debt report."""
        unresolved = self.unresolved()
        critical = self.unresolved(EvalDebtSeverity.CRITICAL)
        high = self.unresolved(EvalDebtSeverity.HIGH)
        
        lines = [
            f"EVAL DEBT REPORT — {datetime.now().strftime('%Y-%m-%d')}",
            f"Total items: {len(self.items)}",
            f"Unresolved: {len(unresolved)}",
            f"  Critical: {len(critical)}",
            f"  High: {len(high)}",
            "",
        ]
        
        if critical:
            lines.append("CRITICAL ITEMS:")
            for item in critical:
                lines.append(f"  🚨 {item.description}")
                lines.append(f"      Interest: {item.estimated_interest}")
                if item.interest_paid:
                    for incident in item.interest_paid:
                        lines.append(f"      🔴 Paid: {incident}")
        
        if high:
            lines.append("HIGH ITEMS:")
            for item in high:
                lines.append(f"  ⚠️  {item.description}")
        
        return "\n".join(lines)


# === Example usage in a sprint retrospective ===

tracker = EvalDebtTracker()

tracker.add_item(EvalDebtItem(
    description="No edge case test set — testing only with clean queries",
    severity=EvalDebtSeverity.HIGH,
    estimated_interest="Will miss noisy-input failures in production",
    interest_paid=[
        "Week 3: Production ticket with typos misclassified as 'other'",
    ],
))

tracker.add_item(EvalDebtItem(
    description="No regression tracking — cannot tell if changes improve or hurt",
    severity=EvalDebtSeverity.CRITICAL,
    estimated_interest="Every deploy is a gamble. Can't optimize systematically.",
    interest_paid=[
        "Week 5: Prompt change dropped accuracy 8%, didn't notice for 2 weeks",
    ],
))

tracker.add_item(EvalDebtItem(
    description="Eval dataset not versioned — can't reproduce past results",
    severity=EvalDebtSeverity.MEDIUM,
    estimated_interest="'I think accuracy improved' — no evidence either way",
))

print(tracker.report())
```

> **🤔 The CTO Question:** Your CTO asks: "Why should we spend 2 weeks building evaluation infrastructure instead of 2 weeks improving the AI system?" What do you say? How do you quantify the cost of NOT having evaluation? Can you point to specific incidents where eval debt would have caught problems earlier?

---

### Example 4: The Eval-First Sprint Template

```python
"""
sprint_template.py

How to structure a 2-week AI sprint using the eval-first approach.
Share this with your team.
"""

SPRINT_TEMPLATE = """
EVAL-FIRST SPRINT STRUCTURE
═══════════════════════════

SPRINT GOAL: [One sentence]

DAY 1-2: EVAL CONTRACT (Phase 1)
  □ Write eval contract (use eval_spec.py template)
  □ Review with stakeholders — get sign-off on metrics
  □ Identify what "good" means, not just "better"
  □ Define baseline approach (cheapest possible)
  □ Deliverable: eval_contract.md (APPROVED)

DAY 3-4: EVAL DATASET (Phase 2)
  □ Curate minimum 50 test cases from production data
  □ Annotate with expected outputs (double-annotate 20% for quality)
  □ Add edge cases (ambiguous, adversarial, unusual)
  □ Build eval harness (use eval_harness.py)
  □ Deliverable: eval_dataset.json + eval_harness.py (RUNNING)

DAY 5: BASELINE (Phase 3)
  □ Run eval on null system → record floor
  □ Run eval on cheapest baseline → record baseline
  □ Set improvement target relative to baseline
  □ Deliverable: baseline_report.md

DAY 6-10: BUILD (Phase 4)
  □ Implement system
  □ Run eval after EVERY significant change
  □ Compare against baseline and previous scores
  □ If score drops, stop and investigate
  □ Deliverable: working system + eval_trajectory.json

DAY 11-12: OPTIMIZE
  □ Profile bottlenecks (eval tells you what to optimize)
  □ Run ablation studies (what happens if we remove X?)
  □ Test edge cases
  □ Deliverable: optimization_report.md

DAY 13-14: SHIP DECISION
  □ Run final eval
  □ Check ALL metrics against contract targets
  □ If critical metrics fail → do NOT ship, write debt items
  □ If all pass → prepare deployment with monitoring
  □ Deliverable: ship/no-ship decision + monitoring plan
"""

print(SPRINT_TEMPLATE)
```

---

## ✅ Good Output Examples

### What an Eval-First Project Looks Like

```
Project: Support Ticket Classifier

Day 1 Artifacts:
  ✅ eval_contract.md — Signed off by PM, Tech Lead, Support Lead
  ✅ eval_dataset.json — 100 tickets (50 clean, 30 medium, 20 edge)
  ✅ eval_harness.py — Running with null system (0% accuracy)
  
Day 2-3: 
  ✅ gpt-4o-mini baseline: 72% accuracy
  ✅ Established improvement target: 90%+

Day 4-10:
  Every code change produces a diff like:
    "system_v1: 78% — better than baseline"
    "system_v2: 84% — improved edge case handling"
    "system_v3: 82% — wait, it regressed. Reverted to v2."

Day 11:
  Best system: 91% accuracy, 96% critical recall
  All contract targets met except latency (2200ms, target 2000ms)
  
Day 12:
  Optimized: 91% accuracy, 96% critical recall, 1800ms p95
  
Day 13:
  Final eval: ALL TARGETS MET ✅
  Monitoring plan deployed alongside system
  Eval debt items documented (3 low-severity items)
```

### What an Eval-Last Project Looks Like

```
Project: Support Ticket Classifier

Week 1-2:
  Built impressive classifier with custom features
  Demo looked great — 95% on self-created test cases
  
Week 3:
  "Let's add evaluation before shipping"
  Curation rushed — 50 cases, not representative
  Eval shows 88% — decent but unclear if real
  
Week 4:
  Deployed with "looks good enough" approval
  
Week 6:
  Production analysis shows 76% accuracy
  Critical tickets missed regularly
  Debugging is hard — no baseline, no trajectory
  
Week 8:
  "We should have built evals first"
  Team spends 2 weeks building proper evaluation
  Rerun: actual accuracy was 76% all along
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The Waterfall Eval

You build the entire system first, then evaluate it after.

```
WRONG:  Build → Build → Build → Build → Evaluate → "Oh no, it's bad"
RIGHT:  Eval spec → Baseline → Build → Evaluate → Build → Evaluate → ...
```

**Why it fails:** When evaluation comes at the end, you can't tell WHICH decision caused the problem. You have no intermediate signal. The cost to fix is highest right when your deadline is closest.

### ❌ Antipattern 2: The Happy Path Dataset

Your eval dataset only contains clean, well-formed examples.

```python
# WRONG: Only clean data
test_cases = [
    {"text": "How do I reset my password?", "category": "account"},
    {"text": "I need a refund for order #12345", "category": "billing"},
]

# RIGHT: Include edge cases, noise, ambiguity
test_cases = [
    {"text": "How do I reset my password?", "category": "account"},
    {"text": "I need a refund for order #12345", "category": "billing"},
    {"text": "reset pwd", "category": "account"},  # Abbreviated
    {"text": "URGENT!!! MY ACCOUNT IS LOCKED!!!!", "category": "account"},  # Shouting
    {"text": "i need a ref", "category": "billing"},  # Incomplete
    {"text": "your app sucks", "category": "other"},  # Venting, no specific issue
]
```

**Why it fails:** Production data is messy. If your eval set is only clean, your eval scores are optimistic. The difference between "works on clean data" and "works on real data" is exactly the gap that frustrates users.

### ❌ Antipattern 3: The Moving Target

You keep changing the eval dataset without versioning it.

```
Week 1: 50 cases, accuracy 88%
Week 2: Add 20 more cases, accuracy 86%
        → "Our system got worse!" (or did the eval get harder?)
Week 3: Remove 10 ambiguous cases, accuracy 90%
        → "We improved!" (or did the eval get easier?)
```

**🔧 Fix:** Version your eval dataset. Use a manifest file that records each version's composition. When you add or remove cases, bump the version. Always compare scores against the SAME eval dataset version.

### ❌ Antipattern 4: The Perfect Baseline

You compare your system against an unrealistically weak baseline.

```python
# WRONG
baseline = random_guesser()  # 5% accuracy
my_system = advanced_classifier()  # 85% accuracy
print("My system beats baseline by 80%!")
# (Misleading — 85% might still be too low for production)

# RIGHT
baseline_cheap = gpt_4o_mini_zeroshot()  # 72% accuracy
baseline_expensive = gpt_4o_fewshot()  # 88% accuracy
my_system = advanced_classifier()  # 85% accuracy
print("My system: 85%. Cheap baseline: 72%. Expensive baseline: 88%.")
print("Decision: Our system is better than cheap but worse than expensive.")
print("We need to improve or justify the tradeoff.")
```

**Why it fails:** A weak baseline makes you look good without telling you whether you're actually good enough for production.

### ❌ Antipattern 5: Eval as a One-Time Activity

You build evaluation for the initial release and never run it again.

```
Ship day: Eval runs, scores are good.
Week 2: Someone updates the system prompt for "better tone."
         No eval is run. The new prompt changes behavior.
Week 4: Users notice the system is less helpful.
         No one knows why. The prompt change is forgotten.
```

**🔧 Fix:** Make evaluation part of your CI/CD pipeline (File 05). Every change — even a prompt change — triggers evaluation. If someone can skip evaluation, someone will.

---

## 🧪 Drills & Challenges

### Drill 1: Write Your Eval Contract (30 min)

You're building a **code review assistant** — an AI that reviews pull requests and flags potential bugs, style issues, and security vulnerabilities.

Write a complete eval contract using the template from Example 1. Include:

1. **Input spec**: What does the system receive? (PR diff, commit message, repo context, etc.)
2. **Output spec**: What does it produce? (List of issues? Severity ratings? Suggested fixes?)
3. **At least 5 metrics** with clear targets and criticality markings
4. **Dataset spec**: What kind of test cases do you need? How many? How are they annotated?
5. **Baseline**: What cheap approach would establish a floor?

**Bonus:** Identify which metrics might conflict. For example, a system that flags EVERYTHING as a bug catches all real bugs (high recall) but is useless (low precision). How does your metric set balance this?

---

### Drill 2: The Baseline Experiment (25 min)

Pick a simple AI task — sentiment analysis, text classification, or question answering. 

1. Write a null system (returns a random/default value)
2. Write the CHEAPEST possible baseline (GPT-4o-mini zero-shot)
3. Run both on 10 test cases
4. Record the scores
5. Set a target that represents "good enough for production"

**Your deliverable:** A one-page report showing the score progression from null → cheap baseline → target. This is your eval-first starting point.

---

### Drill 3: Diagnose the Eval Debt (20 min)

Your team has been running an AI system for 3 months with NO evaluation infrastructure. Here's what has happened:

- Week 2: System deployed. Initial accuracy estimated at "looks good" (~85%)
- Week 5: Prompt update added for "better formatting." No one measured the impact.
- Week 8: User complaints about wrong answers increased 30%. Team can't tell why.
- Week 10: Investigation reveals Week 5's prompt change dropped accuracy 12%. Reverting fixes it.
- Week 12: Team decides to "add evaluation." But they're also building 2 new features.

**Identify 5 eval debt items from this timeline.** For each:
1. What caused it?
2. What was the "interest paid"?
3. What severity is it?
4. What would have prevented it?

---

### Drill 4: The Stakeholder Conversation (20 min)

Your PM says: "I need this feature in 2 weeks. If you spend 3 days on evaluation, we won't make the deadline."

**Your task:** Write the conversation — not a scripted argument, but a realistic back-and-forth where you convince the PM (or compromise). Address:

1. What happens if we skip evaluation and the feature is buggy?
2. What's the minimum viable evaluation we can do in 1 day?
3. How do we track eval debt for what we're deferring?
4. What's the "no go" condition — when do we refuse to ship without eval?

---

### Drill 5: Eval-First Refactoring (30 min)

Take any AI system you've built in a previous phase (or that exists in this course). Imagine you're rebuilding it from scratch using the eval-first approach.

1. Open the system's original code
2. Identify what evaluation (if any) existed
3. Write an eval contract FOR THAT SYSTEM as if you were building it today
4. Identify at least 3 things the original evaluation missed
5. Estimate how much time the eval-first approach would have saved overall (include debugging time saved, not just eval time spent)

---

## 🚦 Gate Check

Before moving to File 02, verify you can:

1. **☐** Explain the 5 phases of the eval-first workflow
2. **☐** Write a complete eval contract for any AI system
3. **☐** Build an eval harness that runs with a null system
4. **☐** Differentiate eval debt from tech debt and explain why it compounds
5. **☐** Identify at least 3 eval antipatterns from your own experience
6. **☐** Explain why "eval last" consistently fails, with concrete reasoning

### 🛑 Stop and Think

**Before you proceed, answer this:**

You've just read about eval-first. It makes sense intellectually. But next time you're in a time crunch — and you WILL be — what specifically will stop you from skipping evaluation? Is it a process (code review requires eval results)? Is it a tool (eval harness auto-runs)? Is it a habit (you feel wrong starting without eval)?

**Your answer to "what prevents me from skipping evaluation" should be structural, not motivational.** If your answer is "I'll just be more disciplined," you'll skip evaluation the next time you're under pressure. What structure will you put in place?

---

## 📚 Resources

- **"Eval-First Development for AI Systems"** (Anthropic, 2025) — The original essay that defined the eval-first approach. Short, practical, worth reading every 6 months as a reset.
- **"The ML Test Score: A Rubric for ML Production Readiness"** (Google, 2020) — The classic paper on ML evaluation maturity. Still relevant for LLM systems. Adapts well to eval contracts.
- **"Hidden Technical Debt in Machine Learning Systems"** (Google, 2015) — The foundational paper on ML tech debt. Read this to understand why eval debt is the most toxic kind.
- **"Unit Testing for LLMs: A Practical Guide"** (Hamel Husain, 2025) — Practical guide that influenced this module's eval contract pattern.
- **"The Cost of Not Testing AI"** (Modal Blog, 2025) — Case study of a company that lost $50K+ because of insufficient evaluation. The ROI math for eval infrastructure is compelling.

**Next Module:**
→ **02-LLM-as-Judge.md**: Building reliable LLM judges — biases, calibration, ensembles, and structured eval prompts.
