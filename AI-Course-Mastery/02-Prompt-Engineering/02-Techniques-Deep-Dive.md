# Techniques Deep Dive — Zero-Shot, Few-Shot, Chain-of-Thought & Beyond

## 🎯 Purpose & Goals

> **🛑 STOP. You know WHY prompt techniques work (Context Engineering). Now you need to know WHEN to use which — and how to combine them.**
>
> Here are 5 techniques. Match each to the right task:
>
> 1. **Zero-shot**: "Translate this to French: Hello"
> 2. **Few-shot**: "Q: 2+2= A: 4 || Q: 5+3= A: 8 || Q: 7+1= A:"
> 3. **Chain-of-Thought**: "Let's solve this step by step..."
> 4. **Self-Consistency**: Sample 5 times, take majority answer
> 5. **Tree-of-Thought**: Explore multiple reasoning branches
>
> Tasks:
> - A) Classifying customer email sentiment (binary)
> - B) Solving a complex math word problem
> - C) Translating 100 medical terms
> - D) Deciding between 2 contradictory research papers
> - E) Generating creative marketing copy
>
> *Which technique for which task? (Answers at the bottom of this file)*

---

**By the end of this file, you will:**
- Know exactly which technique to use for any task
- Understand the token efficiency vs. accuracy tradeoff for each
- Be able to combine techniques (CoT + few-shot + self-consistency)
- Build a technique selector that automatically picks the right approach

**⏱ Time Budget:** 2.5 hours (1.5 hour concept, 1 hour experiments)

---

## 📖 Technique Spectrum — From Cheapest to Most Expensive

```
TOKEN COST (lower = cheaper)
│
│    Zero-shot (cheapest, least accurate)
│     │
│     ├── Zero-shot + persona
│     │
│     ├── Few-shot (2-5 examples)
│     │
│     ├── Few-shot + persona
│     │
│     ├── Chain-of-Thought (medium cost, high accuracy)
│     │
│     ├── CoT + few-shot (best accuracy/cost ratio)
│     │
│     ├── Self-consistency (expensive, robust)
│     │
│     └── Tree-of-Thought (most expensive, highest reasoning)
│
▼ TOKEN COST (higher = more expensive)
```

```
ACCURACY vs COST TRADEOFF for MATH REASONING:

Technique                   | Accuracy | Tokens Used | Cost/Question
----------------------------|----------|-------------|--------------
Zero-shot                   | 54%      | 50          | $0.000008
Zero-shot + "think step"   | 73%      | 120         | $0.000018
Few-shot (2 examples)       | 68%      | 180         | $0.000027
Few-shot (5 examples)       | 75%      | 350         | $0.000053
CoT                         | 79%      | 150         | $0.000023
CoT + 5-shot                | 85%      | 450         | $0.000068
CoT + Self-consistency (5x) | 88%      | 750         | $0.000113
Tree-of-Thought             | 91%      | 2000+       | $0.000300+

Senior Insight: CoT + few-shot gives 85% accuracy at 40% of the cost of ToT.
Only use ToT for the hardest 5% of problems.
```

---

## 💻 Technique Details

### 1. Zero-Shot — The Baseline

```python
"""
Zero-shot: The model uses its training knowledge directly.
No examples, no reasoning chain. Just instruction + question.

Best for:
- Simple classification (sentiment, topic, language)
- Translation (common language pairs)
- Factual lookup (capitals, dates, definitions)
- Format conversion (text to JSON)

Worst for:
- Complex reasoning (math, logic, multi-step)
- Unusual formats (model hasn't seen your exact schema)
- Tasks requiring strict adherence to rules
"""

# ZERO-SHOT EXAMPLES:

# Simple classification — works great
prompt = """
Classify the sentiment of this text as POSITIVE, NEGATIVE, or NEUTRAL.
Text: "This product exceeded my expectations!"
Sentiment:
"""
# Response: POSITIVE ← Works

# Complex reasoning — fails often
prompt = """
Alice has 3 apples. She gives 1 to Bob, then Bob gives 2 to Charlie.
If Charlie gives 1 back to Alice, how many does Alice have?
"""
# Response: 3 ← WRONG (should be 3, but model often says 2 without reasoning)

# SENIOR MOVE: Even for "simple" tasks, zero-shot has failure modes.
# Always test on 20+ examples before trusting zero-shot for production.
```

