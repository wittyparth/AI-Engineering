# 08 — Custom Eval Framework: Building Your Evaluation Platform

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about architecture, you need to understand what happens when NO eval platform exists.

You're the first AI engineer at a company that's building its second AI system. The first system was a simple chatbot — you evaluated it with a Jupyter notebook, manually, whenever someone remembered to run it. It was fine.

Now you have:
- 3 AI systems (chatbot, content classifier, code assistant)
- 4 engineers making changes
- 2 product managers wanting quality dashboards
- 1 CTO asking "how do we know these systems actually work?"
- 0 evaluation infrastructure

Your Jupyter notebook approach doesn't scale. Each engineer runs evals locally. Results are on different machines. Nobody knows what the "current" quality score is. A change that passes one engineer's evals breaks another's. The PM wants a dashboard. The CTO wants to block bad deploys.

**You need an evaluation platform. But should you build it or buy it?**

---

### 🤔 Discovery Question 1: The Build vs. Buy Trap

Before reading further, think about this scenario:

Your team evaluates all three AI systems with Langfuse evaluations and a RAGAS notebook. It works OK. But the PM wants a unified "AI Health Dashboard" and the CTO wants CI/CD eval gates. A vendor shows up promising exactly this for $2,000/month.

**🤔 Before reading on, answer these:**

1. What SPECIFIC needs would a commercial eval platform NOT address for your team? Think about: your custom metrics, your specific judge prompts, your integration with existing infra, your data privacy requirements, your team's workflow.

2. What's the REAL cost of building vs. buying? Normally build looks expensive ($150k engineer salary) and buy looks cheap ($24k/year). But the build solution does EXACTLY what you need, and the buy solution requires you to adapt YOUR workflow to THEIR product. How do you quantify the "adaptation tax" of a commercial platform?

3. **The hidden risk:** You build a custom platform. It takes 3 months of one engineer's time. During those 3 months, your AI systems ship with ZERO evaluation infrastructure. How many regressions slip through? How much tech debt accumulates? How do you value the OPPORTUNITY COST of building vs. the RISK COST of not having evals during the build period?

---

### 🤔 Discovery Question 2: The Dashboard Paradox

You build the eval platform. The dashboard shows:
- 47 metrics across 3 systems
- Multi-colored charts, trend lines, sparklines
- Auto-refreshing every 5 minutes
- Email alerts for every regression

After 2 weeks, the PM stops checking it. After 4 weeks, the engineers stop responding to alerts. After 8 weeks, someone asks "is that dashboard still running?"

**🤔 Before reading on, answer these:**

1. Why do feature-rich dashboards get abandoned? List at least 3 reasons that aren't "the data was wrong" or "it was too slow."

2. **The signal-to-noise ratio:** Your eval platform detects every regression. If 80% of regressions are false alarms (noise from metric variance, flaky judges, test set sampling), and only 20% are real, what happens to team behavior? How does the platform design affect this ratio?

3. **The "green dashboard" problem:** When every metric is green, nobody looks at the dashboard. But the dashboard's real value is in detecting when things ARE green today but trending red. How do you design a dashboard that people check DAILY even when everything looks fine? What psychological principle makes people maintain habits?

---

### 🤔 Discovery Question 3: The Multi-Team Coordination Problem

Your platform launches. Team A (chatbot) uses it. Team B (classifier) doesn't — they already have their own eval scripts and don't want to migrate. Team C (code assistant) wants to use it but needs different metrics that you didn't build.

Now you have:
- Your platform (only Team A uses it)
- Team B's custom scripts (nobody else can run)
- Team C's requirements (would take 2 more weeks to build)

The CTO asks: "I thought we were standardizing on one platform?"

**🤔 Before reading on, answer these:**

1. What architectural decisions would make your platform ADOPTED by other teams rather than MANDATED? (Hint: mandated tools are resented; adopted tools are loved. What makes a tool easy to adopt?)

2. Should your platform REQUIRE all teams to use the same metrics? Or should it ALLOW team-specific metrics while providing COMMON infrastructure (storage, dashboards, alerts)? What are the tradeoffs of each approach?

3. **The migration barrier:** Team B's existing eval script works, has 200 test cases, and is integrated into their CI pipeline. Asking them to migrate means: rewriting 200 test cases, retraining the team, and risking breaking their CI. How do you design your platform so migration is GRADUAL — Team B can keep their existing scripts while sending results to the platform? What interface would allow this?

---

By the end of this module, you will:

1. **Design the database schema** for an eval platform — test cases, runs, results, metrics
2. **Build the API layer** — submitting results, querying history, triggering runs
3. **Implement the dashboard** — real-time views, trend analysis, drill-down
4. **Design multi-team support** — namespacing, permissions, shared datasets
5. **Integrate with existing tools** — CI/CD, Slack, GitHub, PagerDuty
6. **Build scheduled reporting** — weekly summaries, regression reports, trend alerts
7. **Design for adoption** — making it easy to start, hard to leave

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 20 min |
| Platform Architecture Overview | 20 min |
| Database Schema Design | 30 min |
| Code: Eval Run Model & Storage Layer | 35 min |
| Code: API Layer (FastAPI) | 35 min |
| Code: Dashboard Backend | 30 min |
| Code: Slack/CI Integration | 25 min |
| Multi-Team & Namespacing | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. Platform Architecture Overview

An eval platform has 4 layers:

```
┌─────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                     │
│  Dashboard (trends, drill-down, compare)                 │
│  Reports (scheduled, on-demand)                          │
│  Alerts (Slack, email, PagerDuty)                        │
├─────────────────────────────────────────────────────────┤
│                      API LAYER                            │
│  Submit results  │  Query history  │  Trigger runs       │
│  Manage datasets │  Manage metrics │  Configure alerts   │
├─────────────────────────────────────────────────────────┤
│                     STORAGE LAYER                         │
│  Eval Runs       │  Test Cases     │  Results            │
│  Metrics         │  Baselines      │  Configurations     │
├─────────────────────────────────────────────────────────┤
│                   INTEGRATION LAYER                       │
│  CI/CD plugins  │  SDK clients    │  Webhook receivers   │
│  CLI tools      │  Import/export  │  Auth providers      │
└─────────────────────────────────────────────────────────┘
```

**🤔 Before reading the schema, answer this:** What's the SINGLE most important query this platform needs to answer? Not "what are all the metrics" — that's 47 queries. What's the ONE query that someone runs every single day?

