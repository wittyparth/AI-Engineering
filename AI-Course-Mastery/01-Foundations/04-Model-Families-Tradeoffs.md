# Model Families & Tradeoffs — Picking the Right Tool

## 🎯 Purpose & Goals

> **🛑 STOP. Before you read anything, answer this:**
>
> Your PM asks: "We need to analyze 200-page legal documents. Pick a model."
> Your options: GPT-4o, GPT-4o-mini, Claude 3.5 Sonnet, Gemini 1.5 Pro
>
> **What do you pick and why? Write your thought process.**
>
> *Don't worry about being wrong. Worry about not thinking.*

---

**By the end of this file, you will be able to make that decision — and every model selection decision — systematically, not by gut.**

**⏱ Time Budget:** 2 hours

---

## 📖 The Model Landscape (2025)

Here are the major model families you'll work with as an AI engineer. Each has different strengths, weaknesses, and costs.

### 🟢 OpenAI (GPT Family)

| Model | Best For | Context | Cost (Input/1M) | Latency |
|-------|----------|---------|-----------------|---------|
| GPT-4o | General purpose, reasoning, vision | 128K | $2.50 / $10.00 | Medium |
| GPT-4o-mini | Fast, cheap, simple tasks | 128K | $0.15 / $0.60 | Fast |
| o1 | Hard reasoning (math, science) | 128K | $15.00 / $60.00 | Very Slow |
| o1-mini | Cheaper reasoning | 128K | $1.10 / $4.40 | Slow |

### 🔵 Anthropic (Claude Family)

| Model | Best For | Context | Cost (Input/1M) | Latency |
|-------|----------|---------|-----------------|---------|
| Claude 3.5 Sonnet | Long docs, coding, safety | 200K | $3.00 / $15.00 | Medium |
| Claude 3 Haiku | Fast, cheap, classification | 200K | $0.25 / $1.25 | Very Fast |

### 🔴 Google (Gemini Family)

| Model | Best For | Context | Cost (Input/1M) | Latency |
|-------|----------|---------|-----------------|---------|
| Gemini 1.5 Pro | Massive context, multimodal | 1M | $1.25 / $5.00 | Medium |
| Gemini 1.5 Flash | Cheap multimodal | 1M | $0.075 / $0.30 | Very Fast |
| Gemini 2.0 Flash | New gen, coding | 1M | $0.10 / $0.40 | Very Fast |

### 🟣 Open Source (via Ollama)

| Model | Best For | Context | Cost | Quality |
|-------|----------|---------|------|---------|
| Llama 3.1 70B | General (near GPT-4 level) | 128K | Free (self-host) | High |
| Llama 3.2 3B | Fast, simple, mobile | 128K | Free | Medium |
| Mistral | European, efficient | 32K | Free | Medium-High |
| Phi-3 | Small but capable | 128K | Free | Medium |

---

## 📖 Your Turn: Think Before You Read

> **🤔 SCENARIO 1:** You're building a real-time chatbot for customer service. Queries are short (50-200 tokens), answers need to be fast (< 1 second), and you'll handle 1M queries/day. Monthly budget for AI: $2,000.
>
> **Write down which model(s) you'd use and why. Calculate the cost.**

Now think about this while reading the next section...

---

## 📖 The Decision Matrix

Here's how a senior engineer actually chooses models. Not by "which is best" but by matching requirements to model characteristics:

