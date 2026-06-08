# 07 — Advanced Evaluation: Beyond Basic Metrics

## 🎯 Purpose & Goals

Phase 4 gave you the fundamentals of RAG evaluation — RAGAS metrics (faithfulness, answer relevancy, context precision, context recall) and the basic evaluation loop.

That was enough for Phase 4. But as you've built multi-hop retrieval, agentic loops, multimodal pipelines, and graph traversals in Phase 5, the evaluation problem has gotten harder:

- **Multi-hop retrieval:** Standard metrics evaluate one retrieval pass. How do you evaluate a retrieval chain where each step depends on the previous?
- **Agentic decisions:** Did the agent retrieve when it should have? Did it stop at the right time? Standard metrics don't capture these decisions.
- **Multimodal answers:** How do you evaluate whether an image-based answer is correct? Text-only metrics can't see the image.
- **Contradiction handling:** Did the system correctly handle contradicting evidence? Standard metrics assume all retrieved context is equally valid.
- **Latency-quality tradeoffs:** A slow, expensive system that scores high on metrics may be WORSE than a fast, cheap system that scores slightly lower — but no standard metric captures this.

**By the end of this module, you will:**

1. Master **RAGAS advanced usage** — beyond the basic metrics to composite scoring, custom metrics, and regression testing
2. Implement **ARES (Automated RAG Evaluation System)** — a fine-tuned evaluation approach
3. Build a **component-level evaluation harness** that measures retriever, generator, and orchestrator independently
4. Design **synthetic evaluation datasets** that cover edge cases your real users encounter
5. Implement **LLM-as-judge with bias mitigation** — countering position bias, verbosity bias, and self-enhancement bias
6. Build a **human eval integration layer** for the cases where automated metrics fail
7. Create a **CI/CD evaluation pipeline** that catches regressions before deployment
8. Understand the **cost-latency-quality triangle** and how to optimize evaluation budgets

---

### ⏹ STOP. Answer these before proceeding.

**Question 1 — The Multi-Hop Evaluation Problem**

*Your Phase 5 multi-hop system answers: "What research areas does Project Alpha's leader focus on?"*

*The correct answer chain is: Project Alpha → led by → Dr. Chen → directs → AI Research → focuses on → NLP & Computer Vision*

*Your system returns: "Dr. Sarah Chen leads Project Alpha and focuses on NLP and computer vision."*

*This gets faithfulness ≈ 1.0 (all statements supported) and answer relevancy ≈ 1.0 (directly answers the question).*

*But did the system actually TRAVERSE the full chain? Or did it hallucinate the connection between "Project Alpha" and "NLP" because both appeared in nearby chunks?*

*Standard metrics can't distinguish between "correct reasoning path" and "lucky guess from co-occurring terms." How would you design a metric that verifies the REASONING PATH, not just the final answer?*

*What if the correct answer requires 5 hops, and the system only did 3 but still got the right answer because the remaining 2 hops were deducible from context? Is that a success (right answer) or a failure (incomplete reasoning)?*

**Question 2 — The Cost-Quality-Latency Triangle**

*You're evaluating three RAG configurations:*

| Config | Accuracy | Latency | Cost/Query |
|--------|----------|---------|-----------|
| A | 89% | 800ms | $0.008 |
| B | 93% | 3.2s | $0.045 |
| C | 95% | 7.1s | $0.120 |

*Config A (cheap, fast) gets 89% accuracy. Config C (expensive, slow) gets 95%.*

*Is the 6% improvement worth 9x the cost and 9x the latency?*

*For a customer support bot handling 10,000 queries/day:* 
- *Config A: $80/day, 89% auto-resolve rate → 1,100 escalations to human agents*
- *Config C: $1,200/day, 95% auto-resolve → 500 escalations*

*If each human escalation costs $3.50 to handle:*
- *Config A total: $80 + (1,100 × $3.50) = $3,930/day*
- *Config C total: $1,200 + (500 × $3.50) = $2,950/day*

*Config C saves $980/day DESPITE costing 15x more per query!*

*How do you build an evaluation system that accounts for DOWNSTREAM costs — not just accuracy metrics but the business impact of errors? What metrics would you track beyond accuracy?*

**Question 3 — The False Confidence Problem**

*You run 1,000 evaluation queries through your RAG system. Your LLM-as-judge says:*
- *Faithfulness: 0.94*
- *Answer relevancy: 0.91*
- *Context recall: 0.88*

*You're confident. You deploy.*

*In production, users complain about WRONG answers. You investigate and find that:*

1. *Your evaluation dataset has a DISTRIBUTION SHIFT — the test queries are about "policies" but production queries are about "troubleshooting." The evaluation was testing the wrong thing.*
2. *Your LLM-as-judge has a CONFIRMATION BIAS — it rates answers that use similar vocabulary to the context as "faithful" even when the answer is technically wrong. It's rewarding fluent-sounding wrong answers.*
3. *Your evaluation doesn't test the HARD cases — the test set was created from queries your system already handles well. You're measuring recall on easy examples.*

*How do you detect distribution shift between your eval set and production traffic? How do you design an eval set that covers the HARD cases, not just the common ones?*

*What metrics would have caught the problem BEFORE deployment?*

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The Evaluation Maturity Model | 10 min |
| RAGAS Advanced — Beyond Basics | 30 min |
| ARES: Fine-Tuned Evaluation | 25 min |
| Component-Level Evaluation | 30 min |
| Synthetic Evaluation Dataset Generation | 30 min |
| LLM-as-Judge: Bias Analysis & Mitigation | 30 min |
| Human Evaluation Integration | 25 min |
| CI/CD Evaluation Pipeline | 25 min |
| Production Eval System | 30 min |
| Code: Advanced RAGAS Harness | 30 min |
| Code: Component-Level Eval | 30 min |
| Code: Eval Dataset Generator | 30 min |
| Code: Production Eval Dashboard | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~7.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Evaluation Maturity Model

Evaluation is a practice that MATURES over time. Most teams start at Level 1 and never advance. The best teams reach Level 4:

```
Level 1: Ad-hoc (most common)
  "Looks good to me" manual testing
  No eval datasets, no metrics, no regression testing
  → Every deployment is a gamble

Level 2: Automated (you're here after Phase 4)
  RAGAS metrics, fixed eval dataset, automated scoring
  Can catch regressions, measure improvement
  → But: dataset has distribution shift, metrics have biases

Level 3: Systematic (Phase 5 target)
  Component-level evaluation (retriever vs. generator isolated)
  Synthetic dataset generation (covers edge cases)
  LLM-as-judge with bias mitigation
  → Can diagnose exactly which component fails and why

Level 4: Business-integrated (target for Phase 7)
  Evaluation tied to business metrics (escalation rate, user satisfaction)
  Cost-quality-latency optimization
  Continuous monitoring (not just pre-deployment)
  Production traffic sampled into eval set continuously
```

**Where you are now:** After Phase 4, you're at Level 2. After this module, you'll be at Level 3. Phase 7 (Production Evals & Observability) will take you to Level 4.

### 2. RAGAS Advanced — The Unused Power

Phase 4 introduced RAGAS basics. But RAGAS has advanced capabilities most users don't touch:

**Aspect Critique:** Evaluate specific ASPECTS of the answer, not just overall quality:

```python
from ragas.metrics import AspectCritic
from ragas.llms import LangchainLLM

# Instead of a single "faithfulness" score, evaluate multiple aspects:
aspects = [
    AspectCritic(name="correctness", definition="Is the answer factually correct?"),
    AspectCritic(name="completeness", definition="Does the answer cover all aspects of the question?"),
    AspectCritic(name="conciseness", definition="Is the answer free of unnecessary information?"),
    AspectCritic(name="citation_quality", definition="Are claims properly attributed to sources?"),
    AspectCritic(name="tone", definition="Is the tone appropriate for the audience?"),
]
```

**Composite Scoring:** Combine multiple metrics with weighted scoring:

```python
# NOT just averaging — weighted for YOUR priorities
composite_score = (
    0.30 * faithfulness +
    0.25 * answer_relevancy +
    0.20 * context_recall +
    0.15 * context_precision +
    0.10 * conciseness
)
```

**Custom Metrics:** You can define your OWN RAGAS-compatible metrics:

```python
from ragas.metrics.base import Metric
from ragas.dataset_schema import SingleTurnSample

class ContradictionDetectionMetric(Metric):
    """Measure whether the system correctly detects contradictions in sources."""
    
    name = "contradiction_detection"
    
    async def score(self, sample: SingleTurnSample) -> float:
        # Check if the answer acknowledges contradictions when they exist
        # 1.0 = correctly identified contradiction
        # 0.5 = partially acknowledged
        # 0.0 = ignored contradiction or hallucinated one
        ...
```

### 3. ARES: Automated RAG Evaluation System

ARES (Saad-Falcon et al., 2024) takes a different approach: instead of using a general LLM as judge, ARES trains SPECIALIZED classifiers for evaluation.

**The ARES pipeline:**

