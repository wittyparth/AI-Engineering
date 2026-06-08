# 🏗️ Project 2 — Prompt Engineering System

## 🎯 Purpose & Goals

**This is a two-part project. Part 1 is DESIGN. Part 2 is BUILD. Don't skip to Part 2.**

### The Brief

Your company has 5 AI features in production:
1. **Customer support triage** — classifies and routes tickets
2. **Code review assistant** — reviews PRs for bugs
3. **Product description generator** — writes e-commerce descriptions
4. **Internal knowledge QA** — answers employee questions from docs
5. **Email summarizer** — summarizes support emails for agents

Each feature was built by a different engineer. Each has a different prompt style. Each has different reliability. Each breaks at different times for different reasons.

**Your mission:** Build a Prompt Engineering System that:
1. Manages ALL 5 prompts in one place (versions, history, rollback)
2. Evaluates prompt quality automatically (test suite, metrics)
3. Detects regressions when prompts change
4. A/B tests prompt changes before full rollout
5. Monitors prompt performance in production

**⏱ Time Budget:** 6-8 hours over 2 days

---

## Part 1: DESIGN (2 hours)

### Step 1: Understand the Requirements (Don't Build Yet)

Before writing any code, you need to understand what you're building. Here are the problems you need to solve:

**Problem A:** The product description generator started making weird descriptions yesterday. No one changed the prompt... wait, someone DID change it. They forgot to tell anyone. The old prompt is lost.

**Problem B:** You think your customer support prompt could be better. You have 5 ideas. You have no way to test them except "change the prompt and see what happens."

**Problem C:** Every time you change ONE prompt, you're afraid it'll break something else. There's no "safety net."

**Problem D:** The CEO asks: "How good are our AI features?" You have no answer. You have no metrics.

> **🤔 YOUR TURN:**
> 1. List the components you'd need to solve A, B, C, and D.
> 2. Group them into "must have" vs "nice to have" for a first version.
> 3. Sketch a system diagram. What talks to what? What data flows where?
> 4. What's the MINIMUM you can build that still solves ALL 4 problems?

<details>
<summary>After you've designed, compare with this architecture</summary>

```
COMPONENTS NEEDED:

1. Prompt Registry (solves Problem A)
   - Store all prompts with version history
   - Track who changed what and when
   - One-click rollback

2. Evaluation Engine (solves Problem C)
   - Run prompts against a test suite
   - Score accuracy, format compliance, safety
   - Compare versions side by side

3. A/B Testing Framework (solves Problem B)
   - Run two prompt versions against each other
   - Statistical comparison of results
   - Gradual rollout capability

4. Monitoring Dashboard (solves Problem D)
   - Real-time metrics per prompt version
   - Alerting on metric drops
   - Historical comparison

MINIMUM VIABLE:
- Prompt Registry (store/version/rollback) — without this, you have chaos
- Evaluation Engine with test suite — without this, you can't measure quality
- Simple comparison (run A then B, compare scores) — this IS your A/B test for v1
- Dashboard = a JSON file you check — not pretty, but works

WHAT CAN WAIT:
- Gradual rollout automation → manual percentage management is fine
- Alerting → periodic check-ins
- A/B statistical significance → just compare raw scores for v1
- UI → CLI is fine for v1
</details>

---

### Step 2: Design the Data Model

You need to represent prompts, versions, evaluations, and comparisons. 

> **🤔 YOUR TURN:** Design the core data structures. What fields does a `Prompt` have? A `PromptVersion`? An `Evaluation`? A `TestSuite`? What about a `Comparison`?

Don't worry about databases yet — just the Python data classes.

<details>
<summary>After designing, compare with this approach</summary>

Notice: I'll present the PROBLEM-SOLVING thought process, not just the final answer.

**Thought process for Prompt:**
- Needs a name (for humans) and ID (for machines)
- Needs to store the system prompt and user prompt template
- Needs metadata: author, created date, model it's for
- Wait, the system AND user prompt are separate! Because the system prompt stays constant while the user prompt has {variables}

**Thought process for PromptVersion:**
- Every change to a prompt = new version
- Need to know: what changed, who changed it, why
- Need a parent pointer for rollback
- Need the evaluation results for this version

**Thought process for Evaluation:**
- Need to know: what was tested, what were the results, what metric was used
- An evaluation compares ONE prompt version against ONE test suite
- Need per-category breakdowns, not just overall scores

```python
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field
from enum import Enum