```python
"""
SENIOR ENGINEER'S MODEL SELECTION MATRIX:
For each requirement, which models win?
"""

def select_model(requirements: dict) -> str:
    """
    requirements = {
        "task_type": "chat" | "reasoning" | "extraction" | "coding" | "creative",
        "max_latency_ms": int,
        "max_cost_per_query": float,
        "context_window_needed": int,
        "needs_vision": bool,
        "needs_structured_output": bool,
        "data_privacy_critical": bool,
        "language": "english" | "hindi" | "multilingual",
    }
    """
    
    scores = {}
    
    # Score each model on multiple dimensions
    models = {
        "gpt-4o": {"quality": 0.95, "latency": 0.7, "cost": 0.3, "context": 0.5, "vision": True, "structured": True},
        "gpt-4o-mini": {"quality": 0.82, "latency": 0.95, "cost": 0.95, "context": 0.5, "vision": True, "structured": True},
        "claude-3.5-sonnet": {"quality": 0.94, "latency": 0.75, "cost": 0.25, "context": 0.8, "vision": True, "structured": False},
        "claude-3-haiku": {"quality": 0.78, "latency": 0.95, "cost": 0.85, "context": 0.8, "vision": False, "structured": False},
        "gemini-1.5-flash": {"quality": 0.8, "latency": 0.95, "cost": 0.98, "context": 1.0, "vision": True, "structured": True},
    }
    
    for name, specs in models.items():
        score = 0
        
        # Quality requirement
        if requirements.get("task_type") in ["reasoning", "coding"]:
            score += specs["quality"] * 3  # Quality matters most
        else:
            score += specs["quality"] * 1.5
        
        # Latency requirement
        if requirements.get("max_latency_ms", 9999) < 500:
            score += specs["latency"] * 3  # Speed critical
        elif requirements.get("max_latency_ms", 9999) < 2000:
            score += specs["latency"] * 2
        else:
            score += specs["latency"] * 1
        
        # Cost requirement
        cost_budget = requirements.get("max_cost_per_query", 0.01)
        if cost_budget < 0.001:
            score += specs["cost"] * 3  # Cost critical
        elif cost_budget < 0.005:
            score += specs["cost"] * 2
        else:
            score += specs["cost"] * 1
        
        # Context requirement
        if requirements.get("context_window_needed", 0) > 100000:
            score += specs["context"] * 3
        
        # Feature requirements
        if requirements.get("needs_vision") and not specs["vision"]:
            score -= 5  # Disqualify
        if requirements.get("needs_structured_output") and not specs["structured"]:
            score -= 5  # Disqualify
        
        scores[name] = score
    
    return max(scores, key=scores.get)

# Example: Customer service chatbot
model = select_model({
    "task_type": "chat",
    "max_latency_ms": 800,
    "max_cost_per_query": 0.002,
    "context_window_needed": 4000,
    "needs_vision": False,
    "needs_structured_output": False,
})
print(f"Customer service chatbot → {model}")
# Likely: gpt-4o-mini (cheap, fast, good enough)

# Example: Legal document analyzer
model = select_model({
    "task_type": "reasoning",
    "max_latency_ms": 5000,
    "max_cost_per_query": 0.05,
    "context_window_needed": 150000,
    "needs_vision": True,
    "needs_structured_output": True,
})
print(f"Legal document analyzer → {model}")
# Likely: gemini-1.5-flash (massive context, cheap, structured)
```

---

## 📖 The Trap: When "Better" Models Are Worse

### Trap 1: GPT-4o for Everything

```python
"""
COMMON TRAP: "GPT-4o is the best, so use it for everything."
REALITY: GPT-4o costs 16x more than GPT-4o-mini.
For simple classification tasks, they perform almost identically.

SENIOR APPROACH:
- Classify the TASK difficulty first
- Route simple tasks to cheap models
- Route complex tasks to capable models
- This is called a "model cascade" or "router pattern"
"""

class ModelRouter:
    """Route requests to appropriate model based on complexity"""
    
    def __init__(self):
        self.easy_model = "gpt-4o-mini"  # $0.15/1M input
        self.hard_model = "gpt-4o"        # $2.50/1M input
        self.cascade_threshold = 0.85
    
    def classify_difficulty(self, task: str) -> str:
        """Quick classifier to determine if task needs strong model"""
        response = client.chat.completions.create(
            model=self.easy_model,  # Use CHEAP model to classify!
            messages=[{
                "role": "user",
                "content": f"""Classify this task difficulty as 'easy' or 'hard'.
Easy = simple lookup, classification, extraction, formatting
Hard = complex reasoning, multi-step, creative, analysis

Task: {task}
Respond with ONE word: easy or hard"""
            }],
            temperature=0.0,
            max_tokens=10
        )
        return response.choices[0].message.content.strip()
    
    def route(self, task: str) -> str:
        difficulty = self.classify_difficulty(task)
        model = self.easy_model if difficulty == "easy" else self.hard_model
        
        print(f"[Router] Task difficulty: {difficulty} → Using {model}")
        print(f"[Router] Cost: ${'0.00015' if model == self.easy_model else '0.0025'}/query")
        
        response = client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": task}]
        )
        return response.choices[0].message.content

# SAVINGS ANALYSIS:
# 100K queries/day, 70% easy, 30% hard
# All GPT-4o: 100K × $0.0025 = $250/day = $7,500/month
# Routed: 70K × $0.00015 + 30K × $0.0025 = $10.50 + $75 = $85.50/day = $2,565/month
# SAVINGS: $4,935/month (66% reduction)
```

### Trap 2: "Best" Model for Every Language

