# ⛩️ GATE 02 — Wordsmith Domain (Prompt Engineering)

**Rank Requirement:** E → D
**Theme:** Mastering the language of AI — craft, structure, and optimize prompts
**Total Days:** 8 Dungeons + 1 Boss Battle
**Core Skills:** Prompt design, chain-of-thought, structured outputs, system prompting

---

## Dungeon 1 — The Anatomy of a Prompt

**Day 1 | Concept:** Every prompt has four parts: Role, Context, Task, Format.

```
[Role] You are a senior Python developer
[Context] I have a list of user emails and need to validate them
[Task] Write a regex function that checks email format
[Format] Return as a Python function with docstring and type hints
```

**🗡️ Dungeon Mastery:** Write a prompt for an AI tutor that explains recursion to a beginner. Include all four parts.

**💻 Code — Prompt Template Engine:**
```python
class Prompt:
    def __init__(self, role="", context="", task="", format_=""):
        self.role = role
        self.context = context
        self.task = task
        self.format = format_

    def build(self) -> str:
        parts = [
            f"[Role] {self.role}" if self.role else "",
            f"[Context] {self.context}" if self.context else "",
            f"[Task] {self.task}" if self.task else "",
            f"[Format] {self.format}" if self.format else "",
        ]
        return "\n".join(p for p in parts if p)

    def __str__(self):
        return self.build()
```

**👹 Boss Lesson:** A complete prompt is like a well-formed spell — missing any part and the magic falters.

---

## Dungeon 2 — Zero-Shot vs Few-Shot

**Day 2 | Concept:** Zero-shot = no examples. Few-shot = N examples. The more complex the task, the more shots you need.

**Zero-shot:**
```
Classify this review: "The battery lasts 10 hours"
Sentiment:
```

**Few-shot (3-shot):**
```
Review: "This product broke in a week" → Negative
Review: "Works exactly as described" → Positive
Review: "It's okay for the price" → Neutral
Review: "The battery lasts 10 hours" →
```

**🗡️ Dungeon Mastery:** Take a task (extract dates from text). Write a zero-shot prompt, then a 3-shot prompt. Notice the quality difference.

**💻 Code — N-Shot Prompt Builder:**
```python
def build_few_shot(system_template: str, examples: list[dict], query: dict, input_key: str, output_key: str):
    messages = [{"role": "system", "content": system_template}]
    for ex in examples:
        messages.append({"role": "user", "content": ex[input_key]})
        messages.append({"role": "assistant", "content": ex[output_key]})
    messages.append({"role": "user", "content": query[input_key]})
    return messages

# Usage
examples = [
    {"text": "Meeting on June 5th", "dates": "2024-06-05"},
    {"text": "Deadline is next Friday", "dates": "2024-07-12"},
]
query = {"text": "Submit by March 3rd"}
messages = build_few_shot("Extract dates from text. Return YYYY-MM-DD.", examples, query, "text", "dates")
```

**👹 Boss Lesson:** Few-shot is like giving a craftsman reference sketches — the harder the task, the more references they need.

---

## Dungeon 3 — Chain-of-Thought (CoT)

**Day 3 | Concept:** Force the model to reason step-by-step before answering. Dramatically improves math, logic, and multi-step tasks.

**Without CoT:**
```
Q: A bat and ball cost $1.10. Bat costs $1.00 more than ball. How much is the ball?
A: $0.10  ❌
```

**With CoT:**
```
Q: A bat and ball cost $1.10. Bat costs $1.00 more than ball. How much is the ball?
A: Let's think step by step.
1. Let b = ball price. Bat = b + 1.00
2. Total: b + (b + 1.00) = 1.10
3. 2b + 1.00 = 1.10
4. 2b = 0.10
5. b = 0.05
The ball costs $0.05. ✅
```

**🗡️ Dungeon Mastery:** Take a logic puzzle (or a LeetCode easy). Write the prompt WITHOUT CoT, get answer, then WITH CoT. Compare.

**💻 Code — CoT Wrapper:**
```python
def chain_of_thought(prompt: str) -> str:
    return f"{prompt}\n\nLet's think step by step."

# For coding tasks, use structured CoT:
def structured_cot(prompt: str) -> str:
    return f"""{prompt}

Work through this systematically:
1. First, understand what's being asked
2. Break down the requirements
3. Plan your approach
4. Implement step by step
5. Verify the solution"""
```

**👹 Boss Lesson:** CoT is the single highest-ROI prompting technique. Always use it for reasoning tasks.

---

## Dungeon 4 — System Prompts & Personas

**Day 4 | Concept:** System messages set the model's behavior, constraints, and persona. This is your most powerful lever.

**Weak System Prompt:**
```
You are a helpful assistant.
```

**Strong System Prompt:**
```
You are SeniorBackendDev, a principal engineer with 15 years of Python experience.

RULES:
- Always write production-grade code with error handling
- Include type hints and docstrings
- Never use third-party libraries unless specified
- Explain performance implications of your approach
- If there's a security concern, flag it immediately

BEHAVIOR:
- Start with "Here's my approach:" followed by a brief plan
- Then provide the full code
- End with "Potential edge cases:" listing 2-3 scenarios

CONSTRAINTS:
- Max 80 lines of code per function
- Prefer standard library over frameworks
- Never use eval() or exec()
```

