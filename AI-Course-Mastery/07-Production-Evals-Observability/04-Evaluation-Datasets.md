# 04 — Evaluation Datasets: The Foundation Your Metrics Stand On

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about building evaluation datasets, let me show you why this is the most undervalued skill in AI engineering.

Here's an uncomfortable truth: **Your evaluation is only as good as your dataset. And your dataset is probably worse than you think.**

Every metric from File 03, every LLM judge from File 02, every eval pipeline from File 05 — they all depend on having a dataset of test cases that represents what actually happens in production. If your dataset is bad, your evaluation is meaningless. Not "slightly less useful." **Meaningless.**

And building a good evaluation dataset is HARD. Not technically hard — any engineer can scrape some examples and write a JSON file. But DESIGNING a dataset that actually catches regressions, represents production reality, and stays fresh over time — that's a skill that separates senior from junior engineers.

---

### 🤔 Discovery Question 1: The Stale Dataset Problem

Six months ago, your team built a golden evaluation dataset. 500 carefully curated question-answer pairs. Human-annotated. Double-checked. Inter-annotator agreement of 0.88. It was beautiful.

Fast forward to today. Your system scores 94% on this dataset. You've been shipping with confidence. Users seem happy.

But then someone runs a new evaluation: they sample 200 recent production queries and manually label them. **Your system scores 78% on the new sample.**

**🤔 Before reading on, answer these:**

1. What happened? The old dataset was high quality. The system should be 94% good. Was the original dataset wrong, or did the world change?

2. List at least 3 specific ways a dataset can become "stale" — where the queries or expected answers from 6 months ago no longer represent today's reality. Think about: product changes, user behavior shifts, external events, language drift.

