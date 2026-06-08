# Decision Framework for AI Architecture

## 🎯 Purpose & Goals

Every AI engineering decision involves tradeoffs. Better quality costs more. Lower latency reduces accuracy. More context window increases cost. A senior engineer doesn't make these choices by gut — they use a **repeatable decision framework**.

**By the end of this file, you will:**
- Have a 7-step framework for ANY AI architecture decision
- Be able to objectively compare Model A vs Model B, Architecture X vs Architecture Y
- Document tradeoffs so your team can understand your reasoning
- Avoid the most common decision traps in AI engineering

**⏱ Time Budget:** 1.5 hours

---

## 📖 The 7-Step Decision Framework

### Step 1: Define Constraints (Not Preferences)

Before evaluating options, establish hard constraints:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class DecisionConstraints:
    """Hard boundaries for any architectural decision"""
    
    # Performance
    max_p95_latency_ms: int = 2000
    min_accuracy_score: float = 0.85
    
    # Cost
    max_cost_per_query: float = 0.01  # $0.01
    max_monthly_budget: float = 5000  # $5K/month
    
    # Operational
    max_tokens_per_minute: int = 100000
    required_uptime: float = 0.99  # 99% uptime
    
    # Data
    data_residency_required: Optional[str] = None  # e.g., "India"
    pii_handling_required: bool = True
    
    # Integration
    existing_stack: list[str] = None  # e.g., ["PostgreSQL", "AWS", "Redis"]
```

### Step 2: Generate Options (Minimum 3)

Never choose between 2 options. Always find at least 3:

```python
# BAD: Only 2 options, no creativity
options = [
    {"model": "gpt-4o", "cost_per_query": 0.01, "latency_ms": 800},
    {"model": "gpt-4o-mini", "cost_per_query": 0.001, "latency_ms": 400},
]

# GOOD: 4 options + reasoning
options = [
    {
        "name": "GPT-4o Only",
        "model": "gpt-4o",
        "cost": 0.01,
        "latency_ms": 800,
        "quality_estimate": 0.95,
        "pros": ["Best quality", "Proven reliability"],
        "cons": ["Most expensive", "Highest latency"],
    },
    {
        "name": "GPT-4o-mini Only",
        "model": "gpt-4o-mini",
        "cost": 0.001,
        "latency_ms": 400,
        "quality_estimate": 0.82,
        "pros": ["Cheapest", "Fastest"],
        "cons": ["Might be below quality threshold"],
    },
    {
        "name": "Cascade (mini → 4o)",
        "model": "cascade",
        "cost": 0.003,  # 90% mini + 10% 4o
        "latency_ms": 500,
        "quality_estimate": 0.93,
        "pros": ["Good quality most of time", "Cost effective"],
        "cons": ["Complex fallback logic", "Inconsistent latency"],
    },
    {
        "name": "Claude Sonnet",
        "model": "claude-3-5-sonnet",
        "cost": 0.008,
        "latency_ms": 600,
        "quality_estimate": 0.94,
        "pros": ["Excellent quality", "200K context window"],
        "cons": ["Different API to learn", "Slightly slower than GPT-4o"],
    },
]
```

### Step 3: Score Against Constraints

```python
def score_option(option: dict, constraints: DecisionConstraints) -> dict:
    """Score each option against constraints. 0 = violates, 1 = perfect fit."""
    scores = {}
    
    # Latency score (exponential penalty as you approach limit)
    if option["latency_ms"] > constraints.max_p95_latency_ms:
        scores["latency"] = 0.0  # Violates constraint
    else:
        scores["latency"] = 1.0 - (option["latency_ms"] / constraints.max_p95_latency_ms) ** 2
    
    # Cost score (linear degradation)
    if option["cost"] > constraints.max_cost_per_query:
        scores["cost"] = 0.0  # Violates constraint
    else:
        scores["cost"] = 1.0 - (option["cost"] / constraints.max_cost_per_query)
    
    # Quality score (higher is better)
    scores["quality"] = min(option["quality_estimate"], 1.0)
    
    # Composite score (weighted)
    weights = {"latency": 0.2, "cost": 0.3, "quality": 0.5}
    composite = sum(scores[k] * weights[k] for k in weights)
    scores["composite"] = composite
    
    return scores

# Apply to all options
for opt in options:
    opt["scores"] = score_option(opt, constraints)
    print(f"{opt['name']}: {opt['scores']['composite']:.2f}")
