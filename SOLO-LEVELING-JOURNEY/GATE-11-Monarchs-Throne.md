# ⛩️ GATE 11 — Monarch's Throne (Capstone)

**Rank Requirement:** A+
**Theme:** The final synthesis — integrating all 10 gates into a complete, production-ready AI engineering system
**Total Days:** 4 Dungeons + 1 Boss Battle
**Core Skills:** System design, architecture synthesis, production readiness, AI engineering mastery

---

## Dungeon 1 — The Grand Architecture

**Day 1 | Concept:** All the pieces connect. Your complete AI system is a pipeline of gates working together.

**The Complete System:**
```
                    ┌─────────────────────────────────────┐
                    │         User Query                   │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │   GATE 08: Input Guardrails          │
                    │   (Toxicity, injection, length)      │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │   GATE 07: Query Understanding       │
                    │   (Classification, routing)          │
                    └─────────────┬───────────────────────┘
                                  │
              ┌───────────────────┼────────────────────┐
              │                   │                    │
     ┌────────▼────────┐  ┌──────▼───────┐  ┌────────▼────────┐
     │ GATE 03: Simple │  │ GATE 02:     │  │ GATE 06: Complex│
     │ Q&A (Direct LLM)│  │ RAG System   │  │ Agent (Tools)   │
     └────────┬────────┘  └──────┬───────┘  └────────┬────────┘
              │                   │                    │
              └───────────────────┼────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │   GATE 04: Context Assembly          │
                    │   (RAG → Prompt)                     │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │   GATE 09: LLM Generation            │
                    │   (Fine-tuned or base model)        │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │   GATE 08: Output Guardrails         │
                    │   (PII, toxicity, format)            │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │   GATE 07: Evaluation & Monitoring   │
                    │   (Quality, latency, cost)           │
                    └─────────────┬───────────────────────┘
                                  │
                    ┌─────────────▼───────────────────────┐
                    │         User Response                │
                    └─────────────────────────────────────┘
```

**🗡️ Dungeon Mastery:** Systematically trace a user query through ALL 10 gates. For each gate, write: what it contributes, what failure mode it prevents, and what it adds in latency.

**💻 Code — System Architecture:**
```python
from dataclasses import dataclass, field
from typing import Callable
import time
import json

@dataclass
class GateMetrics:
    gate_name: str
    latency_ms: float
    tokens_used: int
    passed: bool
    details: str = ""

class AISystem:
    def __init__(self):
        self.gates: list[tuple[str, Callable]] = []
        self.metrics: list[GateMetrics] = []
        self.observability_enabled = True

    def register(self, name: str, gate_fn: Callable):
        self.gates.append((name, gate_fn))

    def execute(self, user_query: str) -> dict:
        self.metrics = []
        context = {"query": user_query, "state": {}}

        for name, gate_fn in self.gates:
            start = time.time()
            try:
                result = gate_fn(context)
                elapsed = (time.time() - start) * 1000
                self.metrics.append(GateMetrics(
                    gate_name=name, latency_ms=round(elapsed, 1),
                    tokens_used=result.get("tokens", 0),
                    passed=True,
                ))
                context.update(result)
            except Exception as e:
                elapsed = (time.time() - start) * 1000
                self.metrics.append(GateMetrics(
                    gate_name=name, latency_ms=round(elapsed, 1),
                    tokens_used=0, passed=False,
                    details=str(e),
                ))
                # Fail-open or fail-closed based on gate type
                if name in ["input_guardrails"]:
                    return {"error": f"Blocked by {name}", "metrics": self.metrics}

        return {"response": context.get("response", ""), "metrics": self.metrics}

    def report(self) -> dict:
        total_latency = sum(m.latency_ms for m in self.metrics)
        gates_passed = sum(1 for m in self.metrics if m.passed)
        total_tokens = sum(m.tokens_used for m in self.metrics)

        return {
            "total_latency_ms": round(total_latency, 1),
            "gates_passed": f"{gates_passed}/{len(self.metrics)}",
            "total_tokens": total_tokens,
            "per_gate": [{"name": m.gate_name, "latency_ms": m.latency_ms, "passed": m.passed} for m in self.metrics],
        }
```

**👹 Boss Lesson:** A complete AI system is not one model — it's an orchestrated pipeline of specialized components. Each gate has one job and does it well.

