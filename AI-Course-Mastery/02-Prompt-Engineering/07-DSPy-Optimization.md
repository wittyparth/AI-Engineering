# DSPy — Programmatic Prompt Optimization

## 🎯 Purpose & Goals

> **🛑 STOP. Three scenarios. Think before scrolling.**

---

### 🤔 Scenario 1: The Manual Tuning Trap

You have a prompt for classifying customer emails. You've been manually tweaking it for 3 weeks:

```
Attempt 12: Added "Be very careful with edge cases" → No improvement
Attempt 13: Changed "classify as" to "categorize as" → Worse
Attempt 14: Added 3 more examples → +2% accuracy
Attempt 15: Reordered examples → -1% accuracy
Attempt 16: Changed "You are a classifier" to "You are a precise classification system" → +1%
Attempt 17: Added "NEVER explain your reasoning" → +3% accuracy!
Attempt 18: Removed it → Back to baseline. Never mind.
Attempt 19: Changed temperature from 0 to 0.1 → +0.5%
```

**You've spent 3 weeks for +4% improvement. You have no idea which changes actually caused what. Some improvements were random noise.**

**Questions:**
1. What's the cost of 3 weeks of manual prompt tuning? (Salary? Opportunity cost? Deferred features?)
2. How many of your "improvements" were real vs. random variance in the model's output? How would you know?
3. If you could have achieved +8% in 1 day with an automated optimizer, would 3 weeks of manual work be a waste? Or did you learn something valuable?
4. **Research task:** Look up what companies like OpenAI, Anthropic, and Cohere use internally for prompt optimization. Do their best teams tune prompts by hand or use automated systems?

---

### 🤔 Scenario 2: The DSPy Black Box

Your teammate says: "Just use DSPy. It automatically optimizes your prompts. You don't need to write prompts anymore."

You try it:
```python
# 5 lines of code
class MyTask(dspy.Signature):
    """Classify emails"""
    email = dspy.InputField()
    category = dspy.OutputField()

classifier = dspy.ChainOfThought(MyTask)
optimized = dspy.BootstrapFewShot(metric=accuracy).compile(classifier, trainset=trainset)
```

The optimized prompt is 47 lines long with 12 examples, 3 persona shifts, and nested XML tags. Accuracy is 92% vs your manual 87%. Great!

**Then your CEO asks: "Why does the prompt say the model is a 'senior email architect' when we're a restaurant?"**

**Questions:**
1. DSPy found a prompt that works better than yours. But why does it work? Can you explain it to a non-technical stakeholder?
2. If the automated prompt fails on a customer email, how would you debug it? Where would you even start?
3. How do you audit a machine-generated prompt for safety before deploying?
4. When would you REJECT a DSPy-optimized prompt even if it scores higher on accuracy? What non-accuracy criteria matter?

---

### 🤔 Scenario 3: Over-Optimization

DSPy optimizes your prompt on your training set of 500 examples. It achieves 98% accuracy on the training set, 93% on your held-out test set. You deploy.

**Week 1:** Works great. 
**Week 2:** A new type of email arrives that wasn't in the training data. Your old hand-written prompt handles it fine. The DSPy-optimized prompt fails completely.

**The problem:** DSPy found statistical patterns in your training data that don't generalize. It's overfit.

**Questions:**
1. The DSPy prompt is 47 lines with 12 specific examples. Your hand-written prompt is 5 lines with general rules. Which one generalizes better to unseen data? Why?
2. How would you measure overfitting in a prompt optimizer? What's the training/validation/test split strategy?
3. Design a hybrid approach: use DSPy for initial optimization, then manually simplify the result. How would you know when to stop simplifying?
4. **Research task:** Look up "prompt overfitting" and "dataset diversity for prompt optimization." What happens if you optimize on a dataset where 80% of emails are "spam" and only 20% are "customer inquiry"?

---

**By the end of this file, you will:**
- Understand what DSPy actually does (compilers, teleprompters, signatures)
- Know when to use DSPy vs. manual prompting
- Build DSPy pipelines that don't overfit
- Interpret and validate DSPy-optimized prompts
- Integrate DSPy with your evaluation framework

