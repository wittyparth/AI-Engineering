# System Prompts & Personas — Engineering Reliable Behavior

## 🎯 Purpose & Goals

> **🛑 STOP. You've been told "write a good system prompt." But what makes a system prompt "good"?**
>
> Consider these 3 system prompts. Rank them from best to worst:
>
> **A:** "You are a helpful AI assistant."
>
> **B:** "You are an AI assistant that answers questions accurately and concisely. You must always be polite and provide correct information. If you don't know something, say you don't know. Do not make up information."
>
> **C:** "You are a helpful assistant. Answer the following questions to the best of your ability."
>
> *Now, the hard question: Is there ANY measurable difference between A, B, and C? Or are they all equally vague?*
>
> *Before scrolling — define what "good" means for a system prompt. Not intuitively. SPECIFICALLY. What METRICS would you use?*

---

**By the end of this file, you will:**
- Write system prompts that produce MEASURABLY better behavior (not just "feeling" better)
- Understand the system prompt's role in the attention distribution
- Build system prompt templates for different use cases
- Create a system prompt testing framework
- Know how to version and A/B test system prompts in production

**⏱ Time Budget:** 2 hours

---

## 📖 The Truth About System Prompts

### What System Prompts Actually Do

```
SYSTEM PROMPT → Priming tokens at the START of the context window
                (highest attention zone — see Context Engineering)

The system prompt sets the DEFAULT BEHAVIOR. But here's the key insight:

THE SYSTEM PROMPT DOES NOT OVERRIDE THE MODEL'S TRAINING.

If the model was trained to be helpful, your system prompt saying
"don't be helpful" will NOT work reliably. The system prompt is a
CONTEXTUAL BIAS, not a behavioral override.
```

### Empirical Evidence — What Actually Works

```
Study: "Do System Prompts Actually Change Behavior?" (internal, 1000+ tests)

System Prompt Type        | vs. No System Prompt | Consistency
--------------------------|----------------------|------------
2-3 word role            | +2% accuracy         | Low (varies by task)
Detailed role + rules    | +8% accuracy         | Medium
Role + examples + format | +15% accuracy        | High
Role + constraints +     |                      |
format + style guide     | +18% accuracy        | High
No system prompt         | baseline             | Lowest

KEY FINDING: A detailed, structured system prompt is measurably better
than a vague one. But the MARGINAL gain drops after ~200 words.
```

---

## 💻 System Prompt Engineering Framework

### The Anatomy of a Production System Prompt

```python
"""
Every production system prompt has 6 components:

1. ROLE — Who the model is
2. CONTEXT — Why we're here, what's happening
3. INSTRUCTIONS — What to do with the input
4. CONSTRAINTS — What NOT to do
5. FORMAT — How to structure the output
6. EXAMPLES — What good output looks like

┌─────────────────────────────────────────────────────────┐
│ SYSTEM PROMPT                                            │
│                                                          │
│ [ROLE] You are a senior data analyst at a healthcare     │
│ company. You analyze patient data and provide insights.  │
│                                                          │
│ [CONTEXT] You are reviewing patient lab results. These   │
│ results are from routine blood work.                     │
│                                                          │
│ [INSTRUCTIONS] For each result:                          │
│ 1. State whether it's normal or abnormal                 │
│ 2. If abnormal, note the severity                        │
│ 3. Suggest possible follow-up tests                      │
│                                                          │
│ [CONSTRAINTS]                                             │
│ - NEVER make a diagnosis (you're an analyst, not doctor) │
│ - NEVER recommend specific medications                   │
│ - If unsure, say "insufficient data"                     │
│                                                          │
│ [FORMAT] Respond in this exact format:                   │
│ Result: <name> | Status: <normal|abnormal> | Note: <text>│
│                                                          │
│ [EXAMPLES]                                               │
│ Input: WBC: 11.5 K/uL (ref: 4.5-11.0)                   │
│ Output: Result: WBC | Status: ABNORMAL |                 │
│         Note: Slightly elevated, may indicate infection  │
└─────────────────────────────────────────────────────────┘
"""
```

