# 07 — LLMOps & Model Registry: Managing Models in Production

## 🎯 Purpose & Goals

> 🛑 STOP. Before you build the perfect fine-tuned model, you need to answer a question most engineers ignore:

**What happens when you have MORE THAN ONE model?**

Right now you have 1 base model and 1 fine-tuned model. Simple. But six months from now:
- You'll have 3 versions of your FT model (v1, v2, v3 with different data)
- You'll have 2 function-calling variants (general and specialized)
- You'll have 1 base model that got updated by the provider (Llama 2 → Llama 3)
- You'll have 4 experimental models from failed experiments that someone MIGHT want to revisit
- You'll have 3 production deployments (staging, canary, production) running different model versions

Now you have 11 models. Which one is in production? Which one had the best eval scores? Which one is the "summarization" model vs the "function-calling" model? Can you roll back to last week's version?

**Without a model registry, you can't answer these questions. You just have a bunch of model weights with filenames like `ft_final_v2_real_final.pt`.**

**The goal of this module: Build a model registry that tracks versions, experiments, and metadata so you can manage models at scale.**

---

### 🤔 Discovery Question 1: The Deployment Disaster

Your team has been fine-tuning models for 3 months. You have 15 model checkpoints across 4 team members' machines and a shared drive.

One Friday at 4:45 PM, production breaks. The model starts generating nonsensical responses.

**Step 1**: Check which model is deployed.
- The Docker container uses `./models/ft-legal-v3`.
- But `ft-legal-v3` on the production server is a SYMLINK pointing to a network drive.
- The network drive has 5 folders: `ft-legal-v3a`, `ft-legal-v3b`, `ft-legal-v3-final`, `ft-legal-v3-actual-final`, `ft-legal-v3-please-work`.
- Nobody knows which one is actually deployed.

**Step 2**: Check the git log for the deployment config.
- `git log deploy_config.yaml` shows: "Updated model path to v3" — 6 weeks ago.
- But `git diff deploy_config.yaml` shows no changes since then. The config currently points to `models/ft-legal-v3` which is the symlink.

**Step 3**: Check who changed the symlink last.
- `ls -la` shows the symlink was changed 3 days ago.
- 3 people had access. Nobody remembers changing it.

**🤔 Before reading on, answer these:**

1. You're on call at 4:45 PM Friday. Production is broken. **What do you do RIGHT NOW to restore service?** Do you roll back the symlink? Do you deploy from a known good backup? How do you know what "known good" is?

2. The root cause was: someone uploaded a new model to the network drive, updated the symlink, the model was corrupted, and nobody tracked the change. **What SYSTEMIC changes would prevent this from ever happening again?** (Hint: Not "be more careful." Think about IMMUTABLE deployments — no symlinks, no overwriting, each deployment is a new version.)

3. **The hardest question:** Even if you fix the deployment process, the DETECTION took too long. You noticed the broken model because a user complained. The model had been broken for 3 days. **How would you AUTOMATICALLY detect a bad model deployment within minutes?** What metrics would you monitor? What threshold would trigger an alert? How do you distinguish "model is broken" from "unusual user queries"?

---

### 🤔 Discovery Question 2: The A/B Test That Broke Production

Your team decides to A/B test two fine-tuned models — "legal-summarization-v2" and "legal-summarization-v3" — to see which performs better.

The A/B test setup:
- 50% of traffic goes to v2 (the current production model)
- 50% of traffic goes to v3 (the new candidate)
- The test runs for 1 week
- Winner gets deployed to 100%

**Day 2**: A support ticket comes in: "The AI is giving WRONG answers!" A senior engineer investigates and discovers the issue is ONLY affecting users who are logged in via SSO. Further investigation shows: v3 was trained on data that was preprocessed differently for SSO users' queries (a data pipeline bug).

But because the traffic split is random, not all SSO users get v3. SOME SSO users get v2 (working fine). Some get v3 (broken). The random split means the bug is INVISIBLE in aggregate metrics — the v2 good results mask the v3 bad results for SSO users.

**🤔 Before reading on, answer these:**

1. The aggregate A/B test showed v3 performing 2% better than v2. But SSO users were getting terrible results. **How can a model be "2% better" overall but "40% worse" for a specific user segment?** What does this tell you about aggregate metrics?

2. What would you change about the A/B test setup to catch this? (Hint: Stratified A/B testing — split traffic BY user segment, not randomly. Or: monitor per-segment metrics, not just aggregate.)

3. **The hardest question:** The data pipeline bug affected SSO users' query preprocessing DURING TRAINING. But at INFERENCE time, both SSO and non-SSO queries go through the SAME preprocessing pipeline. The bug was only in the TRAINING data pipeline. **How would you detect a training data bug that manifests as degraded performance for a specific user segment?** What monitoring would catch this?

---

### 🤔 Discovery Question 3: The Rollback That Wasn't

You deploy "legal-summarization-v4." After 2 hours, you detect a regression — the model's output quality dropped 15% on legal citation accuracy. You decide to roll back to v3.

**Step 1**: Find the v3 model weights.
- You check the model registry.
- v3 is listed as "deployed from 2024-04-01 to 2024-04-15."
- The registry has a storage path: `s3://models/legal-summarization/v3/`
- You download and deploy v3.

**Step 2**: Test v3 after rollback.
- v3 scores are LOWER than they were when v3 was originally in production.
- Original v3 accuracy: 87%. Current v3 accuracy: 82%.

**What happened?** The A/B test between v2 and v3 ran for a week. During that week, users gave feedback on v3 outputs. The feedback was used to IMPROVE the system prompt. Now v3 is running with a DIFFERENT system prompt than the one it was originally deployed with. The rollback restored the model weights but NOT the system prompt.