**⏱ Time Budget:** 2.5 hours

---

## 📖 What DSPy Actually Does

```
DSPy IS NOT a "prompt engineering tool." It's a PROGRAM COMPILER.

┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Your Program    │     │  DSPy Compiler   │     │  Optimized       │
│  (signatures,    │────→│  (teleprompter)  │────→│  Program         │
│   modules)       │     │                   │     │  (prompts +      │
│                   │     │  Optimizes:       │     │   examples +    │
│                   │     │  - Instructions   │     │   structure)    │
│                   │     │  - Few-shot ex's  │     │                  │
│                   │     │  - Module choices  │     │                  │
│                   │     │  - Pipeline order  │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘

THE KEY INSIGHT:
You don't write prompts. You write a PROGRAM with typed inputs and outputs
(Signatures). DSPy figures out the optimal prompt to run that program.

This is the SAME shift as:
- Assembly → High-level languages (you describe WHAT, compiler figures out HOW)
- Manual SQL → ORM (you describe data, system optimizes queries)
- Manual prompts → DSPy (you describe task, system optimizes prompts)
```

### DSPy Architecture

```
DSPy MODULES — the building blocks:

┌─────────────────────────────────────────────────────────────┐
│ dspy.Predict                                               │
│   Basic LLM call. Input → LLM → Output.                    │
│   "Translate X to Y"                                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ dspy.ChainOfThought                                        │
│   Predict + "Let's think step by step"                     │
│   Generates reasoning before answer.                       │
│   Use for: reasoning, math, analysis, planning             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ dspy.ChainOfThoughtWithHint                                │
│   ChainOfThought + a hint from the program.                │
│   Use for: tasks where you can provide guidance.           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ dspy.ReAct                                                 │
│   Reasoning + Acting (tool use).                           │
│   Model thinks, chooses tools, observes results.           │
│   Use for: agents, multi-step tasks with tools.            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ dspy.MultiChainComparison                                  │
│   Generate multiple reasoning paths, compare, pick best.   │
│   Similar to self-consistency but done in one pass.        │
│   Use for: high-stakes reasoning, verification.            │
└─────────────────────────────────────────────────────────────┘
```

```
DSPY TELEPROMPTERS — the optimization algorithms:

┌─────────────────────────────────────────────────────────────┐
│ BootstrapFewShot                                           │
│   Uses training examples to generate few-shot examples.    │
│   Best for: Small datasets, quick optimization             │
│   Cost: ~1-5x training set size in LLM calls              │
│   Type: Few-shot example selection / generation            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ BootstrapFewShotWithRandomSearch                           │
│   BootstrapFewShot + random search over prompt variations. │
│   Best for: Medium complexity, some budget for search      │
│   Cost: ~10-50x training set size                          │
│   Type: Example selection + prompt instruction search      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ MIPROv2 (Bayesian Optimization)                            │
│   Bayesian optimization over instruction + example space.  │
│   Best for: Complex tasks, production quality              │
│   Cost: ~50-200x training set size                         │
│   Type: Full prompt optimization (instructions + examples) │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ BootstrapFinetune                                          │
│   Generate training data via DSPy, fine-tune a small model.│
│   Best for: Production deployment, latency-sensitive       │
│   Cost: High (fine-tuning), low per-inference              │
│   Type: Optimizes model weights, not prompts               │
└─────────────────────────────────────────────────────────────┘
```

---

## 💻 DSPy in Practice

### Installation & Setup

```python
import dspy
from dspy.teleprompt import BootstrapFewShot, MIPROv2

# Configure DSPy with your LLM of choice
lm = dspy.OpenAI(
    model="gpt-4o-mini",
    api_key=api_key,  # Or set OPENAI_API_KEY env var
    temperature=0.3,
    max_tokens=1000,
)
dspy.settings.configure(lm=lm)
```

### Step 1: Define Signatures (Your Task Contract)