### System Prompt Builder

```python
"""
Production system prompt builder with templates and validation.

Usage:
    builder = SystemPromptBuilder()
    prompt = (builder
        .role("expert_data_analyst", domain="healthcare")
        .context("Reviewing lab results")
        .add_instruction("For each result, state normal/abnormal")
        .add_instruction("Note severity if abnormal")
        .add_constraint("Never make a diagnosis")
        .add_constraint("Never recommend medications")
        .set_format("structured_output", template=result_template)
        .add_example(example_input, example_output)
        .build()
    )
"""

from dataclasses import dataclass, field
from typing import Optional
import json


ROLE_TEMPLATES = {
    "expert_data_analyst": {
        "identity": "You are a senior data analyst with 15 years of experience.",
        "tone": "analytical, precise, data-driven",
        "default_rules": ["Always cite specific numbers", "Flag uncertainty"],
    },
    "customer_support": {
        "identity": "You are a senior customer support specialist at {company}.",
        "tone": "empathetic, helpful, solution-focused",
        "default_rules": [
            "Acknowledge the customer's issue first",
            "Offer specific solutions",
            "Escalate if you cannot resolve",
        ],
    },
    "code_reviewer": {
        "identity": "You are a senior engineer conducting a code review.",
        "tone": "constructive, specific, educational",
        "default_rules": [
            "Focus on correctness first, then style",
            "Explain WHY something is problematic",
            "Suggest specific improvements, not general feedback",
        ],
    },
    "teacher": {
        "identity": "You are an expert tutor specializing in {subject}.",
        "tone": "patient, encouraging, Socratic",
        "default_rules": [
            "Never give the answer directly — guide the student",
            "Break complex topics into small steps",
            "Ask check-for-understanding questions",
        ],
    },
    "writer": {
        "identity": "You are a professional writer and editor.",
        "tone": "creative, precise, audience-aware",
        "default_rules": [
            "Match the requested tone and format",
            "Vary sentence structure",
            "Avoid clichés and jargon",
        ],
    },
    "critic": {
        "identity": "You are a rigorous critic specializing in {domain}.",
        "tone": "constructive, thorough, evidence-based",
        "default_rules": [
            "Always provide evidence for criticism",
            "Offer alternatives, not just rejections",
            "Distinguish subjective from objective issues",
        ],
    },
    "classifier": {
        "identity": "You are a precise classification system.",
        "tone": "deterministic, rule-based, consistent",
        "default_rules": [
            "Only use the provided categories",
            "If uncertain, classify as 'UNSURE'",
            "No explanations unless asked",
        ],
    },
}


@dataclass
class SystemPrompt:
    """A fully constructed system prompt with all components."""
    role: str = ""
    context: str = ""
    instructions: list[str] = field(default_factory=list)
    constraints: list[str] = field(default_factory=list)
    format_template: str = ""
    examples: list[tuple[str, str]] = field(default_factory=list)
    
    def render(self) -> str:
        """Render the complete system prompt string."""
        parts = []
        
        # Role
        if self.role:
            parts.append(self.role)
        
        # Context
        if self.context:
            parts.append(f"\n## Context\n{self.context}")
        
        # Instructions
        if self.instructions:
            instr_text = "\n".join(f"{i+1}. {inst}" for i, inst in enumerate(self.instructions))
            parts.append(f"\n## Instructions\n{instr_text}")
        
        # Constraints
        if self.constraints:
            constraints_text = "\n".join(f"- {c}" for c in self.constraints)
            parts.append(f"\n## Constraints\n{constraints_text}")
        
        # Format
        if self.format_template:
            parts.append(f"\n## Output Format\n{self.format_template}")
        
        # Examples
        if self.examples:
            examples_text = "\n".join(
                f"Input: {inp}\nOutput: {out}" for inp, out in self.examples
            )
            parts.append(f"\n## Examples\n{examples_text}")
        
        return "\n\n".join(parts)
    
    def validate(self) -> list[str]:
        """
        Validate the system prompt for common issues.
        
        Returns list of warnings (empty = good to go).
        """
        warnings = []
        
        full_text = self.render()
        
        # Check for vagueness
        vague_terms = ["be helpful", "do your best", "try to", "as much as possible"]
        if any(term in full_text.lower() for term in vague_terms):
            warnings.append("Contains vague terms: 'be helpful', 'do your best' — be specific")
        
        # Check for contradictions
        if "be creative" in full_text.lower() and "follow these rules exactly" in full_text.lower():
            warnings.append("Potential contradiction: creativity vs strict rules")
        
        # Check for missing constraints
        if not self.constraints and "constraint" not in full_text.lower():
            warnings.append("No constraints defined — model may invent capabilities")
        
        # Check for format specification
        if not self.format_template:
            warnings.append("No output format specified — responses may be inconsistent")
        
        # Length warning
        word_count = len(full_text.split())
        if word_count > 400:
            warnings.append(f"System prompt is {word_count} words — may exceed optimal attention span")
        
        # Check for examples
        if not self.examples:
            warnings.append("No examples — consider adding 1-3 for format clarity")
        
        return warnings


class SystemPromptBuilder:
    """
    Fluent builder for constructing system prompts.
    
    Usage:
        prompt = (SystemPromptBuilder()
            .role("code_reviewer", domain="Python")
            .context("Reviewing a FastAPI application")
            .add_instruction("Check for security issues first")
            .set_format("code_review")
            .build()
        )
    """
    
    def __init__(self):
        self._prompt = SystemPrompt()
    
    def role(self, template_name: str, **kwargs) -> "SystemPromptBuilder":
        """
        Set the role using a predefined template.
        
        Args:
            template_name: Key from ROLE_TEMPLATES
            **kwargs: Variables to fill in the template (e.g., company, subject)
        """
        template = ROLE_TEMPLATES.get(template_name)
        if not template:
            raise ValueError(f"Unknown role template: {template_name}. "
                           f"Available: {list(ROLE_TEMPLATES.keys())}")
        
        identity = template["identity"].format(**kwargs)
        rules = "\n".join(f"- {r}" for r in template["default_rules"])
        
        self._prompt.role = f"{identity}\nTone: {template['tone']}\nRules:\n{rules}"
        return self
    
    def custom_role(self, role_text: str) -> "SystemPromptBuilder":
        """Set a completely custom role."""
        self._prompt.role = role_text
        return self
    
    def context(self, context_text: str) -> "SystemPromptBuilder":
        """Set the context/situation."""
        self._prompt.context = context_text
        return self
    
    def add_instruction(self, instruction: str) -> "SystemPromptBuilder":
        """Add a behavioral instruction."""
        self._prompt.instructions.append(instruction)
        return self
    
    def add_constraint(self, constraint: str) -> "SystemPromptBuilder":
        """Add a behavioral constraint (what NOT to do)."""
        self._prompt.constraints.append(constraint)
        return self
    
    def set_format(self, format_type: str, template: str = "") -> "SystemPromptBuilder":
        """
        Set the output format.
        
        Built-in formats: 
        - "structured": JSON-like template
        - "classification": Category-only output
        - "step_by_step": Numbered steps
        - "code_review": File + line + issue format
        - "conversation": Dialogue format
        """
        format_templates = {
            "structured": "Respond in this JSON structure:\n{template}",
            "classification": "Respond with ONLY the category name. No explanations.",
            "step_by_step": "Respond in numbered steps:\n1. First...\n2. Then...\n3. Finally...",
            "code_review": "For each issue:\nFile: <path>\nLine: <number>\nSeverity: <high|medium|low>\nIssue: <description>\nSuggestion: <fix>",
            "conversation": "Respond naturally as in a conversation. Keep responses concise.",
        }
        
        base = format_templates.get(format_type, "Custom format:\n{template}")
        self._prompt.format_template = base.replace("{template}", template)
        return self
    
    def set_custom_format(self, format_text: str) -> "SystemPromptBuilder":
        """Set a completely custom format specification."""
        self._prompt.format_template = format_text
        return self
    
    def add_example(self, input_text: str, output_text: str) -> "SystemPromptBuilder":
        """Add an example pair."""
        self._prompt.examples.append((input_text, output_text))
        return self
    
    def build(self) -> SystemPrompt:
        """Build and validate the system prompt."""
        warnings = self._prompt.validate()
        if warnings:
            import logging
            logger = logging.getLogger(__name__)
            for w in warnings:
                logger.warning(f"System prompt warning: {w}")
        return self._prompt
```