```

### Step 4: Identify Risks

For the top options, list specific risks:

```python
def identify_risks(option: dict) -> list[dict]:
    """Enumerate specific, concrete risks — not vague fears."""
    risks = []
    
    if option["model"] == "gpt-4o":
        risks.extend([
            {"risk": "Cost overrun", "probability": 0.3, "impact": "High", "mitigation": "Set monthly budget alerts"},
            {"risk": "Latency spike under load", "probability": 0.2, "impact": "Medium", "mitigation": "Implement request queuing"},
        ])
    elif option["model"] == "cascade":
        risks.append({
            "risk": "Complex fallback logic may have bugs",
            "probability": 0.4,
            "impact": "Medium",
            "mitigation": "Thorough integration testing + canary deploys"
        })
    
    return risks
```

### Step 5: Prototype the Top 2

Never pick without prototyping:

```python
def prototype_comparison(option_a: dict, option_b: dict, test_cases: list[str]) -> dict:
    """Run both options on the same test cases and compare"""
    results = []
    
    for test in test_cases:
        result_a = call_model(option_a["model"], test)
        result_b = call_model(option_b["model"], test)
        
        # Compare quality using LLM judge
        comparison = compare_outputs(test, result_a, result_b)
        results.append(comparison)
    
    # Aggregate
    a_wins = sum(1 for r in results if r["winner"] == "A")
    b_wins = sum(1 for r in results if r["winner"] == "B")
    ties = sum(1 for r in results if r["winner"] == "tie")
    
    return {
        "option_a_wins": a_wins,
        "option_b_wins": b_wins,
        "ties": ties,
        "a_latency_avg": sum(r["latency_a"] for r in results) / len(results),
        "b_latency_avg": sum(r["latency_b"] for r in results) / len(results),
    }
```

### Step 6: Make Decision with Rationale

```python
@dataclass
class ArchitectureDecision:
    """The output of the decision framework — document EVERYTHING"""
    
    decision_id: str
    date: str
    decider: str
    
    # What we decided
    chosen_option: str
    rejected_options: list[str]
    
    # Why
    primary_reason: str
    key_tradeoffs: list[str]
    
    # Evidence
    constraints_used: dict
    scores: dict
    prototype_results: dict
    identified_risks: list[dict]
    
    # What would change this decision
    decision_reversal_conditions: list[str]
    
    def to_report(self) -> str:
        """Generate a decision log entry"""
        lines = [
            f"## ADR-{self.decision_id}: {self.chosen_option}",
            f"**Date:** {self.date} | **Decider:** {self.decider}",
            "",
            "### Decision",
            f"Chose {self.chosen_option}. Rejected: {', '.join(self.rejected_options)}",
            "",
            "### Rationale",
            self.primary_reason,
            "",
            "### Key Tradeoffs",
            *[f"- {t}" for t in self.key_tradeoffs],
            "",
            "### Risks & Mitigations",
            *[f"- {r['risk']} (prob: {r['probability']}) → {r['mitigation']}" for r in self.identified_risks],
            "",
            "### When to Revisit",
            *[f"- {c}" for c in self.decision_reversal_conditions],
        ]
        return "\n".join(lines)
```

### Step 7: Set Revisit Conditions

Every decision has a shelf life:

```python
# Conditions that would change this decision:
reversal_conditions = [
    "GPT-4o-mini quality improves to match GPT-4o (model upgrade)",
    "Monthly cost exceeds $5K (budget constraint violated)",
    "New model with better quality/cost ratio enters market",
    "User requirements change (e.g., need 200K context)",
    "Latency requirements tighten below 200ms",
]
```

---

## 📖 Common Decision Traps

### Trap 1: "Newer is Better"

```python
# BAD:
model = "gpt-4o-latest"  # Just released, must be best!

# GOOD:
# Wait for benchmarks and community experience before upgrading
# New models can regress on specific tasks
# Always eval against your golden dataset before switching
```

### Trap 2: "More Context is Always Better"

```python
# BAD:
# "Gemini 1.5 has 1M tokens — let's use it!"
# Problem: 1M tokens = $15-$50 per query (depending on model)

# GOOD:
# Measure actual context needs of your use case
# 95% of RAG queries need < 10K tokens
# Use the SMALLEST context window that meets your needs
```

### Trap 3: "Framework Will Fix It"

```python
# BAD:
# "Our RAG quality is bad. Let's add LangChain."
# LangChain adds abstraction, not quality.

# GOOD:
# Measure WHERE quality breaks first
# Is it retrieval? (chunking? embedding model? search algorithm?)
# Is it generation? (prompt? model? context quality?)
# Fix the actual problem, not a perceived one
```

### Trap 4: "We Need to Fine-Tune"

```python
# BAD:
# "The model doesn't understand our domain. Let's fine-tune."
# Fine-tuning costs: $100-$1000 + ongoing inference complexity