---

## Dungeon 2 — The Capstone Project

**Day 2 | Concept:** Build something real. The capstone project is not a demo — it solves a genuine problem using everything you've learned.

**Capstone Project Options:**

1. **AI Customer Support Agent**
   - RAG over product documentation
   - Multi-step agent (search → analyze → respond)
   - Guardrails for safety and accuracy
   - Evaluation pipeline for quality
   - Production deployment

2. **Code Review Assistant**
   - Fine-tuned on code review patterns
   - Tool use (checkout PR, run tests)
   - Agentic multi-step review
   - Production CI integration

3. **Research Assistant**
   - Web search + RAG hybrid
   - Multi-source synthesis
   - Citation management
   - Report generation

**🗡️ Dungeon Mastery:** Write a 1-page project proposal. Include: problem statement, system diagram, tech stack, gate-by-gate breakdown, evaluation criteria, deployment plan.

**💻 Code — Capstone Scaffold:**
```python
from dataclasses import dataclass, field
from enum import Enum

class CapstoneStatus(Enum):
    PLANNING = "planning"
    BUILDING = "building"
    TESTING = "testing"
    DEPLOYED = "deployed"
    ITERATING = "iterating"

@dataclass
class CapstoneProject:
    name: str
    problem: str
    tech_stack: list[str]
    gates_used: list[str]
    eval_criteria: list[dict]  # [{"name": "...", "target": 0.9}]
    status: CapstoneStatus = CapstoneStatus.PLANNING

    def validate(self) -> list[str]:
        """Check capstone covers all required gates."""
        required_gates = [
            "Prompt Engineering", "Embeddings", "RAG", "Advanced RAG",
            "Agents", "Evaluation", "Guardrails", "Fine-Tuning", "Deployment",
        ]
        missing = [g for g in required_gates if g not in self.gates_used]
        warnings = []

        if missing:
            warnings.append(f"Missing gates: {', '.join(missing)}")

        if len(self.eval_criteria) < 3:
            warnings.append("Need at least 3 evaluation criteria")

        if len(self.tech_stack) < 4:
            warnings.append("Need a more complete tech stack")

        return warnings

    def project_plan(self) -> str:
        gates_summary = "\n".join(f"  - {g}" for g in self.gates_used)
        criteria_summary = "\n".join(f"  - {c['name']}: target {c['target']:.0%}" for c in self.eval_criteria)
        return f"""
# {self.name}

## Problem
{self.problem}

## Gates Applied
{gates_summary}

## Evaluation Criteria
{criteria_summary}

## Tech Stack
{', '.join(self.tech_stack)}
"""


# Example: Customer Support Agent
support_agent = CapstoneProject(
    name="AI Customer Support Agent",
    problem="Customer support team spends 60% of time answering repetitive questions about product features, billing, and troubleshooting.",
    tech_stack=["FastAPI", "ChromaDB", "GPT-4o-mini", "Docker", "LangChain", "Redis", "Prometheus"],
    gates_used=[
        "Prompt Engineering", "Embeddings", "Vector Databases",
        "RAG Foundations", "Advanced RAG", "Agents & Tool Use",
        "Evaluation & Observability", "Guardrails & Safety",
        "Fine-Tuning", "Production Deployment",
    ],
    eval_criteria=[
        {"name": "Answer Accuracy", "target": 0.9},
        {"name": "Retrieval Hit Rate@3", "target": 0.85},
        {"name": "Guardrail Block Rate (Bad Queries)", "target": 0.95},
        {"name": "p95 Latency", "target": 0.9},  # <2s for 90% of queries
    ],
)
```

**👹 Boss Lesson:** A capstone project is not a checkbox exercise. It's proof that you can build real AI systems that solve real problems.

---

## Dungeon 3 — The Decision Matrix

**Day 3 | Concept:** Every AI engineering decision is a trade-off. The matrix helps you make systematic choices.

**The Universal Decision Matrix:**
| Factor | Weight | Option A | Option B | Winner |
|--------|--------|----------|----------|--------|
| Cost | 0.3 | 5 | 3 | A |
| Quality | 0.4 | 4 | 5 | B |
| Latency | 0.2 | 5 | 3 | A |
| Complexity | 0.1 | 3 | 4 | B |
| Total | 1.0 | 4.5 | 3.9 | A |