```
Phase 1: LLM-Powered Synthetic Data Generation
  ┌──────────┐    ┌──────────────┐    ┌──────────────┐
  │  Source   │───→│  Generate    │───→│  Positive +  │
  │ Documents │    │  Q/A Pairs   │    │  Negative    │
  └──────────┘    └──────────────┘    │  Examples    │
                                       └──────────────┘

Phase 2: Fine-Tune Lightweight Evaluators
  ┌──────────────┐    ┌──────────────┐
  │  DeBERTa /   │───→│  Context     │
  │  RoBERTa     │    │  Relevance   │
  └──────────────┘    │  Classifier  │
                      ├──────────────┤
                      │  Answer      │
                      │  Faithfulness│
                      │  Classifier  │
                      ├──────────────┤
                      │  Answer      │
                      │  Utility     │
                      │  Classifier  │
                      └──────────────┘

Phase 3: Few-Shot Domain Adaptation
  ┌──────────────┐    ┌──────────────┐
  │  20 human    │───→│  Calibrate   │
  │  labels      │    │  evaluators  │
  └──────────────┘    │  to domain   │
                      └──────────────┘
```

**Why ARES matters:**

| Aspect | RAGAS (LLM-as-judge) | ARES (Fine-tuned) |
|--------|---------------------|-------------------|
| Cost per evaluation | ~$0.01-0.05 (LLM call) | ~$0.0001 (classifier forward pass) |
| Latency per evaluation | 1-5 seconds | 10-100ms |
| Domain adaptation | Limited (prompt engineering) | Fine-tune on domain data |
| Transparency | Black box ("the LLM said so") | Interpretable features |
| Consistency | Variable (LLM randomness) | Deterministic |

**The tradeoff:** ARES requires labeled training data and fine-tuning infrastructure. RAGAS works zero-shot. The sweet spot: use RAGAS for development iteration (fast setup), ARES for production evaluation (cheap, fast, consistent).

### 4. Component-Level Evaluation

The most important evaluation insight in this module: **evaluate each component independently, not just the end-to-end pipeline.**

A low faithfulness score could mean:
- The retriever returned irrelevant chunks (bad retrieval)
- The LLM ignored relevant chunks (bad generation)
- The context window was too small to fit the answer (bad configuration)
- The instruction prompt was poorly designed (bad orchestration)

Without component-level evaluation, you can't tell WHICH is the problem.

```
Component Evaluation Matrix:

┌─────────────────────────────────────────────────────┐
│                  │ RETRIEVAL │ GENERATION │ ORCHESTRATION│
├──────────────────┼───────────┼────────────┼──────────────┤
│ Faithfulness     │    -      │     ✓      │      -       │
│ Answer Relevancy │    -      │     ✓      │      ✓       │
│ Context Recall   │    ✓      │     -      │      -       │
│ Context Precision│    ✓      │     -      │      -       │
│ Retrieval MRR    │    ✓      │     -      │      -       │
│ Agent Decisions  │    -      │     -      │      ✓       │
│ Cost Efficiency  │    ✓      │     ✓      │      ✓       │
│ Latency          │    ✓      │     ✓      │      ✓       │
└─────────────────────────────────────────────────────┘
```

**Isolating the retriever:** Replace the LLM generator with a PERFECT generator (use gold-standard answers). Any score drop is the retriever's fault.

**Isolating the generator:** Replace the retriever with a PERFECT retriever (inject ground-truth context directly). Any score drop is the generator's fault.

### 5. LLM-as-Judge: The Five Biases

LLM-as-judge evaluation is powerful but has FIVE known biases that can invalidate your results:

| Bias | Description | Impact | Mitigation |
|------|-------------|--------|------------|
| **Position bias** | Judge prefers answers that appear first in the comparison | Overrates config A when comparing A vs B | Swap positions, average scores; use multi-turn evaluation |
| **Verbosity bias** | Judge prefers LONGER answers regardless of quality | Rewards verbose, repetitive answers | Normalize for length; evaluate conciseness separately |
| **Self-enhancement bias** | Judge prefers answers that match its OWN style/language | Biased toward LLM-generated answers | Use different judge model than generator; fine-tune out bias |
| **Contrast bias** | Judge exaggerates differences when comparing | Small quality difference → large score difference | Use absolute scoring (not pairwise) for fine-grained eval |
| **Syndrome of cognitive biases** | LLMs exhibit anchoring, framing, and order effects | Inconsistent evaluations | Use structured rubrics; randomize presentation; average multiple judges |

**Production mitigation strategies:**

1. **Multi-model judging:** Use 3 different LLMs as judges and take majority vote
2. **Rubric-based evaluation:** Give the judge a DETAILED scoring rubric (not just "rate this answer")
3. **Calibration set:** Include 10 known examples to calibrate the judge's scoring scale
4. **Stratified evaluation:** Group results by query type and compare within groups, not across groups

---

## 💻 Code Examples

### 1. Advanced RAGAS Evaluation Harness