**🗡️ Dungeon Mastery:** Write a system prompt for an AI code reviewer. Define persona, rules, behavior format, and constraints.

**💻 Code — System Prompt Manager:**
```python
SYSTEM_PROMPTS = {}

def register_persona(name: str, persona_config: dict):
    parts = [f"You are {persona_config['name']}, {persona_config['title']}."]
    if persona_config.get("rules"):
        parts.append("\nRULES:")
        parts.extend(f"- {r}" for r in persona_config["rules"])
    if persona_config.get("behavior"):
        parts.append("\nBEHAVIOR:")
        parts.extend(f"- {b}" for b in persona_config["behavior"])
    if persona_config.get("constraints"):
        parts.append("\nCONSTRAINTS:")
        parts.extend(f"- {c}" for c in persona_config["constraints"])
    SYSTEM_PROMPTS[name] = "\n".join(parts)
    return SYSTEM_PROMPTS[name]

# Register a code reviewer persona
register_persona("code-reviewer", {
    "name": "CodeReviewBot",
    "title": "senior software engineer specializing in code quality",
    "rules": [
        "Review for: correctness, performance, security, maintainability",
        "Rate code 1-10 before giving suggestions",
        "Always provide concrete examples of improvements",
    ],
    "behavior": [
        "Start with overall assessment (2-3 sentences)",
        "Then list issues by severity: CRITICAL, MAJOR, MINOR",
        "End with positive feedback on what was done well",
    ],
    "constraints": [
        "Never rewrite entire file — only show diff-like snippets",
        "Focus on the 3 most important issues maximum",
    ]
})
```

**👹 Boss Lesson:** Your system prompt is the model's brain implant — design it carefully because it shapes everything that follows.

---

## Dungeon 5 — Structured Outputs

**Day 5 | Concept:** Force the model to return parseable structured data (JSON, XML, YAML) rather than freeform text.

**Bad (freeform):**
```
User: Give me 3 book recommendations for Python beginners
AI: I recommend "Automate the Boring Stuff" by Al Sweigart...
```

**Good (structured):**
```
User: Give me 3 book recommendations for Python beginners as JSON
AI: {
  "recommendations": [
    {"title": "Automate the Boring Stuff", "author": "Al Sweigart", "difficulty": "beginner", "focus": "automation"},
    {"title": "Python Crash Course", "author": "Eric Matthes", "difficulty": "beginner", "focus": "fundamentals"},
    {"title": "Think Python", "author": "Allen Downey", "difficulty": "beginner", "focus": "computer science"}
  ]
}
```

**🗡️ Dungeon Mastery:** Ask the model for 5 meal prep ideas. First freeform, then with a JSON schema. Parse the JSON in Python.

**💻 Code — JSON Mode Wrapper:**
```python
import json
from typing import Type, get_type_hints

def structured_extract(prompt: str, schema: dict, model) -> dict:
    """Ask model to return JSON matching schema."""
    system = f"""You are a structured data extractor.
Return ONLY valid JSON matching this exact schema:
{json.dumps(schema, indent=2)}

Do not include any text before or after the JSON."""

    response = model.chat([
        {"role": "system", "content": system},
        {"role": "user", "content": prompt},
    ])

    # Parse response
    text = response["content"]
    # Find JSON block
    start = text.find("{")
    end = text.rfind("}") + 1
    return json.loads(text[start:end]) if start >= 0 else {}

# Example schema
book_schema = {
    "type": "object",
    "properties": {
        "recommendations": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "title": {"type": "string"},
                    "author": {"type": "string"},
                    "difficulty": {"type": "string", "enum": ["beginner", "intermediate", "advanced"]},
                },
                "required": ["title", "author", "difficulty"],
            }
        }
    },
    "required": ["recommendations"],
}
```

**👹 Boss Lesson:** Structured outputs are the difference between "cool demo" and "production pipeline." Always enforce a schema.

---

## Dungeon 6 — Prompt Chaining

**Day 6 | Concept:** Break complex tasks into a chain of simple prompts. Each prompt's output feeds the next prompt's input.

**Single Prompt (fails on complex tasks):**
```
Write a complete SaaS business plan including market analysis, technical architecture, financial projections, and marketing strategy...
```

**Prompt Chain (succeeds):**
```
Chain Step 1: "Generate 5 SaaS business ideas in the AI space"
Chain Step 2: "For idea #3, create a market analysis: TAM, competitors, differentiation"
Chain Step 3: "Based on the analysis, design the technical architecture"
Chain Step 4: "Create financial projections with pricing tiers"
Chain Step 5: "Write a go-to-market strategy"
```

**🗡️ Dungeon Mastery:** Take a task like "plan a 3-day workshop on Docker." Break it into a 4-step prompt chain. Execute each step.