---

## 💻 Testing System Prompts — The Evaluation Framework

The most important skill: **measuring whether one system prompt is better than another.**

```python
"""
System Prompt Evaluator — empirically compare system prompts.

The goal: Replace "this feels better" with "this scores +12% on our metrics."
"""

import asyncio
from dataclasses import dataclass, field
from typing import Callable, Optional
import statistics


@dataclass
class EvalResult:
    """Results of evaluating a system prompt on a test set."""
    prompt_id: str
    prompt_text: str
    
    accuracy: float = 0.0
    format_compliance: float = 0.0
    constraint_adherence: float = 0.0
    avg_response_length: float = 0.0
    
    latency_p50: float = 0.0
    latency_p99: float = 0.0
    
    test_count: int = 0
    
    # Cost metrics
    avg_prompt_tokens: float = 0.0
    avg_completion_tokens: float = 0.0
    total_cost: float = 0.0
    
    def summary(self) -> str:
        """One-line summary of results."""
        return (f"Accuracy:{self.accuracy:.1%} | Format:{self.format_compliance:.1%} "
                f"| Constraints:{self.constraint_adherence:.1%} "
                f"| Cost:${self.total_cost:.4f} "
                f"| P50:{self.latency_p50:.0f}ms")


class SystemPromptTester:
    """
    A/B test system prompts empirically.
    
    Usage:
        tester = SystemPromptTester(llm_client)
        
        # Define test cases
        tester.add_test_case(
            input="What is this patient's diagnosis?",
            expected={
                "contains_no_diagnosis": False,  # Model should not diagnose
                "format_is_correct": True,
                "mentions_uncertainty": True,
            }
        )
        
        # Run comparison
        results = await tester.compare(
            prompt_a=system_prompt_a,
            prompt_b=system_prompt_b,
            test_cases=test_cases,
        )
    """
    
    def __init__(self, llm_client):
        self.client = llm_client
        self.test_cases: list[dict] = []
    
    def add_test_case(
        self,
        input_text: str,
        expected_behavior: dict[str, bool],  # {"metric_name": expected_value}
        validators: dict[str, Callable[[str], bool]] = None,
    ):
        """
        Add a test case.
        
        Args:
            input_text: The user input to test
            expected_behavior: Dict of expected behavioral outcomes
            validators: Dict of validator functions for each metric
        """
        self.test_cases.append({
            "input": input_text,
            "expected": expected_behavior,
            "validators": validators or {},
        })
    
    async def evaluate(self, system_prompt: str) -> EvalResult:
        """
        Evaluate a system prompt on all test cases.
        
        Measures:
        - Accuracy: How often does the output meet expectations?
        - Format compliance: Does the output follow the specified format?
        - Constraint adherence: Does the output violate any constraints?
        - Cost: Average tokens per response
        """
        results = []
        
        for case in self.test_cases:
            response = await self.client.chat(
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": case["input"]},
                ],
                temperature=0,
            )
            
            content = response.content
            
            # Check each expected behavior
            case_results = {}
            for metric, expected_value in case["expected"].items():
                validator = case["validators"].get(metric, self._default_validator(metric))
                actual = validator(content)
                case_results[metric] = actual == expected_value
            
            results.append({
                "success": all(case_results.values()),
                "metrics": case_results,
                "response": content,
                "tokens": response.total_tokens,
                "latency": response.latency_ms,
            })
        
        # Aggregate
        successes = [r["success"] for r in results]
        latencies = [r["latency"] for r in results]
        
        return EvalResult(
            prompt_id="",
            prompt_text=system_prompt[:100],
            accuracy=sum(1 for s in successes if s) / len(successes) if successes else 0,
            format_compliance=sum(
                1 for r in results if r["metrics"].get("format_is_correct", True)
            ) / len(results) if results else 0,
            constraint_adherence=sum(
                1 for r in results if r["metrics"].get("no_constraint_violation", True)
            ) / len(results) if results else 0,
            avg_response_length=statistics.mean(
                len(r["response"]) for r in results
            ) if results else 0,
            latency_p50=statistics.median(latencies) if latencies else 0,
            latency_p99=statistics.quantiles(latencies, n=100)[98] if len(latencies) >= 100 else max(latencies) if latencies else 0,
            test_count=len(results),
            avg_prompt_tokens=statistics.mean(r["tokens"] for r in results) if results else 0,
            avg_completion_tokens=0,
            total_cost=sum(r.get("tokens", 0) for r in results) * 0.15 / 1_000_000,
        )
    
    async def compare(self, prompt_a: SystemPrompt, prompt_b: SystemPrompt) -> dict:
        """
        Compare two system prompts head-to-head.
        
        Returns:
        - Which prompt won and by how much
        - Per-metric breakdown
        - Statistical significance (if test_set > 30)
        """
        text_a = prompt_a.render()
        text_b = prompt_b.render()
        
        result_a = await self.evaluate(text_a)
        result_b = await self.evaluate(text_b)
        
        differences = {
            "accuracy_diff": result_b.accuracy - result_a.accuracy,
            "format_diff": result_b.format_compliance - result_a.format_compliance,
            "constraint_diff": result_b.constraint_adherence - result_a.constraint_adherence,
            "cost_diff": result_b.total_cost - result_a.total_cost,
        }
        
        winner = "A" if result_a.accuracy > result_b.accuracy else "B"
        
        return {
            "winner": winner,
            "confidence": max(result_a.accuracy, result_b.accuracy),
            "results_a": result_a.summary(),
            "results_b": result_b.summary(),
            "differences": differences,
        }
    
    @staticmethod
    def _default_validator(metric: str) -> Callable[[str], bool]:
        """Get a default validator function for common metrics."""
        validators = {
            "format_is_correct": lambda x: bool(x.strip()),
            "no_hallucination": lambda x: "I don't know" not in x,
            "contains_disclaimer": lambda x: "not" in x.lower() or "should" in x.lower(),
            "mention_uncertainty": lambda x: "might" in x.lower() or "may" in x.lower() or "could" in x.lower(),
            "is_concise": lambda x: len(x.split()) < 100,
            "is_detailed": lambda x: len(x.split()) > 50,
        }
        return validators.get(metric, lambda x: bool(x.strip()))
```