```python
"""
advanced_ragas.py — Advanced RAGAS evaluation with aspect critique,
composite scoring, custom metrics, and regression testing.

Extends Phase 4's evaluation with:
- AspectCritic for multi-dimensional scoring
- Weighted composite metrics
- Per-component evaluation
- Regression test suite
"""

from typing import List, Optional, Dict, Any, Callable
from dataclasses import dataclass, field
import json
import asyncio
import time
import logging

from openai import AsyncOpenAI
import numpy as np

logger = logging.getLogger(__name__)


# ─── Data Models ────────────────────────────────────────────────────────────

@dataclass
class EvalSample:
    """A single evaluation sample."""
    query: str
    ground_truth: str  # The ideal answer
    retrieved_contexts: List[str]  # What the retriever returned
    response: str  # What the system actually generated
    expected_contexts: List[str]  # What the retriever SHOULD have returned
    metadata: Dict[str, Any] = field(default_factory=dict)

    # For multi-hop evaluation
    expected_reasoning_chain: List[str] = field(default_factory=list)
    actual_reasoning_chain: List[str] = field(default_factory=list)


@dataclass
class EvalResult:
    """Results of a single evaluation."""
    faithfulness: float = 0.0
    answer_relevancy: float = 0.0
    context_precision: float = 0.0
    context_recall: float = 0.0
    aspect_scores: Dict[str, float] = field(default_factory=dict)
    composite_score: float = 0.0
    reasoning_accuracy: float = 0.0  # For multi-hop
    latency_ms: float = 0.0
    cost: float = 0.0


@dataclass
class RegressionTest:
    """A regression test case."""
    query: str
    expected_min_scores: Dict[str, float]  # metric → minimum acceptable score
    ground_truth: str = ""
    expected_contexts: List[str] = field(default_factory=list)


@dataclass
class RegressionReport:
    """Report from a regression test run."""
    passed: int = 0
    failed: int = 0
    failures: List[Dict] = field(default_factory=list)
    score_summary: Dict[str, float] = field(default_factory=dict)


# ─── Aspect Critic ──────────────────────────────────────────────────────────

class AspectCritic:
    """
    Evaluate a specific aspect of an answer.

    Unlike a single "faithfulness" score, aspect critique gives you
    dimensional analysis: WHERE does the answer fail?
    """

    ASPECT_DEFINITIONS = {
        "correctness": "Is every factual claim in the answer supported by the provided context? The answer should not contain claims that contradict or are absent from the context.",
        "completeness": "Does the answer address ALL parts of the user's question? Consider implicit questions as well as explicit ones.",
        "conciseness": "Is the answer free of irrelevant information, repetition, and unnecessary elaboration? It should be as short as possible while being complete.",
        "citation_quality": "Are claims properly attributed to specific sources in the context? The answer should indicate WHICH source supports each claim.",
        "coherence": "Is the answer logically structured and easy to follow? Claims should flow logically and transitions should be smooth.",
        "helpfulness": "Does the answer actually help the user solve their problem? Beyond correctness, does it provide actionable information?",
        "tone_appropriateness": "Is the tone appropriate for the audience and use case? Technical queries should get technical answers; casual queries should get accessible answers.",
    }

    SCORE_RUBRIC = {
        5: "Excellent — perfectly addresses this aspect",
        4: "Good — minor issues that don't affect usefulness",
        3: "Adequate — acceptable but has room for improvement",
        2: "Poor — significant issues with this aspect",
        1: "Very poor — completely fails on this aspect",
    }

    def __init__(
        self,
        aspect_name: str,
        llm_client: AsyncOpenAI,
        model: str = "gpt-4o-mini",
    ):
        assert aspect_name in self.ASPECT_DEFINITIONS, \
            f"Unknown aspect: {aspect_name}. Available: {list(self.ASPECT_DEFINITIONS.keys())}"
        self.aspect_name = aspect_name
        self.definition = self.ASPECT_DEFINITIONS[aspect_name]
        self.llm_client = llm_client
        self.model = model

    async def score(
        self,
        query: str,
        response: str,
        contexts: List[str],
    ) -> float:
        """Score the response on this aspect (1-5 scale, normalized to 0-1)."""
        prompt = f"""
You are evaluating a RAG system's response on the aspect of: {self.aspect_name}

Definition: {self.definition}

Scoring rubric:
{json.dumps(self.SCORE_RUBRIC, indent=2)}

User query: {query}

Retrieved context: 
{chr(10).join(f'[{i+1}] {c[:1000]}' for i, c in enumerate(contexts[:3]))}

System response: {response}

Score this response on "{self.aspect_name}" (1-5).
Return JSON: {{"score": int, "reasoning": "why this score"}}
"""
        import json
        resp = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": f"You evaluate RAG system responses on {self.aspect_name}."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
            max_tokens=500,
        )

        try:
            data = json.loads(resp.choices[0].message.content)
            score = data.get("score", 3)
            return max(0.0, min(1.0, (score - 1) / 4))  # Normalize 1-5 → 0-1
        except Exception:
            return 0.5  # Default: middle score


# ─── Composite Scorer ───────────────────────────────────────────────────────

class CompositeScorer:
    """
    Combine multiple metrics into a weighted composite score.

    Different use cases need different weights. A customer support bot
    weights faithfulness higher than conciseness. A research assistant
    weights completeness higher.
    """

    # Default weights for different use cases
    USE_CASE_WEIGHTS = {
        "customer_support": {
            "faithfulness": 0.35,
            "completeness": 0.20,
            "helpfulness": 0.20,
            "conciseness": 0.15,
            "tone_appropriateness": 0.10,
        },
        "research_assistant": {
            "faithfulness": 0.25,
            "completeness": 0.30,
            "citation_quality": 0.25,
            "coherence": 0.10,
            "conciseness": 0.10,
        },
        "code_generation": {
            "correctness": 0.40,
            "completeness": 0.25,
            "conciseness": 0.20,
            "citation_quality": 0.15,
        },
    }

    def __init__(
        self,
        weights: Optional[Dict[str, float]] = None,
        use_case: str = "customer_support",
    ):
        if weights:
            self.weights = weights
        else:
            self.weights = self.USE_CASE_WEIGHTS.get(
                use_case, self.USE_CASE_WEIGHTS["customer_support"]
            )

    def compute(self, scores: Dict[str, float]) -> float:
        """Compute weighted composite score."""
        total = 0.0
        weight_sum = 0.0

        for metric, weight in self.weights.items():
            if metric in scores:
                total += scores[metric] * weight
                weight_sum += weight

        if weight_sum == 0:
            return 0.0

        return total / weight_sum

    def diagnose(self, scores: Dict[str, float]) -> List[str]:
        """
        Diagnose which aspects need improvement.

        Returns prioritized list of weaknesses.
        """
        weaknesses = []
        for metric, weight in sorted(self.weights.items(), key=lambda x: x[1], reverse=True):
            if metric in scores and scores[metric] < 0.7:
                weaknesses.append(
                    f"{metric} ({scores[metric]:.2f}): "
                    f"weight {weight:.0%}, below 0.7 threshold"
                )
        return weaknesses


# ─── Evaluation Harness ─────────────────────────────────────────────────────

class AdvancedRAGEvaluator:
    """
    Advanced RAG evaluation harness with:
    - Aspect critique for multi-dimensional evaluation
    - Weighted composite scoring
    - Per-component isolation
    - Regression testing
    - Cost and latency tracking
    """

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        model: str = "gpt-4o-mini",
        aspects: Optional[List[str]] = None,
        use_case: str = "customer_support",
    ):
        self.llm_client = llm_client
        self.model = model
        self.aspect_names = aspects or list(AspectCritic.ASPECT_DEFINITIONS.keys())
        self.scorer = CompositeScorer(use_case=use_case)
        self._total_eval_cost = 0.0
        self._cached_results: Dict[str, EvalResult] = {}

    async def evaluate(
        self,
        sample: EvalSample,
        track_latency: bool = True,
    ) -> EvalResult:
        """
        Run full evaluation on a single sample.

        Returns scores for all aspects, composite, and reasoning accuracy.
        """
        start = time.time()
        result = EvalResult()

        # Step 1: Context metrics (retrieval quality)
        if sample.expected_contexts:
            result.context_precision = self._compute_context_precision(
                sample.retrieved_contexts, sample.expected_contexts
            )
            result.context_recall = self._compute_context_recall(
                sample.retrieved_contexts, sample.expected_contexts
            )

        # Step 2: Aspect critique (generation quality)
        aspect_tasks = []
        for aspect_name in self.aspect_names:
            critic = AspectCritic(aspect_name, self.llm_client, self.model)
            aspect_tasks.append(critic.score(
                sample.query, sample.response, sample.retrieved_contexts
            ))

        aspect_scores = await asyncio.gather(*aspect_tasks)
        for name, score in zip(self.aspect_names, aspect_scores):
            result.aspect_scores[name] = score

        # Map aspect scores to standard metrics
        result.faithfulness = result.aspect_scores.get("correctness", 0.0)
        result.answer_relevancy = result.aspect_scores.get("completeness", 0.0)

        # Step 3: Reasoning chain accuracy (for multi-hop)
        if sample.expected_reasoning_chain and sample.actual_reasoning_chain:
            result.reasoning_accuracy = self._compute_reasoning_accuracy(
                sample.expected_reasoning_chain,
                sample.actual_reasoning_chain,
            )

        # Step 4: Composite score
        result.composite_score = self.scorer.compute(result.aspect_scores)

        # Step 5: Latency tracking
        if track_latency:
            result.latency_ms = (time.time() - start) * 1000

        # Estimate cost
        result.cost = self._estimate_eval_cost(sample)

        self._cached_results[sample.query] = result
        return result

    def _compute_context_precision(
        self,
        retrieved: List[str],
        expected: List[str],
    ) -> float:
        """What fraction of retrieved chunks were actually relevant?"""
        if not retrieved:
            return 0.0
        # In production: use embedding similarity or LLM to judge relevance
        # Simplified: check text overlap
        relevant = sum(
            1 for r in retrieved
            if any(e.lower() in r.lower() for e in expected)
        )
        return relevant / len(retrieved)

    def _compute_context_recall(
        self,
        retrieved: List[str],
        expected: List[str],
    ) -> float:
        """What fraction of expected chunks were actually retrieved?"""
        if not expected:
            return 1.0
        found = sum(
            1 for e in expected
            if any(e.lower() in r.lower() for r in retrieved)
        )
        return found / len(expected)

    def _compute_reasoning_accuracy(
        self,
        expected_chain: List[str],
        actual_chain: List[str],
    ) -> float:
        """
        How much of the expected reasoning chain was present in the actual chain?

        This is CRITICAL for multi-hop evaluation — it measures whether the
        system actually TRAVERSED the correct reasoning path.
        """
        if not expected_chain:
            return 1.0

        # Simplified: count how many expected steps appear in actual chain
        expected_normalized = [e.lower().strip() for e in expected_chain]
        actual_combined = " ".join(a.lower().strip() for a in actual_chain)

        found = sum(
            1 for step in expected_normalized
            if step in actual_combined
        )
        return found / len(expected_chain) if expected_chain else 1.0

    def _estimate_eval_cost(self, sample: EvalSample) -> float:
        """Estimate the cost of evaluating this sample."""
        # Rough estimation: each LLM call ~$0.0005 with gpt-4o-mini
        num_aspects = len(self.aspect_names)
        return num_aspects * 0.0005

    async def evaluate_batch(
        self,
        samples: List[EvalSample],
        batch_size: int = 10,
    ) -> List[EvalResult]:
        """Evaluate a batch of samples."""
        results = []
        for i in range(0, len(samples), batch_size):
            batch = samples[i:i + batch_size]
            batch_results = await asyncio.gather(
                *[self.evaluate(s) for s in batch]
            )
            results.extend(batch_results)
            logger.info(f"Evaluated batch {i // batch_size + 1}: "
                        f"{len(batch)} samples, "
                        f"avg composite: {np.mean([r.composite_score for r in batch_results]):.3f}")
        return results

    def summary(self, results: List[EvalResult]) -> Dict[str, Any]:
        """Generate a summary report from batch evaluation results."""
        if not results:
            return {"error": "No results"}

        metrics = {
            "faithfulness": [r.faithfulness for r in results],
            "answer_relevancy": [r.answer_relevancy for r in results],
            "context_precision": [r.context_precision for r in results],
            "context_recall": [r.context_recall for r in results],
            "composite": [r.composite_score for r in results],
            "reasoning_accuracy": [r.reasoning_accuracy for r in results],
        }

        # Add aspect-specific summaries
        if results[0].aspect_scores:
            for aspect in results[0].aspect_scores:
                metrics[f"aspect_{aspect}"] = [
                    r.aspect_scores.get(aspect, 0) for r in results
                ]

        summary = {}
        for name, values in metrics.items():
            if values:
                summary[name] = {
                    "mean": float(np.mean(values)),
                    "median": float(np.median(values)),
                    "min": float(np.min(values)),
                    "max": float(np.max(values)),
                    "std": float(np.std(values)),
                    "p25": float(np.percentile(values, 25)),
                    "p75": float(np.percentile(values, 75)),
                    "p95": float(np.percentile(values, 95)),
                }

        summary["total_eval_cost"] = sum(r.cost for r in results)
        summary["avg_latency_ms"] = float(np.mean([r.latency_ms for r in results]))
        summary["sample_count"] = len(results)

        # Weakness diagnosis
        composite_scores = [r.composite_score for r in results]
        avg_composite = np.mean(composite_scores)
        if avg_composite < 0.7 and results:
            diagnosis = self.scorer.diagnose(results[0].aspect_scores)
            summary["diagnosis"] = diagnosis

        return summary


# ─── Regression Test Runner ─────────────────────────────────────────────────

class RegressionTestRunner:
    """
    Run regression tests against the RAG system.

    Maintains a suite of tests that MUST pass before deployment.
    Alerts on score drops, not just absolute failures.
    """

    def __init__(
        self,
        evaluator: AdvancedRAGEvaluator,
        score_history: Optional[Dict[str, List[float]]] = None,
    ):
        self.evaluator = evaluator
        self.score_history = score_history or {}

    async def run_tests(
        self,
        tests: List[RegressionTest],
        system_fn: Callable[[str], Awaitable[Dict]],  # (query) → {response, contexts}
    ) -> RegressionReport:
        """
        Run regression tests and check against minimum scores.

        Also checks for score DROPS compared to historical runs.
        """
        passed = 0
        failed = 0
        failures = []

        for test in tests:
            # Get system output
            system_output = await system_fn(test.query)
            response = system_output.get("response", "")
            contexts = system_output.get("contexts", [])

            # Evaluate
            sample = EvalSample(
                query=test.query,
                ground_truth=test.ground_truth,
                retrieved_contexts=contexts,
                response=response,
                expected_contexts=test.expected_contexts,
            )
            result = await self.evaluator.evaluate(sample)

            # Check against minimum scores
            test_failures = {}
            for metric, min_score in test.expected_min_scores.items():
                actual = getattr(result, metric, None)
                if actual is None:
                    actual = result.aspect_scores.get(metric, 0.0)
                if actual < min_score:
                    test_failures[metric] = {
                        "expected_min": min_score,
                        "actual": actual,
                    }

                # Track history for drop detection
                history_key = f"{test.query}::{metric}"
                if history_key not in self.score_history:
                    self.score_history[history_key] = []
                self.score_history[history_key].append(actual)

                # Check for significant drop (>15%) from historical average
                hist = self.score_history[history_key]
                if len(hist) >= 3:
                    hist_avg = np.mean(hist[:-1])  # Average of previous runs
                    if actual < hist_avg * 0.85:  # >15% drop
                        test_failures[f"{metric}_drop"] = {
                            "historical_avg": hist_avg,
                            "current": actual,
                            "drop_pct": (hist_avg - actual) / hist_avg * 100,
                        }

            if test_failures:
                failed += 1
                failures.append({
                    "query": test.query,
                    "failures": test_failures,
                    "composite": result.composite_score,
                })
            else:
                passed += 1

        return RegressionReport(
            passed=passed,
            failed=failed,
            failures=failures,
            score_summary={
                m: s
                for m in ["faithfulness", "answer_relevancy", "composite"]
                for s in [{"mean": "see full report"}]
            },
        )

    def get_regression_alert(self, report: RegressionReport) -> Optional[str]:
        """Generate a human-readable alert if regression is detected."""
        if report.failed == 0:
            return None

        alert_parts = [
            f"⚠️ REGRESSION DETECTED: {report.failed}/{report.passed + report.failed} tests failed"
        ]

        for failure in report.failures[:5]:
            query = failure["query"][:80]
            details = "; ".join(
                f"{m}: expected {v.get('expected_min', '?')}, got {v.get('actual', '?'):.3f}"
                for m, v in failure["failures"].items()
            )
            alert_parts.append(f"  • '{query}': {details}")

        return "\n".join(alert_parts)
```