**🤔 Before reading on, answer these:**

1. A "rollback" should restore the system to a known good state. But this rollback only restored model weights — the system prompt, RAG configuration, and temperature settings were from v4's deployment. **What does a COMPLETE rollback require?** What artifacts besides model weights need to be versioned?

2. The team now needs to decide: (a) Roll back the system prompt too (but that means losing improvements from the past 2 weeks), or (b) Keep the current system prompt with the old model (but the prompt was optimized for v4, not v3). **What do you choose? Why? What are the risks of each?**

3. **The hardest question:** The rollback took 45 minutes because the team had to:
   1. Find the v3 model (5 min)
   2. Download it from S3 (10 min)
   3. Update the deployment config (5 min)
   4. Restart the service (2 min)
   5. Re-test and verify (23 min)
   
   **Design a deployment system where a rollback takes UNDER 1 MINUTE.** What architecture supports instant rollbacks? (Think blue/green deployment, traffic shifting, immutable deployments.)

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 45 min |
| Code Examples (2) | 1 hr |
| Drills (3) | 30 min |
| Gate Check | 15 min |
| **Total** | **~3 hr** |

---

## 📖 Concept Deep-Dive

### 1. What a Model Registry Actually Does

A model registry is NOT a file server with model weights. It's a DATABASE that tracks:

| Concept | What It Tracks | Why It Matters |
|---------|---------------|----------------|
| **Model Version** | Unique ID for each trained model | Know which model is deployed |
| **Source** | Training script, data version, config | Reproduce any model |
| **Metrics** | Eval scores, latency, cost | Compare models objectively |
| **Lineage** | Base model → FT data → FT model → Deployment | Trace any issue to its root cause |
| **Stage** | development → staging → canary → production | Know what's where |
| **Deployment** | Which environment, when, by whom | Audit trail |

### 2. The Minimum Viable Registry Schema

```python
@dataclass
class ModelRecord:
    model_id: str              # unique ID: "ft-legal-v3-20240401"
    model_name: str            # human name: "legal-summarization"
    version: str               # semantic: "3.0.0" or sequential: "v3"
    
    # Source
    base_model: str            # "meta-llama/Llama-2-7b-hf"
    training_data_version: str # hash or pointer to dataset version
    training_config: Dict      # hyperparameters, LoRA config
    training_script_hash: str  # git hash of training code
    
    # Artifacts
    model_path: str            # Storage location (local, S3, HF Hub)
    tokenizer_path: str        
    quant_config: Optional[Dict]
    
    # Metrics
    eval_metrics: Dict         # {"rougeL": 0.47, "bertscore": 0.85}
    regression_metrics: Dict   # {"code_gen": 0.42, "safety_toxicity": 0.031}
    latency_p50_ms: float
    latency_p99_ms: float
    
    # Lifecycle
    stage: str                 # "dev" | "staging" | "canary" | "production"
    created_at: datetime
    created_by: str
    deployed_at: Optional[datetime]
    retired_at: Optional[datetime]
    
    # Audit
    tags: Dict[str, str]       # "purpose": "summarization", "team": "legal"
    notes: str                 # Free-text notes
```

### 3. Model Versioning Strategies

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Semantic** | v1.0.0, v1.1.0, v2.0.0 | Human-readable, communicates change magnitude | Requires manual version bumps |
| **Sequential** | v1, v2, v3… | Simple, always increments | No info about change size |
| **Date-based** | 2024-04-01, 2024-04-15 | Chronological ordering | Awkward for multiple versions/day |
| **Hash-based** | a3f8c2e, b7d1f4a | Immutable, unique, source-traceable | Not human-friendly |
| **Hybrid** | v3-20240401-a3f8c2e | Best of all worlds | Longer strings |

**Senior Engineer Recommendation**: Use the HYBRID approach. It combines semantic ordering with temporal context and source traceability. `v3-20240401-a3f8c2e` tells you: it's version 3, from April 1 2024, and the training code was at git hash a3f8c2e.

### 4. Model Lifecycle Stages

```
                    ┌──────────┐
                    │  DEV     │  ← First training, local testing
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ STAGING  │  ← Integration testing, eval suite
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ CANARY   │  ← 5-10% production traffic
                    └────┬─────┘
                         │
               ┌─────────┴──────────┐
          PASS │                    │ FAIL (rollback)
      ┌────────▼──────┐      ┌──────▼───────┐
      │  PRODUCTION   │      │  ROLLED BACK │
      └────────┬──────┘      └──────────────┘
               │
          ┌────▼──────┐
          │  RETIRED   │  ← Replaced by newer version
          └───────────┘
```

**Key rule**: A model NEVER goes backwards in stages. It's `dev → staging → canary → production → retired`. Once retired, it can be restored but must go through the pipeline again.

### 5. Experiment Tracking

Beyond model versions, you need to track EXPERIMENTS — groups of training runs that test a hypothesis:

```python
@dataclass
class Experiment:
    experiment_id: str
    hypothesis: str            # "Increasing LoRA rank improves summarization quality"
    runs: List[ModelRecord]    # Multiple training runs testing the hypothesis
    winner: Optional[str]      # The best model_id from this experiment
    conclusion: str            # "Rank 16 is optimal for this dataset size"
```

**Why experiments matter**: Without them, you have isolated model versions but no understanding of WHY one version outperforms another. Experiments capture the LEARNING, not just the artifacts.

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Model Registry (Full of Bugs)