<details>
<summary>My answer (after you've thought about it):</summary>

*"Show me the quality trend of system X over the last N eval runs."*

Not "show me all the data." Not "run an eval." Not "show me current scores." 

**The trend.** Every single daily action starts with checking whether things are getting better, worse, or staying the same. Your schema must make this query FAST — not just possible, but the DEFAULT view.
</details>

---

### 2. Database Schema Design

The core entities and their relationships:

```
EVAL_RUN                    TEST_CASE                    RESULT
┌──────────────┐           ┌──────────────┐           ┌──────────────┐
│ id           │──1:N────▶│ id           │──1:N────▶│ id           │
│ system_name  │           │ input        │           │ eval_run_id  │
│ system_version│          │ expected     │           │ test_case_id │
│ model_config  │           │ category     │           │ passed       │
│ commit_hash   │           │ difficulty   │           │ score        │
│ triggered_by  │           │ metadata     │           │ latency_ms   │
│ started_at    │           │ created_at   │           │ output       │
│ completed_at  │           │ tags[]       │           │ error        │
│ status        │           └──────────────┘           │ metadata     │
│ metadata      │                                      │ judged_at    │
└──────────────┘                                      └──────────────┘
                                                              │
                                                              │ 1:N
                                                              ▼
                                                        METRIC_SNAPSHOT
                                                        ┌──────────────┐
                                                        │ id           │
                                                        │ eval_run_id  │
                                                        │ metric_name  │
                                                        │ metric_value │
                                                        │ p50/p95/p99  │
                                                        │ sample_size  │
                                                        │ computed_at  │
                                                        └──────────────┘
```

**Key design decisions to consider:**

| Decision | Option A | Option B | Why It Matters |
|----------|----------|----------|----------------|
| Result storage | JSON column | Normalized rows | JSON is flexible (any metric), normalised is queryable |
| Metric values | Pre-computed | Computed on read | Pre-computed is faster for dashboards, computed is more flexible |
| Run model | "Eval run" as unit | Individual results | Run model = you know what changed together |
| Baseline storage | Latest run as baseline | Versioned baselines | Versioned = you can compare "this run vs last week's" |
| Time series | Relational DB | TimescaleDB/InfluxDB | Relational works for 100K runs, timeseries for 10M+ |

**🤔 Your design choice:** For a team with 3-5 AI systems, ~50 eval runs per day, ~1M results per month — which choices above make sense? Which choices change when you have 50 systems and 50M results per month?

---

### 3. The Eval Run Model

An "eval run" is THE fundamental unit in the platform. Everything is organized around runs.

```python
"""
eval_platform/models.py

The core data model for the eval platform.

🤔 DESIGN TRADEOFF: Should an eval run be IMMUTABLE (append-only)
or MUTABLE (results can be updated/retracted)?

Immutable: Once results are submitted, they're permanent.
  - PROS: Audit trail, reproducible, no "why did this score change?"
  - CONS: Can't fix wrong results, can't retract bad data

Mutable: Results can be updated or deleted.
  - PROS: Can fix errors, can retract data from bad runs
  - CONS: Audit trail is complex, scores "change" after the fact

This code uses IMMUTABLE with a "retraction" mechanism.
"""

from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
from enum import Enum


class RunStatus(Enum):
    PENDING = "pending"        # Created but not started
    RUNNING = "running"        # Eval is in progress
    COMPLETED = "completed"    # All test cases evaluated
    FAILED = "failed"          # Pipeline crashed
    PARTIAL = "partial"        # Some test cases failed (gate blocked)
    RETRACTED = "retracted"    # Run was invalidated (not deleted — history preserved)


@dataclass
class EvalRun:
    """
    A single evaluation run.
    
    Immutable after completion — except for RETRACTED status.
    Retraction is a LOGICAL delete (preserves the data but marks it invalid).
    """
    id: str
    system_name: str
    system_version: str
    
    # What was being evaluated
    model_config: dict  # model name, temperature, top_k, etc.
    prompt_config: dict  # system prompt, retrieval config, etc.
    commit_hash: str
    
    # Who/what triggered it
    triggered_by: str  # "ci/cd", "manual", "scheduled", "webhook"
    trigger_metadata: dict = field(default_factory=dict)
    
    # Timing
    started_at: datetime
    completed_at: Optional[datetime] = None
    status: RunStatus = RunStatus.PENDING
    
    # Run-level metrics (pre-computed for dashboard speed)
    total_cases: int = 0
    passed: int = 0
    failed: int = 0
    avg_score: float = 0.0
    
    # Git context
    branch: str = ""
    pr_number: Optional[str] = None
    author: str = ""
    
    # Metadata (extensible)
    metadata: dict = field(default_factory=dict)


@dataclass
class TestCase:
    """
    A single test case in the evaluation dataset.
    
    Test cases are REFERENCED by eval runs, not copied into them.
    This means changes to a test case affect ALL historical runs
    that reference it — but we COPY the score at eval time.
    
    🤔 Why not copy the test case into the run?
      - Storage efficiency (test case is large, stored once)
      - Dataset management (fix a bug in a test case? It's fixed everywhere)
      - Historical runs still have the original SCORE (from results table)
    """
    id: str
    system_name: str  # Test cases are per-system
    input: str
    expected_output: Optional[str] = None
    category: str = "standard"  # "critical" | "standard" | "informational"
    difficulty: str = "medium"  # "easy" | "medium" | "hard"
    tags: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
    created_at: datetime = field(default_factory=datetime.now)
    deprecated: bool = False  # Soft-delete for test cases


@dataclass
class EvalResult:
    """
    Result of evaluating ONE test case in ONE run.
    
    This is the CORE data point. Every other data structure
    (metrics, snapshots, trends, reports) is derived from these.
    """
    id: str
    eval_run_id: str
    test_case_id: str
    passed: bool
    score: float
    output: str  # The actual system output
    expected_output: Optional[str] = None  # Snapshot at eval time
    latency_ms: Optional[float] = None
    token_count: Optional[int] = None
    cost: Optional[float] = None
    error: Optional[str] = None
    
    # Judge info
    judge_type: str = "llm"  # "llm" | "exact" | "semantic" | "human"
    judge_config: dict = field(default_factory=dict)
    judge_reasoning: Optional[str] = None  # Why the judge gave this score
    
    # Timing
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
```

---

### 4. The Storage Layer

Now let's build the storage. We'll use SQLite for development (zero setup) with a schema that maps to PostgreSQL in production.

```python
"""
eval_platform/storage.py

SQLite storage for the eval platform.

DESIGN DECISIONS:
1. We use SQLite for dev (zero infrastructure), PostgreSQL for production.
   The SQL is ANSI-compatible WHERE POSSIBLE.
   
2. We store results in a JSON column for flexibility.
   Structured columns + JSON extension = best of both worlds.
   
3. We pre-compute run-level metrics on completion.
   Dashboard queries are FAST because they read aggregate rows,
   not raw results.

⚠️ There's a DESIGN FLAW in this schema. Can you find it?
"""

import sqlite3
import json
from datetime import datetime
from typing import Optional
from contextlib import contextmanager

# Schema
SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS eval_runs (
    id TEXT PRIMARY KEY,
    system_name TEXT NOT NULL,
    system_version TEXT NOT NULL,
    model_config TEXT NOT NULL,  -- JSON
    prompt_config TEXT,          -- JSON
    commit_hash TEXT,
    triggered_by TEXT NOT NULL,
    trigger_metadata TEXT,       -- JSON
    
    started_at TEXT NOT NULL,
    completed_at TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    
    total_cases INTEGER DEFAULT 0,
    passed INTEGER DEFAULT 0,
    failed INTEGER DEFAULT 0,
    avg_score REAL DEFAULT 0.0,
    
    branch TEXT,
    pr_number TEXT,
    author TEXT,
    metadata TEXT,               -- JSON
    
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS test_cases (
    id TEXT PRIMARY KEY,
    system_name TEXT NOT NULL,
    input TEXT NOT NULL,
    expected_output TEXT,
    category TEXT NOT NULL DEFAULT 'standard',
    difficulty TEXT NOT NULL DEFAULT 'medium',
    tags TEXT,                    -- JSON array
    metadata TEXT,                -- JSON
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    deprecated INTEGER DEFAULT 0
);

CREATE TABLE IF NOT EXISTS eval_results (
    id TEXT PRIMARY KEY,
    eval_run_id TEXT NOT NULL,
    test_case_id TEXT NOT NULL,
    passed INTEGER NOT NULL,
    score REAL NOT NULL,
    output TEXT NOT NULL,
    expected_output TEXT,
    latency_ms REAL,
    token_count INTEGER,
    cost REAL,
    error TEXT,
    judge_type TEXT NOT NULL DEFAULT 'llm',
    judge_config TEXT,            -- JSON
    judge_reasoning TEXT,
    started_at TEXT,
    completed_at TEXT,
    
    FOREIGN KEY (eval_run_id) REFERENCES eval_runs(id),
    FOREIGN KEY (test_case_id) REFERENCES test_cases(id)
);

CREATE TABLE IF NOT EXISTS metric_snapshots (
    id TEXT PRIMARY KEY,
    eval_run_id TEXT NOT NULL,
    metric_name TEXT NOT NULL,
    metric_value REAL NOT NULL,
    metric_p50 REAL,
    metric_p95 REAL,
    metric_p99 REAL,
    sample_size INTEGER,
    computed_at TEXT NOT NULL,
    
    FOREIGN KEY (eval_run_id) REFERENCES eval_runs(id)
);

-- Indexes for common queries
CREATE INDEX IF NOT EXISTS idx_runs_system ON eval_runs(system_name, started_at);
CREATE INDEX IF NOT EXISTS idx_runs_status ON eval_runs(status);
CREATE INDEX IF NOT EXISTS idx_results_run ON eval_results(eval_run_id);
CREATE INDEX IF NOT EXISTS idx_results_score ON eval_results(score);
CREATE INDEX IF NOT EXISTS idx_metrics_name ON metric_snapshots(metric_name, computed_at);
"""


class EvalDB:
    """
    Storage layer for the eval platform.
    
    Manages connection, schema, and basic CRUD.
    Production version would use connection pooling and migrations.
    """
    
    def __init__(self, db_path: str = ":memory:"):
        self.db_path = db_path
        self._conn: Optional[sqlite3.Connection] = None
    
    @contextmanager
    def connect(self):
        """Get a database connection."""
        if self._conn is None:
            self._conn = sqlite3.connect(self.db_path)
            self._conn.row_factory = sqlite3.Row
            self._conn.execute("PRAGMA journal_mode=WAL")
            self._conn.execute("PRAGMA foreign_keys=ON")
        yield self._conn
    
    def initialize(self):
        """Create schema tables."""
        with self.connect() as conn:
            conn.executescript(SCHEMA_SQL)
            conn.commit()
    
    def create_run(self, run: dict) -> str:
        """Create a new eval run. Returns the run ID."""
        with self.connect() as conn:
            conn.execute("""
                INSERT INTO eval_runs 
                (id, system_name, system_version, model_config, prompt_config,
                 commit_hash, triggered_by, trigger_metadata, started_at, status,
                 branch, pr_number, author, metadata)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                run["id"],
                run["system_name"],
                run["system_version"],
                json.dumps(run.get("model_config", {})),
                json.dumps(run.get("prompt_config", {})),
                run.get("commit_hash"),
                run["triggered_by"],
                json.dumps(run.get("trigger_metadata", {})),
                run.get("started_at", datetime.now().isoformat()),
                "pending",
                run.get("branch"),
                run.get("pr_number"),
                run.get("author"),
                json.dumps(run.get("metadata", {})),
            ))
            conn.commit()
            return run["id"]
    
    def complete_run(self, run_id: str, results: list[dict]):
        """
        Mark a run as completed with aggregated results.
        
        ⚠️ BUG: This method has a RACE CONDITION. 
        If called twice for the same run_id, it will:
        - Overwrite completed_at (fine)
        - But DOUBLE-COUNT the results (not fine)
        
        🤔 How would you fix this? (Before reading the fix below)
        """
        # Pre-compute aggregates
        total = len(results)
        passed = sum(1 for r in results if r["passed"])
        failed = total - passed
        avg_score = sum(r["score"] for r in results) / max(total, 1)
        
        with self.connect() as conn:
            conn.execute("""
                UPDATE eval_runs SET
                    status = 'completed',
                    completed_at = ?,
                    total_cases = ?,
                    passed = ?,
                    failed = ?,
                    avg_score = ?
                WHERE id = ?
            """, (
                datetime.now().isoformat(),
                total, passed, failed, avg_score,
                run_id,
            ))
            
            # Insert individual results
            for r in results:
                conn.execute("""
                    INSERT INTO eval_results
                    (id, eval_run_id, test_case_id, passed, score,
                     output, expected_output, latency_ms, token_count,
                     cost, error, judge_type, judge_config, judge_reasoning,
                     started_at, completed_at)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                """, (
                    r["id"], run_id, r["test_case_id"], int(r["passed"]),
                    r["score"], r["output"], r.get("expected_output"),
                    r.get("latency_ms"), r.get("token_count"), r.get("cost"),
                    r.get("error"), r.get("judge_type", "llm"),
                    json.dumps(r.get("judge_config", {})),
                    r.get("judge_reasoning"),
                    r.get("started_at"), r.get("completed_at"),
                ))
            
            conn.commit()
```

**🤔 The race condition fix:**

<details>
<summary>Think about it first, then check:</summary>

The fix: use status checks to make the operation idempotent.

```python
def complete_run(self, run_id: str, results: list[dict]):
    with self.connect() as conn:
        # Check current status
        current = conn.execute(
            "SELECT status FROM eval_runs WHERE id = ?", (run_id,)
        ).fetchone()
        
        if current is None:
            raise ValueError(f"Run {run_id} not found")
        if current["status"] != "running":
            raise ValueError(
                f"Run {run_id} has status '{current['status']}', expected 'running'"
            )
        
        # NOW proceed with completing
        # (Safe because the status check prevents double-completion)
```

This is the "optimistic lock" pattern — check before write. In PostgreSQL you'd use `SELECT ... FOR UPDATE` for stronger guarantees.
</details>

---

### 5. The API Layer (FastAPI)

```python
"""
eval_platform/api.py

FastAPI server for the eval platform.

Endpoints:
  POST   /api/v1/runs              → Create a new eval run
  POST   /api/v1/runs/{id}/results → Submit results for a run
  GET    /api/v1/runs/{id}         → Get run details + results
  GET    /api/v1/systems           → List all systems being evaluated
  GET    /api/v1/systems/{name}/trends → Get metric trends over time
  GET    /api/v1/dashboard/{system} → Dashboard data for a system
  POST   /api/v1/test-cases        → Create/update test cases
  GET    /api/v1/reports/weekly    → Generate weekly report

🤔 DESIGN DECISION: Should the API be SYNCHRONOUS (wait for eval)
or ASYNCHRONOUS (submit and poll)?

This API uses ASYNCHRONOUS submission because evals take 1-30 minutes.
The client submits results, not requests for evaluation.
"""
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import Optional
from datetime import datetime
import uuid

app = FastAPI(title="AI Eval Platform")


# =============================================
# Request/Response Models
# =============================================

class EvalRunCreate(BaseModel):
    system_name: str
    system_version: str
    model_config: dict = {}
    prompt_config: Optional[dict] = None
    commit_hash: Optional[str] = None
    triggered_by: str = "manual"
    branch: Optional[str] = None
    pr_number: Optional[str] = None
    author: Optional[str] = None
    metadata: dict = {}


class EvalRunResponse(BaseModel):
    id: str
    system_name: str
    system_version: str
    status: str
    total_cases: int = 0
    passed: int = 0
    failed: int = 0
    avg_score: float = 0.0
    started_at: str
    completed_at: Optional[str] = None
    triggered_by: str


class ResultSubmission(BaseModel):
    test_case_id: str
    passed: bool
    score: float
    output: str
    expected_output: Optional[str] = None
    latency_ms: Optional[float] = None
    token_count: Optional[int] = None
    cost: Optional[float] = None
    error: Optional[str] = None
    judge_type: str = "llm"
    judge_reasoning: Optional[str] = None


class ResultsBatch(BaseModel):
    results: list[ResultSubmission]


class TrendQuery(BaseModel):
    system_name: str
    metric_name: str
    days: int = 30
    granularity: str = "day"  # "run" | "hour" | "day" | "week"


# =============================================
# Endpoints
# =============================================

@app.post("/api/v1/runs", response_model=EvalRunResponse)
def create_run(run_data: EvalRunCreate, background_tasks: BackgroundTasks):
    """
    Create a new eval run.
    
    Returns the run immediately (eval happens asynchronously).
    The client submits results via POST /runs/{id}/results.
    """
    run_id = str(uuid.uuid4())
    
    # Store in database
    db.create_run({
        "id": run_id,
        **run_data.model_dump(),
        "started_at": datetime.now().isoformat(),
    })
    
    return EvalRunResponse(
        id=run_id,
        system_name=run_data.system_name,
        system_version=run_data.system_version,
        status="pending",
        started_at=datetime.now().isoformat(),
        triggered_by=run_data.triggered_by,
    )


@app.post("/api/v1/runs/{run_id}/results")
def submit_results(run_id: str, batch: ResultsBatch):
    """
    Submit evaluation results for a run.
    
    🤔 SECURITY CONCERN: This endpoint doesn't validate that the
    test_case_ids in the results actually EXIST in the database.
    A malicious or buggy client could submit results for
    non-existent test cases.
    
    Should we validate? What's the cost of validation (extra DB query
    for every result) vs. the risk (orphan results that don't link to
    valid test cases)?
    """
    results = []
    for r in batch.results:
        results.append({
            "id": str(uuid.uuid4()),
            "test_case_id": r.test_case_id,
            "passed": r.passed,
            "score": r.score,
            "output": r.output,
            "expected_output": r.expected_output,
            "latency_ms": r.latency_ms,
            "token_count": r.token_count,
            "cost": r.cost,
            "error": r.error,
            "judge_type": r.judge_type,
            "judge_reasoning": r.judge_reasoning,
            "started_at": datetime.now().isoformat(),
            "completed_at": datetime.now().isoformat(),
        })
    
    db.complete_run(run_id, results)
    
    return {"status": "ok", "results_submitted": len(results)}


@app.get("/api/v1/systems/{system_name}/trends")
def get_trends(system_name: str, metric: str = "avg_score", days: int = 30):
    """
    Get trend data for a system over time.
    
    🤔 PERFORMANCE QUESTION: This query reads ALL runs for a system
    in the time window, then computes the trend. For a system with
    50 runs/day × 30 days = 1,500 runs, this is fine.
    
    But what about a dashboard that shows TRENDS for ALL 5 systems
    at once? That's 5 × 1,500 = 7,500 rows. The dashboard loads SLOWLY.
    
    How would you PRE-COMPUTE trend data for fast dashboard loading?
    """
    with db.connect() as conn:
        rows = conn.execute("""
            SELECT 
                strftime('%Y-%m-%d', started_at) as day,
                AVG(avg_score) as avg_score,
                COUNT(*) as run_count
            FROM eval_runs
            WHERE system_name = ?
                AND status = 'completed'
                AND started_at >= datetime('now', ?)
            GROUP BY strftime('%Y-%m-%d', started_at)
            ORDER BY day
        """, (system_name, f"-{days} days")).fetchall()
        
        return {
            "system": system_name,
            "metric": metric,
            "days": days,
            "data_points": [
                {"date": r["day"], "value": r["avg_score"], "runs": r["run_count"]}
                for r in rows
            ]
        }


@app.get("/api/v1/dashboard/{system_name}")
def get_dashboard(system_name: str):
    """
    Get the full dashboard data for a system.
    
    Returns:
    - Latest run summary
    - 7-day trend
    - Category breakdown (critical/standard/info)
    - Recent regressions
    - Performance stats (latency, cost)
    
    This is the endpoint the DASHBOARD calls every time it loads.
    It MUST be fast (< 200ms).
    """
    with db.connect() as conn:
        # Latest run
        latest = conn.execute("""
            SELECT * FROM eval_runs 
            WHERE system_name = ? AND status = 'completed'
            ORDER BY started_at DESC LIMIT 1
        """, (system_name,)).fetchone()
        
        # 7-day trend (daily averages)
        trend = conn.execute("""
            SELECT 
                strftime('%Y-%m-%d', started_at) as day,
                AVG(avg_score) as avg_score,
                SUM(total_cases) as total_cases,
                SUM(passed) as total_passed,
                SUM(failed) as total_failed
            FROM eval_runs
            WHERE system_name = ?
                AND status = 'completed'
                AND started_at >= datetime('now', '-7 days')
            GROUP BY strftime('%Y-%m-%d', started_at)
            ORDER BY day
        """, (system_name,)).fetchall()
        
        # Category breakdown from latest run
        cat_breakdown = None
        if latest:
            cat_breakdown = conn.execute("""
                SELECT 
                    tc.category,
                    COUNT(*) as total,
                    SUM(CASE WHEN er.passed = 1 THEN 1 ELSE 0 END) as passed,
                    AVG(er.score) as avg_score
                FROM eval_results er
                JOIN test_cases tc ON er.test_case_id = tc.id
                WHERE er.eval_run_id = ?
                GROUP BY tc.category
            """, (latest["id"],)).fetchall()
        
        return {
            "system": system_name,
            "latest_run": {
                "id": latest["id"],
                "status": latest["status"],
                "total_cases": latest["total_cases"],
                "passed": latest["passed"],
                "failed": latest["failed"],
                "avg_score": latest["avg_score"],
                "completed_at": latest["completed_at"],
                "triggered_by": latest["triggered_by"],
            } if latest else None,
            "trend_7day": [
                {
                    "date": r["day"],
                    "avg_score": r["avg_score"],
                    "pass_rate": (r["total_passed"] / r["total_cases"]) 
                                 if r["total_cases"] > 0 else 0,
                }
                for r in trend
            ],
            "category_breakdown": [
                {
                    "category": r["category"],
                    "pass_rate": r["passed"] / r["total"] if r["total"] > 0 else 0,
                    "avg_score": r["avg_score"],
                }
                for r in (cat_breakdown or [])
            ],
        }
```

---

### 6. The Dashboard Frontend (Conceptual)

I won't build a full frontend (that's your Phase 7 project), but here's the key architectural decision:

```
┌────────────────────────────────────────────────────┐
│                    DASHBOARD                         │
│                                                      │
│  ┌──────────────────────────────┐                   │
│  │ SYSTEM HEALTH OVERVIEW       │                   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐   │  ← One card      │
│  │  │RAG  │ │Agent│ │Clsf │   │    per system      │
│  │  │ 94% │ │ 87% │ │ 92% │   │    (latest score)  │
│  │  └─────┘ └─────┘ └─────┘   │                   │
│  └──────────────────────────────┘                   │
│                                                      │
│  ┌──────────────────────────────┐                   │
│  │ TREND (selected system)      │                   │
│  │ ╱╲ ╱╲ ╱╲ ╱╲ ╱╲ ╱╲ ╱╲      │  ← 30-day trend  │
│  │╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱   │    with threshold │
│  │         ╌──── REGRESSION ──  │  (80% threshold)  │
│  └──────────────────────────────┘                   │
│                                                      │
│  ┌──────────────┐ ┌──────────────┐                  │
│  │ CATEGORY     │ │ RECENT       │                  │
│  │ BREAKDOWN    │ │ REGRESSIONS  │  ← Drill-down   │
│  │              │ │              │    to specific   │
│  │ Critical: 98%│ │ 2 days ago:  │    failing cases │
│  │ Standard: 91%│ │ faithfulness  │                  │
│  │ Info:     85%│ │ ↓ 0.12       │                  │
│  └──────────────┘ └──────────────┘                  │
└────────────────────────────────────────────────────┘
```