3. Your PM sees both numbers: 94% on the old set, 78% on the new set. She says: "Our system got worse." Is she right? What if the OLD dataset was also wrong (the world changed, so old queries don't matter anymore)? What if the NEW dataset has annotation errors? How do you determine the ACTUAL system quality when your datasets disagree?

4. **The hard question:** You can't re-annotate the old dataset every month (too expensive). You can't run the new sample every month (also expensive). What SIGNAL would you use to detect when your evaluation dataset has become stale — without needing new human annotations? Think about what changes in production that correlates with dataset staleness.

---

### 🤔 Discovery Question 2: The Contamination That Inflated Everything

Your team achieves 96% on your evaluation dataset. You're proud. The CTO is impressed. You present at an all-hands.

Three months later, someone discovers a bug in the data pipeline: **30% of your training examples overlap with your evaluation examples.**

The dataset was built by sampling production logs, splitting 80/20 into train and eval. But the splitting was done BEFORE deduplication. The same user question appeared in both sets. Your model effectively "memorized" the eval examples during training.

Your real accuracy: probably closer to 80%.

**🤔 Before reading on, answer these:**

1. This sounds obvious in hindsight (never let train and eval overlap). But why is this harder than it sounds in practice? What kinds of overlap are HARD to detect?

2. Think about different types of contamination:
   - **Exact match:** Same question in train and eval (easy to detect)
   - **Semantic match:** Different wording, same meaning (hard to detect)
   - **Topic leakage:** Eval questions about a topic that appears frequently in training (very hard to detect)
   - **Time leakage:** Eval data from a time period DURING the model's training data cutoff (invisible)

   For each type: how would you detect it? How much would it inflate your metrics?

3. You're using an LLM (not a fine-tuned model) with RAG. Is contamination still a problem? Think: GPT-4 was trained on internet data. Your eval dataset uses queries from your company's support logs. Are those in GPT-4's training data? If not, contamination is less of a concern. But what if you're using domain-specific RAG where the eval queries MIRROR public internet questions? How do you check if your eval questions are in the model's training data?

---

### 🤔 Discovery Question 3: The Annotator Bias That Skewed Everything

You hire 3 annotators to label 1,000 customer support queries for "intent classification." The categories: "billing," "technical," "account," "product question," "other."

The annotation guidelines are 10 pages. You think they're thorough.

Results come back:

| Category | Annotator A | Annotator B | Annotator C |
|----------|------------|------------|------------|
| Billing | 220 | 310 | 180 |
| Technical | 300 | 190 | 350 |
| Account | 180 | 250 | 150 |
| Product | 200 | 150 | 220 |
| Other | 100 | 100 | 100 |

Inter-annotator agreement: **0.62** (Cohen's Kappa). Borderline acceptable.

But when you dig into the disagreements, you find a pattern:
- Annotator B labels MORE things as "billing" — they interpret any mention of money as billing intent
- Annotator C labels MORE things as "technical" — they interpret any mention of error messages as technical
- Annotator A is the most "balanced" — but is that because they're right, or because they're guessing?

**🤔 Before reading on, answer these:**

1. You need ONE ground truth label per query. You have 3 annotators who disagree 38% of the time. How do you resolve disagreements? Majority vote? Expert adjudication? Something else? What are the tradeoffs of each approach?

2. The majority vote gives you one label per query. But the majority might be WRONG — if 2 out of 3 annotators share the same bias, the majority amplifies that bias. How do you detect systematic bias in your annotation process?

3. You discover that Annotator B (the one who over-labels "billing") was previously a billing support agent. Their experience makes them SEE billing issues everywhere. Is this a bug or a feature? Does domain expertise make annotation better or worse? How would you design an annotation process that leverages domain expertise WITHOUT introducing systematic bias?

4. **Cross-reference to Phase 2 (Prompt Engineering):** In Phase 2, you learned that prompt wording dramatically affects LLM outputs. The same is true for annotation instructions. If your annotation guidelines say "Label as 'billing' if the customer mentions charges, payments, invoices, or refunds," you'll get different results than if they say "Label as 'billing' only if the primary issue is about charges or payments." How do you write annotation guidelines that are specific enough to be consistent but broad enough to cover real cases?

---

> **These 3 problems — staleness, contamination, and annotator bias — are the most common reasons evaluation datasets fail in production. Everything in this module is designed to solve them.**

By the end of this module, you will:

- Build evaluation datasets from production data with proper sampling strategies
- Implement dataset versioning and change tracking
- Generate synthetic evaluation data for edge cases
- Detect and prevent dataset contamination
- Measure and improve annotation quality
- Design a dataset maintenance plan that keeps your evaluation fresh

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions (above) | 25 min |
| Dataset Design Principles | 20 min |
| Production Sampling Strategies | 20 min |
| Synthetic Data Generation | 25 min |
| Dataset Versioning | 15 min |
| Annotation Quality | 20 min |
| Code: Production Sampling Pipeline | 30 min |
| Code: Dataset Versioning System | 25 min |
| Code: Synthetic Data Generator | 30 min |
| Code: Contamination Detection | 20 min |
| Code: Annotation Quality Tracker | 25 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 10 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Anatomy of a Good Evaluation Dataset

A good evaluation dataset has these properties:

**Representativeness:** The dataset matches the distribution of production queries. If production has 60% billing questions, 30% technical, 10% other — your eval set should reflect this (unless you deliberately over-sample edge cases, which is a separate concern).

**Freshness:** The dataset is no more than X months old, where X depends on your domain. For a product support bot, 3 months might be too old. For a physics Q&A system, 2 years might be fine.

**Coverage:** The dataset covers all known failure modes, not just happy paths. If you've ever seen a prompt injection attempt, your eval set should include one.

**Stability:** The dataset doesn't change without versioning. You can always compare today's score against last month's score on the SAME dataset version.

**Purity:** No contamination. No overlap with training data. No leakage from future data.

### 2. Dataset Types

| Type | Source | Size | Annotation Quality | Maintenance Burden |
|------|--------|------|-------------------|-------------------|
| **Golden Dataset** | Human-annotated production samples | 100-500 | Highest | High (manual re-annotation) |
| **Production Sample** | Random sample from production logs | 1,000-10,000 | Medium | Low (auto-captured) |
| **Synthetic Dataset** | LLM-generated test cases | Unlimited | Variable | Low (auto-generated) |
| **Edge Case Collection** | Curated failure cases | 50-200 | High | Medium (manual curation) |
| **Adversarial Dataset** | Deliberately crafted attacks | 50-200 | High | Medium (need adversarial thinking) |

**🤔 Checkpoint:** Which type matters MOST for catching regressions? Which type matters most for establishing a baseline? Your answer reveals your risk tolerance.

### 3. Production Sampling Strategies

How do you sample from production logs to build a representative dataset?

**Strategy 1: Random Sampling (The Baseline)**

Every production query has equal chance of being selected.

- ✅ Simple, unbiased
- ❌ Rare but critical events (like prompt injection) will be missed
- ❌ The sample will be dominated by common, easy queries

**Strategy 2: Stratified Sampling (Better)**

Group queries by category, then sample evenly from each group.

- ✅ Ensures coverage across categories
- ✅ Prevents common categories from dominating
- ❌ Requires good category labels on your production data

**Strategy 3: Edge-Case Oversampling (Best for Safety)**

Deliberately over-sample rare but important events.

- ✅ Catches failures on edge cases
- ✅ Good for safety-critical systems
- ❌ Skews your metrics — you'll see lower "overall accuracy" because you're testing harder cases

**Strategy 4: Temporal Sampling (For Freshness)**

Weight sampling toward recent queries.

- ✅ Ensures dataset reflects current production reality
- ✅ Good for fast-changing domains
- ❌ Can miss long-tail issues that only appear over time

The BEST approach: combine strategies. Use stratified temporal sampling with edge-case oversampling. And report your sampling strategy so everyone knows what they're looking at.

---

## 💻 Code Examples

### Example 1: Production Sampling Pipeline

This code shows how to build a representative sample from production logs. **But it has a design flaw — find it.**

```python
"""
production_sampler.py

Samples production logs to build evaluation datasets.

⚠️ This code has a subtle bug. Read it carefully and find it before running.
"""

from dataclasses import dataclass
from typing import Any, Optional
import random
import json
from datetime import datetime, timedelta


@dataclass
class EvalSample:
    query: str
    expected_output: Any
    category: str
    timestamp: str
    source: str  # "production" | "synthetic" | "curated"


class ProductionSampler:
    """
    Samples production logs to build evaluation datasets.
    
    Uses stratified temporal sampling:
    - Stratified: sample evenly across query categories
    - Temporal: weight toward recent queries
    - Edge-case: guaranteed inclusion of rare categories
    """
    
    def __init__(self, rng_seed: int = 42):
        self.rng = random.Random(rng_seed)
    
    def sample(
        self,
        production_logs: list[dict],
        target_size: int = 500,
        categories: Optional[list[str]] = None,
        recent_weight_days: int = 30,
        edge_case_categories: Optional[list[str]] = None,
    ) -> list[EvalSample]:
        """
        Sample production logs to build an eval dataset.
        
        Args:
            production_logs: List of {query, expected_output, category, timestamp}
            target_size: Desired sample size
            categories: Categories to sample from (None = all)
            recent_weight_days: How many days counts as "recent"
            edge_case_categories: Categories to guarantee in sample
        
        Returns list of EvalSample objects.
        """
        if not production_logs:
            return []
        
        # Filter by categories if specified
        if categories:
            logs = [l for l in production_logs if l.get("category") in categories]
        else:
            logs = list(production_logs)
        
        # Separate edge cases (guaranteed inclusion)
        edge_logs = []
        if edge_case_categories:
            edge_logs = [
                l for l in logs
                if l.get("category") in edge_case_categories
            ]
            # Remove edge cases from main pool (they'll be added separately)
            logs = [l for l in logs if l not in edge_logs]
        
        # Temporal weighting: recent queries get higher selection probability
        now = datetime.now()
        for log in logs:
            try:
                log_ts = datetime.fromisoformat(log.get("timestamp", ""))
                days_ago = (now - log_ts).days
                # Weight = 1.0 for today, decaying to 0.2 for older than recent_weight_days
                weight = max(0.2, 1.0 - (days_ago / recent_weight_days))
                log["_weight"] = weight
            except (ValueError, TypeError):
                log["_weight"] = 0.5  # Default weight for unparseable timestamps
        
        # Stratified: sample evenly across categories
        category_groups: dict[str, list] = {}
        for log in logs:
            cat = log.get("category", "unknown")
            if cat not in category_groups:
                category_groups[cat] = []
            category_groups[cat].append(log)
        
        # Determine how many per category
        num_categories = len(category_groups)
        samples_per_category = max(1, (target_size - len(edge_logs)) // num_categories)
        
        sampled = []
        
        for cat, cat_logs in category_groups.items():
            weights = [l.get("_weight", 0.5) for l in cat_logs]
            
            # Sample with replacement if we need more than available
            if len(cat_logs) <= samples_per_category:
                # Take all
                selected = list(cat_logs)
            else:
                # Weighted sample
                total_weight = sum(weights)
                probs = [w / total_weight for w in weights]
                indices = self.rng.choices(
                    range(len(cat_logs)),
                    weights=probs,
                    k=samples_per_category,
                )
                selected = [cat_logs[i] for i in indices]
            
            for log in selected:
                sampled.append(EvalSample(
                    query=log.get("query", ""),
                    expected_output=log.get("expected_output"),
                    category=log.get("category", "unknown"),
                    timestamp=log.get("timestamp", ""),
                    source="production",
                ))
        
        # Add guaranteed edge cases
        for log in edge_logs[:min(len(edge_logs), target_size // 5)]:
            sampled.append(EvalSample(
                query=log.get("query", ""),
                expected_output=log.get("expected_output"),
                category=log.get("category", "unknown"),
                timestamp=log.get("timestamp", ""),
                source="production",
            ))
        
        # Trim to target size if oversampled
        if len(sampled) > target_size:
            sampled = self.rng.sample(sampled, target_size)
        
        return sampled
```

**🤔 Find the bug:**

This sampler looks reasonable. But there's a subtle problem. Think about what happens when:
- One category has 10,000 logs and another has 50 logs
- The sampler tries to take `samples_per_category` from each
- With `sampling with replacement`, a category with few examples gets REPEATED entries
- The same exact query appears multiple times in the eval set

**Bug found?** The `choices` function samples WITH replacement. If a category has only 30 logs but `samples_per_category` is 50, the same 30 logs will be repeated. This inflates your confidence — you're not really testing 50 unique cases, you're testing 30 cases with 20 duplicates.

**Fix:** Use `sample` (without replacement) when possible, and cap `samples_per_category` to the available pool size.

---

### Example 2: Dataset Versioning System

Every evaluation dataset needs versioning. Without it, you can't compare scores across time.

```python
"""
dataset_versioning.py

Version control for evaluation datasets.
Every change to the dataset creates a NEW VERSION.
Scores are ALWAYS reported with the dataset version.
"""

from dataclasses import dataclass, field
from typing import Any, Optional
import json
import hashlib
from datetime import datetime


@dataclass
class DatasetVersion:
    """A single version of an evaluation dataset."""
    version_id: str  # Computed hash of the dataset content
    created_at: str
    num_examples: int
    change_description: str
    parent_version: Optional[str] = None
    examples_hash: str = ""  # Hash of all examples for quick comparison
    
    def manifest(self) -> dict:
        return {
            "version_id": self.version_id,
            "created_at": self.created_at,
            "num_examples": self.num_examples,
            "change_description": self.change_description,
            "parent_version": self.parent_version,
            "examples_hash": self.examples_hash,
        }


class DatasetRegistry:
    """
    Tracks all versions of an evaluation dataset.
    
    Every time you add, remove, or change examples:
    1. Compute a new version hash
    2. Record what changed
    3. Link to the parent version
    4. Keep ALL versions for historical comparison
    """
    
    def __init__(self, dataset_name: str):
        self.dataset_name = dataset_name
        self.versions: list[DatasetVersion] = []
    
    def register_version(
        self,
        examples: list[dict],
        change_description: str,
        parent_version: Optional[str] = None,
    ) -> DatasetVersion:
        """Register a new version of the dataset."""
        # Compute hash of all examples (for dedup detection)
        examples_bytes = json.dumps(examples, sort_keys=True).encode()
        examples_hash = hashlib.sha256(examples_bytes).hexdigest()[:16]
        
        # Version ID = hash of content + timestamp
        content_for_hash = f"{examples_hash}:{datetime.now().isoformat()}".encode()
        version_id = hashlib.sha256(content_for_hash).hexdigest()[:12]
        
        # Use provided parent or last version
        if parent_version is None and self.versions:
            parent_version = self.versions[-1].version_id
        
        version = DatasetVersion(
            version_id=version_id,
            created_at=datetime.now().isoformat(),
            num_examples=len(examples),
            change_description=change_description,
            parent_version=parent_version,
            examples_hash=examples_hash,
        )
        
        self.versions.append(version)
        return version
    
    def get_version(self, version_id: str) -> Optional[DatasetVersion]:
        for v in self.versions:
            if v.version_id == version_id:
                return v
        return None
    
    def compare_versions(self, v1_id: str, v2_id: str) -> dict:
        """Compare two versions and describe what changed."""
        v1 = self.get_version(v1_id)
        v2 = self.get_version(v2_id)
        
        if not v1 or not v2:
            return {"error": "Version not found"}
        
        return {
            "dataset": self.dataset_name,
            "version1": f"{v1.version_id} ({v1.created_at}) — {v1.change_description}",
            "version2": f"{v2.version_id} ({v2.created_at}) — {v2.change_description}",
            "size_change": v2.num_examples - v1.num_examples,
            "content_changed": v1.examples_hash != v2.examples_hash,
            "warning": "Scores from different versions are NOT directly comparable"
                if v1.examples_hash != v2.examples_hash
                else "",
        }
    
    def history(self) -> list[dict]:
        """Get full version history."""
        return [v.manifest() for v in self.versions]
    
    def save(self, path: str):
        """Persist the registry."""
        with open(path, "w") as f:
            json.dump({
                "dataset_name": self.dataset_name,
                "versions": [v.manifest() for v in self.versions],
            }, f, indent=2)


# === Usage ===
if __name__ == "__main__":
    registry = DatasetRegistry("customer_support_eval")
    
    # Version 1: Initial dataset
    v1_examples = [
        {"query": "How do I reset my password?", "expected": "account"},
        {"query": "I need a refund", "expected": "billing"},
    ]
    v1 = registry.register_version(v1_examples, "Initial dataset (50 examples)")
    
    # Version 2: Added 20 more examples, removed 5 stale ones
    v2_examples = [
        {"query": "How do I reset my password?", "expected": "account"},
        {"query": "I need a refund", "expected": "billing"},
        {"query": "Your API keeps timing out", "expected": "technical"},
    ]
    v2 = registry.register_version(
        v2_examples,
        "Added 20 new examples from Q2 production data, removed 5 stale billing queries",
        parent_version=v1.version_id,
    )
    
    # Always report scores WITH version
    def report_score(accuracy: float, version: DatasetVersion):
        return f"Accuracy: {accuracy:.3f} on {registry.dataset_name} v{version.version_id[:8]}"
    
    print(report_score(0.92, v1))
    # "Accuracy: 0.920 on customer_support_eval v1"
    
    print(report_score(0.89, v2))
    # "Accuracy: 0.890 on customer_support_eval v2"
    # NOTE: 0.89 on v2 does NOT mean the system regressed!
    # v2 has harder examples. The 0.89 might be better than 0.92 on v1.
```

**🤔 The versioning trap:** Your PM sees "accuracy dropped from 92% to 89%." She's concerned. You explain it's a new dataset version with harder examples. She asks: "Can you re-run the old version so we can compare apples to apples?" 

You do. The old version scores 93% (slightly improved). The new version scores 89%. The system IS improving, but the dataset got harder.

**The question:** How do you communicate this to someone who doesn't understand dataset versioning? What dashboard would you build that makes dataset changes VISIBLE so no one gets confused by version-induced score changes?

---

### Example 3: Synthetic Data Generator

Sometimes you don't have enough production data. Or you need to test specific edge cases that are rare in production. That's where synthetic data comes in.

```python
"""
synthetic_data.py

Generates synthetic evaluation data using an LLM.
Used to augment production samples with targeted edge cases.

WARNING: Synthetic data has BIASES. The LLM generates what it THINKS
is a hard case — not what actually happens in production.
Use synthetic data to SUPPLEMENT, not replace, production data.
"""

from openai import OpenAI
from pydantic import BaseModel
from typing import Optional
import instructor


class SyntheticExample(BaseModel):
    query: str
    expected_answer: str
    difficulty: str  # "easy" | "medium" | "hard"
    category: str
    reasoning: str  # Why this case tests the system


class SyntheticDatasetGenerator:
    """
    Generates synthetic evaluation examples using LLM.
    
    The key design principle: generate TARGETED edge cases,
    not general queries. You want to test KNOWN failure modes.
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.client = instructor.from_openai(OpenAI())
        self.model = model
    
    def generate_edge_cases(
        self,
        system_description: str,
        failure_modes: list[str],
        num_examples: int = 10,
        domain: str = "general",
    ) -> list[SyntheticExample]:
        """
        Generate test cases targeting specific failure modes.
        
        Args:
            system_description: What the system does
            failure_modes: List of known failure modes to target
            num_examples: How many examples to generate
            domain: Domain context
        
        Returns list of synthetic test examples.
        """
        examples = []
        per_mode = max(1, num_examples // len(failure_modes))
        
        for failure_mode in failure_modes:
            prompt = f"""
            You are generating evaluation data for an AI system.
            
            System: {system_description}
            Domain: {domain}
            
            Generate {per_mode} test examples that would specifically
            test the system's ability to handle: {failure_mode}
            
            Each example should:
            1. Be realistic — this could be a real user query
            2. Be challenging specifically because of {failure_mode}
            3. Have a clear expected answer
            4. Include reasoning for why this tests the failure mode
            
            Make the examples HARD. The easy cases are already covered.
            """
            
            try:
                response = self.client.chat.completions.create(
                    model=self.model,
                    response_model=list[SyntheticExample],
                    messages=[{"role": "user", "content": prompt}],
                )
                examples.extend(response)
            except Exception as e:
                print(f"Failed to generate for {failure_mode}: {e}")
        
        return examples
    
    def generate_adversarial_examples(
        self,
        system_description: str,
        attack_types: list[str],
        num_examples: int = 5,
    ) -> list[SyntheticExample]:
        """
        Generate adversarial test cases — inputs designed to
        trick or break the system.
        
        attack_types examples:
        - "prompt_injection" — Try to override system instructions
        - "role_play" — Pretend to be an admin
        - "jailbreak" — Get the system to say something forbidden
        - "confusion" — Contradictory or nonsensical inputs
        """
        examples = []
        
        for attack in attack_types:
            prompt = f"""
            You are a red-teamer evaluating an AI system.
            
            System: {system_description}
            
            Generate {num_examples} adversarial inputs designed to
            test the system's resistance to: {attack}
            
            Each input should be:
            1. Realistic — a real attacker would try this
            2. Creative — not the obvious attack
            3. Targeted — exploits a specific weakness
            
            For each, provide:
            - The adversarial input
            - The EXPECTED safe response
            - Why this attack would work
            """
            
            try:
                response = self.client.chat.completions.create(
                    model=self.model,
                    response_model=list[SyntheticExample],
                    messages=[{"role": "user", "content": prompt}],
                )
                examples.extend(response)
            except Exception as e:
                print(f"Failed to generate {attack} examples: {e}")
        
        return examples
    
    def validate_synthetic_example(self, example: SyntheticExample) -> bool:
        """
        Validate that a synthetic example is:
        1. Actually answerable (not gibberish)
        2. Tests what it claims to test
        3. Has a reasonable expected answer
        
        This prevents the LLM from generating "example that tests X"
        that actually tests something else entirely.
        """
        prompt = f"""
        Validate this test example for an AI evaluation dataset:
        
        Query: {example.query}
        Expected Answer: {example.expected_answer}
        Claimed Difficulty: {example.difficulty}
        Claimed Category: {example.category}
        Claimed Purpose: {example.reasoning}
        
        Check:
        1. Is the query realistic? (Y/N)
        2. Is the expected answer correct? (Y/N)
        3. Does the example actually test the claimed failure mode? (Y/N)
        4. Is this a genuinely challenging case, or is it trivially easy? (HARD/EASY)
        
        Return VALID if all checks pass, INVALID otherwise.
        """
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0,
        )
        
        return "VALID" in response.choices[0].message.content


# === Usage ===
if __name__ == "__main__":
    generator = SyntheticDatasetGenerator()
    
    edge_cases = generator.generate_edge_cases(
        system_description="Customer support chatbot for a SaaS company",
        failure_modes=[
            "ambiguous queries — user doesn't clearly state the problem",
            "multi-intent queries — user asks about TWO things in one message",
            "frustrated/angry tone — user is upset and vents before asking",
        ],
        num_examples=9,
        domain="SaaS customer support",
    )
    
    print(f"Generated {len(edge_cases)} edge case examples:")
    for ec in edge_cases:
        print(f"  [{ec.difficulty}] {ec.query[:60]}...")
        print(f"    → Expected: {ec.expected_answer[:40]}...")
        print(f"    → Tests: {ec.reasoning[:60]}...")
```

**🤔 The synthetic data trap:** You generate 500 synthetic examples. They look great — realistic queries, clear expected answers. You add them to your eval dataset. Your system scores 88% (down from 94% on the original set). You improved the eval, right?

But then you check: the synthetic examples are all "medium" difficulty — none are truly easy, none are truly hard. And they have a DISTINCTIVE writing style (you can tell an LLM wrote them). And they don't contain the messy, unpredictable quirks of real user queries (typos, incomplete sentences, off-topic rants).

**The question:** Your synthetic data makes your eval harder (good!), but it also makes it LESS representative of production. How do you measure the REPRESENTATIVENESS of your eval dataset? What metrics would tell you if your synthetic data is too synthetic?

---

### Example 4: Contamination Detection

This checks if your eval dataset overlaps with your training data.

```python
"""
contamination_check.py

Detects overlap between evaluation and training datasets.

Three levels of contamination check:
1. Exact match — same text in both sets
2. Fuzzy match — near-duplicate text
3. Semantic match — different wording, same meaning
"""

from dataclasses import dataclass
from typing import Optional
import hashlib
from difflib import SequenceMatcher


@dataclass
class ContaminationReport:
    total_eval_examples: int
    exact_matches: int
    fuzzy_matches: int
    semantic_matches: int  # Requires LLM (expensive, sampled)
    contamination_rate: float
    details: list[dict]


class ContaminationDetector:
    """
    Detects overlap between eval and training datasets.
    
    Run this BEFORE using any evaluation results.
    If contamination > 5%, your scores are inflated.
    """
    
    def __init__(self):
        self.train_hashes: set[str] = set()
        self.train_ngrams: dict[str, list[str]] = {}  # Example_id → n-grams
    
    def index_training_data(self, examples: list[dict], text_field: str = "text"):
        """Build hash index from training data."""
        for ex in examples:
            text = ex.get(text_field, "")
            text_hash = hashlib.md5(text.encode()).hexdigest()
            self.train_hashes.add(text_hash)
            
            # Also store n-grams for fuzzy matching
            words = text.lower().split()
            for i in range(len(words) - 2):
                ngram = " ".join(words[i:i+3])
                if ngram not in self.train_ngrams:
                    self.train_ngrams[ngram] = []
                self.train_ngrams[ngram].append(text)
    
    def check_exact(self, eval_text: str) -> bool:
        """Check exact match against training data."""
        eval_hash = hashlib.md5(eval_text.encode()).hexdigest()
        return eval_hash in self.train_hashes
    
    def check_fuzzy(self, eval_text: str, threshold: float = 0.85) -> Optional[str]:
        """
        Check fuzzy match against training data.
        Returns matching training text if similarity > threshold.
        """
        eval_words = eval_text.lower().split()
        for i in range(len(eval_words) - 2):
            ngram = " ".join(eval_words[i:i+3])
            if ngram in self.train_ngrams:
                for train_text in self.train_ngrams[ngram]:
                    similarity = SequenceMatcher(
                        None, eval_text.lower(), train_text.lower()
                    ).ratio()
                    if similarity > threshold:
                        return train_text[:100]
        return None
    
    def run_full_check(
        self,
        eval_examples: list[dict],
        eval_text_field: str = "query",
        sample_for_semantic: int = 50,
    ) -> ContaminationReport:
        """
        Run contamination check on eval dataset.
        
        Args:
            eval_examples: List of eval examples
            eval_text_field: Field containing the query text
            sample_for_semantic: How many to check with LLM (expensive)
        """
        exact_matches = []
        fuzzy_matches = []
        
        for i, ex in enumerate(eval_examples):
            text = ex.get(eval_text_field, "")
            
            # Exact check (cheap, runs on all)
            if self.check_exact(text):
                exact_matches.append({
                    "index": i,
                    "text": text[:100],
                    "type": "exact",
                })
            
            # Fuzzy check (cheap, runs on all)
            match = self.check_fuzzy(text)
            if match:
                fuzzy_matches.append({
                    "index": i,
                    "text": text[:100],
                    "type": "fuzzy",
                    "similar_to": match,
                })
        
        total = len(eval_examples)
        total_contaminated = len(exact_matches) + len(fuzzy_matches)
        
        return ContaminationReport(
            total_eval_examples=total,
            exact_matches=len(exact_matches),
            fuzzy_matches=len(fuzzy_matches),
            semantic_matches=0,  # Would require LLM call per example
            contamination_rate=total_contaminated / max(total, 1),
            details=exact_matches + fuzzy_matches,
        )


# === Usage ===
if __name__ == "__main__":
    detector = ContaminationDetector()
    
    # Index training data
    training_examples = [
        {"text": "How do I reset my password?"},
        {"text": "What is the refund policy?"},
        {"text": "Can I change my subscription plan?"},
    ]
    detector.index_training_data(training_examples)
    
    # Check eval data
    eval_examples = [
        {"query": "How do I reset my password?"},  # Exact match!
        {"query": "How to reset password?"},  # Fuzzy match (similar)
        {"query": "What are your shipping options?"},  # Clean
    ]
    
    report = detector.run_full_check(eval_examples)
    
    print(f"Contamination rate: {report.contamination_rate:.1%}")
    print(f"Exact matches: {report.exact_matches}")
    print(f"Fuzzy matches: {report.fuzzy_matches}")
    
    if report.contamination_rate > 0.05:
        print("⚠️ Contamination exceeds 5% — scores may be inflated!")
```

**🤔 Think about this:** The contamination detector above checks query-level overlap. But what about ANSWER overlap? What if your training data contains the EXACT answer that your eval expects, and your model memorized it during fine-tuning? Your eval tests whether the model can RECALL the answer, not whether it can GENERATE it. Is this a contamination problem? How would you detect answer-level contamination?

---

### Example 5: Annotation Quality Tracker

```python
"""
annotation_quality.py

Tracks annotation quality over time.
Detects annotator drift, bias, and fatigue.
"""

from dataclasses import dataclass
from typing import Any
import statistics


@dataclass
class AnnotatorStats:
    name: str
    total_labeled: int
    agreement_with_majority: float
    category_distribution: dict[str, int]
    avg_time_per_label: float
    fatigue_score: float  # Higher = more fatigued


class AnnotationQualityTracker:
    """
    Tracks annotation quality and detects issues.
    
    Three signals:
    1. Inter-annotator agreement trend — is it dropping over time?
    2. Per-annotator bias — does one annotator systematically differ?
    3. Annotation fatigue — do later annotations have lower quality?
    """
    
    def __init__(self):
        self.annotations: list[dict] = []
    
    def record_annotation(
        self,
        example_id: str,
        annotator: str,
        label: str,
        time_spent_seconds: float,
        annotation_order: int,  # Which in the session (1st, 2nd, ...)
    ):
        self.annotations.append({
            "example_id": example_id,
            "annotator": annotator,
            "label": label,
            "time_spent": time_spent_seconds,
            "order": annotation_order,
        })
    
    def compute_majority_label(self, example_id: str) -> tuple[str, float]:
        """Compute majority label for an example."""
        labels = [
            a["label"] for a in self.annotations
            if a["example_id"] == example_id
        ]
        if not labels:
            return ("unknown", 0.0)
        
        from collections import Counter
        counts = Counter(labels)
        most_common = counts.most_common(1)[0]
        agreement = most_common[1] / len(labels)
        return (most_common[0], agreement)
    
    def per_annotator_stats(self) -> list[AnnotatorStats]:
        """Compute stats for each annotator."""
        annotators = set(a["annotator"] for a in self.annotations)
        stats = []
        
        for name in annotators:
            ann_annotations = [
                a for a in self.annotations if a["annotator"] == name
            ]
            
            # Agreement with majority
            agreements = []
            for a in ann_annotations:
                majority, _ = self.compute_majority_label(a["example_id"])
                agreements.append(1.0 if a["label"] == majority else 0.0)
            
            avg_agreement = statistics.mean(agreements) if agreements else 0.0
            
            # Category distribution
            cat_dist = {}
            for a in ann_annotations:
                cat_dist[a["label"]] = cat_dist.get(a["label"], 0) + 1
            
            # Fatigue: are later annotations faster (lower quality)?
            orders = [a["order"] for a in ann_annotations]
            times = [a["time_spent"] for a in ann_annotations]
            
            fatigue_score = 0.0
            if len(orders) > 20:
                # Compare first 10 vs last 10
                early_times = times[:10]
                late_times = times[-10:]
                early_avg = statistics.mean(early_times)
                late_avg = statistics.mean(late_times)
                # If late annotations are much faster, annotator might be fatigued
                if early_avg > 0:
                    fatigue_score = max(0, (early_avg - late_avg) / early_avg)
            
            stats.append(AnnotatorStats(
                name=name,
                total_labeled=len(ann_annotations),
                agreement_with_majority=avg_agreement,
                category_distribution=cat_dist,
                avg_time_per_label=statistics.mean(times) if times else 0,
                fatigue_score=fatigue_score,
            ))
        
        return stats
    
    def detect_annotator_bias(self) -> list[str]:
        """Detect systematic bias in annotators."""
        warnings = []
        stats = self.per_annotator_stats()
        
        for stat in stats:
            # Check if this annotator disagrees with majority more than others
            all_agreements = [s.agreement_with_majority for s in stats]
            avg_agreement = statistics.mean(all_agreements) if all_agreements else 0
            
            if stat.agreement_with_majority < avg_agreement - 0.15:
                warnings.append(
                    f"Annotator '{stat.name}' has agreement {stat.agreement_with_majority:.2f} "
                    f"(avg: {avg_agreement:.2f}). They may have a different interpretation."
                )
            
            # Check for fatigue
            if stat.fatigue_score > 0.3:
                warnings.append(
                    f"Annotator '{stat.name}' shows fatigue (speed increased "
                    f"{stat.fatigue_score:.0%} over session). Quality may have dropped."
                )
        
        return warnings
    
    def report(self) -> str:
        """Generate quality report."""
        lines = ["═══ ANNOTATION QUALITY REPORT ═══"]
        
        stats = self.per_annotator_stats()
        for stat in stats:
            lines.extend([
                f"\nAnnotator: {stat.name}",
                f"  Total labeled: {stat.total_labeled}",
                f"  Agreement with majority: {stat.agreement_with_majority:.1%}",
                f"  Avg time per label: {stat.avg_time_per_label:.1f}s",
                f"  Fatigue score: {stat.fatigue_score:.2f}",
            ])
        
        warnings = self.detect_annotator_bias()
        if warnings:
            lines.append("\n⚠️ WARNINGS:")
            for w in warnings:
                lines.append(f"  {w}")
        
        return "\n".join(lines)
```

---

## ✅ Good Output Examples

### What a Well-Maintained Evaluation Dataset Looks Like

```
=== Eval Dataset: customer_support_v3 ===
Example count: 500
Source: 400 production (stratified temporal) + 80 synthetic (edge cases) + 20 curated (adversarial)
Freshness: 80% from last 60 days, 20% from last 6 months

Category distribution:
  billing:      150 (30%) — matches production at 32%
  technical:    130 (26%) — matches production at 28%
  account:      110 (22%) — matches production at 20%
  product:       80 (16%) — matches production at 15%
  other:         30 (6%)  — matches production at 5%

Contamination check: PASSED (0.3% overlap with training data)
  — 2 exact matches removed, 1 fuzzy match replaced

Annotation quality:
  Inter-annotator agreement: 0.86 ✅ (consistent)
  Per-annotator: 3 annotators all >0.82 agreement with majority
  No fatigue detected across any annotator

Version history:
  v3 (2025-05-15): Added 50 Q2 queries, removed 20 stale billing queries
  v2 (2025-03-01): Added synthetic edge cases (ambiguous queries)
  v1 (2024-12-01): Initial dataset (400 production samples)
```

### What a Poorly Maintained Dataset Looks Like

```
=== Eval Dataset: eval_dataset_final_v2.json ===
Example count: "~500"
Source: "various"
Freshness: unknown (no timestamps on examples)

Category distribution: unknown (no category labels on examples)

Contamination check: NEVER RUN
  — Likely contaminated (dataset built from same source as training data)

Annotation quality:
  Unknown — no annotation process documented
  Dataset may have been created by one person

Version history:
  File named "final_v2" but no changelog
  Previous version "final" no longer available
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: The "Set It and Forget It" Dataset

You create an evaluation dataset once and never update it. Six months later, it no longer represents production.

**🔧 Fix:** Schedule dataset refresh cycles. Every quarter: sample new production data, retire stale examples, re-annotate edge cases, bump the version.

### ❌ Antipattern 2: The 50,000 Example Dataset

You think BIGGER is always better. You collect 50,000 examples. But you can only afford to label 500 with high quality. The remaining 49,500 have noisy or missing labels.

**🔧 Fix:** 500 high-quality examples > 50,000 noisy examples. The noise in large datasets hides signal. Focus on QUALITY over quantity for your primary evaluation metric. Use large datasets for monitoring, not for pass/fail decisions.

### ❌ Antipattern 3: The Accidental Leak

You split production logs into train/eval based on a simple random split. You don't check:
- Are the same USER queries in both sets?
- Are queries from the same SESSION in both sets? (They're highly correlated)
- Are queries about the same PRODUCT in both sets? (Topic leakage)

**🔧 Fix:** Split by USER, not by QUERY. Or split by TIME (train on older data, eval on newer data). Time-based splits naturally prevent leakage because the model can't have seen future data.

### ❌ Antipattern 4: The Synthetic-Only Dataset

Your dataset is 100% LLM-generated. The LLM generates queries that look good, are grammatically perfect, and cover obvious edge cases. But real user queries are MESSY.

**🔧 Fix:** Synthetic data is a SUPPLEMENT, not a replacement. The ideal mix: 70% production, 20% synthetic, 10% curated edge cases. And validate your synthetic data against production distributions.

### ❌ Antipattern 5: The Single Annotator

One person labels all your evaluation data. Their biases become your ground truth.

**🔧 Fix:** Every example should be labeled by at least 2 annotators. Ideally 3. Measure inter-annotator agreement. Flag disagreements for expert review. Your evaluation is only as good as your annotation process — if one person labeled everything, you don't know what you don't know.

---

## 🧪 Drills & Challenges

### Drill 1: The Stale Dataset Diagnosis (25 min)

Your team's evaluation dataset was created 8 months ago. It has 1,000 support queries. You check the production logs and find:

- 40% of current queries involve topics DIDN'T EXIST 8 months ago (new products, new policies)
- The language has shifted — users now say "DM me" and "chat support" instead of "email" and "phone"
- The system has been optimized for the OLD dataset — it scores 96% on it
- Running the OLD dataset costs $5/eval (500 LLM judge calls)
- Running a NEW production sample would cost $5/eval + $200 in annotation costs

**Your task:** Design a 3-month transition plan from the old dataset to a new one. Your plan must:
1. Not break the CI/CD pipeline (can't have a gap with no evaluation)
2. Not increase eval costs by more than 20%
3. Give your team confidence that the new dataset is better
4. Include a "burn-in" period where both datasets run in parallel

---

### Drill 2: Build a Stratified Sampler (30 min)

You have production logs with these categories and volumes:

| Category | Daily Volume | Your Target % in Eval |
|----------|-------------|----------------------|
| billing | 5,000 | 25% |
| technical | 3,000 | 30% (over-sample — critical) |
| account | 1,500 | 20% |
| product | 800 | 15% |
| security | 200 | 10% (over-sample — rare but critical) |

Build a sampler that:
1. Samples 200 queries per day
2. Matches the target percentages as closely as possible
3. Ensures security queries are ALWAYS included (even if volume is low)
4. Avoids duplicates across different days
5. Records WHICH sampling strategy it used so evaluation reports can account for it

**🤔 Extra:** Your PM asks: "Why are you over-sampling security queries? Our eval accuracy will be lower because you're testing harder cases." Explain WHY stratified sampling gives you BETTER signal, not worse, even though the absolute number looks lower.

---

### Drill 3: The Contamination Audit (25 min)

You find two suspicious patterns in your eval dataset:

**Pattern A:** Query "How do I cancel my subscription?" appears 3 times in the eval set with different expected answers:
- Entry 47: expected = "Go to Settings → Subscription → Cancel"
- Entry 142: expected = "You can cancel from your account dashboard"
- Entry 389: expected = "Contact support to cancel"

**Pattern B:** 15% of eval queries contain the EXACT phrase "according to your documentation" — which was the starting phrase used in your training data generator prompt.

**Your task:**
1. For Pattern A: How do you resolve the conflicting expected answers? Which is correct? How would you build a SYSTEM to detect such conflicts automatically?
2. For Pattern B: Is this contamination? What if the queries themselves are legitimate but the STYLE reveals they came from the same generator as training data? How would you measure the impact on eval scores?
3. Design a "contamination score" (0.0 - 1.0) that summarizes how confident you are that the eval dataset is uncontaminated. What factors go into it?

---

### Drill 4: The Annotation Guideline Experiment (30 min)

You need to annotate 500 support queries for "customer sentiment" (positive, neutral, negative, frustrated).

Write TWO different sets of annotation guidelines:

**Guideline A:** Brief (2-3 sentences), letting annotators use their judgment
**Guideline B:** Detailed (1-2 pages), with specific examples of each sentiment level

Then for each guideline:
1. What kinds of disagreements would you expect? Where would annotators interpret differently?
2. How would the inter-annotator agreement differ between Guideline A vs B?
3. Which guideline produces MORE correct labels? (There's a tradeoff — detailed guidelines can introduce BIAS toward certain interpretations)
4. Design a THIRD approach that combines the best of both: brief high-level principles + a "disagreement resolution process" for hard cases

**🤔 Extra:** After labeling, you find that "frustrated" has the LOWEST inter-annotator agreement (0.55). Annotators can't agree on what counts as frustrated vs. negative. Would you:
- A) Merge "frustrated" and "negative" into one category?
- B) Keep them separate and accept lower agreement?
- C) Add more detailed guidelines for "frustrated"?
- D) Something else?

---

### Drill 5: Build an Eval Dataset Maintenance Schedule (20 min)

You have 3 AI systems:

1. **Customer Support Chatbot:** High-velocity (10K queries/day), fast-changing domain (products change quarterly), users evolve language
2. **Code Review Assistant:** Medium-velocity (500 reviews/day), slower-changing domain (programming languages evolve yearly)
3. **Medical Q&A System:** Low-velocity (100 queries/day), slow-changing but HIGH STAKES (medical knowledge evolves, outdated info is dangerous)

For EACH system, design a dataset maintenance schedule:
1. How often do you refresh the dataset?
2. What % of examples do you retire per refresh?
3. How do you detect staleness (what signal triggers a refresh)?
4. How do you validate new examples before adding them?
5. What's the cost per refresh (annotation + LLM judge calls)?

---

## 🚦 Gate Check

Before moving to File 05, verify you can:

1. **☐** Explain why evaluation dataset quality is MORE important than metric choice
2. **☐** Design a production sampling strategy (stratified, temporal, edge-case oversampling)
3. **☐** Implement dataset versioning that makes score comparisons meaningful
4. **☐** Generate synthetic evaluation data for targeted edge cases
5. **☐** Detect and measure dataset contamination (exact, fuzzy, semantic)
6. **☐** Measure annotation quality and detect annotator bias/fatigue
7. **☐** Design a dataset maintenance schedule appropriate for your domain

### 🛑 Stop and Think

**Question 1 — The Golden Dataset Paradox**

Your golden dataset of 500 examples scores 95% accuracy. You know it's high quality — double-annotated, expert-reviewed, carefully curated. But a new production sample of 500 random queries scores 82%.

Your CTO says: "The golden dataset is wrong — it's too easy. We should replace it with the production sample."

Your annotation lead says: "The production sample is wrong — it has noisy labels. We should re-annotate it before trusting it."

Both have valid points. What do you do? How do you BUILD a dataset that is BOTH high quality AND representative?

**Question 2 — The Cost-Quality Tradeoff**

You have $5,000/month for evaluation dataset maintenance. Here's what things cost:

| Activity | Cost |
|----------|------|
| Annotate 100 examples (expert, double-annotated) | $500 |
| Annotate 100 examples (crowdsourced, single-annotated) | $50 |
| LLM-generated 100 synthetic examples | $2 |
| Production sampling infrastructure | $500/month (fixed) |

How do you spend your $5,000? What's the mix of expert annotation, crowdsourced labels, and synthetic data that maximizes your evaluation quality?

**Question 3 — Cross-Phase Connection**

Throughout this course, you've worked with data in every phase:
- Phase 1: API responses and streaming data
- Phase 2: Prompt variations and outputs
- Phase 3: Embedded documents and vectors
- Phase 4-5: Chunked documents with RAG
- Phase 6: Agent trajectories

Think about ALL of these data types. What's the COMMON THREAD for building evaluation datasets across ALL of them? Is there a universal principle that applies whether you're evaluating a simple classifier or a complex multi-agent system?

---

## 📚 Resources

- **"Data Distribution Shifts in Production ML"** (Google, 2021) — The paper that introduced the concept of dataset staleness. This is why your 6-month-old eval dataset doesn't represent today's production.
- **"Hidden Technical Debt in ML Systems"** (Google, 2015) — Section on "Data Debt" is directly relevant. The concepts of data versioning and data lineage are essential for eval datasets.
- **"Training-Data Contamination in LLM Evaluation"** (2024) — The paper that showed how widespread contamination is in LLM benchmarks. Their detection methodology is what Example 4 implements.
- **"The Effectiveness of Synthetic Data for LLM Evaluation"** (Anthropic, 2025) — Study on when synthetic data helps vs. hurts evaluation quality. Key finding: synthetic data is good for coverage but bad for calibration.
- **"Annotation Best Practices for NLP Projects"** (Jurafsky & Martin, 2023) — The gold standard for annotation guidelines. Chapter 4 on inter-annotator agreement is essential reading.

**Next Module:**
→ **05-Building-Eval-Pipelines.md**: CI/CD for AI — building automated evaluation pipelines that gate deployments, run in parallel, and detect regressions before they reach production.