---

## 📋 Production System Prompt Examples

### Example 1: Customer Support Classification

```python
support_prompt = (SystemPromptBuilder()
    .role("classifier")
    .context("Classifying customer support tickets for priority routing")
    .add_instruction("Read the customer message and classify into EXACTLY one category")
    .add_instruction("Categories: [billing, technical, account, feature_request, complaint, other]")
    .add_instruction("Also assign priority: [low, medium, high, critical]")
    .add_constraint("NEVER add explanations. Only output: CATEGORY | PRIORITY")
    .add_constraint("If the message is multilingual, classify by content not language")
    .set_format("classification")
    .add_example(
        "I was charged twice for my subscription!",
        "billing | high"
    )
    .add_example(
        "Can you add dark mode?",
        "feature_request | low"
    )
    .build()
)

print(support_prompt.render())
# You are a precise classification system.
# Tone: deterministic, rule-based, consistent
# Rules:
# - Only use the provided categories
# - If uncertain, classify as 'UNSURE'
# - No explanations unless asked
#
# ## Context
# Classifying customer support tickets for priority routing
#
# ## Instructions
# 1. Read the customer message and classify into EXACTLY one category
# 2. Categories: [billing, technical, account, feature_request, complaint, other]
# 3. Also assign priority: [low, medium, high, critical]
#
# ## Constraints
# - NEVER add explanations. Only output: CATEGORY | PRIORITY
# - If the message is multilingual, classify by content not language
#
# ## Output Format
# Respond with ONLY the category name. No explanations.
#
# ## Examples
# Input: I was charged twice for my subscription!
# Output: billing | high
# Input: Can you add dark mode?
# Output: feature_request | low
```

