# Production Prompt Management — Versioning, Monitoring, Rollback

## 🎯 Purpose & Goals

This file is different. I'm not going to give you the "best way" to manage prompts in production. Instead, I'm going to describe a series of production incidents. Each one reveals a problem. **You design the solution.** Then we compare.

**By the end of this file, you will have designed a prompt management system from scratch — by solving the problems that real teams face.**

**⏱ Time Budget:** 2 hours

---

## 📖 The Problems — Incident Reports

### Incident 1: "Who Changed the Prompt?"

**The situation:** Your team has 3 engineers. Each one modifies the system prompt for the customer support bot. No one tracks changes. One day, CSAT drops 15%. Someone changed the prompt yesterday. No one knows who, what they changed, or why.

> **🤔 YOUR TURN:** You walk into this mess. You need to figure out what changed and revert it. What's your first move? What systems would prevent this? Don't say "use git" — that's obvious. Think about what SPECIFICALLY needs to be tracked beyond just file history.

**Try designing a prompt version format.** What fields would a `PromptVersion` record need to capture the full context of a change?

<details>
<summary>Think about it first, then expand</summary>

A prompt version needs more than just the text. Consider:
- Who made the change? (author)
- Why? (linked to a ticket/issue)
- What metrics were expected to improve? (hypothesis)
- What was the previous version's performance?
- What's the new version's performance?
- Was it A/B tested before full deploy?
- What's the rollback procedure?
</details>

---

### Incident 2: "It Worked in Dev"

Your teammate improves the prompt. Tests it on 10 examples. Looks great. Deploys to production.

The next day, support tickets spike. The new prompt handles the 10 test examples perfectly but fails on the 1,000 production variations it didn't see.

> **🤔 YOUR TURN:** Why do prompts that look perfect on test data fail in production? List at least 3 reasons. Then design a "pre-production validation" step that would catch these before full rollout.

**Now, here's a twist:** Your teammate's change was actually good — it fixed a real issue. But it broke something unrelated. How do you design a system that catches UNINTENDED side effects of prompt changes?

<details>
<summary>Compare with a production approach</summary>

Common failure modes:
1. Test data is too narrow (10 happy paths, no edge cases)
2. Production traffic has distribution shift (different topics, tones, lengths)
3. The metric measured in dev doesn't match what matters in prod (accuracy vs. CSAT)

Production systems catch this with:
- Shadow evaluation (run new prompt alongside old, compare silently)
- Gradual rollout (1% → 5% → 25% → 100%)
- Automated regression test suite (200+ diverse test cases)
- Metric dashboards with anomaly detection

But these are expensive to build. What's the MINIMUM viable version?
</details>

---

### Incident 3: "The Silent Regression"

You change a prompt to improve accuracy on billing questions. Accuracy goes up 5%. Great!

Three weeks later, you notice that "refund" requests have doubled. Turns out, your "improved" prompt makes the model overly agreeable — it approves refunds the old prompt would have escalated.

**No metric caught this** because your dashboards don't track "refund rate" as a prompt quality metric.

> **🤔 YOUR TURN:** 
> 1. List 5 metrics that matter for a customer support prompt that AREN'T "accuracy"
> 2. For each metric, design a way to TRACK it (what data, what pipeline, what alert threshold)
> 3. Now the hard one: how do you know which metrics to track BEFORE an incident happens? You can't track everything.

<details>
<summary>Reflection</summary>

This is the "unknown unknowns" problem. You can't put dashboards on everything. The senior approach:

1. Track METADATA metrics on every prompt change (response length, refusal rate, specific word usage, sentiment)
2. When a change goes out, compare ALL metadata metrics before/after — not just the target metric
3. Accept that some regressions won't be caught. Build fast rollback instead.
4. Have a human-review sampling pipeline (random 1% of responses are manually reviewed)

The real skill isn't "track everything" — it's "knowing what to track" gained FROM experience with past incidents. Document EVERY incident and add the relevant metric.
</details>

---

## 💻 Your Turn: Design a Prompt Management System

Based on the 3 incidents above, design a system. Start with the data model, then the workflow, then the integration points.

### Step 1: Design the Data Model

**Don't read the code below yet.** First, sketch on paper or in your head: what fields does a `Prompt` record need? What about a `PromptVersion`? What about a `Deployment`?

<details>
<summary>Now compare with a production data model</summary>