class Prompt(BaseModel):
    """A prompt template managed by the system."""
    id: str
    name: str
    description: str
    model: str = "gpt-4o-mini"
    temperature: float = 0.0
    system_prompt: str = ""
    user_prompt_template: str = "{input}"  # With {placeholders}
    
    created_at: datetime = Field(default_factory=datetime.now)
    active_version_id: Optional[str] = None


class PromptVersion(BaseModel):
    """A specific version of a prompt."""
    id: str
    prompt_id: str
    version_number: int
    
    system_prompt: str
    user_prompt_template: str
    
    # Traceability
    author: str
    change_description: str
    parent_version_id: Optional[str] = None
    
    created_at: datetime = Field(default_factory=datetime.now)
    hash: str = ""


class TestCase(BaseModel):
    """A single test case for evaluation."""
    id: str
    input: str
    expected_output: str   # Reference answer
    category: str          # "factual", "reasoning", "safety", "format"
    difficulty: str        # "easy", "medium", "hard"
    must_contain: list[str] = []
    must_not_contain: list[str] = []


class TestSuite(BaseModel):
    """A collection of test cases."""
    id: str
    name: str
    prompt_id: str
    cases: list[TestCase]
    
    created_at: datetime = Field(default_factory=datetime.now)


class EvaluationScore(BaseModel):
    """Score for a single category."""
    category: str
    passed: int
    total: int
    score: float  # passed/total


class Evaluation(BaseModel):
    """Results of running a prompt version against a test suite."""
    id: str
    prompt_version_id: str
    test_suite_id: str
    scores: list[EvaluationScore]
    overall_score: float
    total_tokens: int
    total_cost: float
    latency_ms: float
    
    evaluated_at: datetime = Field(default_factory=datetime.now)


class Comparison(BaseModel):
    """Comparison of two prompt versions on the same test suite."""
    id: str
    test_suite_id: str
    version_a_id: str
    version_b_id: str
    evaluation_a_id: str
    evaluation_b_id: str
    
    # Results
    winner: str  # "A", "B", "tie"
    score_difference: float
    significant: bool  # Using simple threshold, not p-value for v1
    
    compared_at: datetime = Field(default_factory=datetime.now)
```
</details>

---

### Step 3: Design the Flow

You have prompts, versions, test suites, evaluations, and comparisons. Now design the workflow.

> **🤔 YOUR TURN:** Trace the lifecycle of a prompt change:
> 1. Engineer has an idea → 
> 2. They create a new version → 
> 3. ??? → 
> 4. The new version is "live" →

**What happens at each step? What gets created? What gets checked?**

<details>
<summary>Compare (in order of your discovery, not presented as "the answer")</summary>

**Engineer has an idea** → They want to change the prompt to improve something.

**Create a new version** → Copy the current active version. Make changes. Add a description of WHY.

**BUT: How do they know the new version is better?** They need to EVALUATE it.

**Evaluate** → Run the new version against the existing test suite. Get scores. Compare with the previous version's scores.

**BUT: What if the test suite doesn't cover the new behavior?** They might need to add test cases.

**After evaluation: Is it better?** Compare scores. If better → can move forward. If worse → iterate or abandon. If same → was the change worth it?

**Deploy** → The new version becomes the "active" version. The old version is still accessible (for rollback).

**Monitor** — Now the REAL test begins. Production data will show if the improvement was real.

**Rollback** — If production metrics drop, switch back to the previous version. Should take < one command.
</details>

---

## Part 2: BUILD (4-6 hours)

Now build it. I'll give you the structure, but **you fill in the critical parts.**

### Step 1: Project Structure

```
prompt-engineering-system/
├── README.md                # Architecture, setup, usage
├── requirements.txt         # fastapi, pydantic, httpx, ...
├── main.py                  # FastAPI app
│
├── core/
│   ├── models.py            # Data models (from your design)
│   ├── registry.py          # Prompt storage, versions, rollback
│   ├── evaluator.py         # Test suite runner
│   └── comparator.py        # A/B comparison logic
│
├── prompts/                 # Your 5 prompts
│   ├── support_triage.json
│   ├── code_review.json
│   ├── product_desc.json
│   ├── knowledge_qa.json
│   └── email_summary.json
│
├── tests/
│   ├── conftest.py
│   ├── test_registry.py
│   └── test_evaluator.py
│
└── test_suites/             # Test cases for each prompt
    ├── support_triage.json
    ├── code_review.json
    └── ...