```python
from typing import Optional
import asyncio


class ZeroShotPrompt:
    """
    Builder for zero-shot prompts.
    
    Zero-shot = instruction + input.
    No examples, no reasoning scaffolding.
    """
    
    @staticmethod
    def classify(
        text: str,
        categories: list[str],
        task_description: Optional[str] = None,
    ) -> str:
        """
        Build a zero-shot classification prompt.
        
        Args:
            text: Text to classify
            categories: List of valid categories
            task_description: Optional specific description
        
        Returns:
            Formatted prompt string
        """
        categories_str = ", ".join(categories)
        prompt = f"""Classify the following text into one of these categories: {categories_str}.

Text: "{text}"

Category:"""
        
        if task_description:
            prompt = f"Task: {task_description}\n\n{prompt}"
        
        return prompt
    
    @staticmethod
    def extract(
        text: str,
        fields: list[str],
        output_format: str = "json",
    ) -> str:
        """
        Build a zero-shot extraction prompt.
        """
        fields_str = ", ".join(fields)
        return f"""Extract the following fields from the text: {fields_str}.

Text: "{text}"

Respond in {output_format} format:
{{"""
    
    @staticmethod
    def transform(
        text: str,
        source_format: str,
        target_format: str,
    ) -> str:
        """
        Build a zero-shot format transformation prompt.
        """
        return f"""Convert the following {source_format} to {target_format}.

Source ({source_format}):
{text}

{target_format}:"""


# Test: Zero-shot accuracy varies wildly by task
test_results = {
    "sentiment_analysis": 0.94,  # Great — simple pattern
    "language_detection": 0.97,  # Excellent — clear signal
    "math_reasoning": 0.54,      # Terrible — needs reasoning
    "entity_extraction": 0.88,   # Good — pattern matching
    "code_generation_simple": 0.91,  # Great — seen in training
    "instruction_following": 0.76,   # Mixed — depends on specificity
}
```

### 2. Few-Shot — Learning from Examples