**🤔 Dashboard design principle:** Every widget must answer exactly ONE question. If a widget requires explanation, it fails. The questions your dashboard answers:

1. **"Are we good right now?"** → Latest scores per system (top row)
2. **"Are we getting better or worse?"** → Trend charts (middle)
3. **"What's broken?"** → Category breakdown + regression list (bottom)
4. **"What do I do about it?"** → Drill-down to specific failing test cases

If you can't map each widget to one of these four questions, remove the widget.

---

### 7. Integration Patterns: CI/CD, Slack, and GitHub

An eval platform is useless if people have to visit it. The platform must COME TO THEM.

```python
"""
eval_platform/integrations.py

Integrations with external tools.

The pattern is always the same:
1. Something happens (eval completes, regression detected, report due)
2. We format a message for the target platform
3. We send it via the target platform's API

🤔 DESIGN QUESTION: Should integrations be PUSH (we send messages)
or PULL (others query our API)?

Both. Push for alerts and notifications (time-sensitive).
Pull for dashboards and reports (user-initiated).
"""

import httpx
import json
from datetime import datetime, timedelta
from typing import Optional


class SlackNotifier:
    """
    Send eval results to Slack channels.
    
    Every eval run completion triggers a message.
    Regressions trigger an ALERT (different formatting, @here).
    Weekly summaries go to the #ai-eng channel.
    """
    
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url
    
    async def notify_run_completed(self, run: dict, summary_url: str):
        """
        Send a formatted message about a completed eval run.
        
        🤔 DESIGN CHOICE: Should we send a message for EVERY run?
        A CI/CD pipeline might run evals 20 times/day.
        That's 20 Slack messages per day. The team will MUTE the channel.
        
        Maybe send only:
        - Runs with regressions (score dropped vs baseline)
        - Runs on main branch (production changes)
        - First run of the day (daily summary)
        """
        emoji = "✅" if run["passed"] >= run["total_cases"] * 0.8 else "⚠️"
        
        blocks = [
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": (
                        f"{emoji} *Eval Run Complete: {run['system_name']} v{run['system_version']}*\n"
                        f"• Score: *{run['avg_score']:.1%}* "
                        f"({run['passed']}/{run['total_cases']} passed)\n"
                        f"• Trigger: {run['triggered_by']} "
                        f"• Branch: `{run['branch'] or 'main'}`\n"
                        f"• Commit: `{run['commit_hash'][:8] if run['commit_hash'] else 'N/A'}`"
                    )
                },
                "accessory": {
                    "type": "button",
                    "text": {"type": "plain_text", "text": "View Details"},
                    "url": summary_url,
                }
            }
        ]
        
        # Add regression alert if score dropped significantly
        if run.get("regression_detected"):
            blocks.insert(0, {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": (
                        f"🚨 *REGRESSION DETECTED*\n"
                        f"Score dropped from {run['previous_score']:.1%} "
                        f"to {run['avg_score']:.1%} "
                        f"({run['score_drop']:.1%} decrease)"
                    )
                }
            })
        
        async with httpx.AsyncClient() as client:
            await client.post(self.webhook_url, json={"blocks": blocks})
    
    async def send_weekly_report(self, systems: list[dict]):
        """
        Send a weekly summary of all system health.
        
        Format:
        📊 *AI System Health — Week of May 10*
        
        • RAG Bot: 93% (↑ 2% vs last week) ✅
        • Classifier: 87% (↓ 3% vs last week) ⚠️
        • Code Assist: 91% (→ 0% vs last week) ✅
        
        :chart_with_upwards_trend: *Trend Highlights*
        • Classifier regression started Tue, caused by embedding model update
        • RAG Bot improving since prompt change on Wed
        
        *Top 3 failing test cases:*
        1. classifier: "refund policy edge case" — 0.42 (critical)
        2. rag-bot: "multi-lingual query" — 0.51
        3. code-assist: "TypeScript generics" — 0.63
        """
        # Same pattern as above — format blocks, send to channel
        pass


class CIGateIntegration:
    """
    Integration with CI/CD pipelines (GitHub Actions, GitLab CI, etc.).
    
    The eval platform provides a CLI tool that:
    1. Creates an eval run
    2. Runs the evaluation
    3. Submits results
    4. Returns pass/fail based on the gate decision
    
    The CI pipeline calls:
    ```bash
    eval-cli run --system rag-bot --config eval-config.yaml
    ```
    
    If the gate fails (score below threshold, critical failures),
    the CLI exits with code 1, which blocks the CI pipeline.
    """
    
    @staticmethod
    def generate_github_action(system_name: str) -> str:
        """
        Generate a GitHub Actions workflow that runs evals on every PR.
        
        🤔 DESIGN QUESTION: Running 500 eval cases on every PR
        takes 5-30 minutes. Developers will HATE waiting.
        
        Solutions:
        A. Run ALL evals on every PR (slow but thorough)
        B. Run CRITICAL evals only (fast but might miss stuff)
        C. Run FULL evals nightly, QUICK evals on PRs
        
        Which one is correct? It depends on team size, deploy frequency,
        and risk tolerance.
        """
        return f"""
name: AI Eval — {system_name}

on:
  pull_request:
    paths:
      - 'systems/{system_name}/**'
      - 'prompts/{system_name}/**'

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Evaluation
        run: |
          eval-cli run \\
            --system {system_name} \\
            --config .eval/{system_name}.yaml \\
            --commit ${{{{ github.sha }}}} \\
            --pr ${{{{ github.event.pull_request.number }}}} \\
            --branch ${{{{ github.head_ref }}}}
        
        # If exit code != 0, the PR is blocked
        # The developer must check eval results before merging
"""
```