```python
"""
A prompt management system tracks:
1. The prompt itself (the text)
2. Every version of it (who, when, why)
3. Every deployment (which version, to which environment, what % of traffic)
4. Metrics associated with each deployment (accuracy, cost, latency, CSAT)
5. Rollback capability (one command to revert)
"""

from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field
from enum import Enum


class Environment(str, Enum):
    DEVELOPMENT = "development"
    STAGING = "staging"
    PRODUCTION = "production"
    SHADOW = "shadow"  # Running alongside production, not serving users


class DeploymentStrategy(str, Enum):
    FULL = "full"                    # 100% traffic immediately
    GRADUAL = "gradual"              # Percentage ramp-up
    SHADOW = "shadow"                # Log-only, no user-facing
    A_B_TEST = "a_b_test"            # Controlled experiment


class Prompt(BaseModel):
    """A prompt template with metadata."""
    id: str
    name: str                        # e.g., "customer-support-v1"
    task_type: str                   # "classification", "generation", "extraction"
    model: str                       # Which model this prompt is for
    template: str                    # The prompt text with {placeholders}
    
    # Metadata
    owner: str                       # Team/person responsible
    description: str                 # What this prompt does
    tags: list[str] = []             # For search/filtering
    
    created_at: datetime = Field(default_factory=datetime.now)
    updated_at: datetime = Field(default_factory=datetime.now)
    
    # Current active version
    active_version_id: Optional[str] = None


class PromptVersion(BaseModel):
    """
    A specific version of a prompt.
    
    This is what gets REVIEWED, DEPLOYED, and ROLLED BACK.
    Every change creates a new version.
    """
    id: str
    prompt_id: str
    version_number: int              # 1, 2, 3...
    
    # The actual content
    content: str                     # Full prompt text
    
    # Change context
    author: str
    change_reason: str               # Link to ticket/issue
    change_description: str          # What changed and why
    
    # Pre-deployment validation
    test_set_accuracy: Optional[float] = None
    regression_test_passed: bool = False
    safety_review_passed: bool = False
    reviewed_by: Optional[str] = None
    
    # Parent version (for rollback tracking)
    parent_version_id: Optional[str] = None
    
    created_at: datetime = Field(default_factory=datetime.now)
    hash: str = ""                   # Content hash for integrity


class PromptDeployment(BaseModel):
    """
    A deployment of a specific prompt version.
    
    Tracks the full lifecycle of a prompt in production.
    """
    id: str
    prompt_id: str
    version_id: str
    environment: Environment
    strategy: DeploymentStrategy
    
    # Gradual rollout fields
    target_percentage: float = 100.0  # For gradual rollouts
    current_percentage: float = 0.0
    
    # Timing
    deployed_at: datetime = Field(default_factory=datetime.now)
    rollback_at: Optional[datetime] = None
    rollback_reason: Optional[str] = None
    
    # Metrics at deployment time
    metrics_snapshot: dict = {}       # Accuracy, latency, cost, etc.
    
    # Status
    status: str = "active"           # active, rolled_back, superseded
```

**Pause and compare with your design:**
- What did you miss?
- What did you include that I didn't?
- Which fields would be MOST critical for incident response?
- Which fields are "nice to have" vs. "essential"?
</details>

---

### Step 2: Design the Workflow

**Before reading — sketch the workflow for making a prompt change:**

1. Discover a problem → 
2. Edit the prompt → 
3. ??? → 
4. Deploy → 
5. ??? → 
6. Done (or rollback)

**What are the ??? steps? Who approves what? What gates must a change pass?**

<details>
<summary>One possible workflow (not the only one)</summary>

```
DISCOVERY → EDIT → VALIDATE → REVIEW → DEPLOY → MONITOR → COMPLETE/ROLLBACK

1. DISCOVERY
   Someone identifies a prompt issue (metric drop, user feedback, new requirement)
   → Creates a ticket describing the problem and desired outcome

2. EDIT
   Author creates a new PromptVersion from the current active version
   → Makes changes in a development environment
   → Runs on 10 test cases → self-review

3. VALIDATE
   Automated test suite runs (100+ test cases)
   → Regression check (does it break anything that used to work?)
   → Metric comparison (previous version vs new version)
   → Safety check (red-team tests)

4. REVIEW
   Another engineer reviews the change
   → Checks: does the prompt accurately reflect the intent?
   → Checks: are there edge cases not handled?
   → Approves or requests changes

5. DEPLOY
   Choose deployment strategy:
   - SHADOW: Run alongside production, log both outputs, compare silently
   - GRADUAL: 1% traffic → monitor 1 hour → 10% → monitor → 50% → monitor → 100%
   - A/B: Random 50/50 split, compare metrics for 24 hours

6. MONITOR
   Compare metrics between old and new versions:
   - Primary metric: Did it fix the problem?
   - Secondary metrics: Did anything else change? (response length, refusals, etc.)
   - Anomaly detection: Spike in any tracked metric?

7. COMPLETE or ROLLBACK
   If metrics are stable and improved → mark deployment as complete
   If regression detected → ONE COMMAND rollback to previous version
   → Document what happened for future reference
```