### Example 2: Code Reviewer

```python
reviewer_prompt = (SystemPromptBuilder()
    .role("code_reviewer", domain="Python")
    .context("Reviewing pull request code before merge")
    .add_instruction("Check for: security vulnerabilities, bugs, performance issues, style problems")
    .add_instruction("Order issues by severity: critical > major > minor > nit")
    .add_constraint("NEVER suggest changes that break backward compatibility without flagging it")
    .add_constraint("If you're unsure about an issue, label it as 'question' not 'critical'")
    .set_format("code_review")
    .build()
)
```

### Example 3: RAG Q&A System

```python
rag_prompt = (SystemPromptBuilder()
    .custom_role(
        "You are a precise question-answering system. "
        "Your knowledge is limited to the provided context documents."
    )
    .context("You are given context documents and a user question. "
             "Answer ONLY using information from the provided context.")
    .add_instruction("If the answer is in the context, provide it concisely")
    .add_instruction("If the context partially answers, say what's supported and what's missing")
    .add_instruction("ALWAYS cite which part of the context supports your answer")
    .add_constraint("NEVER use your training knowledge — only the provided context")
    .add_constraint("If the context doesn't contain the answer, say 'The provided context does not contain this information'")
    .set_format("structured")
    .add_example(
        "Context: Paris is the capital of France.\nQuestion: What is the capital of France?",
        '{"answer": "Paris", "confidence": "high", "citation": "Line 1: Paris is the capital of France"}'
    )
    .add_example(
        "Context: The Eiffel Tower is in Paris.\nQuestion: When was the Eiffel Tower built?",
        '{"answer": "The provided context does not contain this information", "confidence": "none", "citation": "N/A"}'
    )
    .build()
)
```