```

### Step 2: Build the Registry

```python
# core/registry.py
"""
The Prompt Registry stores and versions prompts.
You designed the data model earlier. Now implement the operations.

I'll give you the START of each function. You complete the critical parts.
"""

from datetime import datetime
from typing import Optional
from core.models import Prompt, PromptVersion


class PromptRegistry:
    """
    Prompt storage and version management.
    
    Stores prompts in memory for now (swap for database in production).
    """
    
    def __init__(self):
        self._prompts: dict[str, Prompt] = {}
        self._versions: dict[str, list[PromptVersion]] = {}
        self._version_counter = 0
    
    def register(self, name: str, system_prompt: str, user_template: str,
                 description: str = "", model: str = "gpt-4o-mini") -> Prompt:
        """
        Register a new prompt.
        
        Creates the prompt AND its first version (v1).
        """
        # TODO: Implement this
        # 1. Create a Prompt with a unique ID
        # 2. Create PromptVersion v1 with the initial content
        # 3. Link them
        # 4. Return the prompt
        pass
    
    def create_version(self, prompt_id: str, system_prompt: str,
                       user_template: str, author: str,
                       change_description: str) -> PromptVersion:
        """
        Create a new version of a prompt.
        
        The new version's parent is the previous active version.
        """
        prompt = self._prompts.get(prompt_id)
        if not prompt:
            raise ValueError(f"Prompt {prompt_id} not found")
        
        self._version_counter += 1
        
        # 🔴 YOUR TURN: What fields does this version need?
        # - What version number should it have?
        # - What's its parent?
        # - How do you create a unique ID?
        # Finish this implementation.
        
        version = PromptVersion(
            id=f"v{self._version_counter}",
            prompt_id=prompt_id,
            version_number=0,  # FIXME: calculate correctly
            # ... add remaining fields
        )
        
        # Store version
        if prompt_id not in self._versions:
            self._versions[prompt_id] = []
        self._versions[prompt_id].append(version)
        
        # Update prompt's active version
        prompt.active_version_id = version.id
        prompt.updated_at = datetime.now()
        
        return version
    
    def get_active_version(self, prompt_id: str) -> Optional[PromptVersion]:
        """
        Get the current active version of a prompt.
        """
        # TODO: Implement
        pass
    
    def get_version_history(self, prompt_id: str) -> list[PromptVersion]:
        """
        Get all versions of a prompt, newest first.
        """
        versions = self._versions.get(prompt_id, [])
        return sorted(versions, key=lambda v: v.created_at, reverse=True)
    
    def rollback(self, prompt_id: str) -> PromptVersion:
        """
        Rollback to the previous version.
        
        This creates a NEW version that's a copy of the parent.
        (So you maintain a linear version history)
        """
        # 🔴 YOUR TURN: Implement rollback
        # 1. Get current active version
        # 2. Get its parent version
        # 3. Create a new version with parent's content
        # 4. Make it active
        # 5. Return the new version
        pass
```

**Your job:** Complete the `create_version` and `rollback` methods. Don't look up answers — think through what needs to happen.

<details>
<summary>After implementing, check your logic</summary>

**create_version key decisions:**
- Version number = previous max + 1 (not counter-based — gaps if versions are deleted)
- Parent = previous active version (not the version being edited)
- Hash = SHA256 of content (detect accidental changes)

**rollback key decisions:**
- Don't just "switch active pointer" — that makes the version graph non-linear
- Instead, CREATE A NEW VERSION with the parent's content
- Version history stays linear: v1 → v2 → v3 → v4 (rollback of v3) = copy of v2
- This feels redundant, but it preserves AUDIT TRAIL

```python
def rollback(self, prompt_id: str) -> PromptVersion:
    current = self.get_active_version(prompt_id)
    if not current or not current.parent_version_id:
        raise ValueError("No version to rollback to")
    
    parent = self._get_version(current.parent_version_id)
    if not parent:
        raise ValueError("Parent version not found")
    
    return self.create_version(
        prompt_id=prompt_id,
        system_prompt=parent.system_prompt,
        user_template=parent.user_prompt_template,
        author="system",
        change_description=f"Rollback: reverted to v{parent.version_number} (was v{current.version_number})",
    )