```python
"""
TRAP: Using English-optimized models for non-English text.
REALITY: Non-English text uses 2-5x more tokens.
A "cheap" model on Hindi text costs more than an "expensive" model on English.

SENIOR APPROACH: Consider effective cost per QUERY, not per TOKEN.
"""

def effective_cost_per_query(model: str, text_language: str, word_count: int) -> float:
    """Calculate REAL cost considering language token multiplier"""
    
    token_multipliers = {
        "english": 1.0,    # Baseline
        "spanish": 1.1,    # Slightly more
        "french": 1.1,
        "german": 1.2,
        "hindi": 2.5,      # Significantly more tokens
        "tamil": 3.0,      # Even more
        "telugu": 3.0,
        "bengali": 2.8,
        "chinese": 2.0,
        "japanese": 2.5,
        "arabic": 1.8,
    }
    
    multiplier = token_multipliers.get(text_language, 1.5)
    effective_tokens = word_count * 0.75 * multiplier  # words → English tokens → language-adjusted
    
    pricing = {
        "gpt-4o": (2.50, 10.00),
        "gpt-4o-mini": (0.15, 0.60),
        "claude-3-haiku": (0.25, 1.25),
    }
    
    input_rate, output_rate = pricing.get(model, (0.15, 0.60))
    input_cost = (effective_tokens / 1_000_000) * input_rate
    output_cost = (effective_tokens / 1_000_000) * output_rate  # rough
    
    return input_cost + output_cost

# Compare: 100-word query in Hindi vs English
for lang in ["english", "hindi", "tamil"]:
    for model in ["gpt-4o-mini", "claude-3-haiku"]:
        cost = effective_cost_per_query(model, lang, 100)
        print(f"{lang:10} | {model:15} | ${cost:.5f}/query")

# Results might surprise you — a "cheap" model on Hindi 
# could cost more than a "premium" model on English
```

> **🤔 YOUR TURN:** Go back to the Scenario 1 question at the top. Based on what you've learned, would you change your answer? Write the new answer.

---

## 🧪 Design Challenge: Build a Model Selector

**Time: 30 minutes**

**THE CHALLENGE:** Write a function `recommend_model(task_description: str, constraints: dict) -> str` that:
1. Analyzes the task description to determine complexity
2. Considers constraints: max_cost, max_latency_ms, min_context_window
3. Returns the best model name with a brief justification

**TRY IT YOURSELF FIRST.** Don't look at my solution until you've written yours.

```python
# YOUR CODE HERE:
def recommend_model(task_description: str, constraints: dict) -> str:
    """
    constraints = {
        "max_cost_per_query": float,
        "max_latency_ms": int, 
        "min_context_tokens": int,
        "needs_vision": bool,
        "data_privacy": bool,
    }
    """
    # Write your logic here
    pass
```

<details>
<summary>👀 Click to see my solution AFTER you've written yours</summary>

```python
def recommend_model(task_description: str, constraints: dict) -> str:
    """
    Senior engineer's model recommender.
    Not perfect, but systematic — which is better than guessing.
    """
    
    # Step 1: Determine task complexity from description
    hard_keywords = ["reason", "analyze", "complex", "multi-step", "debug", 
                     "creative", "write", "long", "document", "contract"]
    easy_keywords = ["classify", "extract", "summarize", "translate", 
                     "format", "parse", "simple"]
    
    task_lower = task_description.lower()
    hard_score = sum(2 for k in hard_keywords if k in task_lower)
    easy_score = sum(1 for k in easy_keywords if k in task_lower)
    
    is_complex = hard_score > easy_score
    needs_high_quality = "code" in task_lower or "reason" in task_lower or "math" in task_lower
    
    # Step 2: Apply constraints
    cost_budget = constraints.get("max_cost_per_query", 0.01)
    latency_budget = constraints.get("max_latency_ms", 2000)
    context_needed = constraints.get("min_context_tokens", 4000)
    
    # Step 3: Filter candidate models
    candidates = []
    
    # GPT-4o-mini: cheap, fast, decent quality
    if cost_budget >= 0.001 and latency_budget >= 400:
        score = 0
        if not is_complex:
            score += 3
        score += 1 if cost_budget < 0.005 else 0
        score += 1 if latency_budget < 1000 else 0
        candidates.append(("gpt-4o-mini", score))
    
    # GPT-4o: high quality, moderate speed
    if cost_budget >= 0.01 and latency_budget >= 800:
        score = 0
        if is_complex or needs_high_quality:
            score += 3
        if context_needed <= 128000:
            score += 1
        candidates.append(("gpt-4o", score))
    
    # Claude 3.5 Sonnet: best for long documents
    if context_needed > 100000 and cost_budget >= 0.02:
        score = 0
        if "document" in task_lower or "contract" in task_lower or "long" in task_lower:
            score += 3
        candidates.append(("claude-3-5-sonnet", score))
    
    # Claude 3 Haiku: very fast, very cheap
    if latency_budget < 500 and cost_budget < 0.002:
        candidates.append(("claude-3-haiku", 2))
    
    # Step 4: Handle data privacy
    if constraints.get("data_privacy"):
        # Local model recommended
        return "llama-3.2-70b (local via Ollama)"
    
    # Step 5: Return best match
    if not candidates:
        return "gpt-4o-mini"  # Safe default
    
    candidates.sort(key=lambda x: x[1], reverse=True)
    return candidates[0][0]

# Test it
print(recommend_model("Classify customer sentiment from reviews", 
                      {"max_cost_per_query": 0.001, "max_latency_ms": 500}))
# Likely: gpt-4o-mini

print(recommend_model("Analyze this 150-page legal contract and find risks", 
                      {"max_cost_per_query": 0.05, "max_latency_ms": 5000, "min_context_tokens": 150000}))
# Likely: claude-3-5-sonnet
```