```python
"""
Signatures define the INPUT → OUTPUT contract.
They are NOT prompts — they're type signatures.

DSPy generates prompts FROM these signatures.
"""

class EmailClassification(dspy.Signature):
    """Classify customer emails for routing.
    
    Categories: billing, technical_support, account_management,
                feature_request, complaint, other
    """
    email_text = dspy.InputField(desc="The full customer email content")
    category = dspy.OutputField(desc="One of: billing, technical_support, account_management, feature_request, complaint, other")
    priority = dspy.OutputField(desc="Priority level: low, medium, high, urgent")
    confidence = dspy.OutputField(desc="Confidence score 0.0-1.0")


class EmailSummary(dspy.Signature):
    """Summarize customer email for internal team."""
    email_text = dspy.InputField(desc="Full email to summarize")
    summary = dspy.OutputField(desc="1-2 sentence summary")
    key_issues = dspy.OutputField(desc="Key issues mentioned")
    sentiment = dspy.OutputField(desc="Customer sentiment: positive, neutral, negative, frustrated")


class EmailPipeline(dspy.Module):
    """
    Multi-step email processing pipeline.
    
    DSPy optimizes the ENTIRE pipeline, not individual steps.
    It may discover that a different sequence of steps works better.
    """
    
    def __init__(self):
        self.classify = dspy.ChainOfThought(EmailClassification)
        self.summarize = dspy.ChainOfThought(EmailSummary)
    
    def forward(self, email_text: str) -> dict:
        classification = self.classify(email_text=email_text)
        summary = self.summarize(email_text=email_text)
        
        return dspy.Prediction(
            category=classification.category,
            priority=classification.priority,
            confidence=classification.confidence,
            summary=summary.summary,
            key_issues=summary.key_issues,
            sentiment=summary.sentiment,
        )
```

### Step 2: Define Metrics (How DSPy Optimizes)

```python
"""
The METRIC is the most important part of DSPy optimization.
It tells DSPy what "good" looks like.

A good metric:
1. Is differentiable (gives partial credit, not 0/1)
2. Aligns with actual business goals
3. Is fast to compute (called hundreds of times during optimization)
"""

def classification_metric(
    example: dspy.Example,
    prediction: dspy.Prediction,
    trace=None,
) -> float:
    """
    Score a classification prediction.
    
    Returns 0.0 to 1.0 score.
    
    🔴 SENIOR MOVE: Don't use 0/1 accuracy.
    Use graded scoring that gives PARTIAL CREDIT.
    This helps the optimizer learn.
    """
    score = 0.0
    
    # Category match (0.6 weight)
    if prediction.category == example.category:
        score += 0.6
    elif prediction.category.lower() in example.category.lower() or \
         example.category.lower() in prediction.category.lower():
        score += 0.3  # Partial credit for close match
    
    # Priority match (0.2 weight)
    if prediction.priority == example.priority:
        score += 0.2
    
    # Confidence (0.1 weight)
    try:
        conf = float(prediction.confidence)
        # Penalize low confidence on correct answers and high confidence on wrong ones
        if prediction.category == example.category:
            score += 0.1 * conf
        else:
            score += 0.1 * (1 - conf)
    except (ValueError, TypeError):
        score += 0.05  # Partial for bad format
    
    return min(score, 1.0)


def comprehensive_metric(
    example: dspy.Example,
    prediction: dspy.Prediction,
    trace=None,
) -> float:
    """
    Full pipeline metric.
    
    Scores: classification accuracy + summary quality.
    
    This is called during optimization to evaluate the pipeline.
    DSPy will try to MAXIMIZE this score.
    """
    # Classification part (0-2 points)
    cat_score = 0.0
    if prediction.category == example.category:
        cat_score = 1.0
    elif isinstance(prediction.category, str) and \
         any(c in prediction.category.lower() for c in example.category.lower().split('_')):
        cat_score = 0.5
    
    # Priority part (0-1 point)
    pri_score = 1.0 if prediction.priority == example.priority else 0.0
    
    # Sentiment part (0-1 point)
    sent_score = 1.0 if prediction.sentiment == example.sentiment else 0.0
    
    # Summary part (rough: not-empty check)
    summary_score = 0.5 if len(prediction.summary or '') > 20 else 0.0
    if len(prediction.key_issues or '').split('\n') >= 2:
        summary_score += 0.5
    
    total = cat_score + pri_score + sent_score + summary_score
    return total / 4.0  # Normalize to 0-1
```