```

One question for you: should rollback SKIP evaluation? In a real incident (Problem A), you want to rollback IMMEDIATELY without waiting for tests. Later, you re-evaluate. This means rollback should NOT require passing tests.
</details>

---

### Step 3: Build the Evaluator

```python
# core/evaluator.py
"""
Evaluation engine — runs prompts against test suites and scores them.

The most IMPORTANT part of this system. If your evaluation is wrong,
ALL your prompt decisions are wrong.
"""

from core.models import TestSuite, TestCase, Evaluation, EvaluationScore


class PromptEvaluator:
    """
    Evaluates a prompt version against a test suite.
    
    Uses LLM-as-judge for scoring.
    """
    
    def __init__(self, llm_client):
        self.client = llm_client
    
    async def evaluate(self, system_prompt: str, user_template: str,
                       test_suite: TestSuite) -> Evaluation:
        """
        Run a prompt against all test cases and score results.
        
        🔴 YOUR TURN: Before reading the implementation, design the evaluation logic:
        
        For each test case:
        1. Fill in the user template: user_template.format(input=case.input)
        2. Send system_prompt + filled user prompt to LLM
        3. Check the response against case.expected_output
        
        BUT HOW DO YOU CHECK? 
        - Exact match is too strict ("Paris" vs "The capital is Paris")
        - Contains is too loose
        - What's the RIGHT way to evaluate?
        
        Design the evaluation logic before reading below.
        """
        total_cases = len(test_suite.cases)
        # ... implement evaluation logic
        
        # Hint: For v1, you can use a MULTI-LEVEL scoring:
        # - Exact match (for strict format tasks)
        # - Contains match (for keyword tasks)
        # - LLM judge match (for semantic tasks)
        # - Human review (for the hard cases)
        pass
    
    def _score_response(self, response: str, case: TestCase) -> float:
        """
        Score a single response against expected output.
        
        Returns 0.0 to 1.0.
        """
        score = 0.0
        response_lower = response.lower()
        
        # 🔴 YOUR TURN: Design the scoring
        # Think about what constitutes "correct" for different task types:
        # - Classification: exact category match
        # - Extraction: fields match
        # - Generation: semantic similarity
        # - Safety: proper refusal
        #
        # How would you handle each?
        pass
```

> **🤔 YOUR TURN (critical thinking):**
> 
> The evaluation is the HEART of this system. If it's wrong, everything built on top is wrong.
> 
> **Problems you'll face:**
> 1. The LLM response is "The capital of France is Paris" but expected is "Paris" — is this correct? How do you handle non-exact matches?
> 2. The LLM gives a correct answer but in a different format. Is that a pass or fail?
> 3. The test case says "must not mention competitors" and the response doesn't. Is that a safety pass? What if it also doesn't answer the question?
> 
> **Design a scoring system that handles these cases.** Then implement it.

<details>
<summary>A production approach to scoring (compare with yours)</summary>

**Multi-level scoring approach:**

```python
def _score_response(self, response: str, case: TestCase, task_type: str) -> float:
    """
    Score based on task type.
    
    Different tasks need different scoring logic.
    """
    if task_type == "classification":
        return self._score_classification(response, case)
    elif task_type == "extraction":
        return self._score_extraction(response, case)
    elif task_type == "generation":
        return self._score_semantic(response, case)
    elif task_type == "safety":
        return self._score_safety(response, case)
    else:
        return self._score_general(response, case)