# GOOD:
# Can RAG solve this? (give the model domain docs = cheaper, faster)
# Can prompt engineering solve this? (better instructions = free)
# Is fine-tuning actually the right tool? It teaches PATTERNS, not facts.
# Fine-tune ONLY when: you need the model to behave differently,
# not when you need it to know different things.
```

---

## ✅ Real Decision Example

```python
# CONTEXT: Choosing a vector database for a customer support RAG system
# Constraints: < $200/month, < 100ms p95 query time, 500K documents, 
#              already using PostgreSQL, team knows SQL

# Options considered:
# 1. Pinecone — $0/month free tier → $70/month for starter
# 2. Qdrant — self-hosted on existing infra = $0/month extra
# 3. pgvector — on existing PostgreSQL = $0/month extra
# 4. Weaviate — $25/month cloud starter

# Decision: pgvector
# Rationale: Already using PostgreSQL. 500K docs is well within pgvector
# capability. Zero additional infrastructure. Team knows SQL.
# 
# Tradeoffs accepted: Building our own index management instead of managed service.
# Risk: If we scale to 50M+ docs, we'll need to migrate.
# Reversal condition: Document count exceeds 5M OR query time exceeds 200ms.

# DECISION LOG:
"""
ADR-001: Vector Database Selection
Date: 2025-05-15 | Decider: Partha

Decision: pgvector on existing PostgreSQL
Rejected: Pinecone (cost), Weaviate (complexity), Qdrant (ops overhead)

Rationale:
- Zero additional infrastructure cost (using existing Postgres instance on RDS)
- Team already knows SQL and Postgres operations
- 500K documents × 1536 dimensions ≈ 3GB — fits easily in existing storage
- HNSW index provides ~10ms query time at this scale
- Avoids learning new API and managing another service

Tradeoffs:
- Will need to migrate when/if dataset exceeds 5M documents
- Self-managed index tuning vs managed service

Risks:
- Full table scans during re-indexing (mitigation: do during off-peak)
- pgvector extension update requires downtime (mitigation: blue-green deploy)

When to Revisit:
- Document count exceeds 5M
- Query latency exceeds 200ms at p95
- PostgreSQL CPU usage > 70% from vector queries
"""
```

---

## 🧪 Drill: Make a Decision Using the Framework

**Time: 30 minutes**

**Scenario:** You're building a document analysis SaaS. Users upload PDFs (10-100 pages). The system extracts structured data (entities, dates, amounts, summary). You need to choose the approach.

**Task:** Use all 7 steps of the framework to decide between:
1. GPT-4o-mini (cheap, fast, limited context)
2. Claude 3.5 Sonnet (200K context, excellent document handling, expensive)
3. Local Llama 3.2 via Ollama (free, private, lower quality)

Write the full decision document including constraints, scores, risks, and reversal conditions.

<details>
<summary>Sample decision (check after writing)</summary>

```python
# CONSTRAINTS
constraints = DecisionConstraints(
    max_p95_latency_ms=5000,  # 5 seconds for document analysis is fine
    min_accuracy_score=0.85,
    max_cost_per_query=0.05,  # $0.05 per document
    max_monthly_budget=1000,
    data_residency_required="India",  # Some docs are confidential
)

# SCORES
# 1. GPT-4o-mini: cost=0.001, latency=2000ms, quality=0.78 → VIOLATES quality
# 2. Claude Sonnet: cost=0.03, latency=3500ms, quality=0.95 → BEST FIT
# 3. Llama 3.2 (local): cost=0.0, latency=4000ms, quality=0.72 → VIOLATES quality

# DECISION: Claude 3.5 Sonnet for all document analysis
# Rationale: Only option meeting quality threshold. 200K context handles
# any document. Acceptable cost ($0.03 × 1000 docs/day × 30 = $900/month).
# Data residency: Anthropic doesn't train on API inputs.
# 
# RISK: $900/month is close to $1000 budget ceiling.
# MITIGATION: Monitor monthly usage. If exceeds $800, implement a classifier
# that routes simple documents to GPT-4o-mini and complex ones to Claude.
#
# REVERSAL CONDITIONS:
# - Monthly bill exceeds $1000
# - Llama 4 (or equivalent local model) achieves >0.85 quality
# - GPT-5 release with better document handling
```
</details>

---

## 🚦 Gate Check

- [ ] Can you name all 7 steps of the decision framework?
- [ ] Can you write a decision log for a recent choice you made?
- [ ] Can you identify at least 3 decision traps?
- [ ] Do you know what conditions would reverse your decision?

**Proceed when all four are YES.**