```python
"""
WHAT NOT TO DO — A model tracking system with 10+ bugs.

This "works" for tracking models but fails at every operational scenario.
"""

import json
import os
from datetime import datetime
from typing import Optional, Dict, List

# BUG 1: No database — just a JSON file
# JSON file can't handle concurrent writes, no ACID guarantees

REGISTRY_FILE = "model_registry.json"

# BUG 2: Global mutable state — no locking
registry_data = {"models": [], "current_production": None}

def load_registry():
    """Load registry from JSON file."""
    if not os.path.exists(REGISTRY_FILE):
        return {"models": [], "current_production": None}
    with open(REGISTRY_FILE, 'r') as f:
        return json.load(f)

def save_registry(data):
    """Save registry to JSON file."""
    # BUG 3: No atomic write — partial writes corrupt the file
    # BUG 4: No backup before overwrite
    with open(REGISTRY_FILE, 'w') as f:
        json.dump(data, f, indent=2)

def register_model(model_path: str, metrics: Dict):
    """Register a model in the registry."""
    data = load_registry()
    
    # BUG 5: Missing required metadata
    # No: base_model, training data version, training config, creator
    # Without this, you can't reproduce the model
    model_entry = {
        "id": f"model-{len(data['models']) + 1}",  # BUG 6: ID based on count — not unique if models are deleted
        "path": model_path,
        "metrics": metrics,
        "registered_at": datetime.now().isoformat(),
        # BUG 7: No stage field — can't track lifecycle
        # BUG 8: No tags — can't search/filter
    }
    
    data["models"].append(model_entry)
    save_registry(data)
    return model_entry["id"]

def get_production_model():
    """Get the current production model."""
    data = load_registry()
    
    # BUG 9: Uses a string pointer instead of model reference
    # If "current_production" points to a model that was deleted, this returns None silently
    prod_id = data.get("current_production")
    if not prod_id:
        return None
    
    for model in data["models"]:
        if model["id"] == prod_id:
            return model
    
    return None

def set_production(model_id: str):
    """Set a model as production."""
    data = load_registry()
    
    # BUG 10: No validation — model_id might not exist
    data["current_production"] = model_id
    save_registry(data)
    
    # BUG 11: No audit log — who changed this? when?
    # BUG 12: No rollback — if this change breaks something, can't revert

def list_models():
    """List all registered models."""
    data = load_registry()
    return data["models"]

# BUG 13: No versioning of the registry itself
# If the registry file format changes, old data is incompatible
# BUG 14: No backup/restore
# BUG 15: No search/filter
# BUG 16: No metrics comparison
```

#### 🔍 Find The Bugs

| # | Bug | Why It's Bad | Fix |
|---|-----|-------------|-----|
| 1 | JSON file storage | No ACID, no concurrency | Use SQLite or a proper database |
| 2 | Global mutable state | Race conditions | Add locks or use database transactions |
| 3 | Non-atomic write | Partial write corrupts data | Write to temp file, then rename |
| 4 | No backup before overwrite | Can't recover from corruption | Keep last 3 backups |
| 5 | Missing required metadata | Can't reproduce the model | Require: base_model, data version, config |
| 6 | ID based on count | Not unique, changes | Use UUID or hash-based IDs |
| 7 | No stage field | Don't know model lifecycle | Add stage: dev/staging/canary/production |
| 8 | No tags | Can't search/filter | Add tags dict for arbitrary metadata |
| 9 | Silent None on missing ref | Deployment silently fails | Validate references, raise on error |
| 10 | No validation on set_production | Can point to nonexistent model | Check model exists before setting |
| 11 | No audit log | Can't track changes | Add changelog: who, what, when |
| 12 | No rollback | Can't revert bad changes | Keep version history of registry |
| 13 | No versioning of registry format | Format changes break old data | Include schema version in file |
| 14 | No backup/restore | Can't recover from corruption | Add backup/restore commands |
| 15 | No search/filter | Can't find models | Add query interface |
| 16 | No metrics comparison | Can't compare model quality | Add comparison view |

---

### Example 2: Production Model Registry (Fixed)