def _score_general(self, response: str, case: TestCase) -> float:
    """General scoring: check constraints + semantic match."""
    score = 0.0
    
    # Check required content (0-0.5 points)
    if case.must_contain:
        required_points = 0.5 / len(case.must_contain)
        for pattern in case.must_contain:
            if pattern.lower() in response.lower():
                score += required_points
    
    # Check prohibited content (0-0.3 points)
    if case.must_not_contain:
        for pattern in case.must_not_contain:
            if pattern.lower() not in response.lower():
                score += 0.3 / len(case.must_not_contain)
    
    # Check against expected (0-0.2 points)
    # This uses a SIMILARITY heuristic, not exact match
    expected_lower = case.expected_output.lower()
    response_lower = response.lower()
    
    # Overlap coefficient
    expected_words = set(expected_lower.split())
    response_words = set(response_lower.split())
    if expected_words and response_words:
        overlap = len(expected_words & response_words)
        score += 0.2 * (overlap / len(expected_words))
    
    return min(score, 1.0)
```

**Problems with this approach:**
1. Word overlap is a terrible measure of semantic correctness
2. "The capital is Paris" has 0 words overlapping with "Paris" if your expected is just "Paris"
3. A completely wrong answer with similar words could score high

**Better approach for v2:** Use LLM-as-judge for semantic evaluation. But that costs money per evaluation. Tradeoffs everywhere.
</details>

---

### Step 4: Build the Comparator (A/B Testing)

```python
# core/comparator.py
"""
Compare two prompt versions to determine which is better.

This is NOT just "which has higher accuracy" — it's about understanding
WHERE and WHY one is better.
"""


class PromptComparator:
    """
    Compare two prompt versions on the same test suite.
    
    Gives you: which is better, by how much, in which categories.
    """
    
    def compare(self, eval_a: Evaluation, eval_b: Evaluation) -> dict:
        """
        Compare two evaluations.
        
        🔴 YOUR TURN: Design the comparison output.
        
        What does the engineer who made the change NEED TO KNOW?
        - Which version is better overall?
        - In which categories?
        - By how much?
        - Is the difference meaningful?
        
        DON'T just say "B is 2% better." That's useless.
        What SPECIFIC information helps the engineer decide?
        """
        # Calculate differences
        category_diffs = []
        ...  # YOUR CODE HERE
        
        return {
            "winner": "...",
            "overall_difference": 0.0,
            "category_breakdown": category_diffs,
            "is_significant": False,
            # What else belongs here?
        }
```

> **🤔 YOUR TURN:** What would make you CONFIDENT that version B is actually better than version A?
>
> - If B scores 85% and A scores 84% on a 100-case test suite, is B better?
> - What if they differ by 5+%? By 0.5%?
> - What if A is better on "factual" and B is better on "safety" — which one wins?
>
> **Design a decision rule for "deploy B or keep A"** before implementing it. Then implement it.

<details>
<summary>One approach to comparison decisions</summary>

```python
def compare(self, eval_a: Evaluation, eval_b: Evaluation) -> dict:
    """Compare evaluations with meaningful output."""
    
    # Category-level breakdown
    categories = {}
    for score_a in eval_a.scores:
        score_b = next(s for s in eval_b.scores if s.category == score_a.category)
        diff = score_b.score - score_a.score
        categories[score_a.category] = {
            "a_score": score_a.score,
            "b_score": score_b.score,
            "difference": diff,
            "a_passed": score_a.passed,
            "a_total": score_a.total,
            "b_passed": score_b.passed,
            "b_total": score_b.total,
        }
    
    # Overall comparison
    overall_diff = eval_b.overall_score - eval_a.overall_score
    
    # Is the difference MEANINGFUL?
    # For v1: use a simple threshold (5% difference = meaningful)
    # This is NOT statistically rigorous — that's v2
    meaningful_threshold = 0.05
    
    if overall_diff > meaningful_threshold:
        winner = "B"
    elif overall_diff < -meaningful_threshold:
        winner = "A"
    else:
        winner = "tie"
    
    return {
        "winner": winner,
        "overall_difference": round(overall_diff, 3),
        "category_breakdown": categories,
        "improved_categories": [cat for cat, data in categories.items() if data["difference"] > 0],
        "regressed_categories": [cat for cat, data in categories.items() if data["difference"] < 0],
        "significant_categories": [
            cat for cat, data in categories.items() 
            if abs(data["difference"]) > meaningful_threshold
        ],
        "notes": self._generate_notes(winner, categories, overall_diff),
    }