### Step 3: Optimize with DSPy

```python
# ============================================================ #
# METHOD 1: BootstrapFewShot (fast, good for small datasets)   #
# ============================================================ #

# Create training data
trainset = [
    dspy.Example(
        email_text="I was charged twice for my subscription. Please fix this immediately!",
        category="billing",
        priority="urgent",
        confidence=0.95,
        summary="Customer reports double charge on subscription",
        key_issues="- Double charge\n- Requesting immediate fix",
        sentiment="frustrated",
    ).with_inputs("email_text"),
    # ... add 50-200 more examples
]

# Configure optimizer
config = {
    "max_bootstrapped_demos": 4,  # Max examples per prompt
    "max_labeled_demos": 4,       # Max labeled examples per prompt
    "num_candidate_programs": 10, # Number of candidates to evaluate
}

optimizer = BootstrapFewShot(
    metric=classification_metric,
    max_bootstrapped_demos=config["max_bootstrapped_demos"],
    max_labeled_demos=config["max_labeled_demos"],
)

# Compile
pipeline = EmailPipeline()
optimized_pipeline = optimizer.compile(pipeline, trainset=trainset)

# Evaluate on test set
evaluation_results = []
for example in testset:
    prediction = optimized_pipeline(email_text=example.email_text)
    score = classification_metric(example, prediction)
    evaluation_results.append(score)

avg_score = sum(evaluation_results) / len(evaluation_results)
print(f"Optimized score: {avg_score:.3f}")

# Compare to baseline (unoptimized)
baseline_results = []
for example in testset:
    prediction = pipeline(email_text=example.email_text)
    score = classification_metric(example, prediction)
    baseline_results.append(score)

baseline_avg = sum(baseline_results) / len(baseline_results)
print(f"Baseline score:  {baseline_avg:.3f}")
print(f"Improvement:     {(avg_score - baseline_avg) * 100:.1f}%")
```

### Step 4: Inspect the Optimized Prompt

```python
"""
🔴 SENIOR MOVE: NEVER use a DSPy-optimized prompt blindly.
ALWAYS inspect and validate it first.

DSPy generates the prompt internally. You can inspect it
to understand WHAT changed.
"""

def inspect_optimized_program(optimized_pipeline):
    """
    Extract and display the optimized prompts from a DSPy program.
    
    This is how you audit what DSPy changed.
    """
    # DSPy stores prompts in the program's modules
    for name, module in optimized_pipeline.named_modules():
        if hasattr(module, 'dememoize'):
            # The dememoize'd version shows the actual prompt
            try:
                prompt_data = module.dememoize
                print(f"\n{'='*60}")
                print(f"MODULE: {name}")
                print(f"{'='*60}")
                
                # Show adapted prompt
                if hasattr(prompt_data, 'extended_signature'):
                    print(f"\nEXTENDED SIGNATURE:")
                    print(prompt_data.extended_signature)
                
                # Show few-shot examples (if any)
                if hasattr(prompt_data, 'demonstrations'):
                    print(f"\nFEW-SHOT EXAMPLES ({len(prompt_data.demonstrations)}):")
                    for i, demo in enumerate(prompt_data.demonstrations):
                        print(f"\n  Example {i+1}:")
                        for field, value in demo.items():
                            print(f"    {field}: {str(value)[:100]}")
                
                # Show full prompt
                if hasattr(module, '_cached_prompt'):
                    print(f"\nFULL PROMPT (first 1000 chars):")
                    print(module._cached_prompt[:1000])
            except Exception as e:
                print(f"  Couldn't inspect: {e}")


# Example output:
"""
============================================================
MODULE: classify
============================================================

EXTENDED SIGNATURE:
You are a precise classification system for customer emails.

Given an email, classify it into one of these categories:
- billing: Payment issues, invoices, charges
- technical_support: Bugs, errors, technical problems
- account_management: Password reset, profile changes
- feature_request: Suggestions, feature ideas
- complaint: Dissatisfaction, negative feedback
- other: Everything else

Also assign priority: low, medium, high, urgent

Think step by step:
1. Read the email carefully
2. Identify the primary issue
3. Select the matching category
4. Assess urgency based on language used
5. Rate your confidence

---

FEW-SHOT EXAMPLES (3):

  Example 1:
    email_text: I was charged twice for my subscription...
    reasoning: The customer mentions payment charges...
    category: billing
    priority: urgent
    confidence: 0.95

  Example 2:
    ...
"""
```