**💻 Code — Prompt Chain Executor:**
```python
class PromptChain:
    def __init__(self, model_fn):
        self.model_fn = model_fn
        self.steps = []
        self.context = {}

    def add_step(self, name: str, prompt_template: str):
        self.steps.append({"name": name, "template": prompt_template})
        return self

    def execute(self, initial_input: str):
        result = initial_input
        for step in self.steps:
            prompt = step["template"].format(
                input=result,
                **self.context,
            )
            result = self.model_fn(prompt)
            print(f"[{step['name']}] → ({len(result)} chars)")
            self.context[step["name"]] = result
        return result

# Usage (conceptual):
chain = PromptChain(my_model)
chain.add_step("ideas", "Generate 3 ideas for {input}")
chain.add_step("deep_dive", "Take idea #1 from: {ideas}\nCreate a detailed analysis")
chain.add_step("action", "Based on: {deep_dive}\nCreate an execution plan")
output = chain.execute("AI-powered study tools")
```

**👹 Boss Lesson:** Prompt chaining is the poor man's agent. It's simple, reliable, and surprisingly powerful.

---

## Dungeon 7 — Temperature & Sampling Parameters

**Day 7 | Concept:** Temperature controls randomness. Top-p (nucleus) controls diversity of token pool. Together they shape creativity vs determinism.

| Temp | Top-p | Use Case |
|------|-------|----------|
| 0.0  | 1.0   | Code generation, math, facts |
| 0.3  | 0.9   | Classification, extraction |
| 0.7  | 0.9   | Creative writing, brainstorming |
| 1.0  | 0.95  | Poetry, ideation, diverse outputs |

**🗡️ Dungeon Mastery:** Generate the same prompt ("Write a haiku about AI") at temp=0.0, 0.5, and 1.0. Compare diversity and quality.

**💻 Code — Parameter Grid Search:**
```python
def generate_with_params(model, prompt: str, temps: list[float], top_ps: list[float]):
    results = []
    for temp in temps:
        for top_p in top_ps:
            response = model.generate(prompt, temperature=temp, top_p=top_p)
            results.append({"temp": temp, "top_p": top_p, "output": response})
    return results

# Always run this when doing exploration vs production:
def production_generation(model, prompt: str):
    """For production: deterministic, reliable outputs."""
    return model.generate(prompt, temperature=0.0, top_p=1.0)

def creative_generation(model, prompt: str):
    """For creative: diverse, surprising outputs."""
    return model.generate(prompt, temperature=0.8, top_p=0.95)
```

**👹 Boss Lesson:** Temperature 0.0 for code. Temperature 0.7+ for creative. Never use high temperature for factual tasks.

---

## Dungeon 8 — Prompt Engineering Patterns

**Day 8 | Concept:** Learn the proven patterns: Skeleton-of-Thought, Tree-of-Thought, Reflexion, Self-Consistency, ReAct.

| Pattern | Use When | How |
|---------|----------|-----|
| **Skeleton-of-Thought** | Long-form writing | Generate outline first, then expand each section |
| **Tree-of-Thought** | Complex reasoning | Explore multiple reasoning paths simultaneously |
| **Reflexion** | Code generation | Generate → Test → Fix |
| **Self-Consistency** | Math/facts | Generate N answers, pick most common |
| **ReAct** | Tool use | Reason → Act → Observe → Repeat |

**🗡️ Dungeon Mastery:** Apply Self-Consistency to a math problem: generate 3 answers, pick the majority. Then apply Reflexion to fix buggy code.

**💻 Code — Self-Consistency:**
```python
from collections import Counter

def self_consistency(model, prompt: str, n: int = 5):
    answers = []
    for i in range(n):
        response = model.generate(prompt, temperature=0.5)
        answers.append(response)

    # Find most common answer
    answer_counts = Counter(answers)
    most_common = answer_counts.most_common(1)[0][0]
    return {
        "final_answer": most_common,
        "answers": answers,
        "confidence": answer_counts.most_common(1)[0][1] / n,
    }
```

**👹 Boss Lesson:** Pattern mastery separates novices from experts. Build your mental library of these patterns.

---

## ⚔️ BOSS BATTLE: The Prompt Engineering System

**Objective:** Design a complete prompt engineering system that:
1. Takes a user's raw task description
2. Classifies task type (code, writing, analysis, creative)
3. Applies the optimal prompt pattern for that type
4. Includes system prompt, few-shot examples, and output format
5. Returns the final optimized prompt ready to send to the model

**🗡️ Victory Conditions:**
- Task classification works for 4+ types
- System prompt adapts per type
- Few-shot examples auto-select based on type
- Output format switches automatically (JSON for data, markdown for writing)
- Temperature adjusts per type

**🏆 Rewards:**
- +500 XP
- D-Rank → D+ Rank
- Skill Unlock: **Prompt Architect** — All future prompt-based tasks get 20% better results
- Title: **The Wordsmith**

---

*"The quality of your output is bounded by the quality of your prompt."*