def _generate_notes(self, winner: str, categories: dict, overall_diff: float) -> list[str]:
    """Generate human-readable insights."""
    notes = []
    
    if winner == "tie":
        notes.append("No meaningful overall difference detected.")
    else:
        notes.append(f"{'B' if winner == 'B' else 'A'} is better overall by {abs(overall_diff):.1%}.")
    
    # Find regressions
    regressions = {cat: data for cat, data in categories.items() if data["difference"] < -0.03}
    if regressions:
        for cat, data in regressions.items():
            notes.append(f"⚠️ Regression in '{cat}': {data['a_score']:.0%} → {data['b_score']:.0%}")
    
    # Find improvements
    improvements = {cat: data for cat, data in categories.items() if data["difference"] > 0.03}
    if improvements:
        for cat, data in improvements.items():
            notes.append(f"✅ Improvement in '{cat}': {data['a_score']:.0%} → {data['b_score']:.0%}")
    
    return notes
```

**Your questions to answer:**
1. The threshold is 5%. Why 5%? What if you're in a domain where 1% matters (finance, medical)?
2. What if version B wins on 4 categories but loses badly on 1? Is it still the winner?
3. The comparison doesn't account for sample size. With 10 test cases, a 10% difference is ONE case. With 1000, it's 100. Should the threshold change?
</details>

---

### Step 5: Build the API

```python
# main.py
"""
Prompt Engineering System — FastAPI application.

Exposes:
- POST /prompts — Register a new prompt
- POST /prompts/{id}/versions — Create new version
- POST /prompts/{id}/evaluate — Run evaluation
- POST /prompts/{id}/compare — Compare two versions
- POST /prompts/{id}/deploy — Set active version
- POST /prompts/{id}/rollback — Rollback to previous
- GET /prompts/{id}/history — Version history
- GET /prompts/{id}/metrics — Performance metrics
"""

from fastapi import FastAPI, HTTPException
from core.registry import PromptRegistry
from core.evaluator import PromptEvaluator
from core.comparator import PromptComparator

app = FastAPI(title="Prompt Engineering System")
registry = PromptRegistry()
evaluator = None  # Initialize with your LLM client
comparator = PromptComparator()


@app.post("/prompts")
async def register_prompt(
    name: str, system_prompt: str, user_template: str,
    description: str = "", model: str = "gpt-4o-mini"
):
    """Register a new prompt."""
    prompt = registry.register(name, system_prompt, user_template, description, model)
    return {"status": "created", "prompt_id": prompt.id, "version": 1}


@app.post("/prompts/{prompt_id}/versions")
async def create_prompt_version(
    prompt_id: str, system_prompt: str, user_template: str,
    author: str, change_description: str
):
    """Create and evaluate a new prompt version."""
    # 1. Create the version
    version = registry.create_version(prompt_id, system_prompt, user_template, author, change_description)
    
    # 2. Get the test suite for this prompt
    test_suite = get_test_suite(prompt_id)
    
    # 3. Evaluate
    # evaluation = await evaluator.evaluate(version.system_prompt, version.user_prompt_template, test_suite)
    
    # 4. Compare with previous version
    # comparison = comparator.compare(previous_evaluation, evaluation)
    
    return {
        "version_id": version.id,
        "version_number": version.version_number,
        "parent_version": version.parent_version_id,
        # "evaluation": evaluation,
        # "comparison": comparison,
    }


@app.post("/prompts/{prompt_id}/evaluate")
async def evaluate_prompt(prompt_id: str, version_id: str):
    """
    Evaluate a specific version against its test suite.
    
    This is the CORE operation. Run this BEFORE deploying any change.
    """
    # TODO: Implement
    # 1. Get the prompt version from registry
    # 2. Get the test suite
    # 3. Run evaluation
    # 4. Return scores
    pass


@app.post("/prompts/{prompt_id}/compare")
async def compare_versions(prompt_id: str, version_a_id: str, version_b_id: str):
    """
    Compare two versions side-by-side.
    
    This is your A/B testing endpoint. Run it to see if version B
    is actually better than version A.
    """
    # TODO: Implement
    pass


@app.post("/prompts/{prompt_id}/deploy")
async def deploy_version(prompt_id: str, version_id: str):
    """
    Set a version as the active (production) version.
    
    This should ONLY be called AFTER evaluation passes.
    """
    # TODO: Implement
    pass