---

### 8. Multi-Team Namespacing

When multiple teams use the same platform, you need isolation:

```
API Structure:
  /api/v1/teams/{team_id}/systems/{system_name}/...
  
Database:
  Tables have a team_id column
  Queries always filter by team_id
  
Dashboard:
  Each team sees only their systems
  Admin sees ALL systems (cross-team comparison)
```

**🤔 The hard question:** Should teams SHARE test cases? If Team A finds a great test case for "ambiguous user intent" that catches a subtle bug, should Team B (different system) also be able to use it?

**Arguments for sharing:**
- Better test coverage across teams
- Cross-pollination of failure modes
- Reduced duplication

**Arguments against sharing:**
- Team B's system might not need that test case
- Different quality bars (Team B's "pass" is Team A's "fail")
- Test case deprecation becomes coordination nightmare

**The pragmatic solution:** Shared test case REPOSITORY (find + copy), with team-local COPIES. If Team A's test case is useful to Team B, Team B copies it (fork in database) and can modify it for their needs. The platform tracks the SOURCE of shared test cases so improvements can propagate.

---

## 💻 Code Examples

### Example 1: The Complete Eval Client SDK

Here's what the SDK looks like that teams use to submit results:

```python
"""
eval_platform/client.py

Client SDK for the eval platform.

Usage:
    from eval_platform.client import EvalClient
    
    client = EvalClient(api_url="http://eval.platform.com", api_key="...")
    
    # Create a run
    run = client.create_run(
        system_name="rag-bot",
        system_version="2.1.0",
        model_config={"model": "gpt-4o", "temperature": 0.1},
        commit_hash="abc123",
        triggered_by="ci/cd",
    )
    
    # Run evaluation (your code)
    results = run_evaluation(test_cases)
    
    # Submit results
    client.submit_results(run.id, results)

⚠️ There's a SUBTLE BUG in the submit_results method. Can you spot it?
"""

import httpx
from typing import Optional
from datetime import datetime


class EvalClient:
    """Client for interacting with the eval platform API."""
    
    def __init__(self, api_url: str, api_key: str, timeout: int = 30):
        self.api_url = api_url.rstrip("/")
        self.api_key = api_key
        self.timeout = timeout
        self._client = httpx.Client(
            base_url=self.api_url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=timeout,
        )
    
    def create_run(
        self,
        system_name: str,
        system_version: str,
        model_config: Optional[dict] = None,
        commit_hash: Optional[str] = None,
        triggered_by: str = "manual",
        branch: Optional[str] = None,
        pr_number: Optional[str] = None,
        author: Optional[str] = None,
    ) -> dict:
        """Create a new eval run and return the run metadata."""
        response = self._client.post("/api/v1/runs", json={
            "system_name": system_name,
            "system_version": system_version,
            "model_config": model_config or {},
            "commit_hash": commit_hash,
            "triggered_by": triggered_by,
            "branch": branch,
            "pr_number": pr_number,
            "author": author,
        })
        response.raise_for_status()
        return response.json()
    
    def submit_results(
        self,
        run_id: str,
        results: list[dict],
    ) -> dict:
        """
        Submit evaluation results for a run.
        
        ⚠️ BUG: This method sends ALL results in a single HTTP request.
        For a run with 1000 test cases, the request body could be 5-10MB.
        
        Problems:
        1. HTTP timeout — large payloads take time to serialize/transfer
        2. Server limits — most servers have max request body limits
        3. Partial failure — if one result is malformed, ALL fail
        
        🤔 How would you redesign this for large result sets?
        """
        response = self._client.post(
            f"/api/v1/runs/{run_id}/results",
            json={"results": results},
        )
        response.raise_for_status()
        return response.json()
    
    def get_run(self, run_id: str) -> dict:
        """Get run details including results."""
        response = self._client.get(f"/api/v1/runs/{run_id}")
        response.raise_for_status()
        return response.json()
    
    def get_trends(
        self,
        system_name: str,
        metric: str = "avg_score",
        days: int = 30,
    ) -> dict:
        """Get trend data for a system."""
        response = self._client.get(
            f"/api/v1/systems/{system_name}/trends",
            params={"metric": metric, "days": days},
        )
        response.raise_for_status()
        return response.json()
```

**🤔 Fix for the large payload problem:**

<details>
<summary>Think first, then compare:</summary>

**Solution: Batch submission with chunking.**

```python
def submit_results_batched(
    self,
    run_id: str,
    results: list[dict],
    batch_size: int = 100,
) -> list[dict]:
    """
    Submit results in batches to avoid large payloads.
    
    Each batch is sent independently. If one batch fails,
    the others still succeed (partial completion).
    """
    responses = []
    for i in range(0, len(results), batch_size):
        batch = results[i:i + batch_size]
        try:
            resp = self._client.post(
                f"/api/v1/runs/{run_id}/results",
                json={"results": batch},
            )
            resp.raise_for_status()
            responses.append(resp.json())
        except httpx.HTTPStatusError as e:
            # Log the failure but continue with other batches
            print(f"Batch {i//batch_size} failed: {e}")
            responses.append({
                "status": "error",
                "batch_start": i,
                "batch_size": len(batch),
                "error": str(e),
            })
    
    return responses
```

**Alternative: Streaming upload.** Send results as a stream (NDJSON) instead of a single JSON payload. The server processes results as they arrive, not after the full payload is received. This is more complex but handles arbitrarily large datasets.

```python
def submit_results_streaming(
    self,
    run_id: str,
    results: list[dict],
):
    """Submit results as a streaming NDJSON payload."""
    import json
    
    def generate():
        for r in results:
            yield json.dumps(r) + "\n"
    
    response = self._client.post(
        f"/api/v1/runs/{run_id}/results/stream",
        content=generate(),
        headers={"Content-Type": "application/x-ndjson"},
    )
    response.raise_for_status()
    return response.json()
```
</details>

---

### Example 2: Weekly Report Generator