**Key Decisions You'll Face:**
| Decision | Options | Key Trade-off |
|----------|---------|---------------|
| RAG vs Fine-tune | RAG, FT, Both | Accuracy vs Cost |
| Model size | 7B, 13B, 70B, API | Quality vs Latency |
| Chunk size | 256, 512, 1024 | Precision vs Context |
| Embedding model | MiniLM, MPNet, OpenAI | Quality vs Speed |
| Guardrail strictness | Strict, Tiered, Flag | Safety vs User experience |

**🗡️ Dungeon Mastery:** Build a decision matrix for "Which model to use for my customer support chatbot?" Options: GPT-4o, GPT-4o-mini, Llama 3.1 8B. Weights: Cost 0.4, Quality 0.3, Latency 0.2, Control 0.1. Calculate the winner.

**💻 Code — Decision Matrix:**
```python
from dataclasses import dataclass, field

@dataclass
class DecisionFactor:
    name: str
    weight: float  # 0.0 to 1.0

@dataclass
class Option:
    name: str
    scores: dict[str, float]  # {"factor_name": 1-5}

class DecisionMatrix:
    def __init__(self, factors: list[DecisionFactor]):
        total_weight = sum(f.weight for f in factors)
        assert abs(total_weight - 1.0) < 0.001, f"Weights must sum to 1.0 (got {total_weight})"
        self.factors = factors

    def evaluate(self, options: list[Option]) -> list[dict]:
        results = []
        for option in options:
            total = 0
            breakdown = {}
            for factor in self.factors:
                score = option.scores.get(factor.name, 1)
                weighted = score * factor.weight
                total += weighted
                breakdown[factor.name] = {"score": score, "weighted": weighted}

            results.append({
                "option": option.name,
                "total_score": total,
                "breakdown": breakdown,
            })

        results.sort(key=lambda x: x["total_score"], reverse=True)
        for i, r in enumerate(results):
            r["rank"] = i + 1

        return results


# Example: Model selection
factors = [
    DecisionFactor("cost", 0.4),
    DecisionFactor("quality", 0.3),
    DecisionFactor("latency", 0.2),
    DecisionFactor("control", 0.1),
]

options = [
    Option("GPT-4o", {"cost": 2, "quality": 5, "latency": 3, "control": 1}),
    Option("GPT-4o-mini", {"cost": 4, "quality": 4, "latency": 4, "control": 1}),
    Option("Llama 3.1 8B (self-host)", {"cost": 4, "quality": 3, "latency": 3, "control": 5}),
]

matrix = DecisionMatrix(factors)
results = matrix.evaluate(options)
for r in results:
    print(f"#{r['rank']} {r['option']}: {r['total_score']:.2f}")
```

**👹 Boss Lesson:** If you can't describe a decision in terms of explicit trade-offs, you haven't thought about it enough. The matrix doesn't decide for you — it clarifies your thinking.

---

## Dungeon 4 — The Road Ahead

**Day 4 | Concept:** AI engineering evolves fast. The skills you've learned are foundations, not destinations.

**What's Next:**
| Emerging Area | Why It Matters | When |
|---------------|---------------|------|
| Multi-modal RAG | Images, audio, video retrieval | Now |
| AI agents in production | Autonomous decision-making | Now |
| Small models (3B-8B) | On-device, private, fast | Now |
| Reasoning models (o1, o3) | Better at complex tasks | Now |
| Agent-to-agent communication | Multi-agent orchestration | Emerging |
| Test-time compute scaling | Better answers with more thinking | Emerging |

**The Engineer's Growth Path:**
```
Gate 01: AI Engineering Overview
Gate 02: Prompt Engineering    → You speak the language
Gate 03: Embeddings & Vectors  → You understand meaning
Gate 04-05: RAG               → You connect knowledge
Gate 06: Agents               → You build autonomy
Gate 07: Evaluation           → You measure quality
Gate 08: Guardrails           → You ensure safety
Gate 09: Fine-Tuning          → You shape models
Gate 10: Deployment           → You ship products
Gate 11: Capstone             → You integrate everything

You are now: A Full-Stack AI Engineer
```

**🗡️ Dungeon Mastery:** Write your own "Next 90 Days" learning plan. 3 skills to deepen, 1 project to build, 1 paper to read per week.

