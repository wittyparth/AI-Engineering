# 02 — LLM-as-Judge: When the Evaluator Is Also an LLM

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about building LLM judges, you need to feel the problem first-hand.

LLM-as-Judge is the most popular evaluation technique in AI engineering right now. The idea is simple: instead of humans manually rating every output, you ask another LLM (usually a stronger one) to evaluate your system's outputs.

But here's the thing — **this idea has more hidden failure modes than almost any other technique in this course.** And most people only discover them after they've been using a broken judge for months.

Let me show you what I mean through 3 scenarios. Don't just read them — **pause after each and think about what you'd do.**

---

### 🤔 Discovery Question 1: The Judge That Has Favorites

You're building a summarization system. Your pipeline generates summaries, and you use GPT-4 as a judge to rate them on a scale of 1-5 on "clarity."

You run 500 examples through your judge. Average score: **4.2/5**. Great.

But then you switch your summarization model from GPT-4o-mini to Claude 3.5 Haiku. The scores DROP to **3.5/5**. You panic and revert.

Later, you manually review 50 examples from both models. **Claude's summaries were actually BETTER** — more concise, more accurate, better structured. But GPT-4 consistently scored them lower.

You dig deeper and discover the pattern:
- GPT-4 gave GPT-4o-mini's verbose summaries higher scores ("provides comprehensive detail")
- GPT-4 marked down Claude's concise summaries ("lacks sufficient detail")
- When you ask Claude to judge the same pairs, the reverse happens

**🤔 Before reading on, answer these:**

1. Why does GPT-4 prefer GPT-4o-mini's style? What specifically about LLM training could cause this?

2. If you can't use the SAME model as both generator and judge (biased), but you also can't afford a STRONGER model as judge (too expensive), WHAT do you do? Design a solution that works with what you have.

3. Think about this from a loss function perspective — when your evaluation system has a systematic bias toward one style over another, what happens to your optimization process? If you optimize for a biased judge's score, what kind of system do you end up with?

---

### 🤔 Discovery Question 2: The Position Spinner

You have a simple eval prompt:

```
Rate this answer on accuracy (1-5):

Question: {question}
Reference Answer: {reference}
System Answer: {answer}

Score:
```

You test your judge on 100 examples. Accuracy correlation with human ratings: **0.82**. Solid.

But then someone on your team runs an experiment: they take the SAME answer and place it at DIFFERENT positions in the prompt:

```
Version A: Reference first, then system answer
Version B: System answer first, then reference
```

The judge gives DIFFERENT scores to the same answer depending on which position it appears in. The difference is 0.4-0.7 points on average.

**🤔 Before reading on, answer these:**

1. Why would the position of information in the prompt change the judge's score? What does this tell you about how LLMs process long contexts?

2. You discover that when the reference answer appears FIRST, the judge is stricter (lower scores). When the system answer appears FIRST, the judge is more lenient. What's a plausible explanation for this pattern?

3. Design an eval prompt structure that MINIMIZES position bias. What constraints would you put on the prompt format? Is there a way to "average out" position effects without changing the model?

---

### 🤔 Discovery Question 3: The Compassion Fatigue Problem

You're running a large-scale evaluation. 10,000 system outputs need to be scored. You batch them into groups of 10 and ask GPT-4 to score each batch.

You notice something strange: the scores in the FIRST batch are consistently lower than scores in LATER batches. On average, the first batch scores 3.1/5 and the 10th batch scores 3.8/5.

You test with 5 different evaluation rubrics. **The pattern persists across all of them.** The later in the evaluation session, the higher the scores.

**🤔 Before reading on, answer these:**

1. This is called "compassion fatigue" or "score creep" — the judge gets more lenient over time. Why would an LLM show this pattern? Does it "get tired"? What's actually happening under the hood?

2. You decide to evaluate each output INDIVIDUALLY (no batching) to prevent this. Now 10,000 evaluations cost you 10,000 API calls instead of 1,000. Your eval bill goes up 10x. Your CTO asks: "Is this really necessary?" What do you say? How do you quantify the cost of the bias vs the cost of preventing it?

3. More subtle: What if the score creep is actually BETTER for catching regressions? Think about it — if the bias is CONSISTENT across all evaluation runs, then comparing scores within the same run is still valid. The creep only matters when comparing across different runs. Does this change your strategy for when to batch vs. when to evaluate individually?

---

> **Keep these 3 problems in mind. Everything in this module will help you solve them.**

By the end of this module, you will:
- Understand the 4 major LLM judge biases and how to detect each
- Build an LLM judge pipeline with structured prompts and calibration
- Implement multi-judge ensembles to reduce bias
- Measure judge reliability (not just accuracy, but consistency)
- Know when LLM-as-judge is the WRONG tool — and what to use instead

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions (above) | 20 min — really think about them |
| The 4 Judge Biases — Deep Dive | 30 min |
| Calibration: How to Trust Your Judge | 20 min |
| Structured Eval Prompts | 20 min |
| Code: Basic LLM Judge (with bugs) | 25 min |
| Code: Fixing Position Bias | 20 min |
| Code: Multi-Judge Ensemble | 30 min |
| Code: Judge Calibration & Reliability | 30 min |
| Code: Judge-Generator Pair Analysis | 25 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 50 min |
| Gate Check | 10 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The 4 Judge Biases (That Everyone Encounters)

Every LLM judge has systematic biases. Understanding them is the first step to compensating for them.

#### Bias 1: Self-Enhancement Bias

The judge prefers outputs that match its OWN style.