### Step 5: Advanced — MIPROv2 (Bayesian Optimization)

```python
"""
MIPROv2 is DSPy's most powerful optimizer.
It uses Bayesian optimization to search both instructions AND examples.

COST: 50-200x your dataset size in LLM calls.
For 100 examples, that's 5,000-20,000 LLM calls. 
Budget ~$10-50 for optimization.

WHEN TO USE:
- You have a large enough budget (>$10)
- You need maximum quality
- Your task is complex (multi-step, nuanced)
- You can afford 1-2 hours of optimization time

WHEN NOT TO USE:
- You have <50 training examples
- Your task is simple (classification with clear categories)
- You need to iterate quickly
"""

# ── MIPROv2 Optimization ──

def optimize_with_mipro(
    trainset: list,
    valset: list,
    pipeline: dspy.Module,
    metric: callable,
    num_candidates: int = 10,
    num_trials: int = 30,
) -> dspy.Module:
    """
    Run MIPROv2 optimization.
    
    This is the heavy artillery. Use it when BootstrapFewShot isn't enough.
    """
    optimizer = MIPROv2(
        metric=metric,
        num_candidates=num_candidates,  # Instructions to try per step
        num_trials=num_trials,           # Total optimization trials
        minibatch_size=25,               # Samples per evaluation
        # 🔴 SENIOR MOVE: Use a validation set for early stopping
        # This prevents overfitting to training data
    )
    
    # MIPROv2 generates candidate instructions AND selects examples
    optimized = optimizer.compile(
        pipeline,
        trainset=trainset,
        valset=valset,  # Separate validation set!
        max_bootstrapped_demos=3,
        max_labeled_demos=5,
    )
    
    return optimized


# ── Cost Management for MIPRO ──

def estimate_optimization_cost(
    num_train: int,
    num_trials: int,
    model: str = "gpt-4o-mini",
) -> dict:
    """
    Estimate the cost of MIPROv2 optimization.
    
    This helps you budget before running the optimizer.
    """
    # Each trial: evaluate on a minibatch
    calls_per_trial = 25  # minibatch size
    total_calls = num_trials * calls_per_trial
    
    # Each call: ~500 input + ~200 output tokens
    tokens_per_call = 700
    total_tokens = total_calls * tokens_per_call
    
    # Pricing
    pricing = {
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        "gpt-4o": {"input": 2.50, "output": 10.00},
    }
    
    rates = pricing.get(model, pricing["gpt-4o-mini"])
    input_tokens = total_tokens * 0.7  # ~70% input
    output_tokens = total_tokens * 0.3  # ~30% output
    
    cost = (input_tokens * rates["input"] + output_tokens * rates["output"]) / 1_000_000
    
    return {
        "num_calls": total_calls,
        "estimated_tokens": total_tokens,
        "estimated_cost_usd": round(cost, 2),
        "model": model,
        "recommendation": "Use gpt-4o-mini for optimization, then validate with gpt-4o" 
                         if cost > 20 else "Cost is reasonable, proceed",
    }
```

---

### 🤔 Application Question: The Metric Trap

You define a metric for DSPy optimization: "Match the expected category exactly." This is a 1/0 metric — either the category is right or it's not.

DSPy optimizes for 2 hours. The result: 95% accuracy on your training set (up from 87%). You deploy.

**Problem:** The optimized prompt now ALWAYS classifies emails as "billing" or "complaint" — the two most common categories. It never predicts "feature_request" or "account_management." Accuracy on the rare categories dropped to 0%.

