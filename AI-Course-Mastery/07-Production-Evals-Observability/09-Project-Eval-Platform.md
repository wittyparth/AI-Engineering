# 09 — Project: AI Evaluation Platform

## 🎯 Project Overview

This is your **Phase 7 capstone**. You will build a **production-grade AI Evaluation Platform** — a complete system for measuring, monitoring, and maintaining the quality of AI systems across your entire organization.

This isn't a notebook that runs evals for one chatbot. This is an **evaluation infrastructure platform** — the kind used by companies like Cohere, Anthropic, and the AI teams at Uber/Stripe to ensure every model change, prompt update, or system deployment is measured before it reaches users.

By the end of this project, you'll have a portfolio piece that demonstrates:

- **Evaluation architecture**: Eval run orchestration, dataset management, metric computation
- **LLM-as-Judge infrastructure**: Multi-judge ensemble, bias detection, calibration
- **Metrics platform**: Hierarchical metrics (point-level → system-level → business-level)
- **Production pipeline design**: Parallel execution, caching, tiered gating, flaky detection
- **Observability integration**: Tracing, cost tracking, production-to-eval feedback loops
- **Regression detection**: Statistical shift detection, trend analysis, automated alerting
- **Dashboard & reporting**: Multi-stakeholder dashboards, weekly summaries, drill-down
- **CI/CD integration**: GitHub Actions plugin, CLI client, Slack notifications
- **Production engineering**: FastAPI backend, async evaluation, rate limiting, auth

### Learning Objectives

1. Design and build a complete evaluation platform from scratch (no eval-as-a-service vendor)
2. Implement the full eval run lifecycle — dataset selection, execution, judging, aggregation, gating
3. Build a multi-judge LLM-as-Judge pipeline with bias mitigation
4. Integrate with Langfuse or OpenTelemetry for production tracing
5. Implement CUSUM and moving-average regression detection
6. Design dashboards for different stakeholders (engineers, PMs, executives)
7. Build CI/CD integration that blocks regressions before deployment
8. Deploy the full platform with Docker Compose

---

### 🛑 STOP. Before you start, answer these design questions.

**Question 1 — The Platform Scope Problem**

You have 3 AI systems to evaluate:
1. A **RAG chatbot** — cares about faithfulness, relevance, answer completeness
2. A **content classifier** — cares about precision, recall, latency
3. A **code generation assistant** — cares about correctness, style adherence, compile rate

Each system needs DIFFERENT metrics, DIFFERENT test datasets, and DIFFERENT judges. But they need to share the SAME platform — same database, same dashboard, same alerting.

**🤔 Your design challenge:**
1. How does your platform's DATA MODEL handle system-specific metrics while sharing infrastructure?
2. Should each system have its OWN test dataset, or should there be shared "general AI quality" cases?
3. How do you design the dashboard so an engineer sees ONLY their system, but the CTO sees ALL systems?

**Write your approach before reading the project spec.** Your answers will shape every architectural decision.

---

**Question 2 — The Evaluation Pipeline Bottleneck**