| Judge Model | Generator Model | Typical Score Difference |
|-------------|----------------|-------------------------|
| GPT-4 | GPT-4o-mini | +0.3 to +0.8 (favors GPT) |
| Claude 3 | Claude 3 Haiku | +0.3 to +0.8 (favors Claude) |
| GPT-4 | Claude 3 | -0.4 to -0.9 (penalizes Claude) |
| Claude 3 | GPT-4 | -0.4 to -0.9 (penalizes GPT) |

**Why it happens:** LLMs are trained to predict patterns. Their OWN outputs are the most "natural" to them. When they see a different model's output, it looks slightly "off" — not necessarily wrong, but the judge subconsciously (inasmuch as an LLM can be subconscious) penalizes the difference.

**Detection experiment:** Swap judge and generator. If scores change asymmetrically, you have self-enhancement bias.

#### Bias 2: Position Bias

The judge gives different scores based on where information appears in the prompt.

- **Primary effect:** Information at the beginning gets more weight
- **Recency effect:** Information at the end gets more weight
- Which dominates depends on the prompt structure and model

**Why it happens:** Attention mechanisms aren't perfectly uniform. The model's attention tends to cluster at the beginning or end of long contexts, depending on the architecture.

**Detection experiment:** Create pairs of identical evaluations where you swap the order of reference and system answer. If scores differ, you have position bias.

#### Bias 3: Verbosity Bias

The judge prefers LONGER answers, regardless of quality.

```
Answer A (scored 4/5): "The capital of France is Paris, a city 
located in the north-central part of the country..."
(85 words)

Answer B (scored 3/5): "Paris."
(2 words)
```

Both answers are CORRECT. But the judge penalized the concise answer for lack of "detail."

**Why it happens:** LLMs associate verbosity with thoroughness. A longer answer "feels" more complete, more authoritative. This is a training data artifact — in the training corpus, longer answers tend to be higher quality (blog posts, papers, books). But for evaluation, this is a bug, not a feature.

**Detection experiment:** Test with known pairs where the shorter answer is correct. Check if the judge consistently penalizes brevity.

#### Bias 4: Compassion Fatigue (Score Creep)

The judge gets more lenient over the course of an evaluation session.

| Batch Number | Average Score | Standard Deviation |
|-------------|--------------|-------------------|
| 1 (first 10) | 3.1 | 0.8 |
| 5 (items 41-50) | 3.4 | 0.7 |
| 10 (items 91-100) | 3.8 | 0.6 |
| 20 (items 191-200) | 4.1 | 0.4 |

Four effects compound:
1. **Drift:** The model's internal state changes over long contexts
2. **Contrast effect:** After seeing terrible answers, mediocre ones look good
3. **Satisficing:** The judge gets "lazy" and gives higher scores to avoid thinking
4. **Scale compression:** The variance decreases — scores cluster more tightly

**Detection experiment:** Evaluate the same 10 examples at the START and END of a long evaluation session. If scores are systematically higher at the end, you have compassion fatigue.

---

### 2. Measuring Judge Reliability

Before you trust your judge, you need to measure:

**Self-Consistency:** Give the judge the SAME input twice. Does it give the same score?

```python
self_consistency = same_input_same_output_ratio
# Target: >0.90 (90% of repeated evaluations give the same score)
```

**Inter-Judge Agreement:** Give two DIFFERENT judges the same input. Do they agree?

```python
inter_judge_agreement = correlation(judge_a_scores, judge_b_scores)
# Target: >0.70 (Pearson correlation)
```

**Human Correlation:** Compare judge scores with human expert scores.

```python
human_correlation = correlation(judge_scores, human_scores)
# Target: >0.80 on held-out test set
```

**🤔 Checkpoint:** Which of these 3 metrics is MOST important for catching regressions? Which is MOST important for trusting absolute scores? Your answer reveals what you care about — and what your tradeoff will be.

---

### 3. The Judge-Generator Gap

There's a fundamental tension in LLM-as-Judge:

- Your GENERATOR model is what you're evaluating (e.g., GPT-4o-mini)
- Your JUDGE model should be STRONGER than your generator (e.g., GPT-4)

But what if you can't afford a stronger judge? Or what if your generator is already the strongest model available?

```
┌─────────────────────────────────────────────────────┐
│             JUDGE-GENERATOR STRATEGIES               │
├──────────────┬──────────┬───────────┬────────────────┤
│ Strategy     │ Bias     │ Cost      │ When to Use    │
├──────────────┼──────────┼───────────┼────────────────┤
│ Same model   │ HIGH     │ LOWEST    │ Development    │
│ (judge=gen)  │ (self)   │           │ only           │
├──────────────┼──────────┼───────────┼────────────────┤
│ Stronger     │ LOW      │ MEDIUM    │ Production     │
│ model judge  │          │           │ evaluation     │
├──────────────┼──────────┼───────────┼────────────────┤
│ Different    │ MEDIUM   │ LOW       │ Cross-check    │
│ model judge  │ (cross)  │           │ baseline       │
├──────────────┼──────────┼───────────┼────────────────┤
│ Multi-judge  │ LOWEST   │ HIGHEST   │ Critical       │
│ ensemble     │          │           │ evaluations    │
├──────────────┼──────────┼───────────┼────────────────┤
│ Judge +      │ NEAR     │ HIGHEST   │ Gold standard  │
│ Human spot   │ ZERO     │           │ spot checks    │
│ check        │          │           │                │
└──────────────┴──────────┴───────────┴────────────────┘
```

