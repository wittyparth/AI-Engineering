# 03 — Metrics That Matter: What to Measure and What's Lying to You

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about metrics, you need to understand why most metrics are actually working against you.

There's a famous quote in management: *"What gets measured gets managed."*

There's a LESS famous follow-up that's more important: *"What gets measured also gets GAMED."*

Goodhart's Law states: *"When a measure becomes a target, it ceases to be a good measure."*

AI evaluation is FULL of this. Every metric you use, someone (or some system) will optimize for it. And the moment that happens, the metric stops telling you what you actually want to know.

This module isn't about listing every possible metric. It's about learning to THINK about metrics — how to choose them, how to know when they're lying, and how to design a metric suite that survives optimization pressure.

The three discovery questions below are designed to surface the real problem. Don't rush through them.

---

### 🤔 Discovery Question 1: The 99% Accuracy Trap

You build a content moderation system. It classifies comments as "toxic" or "safe." You test it on 10,000 labeled comments.

| Metric | Value |
|--------|-------|
| Accuracy | 99.1% |
| Precision (toxic) | 95% |
| Recall (toxic) | 45% |

You show the CTO your accuracy metric. She's impressed. You deploy.

**Within a week, you have a PR disaster.** A hate comment slipped through. The press picks it up. The CTO asks: "I thought accuracy was 99%?"

**🤔 Before reading on, answer these:**

1. Why did accuracy lie to you? What's the relationship between the base rate of toxic comments and why accuracy is misleading here?

2. You look closer at the data: only 1.5% of all comments are toxic. Your model correctly identifies 99.9% of SAFE comments but only 45% of TOXIC ones. The 99.1% accuracy is just "predicting 'safe' for everything is right 98.5% of the time." What metric SHOULD you have used instead? 

3. The product team now wants a target: "Improve toxic recall to 90%." You implement it. Recall goes to 90%... but precision drops to 40%. Now you're flagging half your safe comments as toxic. Your moderation team is overwhelmed. What happened? Is there a metric that captures this tradeoff without the gamification?

4. **Cross-reference to Phase 2:** Remember prompt engineering? A prompt that's "95% accurate at classifying sentiment" might achieve that by being conservative — defaulting to "neutral" when uncertain. How would you detect this if your only metric is accuracy? What additional metric would catch the "plays it safe" behavior?

---

### 🤔 Discovery Question 2: The RAG Metrics That Contradict Each Other

You're optimizing a RAG system. You track two key metrics from Phase 4:

- **Faithfulness:** Does the answer stick to the retrieved context? (0-1)
- **Answer Relevance:** Does the answer actually address the question? (0-1)

You make a change: you increase the number of retrieved chunks from 3 to 8.

| Metric | Before (top-3) | After (top-8) | Change |
|--------|----------------|---------------|--------|
| Faithfulness | 0.94 | 0.88 | -6% ❌ |
| Answer Relevance | 0.82 | 0.91 | +9% ✅ |
| Context Recall | 0.71 | 0.88 | +17% ✅ |

**Faithfulness went DOWN. Relevance went UP. Which change is better?**

**🤔 Before reading on, answer these:**

1. Why did faithfulness drop when you added more chunks? Think about it — the model has MORE context now. Shouldn't it be MORE faithful because it has more information? Or... is there something about having MORE context that INCREASES the chance of contradiction or hallucination?

2. Why did answer relevance improve? More chunks means more coverage. But also more noise. At what point does adding MORE chunks HURT relevance? Is there an optimal number?

3. You show these numbers to your PM. She says: "Faithfulness dropped 6%, relevance improved 9%. The net is +3%. Ship it." Is she right? How do you know? What if the 6% drop is concentrated on HIGH-value queries (e.g., medical or financial advice where faithfulness is critical)?

4. **This is the hard one:** Faithfulness and relevance are measured by TWO DIFFERENT LLM judges. What if the faithfulness judge is biased toward shorter contexts (and the longer context from top-8 triggers position bias)? What if the relevance judge is biased toward longer answers (verbosity bias)? The "improvement" might be an artifact of your judges, not your system. How would you RULE THIS OUT?

---

### 🤔 Discovery Question 3: The Business Impact Gap

You run a customer support chatbot. Your eval dashboard shows:

| Metric | This Month | Target | Status |
|--------|-----------|--------|--------|
| Answer Accuracy | 94% | ≥92% | ✅ |
| Response Time | 1.8s | <2s | ✅ |
| User Satisfaction | 4.2/5 | ≥4.0 | ✅ |

All green. Ship it, right?

But the VP of Support emails you: "Our ticket deflection rate dropped 8% this month. More customers are escalating to human agents. What's going on?"

Your chatbot passes every technical metric. But business outcomes are WORSE.

**🤔 Before reading on, answer these:**

1. What's the gap between "the answer is correct" and "the customer's problem is solved"? Think of a scenario where a chatbot gives a PERFECTLY correct answer that still doesn't help the customer.

2. Your accuracy metric measures: "Did the answer contain the correct information?" But it does NOT measure: "Did the customer act on the information?" Give me 3 examples where correct answers lead to no action or wrong action.

3. **The metric design challenge:** Design a metric that captures whether the chatbot actually RESOLVED the customer's issue. You can't use surveys (low response rate, selection bias). You can't use human review at scale (too expensive). What PROXY metric would you use that correlates with real resolution? Think about behavioral signals: what does a CUSTOMER do when their issue is resolved vs. not resolved?

4. **Cross-reference to Phase 4:** Remember how RAG evaluation has component-level metrics (chunk retrieval precision) and system-level metrics (answer faithfulness)? The same hierarchy exists here: technical metrics (accuracy, latency) → product metrics (deflection rate, resolution time) → business metrics (customer retention, cost per ticket). If you could only monitor ONE level continuously, which one? And what's the cost of being wrong about that choice?

---

> **Keep these 3 problems in mind. They reveal the fundamental challenge of metrics: no single metric tells the full story, and every metric can be gamed.**

By the end of this module, you will:

- Choose the RIGHT metrics for different AI systems (RAG, agents, classifiers, generators)
- Understand the 3-level metric hierarchy: component → system → business
- Detect when metrics are lying to you (Goodhart's Law in action)
- Design metric suites that capture tradeoffs (not single numbers)
- Build a metric selection framework you can apply to ANY new AI system
- Know the most common metric mistakes and how to avoid them

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions (above) | 25 min — really engage with each |
| The 3-Level Metric Hierarchy | 20 min |
| Metrics for RAG Systems | 25 min |
| Metrics for Agents | 25 min |
| Metrics for Classifiers | 15 min |
| Metrics for Generators | 15 min |
| Code: Building a Metric Suite | 40 min |
| Code: Detecting Metric Gaming | 30 min |
| Code: Metric Selection Framework | 25 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 10 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The 3-Level Metric Hierarchy

Every AI system has metrics at 3 levels. Understanding which level you're measuring is the most important skill in evaluation.

```
LEVEL 1: COMPONENT METRICS
  What they measure: Individual pieces of the system
  Examples: Retrieval precision, tool call accuracy, embedding quality
  Cost to measure: Cheap ($0.001-$0.01 per eval)
  Can be gamed: YES — easily
  Tells you: "Is part X working correctly?"
  Does NOT tell you: Whether the overall system is good

LEVEL 2: SYSTEM METRICS
  What they measure: End-to-end system quality
  Examples: Answer accuracy, task success rate, user satisfaction
  Cost to measure: Medium ($0.01-$0.10 per eval)
  Can be gamed: YES — with effort
  Tells you: "Is the system producing good outputs?"
  Does NOT tell you: Whether those outputs drive business value

LEVEL 3: BUSINESS METRICS
  What they measure: Real-world impact
  Examples: Retention rate, ticket deflection, revenue per user
  Cost to measure: High (requires analytics infrastructure)
  Can be gamed: HARD — but possible
  Tells you: "Is the system creating value?"
  Does NOT tell you: Why — needs Level 1/2 to diagnose
```

**🤔 Checkpoint:** Most teams measure Level 2 (system metrics) almost exclusively. They ignore Level 1 (can't diagnose failures) and Level 3 (can't prove value). If you had to choose TWO levels to invest in, which ones? Your answer determines your debugging capability AND your ability to justify the system's existence to stakeholders.

### 2. Metrics by AI System Type

#### RAG Systems

| Category | Metric | What It Measures | How It Lies |
|----------|--------|-----------------|-------------|
| **Retrieval** | Context Precision | % of retrieved chunks that are relevant | Can be 100% if you retrieve 1 chunk (easy) but miss others |
| **Retrieval** | Context Recall | % of relevant chunks that were retrieved | Can be 100% if you retrieve all chunks (trivial) but drown the LLM |
| **Generation** | Faithfulness | Does answer stick to context? | LLM can be faithful to WRONG context (retrieval failure hidden) |
| **Generation** | Answer Relevance | Does answer address the question? | LLM can answer a DIFFERENT question with high relevance |
| **Holistic** | F1 Score | Harmonic mean of precision & recall | Hides which side is failing |

**The RAG metric lie:** A system can score 0.90+ on ALL RAG metrics while being useless. How? By retrieving irrelevant chunks but answering from the LLM's training data (faithfulness to context will be low, but the judge might not catch it if the answer sounds correct). Your metrics only catch what you explicitly measure.

#### Agent Systems

See Phase 6 File 08 for the full treatment. Summary:

| Category | Metric | What It Measures | How It Lies |
|----------|--------|-----------------|-------------|
| Decision | Tool Selection Accuracy | Did it pick the right tool? | Can choose the RIGHT tool with the WRONG parameters |
| Decision | Step Progression | Did each step move toward the goal? | Can progress FAST toward the WRONG goal |
| Efficiency | Steps per Task | How many steps needed? | FEWER steps isn't always better — might skip verification |
| Efficiency | Cost per Task | How much did it cost? | Cheap agents might produce more errors (hidden cost) |
| Safety | Violation Rate | Did it do something dangerous? | Zero violations might mean it was too conservative (helpful? no) |

**The agent metric lie:** A perfectly scored agent trajectory (correct tools, efficient steps, zero violations) can still produce a wrong final answer (Phase 6, Scenario 1 from File 08). The opposite also holds: a correct final answer can hide terrible decisions that happened to luck out.

#### Classifiers

| Category | Metric | What It Measures | How It Lies |
|----------|--------|-----------------|-------------|
| Basic | Accuracy | % of correct predictions | Worthless on imbalanced data (see Discovery Q1) |
| Balance | Precision | % of positive predictions that were correct | Can be 100% by only predicting positive when certain (low recall) |
| Balance | Recall | % of actual positives that were found | Can be 100% by predicting positive for everything (low precision) |
| Balance | F1 | Harmonic mean of precision & recall | Doesn't tell you which is worse (need both numbers) |
| Calibration | ECE | How well do confidence scores match actual accuracy? | Can be perfect while accuracy is terrible (if always 50% confident and 50% accurate) |

**The classifier metric lie:** F1 of 0.95 can hide severe calibration issues. Your model might be 95% precise AND 95% recall but completely mis-calibrated — 99% confident on wrong predictions. When you need CONFIDENCE (not just classification), F1 isn't enough.

#### Generators (Summarization, Translation, Creative)

| Category | Metric | What It Measures | How It Lies |
|----------|--------|-----------------|-------------|
| Lexical | ROUGE | N-gram overlap with reference | Penalizes valid rephrasing, favors word-for-word matches |
| Semantic | BERTScore | Embedding similarity with reference | Can't catch factual errors if phrasing is similar |
| LLM-Judge | Overall Score | LLM's holistic assessment | Biased by verbosity, position, self-enhancement (File 02) |
| Task-Specific | Task Success | Did the output achieve the goal? | Hard to measure automatically for creative tasks |

**The generator metric lie:** ROUGE scores correlate POORLY with human judgment for creative tasks (0.15-0.30 correlation). Teams optimize for ROUGE and get systems that produce safe, generic, boring outputs. The metric drives the wrong behavior.

---

### 3. Metric Design Principles

Based on the above, here are principles for designing metric suites:

**Principle 1: Always Use Paired Metrics**

Never use a single metric. Every metric needs a counter-metric that captures the opposite failure mode.

| Metric | Pair With | Why |
|--------|-----------|-----|
| Accuracy | Balanced Accuracy (per-class) | Accuracy hides class imbalance |
| Precision | Recall | Optimizing either alone destroys the other |
| Faithfulness | Answer Relevance | Faithful but irrelevant = useless |
| Efficiency | Success Rate | Fast but wrong = useless |
| Cost per Task | Task Quality | Cheap but bad = waste of money |

**Principle 2: Measure What You Want, Not What's Easy**

```
EASY TO MEASURE          HARD TO MEASURE           WHAT YOU ACTUALLY WANT
────────────────────────────────────────────────────────────────────────
Answer accuracy          User problem resolved     Retained customers
Response time            Time-to-resolution        Customer satisfaction
Tool call success        Task completion           Business outcome achieved
```

If you can't measure what you WANT, at least ACKNOWLEDGE the gap between your proxy and your true goal.

**Principle 3: Track Distributions, Not Just Averages**

```
Average accuracy: 94%  ← hides everything
                          
Per-decile accuracy:    
  0-10% (hardest queries): 72%    ← THIS is where you should focus
 10-20%:                        88%
 20-30%:                        92%
 30-100%:                      97%+
```

The average hides the tail. In production, the tail (hardest queries) determines user perception.

**Principle 4: The Metric Should Be Hard to Game**

Ask: "If someone optimized EXCLUSIVELY for this metric, would the system get better or worse?"

| Metric | Gamified Behavior | Better Alternative |
|--------|-------------------|-------------------|
| Accuracy | Predict majority class | Balanced accuracy or F1 |
| User rating | Ask only happy users (survivorship bias) | Random sampling + NPS |
| Response time | Give up on hard queries | Timeout with fallback + p95 latency |
| Faithfulness | Retrieve nothing (can't be unfaithful to empty context) | Faithfulness + context recall |

---

## 💻 Code Examples

### Example 1: Building a Metric Suite (Not a Single Metric)

```python
"""
metric_suite.py

A complete metric suite for an AI system.
No single metric — always a PAIR or TRIO that captures tradeoffs.

Notice: this code is designed to make you THINK about metric interactions,
not just compute numbers.
"""

from dataclasses import dataclass
from typing import Any, Callable, Optional
import statistics
import json


@dataclass
class MetricResult:
    """Result of a single metric computation."""
    name: str
    value: float
    count: int  # Number of data points
    notes: str = ""


@dataclass
class MetricSuiteResult:
    """Results from a complete metric suite."""
    system_name: str
    metrics: list[MetricResult]
    
    def get(self, name: str) -> Optional[MetricResult]:
        for m in self.metrics:
            if m.name == name:
                return m
        return None
    
    def summary(self) -> str:
        lines = [f"═══ Metric Suite: {self.system_name} ═══"]
        
        for m in self.metrics:
            lines.append(f"  {m.name}: {m.value:.4f} (n={m.count})")
            if m.notes:
                lines.append(f"    → {m.notes}")
        
        # Check for contradictions (pairs that should be considered together)
        contradictions = []
        
        accuracy = self.get("accuracy")
        balanced_acc = self.get("balanced_accuracy")
        if accuracy and balanced_acc:
            gap = accuracy.value - balanced_acc.value
            if gap > 0.1:
                contradictions.append(
                    f"⚠️ Accuracy ({accuracy.value:.3f}) vs Balanced Accuracy "
                    f"({balanced_acc.value:.3f}) — gap of {gap:.3f} suggests "
                    f"class imbalance is hiding failures"
                )
        
        precision = self.get("precision")
        recall = self.get("recall")
        if precision and recall:
            if precision.value > 0.95 and recall.value < 0.7:
                contradictions.append(
                    f"⚠️ High precision ({precision.value:.3f}) but low recall "
                    f"({recall.value:.3f}) — system is too conservative, missing cases"
                )
            elif recall.value > 0.95 and precision.value < 0.7:
                contradictions.append(
                    f"⚠️ High recall ({recall.value:.3f}) but low precision "
                    f"({precision.value:.3f}) — system is too aggressive, false positives"
                )
        
        cost = self.get("cost_per_query")
        success = self.get("success_rate")
        if cost and success:
            lines.append(f"\n  Cost-Efficiency: ${cost.value:.4f}/query at {success.value:.1%} success")
        
        if contradictions:
            lines.append("\n⚠️ METRIC CONTRADICTIONS DETECTED:")
            for c in contradictions:
                lines.append(f"  {c}")
        
        return "\n".join(lines)


class MetricSuite:
    """
    A collection of metrics designed to work together.
    
    Key design principles:
    1. Every important metric has a COUNTER-METRIC
    2. The suite detects contradictions between metrics
    3. Distributions are tracked, not just averages
    """
    
    def __init__(self, name: str):
        self.name = name
        self.metric_fns: dict[str, Callable] = {}
    
    def register(self, name: str, fn: Callable):
        """Register a metric function."""
        self.metric_fns[name] = fn
    
    def evaluate(self, predictions: list[Any], ground_truth: list[Any]) -> MetricSuiteResult:
        """Run all registered metrics on a dataset."""
        results = []
        
        for name, fn in self.metric_fns.items():
            try:
                value, count, notes = fn(predictions, ground_truth)
                results.append(MetricResult(
                    name=name, value=value, count=count, notes=notes
                ))
            except Exception as e:
                print(f"Metric '{name}' failed: {e}")
        
        return MetricSuiteResult(system_name=self.name, metrics=results)


# =============================================
# METRIC DEFINITIONS
# =============================================

def compute_accuracy(predictions, ground_truth) -> tuple:
    """Simple accuracy — but also returns per-class breakdown."""
    correct = sum(1 for p, g in zip(predictions, ground_truth) if p == g)
    total = len(predictions)
    accuracy = correct / total if total > 0 else 0.0
    
    return accuracy, total, ""


def compute_balanced_accuracy(predictions, ground_truth) -> tuple:
    """
    Accuracy computed PER CLASS, then averaged.
    This reveals if the model is good on one class but terrible on others.
    """
    # Group predictions and ground truth by class
    classes = set(ground_truth)
    per_class_accuracies = []
    
    for cls in classes:
        cls_indices = [i for i, g in enumerate(ground_truth) if g == cls]
        if not cls_indices:
            continue
        cls_correct = sum(1 for i in cls_indices if predictions[i] == ground_truth[i])
        cls_acc = cls_correct / len(cls_indices)
        per_class_accuracies.append(cls_acc)
    
    balanced_acc = statistics.mean(per_class_accuracies) if per_class_accuracies else 0.0
    
    # Find the worst class
    class_accs = {}
    for cls in classes:
        indices = [i for i, g in enumerate(ground_truth) if g == cls]
        correct = sum(1 for i in indices if predictions[i] == ground_truth[i])
        class_accs[cls] = correct / len(indices) if indices else 0.0
    
    worst_class = min(class_accs, key=class_accs.get)
    worst_acc = class_accs[worst_class]
    
    notes = f"Worst class: '{worst_class}' with {worst_acc:.1%} accuracy"
    
    return balanced_acc, len(predictions), notes


def compute_precision_recall(predictions, ground_truth, positive_class="toxic") -> tuple:
    """Precision-recall pair for a specific class."""
    true_positives = sum(
        1 for p, g in zip(predictions, ground_truth)
        if p == positive_class and g == positive_class
    )
    false_positives = sum(
        1 for p, g in zip(predictions, ground_truth)
        if p == positive_class and g != positive_class
    )
    false_negatives = sum(
        1 for p, g in zip(predictions, ground_truth)
        if p != positive_class and g == positive_class
    )
    
    precision = true_positives / (true_positives + false_positives) if (true_positives + false_positives) > 0 else 0.0
    recall = true_positives / (true_positives + false_negatives) if (true_positives + false_negatives) > 0 else 0.0
    
    # Return precision and recall together (they're a PAIR)
    return precision, len(predictions), recall  # Returning 3 values — see evaluate() for handling


# Note: The above function returns 3 values, but the metric framework expects 3 values (value, count, notes).
# This is a DESIGN BUG in our framework — precision and recall should be registered as TWO separate metrics
# from one function, or we need a multi-metric return.
# Let's fix this:

class PairedMetric:
    """
    A metric that returns multiple values (like precision AND recall).
    The MetricSuite registers each as a separate metric automatically.
    """
    def __init__(self, name_pair: tuple[str, str], fn: Callable):
        self.name1, self.name2 = name_pair
        self.fn = fn
    
    def compute(self, predictions, ground_truth) -> tuple[MetricResult, MetricResult]:
        v1, count, v2 = self.fn(predictions, ground_truth)
        return (
            MetricResult(name=self.name1, value=v1, count=count, notes=""),
            MetricResult(name=self.name2, value=v2, count=count, notes=""),
        )


# === REAL EXAMPLE: Evaluating a RAG system with paired metrics ===

def evaluate_rag_metrics(
    questions: list[str],
    retrieved_chunks: list[list[str]],
    generated_answers: list[str],
    reference_answers: list[str],
    faithfulness_judge: Callable,
    relevance_judge: Callable,
) -> MetricSuiteResult:
    """
    Evaluate a RAG system using PAIRED metrics.
    
    Every metric has a counterpart that captures the opposite failure mode.
    """
    results = []
    
    # Parallel metric 1: Context precision (are retrieved chunks relevant?)
    # If this is high but recall is low → chunks are good but we're missing content
    context_precisions = []
    for chunks, question in zip(retrieved_chunks, questions):
        # Judge how many chunks are relevant
        relevant = sum(1 for c in chunks if is_chunk_relevant(c, question))
        precision = relevant / len(chunks) if chunks else 0
        context_precisions.append(precision)
    
    avg_precision = statistics.mean(context_precisions)
    results.append(MetricResult(
        name="context_precision",
        value=avg_precision,
        count=len(questions),
        notes="% of retrieved chunks that are relevant. High = good retrieval focus.",
    ))
    
    # Parallel metric 2: Context recall (did we retrieve enough relevant chunks?)
    # If this is low but precision is high → we're being too conservative
    # If both are low → retrieval is fundamentally broken
    context_recalls = []
    for chunks, question in zip(retrieved_chunks, questions):
        # This requires knowing ALL relevant chunks (hard in practice)
        # Proxy: ask an LLM if the retrieved chunks collectively cover the question
        recall = estimate_chunk_recall(chunks, question)
        context_recalls.append(recall)
    
    avg_recall = statistics.mean(context_recalls)
    results.append(MetricResult(
        name="context_recall",
        value=avg_recall,
        count=len(questions),
        notes="% of needed information covered by retrieved chunks. High = comprehensive retrieval.",
    ))
    
    # Check contradiction: high precision + low recall = too conservative
    if avg_precision > 0.9 and avg_recall < 0.6:
        results.append(MetricResult(
            name="⚠️ RETRIEVAL_CONTRADICTION",
            value=1.0,
            count=1,
            notes=f"Precision {avg_precision:.2f} but recall {avg_recall:.2f}. "
                  "Retrieval is too conservative — missing relevant chunks "
                  "while only returning 'safe' ones. Consider increasing top_k "
                  "or adjusting similarity threshold.",
        ))
    
    # Parallel metric 3: Faithfulness (does answer stick to context?)
    # High faithfulness + low relevance = answers are safe but useless
    faithfulness_scores = []
    for ans, chunks in zip(generated_answers, retrieved_chunks):
        score = faithfulness_judge(ans, chunks)
        faithfulness_scores.append(score)
    
    avg_faithfulness = statistics.mean(faithfulness_scores)
    results.append(MetricResult(
        name="faithfulness",
        value=avg_faithfulness,
        count=len(questions),
        notes="Does the answer stick to the retrieved context? "
              "High = no hallucination. But doesn't measure usefulness.",
    ))
    
    # Parallel metric 4: Answer relevance (does answer address the question?)
    # High relevance + low faithfulness = hallucinating useful-sounding info
    relevance_scores = []
    for ans, question in zip(generated_answers, questions):
        score = relevance_judge(question, ans)
        relevance_scores.append(score)
    
    avg_relevance = statistics.mean(relevance_scores)
    results.append(MetricResult(
        name="answer_relevance",
        value=avg_relevance,
        count=len(questions),
        notes="Does the answer address the question? "
              "High = user gets what they asked for. But might hallucinate.",
    ))
    
    return MetricSuiteResult(system_name="RAG_System", metrics=results)


def is_chunk_relevant(chunk: str, question: str) -> bool:
    """Simplified relevance check. In production, use embedding similarity or LLM."""
    # Simple keyword overlap (placeholder)
    question_words = set(question.lower().split())
    chunk_words = set(chunk.lower().split())
    overlap = len(question_words & chunk_words)
    return overlap > 2


def estimate_chunk_recall(chunks: list[str], question: str) -> float:
    """Estimate if chunks cover the question adequately."""
    # In production: ask LLM "Do these chunks contain enough info to answer the question?"
    # Simplified: check keyword coverage
    question_words = set(question.lower().split())
    covered_words = set()
    for chunk in chunks:
        covered_words |= set(chunk.lower().split())
    coverage = len(question_words & covered_words) / max(len(question_words), 1)
    return min(coverage, 1.0)


# === Usage ===
if __name__ == "__main__":
    # Simulated evaluation
    questions = [
        "What is the return policy for electronics?",
        "How do I reset my password?",
        "Do you offer international shipping?",
    ]
    
    result = evaluate_rag_metrics(
        questions=questions,
        retrieved_chunks=[
            ["Return policy: electronics have a 30-day return window..."],
            ["Password reset: go to settings → security → reset password..."],
            ["We ship to 50 countries worldwide..."],
        ],
        generated_answers=[
            "Electronics can be returned within 30 days.",
            "Go to your account settings to reset your password.",
            "Yes, we ship to 50 countries.",
        ],
        reference_answers=[
            "Electronics can be returned within 30 days in original packaging.",
            "Go to Settings > Security > Reset Password.",
            "Yes, we ship to 50 countries. Shipping costs vary by destination.",
        ],
        faithfulness_judge=lambda a, c: 1.0,  # Placeholder
        relevance_judge=lambda q, a: 1.0,  # Placeholder
    )
    
    print(result.summary())
```

**🤔 The real question:** The code above measures precision and recall independently. But it ALSO has a contradiction detector that flags when precision and recall are misaligned. Is this sufficient? What if BOTH precision AND recall are 0.9+ but the system is STILL bad? Think about a scenario where both retrieval metrics are high but the system produces terrible answers. What's missing from the metric suite?

---

### Example 2: The Metric That Lies — Goodhart's Law Detection

This code detects when a metric has been "gamed" — when the system optimizes for the metric at the expense of actual quality.

```python
"""
goodhart_detector.py

Detects when metrics are being gamed (Goodhart's Law in action).

The key insight: when a metric is gamed, you see specific patterns:
1. The metric improves, but related metrics DON'T (uncoupling)
2. The metric shows less variance (compression toward target)
3. Edge cases start failing (overfitting to the metric)
"""

from dataclasses import dataclass
from typing import Any
import statistics
import math


@dataclass
class MetricHistory:
    """History of a metric over time."""
    name: str
    values: list[float]
    timestamps: list[str]
    
    @property
    def mean(self) -> float:
        return statistics.mean(self.values) if self.values else 0.0
    
    @property
    def stdev(self) -> float:
        return statistics.stdev(self.values) if len(self.values) > 1 else 0.0
    
    @property
    def recent_trend(self) -> float:
        """Slope of last 10 points. Positive = improving."""
        if len(self.values) < 10:
            return 0.0
        recent = self.values[-10:]
        return recent[-1] - recent[0]


class GoodhartDetector:
    """
    Detects metric gaming by looking for suspicious patterns.
    
    Three signals:
    1. UNCOUPLING: Primary metric improves, but secondary metrics don't
    2. COMPRESSION: Metric variance decreases unusually (too consistent)
    3. EDGE_DEGRADATION: Performance on hard cases drops while easy cases improve
    """
    
    def __init__(self):
        self.histories: dict[str, MetricHistory] = {}
    
    def record(self, name: str, value: float, timestamp: str = ""):
        if name not in self.histories:
            self.histories[name] = MetricHistory(name=name, values=[], timestamps=[])
        self.histories[name].values.append(value)
        self.histories[name].timestamps.append(timestamp)
    
    def check_uncoupling(
        self,
        primary_metric: str,
        secondary_metrics: list[str],
        window: int = 20,
    ) -> list[str]:
        """
        Check if primary metric improved but secondary metrics didn't.
        This suggests optimization for the primary metric alone.
        """
        if primary_metric not in self.histories:
            return ["No data for primary metric"]
        
        primary = self.histories[primary_metric]
        if len(primary.values) < window:
            return ["Insufficient data"]
        
        primary_trend = primary.recent_trend
        warnings = []
        
        for sec_name in secondary_metrics:
            if sec_name not in self.histories:
                continue
            secondary = self.histories[sec_name]
            sec_trend = secondary.recent_trend
            
            # If primary improved but secondary didn't (or worsened)
            if primary_trend > 0.05 and sec_trend < 0.01:
                warnings.append(
                    f"UNCOUPLING: {primary_metric} improved by {primary_trend:.3f} "
                    f"but {sec_name} only changed by {sec_trend:.3f}. "
                    f"Optimization may be targeting {primary_metric} exclusively."
                )
        
        return warnings
    
    def check_compression(self, metric_name: str, threshold: float = 0.5) -> list[str]:
        """
        Check if metric variance has decreased suspiciously.
        Very low variance + high mean = too consistent = possibly gamed.
        """
        if metric_name not in self.histories:
            return ["No data"]
        
        metric = self.histories[metric_name]
        if len(metric.values) < 20:
            return ["Insufficient data"]
        
        # Compare variance in first half vs second half
        mid = len(metric.values) // 2
        first_half = metric.values[:mid]
        second_half = metric.values[mid:]
        
        first_std = statistics.stdev(first_half) if len(first_half) > 1 else 0
        second_std = statistics.stdev(second_half) if len(second_half) > 1 else 0
        
        warnings = []
        
        # If variance dropped significantly while mean stayed same or improved
        if first_std > 0 and second_std / first_std < threshold:
            mean_first = statistics.mean(first_half)
            mean_second = statistics.mean(second_half)
            
            warnings.append(
                f"COMPRESSION: {metric_name} variance dropped from "
                f"{first_std:.4f} to {second_std:.4f} "
                f"({(1 - second_std/first_std)*100:.0f}% reduction) "
                f"while mean went from {mean_first:.4f} to {mean_second:.4f}. "
                f"Suspiciously consistent — possible metric gaming."
            )
        
        return warnings
    
    def check_edge_degradation(
        self,
        metric_name: str,
        easy_scores: list[float],
        hard_scores: list[float],
    ) -> list[str]:
        """
        Check if easy cases improved but hard cases got worse.
        Classic overfitting signal.
        """
        warnings = []
        
        easy_mean = statistics.mean(easy_scores) if easy_scores else 0
        hard_mean = statistics.mean(hard_scores) if hard_scores else 0
        
        if easy_scores and hard_scores and len(easy_scores) > 5 and len(hard_scores) > 5:
            easy_trend = easy_scores[-1] - easy_scores[0]
            hard_trend = hard_scores[-1] - hard_scores[0]
            
            if easy_trend > 0.1 and hard_trend < -0.05:
                warnings.append(
                    f"EDGE DEGRADATION: {metric_name} improved on easy cases "
                    f"({easy_trend:+.3f}) but degraded on hard cases "
                    f"({hard_trend:+.3f}). System is optimizing for the easy path "
                    f"at the expense of handling difficult queries."
                )
        
        return warnings


# === Usage ===
if __name__ == "__main__":
    detector = GoodhartDetector()
    
    # Simulate a month of daily accuracy measurements
    # Notice: accuracy goes up, but balanced accuracy stays flat
    for day in range(30):
        detector.record("accuracy", 0.90 + day * 0.003, f"day_{day}")
        detector.record("balanced_accuracy", 0.75 + (day % 5) * 0.01, f"day_{day}")
        detector.record("faithfulness", 0.88 + day * 0.001, f"day_{day}")
    
    warnings = detector.check_uncoupling(
        primary_metric="accuracy",
        secondary_metrics=["balanced_accuracy", "faithfulness"],
    )
    
    for w in warnings:
        print(w)
    
    # Check for compression
    compression_warnings = detector.check_compression("accuracy")
    for w in compression_warnings:
        print(w)
```

---

### Example 3: Distribution-Aware Metrics (Not Just Averages)

```python
"""
tail_metrics.py

Most metrics report only the AVERAGE. This hides the tail — the hardest
queries that determine user perception.

This module computes per-decile metrics to reveal tail performance.
"""

from dataclasses import dataclass
from typing import Any
import statistics


@dataclass
class DecileBreakdown:
    """Metric values broken down by difficulty decile."""
    metric_name: str
    decile_scores: list[float]  # 10 values, one per decile
    overall_average: float
    
    @property
    def worst_decile(self) -> float:
        return self.decile_scores[0]  # Decile 0 = hardest
    
    @property
    def best_decile(self) -> float:
        return self.decile_scores[-1]  # Decile 9 = easiest
    
    @property
    def tail_gap(self) -> float:
        """Difference between best and worst decile."""
        return self.best_decile - self.worst_decile
    
    def summary(self) -> str:
        lines = [
            f"═══ {self.metric_name} — Decile Breakdown ═══",
            f"Overall average: {self.overall_average:.4f}",
            f"Worst decile (hardest queries): {self.worst_decile:.4f}",
            f"Best decile (easiest queries): {self.best_decile:.4f}",
            f"Tail gap: {self.tail_gap:.4f}",
            "",
            "Decile scores:",
        ]
        for i, score in enumerate(self.decile_scores):
            marker = " ← TAIL" if i == 0 else ""
            lines.append(f"  Decile {i}: {score:.4f}{marker}")
        
        if self.tail_gap > 0.2:
            lines.append(
                f"\n⚠️ Large tail gap ({self.tail_gap:.3f}): the system performs "
                f"MUCH better on easy queries than hard ones. "
                f"User perception will be driven by the tail, not the average."
            )
        
        return "\n".join(lines)


def compute_decile_breakdown(
    scores_per_query: list[float],
    difficulty_labels: list[float],  # Lower = harder
    metric_name: str = "metric",
    num_deciles: int = 10,
) -> DecileBreakdown:
    """
    Break down metric scores by query difficulty.
    
    Args:
        scores_per_query: Metric score for each query (e.g., accuracy, faithfulness)
        difficulty_labels: Difficulty score for each query (lower = harder)
                           Could be length, ambiguity score, number of retrieval chunks, etc.
    
    Returns decile breakdown showing performance on easiest vs hardest queries.
    """
    if len(scores_per_query) != len(difficulty_labels):
        raise ValueError("Scores and labels must have same length")
    
    # Sort by difficulty (descending — easiest first)
    paired = sorted(
        zip(scores_per_query, difficulty_labels),
        key=lambda x: x[1],
        reverse=True,  # Easiest first
    )
    sorted_scores = [p[0] for p in paired]
    
    # Split into deciles
    n = len(sorted_scores)
    chunk_size = max(1, n // num_deciles)
    deciles = []
    
    for i in range(num_deciles):
        start = i * chunk_size
        end = start + chunk_size if i < num_deciles - 1 else n
        decile_scores = sorted_scores[start:end]
        decile_mean = statistics.mean(decile_scores) if decile_scores else 0.0
        deciles.append(decile_mean)
    
    overall = statistics.mean(scores_per_query)
    
    return DecileBreakdown(
        metric_name=metric_name,
        decile_scores=deciles,
        overall_average=overall,
    )


# === Usage: Discovering the hidden tail ===

# Simulate a RAG system's faithfulness scores
import random

random.seed(42)
queries = 1000

# Generate synthetic data: most queries are easy (high faithfulness),
# but some hard queries (low faithfulness)
difficulties = []
scores = []

for _ in range(queries):
    difficulty = random.random()  # 0 = hard, 1 = easy
    difficulties.append(difficulty)
    
    # Faithfulness correlates with difficulty
    base_score = 0.5 + difficulty * 0.4  # 0.5 to 0.9
    noise = random.gauss(0, 0.05)
    scores.append(min(1.0, max(0.0, base_score + noise)))

breakdown = compute_decile_breakdown(scores, difficulties, metric_name="faithfulness")
print(breakdown.summary())
```

**🤔 Try this yourself:** Run the code above. Look at the decile breakdown. Now imagine you're a product manager who only sees "average faithfulness: 0.84." What decision would you make? Now look at the worst decile: 0.62. Does that change your decision? What if you see a NEW pattern: over the last month, the overall average stayed the same but the tail gap INCREASED? What does that tell you?

---

## ✅ Good Output Examples

### What a Good Metric Suite Looks Like

```
System: Customer Support Classifier

=== Metric Suite Results ===

LEVEL 1 — COMPONENT METRICS:
  Retrieval Precision:      0.88 (n=500)
  Retrieval Recall:         0.82 (n=500)
  ⚠️ Precision-Recall gap: 0.06 (borderline acceptable)

LEVEL 2 — SYSTEM METRICS:
  Answer Accuracy:          0.91 (n=500)
  Balanced Accuracy:        0.87 (n=500)
  Faithfulness:             0.94 (n=500)
  Answer Relevance:         0.89 (n=500)
  Context Adherence:        0.92 (n=500)

LEVEL 3 — BUSINESS METRICS (from production analytics):
  Ticket Deflection Rate:   68% (+5% MoM) ✅
  Avg Resolution Time:      4.2min (-0.8min MoM) ✅
  User Satisfaction:        4.1/5 (+0.2 MoM) ✅

TAIL ANALYSIS:
  Worst Decile Accuracy:    0.72 (hardest queries)
  Best Decile Accuracy:     0.98 (easiest queries)
  Tail gap:                 0.26 ⚠️ — need to focus on hard queries

NO CONTRADICTIONS DETECTED:
  - Accuracy ≈ Balanced Accuracy (gap: 0.04) ✅
  - Faithfulness ≈ Relevance (gap: 0.05) ✅ 
  - Cost per Query: $0.008 at 91% success rate ✅

RECOMMENDATION: SHIP with monitoring on tail accuracy.
```

### What a Metric Suite Hiding Problems Looks Like

```
System: Content Moderation Classifier

=== Metric Suite Results ===

LEVEL 2 — SYSTEM METRICS:
  Overall Accuracy:         0.99 (n=10000) ← looks amazing
  Balanced Accuracy:        0.72 ← wait, this is much lower
  Precision (toxic):        0.95
  Recall (toxic):           0.45 ← this is the real number

⚠️ CONTRADICTION DETECTED:
  Overall accuracy (0.99) vs balanced accuracy (0.72) gap: 0.27.
  This indicates severe CLASS IMBALANCE. The 99% accuracy is
  achieved by predicting "safe" for most things, including
  55% of actual toxic content.

RECOMMENDATION: DO NOT SHIP. Improve toxic recall before deploying.
Target: recall ≥ 0.80 with precision ≥ 0.85.
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The Single Metric Dashboard

You put ONE metric on a dashboard and make decisions based on it. The team optimizes for that metric. The metric stops meaning what you think it means.

```
WRONG: 📊 Dashboard shows only "Accuracy: 99%"
  → Team optimizes for accuracy
  → Model predicts "safe" for everything (98.5% accuracy)
  → Real toxic comments missed
  → PR disaster

RIGHT: 📊 Dashboard shows "Accuracy: 99%, Balanced Accuracy: 72%, Toxic Recall: 45%"
  → Team sees the full picture
  → Cannot hide behind one number
  → Optimization targets ALL metrics
```

### ❌ Antipattern 2: The Vanilla Average

You report the average metric value without showing the distribution.

**Why it fails:** The average hides tails. A system can have 95% average accuracy while failing on the 5% of queries that matter most (security questions, edge cases).

**🔧 Fix:** Always report at least the 10th percentile alongside the average. Better: report decile breakdown.

### ❌ Antipattern 3: The Proxy That Drifted

You chose a proxy metric because the real metric was hard to measure. Over time, the proxy stops correlating with the real metric.

```
Month 1: Response time (proxy) ↔ User satisfaction (real): r = 0.85
Month 3: Response time (proxy) ↔ User satisfaction (real): r = 0.60
Month 6: Response time (proxy) ↔ User satisfaction (real): r = 0.30
```

**Why it happens:** The system optimized for response time (easy to measure). It got faster by cutting corners — shorter answers, fewer retrieval steps, less verification. Users got faster responses but LESS helpful ones.

**🔧 Fix:** Periodically re-validate your proxy metrics against ground truth. If the correlation drops below a threshold, update your proxy or add new metrics.

### ❌ Antipattern 4: The Metric That Encourages Shortcuts

```python
# WRONG: Optimizing for this will make your system worse
eval_prompt = "Score the answer 1-5 based on how well it answers the question."

# The system learns: longer, more confident answers score higher.
# It starts hallucinating confidently.
```

**🔧 Fix:** Before adding any metric, ask: "If someone optimized SOLELY for this metric, would the system get better or worse?" If the answer is "worse," your metric needs a counter-metric.

---

## 🧪 Drills & Challenges

### Drill 1: The Broken Metric Suite (25 min)

You're reviewing a teammate's metric suite for a sentiment analysis system:

```
Metrics Reported:
  - Accuracy: 93%
  - Precision (positive): 91%
  - Recall (positive): 87%
  - F1: 0.89
  - Average confidence: 0.88
  - Response time: 120ms
```

**Find at least 4 things WRONG with this metric suite.** Consider:
- What's missing? (What failure modes aren't captured?)
- What could be gamed?
- What does "average confidence" actually tell you?
- Is there a class imbalance concern?
- Are there paired metrics missing?
- What about the negative class?
- What about calibration?

**For each issue, write what you'd add or change.**

---

### Drill 2: Design the Metric Suite for a Multi-Agent System (30 min)

You're building the multi-agent research system from Phase 6's capstone project. It has:
- A supervisor agent that decomposes questions
- A search worker that finds information
- An analysis worker that synthesizes findings
- A citation worker that verifies sources

**Design a complete metric suite with at least 8 metrics** (4 pairs) that covers:
- Component-level (each worker)
- System-level (end-to-end)
- Business-level (what does good research look like?)

For each PAIR of metrics, explain:
1. What they measure
2. How they interact (when they agree, when they conflict)
3. What a contradiction between them would tell you
4. How someone could game each metric if measured in isolation

---

### Drill 3: The "Metric Lies" Detective (20 min)

For each scenario, identify WHICH metric is lying and WHY:

**Scenario A:** Your RAG system's faithfulness score is 0.95. But users complain the answers are "technically correct but miss the point." The answers stick to the context but the context doesn't fully cover the question.

**Scenario B:** Your agent's task success rate is 92%. Average cost per task dropped 30%. The CTO is thrilled. But the success rate improvement is concentrated on EASY queries — hard queries now succeed LESS often. The agent learned to cherry-pick.

**Scenario C:** Your classifier's F1 score improved from 0.82 to 0.88. Great! But you notice the improvement is entirely on ONE class. The other 4 classes stayed flat or got worse. The average F1 hides this because it's macro-averaged and the improved class was the largest.

**Scenario D:** Your LLM judge gives scores averaging 4.2/5. But you switched the judge model last month without telling anyone. The old judge was stricter (avg 3.8/5). You think the system improved, but the judge just got more lenient.

---

### Drill 4: Build the Tail-Aware Dashboard (35 min)

Take ANY metric from your previous Phase work (RAG faithfulness, agent success rate, accuracy, etc.) and build a dashboard that shows MORE than just the average.

Requirements:
1. Show the metric average (yes, still needed)
2. Show the metric DISTRIBUTION (decile breakdown)
3. Show the metric TREND over time (last N runs)
4. Show a SECOND metric that's the PAIR of the first (captures the opposite failure)
5. Flag when the tail gap increases beyond a threshold

You can use any visualization approach — matplotlib, ASCII art, or even text-based. The goal is to think about what a decision-maker needs to see.

**🤔 Extra:** Your dashboard shows all this data. Your PM looks at it and asks: "So is the system good or not?" What do you say? How do you summarize 5+ metrics into a GO/NO-GO decision without losing the nuance?

---

### Drill 5: The Metric Selection Interview (20 min)

You're interviewing for an AI engineering role. The interviewer asks:

> "You're joining a team that built a code generation assistant. It takes natural language descriptions and generates code. They're currently evaluating it with a single metric: 'Does the generated code compile?' — and it passes 94% of the time. The VP of Engineering wants to improve this to 99%. What do you tell them?"

**Your response should cover:**
1. Why "does it compile?" is an insufficient metric
2. What OTHER metrics are needed and why
3. What failure modes are hidden by the 94% compilation rate
4. What could go wrong if they optimize solely for compilation rate
5. A proposed metric SUITE (at least 4 metrics) that captures actual code quality

---

## 🚦 Gate Check

Before moving to File 04, verify you can:

1. **☐** Explain the 3-level metric hierarchy and give an example of each
2. **☐** Describe Goodhart's Law and how it applies to at least 3 AI metrics
3. **☐** Design a paired metric for any single metric (every metric needs a counter-metric)
4. **☐** Detect when a metric is being gamed (uncoupling, compression, edge degradation)
5. **☐** Explain why average metrics hide tail problems and how to surface them
6. **☐** Choose the right metrics for RAG vs. agents vs. classifiers vs. generators
7. **☐** Build a metric suite that surfaces contradictions, not hides them

### 🛑 Stop and Think

**Question 1 — The Optimization Trap**

Your team decides to optimize for "faithfulness" as the primary metric for your RAG system. After 2 months, faithfulness improved from 0.88 to 0.96. But user satisfaction dropped 10%.

Your analysis reveals: the system learned to produce highly faithful answers by being extremely conservative — only answering when the context was crystal clear, and responding "I don't have enough information to answer this" for anything ambiguous. The answers were faithful, but users hated them.

What metric should have been paired with faithfulness to catch this? How would the PAIR have prevented the optimization trap?

**Question 2 — The Business Connection**

You have 3 AI systems in production. Your dashboard shows 15 different metrics (5 per system). Your CEO asks: "Are our AI systems getting better or worse?"

You can't say "well, System A's faithfulness improved 2% but System B's precision dropped 1%..." — that's not helpful. What SINGLE number do you give the CEO? How do you construct a "System Health Score" that aggregates across all metrics and all systems without hiding problems?

**Question 3 — Cross-Phase Synthesis**

Think back across all 6 previous phases. For EACH phase, identify the ONE metric that was most critical:

- Phase 1 (LLM Gateway): What's the most important metric for a provider abstraction layer?
- Phase 2 (Prompt Engineering): What's the most important metric for a prompt optimization system?
- Phase 3 (Embeddings): What's the most important metric for vector search quality?
- Phase 4 (RAG): Most important metric?
- Phase 5 (Advanced RAG): Most important metric?
- Phase 6 (Agents): Most important metric?

Now: is there a SINGLE metric that spans ALL phases? Something that every AI system should measure, regardless of architecture? If not, why not? If yes, what is it and why does it work across all these different systems?

---

## 📚 Resources

- **"Goodhart's Law and Why Metrics Fail"** (Strathern, 1997) — The original essay on why measurement changes behavior. Short, powerful, essential reading for anyone building evaluation systems.
- **"Hidden Technical Debt in Machine Learning Systems"** (Google, 2015) — The classic paper. Section on "Measurement Debt" is directly applicable to LLM systems.
- **"RAGAS: Automated Evaluation of Retrieval Augmented Generation"** (2024) — The paper behind the RAGAS metrics. Explains why they chose specific metrics and how they validated them.
- **"Beyond Accuracy: Evaluating LLM Systems"** (Anthropic, 2025) — Practical guide on building metric suites. Their "metric pairing" framework influenced this module heavily.
- **"The Tail at Scale"** (Dean, 2013) — Originally about latency, but the principle applies perfectly to AI quality: the tail determines user perception, not the average.

**Next Module:**
→ **04-Evaluation-Datasets.md**: Creating, versioning, and maintaining evaluation datasets — including synthetic data generation and avoiding dataset contamination.