```python
"""
Few-shot: Provide 2-5 examples of the desired input-output pattern.
The model recognizes the pattern and continues it.

CRITICAL INSIGHT: Examples serve TWO purposes:
1. They show the output format (format control)
2. They provide content context (token distribution)

🔴 SENIOR MOVE: Choose examples that cover:
- Edge cases (empty inputs, errors, unusual formats)
- The MOST COMMON case (so the pattern is clear)
- One "hard" example (shows the model how to handle complexity)

DON'T choose:
- Random examples (patterns must be clear)
- All easy examples (model can't handle hard cases)
- Examples that contradict each other (different formats)
"""


class FewShotPrompt:
    """
    Builder for few-shot prompts with example selection.
    
    Features:
    - Automatic example selection (choose best examples for each query)
    - Format validation (examples must follow consistent format)
    - Variable example count (based on query complexity)
    """
    
    def __init__(self):
        self.examples: list[dict] = []  # [{"input": ..., "output": ...}]
    
    def add_example(self, input_text: str, output_text: str):
        """Add an example pair."""
        self.examples.append({"input": input_text, "output": output_text})
    
    def add_examples(self, examples: list[tuple[str, str]]):
        """Add multiple example pairs."""
        for inp, out in examples:
            self.add_example(inp, out)
    
    def format_examples(self, example_prefix: str = "Input: ", 
                       output_prefix: str = "Output: ") -> str:
        """Format all examples into a prompt prefix."""
        parts = []
        for ex in self.examples:
            parts.append(f"{example_prefix}{ex['input']}")
            parts.append(f"{output_prefix}{ex['output']}")
        return "\n".join(parts)
    
    def build(self, query: str, instruction: str = "") -> str:
        """
        Build the complete few-shot prompt.
        
        Structure:
            [instruction]
            [example 1]
            [example 2]
            [query] <-- model completes this
        """
        parts = []
        if instruction:
            parts.append(instruction)
        parts.append(self.format_examples())
        parts.append(f"Input: {query}")
        parts.append("Output: ")  # Model fills this in
        return "\n\n".join(parts)
    
    def select_diverse_examples(
        self, query: str, pool: list[dict], n: int = 3
    ) -> list[dict]:
        """
        Select the N most informative examples for a given query.
        
        Uses simple diversity sampling: choose examples that are
        different from each other AND relevant to the query.
        
        🔴 SENIOR MOVE: Random example selection is BAD.
        Choose examples strategically based on similarity to query.
        """
        if not pool:
            return []
        
        # Convert query to a simple feature representation
        query_features = set(query.lower().split())
        
        def similarity(ex: dict) -> float:
            """Score how similar an example is to the query."""
            ex_features = set(ex["input"].lower().split())
            if not ex_features:
                return 0
            intersection = query_features & ex_features
            return len(intersection) / len(ex_features)
        
        # Score all examples
        scored = [(ex, similarity(ex)) for ex in pool]
        scored.sort(key=lambda x: x[1], reverse=True)
        
        return [ex for ex, _ in scored[:n]]
    
    @staticmethod
    def validate_examples(examples: list[dict]) -> list[str]:
        """
        Validate that examples are consistent and useful.
        
        Returns list of issues (empty = valid).
        """
        issues = []
        
        if not examples:
            return ["No examples provided"]
        
        if len(examples) < 2:
            issues.append("Only 1 example — pattern may not be clear")
        
        if len(examples) > 10:
            issues.append(f"{len(examples)} examples may exceed context window budget")
        
        # Check for format consistency
        first_output = examples[0].get("output", "")
        for i, ex in enumerate(examples[1:], 2):
            if len(ex.get("output", "")) > len(first_output) * 3:
                issues.append(f"Example {i} has much longer output than example 1 — inconsistent formatting")
        
        return issues
```

### 3. Chain-of-Thought — Reasoning Tokens

```python
"""
Chain-of-Thought (CoT): The single most impactful prompt technique.

HOW IT WORKS:
Instead of predicting the answer directly, the model generates
intermediate reasoning steps before the final answer.

Before CoT:
  Q: "Alice has 3 apples. She gives 1 away. How many left?"
  A: "2"  ← Direct prediction (often wrong for complex problems)

After CoT:
  Q: "Alice has 3 apples. She gives 1 away. How many left?"
  A: "Let's think step by step:
      1. Alice starts with 3 apples
      2. She gives 1 away
      3. 3 - 1 = 2
      So Alice has 2 apples left."
      
WHY CoT WORKS (from Context Engineering):
The intermediate tokens shift the attention distribution.
Instead of jumping directly from "gives away" → "subtract",
the model generates tokens that activate arithmetic attention patterns.
"""

import asyncio


class ChainOfThought:
    """
    Multiple CoT strategies for different task types.
    
    The key insight: Different CoT prompts work for different tasks.
    "Let's think step by step" is the default, but not always the best.
    """
    
    @staticmethod
    def math_reasoning() -> str:
        """
        CoT prompt optimized for math/arithmetic.
        
        Forces explicit step-by-step calculation.
        """
        return """Let's solve this step by step.

Step 1: Identify the given information.
Step 2: Identify what we need to find.
Step 3: Set up the equation.
Step 4: Solve step by step.
Step 5: Verify the answer.

Question: {question}
Solution:"""
    
    @staticmethod
    def logical_reasoning() -> str:
        """
        CoT prompt optimized for logic puzzles.
        
        Uses formal logic notation.
        """
        return """Let's reason through this logically:

Given facts:
{fact_list}

Goal: {goal}

Let me work through this:
1. First, what do we know?
2. What can we deduce from each fact?
3. Are there any contradictions?
4. What's the conclusion?

Reasoning:"""
    
    @staticmethod
    def multi_hop_qa() -> str:
        """
        CoT prompt for multi-hop questions (RAG-style).
        
        Breaks down complex questions into sub-questions.
        """
        return """I need to answer this question by breaking it down.

Question: {question}

Let me think about what I need to find out:
1. What sub-questions do I need to answer first?
2. What information do I need for each?
3. Let me solve each sub-question:
   - {sub_question_1}
   - {sub_question_2}
4. Now I can combine my answers.

Final answer:"""
    
    @staticmethod
    def code_generation() -> str:
        """
        CoT prompt for code generation.
        
        Forces planning before coding.
        """
        return """Let me plan the implementation:

1. Understanding the task:
   - Input: {input_description}
   - Output: {output_description}
   - Constraints: {constraints}

2. Design:
   - Functions needed:
   - Data structures:
   - Algorithm:
   - Error cases:

3. Implementation:
```python
# Code here
```

4. Testing:
   - Test case 1: {test_input_1} → {expected_output_1}
   - Edge case: {edge_case_input}"""

    @staticmethod
    def comparison() -> str:
        """
        CoT for compare/contrast tasks.
        """
        return """Let me compare these systematically:

Item A: {item_a}
Item B: {item_b}

Dimensions to compare:
1. Performance: 
2. Cost:
3. Complexity:
4. Trade-offs:

For each dimension:
- Which is better?
- By how much?
- Under what conditions?

Conclusion:"""
```