```python
"""
Production Model Registry & Experiment Tracker.

Features:
- SQLite-backed storage (SQL for flexibility, file for simplicity)
- Immutable model records (never modified, only superseded)
- Full metadata tracking (reproducibility)
- Lifecycle management (stage transitions)
- Audit logging
- Metrics comparison
- Rollback support
- Experiment tracking
"""

import json
import sqlite3
import uuid
import hashlib
from datetime import datetime
from dataclasses import dataclass, asdict
from typing import Optional, Dict, List, Any
from pathlib import Path
import logging
import shutil

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


# ──────────────────────────────────────────────
# DATA MODELS
# ──────────────────────────────────────────────

@dataclass
class ModelRecord:
    """Immutable record of a trained model."""
    model_id: str
    model_name: str
    version: str
    base_model: str
    training_data_version: str
    training_config: Dict[str, Any]
    training_script_hash: str
    model_path: str
    tokenizer_path: str
    eval_metrics: Dict[str, float]
    regression_metrics: Dict[str, float]
    latency_p50_ms: float
    latency_p99_ms: float
    stage: str  # "dev" | "staging" | "canary" | "production" | "retired"
    created_at: str
    created_by: str
    tags: Dict[str, str]
    notes: str
    
    @classmethod
    def create(
        cls,
        model_name: str,
        version: str,
        base_model: str,
        training_data_version: str,
        training_config: Dict,
        training_script_hash: str,
        model_path: str,
        tokenizer_path: str,
        eval_metrics: Dict,
        created_by: str,
        regression_metrics: Optional[Dict] = None,
        latency_p50_ms: float = 0.0,
        latency_p99_ms: float = 0.0,
        tags: Optional[Dict] = None,
        notes: str = "",
    ) -> "ModelRecord":
        """Create a new model record with auto-generated ID."""
        
        # Create a content-hash ID for uniqueness
        content = f"{model_name}-{version}-{training_data_version}-{training_script_hash}-{datetime.now().isoformat()}"
        model_id = f"{model_name}-{version}-{hashlib.md5(content.encode()).hexdigest()[:8]}"
        
        return cls(
            model_id=model_id,
            model_name=model_name,
            version=version,
            base_model=base_model,
            training_data_version=training_data_version,
            training_config=training_config,
            training_script_hash=training_script_hash,
            model_path=model_path,
            tokenizer_path=tokenizer_path,
            eval_metrics=eval_metrics,
            regression_metrics=regression_metrics or {},
            latency_p50_ms=latency_p50_ms,
            latency_p99_ms=latency_p99_ms,
            stage="dev",
            created_at=datetime.now().isoformat(),
            created_by=created_by,
            tags=tags or {},
            notes=notes,
        )


@dataclass
class Experiment:
    """Group of model runs testing a hypothesis."""
    experiment_id: str
    hypothesis: str
    runs: List[str]  # List of model_ids
    winner: Optional[str]
    conclusion: str
    created_at: str
    tags: Dict[str, str]


# ──────────────────────────────────────────────
# MODEL REGISTRY
# ──────────────────────────────────────────────

class ModelRegistry:
    """
    SQLite-backed model registry.
    
    Key design decisions:
    - Model records are IMMUTABLE (never modified after creation)
    - Stage transitions create AUDIT LOG entries
    - All operations are ATOMIC (SQLite transactions)
    - Registry is self-contained (single file)
    """
    
    def __init__(self, db_path: str = "model_registry.db", backup_dir: str = "./registry_backups"):
        self.db_path = db_path
        self.backup_dir = Path(backup_dir)
        self.backup_dir.mkdir(parents=True, exist_ok=True)
        self._init_db()
    
    def _init_db(self):
        """Initialize database schema."""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS models (
                    model_id TEXT PRIMARY KEY,
                    model_name TEXT NOT NULL,
                    version TEXT NOT NULL,
                    base_model TEXT NOT NULL,
                    training_data_version TEXT NOT NULL,
                    training_config TEXT NOT NULL,  -- JSON
                    training_script_hash TEXT NOT NULL,
                    model_path TEXT NOT NULL,
                    tokenizer_path TEXT NOT NULL,
                    eval_metrics TEXT NOT NULL,  -- JSON
                    regression_metrics TEXT DEFAULT '{}',  -- JSON
                    latency_p50_ms REAL DEFAULT 0,
                    latency_p99_ms REAL DEFAULT 0,
                    stage TEXT NOT NULL DEFAULT 'dev',
                    created_at TEXT NOT NULL,
                    created_by TEXT NOT NULL,
                    tags TEXT DEFAULT '{}',  -- JSON
                    notes TEXT DEFAULT ''
                )
            """)
            
            conn.execute("""
                CREATE TABLE IF NOT EXISTS stage_transitions (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    model_id TEXT NOT NULL,
                    from_stage TEXT NOT NULL,
                    to_stage TEXT NOT NULL,
                    changed_by TEXT NOT NULL,
                    changed_at TEXT NOT NULL,
                    reason TEXT DEFAULT '',
                    FOREIGN KEY (model_id) REFERENCES models(model_id)
                )
            """)
            
            conn.execute("""
                CREATE TABLE IF NOT EXISTS experiments (
                    experiment_id TEXT PRIMARY KEY,
                    hypothesis TEXT NOT NULL,
                    runs TEXT NOT NULL,  -- JSON list of model_ids
                    winner TEXT,
                    conclusion TEXT DEFAULT '',
                    created_at TEXT NOT NULL,
                    tags TEXT DEFAULT '{}'
                )
            """)
            
            # Index for fast lookups
            conn.execute("""
                CREATE INDEX IF NOT EXISTS idx_models_stage ON models(stage)
            """)
            conn.execute("""
                CREATE INDEX IF NOT EXISTS idx_models_name ON models(model_name)
            """)
        
        logger.info(f"Model registry initialized at {self.db_path}")
    
    def _atomic_backup(self):
        """Create atomic backup before destructive operations."""
        if Path(self.db_path).exists():
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            backup_path = self.backup_dir / f"registry_backup_{timestamp}.db"
            shutil.copy2(self.db_path, backup_path)
            # Keep only last 10 backups
            backups = sorted(self.backup_dir.glob("registry_backup_*.db"))
            for old_backup in backups[:-10]:
                old_backup.unlink()
    
    # ─── Register ───
    
    def register_model(self, record: ModelRecord) -> str:
        """Register a new model (IMMUTABLE — cannot be modified after)."""
        self._atomic_backup()
        
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                INSERT INTO models (
                    model_id, model_name, version, base_model,
                    training_data_version, training_config, training_script_hash,
                    model_path, tokenizer_path, eval_metrics, regression_metrics,
                    latency_p50_ms, latency_p99_ms, stage,
                    created_at, created_by, tags, notes
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                record.model_id,
                record.model_name,
                record.version,
                record.base_model,
                record.training_data_version,
                json.dumps(record.training_config),
                record.training_script_hash,
                record.model_path,
                record.tokenizer_path,
                json.dumps(record.eval_metrics),
                json.dumps(record.regression_metrics),
                record.latency_p50_ms,
                record.latency_p99_ms,
                record.stage,
                record.created_at,
                record.created_by,
                json.dumps(record.tags),
                record.notes,
            ))
            
            # Log initial creation as a stage transition
            conn.execute("""
                INSERT INTO stage_transitions (model_id, from_stage, to_stage, changed_by, changed_at, reason)
                VALUES (?, 'created', ?, ?, ?, 'model registered')
            """, (record.model_id, record.stage, record.created_by, record.created_at))
        
        logger.info(f"Registered model: {record.model_id}")
        return record.model_id
    
    # ─── Stage Management ───
    
    def _validate_stage_transition(self, from_stage: str, to_stage: str) -> bool:
        """Validate stage transitions (forward-only)."""
        valid_transitions = {
            "created": ["dev"],
            "dev": ["staging"],
            "staging": ["canary"],
            "canary": ["production", "staging"],  # Can go back to staging if canary fails
            "production": ["retired"],
            "retired": [],  # Cannot transition from retired
        }
        allowed = valid_transitions.get(from_stage, [])
        if to_stage not in allowed:
            logger.error(
                f"Invalid stage transition: {from_stage} → {to_stage}. "
                f"Allowed: {allowed}"
            )
            return False
        return True
    
    def transition_stage(
        self,
        model_id: str,
        to_stage: str,
        changed_by: str,
        reason: str = "",
    ) -> bool:
        """Transition a model to a new stage (with validation)."""
        self._atomic_backup()
        
        with sqlite3.connect(self.db_path) as conn:
            # Get current stage
            cursor = conn.execute(
                "SELECT stage FROM models WHERE model_id = ?", (model_id,)
            )
            row = cursor.fetchone()
            if not row:
                logger.error(f"Model not found: {model_id}")
                return False
            
            from_stage = row[0]
            
            # Validate transition
            if not self._validate_stage_transition(from_stage, to_stage):
                return False
            
            # Atomic update + audit log
            conn.execute(
                "UPDATE models SET stage = ? WHERE model_id = ?",
                (to_stage, model_id),
            )
            conn.execute("""
                INSERT INTO stage_transitions (model_id, from_stage, to_stage, changed_by, changed_at, reason)
                VALUES (?, ?, ?, ?, ?, ?)
            """, (
                model_id, from_stage, to_stage,
                changed_by, datetime.now().isoformat(), reason,
            ))
        
        logger.info(f"Model {model_id}: {from_stage} → {to_stage} (by {changed_by}: {reason})")
        return True
    
    def set_production_model(
        self,
        model_id: str,
        changed_by: str,
        reason: str = "Promoted to production",
    ) -> bool:
        """
        Set a model to production stage.
        
        This automatically:
        1. Moves current production model to retired
        2. Sets the new model to production
        3. Logs both transitions
        """
        self._atomic_backup()
        
        with sqlite3.connect(self.db_path) as conn:
            # Get current production model
            cursor = conn.execute(
                "SELECT model_id FROM models WHERE stage = 'production'"
            )
            current_prod = cursor.fetchone()
            
            # Verify new model exists and is in canary stage
            cursor = conn.execute(
                "SELECT stage FROM models WHERE model_id = ?", (model_id,)
            )
            row = cursor.fetchone()
            if not row:
                logger.error(f"Model not found: {model_id}")
                return False
            
            if row[0] not in ("canary", "staging"):
                logger.error(
                    f"Model {model_id} is in '{row[0]}' stage. "
                    f"Must be 'canary' or 'staging' before production."
                )
                return False
            
            # Retire current production model
            if current_prod:
                old_id = current_prod[0]
                conn.execute(
                    "UPDATE models SET stage = 'retired' WHERE model_id = ?",
                    (old_id,),
                )
                conn.execute("""
                    INSERT INTO stage_transitions (model_id, from_stage, to_stage, changed_by, changed_at, reason)
                    VALUES (?, 'production', 'retired', ?, ?, 'replaced by new version')
                """, (old_id, changed_by, datetime.now().isoformat()))
                logger.info(f"Retired previous production model: {old_id}")
            
            # Set new production model
            conn.execute(
                "UPDATE models SET stage = 'production' WHERE model_id = ?",
                (model_id,),
            )
            conn.execute("""
                INSERT INTO stage_transitions (model_id, from_stage, to_stage, changed_by, changed_at, reason)
                VALUES (?, ?, 'production', ?, ?, ?)
            """, (model_id, row[0], changed_by, datetime.now().isoformat(), reason))
        
        logger.info(f"Production model set to: {model_id}")
        return True
    
    def rollback_to(
        self,
        model_id: str,
        changed_by: str,
        reason: str = "Rollback requested",
    ) -> bool:
        """
        Roll back to a specific model.
        
        This is a SET operation — it makes the specified model production.
        The current production model is retired.
        """
        # Rollback is just a special case of set_production
        return self.set_production_model(model_id, changed_by, f"ROLLBACK: {reason}")
    
    # ─── Queries ───
    
    def get_model(self, model_id: str) -> Optional[Dict]:
        """Get a model record by ID."""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                "SELECT * FROM models WHERE model_id = ?", (model_id,)
            )
            row = cursor.fetchone()
            if not row:
                return None
            return self._row_to_dict(row)
    
    def get_production_model(self, model_name: Optional[str] = None) -> Optional[Dict]:
        """Get the current production model (optionally filtered by name)."""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            if model_name:
                cursor = conn.execute(
                    "SELECT * FROM models WHERE stage = 'production' AND model_name = ?",
                    (model_name,),
                )
            else:
                cursor = conn.execute(
                    "SELECT * FROM models WHERE stage = 'production'"
                )
            row = cursor.fetchone()
            if not row:
                return None
            return self._row_to_dict(row)
    
    def search_models(
        self,
        model_name: Optional[str] = None,
        stage: Optional[str] = None,
        tag_key: Optional[str] = None,
        tag_value: Optional[str] = None,
        min_rouge: Optional[float] = None,
        limit: int = 10,
    ) -> List[Dict]:
        """Search models by criteria."""
        query = "SELECT * FROM models WHERE 1=1"
        params = []
        
        if model_name:
            query += " AND model_name = ?"
            params.append(model_name)
        if stage:
            query += " AND stage = ?"
            params.append(stage)
        if tag_key and tag_value:
            # JSON tag search (SQLite JSON support)
            query += " AND json_extract(tags, '$.' || ?) = ?"
            params.append(tag_key)
            params.append(tag_value)
        if min_rouge is not None:
            query += " AND json_extract(eval_metrics, '$.rougeL') >= ?"
            params.append(min_rouge)
        
        query += " ORDER BY created_at DESC LIMIT ?"
        params.append(limit)
        
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(query, params)
            return [self._row_to_dict(row) for row in cursor.fetchall()]
    
    def get_model_history(self, model_id: str) -> List[Dict]:
        """Get stage transition history for a model."""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute("""
                SELECT * FROM stage_transitions
                WHERE model_id = ?
                ORDER BY changed_at ASC
            """, (model_id,))
            return [dict(row) for row in cursor.fetchall()]
    
    def _row_to_dict(self, row: sqlite3.Row) -> Dict:
        """Convert SQLite row to dict with parsed JSON fields."""
        d = dict(row)
        for json_field in ['training_config', 'eval_metrics', 'regression_metrics', 'tags']:
            if isinstance(d.get(json_field), str):
                d[json_field] = json.loads(d[json_field])
        return d
    
    # ─── Comparison ───
    
    def compare_models(self, model_ids: List[str]) -> Dict:
        """Compare multiple models side by side."""
        models = [self.get_model(mid) for mid in model_ids]
        models = [m for m in models if m is not None]
        
        if len(models) < 2:
            return {"error": "Need at least 2 models to compare"}
        
        comparison = {
            "models": [],
            "metrics_comparison": {},
            "best_overall": None,
        }
        
        # Collect all metric names
        all_metrics = set()
        for m in models:
            all_metrics.update(m['eval_metrics'].keys())
            all_metrics.update(m['regression_metrics'].keys())
        
        # Build comparison per metric
        for metric in sorted(all_metrics):
            values = []
            for m in models:
                val = m['eval_metrics'].get(metric) or m['regression_metrics'].get(metric)
                if val is not None:
                    values.append((m['model_id'], val))
            
            if values:
                values.sort(key=lambda x: x[1], reverse=True)
                comparison['metrics_comparison'][metric] = values
        
        # Simple ranking: count how many metrics each model wins
        model_wins = {m['model_id']: 0 for m in models}
        for metric, vals in comparison['metrics_comparison'].items():
            if vals:
                model_wins[vals[0][0]] = model_wins.get(vals[0][0], 0) + 1
        
        comparison['best_overall'] = max(model_wins, key=model_wins.get)
        comparison['models'] = [
            {
                'model_id': m['model_id'],
                'model_name': m['model_name'],
                'version': m['version'],
                'stage': m['stage'],
                'created_at': m['created_at'],
                'eval_metrics': m['eval_metrics'],
            }
            for m in models
        ]
        
        return comparison
    
    # ─── Experiments ───
    
    def create_experiment(
        self,
        hypothesis: str,
        tags: Optional[Dict] = None,
    ) -> str:
        """Create a new experiment."""
        experiment_id = f"exp-{uuid.uuid4().hex[:8]}"
        
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                INSERT INTO experiments (experiment_id, hypothesis, runs, conclusion, created_at, tags)
                VALUES (?, ?, '[]', '', ?, ?)
            """, (
                experiment_id,
                hypothesis,
                datetime.now().isoformat(),
                json.dumps(tags or {}),
            ))
        
        logger.info(f"Created experiment: {experiment_id}: {hypothesis[:60]}...")
        return experiment_id
    
    def add_run_to_experiment(self, experiment_id: str, model_id: str) -> bool:
        """Add a model run to an experiment."""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.execute(
                "SELECT runs FROM experiments WHERE experiment_id = ?",
                (experiment_id,),
            )
            row = cursor.fetchone()
            if not row:
                logger.error(f"Experiment not found: {experiment_id}")
                return False
            
            runs = json.loads(row[0])
            if model_id not in runs:
                runs.append(model_id)
                conn.execute(
                    "UPDATE experiments SET runs = ? WHERE experiment_id = ?",
                    (json.dumps(runs), experiment_id),
                )
        
        logger.info(f"Added model {model_id} to experiment {experiment_id}")
        return True
    
    def conclude_experiment(
        self,
        experiment_id: str,
        winner: str,
        conclusion: str,
    ) -> bool:
        """Mark an experiment as concluded with a winner and conclusion."""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                UPDATE experiments
                SET winner = ?, conclusion = ?
                WHERE experiment_id = ?
            """, (winner, conclusion, experiment_id))
        
        logger.info(f"Experiment {experiment_id} concluded. Winner: {winner}")
        return True


# ──────────────────────────────────────────────
# USAGE EXAMPLE
# ──────────────────────────────────────────────

if __name__ == "__main__":
    """
    Demonstrate complete model lifecycle using the registry.
    """
    
    registry = ModelRegistry("legal_models.db")
    
    # 1. Register models as they're trained
    model_v1 = ModelRecord.create(
        model_name="legal-summarization",
        version="v1",
        base_model="meta-llama/Llama-2-7b-hf",
        training_data_version="legal-corpus-v1",
        training_config={
            "learning_rate": 2e-4,
            "batch_size": 16,
            "epochs": 3,
            "lora_rank": 16,
        },
        training_script_hash="a3f8c2e",
        model_path="s3://models/legal-summarization/v1/",
        tokenizer_path="s3://models/legal-summarization/v1/tokenizer/",
        eval_metrics={"rougeL": 0.42, "bertscore": 0.81},
        regression_metrics={"code_gen": 0.63, "safety": 0.002},
        created_by="Partha",
        tags={"purpose": "summarization", "dataset": "legal-v1"},
        notes="First attempt, moderate quality",
    )
    registry.register_model(model_v1)
    
    model_v2 = ModelRecord.create(
        model_name="legal-summarization",
        version="v2",
        base_model="meta-llama/Llama-2-7b-hf",
        training_data_version="legal-corpus-v2",
        training_config={
            "learning_rate": 1e-4,
            "batch_size": 16,
            "epochs": 5,
            "lora_rank": 32,
        },
        training_script_hash="b7d1f4a",
        model_path="s3://models/legal-summarization/v2/",
        tokenizer_path="s3://models/legal-summarization/v2/tokenizer/",
        eval_metrics={"rougeL": 0.47, "bertscore": 0.85},
        regression_metrics={"code_gen": 0.61, "safety": 0.003},
        created_by="Partha",
        tags={"purpose": "summarization", "dataset": "legal-v2"},
        notes="Higher rank + more epochs, improved ROUGE",
    )
    registry.register_model(model_v2)
    
    # 2. Promote through stages
    registry.transition_stage(model_v1.model_id, "staging", "Partha", "Ready for integration tests")
    registry.transition_stage(model_v1.model_id, "canary", "Partha", "Testing with 5% traffic")
    registry.set_production_model(model_v1.model_id, "Partha", "First production model")
    
    # 3. When v2 is ready, promote and auto-retire v1
    registry.transition_stage(model_v2.model_id, "staging", "Partha", "Better metrics")
    registry.transition_stage(model_v2.model_id, "canary", "Partha", "Canary testing v2")
    registry.set_production_model(model_v2.model_id, "Partha", "v2 outperforms v1 in all metrics")
    
    # 4. Compare models
    comparison = registry.compare_models([model_v1.model_id, model_v2.model_id])
    print(f"Best model: {comparison['best_overall']}")
    
    # 5. Query current production model
    prod = registry.get_production_model("legal-summarization")
    if prod:
        print(f"Production model: {prod['model_id']} (v{prod['version']})")
    
    # 6. Rollback if needed
    # registry.rollback_to(model_v1.model_id, "Partha", "v2 has safety regression")
    
    print("\nModel registry demonstration complete!")
    print(f"Database: {registry.db_path}")
    print(f"Backups: {registry.backup_dir}")
```