Your 1/0 metric was HAPPY with this result: getting the common categories right (95% of cases) is "optimal" even when 100% of rare cases are wrong.

**Questions:**
1. Your metric is "category match accuracy" — is this a bad metric, or is your question distribution imbalanced? Would DSPy behave differently with a balanced dataset?
2. Design a metric that doesn't ignore rare categories. How would you weight them? What if some misclassifications are worse than others? (e.g., classifying "complaint" as "other" vs. "complaint" as "billing")
3. How would you detect this failure BEFORE deployment? What evaluation approach would catch it?
4. **Research task:** Look up "class imbalance in prompt optimization" and "cost-sensitive learning." How would you adapt these concepts for DSPy?

---

## 📊 DSPy Performance Benchmarks

```
Benchmark: Email Classification (500 examples)

Optimizer          | Accuracy | Instructions | Examples | Cost   | Time
-------------------|----------|--------------|----------|--------|------
Manual (hand-tuned)| 87.0%    | 5 lines      | 3        | Free   | 3 weeks
BootstrapFewShot   | 91.2%    | Auto         | 4        | $0.50  | 2 min
BFS + RandomSearch | 93.5%    | Auto (3 tries)| 4        | $2.00  | 5 min
MIPROv2 (30 trials)| 94.8%    | Auto (opt)   | 5        | $15.00 | 15 min
Ensemble (3 models)| 95.2%    | Auto          | N/A      | $45.00 | 30 min

KEY INSIGHT: MIPROv2 gets +3.6% over manual at $15 cost.
The 3-week manual effort was worth LESS than a $15 automated optimization.
```

```
DIMINISHING RETURNS:
Cost        | Accuracy Gain
------------|--------------
$0          | baseline (manual 87%)
$0.50       | +4.2% (BFS)
$2.00       | +2.3% on top of BFS
$15.00      | +1.3% on top of RandomSearch
$45.00      | +0.4% on top of MIPROv2

🔴 SENIOR MOVE: $2 gets you 6.5% improvement. $45 gets you 8.2%.
The last $40 buys only 1.7%. Know your diminishing returns.
```

---

## 💻 Full Production Example