### 4. Self-Consistency — The Voting Ensemble

```python
"""
Self-Consistency: Run the SAME question N times, take majority answer.

This is NOT "asking the same question repeatedly" — each run follows
a DIFFERENT reasoning path (due to sampling temperature > 0).

HOW IT WORKS:
┌─────────────┐
│  Question   │
└──────┬──────┘
       │
       ├─────────┬─────────┬─────────┐
       ▼         ▼         ▼         ▼
    Sampler 1  Sampler 2  Sampler 3  Sampler N
    (temp=0.7) (temp=0.7) (temp=0.7) (temp=0.7)
       │         │         │         │
       ▼         ▼         ▼         ▼
    Answer 1   Answer 2   Answer 3   Answer N
       │         │         │         │
       └─────────┴─────────┴─────────┘
                      │
                      ▼
              MAJORITY VOTE
                      │
                      ▼
             FINAL ANSWER


KEY INSIGHT: For math problems, self-consistency with N=5 improves
accuracy from ~79% (single CoT) to ~88% — an 11% gain for 5x cost.
The gain diminishes after N=5.

TRADEOFF: 5x cost for 11% accuracy gain. Worth it for high-stakes
problems. Wasteful for simple classification.
"""

import asyncio
from collections import Counter


class SelfConsistency:
    """
    Run the same prompt N times with sampling and take majority vote.
    
    🔴 SENIOR MOVE: Only use for tasks where:
    1. The answer is factual (has a right/wrong answer)
    2. Multiple reasoning paths can lead to the answer
    3. Wrong answers are WRONG (not subjective)
    
    DON'T use for: creative writing, summarization, opinion, style transfer
    """
    
    def __init__(self, llm_client, n_samples: int = 5, temperature: float = 0.7):
        self.client = llm_client
        self.n_samples = n_samples
        self.temperature = temperature
    
    async def answer(self, prompt: str, answer_extractor: callable) -> dict:
        """
        Run self-consistency and return results.
        
        Args:
            prompt: The (CoT) prompt to run
            answer_extractor: Function to extract answer from response
                e.g., lambda text: text.split("Answer:")[-1].strip()
        
        Returns:
            Dict with: final_answer, all_answers, distribution, confidence
        """
        tasks = []
        for _ in range(self.n_samples):
            tasks.append(self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                temperature=self.temperature,
            ))
        
        responses = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Extract and count answers
        answers = []
        valid_responses = 0
        for response in responses:
            if isinstance(response, Exception):
                continue
            valid_responses += 1
            try:
                answer = answer_extractor(response.content)
                answers.append(answer)
            except Exception:
                continue
        
        if not answers:
            return {"error": "No valid answers extracted"}
        
        # Majority vote
        counter = Counter(answers)
        most_common = counter.most_common(1)[0]
        
        return {
            "final_answer": most_common[0],
            "vote_count": most_common[1],
            "total_votes": valid_responses,
            "confidence": most_common[1] / valid_responses,
            "distribution": dict(counter.most_common()),
            "all_answers": answers,
        }


# Performance characteristics
SELF_CONSISTENCY_PERFORMANCE = {
    "math_gsm8k": {
        "single_cot": 0.79,
        "sc_n3": 0.85,
        "sc_n5": 0.88,
        "sc_n10": 0.89,
        "cost_multiplier_n5": 5.0,
    },
    "commonsense": {
        "single_cot": 0.75,
        "sc_n3": 0.79,
        "sc_n5": 0.81,
        "cost_multiplier_n5": 5.0,
    },
    "symbolic_reasoning": {
        "single_cot": 0.62,
        "sc_n3": 0.71,
        "sc_n5": 0.74,
        "cost_multiplier_n5": 5.0,
    },
}

# Insight: Self-consistency gains are HIGHEST for tasks where
# there's a clear right answer and multiple valid reasoning paths.
```