```python
"""
eval_platform/reports.py

Automated report generation.

Generates weekly reports that summarize:
- Score trends (up/down/flat for each system)
- Notable regressions (cases that dropped significantly)
- Top failing test cases (what's consistently broken)
- Recommendations (what to fix next week)

🤔 DESIGN DECISION: Should reports be GENERATED (text + charts)
or should they link to the DASHBOARD (interactive)?

Generated reports: Useful for email, Slack, PDF. Harder to maintain.
Dashboard links: Always up-to-date. Requires clicking through.

BEST: Report with key HIGHLIGHTS + links to dashboard for details.
"""

from datetime import datetime, timedelta
from typing import Optional


class WeeklyReport:
    """
    Generates a weekly evaluation summary.
    
    The report answers:
    1. Which systems improved/worsened vs last week?
    2. What are the top 3 failing test cases?
    3. Are there any NEW failure modes (never-passed cases)?
    4. What's the trend direction (improving/stable/declining)?
    """
    
    def __init__(self, db):
        self.db = db
    
    def generate(self, team_id: str) -> dict:
        """
        Generate the weekly report for a team.
        
        Compares THIS week (last 7 days) vs PREVIOUS week (7-14 days ago).
        """
        now = datetime.now()
        this_week_start = now - timedelta(days=7)
        prev_week_start = now - timedelta(days=14)
        
        with self.db.connect() as conn:
            # Get all systems for this team
            systems = conn.execute(
                "SELECT DISTINCT system_name FROM eval_runs WHERE team_id = ?",
                (team_id,)
            ).fetchall()
            
            system_reports = []
            for sys in systems:
                sys_name = sys["system_name"]
                
                # This week's stats
                this_week = conn.execute("""
                    SELECT 
                        AVG(avg_score) as avg_score,
                        COUNT(*) as run_count,
                        SUM(passed) as total_passed,
                        SUM(total_cases) as total_cases
                    FROM eval_runs
                    WHERE system_name = ?
                        AND status = 'completed'
                        AND started_at >= ?
                        AND started_at < ?
                """, (sys_name, this_week_start, now)).fetchone()
                
                # Previous week's stats
                prev_week = conn.execute("""
                    SELECT AVG(avg_score) as avg_score
                    FROM eval_runs
                    WHERE system_name = ?
                        AND status = 'completed'
                        AND started_at >= ?
                        AND started_at < ?
                """, (sys_name, prev_week_start, this_week_start)).fetchone()
                
                # Trend direction
                if this_week["run_count"] == 0:
                    trend = "no_data"
                    change = 0
                elif prev_week["avg_score"] is None:
                    trend = "baseline"
                    change = 0
                else:
                    change = (this_week["avg_score"] or 0) - (prev_week["avg_score"] or 0)
                    if change > 0.02:
                        trend = "improving"
                    elif change < -0.02:
                        trend = "declining"
                    else:
                        trend = "stable"
                
                # Top failing test cases this week
                top_failures = conn.execute("""
                    SELECT 
                        tc.id,
                        tc.input,
                        tc.category,
                        AVG(er.score) as avg_score,
                        COUNT(*) as attempt_count,
                        SUM(CASE WHEN er.passed = 0 THEN 1 ELSE 0 END) as fail_count
                    FROM eval_results er
                    JOIN test_cases tc ON er.test_case_id = tc.id
                    JOIN eval_runs erun ON er.eval_run_id = erun.id
                    WHERE erun.system_name = ?
                        AND erun.status = 'completed'
                        AND erun.started_at >= ?
                    GROUP BY tc.id
                    HAVING fail_count > 0
                    ORDER BY avg_score ASC
                    LIMIT 5
                """, (sys_name, this_week_start)).fetchall()
                
                system_reports.append({
                    "system_name": sys_name,
                    "this_week_score": round(this_week["avg_score"] or 0, 3),
                    "prev_week_score": round(prev_week["avg_score"] or 0, 3),
                    "change": round(change, 3),
                    "trend": trend,
                    "run_count": this_week["run_count"] or 0,
                    "pass_rate": (
                        (this_week["total_passed"] / this_week["total_cases"])
                        if this_week["total_cases"] and this_week["total_cases"] > 0
                        else 0
                    ),
                    "top_failures": [
                        {
                            "id": f["id"],
                            "input": f["input"][:100],  # Truncate for readability
                            "category": f["category"],
                            "avg_score": round(f["avg_score"], 3),
                            "fail_rate": f["fail_count"] / f["attempt_count"],
                        }
                        for f in top_failures
                    ],
                })
            
            return {
                "generated_at": now.isoformat(),
                "period": {
                    "start": this_week_start.isoformat(),
                    "end": now.isoformat(),
                },
                "systems": system_reports,
                "summary": {
                    "improving": sum(1 for s in system_reports if s["trend"] == "improving"),
                    "declining": sum(1 for s in system_reports if s["trend"] == "declining"),
                    "stable": sum(1 for s in system_reports if s["trend"] == "stable"),
                    "no_data": sum(1 for s in system_reports if s["trend"] == "no_data"),
                }
            }
```

---

### Example 3: The Alert Engine

```python
"""
eval_platform/alerts.py

Alert engine that monitors eval runs and triggers alerts.

🤔 DESIGN QUESTION: How do you prevent alert fatigue?

Rule 1: Every alert must be ACTIONABLE. "Score dropped 3%" is not actionable.
         "Faithfulness on 'refund' queries dropped 8% (critical test case C-42 failed)"
         IS actionable.
         
Rule 2: Every alert must specify WHO owns it. Unowned alerts are ignored.
Rule 3: Alert thresholds should be AUTOMATICALLY CALIBRATED based on
         historical variance, not hard-coded.
"""

from datetime import datetime, timedelta
from typing import Optional


class AlertRule:
    """
    A single alert rule.
    
    Rules are evaluated when a new eval run completes.
    If the rule triggers, an alert is sent.
    
    🤔 The problem: Most alert rules are TOO SIMPLE.
    "Score < 0.8" catches regressions but also catches normal variation.
    
    BETTER: "Score dropped > 2 standard deviations below the 30-day moving average
             AND at least 5 critical test cases failed."
    
    This filters out noise while catching meaningful regressions.
    """
    
    def __init__(
        self,
        name: str,
        system_name: str,
        metric: str,
        condition: str,  # "below_threshold" | "drop_vs_baseline" | "zscore_exceeded"
        threshold: float,
        min_failures: int = 1,
        cooldown_minutes: int = 60,
        channels: list[str] = None,
    ):
        self.name = name
        self.system_name = system_name
        self.metric = metric
        self.condition = condition
        self.threshold = threshold
        self.min_failures = min_failures
        self.cooldown_minutes = cooldown_minutes  # Don't re-alert too quickly
        self.channels = channels or ["slack"]
        self._last_alert_at: Optional[datetime] = None
    
    def evaluate(self, run: dict, baseline: dict) -> Optional[dict]:
        """Evaluate this rule against a run. Returns alert dict or None."""
        # Check cooldown
        if self._last_alert_at:
            elapsed = (datetime.now() - self._last_alert_at).total_seconds()
            if elapsed < self.cooldown_minutes * 60:
                return None  # Still in cooldown
        
        # Check condition
        triggered = False
        details = {}
        
        if self.condition == "below_threshold":
            score = run.get("avg_score", 1.0)
            if score < self.threshold:
                triggered = True
                details = {
                    "type": "below_threshold",
                    "score": score,
                    "threshold": self.threshold,
                }
        
        elif self.condition == "drop_vs_baseline":
            current = run.get("avg_score", 1.0)
            previous = baseline.get("avg_score", current)
            drop = previous - current
            if drop > self.threshold:
                triggered = True
                details = {
                    "type": "drop_vs_baseline",
                    "current": current,
                    "previous": previous,
                    "drop": drop,
                    "threshold": self.threshold,
                }
        
        elif self.condition == "zscore_exceeded":
            # Z-score: how many standard deviations from the mean
            current = run.get("avg_score", 1.0)
            mean = baseline.get("mean", 1.0)
            std = baseline.get("std", 0.05)
            if std > 0:
                zscore = (mean - current) / std
                if zscore > self.threshold:
                    triggered = True
                    details = {
                        "type": "zscore_exceeded",
                        "zscore": zscore,
                        "current": current,
                        "mean": mean,
                        "std": std,
                    }
        
        if triggered and run.get("failed", 0) >= self.min_failures:
            self._last_alert_at = datetime.now()
            return {
                "rule_name": self.name,
                "system": self.system_name,
                "triggered_at": datetime.now().isoformat(),
                "severity": self._determine_severity(details),
                "details": details,
            }
        
        return None
    
    def _determine_severity(self, details: dict) -> str:
        """Determine alert severity based on impact."""
        if details.get("zscore", 0) > 5:
            return "critical"
        elif details.get("zscore", 0) > 3:
            return "warning"
        elif details.get("drop", 0) > 0.1:
            return "critical"
        elif details.get("drop", 0) > 0.05:
            return "warning"
        return "info"


class AlertManager:
    """
    Manages multiple alert rules and coordinates alert delivery.
    
    Handles deduplication: if two rules trigger for the same root cause,
    only ONE alert is sent (with both rules cited).
    """
    
    def __init__(self, notifier: SlackNotifier):
        self.rules: list[AlertRule] = []
        self.notifier = notifier
    
    def add_rule(self, rule: AlertRule):
        self.rules.append(rule)
    
    def evaluate_run(self, run: dict, baseline: dict):
        """Evaluate all rules against a completed run."""
        alerts = []
        for rule in self.rules:
            if rule.system_name != run.get("system_name"):
                continue
            alert = rule.evaluate(run, baseline)
            if alert:
                alerts.append(alert)
        
        if alerts:
            # Deduplicate: if multiple alerts point to the same system,
            # combine into one notification
            self._dispatch_alerts(alerts)
    
    def _dispatch_alerts(self, alerts: list[dict]):
        """Send alerts to configured channels."""
        for alert in alerts:
            # In production, this would fan out to Slack, email, PagerDuty
            print(f"ALERT [{alert['severity'].upper()}]: {alert['rule_name']}")
            print(f"  System: {alert['system']}")
            print(f"  Details: {alert['details']}")
```