---

### 2. Component-Level Evaluation

```python
"""
component_eval.py — Evaluate each RAG component independently.

The key insight: isolate components to find the ROOT CAUSE of failures.

- Replace retriever with perfect retriever → measure generator quality
- Replace generator with perfect generator → measure retriever quality
- Compare to identify interaction failures
"""

from typing import List, Optional, Dict, Any, Callable, Tuple
from dataclasses import dataclass, field
import asyncio
import time


@dataclass
class ComponentEvalResult:
    """Results from component-level evaluation."""
    # Component scores
    retriever_recall: float = 0.0
    retriever_precision: float = 0.0
    retriever_mrr: float = 0.0
    generator_faithfulness: float = 0.0
    generator_completeness: float = 0.0
    orchestrator_efficiency: float = 0.0

    # Interaction failures
    context_mismatch: float = 0.0  # Retriever found it, generator didn't use it
    hallucination_rate: float = 0.0  # Generator made up info not in context

    # End-to-end
    end_to_end_score: float = 0.0

    # Recommendations
    recommendations: List[str] = field(default_factory=list)

    def summary(self) -> str:
        """Generate a human-readable summary."""
        lines = [
            "COMPONENT EVALUATION REPORT",
            "=" * 40,
            f"Retriever:  P={self.retriever_precision:.3f} R={self.retriever_recall:.3f} MRR={self.retriever_mrr:.3f}",
            f"Generator:  F={self.generator_faithfulness:.3f} C={self.generator_completeness:.3f}",
            f"Interaction: Context mismatch={self.context_mismatch:.1%}",
            f"            Hallucination rate={self.hallucination_rate:.1%}",
            f"E2E:        {self.end_to_end_score:.3f}",
        ]
        if self.recommendations:
            lines.append("\nRecommendations:")
            for rec in self.recommendations:
                lines.append(f"  • {rec}")
        return "\n".join(lines)


class ComponentEvaluator:
    """
    Evaluate RAG components in isolation.

    Methodology:
    1. PERFECT RETRIEVER: Inject ground-truth context directly into the LLM
       → Any remaining errors are GENERATOR errors

    2. PERFECT GENERATOR: Use ground-truth answer as the "generated" output
       → Any remaining errors are RETRIEVER errors

    3. ACTUAL SYSTEM: Run normally and compare to component baselines
       → Interaction errors are ORCHESTRATOR errors
    """

    def __init__(
        self,
        retriever_fn: Callable,  # (query) → List[chunks]
        generator_fn: Callable,  # (query, contexts) → response
        eval_judge_fn: Callable,  # (query, response, expected) → score
    ):
        self.retriever = retriever_fn
        self.generator = generator_fn
        self.eval_judge = eval_judge_fn

    async def evaluate(
        self,
        test_cases: List[EvalSample],
    ) -> ComponentEvalResult:
        """
        Run component-level evaluation on test cases.

        Each test case needs: query, ground_truth, expected_contexts.
        """
        result = ComponentEvalResult()
        retriever_recalls = []
        retriever_precisions = []
        retriever_mrrs = []
        generator_faithfulness = []
        generator_completeness = []
        context_mismatches = 0
        hallucinations = 0

        for case in test_cases:
            # ── RETRIEVER EVALUATION ──
            # Measure: did the retriever find the right contexts?

            retrieved = await self.retriever(case.query)

            # Recall
            if case.expected_contexts:
                found_relevant = sum(
                    1 for exp in case.expected_contexts
                    if any(exp.lower() in r.get("text", str(r)).lower() for r in retrieved)
                )
                recall = found_relevant / len(case.expected_contexts)
                retriever_recalls.append(recall)
            else:
                retriever_recalls.append(0.0)

            # Precision
            if retrieved:
                relevant_count = sum(
                    1 for r in retrieved
                    if any(e.lower() in r.get("text", str(r)).lower() for e in case.expected_contexts)
                )
                precision = relevant_count / len(retrieved)
                retriever_precisions.append(precision)
            else:
                retriever_precisions.append(0.0)

            # MRR (Mean Reciprocal Rank)
            if case.expected_contexts and retrieved:
                for rank, chunk in enumerate(retrieved, start=1):
                    chunk_text = chunk.get("text", str(chunk))
                    if any(e.lower() in chunk_text.lower() for e in case.expected_contexts):
                        retriever_mrrs.append(1.0 / rank)
                        break
                else:
                    retriever_mrrs.append(0.0)

            # ── GENERATOR EVALUATION (with perfect retrieval) ──
            # Inject ground-truth context as if the retriever found it perfectly

            perfect_contexts = case.expected_contexts
            generated_with_perfect_context = await self.generator(
                case.query, perfect_contexts
            )

            faithfulness = await self.eval_judge(
                case.query, generated_with_perfect_context, case.ground_truth
            )
            generator_faithfulness.append(faithfulness)

            # ── INTERACTION EVALUATION ──
            # Did the generator USE the context that was retrieved?

            if retrieved and case.expected_contexts:
                # Check if context was retrieved but not used
                actual_response = await self.generator(case.query, retrieved)
                for exp in case.expected_contexts:
                    if any(exp.lower() in r.get("text", str(r)).lower() for r in retrieved):
                        # Context was retrieved — was it used?
                        if exp.lower() not in actual_response.lower()[:2000][-1000:]:
                            context_mismatches += 1
                            break

                # Check for hallucination
                response_lower = actual_response.lower()
                all_context_text = " ".join(
                    r.get("text", str(r)).lower() for r in retrieved
                )
                # Simple check: are there claims in the response not in any context?
                # (In production, use an LLM judge for this)
                resp_sentences = [s.strip() for s in actual_response.split(".") if len(s.strip()) > 20]
                for sentence in resp_sentences[:5]:
                    sentence_lower = sentence.lower()
                    # Check if key terms in sentence exist in context
                    import re
                    words = set(re.findall(r'\b[a-z]{4,}\b', sentence_lower))
                    if words:
                        overlap = sum(1 for w in words if w in all_context_text)
                        if overlap / len(words) < 0.3:  # <30% word overlap
                            hallucinations += 1
                            break

        # ── COMPUTE AGGREGATES ──
        if retriever_recalls:
            result.retriever_recall = sum(retriever_recalls) / len(retriever_recalls)
        if retriever_precisions:
            result.retriever_precision = sum(retriever_precisions) / len(retriever_precisions)
        if retriever_mrrs:
            result.retriever_mrr = sum(retriever_mrrs) / len(retriever_mrrs)
        if generator_faithfulness:
            result.generator_faithfulness = sum(generator_faithfulness) / len(generator_faithfulness)

        total_cases = len(test_cases)
        result.context_mismatch = context_mismatches / total_cases if total_cases else 0
        result.hallucination_rate = hallucinations / total_cases if total_cases else 0

        # End-to-end score (weighted combination)
        result.end_to_end_score = (
            0.30 * result.retriever_recall +
            0.25 * result.retriever_precision +
            0.30 * result.generator_faithfulness +
            0.15 * (1.0 - result.hallucination_rate)
        )

        # ── GENERATE RECOMMENDATIONS ──
        if result.retriever_recall < 0.7:
            result.recommendations.append(
                f"Retriever recall is {result.retriever_recall:.2f}. "
                "Consider: better embedding model, query transformation (HyDE), "
                "increase chunk overlap, or add keyword search fallback."
            )
        if result.retriever_precision < 0.7:
            result.recommendations.append(
                f"Retriever precision is {result.retriever_precision:.2f}. "
                "Consider: add reranking, metadata filtering, or reduce chunk size."
            )
        if result.generator_faithfulness < 0.7 and result.retriever_recall >= 0.8:
            result.recommendations.append(
                "Generator faithfulness is low despite good retrieval. "
                "The generator is ignoring the retrieved context. "
                "Consider: stronger system prompt, in-context examples, "
                "or a different generation model."
            )
        if result.context_mismatch > 0.2:
            result.recommendations.append(
                f"Context mismatch rate is {result.context_mismatch:.0%}. "
                "The retriever finds relevant chunks but the generator doesn't use them. "
                "Consider: context window management, instruction placement, "
                "or citation training."
            )
        if result.hallucination_rate > 0.1:
            result.recommendations.append(
                f"Hallucination rate is {result.hallucination_rate:.0%}. "
                "The system is adding information not in the retrieved context. "
                "Consider: stronger anti-hallucination prompting, "
                "lower LLM temperature, or retrieval-augmented verification."
            )

        return result
```