---

### 🔍 Critical Questions

1. **Immutable records**: The registry treats model records as IMMUTABLE — once created, they're never modified. Stage transitions create AUDIT LOG entries instead of modifying the record. Why is immutability important for a model registry? What problems does it prevent?

2. **Backup strategy**: The registry creates a backup before every write operation. But what if the backup ITSELF gets corrupted? How would you verify backup integrity? At what point does the backup strategy become MORE costly than the data loss risk?

3. **Stage transition rules**: The registry enforces `dev → staging → canary → production → retired`. But what if an emergency requires skipping stages (e.g., deploy directly to production)? Should the registry allow this with a flag? Or should emergencies use a DIFFERENT process?

---

## ✅ Good Output Examples

### What a Well-Managed Model Registry Looks Like

```
=== Model Registry Status ===
Database: legal_models.db (42 models, 128 stage transitions)

=== Active Models ===
┌──────────────────────────────┬──────────┬────────┬───────────┬──────────────┐
│ Model ID                     │ Version  │ Stage  │ Rouge-L  │ Deployed At  │
├──────────────────────────────┼──────────┼────────┼───────────┼──────────────┤
│ legal-summarization-v2-a3f8  │ v2       │ PROD   │ 0.4712   │ 2024-04-15   │
│ legal-summarization-v3-b7d1  │ v3       │ CANARY │ 0.4891   │ 2024-04-20   │
│ legal-func-call-v1-c4e2      │ v1       │ STAG   │ 0.9203   │ 2024-04-18   │
│ legal-func-call-v2-d5f3      │ v2       │ DEV    │ 0.9417   │ 2024-04-21   │
└──────────────────────────────┴──────────┴────────┴───────────┴──────────────┘

=== Recent Stage Transitions ===
2024-04-21 14:30: Partha promoted legal-func-call-v2-d5f3 → STAGING
2024-04-20 09:15: Partha promoted legal-summarization-v3-b7d1 → CANARY
2024-04-15 16:00: Partha set legal-summarization-v2-a3f8 → PRODUCTION
  (previous: legal-summarization-v1-c1a2 → RETIRED)

=== Rollback Available ===
Previous production: legal-summarization-v1-c1a2 (retired 2024-04-15)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Models as Files, Not Records

**Problem**: Models are stored as files with names like `final_model.pt`, `model_v2_final.pt`, `model_v2_actual_final.pt`. No metadata, no lineage, no lifecycle.

**Fix**: Every model gets a UNIQUE ID and a DATABASE RECORD. The filename is irrelevant — the record contains the path.

### Antipattern 2: Overwriting Models In Place

**Problem**: Training saves to the same path every time: `./models/ft-model/`. Deploy uses this path. When a new model is trained, it overwrites the old one. The old model is lost forever.

**Fix**: EVERY training run creates a NEW path with a UNIQUE directory. Immutable deployments — once written, never overwritten.

### Antipattern 3: Manual Stage Promotion

**Problem**: "Just copy the model to the production folder." No audit trail, no validation, no rollback capability.

**Fix**: Stage promotions go through the REGISTRY, which validates transitions, creates audit logs, and enables rollback.

### Antipattern 4: No Experiment Tracking

**Problem**: 15 models exist but nobody knows WHY each was trained — what hypothesis was being tested, what the result was, what was learned.

**Fix**: Every model is associated with an EXPERIMENT that captures the hypothesis and conclusion. The learning outlives any single model version.

### Common Registry Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| Database corruption | Registry can't open | Regular backups, integrity checks |
| Stale registries | Models referenced but deleted | Validate paths on registration |
| Permission issues | Wrong team deploys wrong model | Role-based access to stage transitions |
| Schema drift | New metadata fields not in old records | Version the registry format, migrations |
| Backup overload | Too many backups, disk full | Retention policy (keep N backups) |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Rollback Automation (30 min)

**Goal**: Extend the registry to support AUTOMATED rollback based on metric thresholds.

**Task**: Add a `monitor_and_rollback()` method that:
1. Accepts a metric name and threshold
2. If the current production model's metric drops below threshold
3. Automatically rolls back to the PREVIOUS production model
4. Logs the incident
5. Sends an alert

**Implementation sketch:**
```python
def monitor_and_rollback(
    self,
    metric_name: str,
    min_threshold: float,
    alert_callback: Optional[Callable] = None,
) -> bool:
    # Get current production model
    # Check if metric_name exists in eval_metrics or regression_metrics
    # If metric < threshold:
    #   Find previous production model
    #   Rollback to it
    #   Log incident
    #   Call alert_callback if provided
    pass