```python
"""
Complete DSPy pipeline with evaluation, validation, and monitoring.
"""

import dspy
from dspy.teleprompt import BootstrapFewShot, MIPROv2
from typing import Optional
import json
from datetime import datetime


# ── Signatures ──

class SupportTicketAnalysis(dspy.Signature):
    """Analyze support tickets for routing and prioritization."""
    ticket_text = dspy.InputField(desc="Customer support ticket content")
    category = dspy.OutputField(desc="Category: billing, technical, account, feature_request, complaint, other")
    priority = dspy.OutputField(desc="Priority: low, medium, high, critical")
    required_action = dspy.OutputField(desc="Required action: refund, escalate, troubleshoot, update_account, inform, no_action")
    response_template = dspy.OutputField(desc="Brief response direction, not full response")

class TicketRouter(dspy.Module):
    """Route support tickets to the right team."""
    
    def __init__(self):
        self.analyze = dspy.ChainOfThought(SupportTicketAnalysis)
    
    def forward(self, ticket_text: str):
        result = self.analyze(ticket_text=ticket_text)
        return dspy.Prediction(
            category=result.category,
            priority=result.priority,
            required_action=result.required_action,
            response_template=result.response_template,
            confidence=0.0,  # Will be calculated
        )


# ── Metrics ──

def routing_metric(example, prediction, trace=None):
    """
    Comprehensive routing metric.
    
    Considers: category match, priority match, action match.
    Higher weight on category (getting to the right team).
    """
    score = 0.0
    
    # Category: most important (0-0.5)
    if prediction.category == example.category:
        score += 0.5
    elif prediction.category and example.category and \
         prediction.category.strip().lower() == example.category.strip().lower():
        score += 0.4
    
    # Priority: important for SLAs (0-0.25)
    if prediction.priority == example.priority:
        score += 0.25
    
    # Action: useful for automation (0-0.25)
    if prediction.required_action == example.required_action:
        score += 0.25
    elif prediction.required_action and example.required_action:
        # Partial credit for close actions
        close_pairs = [("refund", "escalate"), ("troubleshoot", "escalate"), ("inform", "no_action")]
        if any(prediction.required_action in pair and example.required_action in pair 
               for pair in close_pairs):
            score += 0.15
    
    return score


# ── Evaluation ──

class DSPyEvaluator:
    """Evaluate DSPy-optimized pipelines against baselines."""
    
    def __init__(self, testset: list):
        self.testset = testset
    
    def evaluate(self, pipeline: dspy.Module, name: str = "pipeline") -> dict:
        """Evaluate a pipeline on the test set."""
        scores = []
        errors = []
        latencies = []
        
        import time
        for example in self.testset:
            start = time.time()
            try:
                prediction = pipeline(ticket_text=example.ticket_text)
                score = routing_metric(example, prediction)
                scores.append(score)
            except Exception as e:
                errors.append(str(e))
                scores.append(0.0)
            latencies.append((time.time() - start) * 1000)
        
        import statistics
        return {
            "name": name,
            "mean_score": statistics.mean(scores) if scores else 0,
            "median_score": statistics.median(scores) if scores else 0,
            "error_rate": len(errors) / len(self.testset) if self.testset else 0,
            "p50_latency_ms": statistics.median(latencies) if latencies else 0,
            "p95_latency_ms": sorted(latencies)[int(len(latencies) * 0.95)] if latencies else 0,
            "num_samples": len(scores),
            "num_errors": len(errors),
        }


# ── Full Pipeline ──

async def run_dspy_pipeline(
    trainset: list,
    valset: list,
    testset: list,
    method: str = "mipro",
):
    """
    Full DSPy optimization pipeline.
    
    Args:
        trainset: Training examples (40%)
        valset: Validation examples (30%)
        testset: Test examples (30%)
        method: "bfs" (BootstrapFewShot) or "mipro" (MIPROv2)
    """
    
    # Configure LM
    lm = dspy.OpenAI(model="gpt-4o-mini", max_tokens=500)
    dspy.settings.configure(lm=lm)
    
    # Create evaluator
    evaluator = DSPyEvaluator(testset)
    
    # 1. Baseline (unoptimized)
    baseline = TicketRouter()
    baseline_results = evaluator.evaluate(baseline, "baseline")
    print(f"Baseline: {baseline_results['mean_score']:.3f} "
          f"(errors: {baseline_results['error_rate']:.1%})")
    
    # 2. Optimize
    if method == "mipro":
        optimizer = MIPROv2(
            metric=routing_metric,
            num_candidates=8,
            num_trials=20,
        )
        optimized = optimizer.compile(
            baseline,
            trainset=trainset,
            valset=valset,
            max_bootstrapped_demos=3,
            max_labeled_demos=4,
            requires_permission_to_run=False,
        )
    else:
        optimizer = BootstrapFewShot(
            metric=routing_metric,
            max_bootstrapped_demos=4,
            max_labeled_demos=4,
        )
        optimized = optimizer.compile(baseline, trainset=trainset)
    
    # 3. Evaluate optimized
    optimized_results = evaluator.evaluate(optimized, f"optimized_{method}")
    print(f"Optimized ({method}): {optimized_results['mean_score']:.3f} "
          f"(errors: {optimized_results['error_rate']:.1%})")
    
    # 4. Compare
    improvement = optimized_results['mean_score'] - baseline_results['mean_score']
    latency_cost = optimized_results['p50_latency_ms'] - baseline_results['p50_latency_ms']
    
    print(f"\nImprovement: {improvement:+.3f} "
          f"(+{improvement/baseline_results['mean_score']*100:.1f}%)")
    print(f"Latency impact: {latency_cost:+.1f}ms")
    
    return {
        "baseline": baseline_results,
        "optimized": optimized_results,
        "improvement": improvement,
        "improvement_pct": improvement / baseline_results['mean_score'] * 100,
        "latency_impact_ms": latency_cost,
        "pipeline": optimized,
    }
```