---

### 3. Synthetic Evaluation Dataset Generator

```python
"""
eval_dataset_generator.py — Generate synthetic evaluation datasets
that cover edge cases and corner conditions.

Critical insight: a RAG system that scores 0.95 on standard questions
might score 0.30 on edge cases. Your eval dataset MUST include both.
"""

from typing import List, Optional, Dict, Any, Tuple
from dataclasses import dataclass, field
import json
import asyncio
import random

from openai import AsyncOpenAI


# ─── Edge Case Types ────────────────────────────────────────────────────────

EDGE_CASE_TYPES = {
    "missing_context": {
        "description": "Questions about topics NOT in the knowledge base",
        "prompt_template": "Generate a question that a user might ask which is COMPLETELY OUT OF SCOPE for the provided document. The question should be plausible but about a topic the document doesn't cover.",
    },
    "ambiguous_query": {
        "description": "Queries with multiple interpretations",
        "prompt_template": "Rewrite this query to be intentionally ambiguous — it should have at least 2 plausible interpretations based on the document content.",
    },
    "multi_intent": {
        "description": "Queries asking multiple questions in one",
        "prompt_template": "Create a compound question that asks about MULTIPLE topics from the document. The question should contain 2-3 sub-questions that require different retrieval results.",
    },
    "negation": {
        "description": "Queries using negation that might confuse retrieval",
        "prompt_template": "Create a question about what IS NOT covered, what should NOT be done, or exceptions. Negation should be central to understanding the answer.",
    },
    "temporal": {
        "description": "Queries about time-specific information",
        "prompt_template": "Create a question about timing, deadlines, schedules, durations, or sequences. The answer requires understanding temporal relationships.",
    },
    "comparison": {
        "description": "Queries asking to compare or contrast",
        "prompt_template": "Create a question asking to compare two or more items, options, or approaches mentioned in the document.",
    },
    "conditional": {
        "description": "Queries with if-then conditions",
        "prompt_template": "Create a question with a conditional scenario: 'If X, then what about Y?' The answer depends on understanding the condition.",
    },
    "vocabulary_mismatch": {
        "description": "Queries using different words than the document",
        "prompt_template": "Rephrase a simple question about the document using COMPLETELY DIFFERENT VOCABULARY than the document uses. Use synonyms, colloquial terms, or different technical terms.",
    },
    "multi_hop": {
        "description": "Queries requiring information from multiple documents",
        "prompt_template": "Create a question that requires finding information from MULTIPLE SEPARATE parts of the document to answer. A single chunk cannot provide the full answer.",
    },
    "contradiction": {
        "description": "Queries where the document has contradicting information",
        "prompt_template": "Create a question about a topic where the document contains CONTRADICTORY information (e.g., the document has both 'limit is 25MB' and 'enterprise limit is 100MB'). The ideal answer would note the contradiction.",
    },
}


@dataclass
class EvalTestCase:
    """A single evaluation test case."""
    query: str
    expected_answer: str
    expected_contexts: List[str]
    reasoning_path: List[str]
    difficulty: str  # easy, medium, hard
    edge_case_type: str
    metadata: Dict[str, Any] = field(default_factory=dict)


class EvalDatasetGenerator:
    """
    Generate synthetic evaluation datasets with controlled difficulty
    and edge case coverage.

    The key insight: your eval dataset should NOT be a random sample of
    user queries. It should be STRATIFIED to cover:
    - Common cases (40%) — typical user questions
    - Edge cases (35%) — difficult situations that test system limits
    - Failure recovery (25%) — queries designed to trigger specific failures
    """

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        model: str = "gpt-4o-mini",
    ):
        self.llm_client = llm_client
        self.model = model

    async def generate_from_documents(
        self,
        documents: List[str],
        target_common: int = 20,
        target_edge: int = 15,
        edge_case_types: Optional[List[str]] = None,
    ) -> List[EvalTestCase]:
        """
        Generate a stratified evaluation dataset from documents.

        Args:
            documents: Source documents for context
            target_common: Number of common/case questions
            target_edge: Number of edge case questions
            edge_case_types: Edge case types to include (default: all)

        Returns:
            List of EvalTestCase objects
        """
        all_types = edge_case_types or list(EDGE_CASE_TYPES.keys())
        test_cases = []

        # Step 1: Generate COMMON cases (standard Q&A)
        common_prompt = f"""Given the following document content, generate {target_common} 
question-answer pairs that represent TYPICAL, COMMON user questions.

Each pair should:
- Be a realistic question a user would ask
- Be answerable from the document content
- Require 1-2 chunks of context
- Include the EXACT answer a perfect system should give

For each pair, also list the specific text passages that support the answer.

Document:
{chr(10).join(d[:3000] for d in documents[:3])}

Return JSON:
{{
    "test_cases": [
        {{
            "query": "user question",
            "expected_answer": "complete expected answer",
            "supporting_contexts": ["exact text passage 1", "exact text passage 2"],
            "reasoning_steps": ["Step 1 in reasoning"],
            "difficulty": "easy"
        }}
    ]
}}
"""
        common_cases = await self._generate_batch(common_prompt, target_common)
        test_cases.extend(common_cases)

        # Step 2: Generate EDGE CASES
        for edge_type in all_types:
            type_info = EDGE_CASE_TYPES[edge_type]
            count = max(1, target_edge // len(all_types))

            edge_prompt = f"""Given the following document content, generate {count} 
{edge_type} question-answer pairs.

Edge case type: {type_info['description']}

{type_info['prompt_template']}

Document:
{chr(10).join(d[:3000] for d in documents[:3])}

For each:
1. Create the question
2. Give the expected correct answer
3. List supporting contexts
4. Explain why this is a {edge_type} edge case

Return JSON with the same structure as above.
Assign difficulty based on complexity: easy/medium/hard.
"""
            edge_cases = await self._generate_batch(edge_prompt, count, edge_type)
            test_cases.extend(edge_cases)

        # Deduplicate by query similarity
        seen_queries = set()
        deduped = []
        for tc in test_cases:
            q_lower = tc.query.lower().strip()
            if q_lower not in seen_queries:
                seen_queries.add(q_lower)
                deduped.append(tc)

        logger.info(f"Generated {len(deduped)} eval test cases "
                    f"({target_common} common, {len(deduped) - target_common} edge)")

        return deduped

    async def _generate_batch(
        self,
        prompt: str,
        expected_count: int,
        edge_type: str = "common",
    ) -> List[EvalTestCase]:
        """Generate a batch of test cases from an LLM prompt."""
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You generate evaluation datasets for RAG systems."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.7,
            max_tokens=4000,
        )

        try:
            data = json.loads(response.choices[0].message.content)
        except (json.JSONDecodeError, AttributeError):
            return []

        test_cases = []
        for tc in data.get("test_cases", [])[:expected_count]:
            test_cases.append(EvalTestCase(
                query=tc.get("query", ""),
                expected_answer=tc.get("expected_answer", ""),
                expected_contexts=tc.get("supporting_contexts", []),
                reasoning_path=tc.get("reasoning_steps", []),
                difficulty=tc.get("difficulty", "medium"),
                edge_case_type=edge_type,
            ))

        return test_cases

    def stratified_split(
        self,
        test_cases: List[EvalTestCase],
        train_ratio: float = 0.3,
        test_ratio: float = 0.5,
        validate_ratio: float = 0.2,
    ) -> Tuple[List[EvalTestCase], List[EvalTestCase], List[EvalTestCase]]:
        """
        Split dataset into train/test/validate with stratification.

        Ensures each split has the same distribution of:
        - Edge case types
        - Difficulty levels
        """
        from collections import defaultdict

        # Group by edge case type, then difficulty
        groups = defaultdict(list)
        for tc in test_cases:
            key = f"{tc.edge_case_type}::{tc.difficulty}"
            groups[key].append(tc)

        train_set = []
        test_set = []
        validate_set = []

        for key, group in groups.items():
            random.shuffle(group)
            n = len(group)
            n_train = int(n * train_ratio)
            n_test = int(n * test_ratio)

            train_set.extend(group[:n_train])
            test_set.extend(group[n_train:n_train + n_test])
            validate_set.extend(group[n_train + n_test:])

        random.shuffle(train_set)
        random.shuffle(test_set)
        random.shuffle(validate_set)

        return train_set, test_set, validate_set

    def coverage_report(self, test_cases: List[EvalTestCase]) -> Dict[str, Any]:
        """Analyze what edge cases and difficulties are covered."""
        type_counts = {}
        difficulty_counts = {}
        for tc in test_cases:
            type_counts[tc.edge_case_type] = type_counts.get(tc.edge_case_type, 0) + 1
            difficulty_counts[tc.difficulty] = difficulty_counts.get(tc.difficulty, 0) + 1

        return {
            "total": len(test_cases),
            "by_type": type_counts,
            "by_difficulty": difficulty_counts,
            "coverage_score": len(type_counts) / len(EDGE_CASE_TYPES),
            "missing_types": [
                t for t in EDGE_CASE_TYPES if t not in type_counts
            ],
        }


# ─── Usage ──────────────────────────────────────────────────────────────────

async def generate_eval_dataset_demo():
    """Demo generating a stratified eval dataset."""
    client = AsyncOpenAI()
    generator = EvalDatasetGenerator(llm_client=client)

    documents = [
        "Our API rate limits: Free tier 1,000 requests/hour, Pro tier 10,000/hour, Enterprise 100,000/hour. Rate limits apply per API key.",
        "For Enterprise customers, max upload file size is 100MB. Pro and Free: 25MB. Upload via PUT /files endpoint.",
        "Password reset: Go to Settings > Security > Password. Enter current password, then new password twice. You'll receive a confirmation email.",
        "Enterprise SLA: 99.99% uptime guarantee. 15-minute response time for critical issues. 1-hour for high priority.",
    ]

    cases = await generator.generate_from_documents(
        documents,
        target_common=8,
        target_edge=8,
    )

    report = generator.coverage_report(cases)
    print("Coverage Report:")
    print(f"  Total: {report['total']}")
    print(f"  By type: {report['by_type']}")
    print(f"  By difficulty: {report['by_difficulty']}")
    print(f"  Missing types: {report['missing_types']}")

    print("\nSample test cases:")
    for tc in cases[:3]:
        print(f"  [{tc.difficulty}] [{tc.edge_case_type}] {tc.query[:80]}")
```

