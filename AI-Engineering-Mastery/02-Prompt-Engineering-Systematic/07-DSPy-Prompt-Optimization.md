# DSPy: Programming, Not Prompting

## The Problem

Prompt engineering is manual, fragile, and non-transferable:
- "Vibe-coding" prompts works once, fails on edge cases
- No systematic way to optimize for a metric (accuracy, cost, latency)
- Prompts that work for GPT-4 fail on Llama 3
- No regression testing when you change providers

## DSPy Philosophy

**"Compile, don't prompt."** Instead of hand-tuning prompts, DSPy treats LLM programs as parameterized modules that can be optimized automatically.

```python
import dspy
from dspy.datasets import HotPotQA

# Before DSPy: hand-craft a prompt for QA
# After DSPy: define a module, compile against a metric

class QA(dspy.Signature):
    """Answer questions with citations."""
    question: str = dspy.InputField()
    context: list[str] = dspy.InputField()
    answer: str = dspy.OutputField()
    citation: str = dspy.OutputField()

class RAG(dspy.Module):
    def __init__(self, num_passages=3):
        self.retrieve = dspy.Retrieve(k=num_passages)
        self.respond = dspy.ChainOfThought(QA)

    def forward(self, question):
        context = self.retrieve(question).passages
        return self.respond(question=question, context=context)

# Compile: auto-optimize prompts against a metric
turbo = dspy.OpenAI(model="gpt-4o-mini", max_tokens=250)
dspy.settings.configure(lm=turbo)

# Define metric
def gold_answer_metric(gold, pred, trace=None):
    return gold.answer == pred.answer

# Compile optimizes: few-shot examples, prompt structure, chain-of-thought
optimized_rag = dspy.teleprompt.BootstrapFewShot(
    metric=gold_answer_metric,
    max_bootstrapped_demos=4,
    max_labeled_demos=8,
).compile(RAG(), trainset=trainset, valset=valset)
```

## DSPy Optimizers

| Optimizer | What It Does | When to Use |
|-----------|-------------|-------------|
| `BootstrapFewShot` | Auto-selects few-shot examples from training data | You have labeled examples |
| `BootstrapFewShotWithRandomSearch` | Random search over example subsets + prompt variations | More data, more budget |
| `MIPROv2` | Bayesian optimization over instructions + examples | Highest quality, most expensive |
| `SignatureOptimizer` | Optimizes the signature/field descriptions | Quick wins with minimal data |
| `COPRO` | Generates and refines instructions iteratively | Cold start (no examples) |

## DSPy Assertions (Runtime Guards)

```python
class FactCheck(dspy.Signature):
    claim: str = dspy.InputField()
    evidence: list[str] = dspy.InputField()
    supported: bool = dspy.OutputField()
    reasoning: str = dspy.OutputField()

class VerifiedQA(dspy.Module):
    def __init__(self):
        self.generate = dspy.ChainOfThought(QA)
        self.verify = dspy.ChainOfThought(FactCheck)

    def forward(self, question, context):
        answer = self.generate(question=question, context=context)
        dspy.Assert(
            len(answer.answer) > 0,
            "Answer should not be empty",
            target_module=FactCheck,
        )
        verification = self.verify(
            claim=answer.answer,
            evidence=context,
        )
        dspy.Assert(
            verification.supported,
            "Answer must be supported by evidence",
            target_module=self.generate,
        )
        return answer
```

## End-to-End Pipeline

```python
# 1. Collect labeled data (20-100 examples minimum)
trainset = [
    dspy.Example(question="...", answer="...").with_inputs("question"),
    # ...
]

# 2. Define metric
def accuracy_metric(gold, pred, trace=None):
    return gold.answer == pred.answer

# 3. Define program
class MyProgram(dspy.Module):
    def __init__(self):
        self.step1 = dspy.ChainOfThought(Signature1)
        self.step2 = dspy.Predict(Signature2)

    def forward(self, input):
        result1 = self.step1(input=input)
        return self.step2(context=result1.output)

# 4. Compile (optimize)
optimized = dspy.teleprompt.BootstrapFewShot(
    metric=accuracy_metric,
).compile(MyProgram(), trainset=trainset)

# 5. Use (prompts are now optimized)
result = optimized(input="new query")
```

## 🔴 Senior: DSPy vs Manual Prompting

| Aspect | Manual Prompting | DSPy |
|--------|-----------------|------|
| Effort per task | High (tweak → test → repeat) | Medium (write module + metric) |
| Transferability | None (rewrite for new model) | High (recompile for new model) |
| Optimality | Gut feel | Metric-driven search |
| Regression testing | Manual | Built-in (eval on valset) |
| Debugging | Inspect raw prompts | Inspect signatures + traces |
| Team collaboration | Shared docs | Shared modules + metrics |

## When DSPy Falls Short

- **Need < 20 examples**: Insufficient data for optimization
- **Highly creative tasks**: "Maximize interestingness" is hard to metricize
- **Non-deterministic metrics**: If the metric has high variance, optimization is noisy
- **Extreme latency sensitivity**: Compilation adds overhead; use compiled prompts in prod

## Drill: Optimize a Prompt with DSPy

1. Pick a task (classification, extraction, QA)
2. Create 30 labeled examples
3. Define a DSPy module for the task
4. Define a metric (accuracy, F1, or custom)
5. Compile with `BootstrapFewShot` and `MIPROv2`
6. Compare: hand-tuned vs DSPy-optimized on a held-out test set
7. Switch from GPT-4o-mini to Llama 3 (via Ollama) and recompile — note the difference