**Now questions for you:**
1. Which of these steps could be automated? Which require human judgment?
2. In a startup with 2 engineers, which steps would you skip? What risk does that create?
3. In a regulated industry (fintech, healthcare), which steps are mandatory?
4. The workflow above has 7 steps. How long should each take? If the whole process takes 3 days, is that too slow? Too fast?
</details>

---

### Step 3: Design the Rollback

An incident happens. You need to rollback a prompt change. You have 2 minutes before 10,000 users see broken responses.

> **🤔 YOUR TURN:** Design the rollback mechanism. Don't just say "revert the git commit" — that takes too long. What does a FAST rollback look like?

<details>
<summary>Compare approaches</summary>

**Approach 1: Git revert (slow, ~5-10 minutes)**
```
git revert HEAD
git push
CI/CD pipeline deploys
→ 5-10 minutes of broken responses
```

**Approach 2: Feature flag (fast, ~10 seconds)**
```
# Prompt is stored in a database/config service
# Deployment points to a version ID
# Rollback = change version ID in database
# No code deploy needed

rollback(prompt_name="customer-support", version_id="v42")
# → Next request uses v42 instead of v43
```

**Approach 3: Canary + auto-rollback (fastest, ~0 seconds for users)**
```
# If error rate > 5% above baseline, auto-rollback
deploy(prompt, strategy="gradual", auto_rollback=True)
# → System handles the rollback before most users even see it

async def monitor_deployment(deployment):
    baseline_error_rate = get_baseline_error_rate()
    
    while deployment.current_percentage < 100:
        current_error_rate = get_error_rate(deployment.version_id)
        
        if current_error_rate > baseline_error_rate * 1.5:  # 50% increase
            auto_rollback(deployment.id, reason=f"Error spike: {current_error_rate} vs {baseline_error_rate}")
            break
        
        await asyncio.sleep(60)  # Check every minute
```

**Questions:**
1. Approach 2 (feature flag) requires infrastructure. What's the simplest version that still works? Could you use an environment variable? A file on disk?
2. Approach 3 (auto-rollback) is powerful but dangerous — what if the baseline is wrong? What if the error spike is a DDoS, not a prompt issue?
3. Design a rollback that's "fast enough" without requiring complex infrastructure. What's the minimum viable rollback?
</details>

---

### Step 4: Design the Monitoring

A prompt goes live. How do you know if it's working? Not just "accuracy" — what signals tell you the prompt is healthy or failing?

> **🤔 YOUR TURN:** List 10 metrics you'd monitor for a production prompt. Group them into categories. Then: which 3 would you put on a DASHBOARD that an on-call engineer glances at?

<details>
<summary>Example monitoring categories</summary>

**PERFORMANCE METRICS (is the prompt doing its job?)**
1. Task accuracy (% correct responses)
2. Format compliance (% following the specified format)
3. Refusal rate (% "I can't answer" responses)
4. Hallucination rate (% unsupported claims)

**BEHAVIORAL METRICS (is the prompt behaving differently than expected?)**
5. Average response length (sudden change = behavior shift)
6. Sentiment of responses (becoming more negative/positive?)
7. Vocabulary shift (new words appearing → style change)
8. Specific phrase frequency (track "I think", "maybe", "definitely")

**OPERATIONAL METRICS (is the prompt costing what expected?)**
9. Average tokens per response
10. Cost per request
11. Latency (longer responses = higher latency)
12. Cache hit rate (if caching is used)

**USER-FACING METRICS (are users happy?)**
13. CSAT / thumbs up/down
14. Escalation rate (users asking for human)
15. Re-ask rate (user asks again = first answer was bad)
16. Task completion rate (did the user get what they needed?)

---

**The dashboard question again:** You can't show 16 metrics on a glanceable dashboard. Which 3?

My pick:
1. **Task accuracy** (primary goal)
2. **Refusal rate** (indicator of "safe vs useless" balance)
3. **Cost per request** (someone has to pay)

Your pick might be different — and that's fine. The right answer depends on your business goals.
</details>

---

### Step 5: Bring It Together — The Prompt Registry

Now here's a complete implementation of a prompt management service. **Before reading it, try to design your own API. What endpoints would a prompt registry need?**

<details>
<summary>Production prompt registry API</summary>