**💻 Code — Learning Tracker:**
```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta

@dataclass
class LearningGoal:
    skill: str
    current_level: int  # 1-5
    target_level: int
    resources: list[str] = field(default_factory=list)
    deadline: str = ""

@dataclass
class WeeklyPlan:
    week: int
    focus: str
    actions: list[str]
    metric: str

class ContinuingEducation:
    def __init__(self):
        self.goals: list[LearningGoal] = []
        self.weekly_plans: list[WeeklyPlan] = []

    def add_goal(self, goal: LearningGoal):
        self.goals.append(goal)

    def plan_next_12_weeks(self) -> list[WeeklyPlan]:
        plans = []
        weeks = [
            ("Deepen RAG", ["Build multi-modal RAG", "Implement CRAG", "A/B test chunk sizes"], "Hit Rate +5%"),
            ("Production Agents", ["Deploy agent to prod", "Add tracing", "Handle 5 failure modes"], "Agent success rate >90%"),
            ("Advanced Eval", ["Build LLM-as-judge pipeline", "Create 50-case dataset", "Automate in CI"], "Eval coverage 100%"),
        ]
        for i, (focus, actions, metric) in enumerate(weeks, 1):
            for j in range(4):  # 4 weeks per focus
                week_num = (i - 1) * 4 + j
                if week_num > 12:
                    break
                plans.append(WeeklyPlan(
                    week=week_num,
                    focus=focus if j == 0 else f"{focus} (cont.)",
                    actions=actions if j == 0 else ["Continue previous week's work"],
                    metric=metric,
                ))
        return plans

    def status_report(self) -> str:
        lines = ["## AI Engineering Growth Report\n"]
        for goal in self.goals:
            progress = "■" * goal.current_level + "□" * (5 - goal.current_level)
            lines.append(f"{goal.skill}: {progress} ({goal.current_level}→{goal.target_level})")
        return "\n".join(lines)


# Example: Post-capstone learning plan
education = ContinuingEducation()
education.add_goal(LearningGoal("Multi-modal RAG", 2, 4, ["CLIP paper", "Llava docs", "Build image search"]))
education.add_goal(LearningGoal("Production Agents", 3, 5, ["LangGraph", "CrewAI", "Build multi-agent system"]))
education.add_goal(LearningGoal("Model Deployment", 2, 4, ["vLLM", "Triton", "Deploy 2 models"]))
weekly_plans = education.plan_next_12_weeks()
```

**👹 Boss Lesson:** Mastery is not a destination. It's a direction. The moment you stop learning in AI engineering is the moment you become obsolete.

---

## ⚔️ FINAL BOSS BATTLE: The Monarch's Trial

**Objective:** Design a complete AI system proposal. Your proposal must demonstrate mastery of ALL 11 gates.

**System Requirements:**
1. **Problem statement** — What real problem does this solve?
2. **Architecture** — Complete system diagram with all gates mapped
3. **Core pipeline** — RAG + Agent hybrid with guardrails
4. **Evaluation** — Automated eval suite with ≥ 4 criteria
5. **Fine-tuning plan** — When and how to fine-tune
6. **Deployment** — Production architecture with scaling
7. **Decision rationale** — Key decisions with trade-off analysis
8. **Growth plan** — How the system improves over time

**🗡️ Victory Conditions:**
- All 11 gates are explicitly addressed in the proposal
- Architecture has no single point of failure
- Evaluation criteria are specific and measurable
- Fine-tuning justification is clear (why not RAG alone)
- Deployment plan is complete (containerization, CI/CD, monitoring)
- Decision matrix is used for at least 2 key decisions
- The system is buildable by a single engineer in 4-8 weeks

**🏆 Rewards:**
- +1000 XP
- A+ Rank → S Rank
- Skill Unlock: **Monarch's Wisdom** — All future AI system designs are 2x more effective
- Final Title: **THE MONARCH OF AI ENGINEERING**

🎉 **Congratulations, Monarch!** 🎉

You have conquered all 11 gates and mastered the complete arc of AI Engineering — from the first prompt to the deployed production system. The throne is yours.

---

*"Eleven gates you have crossed. Eleven lessons you have learned. But the true journey — building systems that make a difference — begins now. Go forth and build."*

— The Architect of the Gates