### 5. Tree-of-Thought — Exploring Multiple Paths

```python
"""
Tree-of-Thought (ToT): Like CoT, but explores MULTIPLE reasoning
branches and evaluates each.

┌────────────────────────────────────────────┐
│              QUESTION                       │
└────────────────┬───────────────────────────┘
                 │
    ┌────────────┴────────────┐
    ▼                         ▼
  Branch A                  Branch B
  "Approach 1..."           "Approach 2..."
    │                         │
    ├──────┬──────┐           ├──────┬──────┐
    ▼      ▼      ▼           ▼      ▼      ▼
  A1      A2     A3         B1     B2     B3
  "Step"  "Step" "Step"     "..."  "..."  "..."
    │      │      │           │      │      │
    └──────┴──────┘           └──────┴──────┘
         Evaluation               Evaluation
         (score: 0.8)             (score: 0.3)
              │                        │
              └──────────┬─────────────┘
                         ▼
              BEST PATH (A2)
                         │
                         ▼
                   FINAL ANSWER

ToT is CoT + search. At each step, the model generates multiple
candidate "thoughts" and evaluates them, keeping the most promising.

COST: EXTREMELY expensive. Each branch generates tokens at every level.
For a tree of depth 5 with 3 branches each: 3^5 = 243 reasoning paths.

WHEN TO USE:
- Only for the hardest reasoning problems (math olympiad, puzzles)
- When wrong answers are expensive (medical diagnosis, legal)
- As a research tool to understand model reasoning

NEVER use ToT for: classification, extraction, summarization, chat.
"""

# ToT is typically not implemented as a prompt — it's an algorithm
# that calls the model multiple times. Implementation is framework-specific.

# Pseudo-implementation:
"""
async def tree_of_thought(
    question: str,
    n_branches: int = 3,
    max_depth: int = 3,
    evaluator_prompt: str = "Rate this reasoning from 0-1:"
):
    tree = {root: question}
    
    for depth in range(max_depth):
        new_branches = {}
        for node, path in tree.items():
            # Generate next steps for this path
            candidates = await generate_n_steps(path, n_branches)
            
            # Evaluate each candidate
            for candidate in candidates:
                score = await evaluate_step(
                    f"{evaluator_prompt}\n\nReasoning: {candidate}"
                )
                new_branches[candidate] = score
        
        # Prune: keep top 1/3 of branches
        tree = dict(sorted(new_branches.items(), key=lambda x: x[1], reverse=True)[:n_branches])
    
    # Take best leaf
    best_path = max(tree, key=tree.get)
    return best_path
"""
```

---

## 💻 Technique Selector — The Decision System