---

### 4. LLM-as-Judge Bias Mitigation

```python
"""
bias_mitigation.py — Detect and mitigate LLM-as-judge biases.

Five known biases and their mitigations:
1. Position bias — judge prefers first/last answer
2. Verbosity bias — judge prefers longer answers
3. Self-enhancement bias — judge prefers similar style
4. Contrast bias — exaggerates differences in comparison
5. Order effects — preceding evaluations influence current
"""

from typing import List, Optional, Dict, Any, Callable
from dataclasses import dataclass
import json
import random


@dataclass
class BiasReport:
    """Report on detected biases in LLM-as-judge evaluations."""
    position_bias_score: float = 0.0  # 0=no bias, 1=strong bias
    verbosity_bias_score: float = 0.0
    self_enhancement_bias_score: float = 0.0
    consistency_score: float = 0.0
    recommendations: List[str] = field(default_factory=list)


class BiasDetector:
    """
    Detect biases in LLM-as-judge evaluations.

    Run this periodically on your evaluation pipeline to ensure
    your metrics aren't being distorted by judge bias.
    """

    def __init__(self, judge_fn: Callable):
        """
        Args:
            judge_fn: A function that takes (query, response, context) and returns a score
        """
        self.judge = judge_fn

    async def detect_position_bias(
        self,
        test_queries: List[str],
        num_swaps: int = 10,
    ) -> float:
        """
        Detect position bias by running the same evaluation with
        different orderings of the contexts.

        Score close to 0 = no position bias.
        Score close to 1 = strong position bias.
        """
        biases = []
        for query in test_queries[:num_swaps]:
            score_original = await self.judge(query, "This is a test answer", ["Context A", "Context B"])
            score_swapped = await self.judge(query, "This is a test answer", ["Context B", "Context A"])

            # If order changes score significantly (>20%), that's position bias
            if score_original > 0 and score_swapped > 0:
                bias = abs(score_original - score_swapped) / max(score_original, score_swapped)
                biases.append(bias)

        return sum(biases) / len(biases) if biases else 0.0

    async def detect_verbosity_bias(
        self,
        test_queries: List[str],
    ) -> float:
        """
        Detect verbosity bias by comparing scores for:
        - Short, correct answer
        - Long, correct answer (same content, more verbose)
        - Short, incorrect answer
        - Long, incorrect answer

        If the judge consistently prefers longer answers regardless of
        correctness, there's verbosity bias.
        """
        biases = []

        test_pairs = [
            ("Short and direct answer: The rate limit is 1000/hour.", "Short"),
            ("The rate limit for the free tier is set at 1,000 requests per hour, which means you can make up to one thousand API calls in any given hour window. Please note that this limit applies per API key and resets at the beginning of each hour.", "Long"),
        ]

        for query in ["What's the rate limit?"]:
            for answer, length in test_pairs:
                score = await self.judge(query, answer, ["Rate limit: 1000 per hour. Resets hourly per API key."])
                biases.append((score, length))

        # If "Long" answers score consistently higher than "Short" answers
        if len(biases) >= 4:
            long_scores = [s for s, l in biases if l == "Long"]
            short_scores = [s for s, l in biases if l == "Short"]
            if long_scores and short_scores:
                avg_long = sum(long_scores) / len(long_scores)
                avg_short = sum(short_scores) / len(short_scores)
                if avg_short > 0:
                    return max(0, (avg_long - avg_short) / avg_short)

        return 0.0

    async def run_full_audit(
        self,
        test_queries: List[str],
    ) -> BiasReport:
        """Run full bias detection audit."""
        position = await self.detect_position_bias(test_queries)
        verbosity = await self.detect_verbosity_bias(test_queries)

        recommendations = []
        if position > 0.2:
            recommendations.append(
                f"Position bias detected ({position:.2f}). Mitigation: "
                "randomize context order, evaluate multiple orderings and average."
            )
        if verbosity > 0.2:
            recommendations.append(
                f"Verbosity bias detected ({verbosity:.2f}). Mitigation: "
                "add conciseness to your scoring rubric, normalize scores by length."
            )

        return BiasReport(
            position_bias_score=position,
            verbosity_bias_score=verbosity,
            consistency_score=1.0 - max(position, verbosity),
            recommendations=recommendations,
        )


class BiasMitigatedJudge:
    """
    LLM-as-judge with built-in bias mitigation strategies.

    Use this instead of raw LLM judge calls for production evaluation.
    """

    def __init__(self, llm_client, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def evaluate(
        self,
        query: str,
        response: str,
        contexts: List[str],
        num_rounds: int = 2,
    ) -> float:
        """
        Evaluate with bias mitigation.

        Strategy:
        1. Randomize context order each round
        2. Average scores across rounds (mitigates position bias)
        3. Use structured rubric (mitigates verbosity bias)
        """
        scores = []

        for round_num in range(num_rounds):
            shuffled_contexts = contexts.copy()
            random.shuffle(shuffled_contexts)

            score = await self._single_evaluation(
                query, response, shuffled_contexts
            )
            scores.append(score)

        # Average across rounds
        return sum(scores) / len(scores)

    async def _single_evaluation(
        self,
        query: str,
        response: str,
        contexts: List[str],
    ) -> float:
        """Single evaluation call with structured rubric."""
        prompt = f"""Evaluate this RAG system response using the rubric below.

RUBRIC:
- 5 (Excellent): Complete, accurate, concise, well-cited
- 4 (Good): Mostly correct, minor issues
- 3 (Adequate): Acceptable but has significant room for improvement  
- 2 (Poor): Missing key information or inaccurate
- 1 (Very Poor): Wrong or unhelpful

IMPORTANT: Length does NOT equal quality. A concise correct answer scores higher
than a verbose answer with the same information.

Query: {query}

Context:
{chr(10).join(f'[{i+1}] {c[:1500]}' for i, c in enumerate(contexts[:3]))}

Response to evaluate: {response}

Focus on:
1. Is EVERY claim in the response supported by the context?
2. Does the response fully address the query?
3. Is the response concise (no unnecessary information)?

Return JSON: {{"score": int (1-5), "reasoning": "explanation"}}
"""
        resp = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You are a RAG evaluation judge. Be strict and fair."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
        )

        try:
            data = json.loads(resp.choices[0].message.content)
            return max(0.0, min(1.0, (data.get("score", 3) - 1) / 4))
        except Exception:
            return 0.5


# ─── Multi-Judge Consensus ──────────────────────────────────────────────────

class MultiJudgeConsensus:
    """
    Evaluate using MULTIPLE judge models and take consensus.
    More expensive but more reliable than any single judge.
    """

    def __init__(self, judges: List[Callable]):
        """
        judges: List of judge functions, ideally using different models.
        """
        self.judges = judges

    async def evaluate(self, query: str, response: str, context: str) -> Dict:
        """Run all judges and compute consensus."""
        results = await asyncio.gather(*[
            judge(query, response, context) for judge in self.judges
        ])

        mean_score = sum(results) / len(results)
        std_dev = (
            sum((s - mean_score) ** 2 for s in results) / len(results)
        ) ** 0.5

        return {
            "mean_score": mean_score,
            "std_dev": std_dev,
            "individual_scores": results,
            "agreement": 1.0 - (std_dev / max(mean_score, 0.01)),
            # Agreement: 1.0 = perfect, lower = more disagreement
        }
```