**🤔 Your turn to decide:** You have a budget of $100/day for evaluation. Your generator model costs $0.01/query. A judge-as-stronger-model costs $0.05/eval. A multi-judge ensemble (3 judges) costs $0.15/eval.

You need to evaluate 2,000 queries per day. What's your strategy? How many samples do you evaluate with each method? How do you use the EXPENSIVE methods to validate the CHEAP ones?

---

## 💻 Code Examples

### Example 1: The "Almost Right" LLM Judge (With Deliberate Bugs)

Let me show you a judge that looks reasonable but has hidden problems. **Read this carefully and find the bugs before running it.**

```python
"""
llm_judge_v1.py — A judge with 3 hidden bugs.

Read through this code. Find the bugs BEFORE executing.
Think about what each bug would cause in production.
"""

from openai import OpenAI
from pydantic import BaseModel
import json


class EvalScore(BaseModel):
    score: int  # 1-5
    reasoning: str


def judge_answer(question: str, reference: str, answer: str) -> EvalScore:
    """Evaluate a system answer against a reference answer."""
    client = OpenAI()
    
    prompt = f"""
    You are an AI evaluator. Rate the system answer on a scale of 1-5.
    
    Question: {question}
    Reference Answer: {reference}
    System Answer: {answer}
    
    Score the system answer based on accuracy and completeness.
    
    Return your score and reasoning as JSON:
    {{"score": int, "reasoning": "..."}}
    """
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",  # Using same model as generator to save cost
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,
    )
    
    result = json.loads(response.choices[0].message.content)
    return EvalScore(**result)
```

**🤔 Find the 3 bugs:**

1. *Think about what we discussed above. Which biases would this prompt trigger?*
2. *Look at the model choice. Remember the judge-generator gap.*
3. *Look at the output parsing. What happens if the model doesn't return valid JSON?*

**Don't scroll further until you've found all 3. Write them down.**

---

...

**Bug 1: Self-enhancement bias.** The judge (GPT-4o-mini) is the SAME model as the generator. This is the worst possible configuration — maximum self-enhancement bias with minimum evaluation quality. You're evaluating a model with itself. It will almost always rate itself highly.

**Bug 2: Unstructured prompt invites position bias.** The reference and answer are placed in an unstructured way. The model can arbitrarily weight whichever it "notices" first. No instructions about how to handle the comparison, no explicit criteria, no guidelines about brevity vs. verbosity.