Your eval platform runs 500 test cases per system, across 3 systems — 1,500 evaluations total. Each eval requires an LLM call for judging (in addition to the system under test's own LLM call).

At 2 LLM calls per test case (system + judge), that's 3,000 LLM calls per full eval run. At ~$0.002 per call (GPT-4o-mini), that's $6 per full run.

You deploy 10 times per day across all systems. That's 30,000 LLM calls/day, $60/day, ~$1,800/month. PLUS the cost of the system under test's own LLM calls.

**🤔 Your design challenge:**
1. How do you make this AFFORDABLE? What caching strategies reduce judge calls?
2. How do you decide WHICH test cases to run on every deploy vs. nightly vs. weekly?
3. If an engineer triggers an eval on their branch, who pays for it? (Team budget? Central budget?)
4. What's the cost of NOT running evals? (Hint: a single regression that takes 2 engineers 3 days to debug costs way more than $1,800.)

**Write your cost optimization strategy before reading on.**

---

**Question 3 — The Calibration Crisis**

Your platform uses an LLM judge. It works well for 6 months. Then you update the judge from GPT-4o-mini to GPT-4o (better quality, lower cost because fewer re-reads). Suddenly, scores DROP by 5% across ALL systems.

But the systems haven't changed — only the JUDGE changed. The new judge is STRICTER.

**🤔 Your design challenge:**
1. How do you DISTINGUISH a "system regression" (the AI got worse) from a "judge shift" (the evaluator got stricter)?
2. How would you CALIBRATE the new judge so its scores are COMPARABLE to the old judge's scores?
3. What metadata must you collect at eval time to make this distinction POSSIBLE?
4. What if the old judge was TOO LENIENT, and the new judge is CORRECTLY stricter — how do you convince stakeholders that "lower scores" actually mean "better evaluation" and not "worse AI"?

**Write your calibration and change-detection strategy before reading the project spec.**

---

## ⏱ Time Budget

| Milestone | Estimated Time |
|-----------|---------------|
| Milestone 0: Setup & Core Data Model | 3-4 hours |
| Milestone 1: Eval Run Engine | 6-8 hours |
| Milestone 2: LLM-as-Judge Pipeline | 6-8 hours |
| Milestone 3: Dashboard & Reporting | 6-8 hours |
| Milestone 4: Observability & Regression Detection | 6-8 hours |
| Milestone 5: CI/CD Integration & Production Polish | 6-8 hours |
| Milestone 6: Deployment & Portfolio Prep | 3-4 hours |
| **Total** | **36-48 hours** |

---

## 📋 Project Specification

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         EVAL PLATFORM                                │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                        API LAYER (FastAPI)                    │    │
│  │  POST /runs  │  GET /runs/{id}  │  GET /dashboard/{system}   │    │
│  │  POST /results │  GET /trends  │  POST /alerts/config        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│           │                    │                    │                │
│           ▼                    ▼                    ▼                │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │ RUN ENGINE   │  │ LLM JUDGE        │  │ OBSERVABILITY        │  │
│  │              │  │ PIPELINE         │  │                       │  │
│  │ • Orchestrate│  │ • Multi-judge    │  │ • Langfuse tracing    │  │
│  │ • Parallelize│  │ • Calibration    │  │ • Cost tracking       │  │
│  │ • Cache      │  │ • Bias detection │  │ • Feedback loop       │  │
│  │ • Gate logic │  │ • Ensemble       │  │ • Alerting            │  │
│  └──────────────┘  └──────────────────┘  └──────────────────────┘  │
│           │                    │                    │                │
│           ▼                    ▼                    ▼                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    STORAGE LAYER (SQLite/Postgres)           │    │
│  │  eval_runs │ test_cases │ eval_results │ metric_snapshots    │    │
│  │  baselines │ alerts     │ teams        │ system_configs      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                   INTEGRATION LAYER                           │    │
│  │  CLI Tool  │  GitHub Action  │  Slack Bot  │  Python SDK     │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Responsibility | Phase 7 Reference |
|-----------|---------------|-------------------|
| **Eval Run Engine** | Orchestrate eval runs — dataset selection, parallel execution, result aggregation | File 05 (Eval Pipelines) |
| **LLM Judge Pipeline** | Evaluate outputs using LLMs as judges, with bias detection and calibration | File 02 (LLM as Judge) |
| **Metrics Engine** | Compute metrics at 3 levels: point-level, run-level, trend-level | File 03 (Metrics That Matter) |
| **Dataset Manager** | Version and curate evaluation datasets, synthetic data generation | File 04 (Eval Datasets) |
| **Observability Module** | Production tracing, cost tracking, user feedback collection | File 06 (Observability) |
| **Regression Detector** | Statistical shift detection, trend analysis, automated alerting | File 07 (Regression Detection) |
| **Dashboard API** | Data endpoints for frontend, multi-stakeholder views | File 08 (Eval Framework) |
| **Integration SDK** | CLI tool, Python client, GitHub Action, Slack notifier | File 08 (Eval Framework) |

### Tool Requirements

Your platform must include these capabilities:

| Capability | What It Does | Key Files |
|------------|-------------|-----------|
| `create_run()` | Initialize an eval run with dataset + system config | File 05 |
| `execute_run()` | Parallel eval with caching, tiered execution, timeout | File 05 |
| `judge_output()` | LLM-as-Judge with multi-judge ensemble, bias detection | File 02 |
| `compute_metrics()` | Point → run → trend metric computation | File 03 |
| `manage_dataset()` | CRUD for test cases, versioning, synthetic generation | File 04 |
| `trace_run()` | Record run in Langfuse with spans and cost tracking | File 06 |
| `detect_regression()` | CUSUM + moving average shift detection | File 07 |
| `check_gate()` | Gate decision (pass/warn/block) with configurable rules | File 05/08 |
| `send_alert()` | Notify Slack/email/PagerDuty with formatted alerts | File 08 |
| `generate_report()` | Weekly summary, regression report, trend analysis | File 08 |

### Constraints & Requirements

1. **No eval-as-a-service vendor**: Build your own LLM judge. No Galileo, Arize, or similar.
2. **Support at least 2 different system types**: Design the data model to handle RAG systems AND non-RAG systems (classifiers, generators, agents).
3. **Eval run must be reproducible**: Same dataset + same system config + same judge config = same scores.
4. **Cost tracking required**: Every eval run must report its cost (judge LLM calls + system LLM calls).
5. **CI/CD integration**: At minimum, a CLI tool that exits with code 0/1 based on gate decision.
6. **Flaky test detection**: The platform must identify and flag test cases with high variance.
7. **Judge calibration**: When updating the judge, the platform must detect score shifts caused by the judge change (not the system change).
8. **Docker Compose deployment**: The entire platform runs via `docker compose up`.
9. **Async evaluation**: The API must handle concurrent eval runs without blocking.
10. **Graceful degradation**: If the LLM judge API fails, the platform should retry (with backoff) or use a fallback judge (e.g., exact match for classification tasks).

---

## Milestone 0: Setup & Core Data Model

### Step 1: Environment Setup

```bash
# Create project structure
mkdir ai-eval-platform && cd ai-eval-platform
mkdir -p api core judge datasets storage integration dashboard
mkdir -p tests/unit tests/integration tests/fixtures
mkdir -p docker

# Core dependencies
pip install fastapi uvicorn pydantic sqlalchemy aiosqlite httpx

# LLM dependencies  
pip install openai anthropic

# Observability
pip install langfuse opentelemetry-api opentelemetry-sdk

# Dashboard (pick one)
# Option A: FastAPI + Jinja2 templates (simpler)
# Option B: FastAPI + React frontend (more impressive)
pip install jinja2  # if Option A

# Testing
pip install pytest pytest-asyncio
```

### Step 2: Core Data Model

Build on the schema from File 08. Your project should implement it as actual SQLAlchemy models:

```python
"""
core/models.py

SQLAlchemy models for the eval platform.

🤔 DESIGN DECISION: SQLAlchemy vs raw SQL?

SQLAlchemy: More code up front, but gives you:
- Migration support (Alembic)
- Relationship loading (lazy/eager)
- Type coercion
- Async support

Raw SQL: Less magic, but:
- You write every query
- No automatic migration support
- Manual type handling

This project uses SQLAlchemy for migrations and async support.
"""

from datetime import datetime
from typing import Optional
import uuid

from sqlalchemy import (
    Column, String, Float, Integer, Boolean, DateTime,
    ForeignKey, JSON, Text, create_engine, Index,
)
from sqlalchemy.orm import DeclarativeBase, relationship
from sqlalchemy.sql import func


class Base(DeclarativeBase):
    pass


class EvalRun(Base):
    __tablename__ = "eval_runs"
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    system_name = Column(String, nullable=False)
    system_version = Column(String, nullable=False)
    
    # Config (JSON serialized)
    model_config = Column(JSON, default=dict)
    prompt_config = Column(JSON, nullable=True)
    judge_config = Column(JSON, default=dict)
    
    # Git context
    commit_hash = Column(String, nullable=True)
    branch = Column(String, nullable=True)
    pr_number = Column(String, nullable=True)
    author = Column(String, nullable=True)
    
    # Trigger info
    triggered_by = Column(String, default="manual")  # "ci/cd", "manual", "scheduled"
    trigger_metadata = Column(JSON, default=dict)
    
    # Status
    status = Column(String, default="pending")  # pending, running, completed, failed, partial
    started_at = Column(DateTime, default=func.now())
    completed_at = Column(DateTime, nullable=True)
    
    # Run-level aggregate metrics
    total_cases = Column(Integer, default=0)
    passed = Column(Integer, default=0)
    failed = Column(Integer, default=0)
    avg_score = Column(Float, default=0.0)
    
    # Cost tracking
    total_cost = Column(Float, default=0.0)
    total_tokens = Column(Integer, default=0)
    
    # Team/namespace
    team_id = Column(String, default="default")
    
    # Metadata
    metadata = Column(JSON, default=dict)
    
    # Relationships
    results = relationship("EvalResult", back_populates="eval_run", lazy="selectin")
    metric_snapshots = relationship("MetricSnapshot", back_populates="eval_run", lazy="selectin")
    
    __table_args__ = (
        Index("idx_runs_system_time", "system_name", "started_at"),
        Index("idx_runs_status", "status"),
        Index("idx_runs_team", "team_id", "system_name"),
    )


class TestCase(Base):
    __tablename__ = "test_cases"
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    system_name = Column(String, nullable=False)
    input = Column(Text, nullable=False)
    expected_output = Column(Text, nullable=True)
    
    category = Column(String, default="standard")  # critical, standard, informational
    difficulty = Column(String, default="medium")  # easy, medium, hard
    tags = Column(JSON, default=list)
    
    # Dataset versioning
    dataset_version = Column(String, default="1.0")
    source = Column(String, default="manual")  # manual, synthetic, production, human_curated
    
    metadata = Column(JSON, default=dict)
    deprecated = Column(Boolean, default=False)
    
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
    
    __table_args__ = (
        Index("idx_tc_system", "system_name", "dataset_version"),
        Index("idx_tc_category", "system_name", "category"),
    )


class EvalResult(Base):
    __tablename__ = "eval_results"
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    eval_run_id = Column(String, ForeignKey("eval_runs.id"), nullable=False)
    test_case_id = Column(String, ForeignKey("test_cases.id"), nullable=False)
    
    # Evaluation
    passed = Column(Boolean, nullable=False)
    score = Column(Float, nullable=False)
    
    # Outputs
    output = Column(Text, nullable=False)  # System's actual output
    expected_output = Column(Text, nullable=True)  # Snapshot at eval time
    
    # Performance
    latency_ms = Column(Float, nullable=True)
    token_count = Column(Integer, nullable=True)
    cost = Column(Float, nullable=True)
    
    # Error handling
    error = Column(Text, nullable=True)
    retry_count = Column(Integer, default=0)
    
    # Judge info
    judge_type = Column(String, default="llm")  # llm, exact, semantic, human
    judge_model = Column(String, nullable=True)
    judge_config = Column(JSON, default=dict)
    judge_reasoning = Column(Text, nullable=True)  # Judge's reasoning for score
    
    # Timing
    started_at = Column(DateTime, nullable=True)
    completed_at = Column(DateTime, nullable=True)
    
    # Relationships
    eval_run = relationship("EvalRun", back_populates="results")
    
    __table_args__ = (
        Index("idx_result_run", "eval_run_id"),
        Index("idx_result_score", "score"),
    )


class MetricSnapshot(Base):
    __tablename__ = "metric_snapshots"
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    eval_run_id = Column(String, ForeignKey("eval_runs.id"), nullable=False)
    
    metric_name = Column(String, nullable=False)
    metric_value = Column(Float, nullable=False)
    
    # Distribution
    p50 = Column(Float, nullable=True)
    p95 = Column(Float, nullable=True)
    p99 = Column(Float, nullable=True)
    std = Column(Float, nullable=True)
    
    sample_size = Column(Integer, nullable=True)
    computed_at = Column(DateTime, default=func.now())
    
    # Relationships
    eval_run = relationship("EvalRun", back_populates="metric_snapshots")
    
    __table_args__ = (
        Index("idx_metric_name_time", "metric_name", "computed_at"),
    )


class AlertConfig(Base):
    __tablename__ = "alert_configs"
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    system_name = Column(String, nullable=False)
    metric_name = Column(String, nullable=False)
    
    alert_type = Column(String, nullable=False)  # below_threshold, drop_vs_baseline, zscore
    threshold = Column(Float, nullable=False)
    cooldown_minutes = Column(Integer, default=60)
    
    enabled = Column(Boolean, default=True)
    channels = Column(JSON, default=list)  # ["slack", "email", "pagerduty"]
    
    created_at = Column(DateTime, default=func.now())
```

---

## Milestone 1: Eval Run Engine

The **Eval Run Engine** orchestrates the full evaluation lifecycle:
1. Create run (select dataset, configure system)
2. Execute test cases (parallel, cached, tiered)
3. Judge outputs (LLM-as-Judge)
4. Aggregate results (compute metrics)
5. Check gate (pass/warn/block)
6. Store results (persist everything)

### Step 1: Run Orchestrator

```python
"""
core/run_engine.py

The central orchestrator for evaluation runs.

🤔 DESIGN PATTERN: This uses the "Strategy" pattern.
Different systems (RAG, classifier, agent) can have
DIFFERENT evaluation strategies while sharing the SAME
orchestration framework.

The run engine doesn't care WHAT you evaluate — it just
orchestrates the lifecycle.
"""

import asyncio
import time
import uuid
from datetime import datetime
from typing import Optional, Callable
from dataclasses import dataclass


@dataclass
class RunConfig:
    """Configuration for a single eval run."""
    system_name: str
    system_version: str
    model_config: dict
    prompt_config: Optional[dict] = None
    judge_config: Optional[dict] = None
    
    # Dataset selection
    dataset_version: str = "latest"
    categories: Optional[list[str]] = None  # None = all
    max_cases: Optional[int] = None  # Limit for quick runs
    
    # Execution
    max_concurrency: int = 20
    timeout_per_case: int = 60  # seconds
    cache_results: bool = True
    
    # Gate
    pass_threshold: float = 0.8
    require_all_critical: bool = True
    
    # Git context
    commit_hash: Optional[str] = None
    branch: Optional[str] = None
    pr_number: Optional[str] = None
    author: Optional[str] = None


class RunEngine:
    """
    Orchestrates evaluation runs.
    
    The lifecycle:
    1. CREATE: Initialize the run in the database
    2. LOAD: Load test cases based on config
    3. EVAL: Execute all test cases (parallel, tiered)
    4. JUDGE: Score each output
    5. AGGREGATE: Compute run-level metrics
    6. GATE: Check pass/fail criteria
    7. STORE: Persist everything
    
    ⚠️ DESIGN FLAW: This engine evaluates ALL test cases
    regardless of cost. A full run with 500 cases costs
    ~$6 in judge calls. A more sophisticated engine would:
    
    - Run CRITICAL cases first → if any fail, BLOCK early (save $)
    - Run EASY cases → if these fail, something is fundamentally broken
    - Run HARD cases → most informative, but also most expensive
    
    Can you implement this "tiered early-termination" pattern?
    """
    
    def __init__(self, db_session, judge_pipeline, cache=None):
        self.db = db_session
        self.judge = judge_pipeline
        self.cache = cache
    
    async def run(self, config: RunConfig) -> dict:
        """Execute a full evaluation run."""
        run_id = str(uuid.uuid4())
        
        # Phase 1: Load test cases
        cases = await self._load_cases(config)
        
        # Phase 2: Evaluate all cases
        start_time = time.time()
        results = await self._execute_all(cases, config)
        duration = time.time() - start_time
        
        # Phase 3: Compute aggregates
        metrics = self._compute_metrics(results, duration)
        
        # Phase 4: Check gate
        gate_result = self._check_gate(metrics, config)
        
        # Phase 5: Store everything
        stored_run = await self._store_results(
            run_id, config, results, metrics, gate_result
        )
        
        return {
            "run_id": run_id,
            "system_name": config.system_name,
            "system_version": config.system_version,
            "status": "completed" if gate_result["passed"] else "partial",
            "total_cases": len(cases),
            "passed": metrics["passed"],
            "failed": metrics["failed"],
            "avg_score": metrics["avg_score"],
            "duration_seconds": duration,
            "total_cost": sum(r.get("cost", 0) for r in results),
            "gate": gate_result,
        }
    
    async def _load_cases(self, config: RunConfig) -> list:
        """Load test cases based on config."""
        # In production, this queries the database
        # For now, return a sample set
        return []  # Placeholder
    
    async def _execute_all(self, cases: list, config: RunConfig) -> list:
        """Execute all test cases in parallel with concurrency limit."""
        semaphore = asyncio.Semaphore(config.max_concurrency)
        
        async def execute_one(case):
            async with semaphore:
                return await self._evaluate_single(case, config)
        
        tasks = [execute_one(case) for case in cases]
        return await asyncio.gather(*tasks)
    
    async def _evaluate_single(self, case: dict, config: RunConfig) -> dict:
        """Evaluate a single test case."""
        # 1. Run system under test
        system_output = await self._call_system(case["input"], config)
        
        # 2. Judge the output
        judge_result = await self.judge.evaluate(
            input=case["input"],
            output=system_output,
            expected=case.get("expected_output"),
        )
        
        return {
            "test_case_id": case["id"],
            "passed": judge_result["score"] >= 0.7,
            "score": judge_result["score"],
            "output": system_output,
            "judge_reasoning": judge_result.get("reasoning", ""),
            "cost": judge_result.get("cost", 0),
        }
    
    async def _call_system(self, input_text: str, config: RunConfig) -> str:
        """
        Call the system under test.
        
        🤔 DESIGN QUESTION: How does the run engine know HOW
        to call the system under test?
        
        Options:
        A. The system registers a callback function
        B. The system exposes a standard API endpoint
        C. The engine uses a plugin system
        
        This version uses option A (callback), which is simple
        but means the system must be in the same process.
        """
        # In production, this would call the actual system
        # For now, return a placeholder
        return "[system output placeholder]"
    
    def _compute_metrics(self, results: list, duration: float) -> dict:
        """Compute aggregate metrics from individual results."""
        total = len(results)
        passed = sum(1 for r in results if r["passed"])
        scores = [r["score"] for r in results]
        costs = [r.get("cost", 0) for r in results]
        
        sorted_scores = sorted(scores)
        
        return {
            "total": total,
            "passed": passed,
            "failed": total - passed,
            "pass_rate": passed / max(total, 1),
            "avg_score": sum(scores) / max(len(scores), 1),
            "median_score": sorted_scores[len(sorted_scores) // 2] if sorted_scores else 0,
            "p95_score": sorted_scores[int(len(sorted_scores) * 0.95)] if sorted_scores else 0,
            "min_score": min(scores) if scores else 0,
            "max_score": max(scores) if scores else 0,
            "total_cost": sum(costs),
            "total_duration": duration,
        }
    
    def _check_gate(self, metrics: dict, config: RunConfig) -> dict:
        """Check whether the run passes the gate."""
        passed = True
        reasons = []
        
        if metrics["avg_score"] < config.pass_threshold:
            passed = False
            reasons.append(
                f"Average score {metrics['avg_score']:.3f} below threshold "
                f"{config.pass_threshold}"
            )
        
        if config.require_all_critical and metrics.get("critical_failed", 0) > 0:
            passed = False
            reasons.append(
                f"{metrics['critical_failed']} critical test cases failed"
            )
        
        return {
            "passed": passed,
            "decision": "pass" if passed else "block",
            "reasons": reasons,
        }
    
    async def _store_results(self, run_id, config, results, metrics, gate):
        """Store all results in the database."""
        # In production, persist via SQLAlchemy
        return {"id": run_id, "status": "completed"}
```

### Step 2: Implement Tiered Execution

Your run engine currently runs ALL test cases in parallel. Add a **tiered execution** that:

1. Runs critical cases FIRST (parallel batch)
2. If any critical case fails → BLOCK immediately (stop, don't run standard/info cases)
3. If all critical pass → run STANDARD cases
4. If pass rate on standard is > 90% → run INFORMATIONAL cases
5. If pass rate on standard is < 70% → BLOCK, don't waste money on info cases

**🤔 Your task:** Modify `_execute_all()` to support this tiered execution. What's the decision logic? How much does this save per run (in cost and time)?

---

## Milestone 2: LLM-as-Judge Pipeline

Build the judge pipeline from File 02 — but with production-grade features:

### Step 1: Multi-Judge Ensemble

```python
"""
judge/pipeline.py

LLM-as-Judge pipeline with multi-judge ensemble.

A SINGLE judge has biases (File 02). Three judges
voting reduces bias. If all three agree, trust the result.
If they disagree, flag for human review.

🤔 COST CONSIDERATION: Each judge costs ~$0.002 (GPT-4o-mini).
Three judges per test case × 500 cases = $3 per run.
Is this worth it vs. single judge at $1 per run?

It depends on the COST OF BEING WRONG. For a customer-facing
chatbot where a wrong answer costs $100 in support tickets,
$2 extra per run is trivial.

For an internal tool where wrong answers are low-impact,
single judge is fine.
"""

from typing import Optional
import asyncio


class JudgeResult:
    """Result from a single judge evaluation."""
    score: float
    passed: bool
    reasoning: str
    judge_model: str
    cost: float
    confidence: float  # How confident the judge is (0-1)


class LLMJudge:
    """
    Single LLM judge.
    
    Evaluates input/output pairs and returns a score.
    """
    
    def __init__(self, model: str, system_prompt: str, temperature: float = 0.0):
        self.model = model
        self.system_prompt = system_prompt
        self.temperature = temperature
    
    async def evaluate(
        self,
        input_text: str,
        output: str,
        expected: Optional[str] = None,
    ) -> JudgeResult:
        """
        Evaluate an output using the LLM judge.
        
        Returns a score 0-1, reasoning, and cost.
        
        🤔 BIAS WARNING: If you ALWAYS use the same judge prompt,
        the judge will develop SYSTEMATIC BIASES (File 02).
        
        For example, a judge that always starts with
        "Evaluate the QUALITY of this response..." will
        systematically prefer verbose answers over concise ones.
        
        MITIGATION: Rotate through MULTIPLE judge prompt variants.
        """
        # In production, this calls the LLM API
        # Placeholder implementation
        return JudgeResult(
            score=0.92,
            passed=True,
            reasoning="The response is accurate and complete.",
            judge_model=self.model,
            cost=0.002,
            confidence=0.85,
        )


class MultiJudgeEnsemble:
    """
    Multiple judges, one evaluation.
    
    Uses k judges (default: 3) to evaluate each output.
    If judges agree, use the average score.
    If judges disagree (high variance), flag for human review.
    """
    
    def __init__(self, judges: list[LLMJudge]):
        if len(judges) < 2:
            raise ValueError("Ensemble needs at least 2 judges")
        self.judges = judges
    
    async def evaluate(
        self,
        input_text: str,
        output: str,
        expected: Optional[str] = None,
    ) -> dict:
        """
        Evaluate using all judges and aggregate results.
        
        Returns:
        - score: Average of all judge scores
        - confidence: Agreement level (1 - variance)
        - individual_scores: Each judge's score
        - consensus: Whether judges agreed
        - needs_review: Flag for human review
        """
        # Run all judges in parallel
        results = await asyncio.gather(*[
            judge.evaluate(input_text, output, expected)
            for judge in self.judges
        ])
        
        scores = [r.score for r in results]
        avg_score = sum(scores) / len(scores)
        variance = sum((s - avg_score) ** 2 for s in scores) / len(scores)
        
        # Agreement metrics
        max_diff = max(scores) - min(scores)
        consensus = max_diff < 0.2  # Judges agree within 20%
        
        return {
            "score": avg_score,
            "confidence": 1.0 - min(variance * 5, 0.5),  # Normalize to 0.5-1.0
            "individual_scores": [
                {"model": r.judge_model, "score": r.score}
                for r in results
            ],
            "consensus": consensus,
            "needs_review": not consensus and max_diff > 0.3,
            "total_cost": sum(r.cost for r in results),
            "reasonings": [r.reasoning for r in results],
        }


class CalibratedJudge:
    """
    A judge wrapper that tracks and corrects for judge drift.
    
    When you change the judge model (e.g., GPT-4o-mini → GPT-4o),
    scores might shift. This wrapper DETECTS the shift and
    CALIBRATES the new judge to produce comparable scores.
    
    How calibration works:
    1. Run BOTH judges on a calibration set (50-100 cases)
    2. Compute the calibration offset (average score difference)
    3. Apply the offset to new judge's scores until recalibrated
    """
    
    def __init__(
        self,
        primary_judge: LLMJudge,
        calibration_set: list[dict],  # test cases with expected scores
        calibration_threshold: float = 0.03,  # Max acceptable drift
    ):
        self.judge = primary_judge
        self.calibration_set = calibration_set
        self.calibration_threshold = calibration_threshold
        self.calibration_offset = 0.0
        self.last_calibration_date = None
    
    async def calibrate(self, reference_judge: LLMJudge) -> dict:
        """
        Calibrate this judge against a reference judge.
        
        Returns calibration results including offset and confidence.
        """
        scores_primary = []
        scores_reference = []
        
        for case in self.calibration_set:
            # Both judges evaluate the same cases
            p_result = await self.judge.evaluate(
                case["input"], case["output"], case.get("expected")
            )
            r_result = await reference_judge.evaluate(
                case["input"], case["output"], case.get("expected")
            )
            scores_primary.append(p_result.score)
            scores_reference.append(r_result.score)
        
        # Compute statistics
        avg_primary = sum(scores_primary) / len(scores_primary)
        avg_reference = sum(scores_reference) / len(scores_reference)
        self.calibration_offset = avg_reference - avg_primary
        
        # Drift detection
        abs_drift = abs(self.calibration_offset)
        drift_detected = abs_drift > self.calibration_threshold
        
        return {
            "calibration_offset": self.calibration_offset,
            "avg_primary": avg_primary,
            "avg_reference": avg_reference,
            "drift_detected": drift_detected,
            "sample_size": len(self.calibration_set),
        }
    
    async def evaluate(
        self,
        input_text: str,
        output: str,
        expected: Optional[str] = None,
    ) -> JudgeResult:
        """Evaluate with calibration offset applied."""
        result = await self.judge.evaluate(input_text, output, expected)
        
        # Apply calibration offset to correct for judge drift
        calibrated_score = max(0.0, min(1.0, result.score + self.calibration_offset))
        
        result.score = calibrated_score
        result.reasoning += (
            f"\n[Calibrated: raw={result.score:.3f}, "
            f"offset={self.calibration_offset:+.3f}]"
        )
        
        return result
```

### Step 2: Judge Bias Detection

Add a **Bias Detector** that analyzes judge behavior over time:

- **Position bias**: Does the judge prefer first/last answers?
- **Verbosity bias**: Does the judge prefer longer/shorter outputs?
- **Self-enhancement bias**: Does the judge prefer outputs from its own family of models?
- **Compassion fatigue**: Does the judge get stricter/tolerant over a long eval run?

```python
"""
judge/bias_detector.py

Detects systematic biases in LLM judges.

Analyzes historical judge results to find patterns.
If bias is detected, flags the judge for recalibration.
"""

import statistics
from typing import Optional


class BiasReport:
    """Report of detected biases in a judge."""
    has_bias: bool
    biases: list[dict]  # [{type, severity, evidence}]
    recommendation: str


class BiasDetector:
    """
    Detects biases in judge behavior by analyzing
    historical evaluation results.
    
    Uses statistical tests to compare score distributions
    across different dimensions (length, position, model).
    """
    
    def __init__(self, judge_name: str):
        self.judge_name = judge_name
        self.history: list[dict] = []  # Accumulated eval results
    
    def record_evaluation(self, result: dict):
        """Record a single evaluation for bias analysis."""
        self.history.append({
            "score": result.get("score", 0),
            "output_length": len(result.get("output", "")),
            "output_model": result.get("output_model", "unknown"),
            "position_in_run": result.get("position", 0),
            "total_in_run": result.get("total", 1),
            "timestamp": result.get("timestamp"),
        })
    
    def analyze(self, min_samples: int = 100) -> BiasReport:
        """
        Analyze historical data for biases.
        
        Requires at least min_samples to produce reliable results.
        """
        if len(self.history) < min_samples:
            return BiasReport(
                has_bias=False,
                biases=[],
                recommendation=f"Need {min_samples - len(self.history)} more samples",
            )
        
        detected_biases = []
        
        # 1. VERBOSITY BIAS: Do longer outputs get higher scores?
        lengths = [r["output_length"] for r in self.history]
        scores = [r["score"] for r in self.history]
        
        if lengths and scores:
            median_length = statistics.median(lengths)
            short_scores = [
                s for l, s in zip(lengths, scores)
                if l < median_length
            ]
            long_scores = [
                s for l, s in zip(lengths, scores)
                if l >= median_length
            ]
            
            if short_scores and long_scores:
                short_avg = sum(short_scores) / len(short_scores)
                long_avg = sum(long_scores) / len(long_scores)
                diff = long_avg - short_avg
                
                if diff > 0.05:  # 5%+ score difference
                    detected_biases.append({
                        "type": "verbosity_bias",
                        "severity": "high" if diff > 0.1 else "medium",
                        "evidence": (
                            f"Long outputs score {diff:.1%} higher on average. "
                            f"Short avg: {short_avg:.3f}, Long avg: {long_avg:.3f}"
                        ),
                    })
        
        # 2. POSITION BIAS: Do scores change based on position in run?
        # (Scores at the beginning vs end of a long eval run)
        
        # 3. SELF-ENHANCEMENT BIAS: Does the judge prefer its own model's outputs?
        
        # (Implementation for 2 and 3 follows the same pattern as 1)
        
        return BiasReport(
            has_bias=len(detected_biases) > 0,
            biases=detected_biases,
            recommendation=self._generate_recommendation(detected_biases),
        )
    
    def _generate_recommendation(self, biases: list) -> str:
        """Generate human-readable recommendation based on detected biases."""
        if not biases:
            return "No significant biases detected."
        
        if any(b["severity"] == "high" for b in biases):
            return (
                f"CRITICAL: {len(biases)} high-severity biases detected. "
                f"Recommend recalibrating or replacing this judge immediately."
            )
        
        return (
            f"{len(biases)} bias(es) detected at medium-low severity. "
            f"Consider rotating judge prompts or using multi-judge ensemble."
        )
```

---

## Milestone 3: Dashboard & Reporting

Build the backend endpoints for a dashboard that serves 3 stakeholders:

### Stakeholder Views

```
ENGINEER VIEW (drills into one system):
┌─────────────────────────────────────────────┐
│ RAG Bot — Latest Run: 2 min ago             │
│ Score: 94% ✅ (vs baseline: 93%)            │
│                                             │
│ Trend (30d):  ▁▂▃▅▇▆▇▆▇▇▆▅▆▇▇  → +1.1%    │
│                                             │
│ Category Breakdown:                         │
│ Critical       8/8  ✅ 100%                 │
│ Standard      42/46 ✅ 91%                  │
│ Informational 12/14 ✅ 86%                  │
│                                             │
│ Top Regressions (last 7 days):              │
│ 1. "multi-currency" case: 0.72 ↓0.15       │
│ 2. "holiday date" case: 0.81 ↓0.08         │
└─────────────────────────────────────────────┘

PM VIEW (across all systems):
┌─────────────────────────────────────────────┐
│ AI System Health Overview                    │
│                                             │
│ 🟢 RAG Bot        94%  (↑1.1% this week)   │
│ 🟡 Classifier     87%  (↓2.3% this week)   │
│ 🔴 Code Assist    76%  (↓5.1% this week)   │
│                                             │
│ This week: 2 improving, 1 stable, 1 declining
│ Action needed: Code Assist regression        │
└─────────────────────────────────────────────┘

EXECUTIVE VIEW (one number):
┌─────────────────────────────────────────────┐
│ AI Platform Health Score: 87/100   🟡       │
│ (Weighted avg of all systems + trends)      │
│                                             │
│ 3 systems: 2 healthy, 1 needs attention    │
│ Cost this month: $1,240 (within budget)    │
│ Evals run this week: 142                    │
│ Regressions detected: 3 (all resolved)      │
└─────────────────────────────────────────────┘
```

**Your task:** Implement the API endpoints that power each of these views. The endpoints should:
1. Query the database efficiently (no N+1 queries)
2. Return pre-computed metrics where possible
3. Support filtering by time range and system

```python
"""
api/dashboard.py

Dashboard API endpoints for different stakeholders.

🤔 PERFORMANCE CHALLENGE: The engineer view requires
computing 30-day trends, category breakdowns, AND
top regressions. That's 3-5 database queries per view.

For a single user loading the dashboard: fine.
For 50 users loading simultaneously: database hammering.

SOLUTION: Pre-compute and cache dashboard data.
- Cache engineer view for 30 seconds
- Cache PM view for 60 seconds
- Cache executive view for 5 minutes

(These are AI eval metrics, not stock prices —
millisecond accuracy is unnecessary.)
"""

from fastapi import APIRouter, Query
from datetime import datetime, timedelta

router = APIRouter(prefix="/api/v1/dashboard")


@router.get("/engineer/{system_name}")
async def engineer_view(
    system_name: str,
    days: int = Query(30, ge=1, le=90),
):
    """
    Engineer view: Deep drill-down into one system.
    
    Returns:
    - Latest run summary
    - Trend data
    - Category breakdown  
    - Recent regressions
    - Top failing test cases
    """
    # Implementation queries the database and returns structured data
    return {
        "system": system_name,
        "latest_run": {},
        "trend": [],
        "category_breakdown": [],
        "regressions": [],
        "top_failures": [],
    }


@router.get("/pm")
async def pm_view(
    team_id: str = Query("default"),
    days: int = Query(7, ge=1, le=30),
):
    """
    PM view: All systems, health overview.
    
    Returns a card for each system with:
    - Current score + trend direction
    - Week-over-week change
    - Alert status (needs attention? regression?)
    """
    pass


@router.get("/executive")
async def executive_view():
    """
    Executive view: Single health score for all AI systems.
    
    The "AI Platform Health Score" combines:
    - Average score across all systems (weighted by query volume)
    - Trend direction (improving/stable/declining)
    - Regression count (recent)
    - Cost vs. budget
    """
    pass


@router.get("/reports/weekly")
async def weekly_report(
    team_id: str = Query("default"),
    format: str = Query("json"),  # "json" or "markdown"
):
    """
    Generate a weekly report.
    
    If format=markdown, returns markdown suitable for Slack.
    If format=json, returns structured data for custom rendering.
    """
    pass
```

---

## Milestone 4: Observability & Regression Detection

### Step 1: Langfuse Integration

Integrate your eval platform with Langfuse (or OpenTelemetry) for production tracing. Every eval run should produce traces that show:
- The full eval run as a trace
- Each test case evaluation as a span
- Judge calls as sub-spans (with model, cost, latency)
- Gate decisions as span attributes

```python
"""
integration/observability.py

Langfuse integration for eval platform tracing.

The pattern:
1. Start a trace when an eval run starts
2. Create a span for each test case evaluation
3. Record judge calls as sub-spans
4. Record cost, latency, and scores as span attributes
5. End the trace when the run completes

🤔 COST OF TRACING: Each span costs $0.00005 in Langfuse.
500 test cases × 3 spans per case = 1,500 spans = $0.075/run.
That's 7.5 cents per run. For 50 runs/day: $3.75/day.
This is CHEAP compared to the $6/run in judge costs.

QUESTION: When is tracing NOT worth the cost?
Answer: When you're running 1000+ evals/day and the tracing
cost exceeds the debugging value. At that point, use SAMPLING.
"""

from langfuse import Langfuse
from datetime import datetime
from typing import Optional


class EvalTracer:
    """
    Traces eval runs in Langfuse for observability.
    
    Every eval run → one trace
    Every test case evaluation → one span (within trace)
    Every judge call → one sub-span (within test case span)
    """
    
    def __init__(self, langfuse: Langfuse, sampling_rate: float = 1.0):
        self.langfuse = langfuse
        self.sampling_rate = sampling_rate  # 1.0 = trace all, 0.1 = trace 10%
    
    def start_run_trace(self, run_id: str, config: dict):
        """Start a trace for an eval run."""
        trace = self.langfuse.trace(
            name=f"eval_run_{config['system_name']}",
            id=run_id,
            input=config.get("input_preview"),
            metadata={
                "system_name": config.get("system_name"),
                "system_version": config.get("system_version"),
                "triggered_by": config.get("triggered_by"),
                "branch": config.get("branch"),
                "pr_number": config.get("pr_number"),
            },
        )
        return trace
    
    def create_eval_span(self, trace, case_id: str, input_text: str):
        """Create a span for evaluating a single test case."""
        return trace.span(
            name=f"evaluate_{case_id[:8]}",
            input={"test_case_id": case_id, "input": input_text[:500]},
        )
    
    def record_judge_call(
        self,
        parent_span,
        judge_model: str,
        score: float,
        cost: float,
        latency_ms: float,
        reasoning: Optional[str] = None,
    ):
        """Record an LLM judge call as a span."""
        generation = parent_span.generation(
            name=f"judge_{judge_model}",
            model=judge_model,
            usage={"output": 1},  # Simplified
            model_parameters={"temperature": 0.0},
            output=reasoning or "",
        )
        
        # In production, set actual token counts
        # generation.end(usage={"output": actual_tokens})
    
    def end_run_trace(self, trace, result: dict):
        """End a trace with the final result."""
        trace.update(
            output={
                "status": result.get("status"),
                "avg_score": result.get("avg_score"),
                "passed": result.get("passed"),
                "failed": result.get("failed"),
                "total_cost": result.get("total_cost"),
            },
        )
```

### Step 2: Regression Detection Integration

Integrate the CUSUM and moving-average detectors from File 07 into your platform:

```python
"""
core/regression_detector.py

Integrated regression detection that runs after every eval run.

Checks:
1. CUSUM: Sudden shifts from baseline
2. Moving Average: Gradual trend changes
3. Z-Score: Anomalous single-run drops

On detection:
1. Log the regression (database)
2. Evaluate alert rules (File 08)
3. If alert triggers → notify (Slack/email)
4. If auto-rollback enabled → trigger rollback
"""

import statistics
from datetime import datetime, timedelta
from typing import Optional


class RegressionDetector:
    """
    Detects regressions in eval metrics using multiple methods.
    
    Uses CUSUM for sudden shifts and moving average for gradual trends.
    Requires configurable sensitivity and false-positive management.
    """
    
    def __init__(
        self,
        db_session,
        cusum_k: float = 0.5,
        cusum_h: float = 4.0,
        trend_window: int = 7,     # days for trend detection
        min_samples: int = 10,     # minimum data points
    ):
        self.db = db_session
        self.cusum_k = cusum_k
        self.cusum_h = cusum_h
        self.trend_window = trend_window
        self.min_samples = min_samples
    
    async def analyze_run(self, run: dict) -> dict:
        """
        Analyze a completed run for regressions.
        
        Returns:
        - regression_detected: bool
        - methods: Results from each detection method
        - severity: "none" | "warning" | "critical"
        - details: Human-readable explanation
        """
        system = run["system_name"]
        current_score = run["avg_score"]
        
        # Get historical scores
        history = await self._get_history(system)
        
        if len(history) < self.min_samples:
            return {"regression_detected": False, "severity": "none"}
        
        results = {}
        
        # Method 1: CUSUM (sudden shifts)
        results["cusum"] = self._check_cusum(current_score, history)
        
        # Method 2: Moving average (gradual trends)
        results["moving_avg"] = self._check_moving_avg(history)
        
        # Method 3: Z-score (anomalous drops)
        results["zscore"] = self._check_zscore(current_score, history)
        
        # Aggregate
        any_detected = any(r.get("detected") for r in results.values())
        severities = [r.get("severity", "none") for r in results.values()]
        
        if "critical" in severities:
            overall_severity = "critical"
        elif "warning" in severities:
            overall_severity = "warning"
        else:
            overall_severity = "none" if not any_detected else "info"
        
        return {
            "regression_detected": any_detected,
            "severity": overall_severity,
            "methods": results,
            "details": self._build_summary(results, system, current_score),
        }
    
    def _check_cusum(
        self,
        current: float,
        history: list[float],
    ) -> dict:
        """
        CUSUM shift detection.
        
        CUSUM accumulates deviations from the target mean.
        When the cumulative sum exceeds threshold h,
        a shift is detected.
        """
        target = sum(history) / len(history)
        std = statistics.stdev(history) if len(history) > 1 else 0.05
        
        # CUSUM parameters
        k = self.cusum_k * std  # Allowable slack
        h = self.cusum_h * std  # Decision threshold
        
        # Run CUSUM on historical data to find if there's already a shift
        cusum_pos = 0
        cusum_neg = 0
        max_cusum = 0
        
        for value in history + [current]:
            cusum_pos = max(0, cusum_pos + (value - target) + k)
            cusum_neg = max(0, cusum_neg - (value - target) + k)
            max_cusum = max(max_cusum, cusum_pos, cusum_neg)
        
        # Add current point
        shift_detected = max_cusum > h
        severity = "critical" if max_cusum > 2 * h else "warning" if shift_detected else "none"
        
        return {
            "detected": shift_detected,
            "severity": severity,
            "cusum_value": max_cusum,
            "threshold": h,
            "target_mean": target,
        }
    
    def _check_moving_avg(self, history: list[float]) -> dict:
        """Moving average trend detection."""
        if len(history) < self.trend_window:
            return {"detected": False, "severity": "none"}
        
        recent = history[-self.trend_window:]
        older = history[:self.trend_window]
        
        recent_avg = sum(recent) / len(recent)
        older_avg = sum(older) / len(older)
        
        drop = older_avg - recent_avg
        
        if drop > 0.1:  # 10%+ drop over trend window
            return {
                "detected": True,
                "severity": "critical",
                "recent_avg": recent_avg,
                "older_avg": older_avg,
                "drop": drop,
            }
        elif drop > 0.05:
            return {
                "detected": True,
                "severity": "warning",
                "recent_avg": recent_avg,
                "older_avg": older_avg,
                "drop": drop,
            }
        
        return {"detected": False, "severity": "none", "drop": drop}
    
    def _check_zscore(self, current: float, history: list[float]) -> dict:
        """Z-score anomaly detection."""
        mean = sum(history) / len(history)
        std = statistics.stdev(history) if len(history) > 1 else 0.05
        
        if std == 0:
            return {"detected": False, "severity": "none"}
        
        zscore = (mean - current) / std  # Positive = below mean
        
        if zscore > 5:
            return {"detected": True, "severity": "critical", "zscore": zscore}
        elif zscore > 3:
            return {"detected": True, "severity": "warning", "zscore": zscore}
        
        return {"detected": False, "severity": "none", "zscore": zscore}
    
    async def _get_history(self, system_name: str) -> list[float]:
        """Get historical scores for a system."""
        # In production, query the database
        # In development, return sample data
        return [0.92, 0.93, 0.91, 0.94, 0.92, 0.93, 0.91, 0.90]
    
    def _build_summary(
        self,
        results: dict,
        system: str,
        current_score: float,
    ) -> str:
        """Build human-readable summary of regression analysis."""
        parts = [f"Regression analysis for {system} (current: {current_score:.3f}):"]
        
        for method, result in results.items():
            if result.get("detected"):
                parts.append(
                    f"  ⚠️ {method}: {result.get('severity', 'alert').upper()} "
                    f"(z-score: {result.get('zscore', 'N/A')})"
                )
            else:
                parts.append(f"  ✅ {method}: No regression detected")
        
        return "\n".join(parts)
```

### Step 3: Production Feedback Loop

```python
"""
integration/feedback_loop.py

The production-to-eval feedback loop (File 06, File 07).

Collects production traces that had issues (user negative feedback,
unusual patterns, low confidence) and adds them to the eval dataset.

This is how your eval dataset STAYS RELEVANT — by continuously
incorporating real production failures.
"""

class ProductionFeedbackLoop:
    """
    Closes the gap between eval and production.
    
    Pipeline:
    1. Production system runs → user provides feedback (implicit/explicit)
    2. Low-scoring or negatively-received responses are flagged
    3. Flagged cases are sampled and added to eval dataset
    4. Next eval run includes these new production-sourced cases
    5. If the system scores LOW on production-sourced cases → ALERT
    
    🤔 KEY DESIGN QUESTION: How do you prevent the eval dataset
    from growing unboundedly? Each new production case added
    increases eval time and cost.
    
    SOLUTION: Dataset rotation.
    - Keep a FIXED-SIZE "golden dataset" (500 cases, manually curated)
    - Keep a GROWING "production dataset" (all production-sourced cases)
    - On each eval run, use: GOLDEN (always) + PRODUCTION_SAMPLE (random 100)
    - This ensures coverage of known issues without unbounded growth.
    """
    
    def __init__(self, db_session, max_cases_per_run: int = 100):
        self.db = db_session
        self.max_cases_per_run = max_cases_per_run
    
    async def collect_production_cases(
        self,
        system_name: str,
        since_hours: int = 24,
    ) -> list[dict]:
        """
        Collect production cases that should be added to eval dataset.
        
        Sources:
        1. User negative feedback (thumbs down, complaints)
        2. Low-confidence responses (model confidence < threshold)
        3. Escalated queries (user asked for human)
        4. Anomalous patterns (unusually long/short responses)
        """
        # In production, query observability platform
        return []
    
    async def add_to_dataset(
        self,
        system_name: str,
        cases: list[dict],
        max_batch: int = 50,
    ):
        """
        Add production cases to the eval dataset.
        
        Each case includes:
        - input (the production query)
        - expected_output (manually verified or LLM-generated)
        - source: "production_feedback"
        - metadata: {original_id, timestamp, user_satisfaction}
        """
        pass
    
    async def balance_dataset(self, system_name: str):
        """
        Balance the eval dataset to prevent distribution drift.
        
        If production cases are over-represented in one category
        (e.g., 60% of new cases are "billing"), the eval becomes
        biased toward that category.
        
        Use STRATIFIED SAMPLING to maintain the original category distribution.
        """
        pass
```

---

## Milestone 5: CI/CD Integration & Production Polish

### Step 1: CLI Tool

```python
"""
cli/eval_cli.py

Command-line tool for running evals and checking gates.

Usage:
  eval-cli run --system rag-bot --config eval-config.yaml
  eval-cli status --system rag-bot
  eval-cli trends --system rag-bot --days 30
  eval-cli gate --run-id abc-123
  
Exit codes:
  0 — Gate passed (safe to deploy)
  1 — Gate blocked (regression detected)
  2 — Error (invalid config, API unavailable)
"""

import argparse
import sys
import yaml
from typing import Optional


class EvalCLI:
    """Command-line interface for the eval platform."""
    
    def __init__(self, platform_client):
        self.client = platform_client
    
    def run_command(self, args):
        """Run an eval and return gate decision."""
        # Load config
        with open(args.config) as f:
            config = yaml.safe_load(f)
        
        # Execute run via platform API
        result = self.client.create_run(
            system_name=args.system,
            system_version=config.get("version", "latest"),
            model_config=config.get("model", {}),
            prompt_config=config.get("prompt", {}),
            triggered_by="ci/cd",
            branch=args.branch,
            pr_number=args.pr,
        )
        
        # Check gate decision
        if result["gate"]["passed"]:
            print(f"✅ Eval passed: {result['avg_score']:.1%}")
            return 0
        else:
            print(f"❌ Eval blocked: {result['gate']['reasons']}")
            return 1
    
    def status_command(self, args):
        """Show current eval status for a system."""
        status = self.client.get_latest_run(args.system)
        print(f"System: {args.system}")
        print(f"Latest run: {status.get('status', 'N/A')}")
        print(f"Score: {status.get('avg_score', 'N/A')}")
        print(f"Trend: {status.get('trend', 'N/A')}")
        return 0
    
    def trends_command(self, args):
        """Show trend data for a system."""
        trends = self.client.get_trends(args.system, days=args.days)
        for point in trends.get("data_points", []):
            print(f"{point['date']}: {point['value']:.3f}")
        return 0


def main():
    parser = argparse.ArgumentParser(description="AI Eval Platform CLI")
    parser.add_argument("--api-url", default="http://localhost:8000")
    parser.add_argument("--api-key", default=None)
    
    subparsers = parser.add_subparsers(dest="command")
    
    # run
    run_parser = subparsers.add_parser("run", help="Run evaluation")
    run_parser.add_argument("--system", required=True)
    run_parser.add_argument("--config", required=True)
    run_parser.add_argument("--branch", default=None)
    run_parser.add_argument("--pr", default=None)
    
    # status
    status_parser = subparsers.add_parser("status", help="Check system status")
    status_parser.add_argument("--system", required=True)
    
    # trends
    trends_parser = subparsers.add_parser("trends", help="Show trend data")
    trends_parser.add_argument("--system", required=True)
    trends_parser.add_argument("--days", type=int, default=30)
    
    args = parser.parse_args()
    
    # Initialize client and run command
    cli = EvalCLI(platform_client=None)  # In production, pass real client
    sys.exit(cli.run_command(args))


if __name__ == "__main__":
    main()
```

### Step 2: GitHub Actions Integration

Create a GitHub Action that teams can use in their workflows:

```yaml
# .github/actions/eval-gate/action.yml
name: 'AI Eval Gate'
description: 'Run evaluation and block PR if quality regressed'
inputs:
  system:
    description: 'System to evaluate'
    required: true
  config:
    description: 'Path to eval config'
    required: true
    default: '.eval/config.yaml'
  api-url:
    description: 'Eval platform API URL'
    required: true
  api-key:
    description: 'API key for eval platform'
    required: true
outputs:
  passed:
    description: 'Whether the gate passed'
    value: ${{ steps.eval.outputs.passed }}
  score:
    description: 'Average score'
    value: ${{ steps.eval.outputs.score }}
runs:
  using: 'composite'
  steps:
    - name: Run Evaluation
      id: eval
      run: |
        eval-cli run \
          --system ${{ inputs.system }} \
          --config ${{ inputs.config }} \
          --api-url ${{ inputs.api-url }} \
          --api-key ${{ inputs.api-key }}
        echo "passed=$([ $? -eq 0 ] && echo 'true' || echo 'false')" >> $GITHUB_OUTPUT
      shell: bash
    
    - name: Comment on PR
      if: failure()
      uses: actions/github-script@v7
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: '🚨 **AI Eval Failed**\n\n' +
                  'The evaluation detected a regression. ' +
                  'Please check the [eval dashboard](${{ inputs.api-url }}) for details.'
          })
```

### Step 3: Docker Compose Deployment

```yaml
# docker/docker-compose.yml

version: '3.8'

services:
  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=sqlite:///data/eval.db
      - LANGFS_PUBLIC_KEY=${LANGFS_PUBLIC_KEY}
      - LANGFS_SECRET_KEY=${LANGFS_SECRET_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - ./data:/app/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  dashboard:
    build:
      context: ..
      dockerfile: docker/Dockerfile.dashboard
    ports:
      - "3000:3000"
    environment:
      - API_URL=http://api:8000
    depends_on:
      - api
  
  # Optional: PostgreSQL for production
  # db:
  #   image: postgres:16
  #   environment:
  #     POSTGRES_DB: eval_platform
  #     POSTGRES_PASSWORD: ${DB_PASSWORD}
  #   volumes:
  #     - pgdata:/var/lib/postgresql/data
  #   ports:
  #     - "5432:5432"

# volumes:
#   pgdata:
```

**🤔 Your task:** Build the actual Dockerfiles and add:
- A `docker-compose.override.yml` for development (hot reload)
- Environment variable validation on startup
- A database migration script (Alembic or raw SQL) that runs on first startup
- A health check endpoint that verifies database connectivity and LLM API access

---

## Milestone 6: Deployment & Portfolio Prep

### Step 1: Deploy to Production

Deploy your platform:

```bash
# Build and start
docker compose up --build -d

# Verify
curl http://localhost:8000/health
curl http://localhost:8000/api/v1/systems

# Run a test eval
eval-cli run --system test-system --config .eval/test-config.yaml
```

**Deployment checklist:**
- [ ] API is running and responding
- [ ] Database is initialized with schema
- [ ] Eval runs complete without errors
- [ ] Dashboard loads and shows correct data
- [ ] CLI tool can create runs and check status
- [ ] Slack notifications work (if configured)
- [ ] Health check endpoint returns 200
- [ ] Logs are being collected

### Step 2: Portfolio README

Create a README.md that explains your project:

```markdown
# AI Evaluation Platform

A production-grade evaluation platform for measuring, monitoring, and maintaining
the quality of AI systems — from RAG chatbots to classifiers to agents.

## Architecture

[Include architecture diagram from above]

## Features

- **Multi-System Evaluation**: Evaluate any AI system with system-specific metrics
- **LLM-as-Judge Pipeline**: Multi-judge ensemble with bias detection and calibration
- **Smart Execution**: Tiered evaluation (critical → standard → info) with caching
- **Regression Detection**: CUSUM + moving average + z-score detection methods
- **CI/CD Integration**: CLI tool + GitHub Action to block regressions before deploy
- **Multi-Stakeholder Dashboard**: Engineer, PM, and executive views
- **Production Feedback Loop**: Continuous dataset improvement from production data
- **Observability**: Langfuse tracing for every eval run with cost tracking

## Quick Start

```bash
git clone <your-repo>
cd ai-eval-platform
docker compose up
```

## Tech Stack

- **Backend**: FastAPI, SQLAlchemy, SQLite/PostgreSQL
- **AI**: OpenAI, Anthropic APIs for LLM judges
- **Observability**: Langfuse, OpenTelemetry
- **Deployment**: Docker Compose

## Project Structure

```
ai-eval-platform/
├── api/              # FastAPI endpoints
├── core/             # Core data models, run engine
├── judge/            # LLM judge pipeline, bias detection
├── datasets/         # Test case management
├── integration/      # Langfuse, Slack, CI/CD
├── cli/              # Command-line tool
├── dashboard/        # Frontend (React/Jinja2)
├── docker/           # Docker configs
└── tests/            # Test suite
```

## Design Decisions

[Explain 3-4 key architectural decisions and why you made them]

## Lessons Learned

[What surprised you? What broke? What would you do differently?]
```

### Step 3: Demo Script

Create a demo script that walks through the complete flow:

```
1. Start the platform: docker compose up
2. Create test cases: eval-cli create-cases --system rag-bot
3. Run evaluation: eval-cli run --system rag-bot --config eval-config.yaml
4. Check dashboard: open http://localhost:3000
5. Introduce a bug (change system prompt)
6. Run evaluation again — regression detected!
7. Check the gate blocks the deployment
8. Show the Slack alert
9. Show the weekly report
```

### Step 4: Evaluation of Your Platform

You've built an eval platform. Now EVALUATE IT.

```python
"""
tests/test_platform.py

Self-evaluation: does your platform catch known regressions?
"""

import pytest


class TestPlatformSelfEvaluation:
    """
    Your eval platform must pass its OWN tests.
    
    Test 1: Known regression detection
    - Create a "good" system version
    - Run eval → score is high
    - Create a "bad" system version (introduce a known bug)
    - Run eval → score drops
    - Check: did the gate block the bad version?
    
    Test 2: Noise rejection
    - Run the same eval 10 times
    - Check: is score variance < 0.02?
    - (If variance is high, your judge is flaky)
    
    Test 3: Coordination
    - Run eval on System A and System B simultaneously
    - Check: results don't interfere
    """
    
    def test_detects_known_regression(self):
        """If system output quality drops by 20%, the gate should block."""
        assert False  # Implement this
    
    def test_low_variance_on_repeated_runs(self):
        """Same system + same dataset = same results (variance < 2%)."""
        assert False  # Implement this
    
    def test_multi_system_isolation(self):
        """Two simultaneous eval runs don't affect each other."""
        assert False  # Implement this
    
    def test_judge_calibration(self):
        """Two different judges produce correlated scores (>0.8 correlation)."""
        assert False  # Implement this
    
    def test_regression_detection_accuracy(self):
        """CUSUM detects synthetic 5% drops within 3 runs."""
        assert False  # Implement this
```

---

## ✅ Success Criteria

Your project is complete when:

1. **Core functionality**: You can create an eval run, execute test cases, compute metrics, and check the gate — all via API.
2. **Multi-system support**: The platform evaluates at least 2 different system types with different metrics.
3. **LLM judge pipeline**: Multi-judge ensemble works with bias detection and calibration.
4. **Regression detection**: CUSUM and moving average methods detect regressions from historical data.
5. **CI/CD integration**: The CLI tool exits with 0/1 based on gate decision, usable in any CI pipeline.
6. **Observability**: Every eval run is traced in Langfuse with cost and latency data.
7. **Dashboard**: At least one dashboard view is functional (engineer, PM, or executive).
8. **Docker deployment**: The entire stack runs via `docker compose up`.
9. **Self-evaluation**: The platform can detect its own test regressions (the platform tests pass).
10. **README**: Project is documented with architecture, setup instructions, and design decisions.

### 🏆 Stretch Goals

- **Slack bot**: `/eval status` command in Slack shows current system health
- **Automated rollback**: Platform triggers rollback if regression is critical
- **A/B eval**: Compare two system versions side by side (same dataset, different configs)
- **Multi-model judge**: Compare GPT-4o, Claude 3.5, and Gemini as judges — which is best for YOUR use case?
- **Flaky test auto-removal**: Platform automatically removes test cases with >30% variance
- **Personalized dashboard**: Users configure their own dashboard layout

---

## 📚 Resources

- **Phase 7 Files 01-08** — Every single concept from this phase is integrated into this project. If you're stuck, re-read the relevant file.
- **"Building Secure and Reliable Systems"** (Google, 2020) — Chapter 12 on "Designing for Reliability" directly applies to eval platform design (circuit breakers, graceful degradation, retry strategies).
- **"FastAPI in Production"** (tiangolo, 2024) — Best practices for deploying FastAPI apps including async patterns, dependency injection, and background tasks.
- **"Langfuse Documentation"** (Langfuse, 2025) — The tracing SDK documentation. Covers spans, generations, and cost tracking.
- **"SQLAlchemy Performance Patterns"** (SQLAlchemy, 2024) — N+1 query prevention, eager loading strategies, and connection pooling for high-throughput APIs.
- **"GitHub Actions: Creating Actions"** (GitHub, 2024) — How to package your CLI tool as a reusable GitHub Action with inputs, outputs, and post-run cleanup.
- **Cross-Phase: Phase 5 File 09 "Advanced RAG Capstone"** — If you built the Advanced RAG Engine in Phase 5, integrate it as one of the "systems under test" for your eval platform. The RAGAS metrics from Phase 5 can feed into your platform's metric pipeline.
- **Cross-Phase: Phase 6 File 09 "Multi-Agent Research System"** — If you built the multi-agent system, its evaluation needs (trajectory quality, decision efficiency, tool usage) create excellent test cases for your eval platform.

---

**🎉 Congratulations! You've completed Phase 7 — Production Evals & Observability.**

You now have a complete, deployable evaluation platform that can measure, monitor, and maintain the quality of ANY AI system — RAG chatbots, classifiers, agents, generators, whatever comes next.

When you deploy your Phase 8 (Guardrails & Safety) or Phase 9 (Fine-Tuning) systems, your eval platform will be there to measure them from Day 1. That's the eval-first mindset in action.