**Compare with your solution. Where did your thinking differ from mine?**
- Did you consider latency as a constraint?
- Did you check context window requirements?
- Did you think about cost-to-quality ratio?

**The gap you find is exactly what you need to learn.**
</details>

---

## 🧪 Debug Challenge: Wrong Model Selection

```python
# BUGGY CODE — The team chose the wrong model. Why?

task = "Extract company names and funding amounts from 10,000 news articles"
selected_model = "claude-3-5-sonnet"  # $3.00/1M input tokens

# Expected monthly cost:
# Average article: 500 tokens
# 10,000 articles × 500 tokens = 5,000,000 tokens
# Monthly cost: (5,000,000 / 1,000,000) × $3.00 = $15.00

print(f"Expected monthly cost: $15.00")  # Seems fine!

# But wait... each article needs a structured output (JSON with company, amount, round)
# Claude returns text, not guaranteed JSON
# You'd need retries, validation, error handling
# Real cost: $15 + 30% retries + 20% wasted outputs = ~$23/month

# ACTUAL PROBLEM: GPT-4o-mini ($0.15/1M) with structured outputs would cost:
# (5,000,000 / 1,000,000) × $0.15 = $0.75/month + guaranteed valid JSON = no retries
# Total real cost: ~$0.75/month

# SENIOR INSIGHT:
# For BULK PROCESSING with STRUCTURED OUTPUTS:
# - Cost per token matters MORE than accuracy (you can retry)
# - Structured output support matters (Saves validation code)
# - Claude 3.5 Sonnet is OVERKILL for simple extraction
# - GPT-4o-mini with response_format="json" is the RIGHT choice
```

**🔍 What was the senior insight the team missed?** (Think before scrolling)

<details>
<summary>Answer</summary>
They optimized for QUALITY instead of TOTAL COST OF OWNERSHIP.
Yes, Claude is more accurate. But for simple extraction at scale:
- GPT-4o-mini is 20x cheaper
- GPT-4o-mini supports structured outputs (guaranteed JSON)
- Cheaper = can afford more retries if quality is low sometimes
- Total cost: $0.75 vs $23 — for the SAME usable results
</details>

---

## ✅ Model Selection Checklist

Before picking a model for ANY task:

- [ ] Is the model overkill for this task? (Could a cheaper model do 90% as well?)
- [ ] Have I calculated the real cost at scale (not just per-query)?
- [ ] Have I considered language token multipliers?
- [ ] Does the model support structured outputs if needed?
- [ ] What's the latency requirement? Does the model meet it?
- [ ] What's the context requirement? (Pro tip: measure, don't guess)
- [ ] Is there a data privacy requirement? (Local model vs API)
- [ ] Have I considered a model cascade (cheap + expensive fallback)?

---

## 🚦 Gate Check

- [ ] I can pick the right model for a given task and justify WHY
- [ ] I can calculate REAL cost considering language, retries, and scale
- [ ] I understand when "worse" models are actually better (cost, features)
- [ ] I've written my own model selector function
- [ ] I can identify over-engineering in model selection

---

**→ Continue to `05-Structured-Outputs-Basics.md`**