---

## ✅ Good Output Examples

### What a well-built eval platform looks like

**1. The "5-second dashboard" test**

An engineer opens the dashboard. Within 5 seconds they can answer:
- Is my system healthy? (large colored indicator at top)
- Has quality changed since yesterday? (trend sparkline beside each system)
- Is there anything I need to investigate? (prominent "regressions" section)

```
┌──────────────────────────────────────────────────────────────┐
│  AI System Health Dashboard                     Last updated: │
│                                                    2 min ago │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ 🟢 RAG Bot       94.2%  ▁▃▆▇▇▆▇  +0.8% this week       ││
│  │ 🟡 Code Assist   87.1%  ▇▅▄▃▂▃▄  -2.1% this week ⚠️     ││
│  │ 🔴 Classifier    72.3%  ▇▆▅▃▂▁▁  -11.4% this week 🚨   ││
│  └──────────────────────────────────────────────────────────┘│
│  ...drill into classifier...                                  │
│  Regression started May 14 (3 days ago)                       │
│  Root cause: embedding model update on May 14                 │
│  Affected categories: technical (-18%), billing (-9%)         │
│  Top failing case: "multi-language support query" (0.31)      │
└──────────────────────────────────────────────────────────────┘
```

**2. The "10-second PR review"**

A developer opens a PR. The CI check "AI Eval" is green. The GitHub check shows:
- ✅ Eval passed: 94.2% (baseline: 93.8%, +0.4%)
- ✅ No regressions detected
- ✅ All critical test cases passed
- 📊 [View full eval report](link)

The developer merges with confidence.

**3. The "Monday morning scan"**

The team's Slack channel shows:

```
📊 AI Weekly Report — Week of May 10

🟢 RAG Bot: 94.2% (+0.8%) — Improving
   • "ambiguous refund query" regression fixed (was 0.52, now 0.89)
   
🟡 Code Assist: 87.1% (-2.1%) — Declining ⚠️
   • New regression in TypeScript generics (dropped from 0.91 to 0.63)
   • Investigation needed: did the prompt change on Tuesday cause this?
   
🔴 Classifier: 72.3% (-11.4%) — Critical 🚨
   • Embedding model update on May 14 caused retrieval shift
   • Rollback recommended if not fixed within 48 hours
   • [View details](link)
```

The engineering manager reads this in 30 seconds and knows exactly what to prioritize.

---

**4. The adoption metric**

The platform's success isn't measured by features but by ADOPTION:
- 90%+ of deploys go through eval gates
- 100% of systems have eval datasets
- Weekly report read rate > 80%
- Mean time to detect regression < 1 hour
- Mean time to respond to regression < 2 hours

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The Dashboard of Everything

**What:** 47 metrics, 12 charts, 8 system cards, auto-refresh every 30 seconds. Every metric is green, so nobody looks at it. When a metric goes red, nobody notices because they stopped checking.

**Why it happens:** The builder (you) wants to be thorough. Every metric seems important. "What if someone needs this?"

**The fix:** The 4-question dashboard (above). Every widget must answer exactly one of: "Are we good?", "Getting better or worse?", "What's broken?", "What do I do?"

### ❌ Antipattern 2: The Silent Platform

**What:** The platform has all the data. But nobody knows it exists, or knows how to use it, or remembers to check it.

**Why it happens:** You built the platform, you know how to use it. You assume others will figure it out. They don't.

**The fix:** The platform must PUSH notifications, not wait for PULL visits. Every eval run → Slack message. Every regression → alert. Weekly → report. The platform comes to them.

### ❌ Antipattern 3: The Monolithic Eval

**What:** One eval dataset for all systems. All teams use the same metrics. Any change to the eval framework affects everyone.

**Why it happens:** It's easier to build one platform than to design for extensibility.

**The fix:** Platform provides INFRASTRUCTURE (storage, API, dashboards, alerts). Teams provide CONTENT (test cases, metrics, thresholds). The platform is the scaffolding; teams fill in the details.

### ❌ Antipattern 4: The "Just Check the Dashboard" Problem

**Scenario:** A developer asks "is this deployment safe?" The answer is "just check the dashboard."

**Problem:** The developer has to LEAVE their workflow (IDE, terminal, CI pipeline) to visit a separate website. The friction means they either don't check or forget.

**The fix:** Embed eval results WHERE the developer already works:
- GitHub PR checks (✅/❌ on the PR page)
- CI pipeline output (pass/fail with details)
- CLI tool (`eval-cli status --system rag-bot`)
- IDE plugin showing eval status for current branch

### ❌ Antipattern 5: The Unclear Ownership Model

**Scenario:** A regression is detected. The alert goes to #ai-eng channel. Everyone looks at it. Nobody owns it. By end of day, it's still unfired because "I thought Bob was handling it."

**The fix:** Every alert must have an OWNER. If no owner is assigned within 30 minutes, the alert escalates. The system:
1. Auto-assigns based on: "which team last changed this system?"
2. If unowned: escalates to team lead
3. If team lead doesn't respond: escalates to engineering manager

---

## 🧪 Drills & Challenges

### Drill 1: Schema Critique (20 min)

You're reviewing an eval platform schema designed by a colleague:

```sql
CREATE TABLE runs (
    id INTEGER PRIMARY KEY,
    system TEXT,
    score REAL,
    passed INTEGER,
    total INTEGER,
    created_at TEXT
);

CREATE TABLE results (
    id INTEGER PRIMARY KEY,
    run_id INTEGER,
    case_input TEXT,
    expected TEXT,
    actual TEXT,
    score REAL
);
```

**Your task:** List EVERY problem with this schema. Think about:
- What queries would be SLOW?
- What data is LOST?
- What can't you answer?
- What happens at scale (10M+ rows)?
- What metadata is missing?

**🤔 Bonus:** Redesign the schema. What would you ADD, REMOVE, or CHANGE?

---

### Drill 2: Build a Dashboard Widget (30 min)

You need to build a "System Health Score" widget that combines multiple metrics into ONE number (0-100).

**Inputs:**
- `avg_score`: 0.0-1.0 (overall evaluation score) — weight: 40%
- `pass_rate`: 0.0-1.0 (% test cases passed) — weight: 30%
- `trend_direction`: -1.0 to 1.0 (30-day trend) — weight: 20%
- `coverage`: 0.0-1.0 (% test categories covered) — weight: 10%

**Your task:**
1. Write the formula that combines these into a 0-100 score
2. Add at least ONE penalty (capped at -15 points) for BAD conditions
3. Add at least ONE bonus (capped at +5 points) for GOOD conditions
4. Test your formula on these scenarios:
   - System A: 0.95 score, 0.98 pass, +0.05 trend, 0.80 coverage
   - System B: 0.72 score, 0.68 pass, -0.12 trend, 0.45 coverage
   - System C: 0.88 score, 0.85 pass, -0.02 trend, 0.95 coverage