**Bug 3: No error handling on JSON parsing.** If the model returns slightly malformed JSON (and it will — especially with temperature=0, which doesn't guarantee valid JSON), the entire evaluation crashes. A single malformed response takes down your whole batch.

---

### Example 2: The Fixed Judge (Structured Prompt + Calibration)

Now let's fix the judge. Notice how each fix addresses a specific bias.

```python
"""
llm_judge_v2.py — Fixed judge with structured prompts and calibration.
"""

from openai import OpenAI
from pydantic import BaseModel, Field
from typing import Optional
import instructor
import json


class EvalCriteria(BaseModel):
    """Individual criterion score."""
    score: int = Field(..., ge=1, le=5, description="Score 1-5")
    reasoning: str = Field(..., description="Brief justification")


class EvalResult(BaseModel):
    """Structured evaluation output."""
    accuracy: EvalCriteria
    completeness: EvalCriteria
    clarity: EvalCriteria
    overall_score: float = Field(..., ge=1.0, le=5.0)
    explanation: str = ""


class LLMJudge:
    """
    A production-grade LLM judge with bias mitigation.
    
    Features:
    - Structured output via Instructor (not raw JSON parsing)
    - Multi-criteria scoring (not one blob score)
    - Explicit instruction to handle verbosity
    - Position-randomized comparison
    """
    
    def __init__(self, model: str = "gpt-4o"):
        # Use a STRONGER model as judge than your generator
        # If generator is gpt-4o-mini, judge should be gpt-4o
        # If generator is gpt-4o, judge should be gpt-4o or Claude 3.5 Sonnet
        self.model = model
        self.client = instructor.from_openai(OpenAI())
    
    def evaluate(
        self,
        question: str,
        reference: str,
        answer: str,
        swap_position: bool = False,
    ) -> EvalResult:
        """
        Evaluate a system answer against reference.
        
        Args:
            swap_position: If True, puts answer BEFORE reference.
                          Run both ways and average to fix position bias.
        """
        if swap_position:
            # Randomize order to detect position bias
            comparison = f"System Answer: {answer}\n\nReference Answer: {reference}"
        else:
            comparison = f"Reference Answer: {reference}\n\nSystem Answer: {answer}"
        
        prompt = f"""
        You are evaluating a system's answer to a question.
        
        Question: {question}
        
        {comparison}
        
        Evaluate the SYSTEM ANSWER (not the reference) on three criteria:
        
        1. ACCURACY — Is the system answer factually correct? 
           Does it match or exceed the reference?
           NOTE: A concise correct answer is better than a verbose partially-wrong one.
           Do NOT penalize shorter answers if they are accurate.
        
        2. COMPLETENESS — Does the answer fully address the question?
           Consider: does it cover everything the question asks?
        
        3. CLARITY — Is the answer well-structured and easy to understand?
        
        For each criterion, provide a score (1-5) and brief reasoning.
        Then provide an overall weighted score (1-5) and a short explanation.
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            response_model=EvalResult,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
        )
        
        return response
    
    def evaluate_with_position_correction(
        self,
        question: str,
        reference: str,
        answer: str,
    ) -> EvalResult:
        """
        Evaluate twice with swapped positions and average the results.
        This mitigates position bias.
        """
        result_a = self.evaluate(question, reference, answer, swap_position=False)
        result_b = self.evaluate(question, reference, answer, swap_position=True)
        
        # Average the scores
        avg_score = (result_a.overall_score + result_b.overall_score) / 2.0
        
        # Check for position bias
        position_diff = abs(result_a.overall_score - result_b.overall_score)
        if position_diff > 0.5:
            print(f"⚠️ Position bias detected: diff={position_diff:.2f}")
            print(f"  Order A (reference first): {result_a.overall_score}")
            print(f"  Order B (answer first): {result_b.overall_score}")
        
        return EvalResult(
            accuracy=EvalCriteria(
                score=round((result_a.accuracy.score + result_b.accuracy.score) / 2),
                reasoning=f"A: {result_a.accuracy.reasoning} | B: {result_b.accuracy.reasoning}",
            ),
            completeness=EvalCriteria(
                score=round((result_a.completeness.score + result_b.completeness.score) / 2),
                reasoning=f"A: {result_a.completeness.reasoning} | B: {result_b.completeness.reasoning}",
            ),
            clarity=EvalCriteria(
                score=round((result_a.clarity.score + result_b.clarity.score) / 2),
                reasoning=f"A: {result_a.clarity.reasoning} | B: {result_b.clarity.reasoning}",
            ),
            overall_score=round(avg_score, 2),
            explanation=f"Averaged across 2 evaluations. Position diff: {position_diff:.2f}",
        )
    
    def detect_verbosity_bias(self, test_pairs: list[tuple]) -> float:
        """
        Detect verbosity bias by testing known pairs.
        
        test_pairs: list of (short_good_answer, long_bad_answer, question, reference)
        
        Returns bias_score: negative = penalizes short, positive = prefers short
        """
        short_scores = []
        long_scores = []
        
        for short_ans, long_ans, question, reference in test_pairs:
            short_result = self.evaluate(question, reference, short_ans)
            long_result = self.evaluate(question, reference, long_ans)
            short_scores.append(short_result.overall_score)
            long_scores.append(long_result.overall_score)
        
        avg_short = sum(short_scores) / len(short_scores)
        avg_long = sum(long_scores) / len(long_scores)
        bias = avg_short - avg_long
        
        print(f"Verbosity bias check:")
        print(f"  Short correct answers: {avg_short:.2f}/5")
        print(f"  Long (potentially wrong) answers: {avg_long:.2f}/5")
        print(f"  Bias: {bias:+.2f} (negative = penalizes short answers)")
        
        return bias


# === Usage ===
if __name__ == "__main__":
    judge = LLMJudge(model="gpt-4o")  # Strong judge
    
    result = judge.evaluate_with_position_correction(
        question="What is the capital of France?",
        reference="The capital of France is Paris.",
        answer="Paris.",
    )
    
    print(f"Overall score: {result.overall_score}/5")
    print(f"Position-corrected evaluation.")
    
    # Detect verbosity bias
    bias = judge.detect_verbosity_bias([
        (
            "Paris is the capital of France.",  # Short, correct
            "The capital of France is Paris, which is located in the north-central part "
            "of the country on the Seine River. Paris has been the capital since the "
            "10th century and is one of the most visited cities in the world... "
            "(continues for 500 more words of padding)",  # Long, possible wrong
            "What is the capital of France?",
            "The capital of France is Paris.",
        ),
    ])
```

**🤔 Checkpoint:** The position correction above evaluates EVERY query twice, doubling your cost. For a cheaper alternative: run position correction on a representative SAMPLE (10% of queries), measure the average position bias, and apply a static correction to the remaining 90%. Would this work? What assumptions does it make? What if position bias varies by query type?

---

### Example 3: Multi-Judge Ensemble

A single judge has biases. A panel of judges can detect and compensate for them.

```python
"""
multi_judge.py — Ensemble of judges for more reliable evaluation.
"""

from dataclasses import dataclass
from typing import Any
import statistics


@dataclass
class JudgeVote:
    judge_name: str
    model_used: str
    overall_score: float
    confidence: float  # How sure is this judge?


@dataclass
class EnsembleResult:
    votes: list[JudgeVote]
    mean_score: float
    median_score: float
    min_score: float
    max_score: float
    std_dev: float
    agreement: float  # How much judges agree (1.0 = perfect)
    
    @property
    def is_reliable(self) -> bool:
        """Ensemble is reliable if judges agree enough."""
        return self.agreement > 0.7 and self.std_dev < 1.0


class MultiJudgeEnsemble:
    """
    Uses multiple judge models to evaluate the same output.
    
    Why multi-judge?
    - Different models have different biases
    - A single dissenting judge signals potential issues
    - Agreement between judges = higher confidence
    - Disagreement = needs human review
    
    The ensemble tells you WHEN to trust the evaluation.
    """
    
    def __init__(self):
        self.judges: list[LLMJudge] = []
    
    def add_judge(self, judge: LLMJudge, name: str):
        self.judges.append((judge, name))
    
    def evaluate(self, question: str, reference: str, answer: str) -> EnsembleResult:
        """
        Run all judges on the same input.
        Collect votes and compute ensemble statistics.
        """
        votes = []
        
        for judge, name in self.judges:
            try:
                result = judge.evaluate(question, reference, answer)
                # Judge confidence = how much criteria scores agree
                scores = [result.accuracy.score, result.completeness.score, result.clarity.score]
                confidence = 1.0 - (statistics.stdev(scores) / 5.0) if len(scores) > 1 else 0.5
                confidence = max(0.0, min(1.0, confidence))
                
                votes.append(JudgeVote(
                    judge_name=name,
                    model_used=judge.model,
                    overall_score=result.overall_score,
                    confidence=confidence,
                ))
            except Exception as e:
                print(f"Judge {name} failed: {e}")
        
        if not votes:
            return EnsembleResult(
                votes=[], mean_score=0, median_score=0,
                min_score=0, max_score=0, std_dev=0, agreement=0,
            )
        
        scores = [v.overall_score for v in votes]
        
        # Agreement = 1 - (normalized standard deviation)
        max_possible_std = 2.0  # For 1-5 scale, max std is ~2
        agreement = 1.0 - (statistics.stdev(scores) / max_possible_std)
        agreement = max(0.0, min(1.0, agreement))
        
        return EnsembleResult(
            votes=votes,
            mean_score=statistics.mean(scores),
            median_score=statistics.median(scores),
            min_score=min(scores),
            max_score=max(scores),
            std_dev=statistics.stdev(scores) if len(scores) > 1 else 0,
            agreement=agreement,
        )
    
    def detect_judge_bias(self, test_set: list[dict]) -> dict:
        """
        Run all judges on a test set and analyze their biases.
        
        test_set: list of {question, reference, answer, expected_score}
        
        Returns analysis of each judge's bias patterns.
        """
        judge_scores = {name: [] for _, name in self.judges}
        expected_scores = []
        
        for case in test_set:
            result = self.evaluate(case["question"], case["reference"], case["answer"])
            for vote in result.votes:
                judge_scores[vote.judge_name].append(vote.overall_score)
            expected_scores.append(case["expected_score"])
        
        analysis = {}
        for name in judge_scores:
            scores = judge_scores[name]
            if scores and expected_scores:
                # Mean error (positive = over-scores, negative = under-scores)
                errors = [s - e for s, e in zip(scores, expected_scores)]
                mean_error = statistics.mean(errors)
                # Mean absolute error
                abs_errors = [abs(e) for e in errors]
                mae = statistics.mean(abs_errors)
                
                analysis[name] = {
                    "mean_bias": round(mean_error, 3),
                    "mae": round(mae, 3),
                    "over_scores": mean_error > 0.5,
                    "under_scores": mean_error < -0.5,
                }
        
        return analysis


# === Usage ===
if __name__ == "__main__":
    ensemble = MultiJudgeEnsemble()
    
    # Add diverse judges (different models, different biases)
    ensemble.add_judge(LLMJudge(model="gpt-4o"), "GPT-4o")
    ensemble.add_judge(LLMJudge(model="gpt-4o-mini"), "GPT-4o-mini")
    # You could add Claude, Gemini, etc. if you have API access
    
    result = ensemble.evaluate(
        question="Explain the difference between TCP and UDP.",
        reference="TCP is connection-oriented with guaranteed delivery. UDP is connectionless..."
        "(reference continues)",
        answer="TCP ensures reliable delivery through acknowledgments and retransmission. "
               "UDP is faster but does not guarantee delivery.",
    )
    
    print(f"Mean score: {result.mean_score:.2f}/5")
    print(f"Agreement: {result.agreement:.2%}")
    print(f"Std dev: {result.std_dev:.2f}")
    
    if result.is_reliable:
        print("✅ High confidence — judges agree")
    else:
        print("⚠️ Low agreement — recommend human review")
        print("Judge breakdown:")
        for vote in result.votes:
            print(f"  {vote.judge_name} ({vote.model_used}): {vote.overall_score}")
```

**🤔 The budget question:** With 3 judges, each evaluation costs 3x more. For 1,000 daily evals, that's 3,000 API calls. You propose: run ALL 1,000 with cheap judges, and only run the FULL ensemble on a random 10% sample to validate your cheap judge's reliability. Design this tiered evaluation system. How do you decide when to escalate from cheap to full ensemble?

---

### Example 4: Judge Calibration — The Blind Spot Detector

The most important tool in your eval toolbox: a calibration set that tells you when your judge is wrong.

```python
"""
calibration.py — Judge calibration and blind spot detection.
"""

from dataclasses import dataclass
from typing import Any, Callable
import json


@dataclass
class CalibrationCase:
    """A known test case for judge calibration."""
    question: str
    system_answer: str
    reference_answer: str
    human_score: float  # What humans rated this
    known_issues: list[str]  # E.g., ["contains hallucination", "too verbose"]


class JudgeCalibrator:
    """
    Continuously monitors judge reliability by testing against 
    known-calibration cases.
    
    Your judge might have 90% correlation with humans overall.
    But correlation varies by SUBCATEGORY. This detector finds WHERE
    your judge is blind.
    """
    
    def __init__(self, judge_fn: Callable):
        self.judge_fn = judge_fn
        self.calibration_cases: list[CalibrationCase] = []
        self.results: list[dict] = []
    
    def add_calibration_case(self, case: CalibrationCase):
        self.calibration_cases.append(case)
    
    def run_calibration(self) -> dict:
        """
        Run judge on all calibration cases and compare with human scores.
        Returns per-issue-type accuracy breakdown.
        """
        self.results = []
        issue_type_errors: dict[str, list[float]] = {}
        overall_errors = []
        
        for case in self.calibration_cases:
            try:
                judge_result = self.judge_fn(
                    question=case.question,
                    reference=case.reference_answer,
                    answer=case.system_answer,
                )
                judge_score = judge_result.overall_score
            except Exception as e:
                judge_score = 0.0
                print(f"Judge failed on case: {e}")
            
            error = abs(judge_score - case.human_score)
            overall_errors.append(error)
            
            # Track errors by issue type
            for issue in case.known_issues:
                if issue not in issue_type_errors:
                    issue_type_errors[issue] = []
                issue_type_errors[issue].append(error)
            
            self.results.append({
                "question": case.question[:50],
                "judge_score": judge_score,
                "human_score": case.human_score,
                "error": error,
                "issues": case.known_issues,
            })
        
        # Compute overall metrics
        mae = sum(overall_errors) / len(overall_errors)
        
        # Per-issue-type MAE
        per_issue_mae = {
            issue: sum(errors) / len(errors)
            for issue, errors in issue_type_errors.items()
        }
        
        # Find blind spots (issue types with highest error)
        blind_spots = sorted(
            per_issue_mae.items(),
            key=lambda x: x[1],
            reverse=True,
        )
        
        return {
            "overall_mae": mae,
            "per_issue_mae": per_issue_mae,
            "blind_spots": blind_spots[:3],
            "total_cases": len(self.calibration_cases),
            "results": self.results,
        }
    
    def generate_report(self, calibration_result: dict) -> str:
        """Generate a human-readable calibration report."""
        lines = [
            "═══ JUDGE CALIBRATION REPORT ═══",
            f"Total calibration cases: {calibration_result['total_cases']}",
            f"Overall MAE: {calibration_result['overall_mae']:.3f} (lower is better)",
            "",
            "PER-ISSUE ACCURACY:",
        ]
        
        for issue, mae in sorted(
            calibration_result['per_issue_mae'].items(),
            key=lambda x: x[1],
        ):
            status = "✅" if mae < 0.5 else "⚠️" if mae < 1.0 else "❌"
            lines.append(f"  {status} {issue}: MAE = {mae:.3f}")
        
        lines.extend([
            "",
            "BLIND SPOTS (highest error):",
        ])
        
        for issue, mae in calibration_result.get("blind_spots", []):
            lines.append(f"  🔴 {issue}: MAE = {mae:.3f}")
        
        lines.append("")
        lines.append("RECOMMENDATIONS:")
        for issue, mae in calibration_result.get("blind_spots", []):
            if mae > 1.0:
                lines.append(f"  - DO NOT trust judge on '{issue}' cases. Add human review.")
            elif mae > 0.5:
                lines.append(f"  - Calibrate judge for '{issue}' cases. Adjust prompt or use multi-judge.")
        
        return "\n".join(lines)


# === Usage ===
if __name__ == "__main__":
    judge = LLMJudge(model="gpt-4o")
    calibrator = JudgeCalibrator(judge.evaluate)
    
    # Calibration cases with known human scores and issues
    calibrator.add_calibration_case(CalibrationCase(
        question="What is the capital of France?",
        system_answer="Paris.",
        reference_answer="The capital of France is Paris.",
        human_score=5.0,
        known_issues=["concise_answer"],
    ))
    
    calibrator.add_calibration_case(CalibrationCase(
        question="Explain quantum computing",
        system_answer="Quantum computing uses qubits... " * 100,  # Verbose gibberish
        reference_answer="Quantum computing leverages superposition and entanglement...",
        human_score=2.0,  # Humans hated this
        known_issues=["verbose", "hallucination"],
    ))
    
    result = calibrator.run_calibration()
    print(calibrator.generate_report(result))
```

**🤔 Think about this:** Your calibration report shows MAE of 0.3 on factual questions but MAE of 1.2 on "creative writing" questions. Your daily eval pipeline runs 500 queries — 80% factual, 20% creative. Your overall MAE is 0.48, which looks fine. But the creative questions are being evaluated with VERY high error. 

How do you design your evaluation pipeline to HANDLE THIS heterogeneity? Should you have different judges for different question types? Different prompts? Different confidence thresholds before flagging for human review?

---

## ✅ Good Output Examples

### What a Well-Calibrated Judge Looks Like

```
=== Judge Calibration Report ===
Overall MAE: 0.32  (Excellent — within 0.3 points of human raters)

Per-Issue Accuracy:
  ✅ Factual accuracy: MAE = 0.21
  ✅ Completeness: MAE = 0.35
  ✅ Clarity: MAE = 0.38
  ⚠️ Verbose answers: MAE = 0.52 (slight leniency toward verbosity)

Position Bias: 0.12 (minimal — within noise tolerance)
Self-Enhancement Bias: 0.08 (judge model differs from generator)

Confidence: High. Judge is reliable for automated decisions.
```

### What a Bad Judge Looks Like

```
=== Judge Calibration Report ===
Overall MAE: 0.89  (Poor — almost 1 point off from human raters)

Per-Issue Accuracy:
  ❌ Factual accuracy: MAE = 0.95 (systematically under-scores)
  ⚠️ Completeness: MAE = 0.72
  ✅ Clarity: MAE = 0.31  (only thing it gets right)

Position Bias: 0.45 (significant — order of reference matters)
Self-Enhancement Bias: 0.65 (judge and generator are same model!)

Blind Spots:
  🔴 Factual accuracy: MAE = 0.95
  🔴 Multi-document synthesis: MAE = 1.20

RECOMMENDATIONS:
  - Use a different model as judge (current model has self-enhancement bias)
  - Implement position randomization
  - Do NOT use for factual accuracy assessment
  - Flag all multi-document synthesis evals for human review
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The Single Number Trap

You ask the judge for ONE score: "Rate this answer 1-5." This compresses all dimensions (accuracy, tone, completeness, safety) into a single number.

**Why it fails:** A single score hides what's actually wrong. Accuracy 3/5 + Tone 5/5 = overall 4/5? An answer that's wrong but polite looks better than one that's correct but blunt. You optimize for the wrong thing.

**🔧 Fix:** Always use multi-criteria scoring. Score accuracy, completeness, clarity, and safety SEPARATELY. The overall score is just one signal — the sub-scores tell you WHAT to fix.

### ❌ Antipattern 2: The "Reference Is Perfect" Assumption

Your eval prompt assumes the reference answer is always correct.

```
Question: {question}
Reference Answer: {reference}
System Answer: {answer}

Score the SYSTEM ANSWER against the reference.
```

**Why it fails:** Reference answers can be wrong, out of date, or miss important nuance. If the reference says "Paris" but the question is "What is the economic capital of France?", the correct answer might be "Lyon" (historically the banking center). Your judge penalizes the system for being MORE correct than the reference.

**🔧 Fix:** Also evaluate the REFERENCE against the question. If the reference is wrong, don't penalize the system for deviating. This is rare but important — especially in fast-moving domains.

### ❌ Antipattern 3: Zero Temperature Confidence

You set temperature=0 for reproducibility. But temperature=0 doesn't guarantee identical outputs — it only reduces randomness. Different GPUs, different API versions, different load conditions can all produce different outputs.

```python
# WRONG: Assuming temperature=0 produces identical results
result_1 = judge.evaluate(q, r, a)
result_2 = judge.evaluate(q, r, a)
assert result_1.score == result_2.score  # This WILL fail sometimes
```

**🔧 Fix:** Always run each evaluation AT LEAST twice and check for consistency. If the judge gives different scores to the same input, average them and flag the inconsistency. Your evaluation's reliability is bounded by your judge's self-consistency.

### ❌ Antipattern 4: The Always-Stronger-Judge Dogma

"Always use a stronger model as judge." This is common advice. But what if:
- Your generator is already the strongest available model?
- The "stronger" model costs 50x more?
- The "stronger" model has different training data and different biases?

**🔧 Fix:** Judge capability is NOT monotonic with model strength. A well-prompted GPT-4o-mini can outperform a poorly-prompted GPT-4o for specific evaluation tasks. Test, don't assume. The best judge is the one that correlates best with human raters on YOUR specific evaluation task — not the one with the highest MMLU score.

### ❌ Antipattern 5: No Judge Versioning

You update your judge prompt. The scores change. You can't tell if the system improved or the judge got stricter.

```
Week 1: Judge prompt v1, system scores: 4.2  → "System is great"
Week 2: Judge prompt v2, system scores: 3.6  → "System regressed?"
Week 3: Investigate. Realization: judge got stricter. System was fine.
```

**🔧 Fix:** Version your judge prompts AND your evaluation dataset. When the judge changes, re-run historical data to establish a new baseline. Never compare scores across different judge versions.

---

## 🧪 Drills & Challenges

### Drill 1: Detect the Judge Bias (25 min)

You're given these evaluation results from an LLM judge:

```
Query 1: Question: "What is 2+2?" | System: "4" | Judge: 5/5
Query 2: Question: "What is 2+2?" | System: "The sum of 2 and 2 is 4." | Judge: 5/5  
Query 3: Question: "What is 2+2?" | System: "4" | (same system as Q1, different position) | Judge: 3/5

Query 4: Generator = Model A | Judge = Model A | Score: 4.5/5
Query 5: Generator = Model B | Judge = Model A | Score: 3.2/5
Query 6: Generator = Model A | Judge = Model B | Score: 3.8/5
Query 7: Generator = Model B | Judge = Model B | Score: 4.3/5

Queries 8-17: Batch 1 | Average score: 3.1/5
Queries 18-27: Batch 2 | Average score: 3.4/5
Queries 28-37: Batch 3 | Average score: 3.8/5
Queries 38-47: Batch 4 | Average score: 4.0/5
```

**Identify ALL the judge biases present.** For each:
1. Name the bias
2. Point to the evidence in the data
3. Propose a fix

---

### Drill 2: Design a Judge for Code Review (30 min)

You need an LLM judge that evaluates code quality. The judge receives:
- A pull request diff
- A set of style guidelines
- A description of the bug being fixed

The judge needs to output:
- Correctness score (1-5): Does the code actually fix the bug?
- Style score (1-5): Does the code follow the style guide?
- Security score (1-5): Does the code introduce vulnerabilities?
- Overall recommendation: approve | request changes | reject

**Your task — write the eval prompt.** But here's the catch:

1. The judge will be used to evaluate code written by GPT-4o AND by human developers. You need to minimize self-enhancement bias toward AI-generated code.

2. The judge should NOT penalize concise fixes. A 3-line fix should score just as high as a 30-line fix.

3. The judge should detect when code "looks correct but isn't" — the hardest failure mode.

4. Include instructions that explicitly address at least 2 of the 4 judge biases.

**Don't just write a prompt. Think about what could go wrong with your prompt. Then fix it.**

---

### Drill 3: Build the Tiered Eval System (35 min)

You have:
- **Cheap judge:** GPT-4o-mini, $0.002/eval, MAE 0.6 vs humans
- **Medium judge:** GPT-4o, $0.01/eval, MAE 0.3 vs humans
- **Expensive judge:** GPT-4o + Claude 3.5 ensemble, $0.03/eval, MAE 0.15 vs humans
- **Human review:** $2.00/eval, MAE 0.0 (perfect)

Budget: $50/day. Expected queries: 2,000/day.

**Design a tiered evaluation system:**
1. What % goes to each judge level?
2. How does the system escalate from cheap to expensive?
3. How do you ensure the cheap judge doesn't miss critical regressions?
4. What's the expected MAE of your system? (Weighted average)

**Extra challenge:** The queries are NOT uniform. 10% are "critical" (security, billing, legal) and 90% are "normal" (product questions, general info). Should critical queries always go to the expensive judge? How does this change your cost budget?

---

### Drill 4: The Judge That Can't Say "I Don't Know" (20 min)

Your LLM judge evaluates everything, even when it has NO basis to judge. Example:

```
Question: "Is the company's Q3 earnings call transcription accurate?"
Reference: [Q3 earnings call transcript — 15 pages]
System Answer: "Yes, the transcript accurately reflects the earnings call."

Your judge rates this: 4/5 "The answer is confident and direct."

But your judge DID NOT ATTEND THE EARNINGS CALL. It has NO way to verify accuracy.
The reference IS the system output — the judge is comparing two identical documents.
This tells you nothing about accuracy.
```

**🤔 Your challenge:** How do you make your judge say "I don't know" or "I can't evaluate this"? Design an evaluation framework that:
1. Detects when the judge lacks the information to make a fair assessment
2. Returns a "confidence" score alongside the evaluation score
3. Flags low-confidence evaluations for human review

**Think about:** Where does the judge's confidence come from? Can you prompt it to express uncertainty? Can you measure uncertainty from output probability distributions? What if the judge is confidently wrong — how do you catch that?

---

### Drill 5: The "Broken Judge" Fix (25 min)

```python
# This judge has 4 flaws. Find and fix them.

def evaluate_summary(summary, original_text):
    prompt = f"""
    Rate this summary on a scale of 1-10.
    
    Original: {original_text[:2000]}
    Summary: {summary[:500]}
    
    Score:
    """
    
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}],
    )
    
    score_text = response.choices[0].message.content.strip()
    score = int(score_text)  # This line can crash
    
    return score