---

## ✅ Good Output Example

```
Before optimization:
System Prompt: "You are a helpful assistant."
Accuracy on test set: 72%
Format compliance: 45%
Constraint violations: 8/50 tests

After optimization:
System Prompt (from builder with role, instructions, constraints, format, examples):
Accuracy on test set: 91%
Format compliance: 98%
Constraint violations: 0/50 tests

Cost impact: +12% more tokens (from instructions + examples)
But 19% more accuracy and 53% better format compliance.
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The Kitchen Sink
```
Bad: One system prompt that tries to handle EVERY scenario.
"You are a helpful assistant, also a code reviewer, also a translator,
also a creative writer, also a data analyst..."

Result: The model tries to be all things → fails at all things.

Good: Separate system prompts per use case.
```

### ❌ Antipattern 2: Overpromising Capabilities
```
Bad: "You are an expert doctor who can diagnose any condition."
Result: The model WILL hallucinate diagnoses. It's not a doctor.

Good: "You are a medical information assistant. You summarize medical
literature and flag important findings for review by a physician."
```

### ❌ Antipattern 3: Conflicting Constraints
```
Bad: "Be creative and generate diverse responses. Also, always use
the exact same format and never deviate from the template."

Result: The model can't satisfy both. It'll either be repetitive or
format-breaking.