```python
"""
Prompt Registry — manages prompt versions, deployments, and rollbacks.

This is a service that runs alongside your LLM Gateway (Phase 1).
Every time the gateway needs a prompt, it asks the registry:
"What's the current active version of prompt X in environment Y?"
"""

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from datetime import datetime
from typing import Optional
import json

app = FastAPI(title="Prompt Registry")


# ── Storage (in production, use a database) ──

prompts: dict[str, Prompt] = {}
versions: dict[str, PromptVersion] = {}
deployments: list[PromptDeployment] = []
active_deployments: dict[str, PromptDeployment] = {}  # prompt_name → active deployment


# ── API Endpoints ──

@app.post("/prompts")
async def create_prompt(prompt: Prompt):
    """Register a new prompt template."""
    prompts[prompt.id] = prompt
    return {"status": "created", "id": prompt.id}


@app.get("/prompts/{prompt_id}/active")
async def get_active_prompt(prompt_id: str, environment: str = "production"):
    """
    Get the currently active prompt version for an environment.
    
    This is called by the LLM Gateway on every request.
    It needs to be FAST — cache the result.
    """
    deployment_key = f"{prompt_id}:{environment}"
    deployment = active_deployments.get(deployment_key)
    
    if not deployment:
        raise HTTPException(status_code=404, detail="No active deployment for this prompt")
    
    version = versions.get(deployment.version_id)
    if not version:
        raise HTTPException(status_code=404, detail="Version not found")
    
    return {
        "prompt_id": prompt_id,
        "version_id": version.id,
        "version_number": version.version_number,
        "content": version.content,
        "deployed_at": deployment.deployed_at,
    }


@app.post("/prompts/{prompt_id}/versions")
async def create_version(prompt_id: str, version: PromptVersion):
    """Create a new version of a prompt."""
    if prompt_id not in prompts:
        raise HTTPException(status_code=404, detail="Prompt not found")
    
    versions[version.id] = version
    return {"status": "created", "version_id": version.id, "version_number": version.version_number}


@app.post("/deployments")
async def create_deployment(deployment: PromptDeployment):
    """Deploy a prompt version to an environment."""
    deployment_key = f"{deployment.prompt_id}:{deployment.environment}"
    
    # Record the deployment
    deployments.append(deployment)
    active_deployments[deployment_key] = deployment
    
    # Update prompt's active version
    if deployment.prompt_id in prompts:
        prompts[deployment.prompt_id].active_version_id = deployment.version_id
        prompts[deployment.prompt_id].updated_at = datetime.now()
    
    return {
        "status": "deployed",
        "deployment_id": deployment.id,
        "prompt": deployment.prompt_id,
        "version": deployment.version_id,
        "environment": deployment.environment,
    }


@app.post("/deployments/{deployment_id}/rollback")
async def rollback_deployment(deployment_id: str, reason: str = ""):
    """
    Rollback a deployment to the previous version.
    
    This is the CRITICAL incident response endpoint.
    It should be SIMPLE and FAST — one API call to undo.
    """
    deployment = next((d for d in deployments if d.id == deployment_id), None)
    if not deployment:
        raise HTTPException(status_code=404, detail="Deployment not found")
    
    # Find the previous version
    current_version = versions.get(deployment.version_id)
    if not current_version or not current_version.parent_version_id:
        raise HTTPException(status_code=400, detail="No previous version to rollback to")
    
    # Create a rollback deployment (deploy the parent version)
    rollback_deploy = PromptDeployment(
        id=f"rollback_{deployment_id}_{datetime.now().timestamp()}",
        prompt_id=deployment.prompt_id,
        version_id=current_version.parent_version_id,
        environment=deployment.environment,
        strategy=DeploymentStrategy.FULL,
        status="active",
    )
    
    # Execute
    deployments.append(rollback_deploy)
    deployment_key = f"{deployment.prompt_id}:{deployment.environment}"
    active_deployments[deployment_key] = rollback_deploy
    
    # Mark old deployment as rolled back
    deployment.status = "rolled_back"
    deployment.rollback_at = datetime.now()
    deployment.rollback_reason = reason
    
    return {
        "status": "rolled_back",
        "previous_version": deployment.version_id,
        "rolled_back_to": current_version.parent_version_id,
        "reason": reason,
    }


@app.get("/history/{prompt_id}")
async def get_prompt_history(prompt_id: str, limit: int = 10):
    """Get deployment history for a prompt."""
    prompt_versions = [v for v in versions.values() if v.prompt_id == prompt_id]
    prompt_versions.sort(key=lambda v: v.created_at, reverse=True)
    
    return {
        "prompt_id": prompt_id,
        "total_versions": len(prompt_versions),
        "recent_versions": [
            {
                "version_id": v.id,
                "version_number": v.version_number,
                "author": v.author,
                "change_description": v.change_description,
                "created_at": v.created_at,
                "test_set_accuracy": v.test_set_accuracy,
            }
            for v in prompt_versions[:limit]
        ],
    }
```