@app.post("/prompts/{prompt_id}/rollback")
async def rollback_prompt(prompt_id: str):
    """
    Emergency rollback to previous version.
    
    Called when a deployed prompt is causing issues.
    This should be FAST — no evaluation needed.
    """
    version = registry.rollback(prompt_id)
    return {
        "status": "rolled_back",
        "new_active_version": version.id,
        "new_version_number": version.version_number,
    }


@app.get("/prompts/{prompt_id}/history")
async def get_prompt_history(prompt_id: str):
    """Get full version history with evaluation results."""
    versions = registry.get_version_history(prompt_id)
    return {
        "prompt_id": prompt_id,
        "versions": [
            {
                "version_number": v.version_number,
                "author": v.author,
                "change": v.change_description,
                "created": v.created_at,
                "is_active": v.id == registry.get_active_version(prompt_id).id,
            }
            for v in versions
        ],
    }
```

---

### Step 6: Create Your 5 Prompts

This is where you actually write prompts for the 5 features. But don't just write them — follow the process you've built:

1. Write an INITIAL version (v1)
2. Create a test suite (at least 10 test cases per prompt)
3. Run evaluation — what scores do you get?
4. Identify weaknesses — where does the prompt fail?
5. Write v2 — address the weaknesses
6. Run comparison — is v2 better?
7. Iterate until satisfied

**Start with ONE prompt (customer support triage).** Don't do all 5 until you've validated the process on one.

---

### 🤔 Final Integration Questions

Your system is built. You have 5 prompts managed, evaluated, compared, and deployed. Now answer these:

**Question 1:** You deploy a new version of the product description prompt. It scores 94% on the test suite (previous version: 91%). A week later, the marketing team complains that descriptions are "boring and formulaic." Your test suite doesn't measure "interestingness."

- How would you add "interestingness" to your evaluation?
- Should this block deployment going forward? Why or why not?
- If your metrics AND human judgment disagree, who wins?

**Question 2:** An engineer creates a new version of the email summarizer. It scores 96% — AMAZING! But looking closer, you notice the test suite only has 12 cases. And comparing outputs, the "improved" version is actually just returning the first sentence of each email. The test cases happened to be short emails where the first sentence contained the summary.

- Your test suite is GARBAGE. It's measuring the wrong thing and giving false confidence. How do you detect this?
- Design a test suite QUALITY metric — something that tells you "this test suite is good" vs "this test suite is misleading."
- How often should test suites be audited? By whom?

**Question 3:** The A/B test says version B is 2% better. You deploy B. Two weeks later, B is still scoring 2% better on your evaluations, but customer satisfaction dropped 10%. 

- What's happening? Your internal metric and your user metric disagree.
- Should you trust your evaluation system or your user feedback?
- How would you DETECT this blind spot before deploying?

---

## ✅ Project Completion Checklist

- [ ] Registry implemented: create, get, version, rollback all work
- [ ] Evaluator implemented: runs prompts against test cases, returns scores
- [ ] Comparator implemented: compares two evaluations, identifies category-level differences
- [ ] API built: all 6 endpoints respond correctly
- [ ] At least 1 prompt with 10+ test cases exists
- [ ] You ran an evaluation and saw scores
- [ ] You created a v2, ran comparison, and decided whether to deploy
- [ ] You successfully rolled back a version
- [ ] README written explaining architecture and decisions
- [ ] Pushed to GitHub

---

## 📚 Cross-References

- [Evaluation & A/B Testing](06-Evaluation-AB-Testing.md) — The evaluation methodology
- [Production Prompt Management](09-Production-Prompt-Management.md) — Registry and deployment design
- [Red-Teaming & Anti-Patterns](05-Red-Teaming-Antipatterns.md) — Test cases for safety evaluation
- [System Prompts & Personas](03-System-Prompts-Personas.md) — The prompts you're managing
- [Phase 1 Gateway Project](../01-Foundations/Project-Cornerstone-Multi-Provider-LLM-Gateway.md) — Your gateway consumes these prompts
- [Phase 7: Evals & Observability](../07-Production-Evals-Observability/) — Production monitoring for prompts