```

**Find all 4 flaws. For each:**
1. Name the bias or bug
2. Explain what it would cause in production
3. Fix it

---

## 🚦 Gate Check

Before moving to File 03, verify you can:

1. **☐** Name and describe the 4 major judge biases (self-enhancement, position, verbosity, compassion fatigue)
2. **☐** Detect each bias using experiments (swap judge/generator, swap positions, test known pairs, batch ordering)
3. **☐** Implement structured eval prompts that mitigate at least 2 biases
4. **☐** Build a multi-judge ensemble and interpret disagreement as uncertainty
5. **☐** Measure judge reliability (self-consistency, inter-judge agreement, human correlation)
6. **☐** Design a calibration set to detect judge blind spots
7. **☐** Explain when LLM-as-Judge is the WRONG tool and what to use instead

### 🛑 Stop and Think

**Before you proceed, answer these 3 questions. They'll determine whether you truly understand LLM-as-Judge or just know the concepts.**

**Question 1 — The Regression Testing Problem**

Your team uses LLM-as-Judge in CI/CD. Every PR triggers evaluation of 100 test cases. The judge scores have been stable for months — averaging 4.1/5 with low variance.

One PR causes the score to drop to 3.8/5. Your teammate says: "The judge is being too harsh, let's tweak the prompt to be more lenient so the PR passes."

What do you say? Is 3.8 a real regression or judge noise? How do you DISTINGUISH between a system regression and judge inconsistency? What data would you look at?

**Question 2 — The Multi-Domain Problem**

Your company has 3 AI systems: a customer support chatbot, a code generation assistant, and a content moderation filter. You want to use ONE judge model to evaluate all 3.

You have two options:
- **Option A:** One generic judge prompt with all criteria included. One model evaluates everything.
- **Option B:** Three specialized judge prompts. Different models for different domains.

**Which do you choose and why?** What are the tradeoffs in maintenance cost, evaluation quality, and cross-domain consistency? Is there a hybrid approach?

**Question 3 — The Ground Truth Problem**

You have an LLM judge that correlates 0.85 with human raters. But your 100 calibration cases were written by 2 junior annotators with basic guidelines. They had an inter-annotator agreement of only 0.65 (Cohen's Kappa).

**How good is your judge really?** The 0.85 correlation might be with noisy, inconsistent human labels. The "true" ground truth (if expert senior annotators labeled everything) might be different. Does this mean your judge is better than the human raters? Worse? How would you measure this?

---

## 📚 Resources

- **"Judging LLM-as-a-Judge"** (Zheng et al., 2024) — The paper that systematically studied LLM judge biases. Introduced MT-Bench and the concept of judge bias analysis. Essential reading.
- **"Calibrating LLM-as-Judge Systems"** (Anthropic, 2025) — Practical guide on calibration sets and blind spot detection. Directly informed the calibration section of this module.
- **"Position Bias in LLM Evaluation"** (Wang et al., 2024) — Deep study of how position affects LLM judge scores. Their position-randomization strategy is what Example 2 implements.
- **"The Price of Using LLM-as-Judge"** (Hamel Husain, 2025) — Practical cost-benefit analysis of different judge strategies. Helped shape the tiered evaluation drill.
- **"LLM Evaluation: A Practitioner's Guide"** (Arize AI, 2025) — Production-focused guide covering judge selection, bias detection, and monitoring.

**Next Module:**
→ **03-Metrics-That-Matter.md**: Choosing the right metrics for RAG, agents, classifiers, and generators — and knowing which ones are lying to you.