**Questions for you:**
1. This API is stateless in the example (in-memory). In production, what database would you use? Why?
2. The `get_active_prompt` endpoint is on the critical path of every LLM call. How do you make it fast? (Hint: caching)
3. The rollback endpoint reverts to the parent version. What if you need to rollback 3 versions? (Not just 1)
4. This API handles prompts. But teams also need to manage model parameters (temperature, max_tokens). Should that be in the same system or separate? Why?
</details>

---

### 🤔 Final Integration Question

You now have:
- A prompt management system (this file)
- An evaluation pipeline (file 06)
- A/B testing methodology (file 06)
- A red-teaming framework (file 05)

**Design the complete workflow for this scenario:**

Your CTO says: "We need to change the customer support prompt to reduce transfer rate by 20%. We don't want to break anything. We need it live in 1 week."

You have 1 week to:
1. Design the new prompt
2. Validate it
3. Get approval
4. Deploy safely
5. Monitor for regressions
6. Have a rollback plan

**Write the plan. Day by day. Hour by hour. What exactly happens at each step?**

Don't just describe — think about:
- What if validation FAILS on day 3? Does the timeline slip?
- What if the A/B test is inconclusive after 3 days?
- What if the metrics look good but CSAT drops? (lagging indicator vs leading indicator)
- Who approves each step? What if they're unavailable?

---

## ✅ What a Working System Looks Like

After you've designed your own system, here's what a production setup might look like:

```
PROMPT MANAGEMENT DEPLOYMENT:

┌─────────────────────────────────────────────────────────────┐
│                     PROMPT REGISTRY                           │
│  Postgres: prompts, versions, deployments                    │
│  Redis: active version cache (1ms lookup)                    │
│  API: FastAPI service (internal)                             │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                  LLM GATEWAY (your Phase 1 project)           │
│  On every request:                                            │
│  1. Check cache for active prompt                            │
│  2. If cached, use it (1ms)                                  │
│  3. If not, query registry, cache result                     │
│  4. Apply prompt to request                                  │
│  5. Send to LLM                                              │
│  6. Log metrics (latency, tokens, response)                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    MONITORING                                 │
│  - Prometheus: metrics collection                            │
│  - Grafana: dashboards by prompt version                     │
│  - Alerts: anomaly detection                                 │
│  - Logs: every prompt response tracked by version            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧪 Drills

### Drill 1: Audit Your Current Process

You manage at least one prompt (from the Gateway project). Answer honestly:

1. What's the CURRENT version of your prompt?
2. What was the PREVIOUS version? What changed?
3. Who made the change? Why?
4. What metrics showed the change was an improvement?
5. If you needed to revert, how long would it take?

If you can't answer any of these — congratulations, you've found your first production risk.

### Drill 2: Design a Rollback Drill

You're on call. A prompt change goes wrong. You have 2 minutes to rollback. 

1. Write the EXACT commands/steps you'd take
2. Time yourself doing it (don't optimize — just measure)
3. If it takes more than 2 minutes, redesign the process
4. Repeat until rollback takes <30 seconds

### Drill 3: The Incident Postmortem

Pick one of your prompt changes that caused a regression (or imagine one). Write a postmortem:

```
TITLE: [What went wrong?]
DATE: [When?]
IMPACT: [What happened to users?]
ROOT CAUSE: [Why did it happen?]
DETECTION: [How did we find out?]
FIX: [What did we do?]
PREVENTION: [How to stop it happening again?]
MONITORING: [What metric would have caught it earlier?]
```

---

## 🚦 Gate Check

Before moving on:

- [ ] You can explain the difference between a prompt, a prompt version, and a prompt deployment
- [ ] You can design a rollback mechanism without needing a code deploy
- [ ] You've identified at least 3 gaps in how YOU currently manage prompts
- [ ] You understand why "it worked in dev" doesn't mean "it works in production"
- [ ] You can list 5 metrics you'd monitor alongside accuracy
- [ ] **You designed your own prompt management workflow before reading the solutions**

This file didn't teach you a "best way" to manage prompts. It gave you incidents and asked you to design solutions. The VALUE isn't in the code — it's in the thinking process you went through to design each component. When you face a real prompt incident, you won't remember my code. You'll remember the PROBLEMS and reason through the solution again. That's the point.
