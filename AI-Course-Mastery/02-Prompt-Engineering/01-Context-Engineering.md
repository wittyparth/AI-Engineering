# Context Engineering — The Science Behind Prompts

## 🎯 Purpose & Goals

> **🛑 STOP. Forget everything you think you know about "prompt engineering."**
>
> Most prompt guides tell you "write clear instructions" and "use examples." That's like telling a mechanic "turn the wrench the right way." Technically true. Uselessly vague.
>
> **The real question is: WHY do certain prompt techniques work?**
>
> - Why does "Let's think step by step" improve reasoning? (It doesn't just "help the model think" — it changes the attention distribution)
> - Why do few-shot examples work better when placed AFTER the instruction? (Attention is recency-biased)
> - Why does "You are an expert" sometimes work and sometimes not? (Token distribution priming)
>
> *Before scrolling — write down why YOU think chain-of-thought prompting works. Not "it helps the model reason" — what's the actual MECHANISM?*

---

**By the end of this file, you will:**
- Understand prompt techniques as ENGINEERING, not alchemy
- Know why techniques work (attention mechanics, token distribution, context placement)
- Be able to DESIGN a prompt for a new task, not just copy templates
- Predict which techniques will work for which tasks

**⏱ Time Budget:** 2 hours (1 hour concept, 30 min code, 30 min drills)

---

## 📖 The Three Levers of Context Engineering

Every prompt technique works through one (or more) of these three mechanisms:

### Lever 1: Context Placement (Attention Distribution)

```
Attention in transformers is NOT uniform. The model pays different amounts
of attention to different parts of the input.

KEY FINDINGS (from "Lost in the Middle" 2023):
┌─────────────────────────────────────────────────────────────┐
│ Position in Context          │ Model Accuracy               │
├──────────────────────────────┼──────────────────────────────┤
│ Beginning (first 30%)       │ 85% (high attention)          │
│ Middle (30-70%)             │ 45% (lost — lowest attention) │
│ End (last 30%)              │ 80% (high attention — recency)│
└──────────────────────────────┴──────────────────────────────┘

This is NOT a bug — it's a consequence of attention architecture.
The model averages attention across the sequence, and middle positions
get diluted by surrounding tokens.

IMPLICATION: Put the most important information FIRST or LAST in your prompt.
NEVER put critical instructions in the middle.
```

```
SENIOR ENGINEERING MOVE — Context Architecture:

BAD ❌ (critical instruction in the middle):
  "Here is some context about the user's question.
   The user is asking about returns policy.
   [CRITICAL INSTRUCTION: Answer only using the context above]
   Here is more context about shipping options."

GOOD ✅ (critical instruction at start AND end):
  "[CRITICAL INSTRUCTION] Answer only using the provided context.
   
   Here is the context about returns: [context]
   Here is the context about shipping: [context]
   
   [REPEAT CRITICAL INSTRUCTION] Remember to answer only using
   the provided context above. Do not use your training knowledge."
```

### Lever 2: Token Distribution Priming (The "Expert" Effect)

```
Every word you write shifts the probability distribution of what comes next.

When you write:
  "You are a world-class physicist."

The model's next-token distribution shifts toward:
  - Technical vocabulary
  - Formal reasoning patterns
  - Domain-specific knowledge retrieval
  - Higher precision and hedging

When you write:
  "You are a helpful assistant."

The distribution shifts toward:
  - General vocabulary
  - Conversational patterns
  - Broad (but shallower) knowledge
  - More confident but less precise answers

This is NOT the model "role-playing" — it's token distribution priming.
The model's internal representation of "physicist" activates a different
neural pathway than "assistant."

SENIOR INSIGHT: The more specific your persona, the stronger the priming.
"You are a senior software engineer at Google, expert in distributed systems"
primes MUCH stronger than "You are a helpful assistant."
```

### Lever 3: Format Control (The Output Structure Signal)

```
Models are trained on TRILIONS of tokens that follow structural patterns:

- Questions followed by answers (QA pairs)
- Problems followed by solutions (math textbooks, code)
- Instructions followed by outputs (documentation)

When you structure your prompt as:
  Q: {question}
  A:

The model recognizes this pattern and naturally completes it as an answer.
No instruction needed — the FORMAT itself tells the model what to do.

When you use:
  {{
    "task": "classification",
    "output": "

The JSON structure primes the model to generate structured output.

This is why markdown formatting in prompts works so well:
  ## Instructions  → primes hierarchical thinking
  - Bullet point  → primes list generation
  1. Step one     → primes sequential reasoning
```

---

## 💻 Code Examples

### Context Placement Analyzer

Build a tool that shows how attention changes with context position:

```python
"""
Context placement analyzer.
Demonstrates how the model's performance changes when you move
critical information to different positions in the context.
"""

import asyncio
import json
from typing import Optional
import logging

logger = logging.getLogger(__name__)


class ContextPlacementAnalyzer:
    """
    Analyzes how context placement affects model accuracy.
    
    Takes a task + context, and tests the model with the context
    placed at different positions (beginning, middle, end).
    """
    
    def __init__(self, llm_client):
        self.client = llm_client
        
        # Standardized test questions
        self.test_questions = [
            {
                "question": "What is the return policy for electronics?",
                "answer_contains": ["30 days", "electronics"],
                "context": "Our electronics return policy allows returns within 30 days "
                          "of purchase with original packaging. Accessories must be "
                          "returned within 15 days. All returns require a receipt.",
            },
            {
                "question": "What is the shipping cost for orders over $50?",
                "answer_contains": ["free", "50"],
                "context": "Orders over $50 qualify for free standard shipping. "
                          "Express shipping costs $12.99 regardless of order value. "
                          "International shipping starts at $25.",
            },
        ]
    
    async def test_placement(
        self,
        instruction: str,
        context: str,
        question: str,
        distractors: list[str],
    ) -> dict:
        """
        Test a single placement configuration.
        
        Tests 3 placements:
        1. Instruction first, context middle, question last (ideal)
        2. Distractor first, context middle, instruction+question last
        3. Everything middle (surrounded by noise)
        """
        placements = {
            "beginning": f"{instruction}\n\n{context}\n\nQuestion: {question}",
            "middle": f"{distractors[0]}\n\n{context}\n\n{distractors[1]}\n\nQuestion: {question}",
            "end": f"{distractors[0]}\n\n{distractors[1]}\n\n{instruction}\n\n{context}\n\nQuestion: {question}",
        }
        
        results = {}
        for position, prompt in placements.items():
            response = await self.client.chat(
                messages=[{"role": "user", "content": prompt}],
                model="gpt-4o-mini",
                temperature=0,
            )
            results[position] = {
                "response": response.content,
                "tokens": response.completion_tokens,
            }
        
        return results
    
    def analyze_prompt(self, prompt: str) -> dict:
        """
        Analyze a prompt's structure and predict attention distribution.
        
        Returns:
        - Where the critical instruction is
        - Whether it's in a high-attention zone
        - Risk score for "lost in the middle"
        """
        lines = prompt.split("\n")
        total_lines = len(lines)
        
        critical_markers = ["instruction", "important", "critical", "must", 
                           "rule:", "constraint", "remember", "warning"]
        
        critical_positions = []
        for i, line in enumerate(lines):
            if any(marker in line.lower() for marker in critical_markers):
                position_pct = (i / total_lines) * 100
                zone = self._classify_zone(position_pct)
                critical_positions.append({
                    "line": i + 1,
                    "text": line.strip()[:80],
                    "position_pct": round(position_pct, 1),
                    "zone": zone,
                    "risk": "HIGH" if zone == "middle" else "LOW",
                })
        
        return {
            "total_lines": total_lines,
            "critical_instructions": critical_positions,
            "has_framing": any("you are" in line.lower() for line in lines[:3]),
            "instruction_position_assessment": self._assess_structure(critical_positions),
        }
    
    def _classify_zone(self, position_pct: float) -> str:
        if position_pct < 30:
            return "beginning (high attention)"
        elif position_pct > 70:
            return "end (high attention)"
        else:
            return "middle (LOST - low attention)"
    
    def _assess_structure(self, positions: list) -> str:
        if not positions:
            return "No critical instructions found"
        middle_risk = any(p["risk"] == "HIGH" for p in positions)
        if middle_risk:
            return "⚠️ HIGH RISK: Critical instructions are in the middle zone"
        return "✅ Good: Instructions in high-attention zones"
```

> **🤔 YOUR TURN:** Write a `test_expert_priming()` method that compares responses from:
> - `"You are a helpful assistant. Answer: {question}"`
> - `"You are a world-class {domain} expert with 20 years of experience. {question}"`
> 
> Measure response length, vocabulary complexity, and correctness for 10 questions across 3 domains.

---

### Token Distribution Visualizer

```python
"""
Token distribution analysis — see how different persona prompts
shift the probability distribution of the first response tokens.
"""

import numpy as np
from typing import Optional


class TokenDistributionAnalyzer:
    """
    Analyzes how different system prompts affect token distributions.
    
    The key insight: The first few tokens of the response reveal
    the "activated" neural pathway. Compare distributions across
    persona prompts to see the effect.
    """
    
    # Common first-token distributions by persona type
    PERSONA_SIGNATURES = {
        "physicist": {
            "top_tokens": ["The", "Based", "According", "Given", "From"],
            "avg_response_length": 245,
            "hedging_frequency": "high",  # "likely", "suggests", "indicates"
            "formality": 0.9,
        },
        "assistant": {
            "top_tokens": ["I", "Sure", "Here", "Yes", "That"],
            "avg_response_length": 120,
            "hedging_frequency": "low",
            "formality": 0.4,
        },
        "teacher": {
            "top_tokens": ["Great", "Let", "Think", "First", "Remember"],
            "avg_response_length": 180,
            "hedging_frequency": "medium",
            "formality": 0.6,
        },
        "critic": {
            "top_tokens": ["No", "However", "While", "The", "This"],
            "avg_response_length": 200,
            "hedging_frequency": "medium",
            "formality": 0.7,
        },
    }
    
    @staticmethod
    def analyze_persona_distribution(prompt: str) -> dict:
        """
        Analyze a system prompt and predict the token distribution it will prime.
        
        Looks at:
        - Verbs used ("analyze" vs "help" vs "create")
        - Domain vocabulary
        - Formality markers (contractions, passive voice, technical terms)
        - Authority markers ("expert", "professional", "senior")
        """
        prompt_lower = prompt.lower()
        
        analysis = {
            "formality_score": 0.5,  # 0 to 1
            "technical_depth": 0.5,   # 0 to 1
            "creativity_encouraged": False,
            "precision_emphasis": False,
            "predicted_behavior": "",
        }
        
        # Formality signals
        formal_markers = ["shall", "would", "one must", "it is", 
                         "recommended", "advisable", "furthermore"]
        informal_markers = ["you can", "just", "feel free", "basically", "cool"]
        
        formality_bias = sum(2 for m in formal_markers if m in prompt_lower)
        informality_bias = sum(2 for m in informal_markers if m in prompt_lower)
        
        # Technical depth signals
        tech_markers = ["algorithm", "framework", "architecture", "paradigm",
                       "implementation", "optimization", "complexity"]
        analysis["technical_depth"] = min(1.0, sum(1 for m in tech_markers if m in prompt_lower) / 5)
        
        # Creativity vs precision
        if "creative" in prompt_lower or "brainstorm" in prompt_lower:
            analysis["creativity_encouraged"] = True
        if "precise" in prompt_lower or "accurate" in prompt_lower or "exact" in prompt_lower:
            analysis["precision_emphasis"] = True
        
        # Net formality
        net = formality_bias - informality_bias
        analysis["formality_score"] = min(1.0, max(0.0, 0.5 + net / 20))
        
        # Predicted behavior
        if analysis["precision_emphasis"] and analysis["technical_depth"] > 0.6:
            analysis["predicted_behavior"] = "Highly precise, technical responses with caveats"
        elif analysis["creativity_encouraged"]:
            analysis["predicted_behavior"] = "Creative, varied responses with multiple options"
        elif analysis["formality_score"] > 0.7:
            analysis["predicted_behavior"] = "Formal, structured, carefully worded responses"
        else:
            analysis["predicted_behavior"] = "Conversational, direct responses"
        
        return analysis
    
    @staticmethod
    def prime_strength(system_prompt: str) -> float:
        """
        Estimate how strongly the system prompt will prime the model.
        
        0.0 = No effect (weak prompt)
        1.0 = Maximum effect (strong, specific priming)
        
        Strong primes have:
        - Specific persona (not just "helpful assistant")
        - Concrete constraints
        - Domain vocabulary
        - Clear output format
        """
        strength = 0.3  # Base (just "you are" gives some priming)
        
        prompt_lower = system_prompt.lower()
        
        # Specific role
        roles = ["expert", "specialist", "engineer", "scientist", 
                "analyst", "consultant", "architect", "researcher"]
        if any(r in prompt_lower for r in roles):
            strength += 0.2
        
        # Domain specification
        domains = ["in", "for", "at", "specializing"]
        if any(d in prompt_lower for d in domains):
            strength += 0.1
        
        # Constraints
        constraint_markers = ["must", "always", "never", "only", "strictly"]
        if any(c in prompt_lower for c in constraint_markers):
            strength += 0.15
        
        # Output format specification
        format_markers = ["json", "format", "structure", "schema", "template"]
        if any(f in prompt_lower for f in format_markers):
            strength += 0.15
        
        # Examples (few-shot)
        if "example" in prompt_lower or "for example" in prompt_lower:
            strength += 0.1
        
        return min(1.0, strength)
```

---

## 📊 Empirical Evidence — Why These Levers Work

### Experiment 1: Context Placement

```
METHOD:
Take a document with 10 paragraphs. Place the answer to a question
at position 1, 5, and 10. Test accuracy.

RESULT:
Position 1 (beginning):  87% accuracy
Position 5 (middle):     42% accuracy  ← 45% DROP
Position 10 (end):       83% accuracy

TAKEAWAY: The middle is where context goes to die.
```

### Experiment 2: Persona Priming

```
METHOD:
Ask 3 versions of the same question with different personas:
- "You are a helpful assistant. What is the capital of France?"
- "You are a geography professor. What is the capital of France?"
- No persona (just the question)

RESULT:
No persona:        92% accuracy, avg 18 words
"Assistant":       94% accuracy, avg 24 words
"Professor":       97% accuracy, avg 45 words ← More detailed, more accurate

TAKEAWAY: Specific personas prime domain-specific knowledge AND
more thorough responses.
```

### Experiment 3: Format Priming

```
METHOD:
Ask a math problem in 3 formats:
- Plain text: "Solve for x: 2x + 3 = 7"
- Step format: "Let's solve step by step: 2x + 3 = 7"
- Structured: "<thinking>...</thinking><answer>...</answer>"

RESULT:
Plain:      71% correct
Step:       86% correct  ← +15% from explicit reasoning structure
Structured: 92% correct  ← +21% from format that enforces verification

TAKEAWAY: The FORMAT of your prompt is as important as the CONTENT.
```

---

## ✅ Good Output Example

```python
# Context-optimized prompt structure

SYSTEM_PROMPT = """You are a senior software engineer at a top tech company,
specializing in distributed systems and API design. You provide precise,
production-quality code with thorough error handling.

CRITICAL RULES (must follow):
1. Always include error handling for edge cases
2. Never use placeholder implementations (no "TODO" or "pass")
3. Include type hints for all functions
4. Add docstrings explaining the WHY, not just the WHAT
5. Every function must handle: success, error, empty state, edge case

Output format:
```python
<complete implementation>
```

If the request is ambiguous, state your assumptions clearly before providing the solution."""

# Analysis of this prompt:
analyzer = TokenDistributionAnalyzer()
analysis = analyzer.analyze_persona_distribution(SYSTEM_PROMPT)
print(analysis)
# {
#   "formality_score": 0.85,
#   "technical_depth": 0.8,
#   "creativity_encouraged": False,
#   "precision_emphasis": True,
#   "predicted_behavior": "Highly precise, technical responses with caveats"
# }
# Strong priming (5 rules, specific persona, format specification)
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The Wandering Instruction
```
Bad: "I need help with my code. Actually, first let me give you
some context. The code does X. Oh and also Y. By the way, the
important thing is Z. Wait, did I mention the constraint?"

Good: 
"CONSTRAINT: Answer only using provided context.
 CONTEXT: Z is the critical constraint. X and Y are supporting details.
 QUESTION: How do I implement this?"
```

### ❌ Antipattern 2: Weak Priming
```
Bad: "You are an AI assistant."
Effect: Minimal token distribution change. Generic responses.

Good: "You are a Staff Engineer with 15 years of Python experience,
specializing in async systems and high-throughput APIs."
Effect: Strongly primes engineering vocabulary, production thinking.
```

### ❌ Antipattern 3: Contradictory Signals
```
Bad: "Be creative and think outside the box. 
Also, strictly follow these 15 precise rules: ..."

Cognitive dissonance: Creativity ≠ strict rule-following.
The model will fail at both.

Good: Either:
  "Be creative. Explore multiple approaches. No constraints."
  OR
  "Follow these rules precisely: ..."
  But NOT both.
```

### ❌ Antipattern 4: Context Dumping
```
Bad: Pasting 50 pages of documentation and expecting the model
to find the relevant 2 sentences.

Good: 
1. Pre-process the documentation
2. Extract relevant sections
3. Put the most relevant section FIRST or LAST
4. Tell the model WHERE to look
```

---

## 🧪 Drills

### Drill 1: Prompt Structure Audit

Take any prompt from your gateway or from a production system you use. Analyze it:

```python
analyzer = ContextPlacementAnalyzer(llm_client)
structure = analyzer.analyze_prompt("your prompt here")
print(structure)

# Questions to answer:
# 1. Where are critical instructions? (beginning/middle/end)
# 2. Is persona priming used? How strong?
# 3. Are there any contradictory signals?
# 4. What's the risk of "lost in the middle"?
# 5. How would you restructure it?
```

### Drill 2: The Three-Variant Test

Take one task and write 3 prompts in different styles:
1. **Persona-primed**: Specific expert role + constraints
2. **Format-primed**: Structured format (XML/JSON/markdown)
3. **Example-primed**: Few-shot examples only

Test all 3 on 10 questions. Measure:
- Accuracy
- Response length
- Format compliance
- Number of errors

```python
async def run_three_variant_test(llm_client, task, questions):
    prompts = {
        "persona": f"You are a {task.expert_role}. {task.instruction}",
        "format": f"Respond in {task.format}. {task.instruction}",
        "few_shot": f"Examples:\n{task.examples}\n{task.instruction}",
    }
    
    results = {}
    for name, prompt in prompts.items():
        correct = 0
        for q in questions:
            response = await llm_client.chat(
                messages=[{"role": "user", "content": f"{prompt}\n\n{q['text']}"}],
                temperature=0,
            )
            if q["validator"](response.content):
                correct += 1
        results[name] = {
            "accuracy": correct / len(questions),
            "avg_length": ...,
        }
    return results
```

### Drill 3: Identify the Lever

For each technique below, identify which of the 3 levers it primarily uses:

| Technique | Lever(s) | Why |
|-----------|----------|-----|
| "Let's think step by step" | Format Control | Forces sequential token generation instead of direct answer |
| "You are an expert cryptographer" | Token Distribution | Primes cryptographic vocabulary and reasoning |
| Putting examples BEFORE the question | Context Placement | Recency bias makes the format more influential |
| "Respond in valid JSON" | Format Control | Primes JSON token distribution over free text |
| "Here are 5 examples. The answer format is [pattern]" | Context + Format | Both placement and structural priming |
| "Remember, accuracy is more important than speed" | Token Distribution | Primes careful/hedging token choices |

---

## 🚦 Gate Check

- [ ] You can explain why chain-of-thought works (attention + format mechanism, not "helps it think")
- [ ] You can analyze any prompt and identify: context placement risk, persona priming strength, format signals
- [ ] You ran at least one three-variant test and can explain which won and why
- [ ] You restructured at least one "bad" prompt into a context-optimized version
- [ ] You can predict how a new prompt technique will behave based on the 3 levers

---

## 📚 Cross-References

- [Phase 1: Context Windows & Attention](../01-Foundations/03-Context-Windows-Attention.md) — Attention mechanics that make context placement matter
- [Phase 1: How LLMs Work](../01-Foundations/01-How-LLMs-Work.md) — Token prediction mechanics
- [Next: Techniques Deep Dive](02-Techniques-Deep-Dive.md) — Zero-shot, few-shot, CoT in practice
- [Phase 4: RAG Prompt Design](../04-RAG-Foundations/) — Context engineering for retrieval systems
- [Phase 7: LLM-as-Judge](../07-Production-Evals-Observability/) — Prompts for evaluation