---

### 5. CI/CD Evaluation Pipeline

```python
"""
eval_pipeline.py — CI/CD integration for RAG evaluation.

This runs evaluation as part of your deployment pipeline.
It catches regressions BEFORE they reach production.
"""

from typing import List, Optional, Dict, Any
from dataclasses import dataclass, field
import json
import time
import os


@dataclass
class PipelineConfig:
    """Configuration for the evaluation pipeline."""
    eval_dataset_path: str = "eval_dataset.json"
    min_composite_score: float = 0.75
    min_faithfulness: float = 0.80
    max_hallucination_rate: float = 0.10
    max_latency_ms: float = 5000.0
    max_cost_per_query: float = 0.05
    run_component_eval: bool = True
    run_regression_tests: bool = True
    fail_on_regression: bool = True
    report_path: str = "eval_report.json"


@dataclass
class PipelineResult:
    """Result of an evaluation pipeline run."""
    passed: bool
    config: PipelineConfig
    component_results: Optional[ComponentEvalResult] = None
    regression_results: Optional[RegressionReport] = None
    summary: Dict[str, Any] = field(default_factory=dict)
    failures: List[str] = field(default_factory=list)
    run_id: str = ""
    timestamp: str = ""


class EvalPipeline:
    """
    CI/CD pipeline for RAG evaluation.

    Runs as part of deployment. Fails the build if:
    - Any score drops below minimum threshold
    - Hallucination rate exceeds threshold
    - Latency exceeds threshold
    - Regression tests fail
    """

    def __init__(
        self,
        config: PipelineConfig,
        evaluator: AdvancedRAGEvaluator,
        component_evaluator: ComponentEvaluator,
        regression_runner: RegressionTestRunner,
    ):
        self.config = config
        self.evaluator = evaluator
        self.component_evaluator = component_evaluator
        self.regression_runner = regression_runner

    async def run(
        self,
        test_cases: List[EvalSample],
        regression_tests: Optional[List[RegressionTest]] = None,
        system_fn: Optional[Callable] = None,
    ) -> PipelineResult:
        """
        Run the full evaluation pipeline.

        Returns PipelineResult with pass/fail status and detailed results.
        """
        import uuid
        from datetime import datetime

        run_id = str(uuid.uuid4())[:8]
        timestamp = datetime.utcnow().isoformat()
        failures = []

        print(f"\n{'='*60}")
        print(f"RAG EVALUATION PIPELINE — RUN {run_id}")
        print(f"{'='*60}")

        # Step 1: End-to-end evaluation
        print(f"\n📊 Evaluating {len(test_cases)} test cases...")
        eval_results = await self.evaluator.evaluate_batch(test_cases)
        summary = self.evaluator.summary(eval_results)

        composite = summary.get("composite", {}).get("mean", 0)
        faithfulness = summary.get("faithfulness", {}).get("mean", 0)

        print(f"  Composite: {composite:.3f} (threshold: {self.config.min_composite_score})")
        print(f"  Faithfulness: {faithfulness:.3f} (threshold: {self.config.min_faithfulness})")

        if composite < self.config.min_composite_score:
            failures.append(f"Composite score {composite:.3f} below threshold {self.config.min_composite_score}")
        if faithfulness < self.config.min_faithfulness:
            failures.append(f"Faithfulness {faithfulness:.3f} below threshold {self.config.min_faithfulness}")

        # Step 2: Component-level evaluation
        component_result = None
        if self.config.run_component_eval and self.component_evaluator:
            print(f"\n🔧 Running component-level evaluation...")
            component_result = await self.component_evaluator.evaluate(test_cases)
            print(component_result.summary())

            if component_result.hallucination_rate > self.config.max_hallucination_rate:
                failures.append(
                    f"Hallucination rate {component_result.hallucination_rate:.1%} "
                    f"exceeds threshold {self.config.max_hallucination_rate:.0%}"
                )

        # Step 3: Regression tests
        regression_result = None
        if self.config.run_regression_tests and regression_tests and system_fn:
            print(f"\n🧪 Running {len(regression_tests)} regression tests...")
            regression_result = await self.regression_runner.run_tests(
                regression_tests, system_fn
            )
            print(f"  Passed: {regression_result.passed}")
            print(f"  Failed: {regression_result.failed}")

            if regression_result.failed > 0 and self.config.fail_on_regression:
                failures.append(f"{regression_result.failed} regression tests failed")
                for f in regression_result.failures[:3]:
                    failures.append(f"  - {f['query'][:60]}: {list(f['failures'].keys())}")

        # Step 4: Latency check
        avg_latency = summary.get("avg_latency_ms", 0)
        if avg_latency > self.config.max_latency_ms:
            failures.append(
                f"Avg latency {avg_latency:.0f}ms exceeds threshold {self.config.max_latency_ms}ms"
            )

        passed = len(failures) == 0

        result = PipelineResult(
            passed=passed,
            config=self.config,
            component_results=component_result,
            regression_results=regression_result,
            summary=summary,
            failures=failures,
            run_id=run_id,
            timestamp=timestamp,
        )

        # Write report
        self._write_report(result)

        print(f"\n{'='*60}")
        print(f"{'✅ PASSED' if passed else '❌ FAILED'}")
        if failures:
            for f in failures:
                print(f"  • {f}")
        print(f"{'='*60}")

        return result

    def _write_report(self, result: PipelineResult):
        """Write evaluation report to disk."""
        report = {
            "run_id": result.run_id,
            "timestamp": result.timestamp,
            "passed": result.passed,
            "failures": result.failures,
            "config": {
                "min_composite_score": self.config.min_composite_score,
                "min_faithfulness": self.config.min_faithfulness,
                "max_hallucination_rate": self.config.max_hallucination_rate,
            },
            "summary": result.summary,
        }

        os.makedirs(os.path.dirname(self.config.report_path) or ".", exist_ok=True)
        with open(self.config.report_path, "w") as f:
            json.dump(report, f, indent=2, default=str)

        logger.info(f"Report written to {self.config.report_path}")
```

---

## ✅ Good Output Examples

### Component Diagnosis Example

```
SYMPTOM: End-to-end faithfulness = 0.72

COMPONENT EVALUATION:
  Retriever:     P=0.95  R=0.88  MRR=0.91  ✅ Excellent
  Generator:     F=0.75  C=0.82               ❌ Faithfulness issue
  
  Interaction:   Context mismatch = 35%       ❌ Generator ignores context
                 Hallucination rate = 12%      ❌ Hallucinates

DIAGNOSIS:
  The retriever finds the right documents (88% recall).
  But the generator doesn't use them properly:
  - 35% of the time, relevant context is retrieved but not reflected in the answer
  - 12% of answers contain claims not supported by any context

ROOT CAUSE:
  Generator instruction prompt doesn't emphasize citation.
  Model is too creative (temperature too high).
  Context window shows retrieved chunks but doesn't label them clearly.

RECOMMENDATIONS:
  1. Lower generator temperature to 0.1
  2. Add "Cite your sources using [Source: N]" to system prompt
  3. Add 3 in-context examples showing proper citation
  4. Implement citation verification post-processing
```

### Evaluation Dataset Stratification

```
DATASET COVERAGE REPORT:
  Total: 100 test cases
  By type:
    common:             40 (40%)  ✅ Good coverage
    missing_context:    10 (10%)  ✅ 
    ambiguous_query:    8  (8%)   ✅
    multi_intent:       8  (8%)   ✅
    negation:           6  (6%)   ✅
    temporal:           6  (6%)   ✅
    comparison:         6  (6%)   ✅
    conditional:        5  (5%)   ✅
    vocabulary_mismatch:5  (5%)   ⚠️ Low coverage
    multi_hop:          4  (4%)   ⚠️ Low coverage
    contradiction:      2  (2%)   ❌ Too few

  By difficulty:
    easy:   40 (40%)
    medium: 40 (40%)
    hard:   20 (20%)
```

### CI/CD Pipeline Output

```
RAG EVALUATION PIPELINE — RUN a1b2c3d4
════════════════════════════════════════════════════════

📊 Evaluating 80 test cases...
  Composite:   0.812 ✅ (threshold: 0.75)
  Faithfulness: 0.834 ✅ (threshold: 0.80)

🔧 Component-level evaluation...
  Retriever:   P=0.88 R=0.85 MRR=0.89
  Generator:   F=0.83 C=0.86
  Context mismatch: 12% ✅
  Hallucination rate: 3% ✅ (threshold: 10%)

🧪 Running 20 regression tests...
  Passed: 19  Failed: 1
  Regression detected in: "What's the enterprise rate limit?"
    → Faithfulness dropped from 0.92 → 0.71 (22% drop)
    → Cause: Retrieved enterprise docs but answered with free tier info

❌ FAILED
  • Regression test: "What's the enterprise rate limit?" — 22% faithfulness drop
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Evaluating on a "Convenience" Dataset

```
WRONG:
  Test set = 20 questions you wrote yourself in 10 minutes
  → Easy questions, no edge cases, biased toward what you know
  → Score = 0.95 → gives false confidence