```python
"""
Automatic technique selection based on task characteristics.

🔴 SENIOR MOVE: Don't guess which technique to use. Let the system
choose based on the task type, accuracy requirements, and cost budget.
"""

from enum import Enum
from dataclasses import dataclass
from typing import Optional


class TaskType(Enum):
    CLASSIFICATION = "classification"
    EXTRACTION = "extraction"
    SUMMARIZATION = "summarization"
    TRANSLATION = "translation"
    REASONING = "reasoning"
    CREATIVE = "creative"
    CODE = "code"
    COMPARISON = "comparison"
    FACTUAL = "factual_qa"


@dataclass
class TaskProfile:
    """Profile of a task for technique selection."""
    type: TaskType
    required_accuracy: float      # 0.0 to 1.0
    cost_tolerance: str           # "low", "medium", "high"
    complexity: str               # "simple", "medium", "complex"
    has_right_answer: bool        # True if factual
    examples_available: int       # Number of good examples you have


TECHNIQUE_RECOMMENDATIONS = {
    TaskType.CLASSIFICATION: {
        "simple": "zero_shot",
        "medium": "few_shot",
        "complex": "few_shot_cot",
    },
    TaskType.EXTRACTION: {
        "simple": "zero_shot",
        "medium": "few_shot",
        "complex": "few_shot_structured",
    },
    TaskType.REASONING: {
        "simple": "zero_shot_cot",
        "medium": "few_shot_cot",
        "complex": "cot_self_consistency",
    },
    TaskType.CODE: {
        "simple": "zero_shot",
        "medium": "few_shot",
        "complex": "cot_planning",
    },
    TaskType.CREATIVE: {
        "simple": "zero_shot_creative",
        "medium": "few_shot_creative",
        "complex": "multi_turn_refinement",
    },
}


def select_technique(task: TaskProfile) -> dict:
    """
    Select the optimal prompt technique for a task.
    
    Returns:
        technique: Name of the technique
        parameters: Specific parameters (n_shots, temperature, etc.)
        estimated_cost: Token cost estimate
        reasoning: Why this technique was chosen
    """
    technique = TECHNIQUE_RECOMMENDATIONS[task.type][task.complexity]
    
    params = {
        "zero_shot": {"temperature": 0.0, "n_shots": 0, "use_cot": False},
        "zero_shot_cot": {"temperature": 0.3, "n_shots": 0, "use_cot": True},
        "few_shot": {"temperature": 0.0, "n_shots": min(task.examples_available, 5), "use_cot": False},
        "few_shot_cot": {"temperature": 0.3, "n_shots": min(task.examples_available, 3), "use_cot": True},
        "cot_self_consistency": {"temperature": 0.7, "n_samples": 5, "use_cot": True},
    }.get(technique, {"temperature": 0.0, "n_shots": 0, "use_cot": False})
    
    # Cost estimation
    cost_multipliers = {
        "zero_shot": 1.0,
        "zero_shot_cot": 2.5,
        "few_shot": 2.0 + params.get("n_shots", 0) * 0.3,
        "few_shot_cot": 4.0 + params.get("n_shots", 0) * 0.5,
        "cot_self_consistency": 12.5,  # 5x CoT
    }
    
    return {
        "technique": technique,
        "parameters": params,
        "estimated_cost_multiplier": cost_multipliers.get(technique, 1.0),
        "reasoning": f"{task.type.value} task with {task.complexity} complexity → {technique}",
    }


# Usage
task = TaskProfile(
    type=TaskType.REASONING,
    required_accuracy=0.85,
    cost_tolerance="medium",
    complexity="complex",
    has_right_answer=True,
    examples_available=10,
)

result = select_technique(task)
print(result)
# {
#     "technique": "cot_self_consistency",
#     "parameters": {"temperature": 0.7, "n_samples": 5, "use_cot": True},
#     "estimated_cost_multiplier": 12.5,
#     "reasoning": "reasoning task with complex complexity → cot_self_consistency"
# }
```

---

## 📊 Answer Matching Quiz (From Top)

```
1. C) Translation of medical terms → Zero-shot (translation is well-learned)
2. A) Email sentiment → Zero-shot or 1-shot (simple classification)
3. E) Creative copy → Zero-shot creative (don't constrain with examples)
4. B) Math word problem → CoT or CoT + self-consistency
5. D) Contradictory papers → CoT + few-shot (reasoning + examples)
```

---

## ✅ Good Output Example