---

## ✅ Good Output Example

```python
# DSPy optimization of a medical triage classifier

"""
BEFORE DSPy:
System prompt: "Classify medical urgency as routine, urgent, or emergency."
Accuracy: 78%
Issues: Confuses "urgent" with "emergency", gives no reasoning

AFTER DSPy (MIPROv2, 30 trials, $12 cost):
System prompt: 
"You are a trained medical triage nurse with 10 years of ER experience.
Classify each case by severity using the standard triage framework.

Follow this reasoning:
1. Identify primary symptom and duration
2. Check for red flags (chest pain, difficulty breathing, severe bleeding)
3. Check for yellow flags (high fever, persistent pain, confusion)
4. Consider patient demographics (age, pre-existing conditions)
5. Make final triage decision

Categories:
- emergency: Immediate threat to life/limb
- urgent: Needs attention within hours
- semi_urgent: Needs attention within 24 hours
- routine: Can wait for scheduled appointment

Examples: [4 examples covering different triage levels]
"""

Accuracy: 91% (+13%)
False negatives (emergency classified as routine): Reduced by 60%
Latency: +200ms (from reasoning chain) — acceptable for triage
"""

# DSPy didn't just "optimize the prompt" — it restructured the ENTIRE
# reasoning approach, added a triage framework the engineer didn't know about,
# and selected examples that cover edge cases. This is the power of
# programmatic optimization.
```

---

## 🧪 Drills

### Drill 1: DSPy vs. Manual

Take a task you've been doing manually (classification, extraction, summarization). Implement it with DSPy:
1. Write the signature
2. Define a comprehensive metric
3. Create a training set (50+ examples)
4. Run BootstrapFewShot optimization
5. Compare: optimized vs. your best manual prompt

Report: Which won? By how much? What did DSPy change?

### Drill 2: Inspect the Black Box

Run MIPROv2 on a task. Inspect the optimized prompts for each module:
1. What instructions did DSPy add?
2. Which few-shot examples were selected?
3. Are there any instructions that seem WRONG or unsafe?
4. Can you simplify the optimized prompt without losing accuracy?

### Drill 3: The Overfitting Test

Split your data: 50% train, 25% validation, 25% test.
1. Optimize with BootstrapFewShot on training set
2. Evaluate on validation set — is there a gap? (train vs validation score)
3. If gap > 5%, you're overfitting. How would you reduce overfitting?
4. Test on held-out test set — report final results

### Drill 4: Custom Metric Design

Your current metric is simple accuracy. Design a better one:
1. What are the different types of errors? (e.g., false positives vs false negatives)
2. Which errors are more expensive? (classifying an "emergency" as "routine" is worse than the reverse)
3. Design a cost matrix for your task
4. Implement a weighted metric that reflects real costs
5. Re-run optimization — does the new metric change DSPy's behavior?

---

## 🚦 Gate Check

- [ ] You've installed DSPy and run at least one optimization
- [ ] You can explain what DSPy signatures, modules, and teleprompters do
- [ ] You've inspected an optimized prompt and can interpret the changes
- [ ] You understand the cost/quality tradeoffs between optimization methods
- [ ] You've designed a custom metric that goes beyond simple accuracy
- [ ] You can detect and prevent overfitting in prompt optimization
- [ ] **You thought through all 3 production scenarios at the top**
- [ ] **You've compared DSPy-optimized vs. manual prompts on a real task**

---

## 📚 Cross-References

- [Evaluation & A/B Testing](06-Evaluation-AB-Testing.md) — Metrics for DSPy optimization
- [Context Engineering](01-Context-Engineering.md) — Understanding what DSPy changes in prompts
- [System Prompts & Personas](03-System-Prompts-Personas.md) — DSPy-optimized system prompts
- [Phase 9: Fine-Tuning](../09-Fine-Tuning-LLMOps/) — BootstrapFinetune uses DSPy to generate fine-tuning data
- [Phase 6: AI Agents](../06-AI-Agents/) — DSPy's ReAct module for agent optimization