```

**Questions:**
1. How do you determine the "previous" production model? By stage transition history?
2. What if the previous production model ALSO fails the threshold?
3. How do you prevent ROLLBACK LOOPS (rollback A → B, rollback B → A, repeat)?

---

### Drill 2: Design a Multi-Model Routing System (30 min)

**Goal**: Design a system that routes requests to the RIGHT model based on the query.

**Scenario**: You have 3 fine-tuned models:
- `legal-summarization-v2` (production): For legal document summarization
- `func-calling-v3` (production): For function-calling tasks
- `general-chat-v1` (canary): For general conversation

**Task**: Design a routing system that:
1. Detects which model should handle a query
2. Routes to the right model
3. Falls back gracefully if the chosen model fails
4. Logs routing decisions for analysis

**Questions to answer:**
1. How does the router classify queries? (Classifier model? Keyword matching? User-provided hint?)
2. What happens if the query matches MULTIPLE models? (e.g., "summarize this legal document and email it to Alice" — both summarization AND function-calling)
3. How do you evaluate the router's accuracy separately from the models' accuracy?

---

### Drill 3: Build a Metrics Dashboard (45 min)

**Goal**: Create a dashboard that shows model comparison over time.

**Task**: Extend the registry with a dashboard generator:

```python
def generate_dashboard(self, model_name: str) -> Dict:
    """
    Generate a dashboard for a model family.
    
    Returns:
    - Timeline of all models with key metrics
    - Best/worst model for each metric
    - Stage transition timeline
    - Metric trends over model versions
    """
    pass