```python
# Real production decision:
# You're building a medical QA system. Questions require reasoning.
# You need >90% accuracy. Cost is a concern.

task = TaskProfile(
    type=TaskType.REASONING,
    required_accuracy=0.90,
    cost_tolerance="medium",
    complexity="complex",
    has_right_answer=True,
    examples_available=20,
)

decision = select_technique(task)
# Chose: few_shot_cot with n_shots=3
# Cost multiplier: 5.5x base
# Expected accuracy: ~85% (need self-consistency for 90%)

# 🔴 SENIOR MOVE: If you need >90% accuracy:
# 1. Run few-shot CoT (3 examples, temp=0.3)
# 2. For cases where confidence < 0.7, upsample to self-consistency
# This hybrid approach costs 2-3x base, not 12.5x
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: Using CoT for Non-Reasoning Tasks
```
Bad: "Let's think step by step about how to translate 'Hello' to French."
Result: Overkill. Wastes tokens. Sometimes hurts accuracy.

Good: "Translate this to French: Hello"
```

### ❌ Antipattern 2: Too Many Examples in Few-Shot
```
Bad: 20 examples of email classification.
Problem: Each example increases token count. Beyond ~5-7, 
diminishing returns kick in HARD.

Good: 3-5 diverse examples covering edge cases.
```

### ❌ Antipattern 3: Self-Consistency at Temperature 0
```
Bad: n_samples=5, temperature=0
Result: All 5 answers are identical. Waste of 4x cost.

Good: n_samples=5, temperature=0.3-0.7
Result: Diverse reasoning paths, proper majority vote.
```

### ❌ Antipattern 4: ToT for Simple Tasks
```
Bad: "Use tree-of-thought to classify this email as spam or not."
Result: 243 reasoning branches to decide "spam/not spam." 
Cost: $0.05/question for a task that zero-shot handles at $0.00001.
```

---

## 🧪 Drills

### Drill 1: Technique Comparison

Create a test dataset of 20 questions across 4 task types. Run each through:
1. Zero-shot
2. Few-shot (3 examples)
3. CoT
4. CoT + few-shot (2 examples)

Measure accuracy and token cost. Which technique wins for each task type?

```python
async def compare_techniques(llm_client, test_set):
    scores = {}
    for name, tasks in test_set.items():
        for technique in ["zero_shot", "few_shot", "cot", "cot_few_shot"]:
            correct = 0
            total_tokens = 0
            for task in tasks:
                prompt = build_prompt(technique, task)
                response = await llm_client.chat(...)
                if evaluate(response, task["expected"]):
                    correct += 1
                total_tokens += response.total_tokens
            
            scores[f"{name}_{technique}"] = {
                "accuracy": correct / len(tasks),
                "avg_tokens": total_tokens / len(tasks),
            }
    return scores
```

### Drill 2: Build a Self-Consistency Evaluator

For any math/QA task, implement self-consistency with N=5. Measure:
- How often does majority vote differ from single-shot?
- When self-consistency disagrees with single-shot, which is right?
- What's the optimal temperature for your task?

### Drill 3: The "One Example" Optimization

Take a zero-shot task that's failing. Add ONE carefully chosen example.
Does it fix the issue? Why? (This reveals whether the problem was
format uncertainty vs. knowledge gap.)

---

## 🚦 Gate Check

- [ ] You can select the right technique for any task type without guessing
- [ ] You've run a technique comparison on 20+ questions
- [ ] You understand when CoT helps and when it doesn't
- [ ] You know the token cost multiplier for each technique
- [ ] You've implemented self-consistency for at least one task
- [ ] You can explain why few-shot examples work (format control + token distribution, NOT "teaching")

---

## 📚 Cross-References

- [Context Engineering](01-Context-Engineering.md) — Why these techniques work (attention mechanics)
- [System Prompts & Personas](03-System-Prompts-Personas.md) — Combining personas with techniques
- [Advanced Techniques](04-Advanced-Techniques.md) — Beyond CoT: reflection, self-critique, multi-step
- [Phase 4: RAG](../04-RAG-Foundations/) — Few-shot for retrieval-based QA
- [Phase 6: AI Agents](../06-AI-Agents/) — CoT in agent reasoning loops
- [Phase 7: Evals](../07-Production-Evals-Observability/) — Measuring technique effectiveness