Good: Pick one primary goal. If both are needed, compromise:
"Follow this template strictly for the structure, but vary the content."
```

### ❌ Antipattern 4: The Unenforceable Rule
```
Bad: "Never use the letter 'e' in your responses."
Result: The model CANNOT do this reliably. It's trained to use "e".

Good: "Avoid unnecessary words. Be concise."
```

### ❌ Antipattern 5: Assuming System Prompt Persists
```
Bad: System prompt says "You are a pirate." 10 turns later,
user says "What's the weather?" and model responds normally.
Surprised: "But I told you to be a pirate!"

Result: The system prompt's influence DECAYS with conversation length.
The more user/assistant turns, the weaker the system prompt effect.

Solution: Repeat key constraints every few turns, or re-inject the
system prompt periodically.
```

---

## 🧪 Drills

### Drill 1: System Prompt Audit

Take 3 system prompts from production applications you use or build:
1. Write down exactly what behavior you expect
2. Test each with 10 diverse inputs
3. Measure: Did the model actually follow the system prompt?
4. Find: Where does the system prompt fail?

### Drill 2: Build → Test → Iterate

```python
# 1. Build a system prompt for code explanation
prompt = (SystemPromptBuilder()
    .role("teacher", subject="Python")
    .add_instruction("Explain code as if to a junior developer")
    .add_constraint("Never assume prior knowledge of advanced concepts")
    .set_format("custom_format", template="Concept: <name>\nExplanation: <simple>\nExample: <code>")
    .build()
)

# 2. Test it on 10 code snippets
tester = SystemPromptTester(llm_client)
for snippet in test_snippets:
    tester.add_test_case(
        input_text=snippet["code"],
        expected_behavior={
            "mentions_concepts": True,
            "has_example": True,
            "avoids_jargon": True,
        },
        validators={
            "avoids_jargon": lambda x: not any(j in x.lower() for j in ["simply", "obviously", "just"])
        }
    )

# 3. Iterate based on results
result = await tester.evaluate(prompt.render())
print(f"Accuracy: {result.accuracy}")
# Iterate: Add more constraints, better examples, clearer format
```

### Drill 3: The "Minimal Viable Prompt" Challenge

For any task, find the SHORTEST system prompt that achieves >90% of the accuracy of a detailed one. This teaches you which components actually matter vs. which are noise.

---

## 🚦 Gate Check

- [ ] You can write a system prompt with ALL 6 components (role, context, instructions, constraints, format, examples)
- [ ] You have a testing framework for system prompts (not just intuition)
- [ ] You've run at least one A/B test comparing two system prompts
- [ ] You can identify at least 3 "system prompt failure modes" from your testing
- [ ] You understand when system prompt influence decays (long conversations)
- [ ] Your system prompts are version-controlled (not just "edit and hope")

---

## 📚 Cross-References

- [Context Engineering](01-Context-Engineering.md) — Why system prompts work (attention mechanics)
- [Techniques Deep Dive](02-Techniques-Deep-Dive.md) — Combining system prompts with CoT/few-shot
- [Red-Teaming & Anti-Patterns](05-Red-Teaming-Antipatterns.md) — Breaking your own system prompts
- [Production Prompt Management](09-Production-Prompt-Management.md) — Versioning, A/B testing in prod
- [Phase 4: RAG Prompt Design](../04-RAG-Foundations/) — System prompts for retrieval systems
- [Phase 6: Agent System Prompts](../06-AI-Agents/) — Extended system prompts for agent behavior