```

**Requirements:**
- Show ROUGE-L trend across v1, v2, v3, etc.
- Show regression metrics (safety, code gen) as OVERLAY — did these improve or degrade?
- Show stage transition duration (how long did each model stay in canary?)
- Flag any model where regression metrics degraded more than 10%

---

## 🚦 Gate Check

Before moving to File 08 (Deployment of Fine-Tuned Models), verify you can:

1. **Design a model registry schema** — For a team of 5 ML engineers running 3 training experiments per week, design: table structure, metadata fields, stage transitions, audit trail

2. **Plan a rollback** — Given: production model v3 has a safety regression detected at 3 PM Friday. v2 was the previous production model. v1 was retired. Design the rollback process and verify the design handles: model weights, tokenizer, config, system prompt, and RAG settings

3. **Design stage transitions** — For a high-risk model (medical diagnosis), design a 4-stage gating process with: automated eval checks at each gate, manual approval for canary→production, rollback criteria

4. **Implement model comparison** — Given these 3 models:
   - v1: ROUGE=0.42, Safety=0.2%, Latency=1.2s
   - v2: ROUGE=0.47, Safety=0.3%, Latency=1.4s
   - v3: ROUGE=0.49, Safety=4.1%, Latency=1.8s
   
   Which would you deploy? What additional information would you want?

5. **Design experiment tracking** — You want to test: "Does increasing LoRA rank from 16 to 32 improve summarization quality?" Design: number of runs, controlled variables, how you'll measure "improvement," how you'll conclude the experiment

---

## 📚 Resources

### Model Registry Tools
- **[MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)** — Industry standard, integrates with MLflow tracking
- **[Hugging Face Hub](https://huggingface.co/docs/hub/en/models)** — De facto standard for open model distribution, includes versioning
- **[DVC (Data Version Control)](https://dvc.org/)** — Version control for ML experiments and models
- **[Weights & Biases (WandB)](https://wandb.ai/site/model-management)** — Managed experiment tracking and model registry

### Deployment Patterns
- **[Blue/Green Deployments](https://martinfowler.com/bliki/BlueGreenDeployment.html)** — Zero-downtime deployment with instant rollback
- **[Canary Deployments](https://martinfowler.com/bliki/CanaryRelease.html)** — Gradual traffic shifting for risk reduction
- **[Kubernetes Model Serving](https://kubernetes.io/blog/2022/08/23/machine-learning-model-serving-kubernetes/)** — Production-grade model deployment

### Phase Cross-References
- **Phase 10 (Deployment)** → The model registry is the SOURCE OF TRUTH for deployment. Phase 10 deployment picks models from the registry.
- **Phase 7 (Evals)** → Registry stores eval results. Your Phase 7 evaluation platform should automatically register results in the registry.
- **Phase 8 (Guardrails)** → The registry should track guardrail versions alongside model versions. A model+guardrail pair is a deployable unit.

> **Next up: File 08 — Deployment of Fine-Tuned Models.** You have registered models. Now actually DEPLOY them — serving infrastructure, quantization for production, A/B testing, scaling, and cost optimization. This is where fine-tuned models face real users.