**🤔 Thought question:** Should SYSTEM_HEALTH be an AGGREGATE formula or should it be the WORST single metric? (Worst metric: if any one thing is broken, the system is "unhealthy" even if everything else is great. Aggregate: high scores in 3 areas can hide a problem in 1 area.)

---

### Drill 3: The Integration Migration (25 min)

Team B has an existing eval script that:
1. Reads test cases from a JSON file (200 cases)
2. Runs the system under test
3. Judges outputs with a custom LLM judge prompt
4. Prints a summary to stdout
5. Exits with code 0 (all pass) or 1 (some failed)

Your eval platform has a REST API. The team doesn't want to rewrite their script.

**Your task:** Design a MINIMAL wrapper script (pseudocode OK) that:
- Wraps Team B's existing script
- Captures results
- Sends them to the platform API
- Preserves Team B's stdout output (they still see results in their terminal)
- Adds the platform upload as a NON-BLOCKING step (CI doesn't fail if upload fails)

Write the wrapper. Show the actual interface between the old script and the new platform.

---

### Drill 4: The Adoption Plan (20 min)

Your eval platform is built. You need the team to USE it. Write a 1-week rollout plan:

```
Day 1: [What happens?]
Day 2: [What happens?]
Day 3: [What happens?]
Day 4: [What happens?]
Day 5: [What happens?]
```

**Your plan must address:**
1. What's the BARRIER TO ENTRY? (How do they start with minimal effort?)
2. What's the INCENTIVE to use the platform? (What pain does it remove?)
3. What's the ESCALATION path for problems?
4. How do you measure adoption? What's the target?

**🤔 The hard question:** What if a team REFUSES to adopt the platform? They have their own system, it works for them, they don't see the value. The CTO says "you need to make them use it." What do you do? (Hint: mandating tools creates resentment. How do you CREATE desire?)

---

### Drill 5: The Alert Tuning Problem (30 min)

Your platform has 3 alert rules for the RAG system:

```
Rule 1: avg_score < 0.80 → critical alert
Rule 2: avg_score dropped > 0.05 vs previous run → warning alert  
Rule 3: any critical test case failed → critical alert
```

Here's the last 10 eval runs:

```
Run  1: avg_score 0.92, 0 critical failures
Run  2: avg_score 0.91, 0 critical failures
Run  3: avg_score 0.93, 0 critical failures
Run  4: avg_score 0.91, 0 critical failures
Run  5: avg_score 0.78, 0 critical failures  ← ALERT (Rule 1)
Run  6: avg_score 0.90, 0 critical failures  ← No alert (recovered?)
Run  7: avg_score 0.89, 0 critical failures
Run  8: avg_score 0.87, 1 critical failure   ← ALERT (Rule 3)
Run  9: avg_score 0.88, 0 critical failures
Run 10: avg_score 0.86, 0 critical failures
```

**Your task:**

1. Run 5 dropped to 0.78 but recovered to 0.90 on Run 6. Was this a real regression or noise? How would you TELL the difference with more data?

2. Run 8 had a critical test case failure. The alert fired. The team investigated for 2 hours. Root cause: the test case was flaky (fails 20% of the time due to judge variance). How do you DISCOVER flaky test cases from historical data? What metric would identify them?

3. **Design a FLAKY TEST DETECTOR.** Given the historical results of a test case (list of scores over many runs), write the logic that marks a test case as "potentially flaky" and should be investigated. What's your detection criteria?

---

### Drill 6: The Cross-Platform Export (20 min)

Your company is acquired. Their eval platform uses a different data model. Your CTO asks: "Can we BOTH keep using our platforms but have a UNIFIED view across both?"

The foreign platform's API returns:
```json
{
  "evaluations": [
    {
      "id": "eval-001",
      "timestamp": "2026-05-10T14:30:00Z",
      "model": "claude-sonnet-4",
      "metrics": [
        {"name": "accuracy", "value": 0.94},
        {"name": "latency_ms", "value": 2450}
      ],
      "test_name": "customer-support-v2",
      "passed": true,
      "metadata": {"prompt_version": "2.1", "dataset": "prod-sample-05"}
    }
  ]
}
```

**Your task:** Write a TRANSLATOR that:
1. Fetches evaluations from their API
2. Maps their data model to YOUR eval platform's model
3. Inserts them into your database with a `source: "acquired-company"` tag
4. Handles the fact that their metrics are DIFFERENT from yours (they have "accuracy", you have "faithfulness")

---

## 🚦 Gate Check

Before moving to File 09, verify you can:

1. **☐** Design the database schema for an eval platform (tables, indexes, relationships)
2. **☐** Build a REST API for submitting and querying eval results
3. **☐** Implement the eval run model — creating runs, submitting results, aggregating
4. **☐** Design a dashboard that answers the 4 key questions (health, trend, breakdown, action)
5. **☐** Integrate with external tools (Slack, CI/CD, GitHub)
6. **☐** Build an alert engine that avoids noise
7. **☐** Design for multi-team adoption (namespacing, team-specific configs)
8. **☐** Generate automated reports (weekly summaries, trend analysis)
9. **☐** Distinguish flaky test cases from real regressions

### 🛑 Stop and Think

**Question 1 — The Data Quality Problem**

Your eval platform has been running for 6 months. You have 10,000 eval runs, 500,000 test case results. But you suspect that 20% of the data is bad — flaky judges, misconfigured runs, corrupted test cases.

How do you CLEAN 6 months of data? How do you identify which runs are trustworthy and which aren't? What METADATA should you have been collecting at submission time to make this possible?

**Question 2 — The Scaling Question**

Your platform was built for 3 systems, 50 runs/day. Now you have 50 systems, 5,000 runs/day. The dashboard is slow. The alerts are overwhelming (50 alerts/day, 35 are noise). The database is 500GB.

What's your performance budget? Which queries must be FAST (< 100ms) vs. which can be SLOW (< 10s)? What architectural changes do you make?

**Question 3 — Cross-Phase Integration**

Choose ONE system from a previous phase:
- **Phase 1** — Your LLM Gateway's provider routing
- **Phase 2** — Your Prompt Optimization System's prompt variations
- **Phase 3** — Your Semantic Search Engine's embedding/retrieval
- **Phase 4** — Your Customer Support Bot's RAG pipeline
- **Phase 5** — Your Advanced RAG Engine's reranking
- **Phase 6** — Your Multi-Agent Research System's orchestration

Design the eval platform integration for that system:
1. What's the "eval submission" flow? (How does the system send results?)
2. What's the "alert rule" configuration? (What regressions matter?)
3. What's the "dashboard view"? (What does the system's card look like?)
4. What's the "weekly report" highlight? (What story does the report tell about this system?)

---

## 📚 Resources

- **"Building a Testing Platform for ML Systems"** (Uber, 2020) — How Uber built Eva, their ML evaluation platform. The architecture decisions (especially around dataset versioning and metric computation) directly informed this module's schema design.
- **"The Missing Piece in MLOps: Evaluation Infrastructure"** (Hugging Face, 2024) — Discusses why most MLOps platforms neglect evaluation infrastructure and how the community is filling the gap.
- **"Flink: The Dashboard Design Framework"** (Stripe, 2021) — Stripe's guide to designing dashboards that people actually use. The "one question per widget" principle is from this talk.
- **"Honeycomb: The Power of High-Cardinality Observability"** (Honeycomb, 2022) — How to think about high-cardinality data (which eval metrics are) and why traditional monitoring tools fail for evaluation data.
- **"Building a Multi-Tenant SaaS Platform"** (AWS, 2023) — Architecture patterns for multi-tenant platforms. The team namespacing design in this module follows the "silo" model (complete data isolation).
- **Cross-Phase: Phase 6 File 08 "Agent Evaluation"** — The four-level evaluation hierarchy defined there (atomic action → step → trajectory → task outcome) maps to four different dashboard views. A well-designed dashboard lets you drill from "overall system health" (task outcome) down to "this specific failing test case" (atomic action).
- **Cross-Phase: Phase 7 File 05 "Building Eval Pipelines"** — The hybrid pipeline design (tiered evaluation with early termination) from File 05 is what the platform's CI integration triggers. The platform receives the results; File 05 describes how those results are GENERATED.

**Next Module:**
→ **09-Project-Eval-Platform.md**: The capstone — build a complete AI Evaluation Platform that integrates everything from Files 01-08.