RIGHT:
  Test set = 100 questions generated with stratified distribution
  → 40% common, 35% edge cases, 25% recovery patterns
  → Score = 0.78 → honest assessment of gaps
```

### Antipattern 2: Using the Same LLM as Generator and Judge

```
WRONG:
  Generator: GPT-4o-mini
  Judge: GPT-4o-mini
  → Self-enhancement bias: judge prefers similar-sounding answers
  → Misses subtle errors because judge shares the same failure modes

RIGHT:
  Generator: GPT-4o-mini
  Judge: Claude 3 Haiku or Gemini 1.5 Flash
  → Different model families → different failure modes
  → More reliable evaluation
```

### Antipattern 3: Only Evaluating End-to-End

```
WRONG:
  Score = faithfulness(query, response, context)
  → Can't tell if retriever or generator is the problem
  → Don't know where to focus improvement effort

RIGHT:
  Component-level evaluation + end-to-end
  → Knows retriever recall is fine but generator faithfulness is low
  → Actionable: improve generator prompting, not retriever
```

### Antipattern 4: Threshold-Based Pass/Fail Without Trends

```
WRONG:
  if score > 0.80: PASS
  if score < 0.80: FAIL
  → Ignores trend: score dropping from 0.89 → 0.85 → 0.82 → 0.79
  → The drop is a warning signal LONG before it hits 0.80

RIGHT:
  Track score history, alert on drops > 10% even if above threshold
  → Catches regressions early, when they're small
  → Prevents sudden threshold violations at deployment time
```

### Failure Mode: Judge Hallucination

The LLM judge itself can hallucinate — claiming the response contains information that isn't there, or missing actual errors.

**Mitigation:** Use calibrated examples. Include 5 known-good and 5 known-bad answers in every eval batch. If the judge scores known-bad answers above 0.5, the judge is unreliable.

### Failure Mode: Test Set Contamination

If your test queries are GENERATED from the same documents your RAG system indexes, you're measuring MEMORIZATION, not RETRIEVAL + REASONING. The system might answer correctly because the training data included similar Q&A pairs, not because it retrieved and reasoned.

**Mitigation:** Hold out 20% of documents from indexing. Generate test queries ONLY from held-out documents. If the system answers correctly, it's actually retrieving and reasoning.

### Failure Mode: Feedback Loop Blindness

Your evaluation says faithfulness is 0.92. You ship. Users report wrong answers.

The issue: your evaluation dataset consists of queries where the GROUND TRUTH matches the DOCUMENTS perfectly. In production, users ask questions where:
- The documents don't contain the answer → the system should say "I don't know"
- The documents contradict each other → the system should surface the contradiction
- The question is ambiguous → the system should ask clarifying questions

None of these are captured by "faithfulness" because faithfulness assumes the answer IS in the context.

**Mitigation:** Include "missing context" and "contradiction" cases in your eval dataset. Measure whether the system correctly declines to answer when information is absent.

---

## 🧪 Drills & Challenges

### Drill 1: Build an Eval Dataset with 100% Edge Case Coverage (45 min)

**Task:** Generate an evaluation dataset with AT LEAST 2 test cases for each of the 10 edge case types.

**Success criteria:** Your `coverage_report` shows `coverage_score: 1.0` — no missing types.

**Test:** Run your RAG system against all edge cases. Which types score lowest? That's your improvement target.

### Drill 2: Detect and Fix Judge Bias (45 min)

**Task:** Create a deliberately biased judge by modifying your evaluation prompt:
1. Add "Always prefer longer, more detailed answers" to the judge prompt → simulate verbosity bias
2. Run the `BiasDetector` — confirm >0.3 verbosity bias
3. Fix the judge prompt to remove the bias
4. Verify bias score drops below 0.15

**Success criteria:** Bias score drops from >0.3 to <0.15 after prompt fix.

### Drill 3: Component-Level Diagnosis (30 min)

**Task:** Given a RAG system with these symptoms:
- End-to-end faithfulness score = 0.65
- Users report "wrong answers"
- Some answers are correct, some are totally wrong

Use component-level evaluation to determine:
1. Is the retriever finding the right docs?
2. Is the generator using the retrieved docs?
3. Is the system hallucinating?
4. What's the ROOT CAUSE?

**Success criteria:** Your diagnosis correctly identifies the failing component and recommends a specific fix.

### Drill 4: Regression Test Suite (30 min)

**Task:** Create a regression test suite with 10 queries. For each query:
1. Run it on your CURRENT system → record baseline scores
2. INTENTIONALLY break your system (e.g., remove reranking, increase chunk size, change model)
3. Run regression tests → confirm they DETECT the regression
4. Fix the system → confirm tests PASS again

**Success criteria:** Regression tests catch ALL intentional degradations.

### Drill 5: Cost-Quality-Latency Optimization (45 min)

**Task:** Run a full evaluation across 3 configurations:
1. Cheap: gpt-4o-mini, no reranking, small context
2. Balanced: gpt-4o-mini, with reranking, medium context
3. Premium: gpt-4o, with reranking + HyDE, large context

For each config, measure:
- Composite quality score
- Average latency per query
- Cost per query
- Estimated daily cost (10,000 queries/day)

**Calculate:** Which configuration gives the BEST quality-per-dollar?

**Challenge:** Can you build a TIERED system that routes simple queries to config 1 and hard queries to config 3, achieving 90% of config 3's quality at 40% of its cost?

### Drill 6: Multi-Judge Consensus (30 min)

**Task:** For 5 test queries, compare:
1. Single judge (gpt-4o-mini) score
2. Multi-judge consensus (3 different models)
3. Your own human score

**Analyze:**
- How often does the single judge disagree with the consensus?
- How often does the consensus disagree with human judgment?
- Is the multi-judge consensus worth 3x the eval cost?

### Drill 7: Production Eval Dashboard (60 min)

**Task:** Build a simple dashboard that shows:
1. Score trends over time (last 10 eval runs)
2. Per-component scores (retriever vs. generator vs. orchestrator)
3. Regression alerts (new vs. old system)
4. Cost-latency-quality summary

**Data model:**

```python
@dataclass
class EvalHistoryEntry:
    run_id: str
    timestamp: str
    version: str  # system version
    composite: float
    retriever_recall: float
    generator_faithfulness: float
    hallucination_rate: float
    avg_latency_ms: float
    cost_per_query: float
    regression_failures: int
```

**Output:** A text-based or HTML dashboard that a team lead can review before approving deployment.

---

## 🚦 Gate Check

Before moving to File 08 (Advanced Drills), confirm you can:

- [ ] Build an advanced RAG evaluation with aspect critique and composite scoring
- [ ] Implement component-level evaluation that isolates retriever from generator
- [ ] Generate a stratified evaluation dataset covering 10 edge case types
- [ ] Detect and mitigate LLM-as-judge biases (position, verbosity, self-enhancement)
- [ ] Implement multi-judge consensus evaluation
- [ ] Build a CI/CD evaluation pipeline with regression tests
- [ ] Design a cost-quality-latency optimization study
- [ ] Know the difference between measuring RETRIEVAL vs. GENERATION quality
- [ ] Understand when evaluation metrics give FALSE CONFIDENCE

**Sample gate questions:**

1. **Your RAG system scores 0.92 faithfulness but users report wrong answers. What's the most likely cause? How would you diagnose it?**

2. **Design an evaluation strategy for a multi-hop RAG system. The system may: (a) traverse the full reasoning chain, (b) short-circuit to a correct answer without full reasoning, or (c) guess wrong. How do you evaluate each scenario differently?**

3. **Your LLM-as-judge is biased toward verbose answers. What are 3 concrete mitigations you can implement, from least to most expensive?**

4. **You have $100/month for evaluation. Your RAG system handles 5,000 queries/day. Design an evaluation strategy that maximizes confidence within your budget.**

---

## 📚 Resources

- **RAGAS (Shahul et al., 2024):** RAGAS: Automated Evaluation of Retrieval Augmented Generation — arXiv:2309.15217
- **ARES (Saad-Falcon et al., 2024):** ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems — arXiv:2311.09476
- **RGB (Chen et al., 2024):** RGB: A Comprehensive Evaluation Benchmark for Retrieval-Augmented Generation — arXiv:2401.00336
- **LLM-as-Judge Bias Survey (2024):** Large Language Models are Not Fair Evaluators — arXiv:2405.01576
- **Position Bias in LLM Evaluation:** Examining the Impact of Presentation Order — Wang et al., 2024
- **RAGAS Documentation:** https://docs.ragas.io/
- **DeepEval:** Open-source LLM evaluation framework — https://github.com/confident-ai/deepeval
- **Arize Phoenix:** LLM observability with eval integrations — https://github.com/Arize-ai/phoenix
- **Langfuse Evaluation:** https://langfuse.com/docs/scores/model-based-evals
- **RAG Triad (TruLens):** A holistic evaluation framework — https://www.trulens.org/
