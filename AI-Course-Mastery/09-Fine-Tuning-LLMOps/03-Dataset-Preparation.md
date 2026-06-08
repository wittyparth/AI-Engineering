# 03 — Dataset Preparation: The Most Important Step

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about data formats and cleaning scripts, you need to understand the single most important truth about fine-tuning:

**Your model is only as good as your data.**

Not your LoRA rank. Not your learning rate. Not your number of epochs. YOUR DATA.

I've seen fine-tuning runs where:
- Bad data → 62% accuracy (worse than prompting)
- Same model, same config, good data → 94% accuracy

The difference wasn't the training. It was the DATA.

Here's what happens with bad data:
- **Duplicate examples**: Model memorizes, doesn't generalize
- **Inconsistent labels**: Model learns the average of conflicting answers (wrong for everyone)
- **Wrong format**: Model learns the wrong output structure
- **Spurious correlations**: Model learns "if input contains 'contract' → output starts with 'Sure!'" because 90% of contract examples start with "Sure!"
- **Distribution mismatch**: Model learns your curated examples, fails on real user language

But "good data" is HARDER than it sounds. You can't just collect examples and clean them. You need to understand WHAT the model needs to learn, and build data that teaches that.

---

### 🤔 Discovery Question 1: The Duplicate Disaster

You collect 10,000 examples for fine-tuning. You deduplicate — remove exact duplicates. Down to 9,500 examples.

Your model trains. Eval loss: 1.2. Accuracy: 88%. Looks great.

In production: 67% accuracy.

You investigate. You find:
- 500 examples are NEAR-duplicates (same question, slightly different wording)
- The model memorized the answer pattern from 500 variations of the SAME question
- When users ask the question in a DIFFERENT way (not in your 500 variations), the model fails

**🤔 Before reading on, answer these:**

1. Exact deduplication caught EXACT duplicates. But NEAR-duplicates (same question with different wording) passed through. How many "unique" questions do you actually have after removing near-duplicates? If 500 variants of one question are in the dataset, the model spends 5% of its training on ONE concept — that's severe imbalance.

2. How do you define "near-duplicate"? Two questions:
   - "How do I reset my password?"
   - "What's the process for password reset?"
   
   Are these duplicates? They ask the same thing with different words. Should BOTH be in the dataset? (They teach the same pattern — the model doesn't need both. But they also teach lexical diversity — the model learns different phrasings.)

3. **The hardest question:** You have 500 near-duplicates of "password reset" and 10 examples of "API key rotation." The model becomes GREAT at password reset (95%) and TERRIBLE at API key rotation (55%). Your production traffic is 20% password reset and 20% API key rotation. **How do you balance the dataset for PRODUCTION performance, not training performance?** Do you undersample password reset (and lose good training data) or oversample API key rotation (and repeat examples)?

---

### 🤔 Discovery Question 2: The Label Inconsistency Trap

You have 3 subject matter experts labeling 2,000 examples each (6,000 total). Each label includes: "correctness" (1-5) and "preferred response."

You combine all 6,000 examples and train. Accuracy: 74%.

You check label consistency:
- Expert A: average score 4.2, prefers detailed technical answers
- Expert B: average score 3.8, prefers concise practical answers  
- Expert C: average score 4.5, prefers analogies and examples

**🤔 Before reading on, answer these:**

1. For the SAME input question, Expert A prefers a technical answer (score 4) and Expert B prefers a concise answer (score 3). Both are "correct" by their standards. But the model sees: input → technical answer and input → concise answer. It learns the AVERAGE of both: a mediocre answer that's neither technical enough for A nor concise enough for B. How do you resolve labeling disagreements?

2. If you use AVERAGE of all labels as the "correct" answer, you get a BLAND model that satisfies no one. If you pick ONE expert's labels, you get a model that matches that expert's style but may not generalize. What's the right approach?

3. **The hardest question:** The experts don't just DISAGREE on style — they disagree on FACTS. For the question "What's the best way to handle refunds?" Expert A says "Process within 5 business days" (company policy), Expert B says "Process immediately" (customer-first philosophy). These are CONTRADICTORY. **The model learns: sometimes refunds take 5 days, sometimes they're immediate. It produces inconsistent answers in production.** How do you detect and resolve factual contradictions in your training data?

---

### 🤔 Discovery Question 3: The Format Mismatch

You collect data from your existing customer support system. Each example is:
```
User: How do I reset my password?
Agent: Go to Settings → Account → Reset Password → Follow the email instructions.
```

You format this for fine-tuning:
```
<|user|>
How do I reset my password?
<|assistant|>
Go to Settings → Account → Reset Password → Follow the email instructions.
```

You train. The model learns to answer like your best support agents. Eval: 91%.

In production, users ask differently:
```
"can't login forgot password help!!!" (real user)
vs.
"How do I reset my password?" (training data)
```

The model fails on real user language.

**🤔 Before reading on, answer these:**

1. Your training data is CLEAN, POLITE, and WELL-FORMATTED (written by your support team). Your production data is MESSY, IMPOLITE, and POORLY-FORMATTED (written by real users). Your model learned the TEAM's language, not the USERS' language. How do you build a training set that generalizes to messy real-world inputs?

2. One approach: Include REAL user queries (messy) with the agent responses. But real user data might contain PII, offensive language, or incomplete queries. Your Phase 5 PII redaction can clean it, but what about the offensive language? Should you include a user swearing in the training data?

3. **The hardest question:** Your production data IS your best training data — it shows what REAL users actually ask. But you don't HAVE production data before you launch. You're building a model for a NEW feature that doesn't exist yet. **How do you generate realistic training data for a system that hasn't launched?** How do you predict what users will actually say?

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 30 min |
| Data Collection Strategies | 20 min |
| Deduplication (Exact + Semantic) | 25 min |
| Quality Filtering & Scoring | 25 min |
| Code: Data Cleaning Pipeline | 35 min |
| Code: Quality Scorer | 30 min |
| Code: Synthetic Data Generation | 30 min |
| Train/Eval/Test Splitting (Advanced) | 20 min |
| Formatting for Different Models | 15 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 50 min |
| Gate Check | 15 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Data Quality Hierarchy

Not all data problems are equal. Prioritize in this order:

```
LEVEL 1: CORRECTNESS (MUST HAVE)
  └── Are the answers FACTUALLY correct?
  └── Are there factual contradictions?
  └── If not, the model learns WRONG things
  └── Fix: Expert review, cross-reference with sources

LEVEL 2: CONSISTENCY (MUST HAVE)
  └── Do different examples of the same type have the SAME format?
  └── Do multiple experts agree on the answer?
  └── If not, the model learns the AVERAGE (mediocre for everyone)
  └── Fix: Style guide, consensus labeling, expert arbitration

LEVEL 3: COVERAGE (SHOULD HAVE)
  └── Do you cover the FULL range of user inputs?
  └── Are edge cases represented?
  └── If not, the model fails on unseen inputs
  └── Fix: Diverse sourcing, synthetic generation, gap analysis

LEVEL 4: BALANCE (SHOULD HAVE)
  └── Is each type of question proportionally represented?
  └── Are rare-but-important cases oversampled?
  └── If not, the model over-optimizes for common cases
  └── Fix: Stratified sampling, importance weighting

LEVEL 5: DIVERSITY (NICE TO HAVE)
  └── Are there multiple phrasings of the same question?
  └── Is there lexical and stylistic variety?
  └── If not, the model is brittle to rephrasing
  └── Fix: Paraphrase generation, multi-source collection
```

### 2. The Dataset Size Myth

How many examples do you NEED?

The answer isn't a number. It's: **Enough to cover the VARIATION in your task.**

| Task Type | Min Examples | Typical | Why |
|-----------|-------------|---------|-----|
| Format change (JSON → Markdown) | 50 | 200-500 | Low variation — just structure |
| Style transfer (formal → casual) | 100 | 500-2K | Moderate variation — tone and word choice |
| Classification (intent detection) | 500 | 2-5K per class | Each class needs enough variation |
| Generation (summarization, QA) | 1K | 5-20K | High variation — content, length, style |
| Complex reasoning (multi-step) | 2K | 10-50K | Very high variation — reasoning paths |

**The key insight:** It's not about the NUMBER of examples. It's about whether your examples COVER the full distribution of inputs the model will see in production.

Better question: How many UNIQUE input patterns exist? If you need the model to handle 50 question types, you need ~50-100 examples per type = 2,500-5,000 examples. If you have 5 question types, 250-500 is enough.

### 3. Deduplication: Exact vs. Semantic

```python
# Why deduplication matters:
# 10,000 examples with 2,000 duplicates = effectively 8,000 unique examples
# The model spends 20% of training on REPEATED content
# It memorizes the repeated patterns, doesn't generalize

# Exact dedup (easy, not enough):
seen = set()
unique = []
for example in data:
    key = example["text"]  # Exact match
    if key not in seen:
        seen.add(key)
        unique.append(example)

# Semantic dedup (hard, necessary):
# Two questions that ask the same thing with different words
# "Reset password" vs "Change my password" vs "Forgot password recovery"
# These don't need to ALL be in the training set
```

### 4. Train/Eval/Test Splitting — Advanced

The standard 80/10/10 split is WRONG for fine-tuning in most cases.

**Problem 1: Temporal leakage**
```python
# BAD: Random split
all_data = load_all_data()  # Mixed time periods
random.shuffle(all_data)
train = all_data[:8000]     # Has 2020 AND 2024 data
eval = all_data[8000:9000]   # Also has 2020 AND 2024 data
# The eval set has examples from the SAME period as training
# → eval accuracy is INFLATED (doesn't measure temporal generalization)

# GOOD: Temporal split
train = all_data[:8000]      # 2020-2023 data
eval = all_data[8000:9000]   # Early 2024 data (future from training)
test = all_data[9000:]       # Late 2024 data (more future)
# Eval measures: Can the model generalize to the FUTURE?
```

**Problem 2: Labeler leakage**
```python
# BAD: Random split with multiple labelers
# Same labeler's examples in both train and eval
# Eval accuracy reflects "can the model reproduce Labeler A's style?"

# GOOD: Split by labeler
labeler_a_data = [e for e in data if e["labeler"] == "A"]
labeler_b_data = [e for e in data if e["labeler"] == "B"]
# Train on A, eval on B
# Measures: Can the model generalize to a DIFFERENT labeler's style?
```

---

## 💻 CODE EXAMPLES

### Example 1: The Naive Data Pipeline (Full of Bugs)

```python
# BAD: Every common data preparation mistake

import json
import random

# Load data — assume it's all good
with open("training_data.json") as f:
    data = json.load(f)

# PROBLEM 1: No data inspection
# What does the data look like? How many fields?
# Are there missing values? Empty strings? NULLs?
# We don't check ANY of this.

# PROBLEM 2: Random shuffle with no seed
# If you run this script twice, you get DIFFERENT splits
# Experiments are NOT reproducible
random.shuffle(data)

# PROBLEM 3: No test set
# Only "train" and "eval" — no held-out test set
# You'll tune hyperparameters to eval set performance
# And wonder why production is worse
split_idx = int(len(data) * 0.9)
train_data = data[:split_idx]
eval_data = data[split_idx:]

# PROBLEM 4: No quality filtering
# What if some examples have empty responses?
# What if some are just "I don't know"?
# They go straight into training and teach the model to say "I don't know"
print(f"Training on {len(train_data)} examples")

# PROBLEM 5: No deduplication
# If the same question appears 10 times, the model sees it 10 times
# With 3 epochs, that's 30 repetitions of the SAME example
# → Model memorizes the exact wording, can't generalize

# PROBLEM 6: No length check
# Some examples may be 10,000 tokens long → OOM during training
# Some may be 0 tokens → wastes computation, causes errors
# We don't check either.

# PROBLEM 7: No format validation
# Does the data use the correct chat template?
# Does it have the right fields?
# If data format changed between collection and training, silent failure

# Problem 8: No label consistency check
# Are the labels consistent across examples?
# Are there contradictions?
# We don't check

print("Data prepared! Ready for training.")
```

### 🔍 Find The Bugs

1. **The reproducibility bug**: `random.shuffle(data)` without a seed means every run produces different train/eval splits. Why is reproducibility critical for fine-tuning?

2. **The missing test set**: This creates train and eval sets but no FINAL held-out test set. What's wrong with using the eval set for both hyperparameter tuning AND final evaluation?

3. **The quality filter gap**: Empty responses, "I don't know" answers, and nonsensical examples all go into training. What does the model learn from these?

---

### Example 2: Production Data Pipeline (Fixed)

```python
"""
Production Data Pipeline for Fine-Tuning

What this does RIGHT:
1. Data inspection and quality metrics
2. Near-deduplication (semantic similarity)
3. Train/eval/test split with temporal + labeler awareness
4. Quality scoring and filtering
5. Format validation and standardization
6. Reproducible splits with seed control
7. Length checking and padding
8. Label consistency analysis
"""

import json
import random
import hashlib
from collections import Counter, defaultdict
from typing import Optional


class DataQualityReport:
    """Analyze dataset quality before training."""
    
    def __init__(self, data: list[dict]):
        self.data = data
        self.report = {}
    
    def generate(self) -> dict:
        """Generate comprehensive quality report."""
        self.report["total_examples"] = len(self.data)
        self._check_missing_values()
        self._check_lengths()
        self._check_label_consistency()
        self._check_duplicates()
        self._check_format_validity()
        self._check_class_balance()
        return self.report
    
    def _check_missing_values(self):
        """Find examples with missing or empty fields."""
        missing = 0
        for i, example in enumerate(self.data):
            if not example.get("instruction") or not example.get("response"):
                missing += 1
        
        self.report["missing_values"] = missing
        if missing > 0:
            self.report["warnings"].append(
                f"⚠️ {missing} examples with missing fields ({(missing/len(self.data))*100:.1f}%)"
            )
    
    def _check_lengths(self):
        """Analyze token length distribution."""
        lengths = []
        for example in self.data:
            text = f"User: {example.get('instruction', '')}\nAssistant: {example.get('response', '')}"
            # Approximate token count (4 chars per token)
            lengths.append(len(text) // 4)
        
        self.report["length_stats"] = {
            "min_tokens": min(lengths),
            "max_tokens": max(lengths),
            "avg_tokens": sum(lengths) / len(lengths),
            "p95_tokens": sorted(lengths)[int(len(lengths) * 0.95)],
        }
        
        if self.report["length_stats"]["max_tokens"] > 2048:
            self.report["warnings"].append(
                f"⚠️ Max length {self.report['length_stats']['max_tokens']} tokens — "
                f"may cause OOM. Consider truncating to 2048."
            )
    
    def _check_label_consistency(self):
        """Check for label conflicts (same input, different output)."""
        input_to_outputs = defaultdict(set)
        for example in self.data:
            key = example.get("instruction", "").strip().lower()
            output = example.get("response", "").strip()
            input_to_outputs[key].add(output)
        
        conflicts = {
            k: list(v) for k, v in input_to_outputs.items()
            if len(v) > 1
        }
        
        self.report["label_conflicts"] = len(conflicts)
        if len(conflicts) > 0:
            self.report["warnings"].append(
                f"⚠️ {len(conflicts)} inputs have multiple DIFFERENT outputs "
                f"({(len(conflicts)/len(input_to_outputs))*100:.1f}% of unique inputs)"
            )
            # Show first 5 conflicts
            self.report["example_conflicts"] = [
                {"input": k, "outputs": list(v)[:3]}
                for k, v in list(conflicts.items())[:5]
            ]
    
    def _check_duplicates(self):
        """Check for exact and near-duplicate examples."""
        inputs = [e.get("instruction", "").strip().lower() for e in self.data]
        exact_dupes = len(inputs) - len(set(inputs))
        
        self.report["exact_duplicates"] = exact_dupes
        if exact_dupes > 0:
            self.report["warnings"].append(
                f"⚠️ {exact_dupes} exact duplicate inputs found "
                f"({(exact_dupes/len(self.data))*100:.1f}%)"
            )
    
    def _check_format_validity(self):
        """Check that all examples have required fields."""
        required_fields = ["instruction", "response"]
        valid = 0
        for example in self.data:
            if all(f in example and example[f] for f in required_fields):
                valid += 1
        
        self.report["format_valid"] = valid
        self.report["format_invalid"] = len(self.data) - valid
    
    def _check_class_balance(self):
        """Check distribution of data across categories (if available)."""
        categories = Counter()
        for example in self.data:
            cat = example.get("category", "uncategorized")
            categories[cat] += 1
        
        self.report["categories"] = dict(categories.most_common())
        if len(categories) > 1:
            counts = list(categories.values())
            max_cat = max(counts)
            min_cat = min(counts)
            if max_cat > min_cat * 10:  # 10x imbalance
                self.report["warnings"].append(
                    f"⚠️ Severe class imbalance: max={max_cat}, min={min_cat} "
                    f"(ratio: {max_cat/min_cat:.1f}x)"
                )


class DataPreprocessor:
    """
    Prepare data for fine-tuning.
    
    Flow:
    1. Load and inspect
    2. Quality filter (remove bad examples)
    3. Deduplicate (exact + semantic)
    4. Split (train/eval/test)
    5. Format for model
    6. Validate final output
    """
    
    def __init__(self, seed: int = 42):
        self.seed = seed
        random.seed(seed)
    
    def load_and_inspect(self, filepath: str) -> list[dict]:
        """Load data and generate quality report."""
        with open(filepath) as f:
            data = json.load(f)
        
        print(f"Loaded {len(data)} examples")
        
        report = DataQualityReport(data).generate()
        
        if report.get("warnings"):
            print("\n⚠️ QUALITY WARNINGS:")
            for w in report["warnings"]:
                print(f"  {w}")
        
        return data
    
    def quality_filter(self, data: list[dict], 
                        min_response_length: int = 10,
                        max_response_length: int = 4000) -> list[dict]:
        """
        Remove low-quality examples.
        
        What gets removed:
        - Empty or too-short responses ("OK", "I don't know", blank)
        - Too-long responses (will cause OOM)
        - Examples with missing fields
        """
        before = len(data)
        
        filtered = []
        for example in data:
            response = example.get("response", "")
            
            # Check response length
            if len(response) < min_response_length:
                continue
            if len(response) > max_response_length:
                continue
            
            # Check for required fields
            if not example.get("instruction"):
                continue
            
            filtered.append(example)
        
        after = len(filtered)
        print(f"Quality filter: {before} → {after} "
              f"(removed {before-after}, {((before-after)/before)*100:.1f}%)")
        
        return filtered
    
    def deduplicate(self, data: list[dict]) -> list[dict]:
        """
        Remove exact and near-duplicate examples.
        
        Strategy:
        1. Remove exact duplicates (same instruction)
        2. Remove near-duplicates (similar instruction)
        
        For near-duplicates, keep the HIGHEST quality version.
        """
        # Step 1: Exact dedup by instruction
        seen_inputs = set()
        unique = []
        
        for example in sorted(data, key=lambda x: len(x.get("response", "")), reverse=True):
            # Sort by response length (longer response = more detailed = better)
            key = example.get("instruction", "").strip().lower()
            
            if key not in seen_inputs:
                seen_inputs.add(key)
                unique.append(example)
        
        exact_before = len(data)
        exact_after = len(unique)
        print(f"Exact dedup: {exact_before} → {exact_after} "
              f"(removed {exact_before - exact_after} duplicates)")
        
        # Step 2: Near-dedup by normalized instruction
        # Remove instructions that are just rephrasings of each other
        final = []
        normalized_seen = set()
        
        for example in unique:
            # Normalize: lowercase, remove punctuation, strip whitespace
            instruction = example.get("instruction", "").lower().strip()
            # Remove common punctuation
            import re
            normalized = re.sub(r'[^\w\s]', '', instruction)
            # Remove extra whitespace
            normalized = re.sub(r'\s+', ' ', normalized).strip()
            
            # Create a "fingerprint" — first 50 chars + last 20 chars
            # This catches near-duplicates that start and end similarly
            fingerprint = normalized[:50] + normalized[-20:]
            
            if fingerprint not in normalized_seen:
                normalized_seen.add(fingerprint)
                final.append(example)
        
        print(f"Near-dedup: {exact_after} → {len(final)} "
              f"(removed {exact_after - len(final)} near-duplicates)")
        
        return final
    
    def stratified_split(self, data: list[dict],
                          train_ratio: float = 0.8,
                          eval_ratio: float = 0.1,
                          stratify_key: str = "category") -> tuple:
        """
        Split data preserving class distribution.
        
        Unlike random split, stratified split ensures each category
        has proportional representation in train/eval/test.
        """
        # Group by category
        groups = defaultdict(list)
        for example in data:
            category = example.get(stratify_key, "uncategorized")
            groups[category].append(example)
        
        train, eval_set, test = [], [], []
        
        for category, group in groups.items():
            random.shuffle(group)
            n = len(group)
            
            n_train = int(n * train_ratio)
            n_eval = int(n * eval_ratio)
            
            train.extend(group[:n_train])
            eval_set.extend(group[n_train:n_train + n_eval])
            test.extend(group[n_train + n_eval:])
        
        random.shuffle(train)
        random.shuffle(eval_set)
        random.shuffle(test)
        
        print(f"Stratified split: train={len(train)}, eval={len(eval_set)}, test={len(test)}")
        
        return train, eval_set, test
    
    def format_for_model(self, data: list[dict], 
                          model_format: str = "phi-2") -> list[dict]:
        """
        Format data for a specific model's expected format.
        
        Different models expect different chat templates:
        - Phi-2: <|user|>\n...\n<|assistant|>\n...
        - Mistral: [INST] ... [/INST] ...
        - Llama 3: <|start_header_id|>user<|end_header_id|>...<|eot_id|>
        """
        formatted = []
        
        for example in data:
            instruction = example.get("instruction", "")
            response = example.get("response", "")
            
            if model_format == "phi-2":
                text = f"<|user|>\n{instruction}\n<|assistant|>\n{response}"
            elif model_format == "mistral":
                text = f"[INST] {instruction} [/INST] {response}"
            elif model_format == "llama3":
                text = (f"<|start_header_id|>user<|end_header_id|>\n\n"
                       f"{instruction}<|eot_id|>"
                       f"<|start_header_id|>assistant<|end_header_id|>\n\n"
                       f"{response}<|eot_id|>")
            elif model_format == "chatml":
                text = (f"<|im_start|>user\n{instruction}<|im_end|>\n"
                       f"<|im_start|>assistant\n{response}<|im_end|>")
            else:
                text = f"User: {instruction}\nAssistant: {response}"
            
            formatted.append({
                "text": text,
                "instruction": instruction,
                "response": response,
                "format": model_format,
            })
        
        return formatted
    
    def validate_final_data(self, data: list[dict], tokenizer) -> dict:
        """
        Validate that the data is ready for training.
        
        Checks:
        1. All examples tokenize correctly
        2. No examples exceed max_length
        3. No empty token sequences
        4. All examples have the same format
        """
        validation = {
            "total": len(data),
            "valid": 0,
            "invalid": 0,
            "issues": [],
        }
        
        for i, example in enumerate(data):
            try:
                tokens = tokenizer(example["text"], truncation=True, max_length=2048)
                
                if len(tokens["input_ids"]) == 0:
                    validation["issues"].append(f"Example {i}: Empty token sequence")
                    validation["invalid"] += 1
                elif len(tokens["input_ids"]) >= 2048:
                    validation["issues"].append(
                        f"Example {i}: {len(tokens['input_ids'])} tokens (truncated)"
                    )
                    validation["valid"] += 1
                else:
                    validation["valid"] += 1
                    
            except Exception as e:
                validation["issues"].append(f"Example {i}: Tokenization error: {e}")
                validation["invalid"] += 1
        
        print(f"Validation: {validation['valid']}/{validation['total']} valid")
        if validation["invalid"] > 0:
            print(f"  ⚠️ {validation['invalid']} invalid examples detected")
        
        return validation
    
    def save_splits(self, train, eval_set, test, output_dir: str):
        """Save all splits as JSON files."""
        import os
        os.makedirs(output_dir, exist_ok=True)
        
        splits = {
            "train.json": train,
            "eval.json": eval_set,
            "test.json": test,
        }
        
        for filename, split_data in splits.items():
            filepath = os.path.join(output_dir, filename)
            with open(filepath, "w") as f:
                json.dump(split_data, f, indent=2)
            print(f"Saved {len(split_data)} examples to {filepath}")


# USAGE: The processing pipeline

def prepare_dataset(
    input_file: str,
    output_dir: str,
    model_format: str = "phi-2",
    seed: int = 42,
):
    """
    Complete data preparation pipeline.
    
    Before you run this, think about:
    - Does your data NEED near-deduplication? (If low variation, yes)
    - Does your data HAVE categories for stratified split?
    - What format does YOUR model expect?
    """
    preprocessor = DataPreprocessor(seed=seed)
    
    # 1. Load and inspect
    data = preprocessor.load_and_inspect(input_file)
    
    # 2. Quality filter (remove bad examples)
    data = preprocessor.quality_filter(data)
    
    # 3. Deduplicate
    data = preprocessor.deduplicate(data)
    
    # 4. Stratified split (uses "category" field or falls back to random)
    train, eval_set, test = preprocessor.stratified_split(data)
    
    # 5. Format for model
    train = preprocessor.format_for_model(train, model_format)
    eval_set = preprocessor.format_for_model(eval_set, model_format)
    test = preprocessor.format_for_model(test, model_format)
    
    # 6. Save
    preprocessor.save_splits(train, eval_set, test, output_dir)
    
    return train, eval_set, test


# BUT WAIT — before you use this pipeline, ask yourself:
# Are these defaults RIGHT for YOUR data?

# 1. Quality filter removes responses < 10 chars
#    → But what if your task needs short responses? (e.g., classification labels)
#    → Adjust min_response_length for your use case!

# 2. Dedup uses first-50 + last-20 character fingerprint
#    → This works for question-answering but might be wrong for code generation
#    → Two similar code snippets might have the same fingerprint but do different things!

# 3. Stratified split uses "category" field
#    → What if your data doesn't have categories?
#    → Falls back to random split — but random ISN'T always right
```

---

### Example 3: Semantic Deduplicator (Advanced)

```python
"""
Semantic Deduplication — Find Near-Duplicates Using Embeddings

Exact dedup catches identical questions.
Semantic dedup catches questions that MEAN the same thing
but use different words.

Use case:
  "How do I reset my password?" 
  "What's the process for changing my password?"
  "I forgot my password, what do I do?"
  → These are SEMANTICALLY the same
  → Only keep the BEST one
"""

import numpy as np
from sklearn.metrics.pairwise import cosine_similarity


class SemanticDeduplicator:
    """
    Remove near-duplicate examples using embedding similarity.
    
    How it works:
    1. Compute embeddings for all instructions
    2. Compute pairwise similarity matrix
    3. If two instructions have similarity > threshold, they're duplicates
    4. Keep only the highest quality version
    
    BUT — how do you define "highest quality"?
    - Longest response? (most detailed)
    - Highest score from labeling? (most accurate)
    - Most recent? (latest knowledge)
    """
    
    def __init__(self, similarity_threshold: float = 0.85):
        self.threshold = similarity_threshold
    
    def deduplicate(self, data: list[dict]) -> list[dict]:
        """
        Remove near-duplicates.
        
        The choice of threshold MATTERS:
        - 0.95: Only remove VERY similar (almost exact rephrases)
        - 0.85: Remove moderate similarity (same intent, different words)
        - 0.70: Aggressive — may remove genuinely different questions
        """
        # In production, you'd compute embeddings using an embedding model
        # Here we simulate with random vectors
        
        n = len(data)
        embeddings = np.random.randn(n, 384)  # Simulated embeddings
        
        # Compute similarity matrix
        similarity = cosine_similarity(embeddings)
        
        # Greedy dedup: keep higher quality examples
        kept = []
        removed = set()
        
        for i in range(n):
            if i in removed:
                continue
            
            kept.append(data[i])
            
            # Find all near-duplicates of this example
            for j in range(i + 1, n):
                if j in removed:
                    continue
                if similarity[i][j] > self.threshold:
                    removed.add(j)
        
        print(f"Semantic dedup ({self.threshold:.2f} threshold): "
              f"{n} → {len(kept)} (removed {len(removed)} near-duplicates)")
        
        return kept
```

---

### Example 4: Synthetic Data Generation

```python
"""
Synthetic Data Generation for Fine-Tuning

When to use synthetic data:
1. You don't have enough real examples (cold start problem)
2. You need to cover edge cases not in your data
3. You need to balance an imbalanced dataset
4. You want to teach the model specific patterns

When NOT to use synthetic data:
1. You have enough real, high-quality data (real > synthetic always)
2. The synthetic data contains errors that confuse the model
3. You're introducing biases that weren't there before

How to generate synthetic data:
- Use a STRONGER model (GPT-4, Claude) to generate training data
- For a WEAKER model (Phi-2, Llama-3.2-1B)
- The stronger model generates examples → weaker model learns from them
"""

import json
from openai import OpenAI


class SyntheticDataGenerator:
    """
    Generate synthetic training data using a stronger LLM.
    
    Strategy:
    1. Provide seed examples (real data) as few-shot prompts
    2. Request the LLM to generate MORE examples
    3. Validate the generated examples (quality check)
    4. Add to training set
    
    Risks:
    - The strong model's style LEAKS into training data
      → Your fine-tuned model tries to "sound like GPT-4"
      → Fine-tuning on GPT-4 outputs → model learns GPT-4's style
      → Sometimes this is GOOD (improved quality), sometimes BAD (style mismatch)
    """
    
    def __init__(self, client: OpenAI):
        self.client = client
    
    def generate_examples(
        self,
        seed_examples: list[dict],
        count: int = 100,
        domain: str = "customer support",
    ) -> list[dict]:
        """Generate synthetic examples based on seed examples."""
        
        # Build few-shot prompt
        seed_text = "\n\n".join([
            f"Input: {e['instruction']}\nOutput: {e['response']}"
            for e in seed_examples[:5]  # Use first 5 as examples
        ])
        
        prompt = f"""You are generating training data for a {domain} AI assistant.
        
Here are {min(5, len(seed_examples))} example interactions:

{seed_text}

Now generate {count} MORE diverse examples of user questions and assistant responses.
Each example should be realistic and cover different scenarios within {domain}.

IMPORTANT:
- Make questions diverse (different phrasings, intents, complexity)
- Make responses accurate and helpful
- Cover edge cases (angry users, confused users, technical users)
- Do NOT include PII (no real names, emails, phone numbers)

Format each example as JSON with "instruction" and "response" fields.
Return a JSON array."""
        
        response = self.client.chat.completions.create(
            model="gpt-4o",  # Strong model generates data
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
            temperature=0.8,  # Higher temperature = more diversity
        )
        
        generated = json.loads(response.choices[0].message.content)
        
        print(f"Generated {len(generated)} synthetic examples")
        return generated
    
    def validate_examples(self, examples: list[dict]) -> list[dict]:
        """
        Validate generated examples for quality.
        
        Remove:
        - Examples that are too short
        - Examples that don't match the domain
        - Examples with PII (phone numbers, emails)
        - Examples that are clearly wrong
        """
        import re
        
        valid = []
        for ex in examples:
            instruction = ex.get("instruction", "")
            response = ex.get("response", "")
            
            # Check length
            if len(instruction) < 5 or len(response) < 10:
                continue
            
            # Check for PII patterns
            if re.search(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', response):  # Phone
                continue
            if re.search(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', response):  # Email
                continue
            
            valid.append(ex)
        
        print(f"Validation: {len(examples)} → {len(valid)} valid "
              f"(removed {len(examples) - len(valid)} invalid)")
        
        return valid
```

---

## ✅ Good Output Examples

### What a Well-Prepared Dataset Looks Like

```
DATA QUALITY REPORT
━━━━━━━━━━━━━━━━━━━━

Raw data:              12,450 examples
After quality filter:  11,823 (removed 627 low-quality)
After exact dedup:     9,456  (removed 2,367 exact dupes)
After semantic dedup:  7,234  (removed 2,222 near-dupés)
Final dataset:         7,234 examples

SPLITS:
  Train:  5,787 (80%)
  Eval:    724 (10%)
  Test:    723 (10%)
  → Stratified by category ✅

DISTRIBUTION:
  Category              Count    %       Eval %
  password_reset        1,447   20.0%    19.8% ✅
  account_issues        1,158   16.0%    16.2% ✅
  billing               1,014   14.0%    13.9% ✅
  feature_questions     868     12.0%    12.1% ✅
  technical_support     723     10.0%    10.0% ✅
  api_integration       579      8.0%     7.9% ✅
  complaints            434      6.0%     6.1% ✅
  general_inquiry       362      5.0%     5.0% ✅
  feedback              289      4.0%     3.9% ✅
  other                 360      5.0%     5.1% ✅

QUALITY CHECKS:
  ✅ No missing fields
  ✅ No label conflicts detected
  ✅ Length distribution: 64-1,024 tokens (P95: 512)
  ✅ Format validated for phi-2 template
  ✅ No PII detected in outputs
  ✅ All examples tokenize correctly

READY FOR TRAINING ✅
```

### What a Failed Dataset Looks Like

```
DATA QUALITY REPORT — WARNINGS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Raw data:              5,000 examples
After quality filter:  3,842 (removed 1,158 — HIGH!)
After exact dedup:     3,200
After semantic dedup:  2,800
Final dataset:         2,800 examples

⚠️ ISSUES FOUND:

CRITICAL:
  1. 23% of examples have missing "category" field
     → Can't do stratified split
     → Category-specific eval will be impossible
  
  2. 47 label conflicts (same input, different output)
     → Model will learn contradictory responses
     → Example: "What's your return policy?" has 3 different answers
     → RECOMMENDATION: Resolve before training

  3. 12 examples contain email addresses in responses
     → PII LEAKAGE — model may learn to generate PII
     → Remove and retrain

WARNINGS:
  4. Severe class imbalance: "password_reset" (45%) vs "api" (2%)  
     → Model will over-optimize for password reset
     → Consider oversampling "api" category

  5. 15% of responses are < 20 characters
     → May teach the model to give short, unhelpful answers
     → Consider raising minimum response length

  6. Format not validated — model template unknown
     → May cause training errors if format is wrong

RECOMMENDATION: DO NOT TRAIN with this dataset. Fix issues first.
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Garbage In, Gospel Out

```python
# BAD: Trust all data equally
with open("data.json") as f:
    data = json.load(f)
# No inspection, no filtering, no validation
# Bad examples get equal weight to good ones

# GOOD: Quality score each example
for example in data:
    example["quality_score"] = score_quality(example)
# Filter low-quality examples
data = [e for e in data if e["quality_score"] > 0.7]
# Weight high-quality examples more during training
```

### Antipattern 2: Quantity Over Quality

```python
# BAD: "More data is always better"
data = load_everything_we_have()  # 100K examples
# 20K are excellent, 80K are mediocre
# The model learns from the MEDIAN quality, not the BEST
# Result: Mediocre model

# GOOD: Quality over quantity
data = load_highest_quality()  # 5K excellent examples
# Less data, but every example teaches the RIGHT pattern
# Result: Excellent model (on the covered distribution)
```

### Antipattern 3: Leaky Test Set

```python
# BAD: Same distribution in train and test
# All data collected from same source at same time
# Test doesn't measure generalization — it measures memorization

# GOOD: Independent test set
# Train: Current data
# Test: Data collected 3 months LATER (from production)
# This measures: Can the model handle FUTURE inputs?
```

### Common Failure Modes:

| Failure Mode | Symptom | Root Cause | Fix |
|---|---|---|---|
| **Model memorizes** | High train accuracy, low eval | Too many duplicates, too many epochs | Dedup, fewer epochs, early stopping |
| **Model contradicts itself** | Same question, different answers | Label conflicts in training data | Conflict detection, consensus labeling |
| **Model is bland** | Outputs are generic, safe, boring | Labelers averaged their answers | Pick one consistent style, not average |
| **Model fails on real data** | 94% eval, 62% production | Distribution mismatch: train ≠ prod | Better data collection (real user data) |
| **Model generates PII** | Outputs contain emails/phones | PII in training data wasn't cleaned | PII redaction pipeline in data prep |
| **Model is biased** | Performs worse for certain groups | Imbalanced or biased training data | Stratified sampling, bias audit |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Quality Scorer (30 min)

```python
def score_example_quality(example: dict) -> float:
    """
    Score the quality of a single training example (0.0 to 1.0).
    
    Factors to consider:
    - Response length (too short = low quality)
    - Response specificity (vague = low quality)
    - Has PII? (PII = low quality — should be removed)
    - Is it a question-answer pair? (complete = higher quality)
    - Does the response answer the question? (relevant = higher)
    - Is the language professional? (profanity = lower)
    
    Returns a quality score.
    """
    # Your implementation here
    pass


# Test your scorer
examples = [
    {"instruction": "What is Python?", "response": "A programming language."},
    {"instruction": "How do I reset my password?", "response": "Go to settings."},
    {"instruction": "Help me!", "response": "Sure, I can help."},
    {"instruction": "What's your email?", "response": "Contact us at support@company.com"},
    {"instruction": "Explain quantum computing", "response": "...(2000 word detailed explanation)..."},
]

for ex in examples:
    score = score_example_quality(ex)
    print(f"Score {score:.2f}: {ex['instruction'][:30]}")
```

**🤔 Questions:**
1. What score does "Sure, I can help" get? Is that a high-quality training example? (The response doesn't actually help — it's a non-answer.)
2. What weight should each factor have? Is response length 50% of the score? Is PII detection an auto-0?
3. How do you validate that your quality scorer is CORRECT? What makes a "high-quality" training example?

---

### Drill 2: Detect Label Conflicts (30 min)

```python
def find_label_conflicts(data: list[dict]) -> list[dict]:
    """
    Find examples where the SAME input has DIFFERENT outputs.
    
    These are LABEL CONFLICTS — the model sees contradictory
    information and learns the average of both (wrong for everyone).
    
    Return: List of conflicts, each showing the input and all conflicting outputs.
    """
    # Your implementation here
    pass


def resolve_conflict(examples_with_same_input: list[dict]) -> str:
    """
    Given multiple responses to the same input, determine the BEST response.
    
    Strategies:
    1. Vote (most common answer wins)
    2. Quality score (highest scored response wins)
    3. Recency (most recent response wins)
    4. Source authority (expert > junior)
    5. Human re-review (new expert judgment)
    
    Which strategy is best? It DEPENDS on your data.
    """
    # Your implementation here
    pass
```

**🤔 Questions:**
1. What if the SAME input legitimately has MULTIPLE correct answers? (e.g., "What's a good restaurant?" — many valid answers.) Is this a label conflict, or is it just diversity?
2. How do you distinguish "multiple correct answers" from "contradictory answers"?
3. If you take the MOST COMMON answer, you might be amplifying the majority's mistakes. If you take the EXPERT answer, you might be ignoring valid alternatives. What's your tiebreaker?

---

### Drill 3: Design a Synthetic Data Strategy (45 min)

You're building a fine-tuning dataset for a new feature: an AI that helps users debug Python code. You have NO real data (the feature doesn't exist yet).

Design a synthetic data generation strategy:

```python
def generate_debugging_dataset(
    seed_topics: list[str],
    examples_per_topic: int = 100,
) -> list[dict]:
    """
    Generate a synthetic dataset for a Python debugging assistant.
    
    Seed topics might include:
    - "IndexError: list index out of range"
    - "KeyError in dictionary"
    - "ImportError: No module named X"
    - "TypeError: unsupported operand type"
    - "AttributeError: 'NoneType' object has no attribute"
    
    For EACH topic, generate:
    - Multiple phrasings of the problem (different user styles)
    - Multiple solution approaches (different assistant styles)
    - Different code complexity levels (beginner, intermediate, advanced)
    
    Return: List of {"instruction": ..., "response": ...}
    """
    # Your implementation here
    pass
```

**🤔 Questions:**
1. How do you ensure the synthetic examples are REALISTIC? (Real users don't say "I'm getting an IndexError on line 42" — they say "My code broke, fix it!")
2. How do you ensure the solutions are CORRECT? (The generating model might produce plausible-sounding but wrong code.)
3. How do you cover the "unknown unknowns" — bugs you haven't thought of? (You can only generate data for KNOWN error types.)

---

## 🚦 Gate Check

Before moving to File 04 (Training with Unsloth/TRL), verify you can:

1. **Generate a data quality report** — Given a raw dataset, identify: duplicates, missing values, label conflicts, length distribution, class balance, format issues, PII leakage

2. **Build a data cleaning pipeline** — Implement: exact dedup, semantic near-dedup, quality filtering, length validation, format standardization. The pipeline should be reproducible (seed-controlled).

3. **Design a train/eval/test split strategy** — Given data with temporal structure, labeler information, and class imbalance, design a splitting strategy that measures TRUE generalization (not in-distribution performance)

4. **Quality-filter a dataset** — Given 1,000 raw examples with varying quality, score each one and filter to keep only the top 70%. Validate that the filtered set has higher quality.

5. **Detect and resolve label conflicts** — Given a dataset with 20 conflicting labels (same input, different outputs), identify all conflicts and resolve them using a defensible strategy

6. **Generate synthetic data responsibly** — Given a cold-start scenario (new feature, no data), design a synthetic generation strategy that covers the full input distribution without introducing errors or bias

7. **Calculate data preparation effort** — Given:
   - Raw data: 50,000 examples
   - 15% are duplicates
   - 8% have missing fields
   - 5% have PII
   - 3% have label conflicts
   - Cleaning rate: 100 examples/hour per person
   
   How many person-hours to clean? What's the cost? Is it worth cleaning the bottom 20% or just dropping them?

---

## 📚 Resources

### Data Quality
- **[Data Quality for ML](https://docs.cleanlab.ai/)** — CleanLab: automated data quality assessment
- **[Deduplication at Scale](https://github.com/google-research/deduplicate-text-datasets)** — Google's approach to text deduplication

### Synthetic Data
- **[Generating Training Data with LLMs](https://arxiv.org/abs/2305.13683)** — Using strong models to generate data for weaker ones
- **[Self-Instruct](https://arxiv.org/abs/2212.10560)** — Bootstrapping synthetic data from a base model

### Dataset Curation
- **[Alpaca Dataset](https://github.com/tatsu-lab/stanford_alpaca)** — Influential synthetic dataset for instruction tuning
- **[LIMA: Less Is More](https://arxiv.org/abs/2305.11206)** — Paper showing that 1,000 high-quality examples can match 50,000 lower-quality ones

### Phase Cross-References
- **Phase 4 (RAG)** → Fine-tuning datasets often come from RAG pipelines (question + retrieved context + answer). The data preparation in this file connects to your Phase 4 RAG system.
- **Phase 5 (PII)** → Your PII redaction module (Phase 5, File 05) is ESSENTIAL for cleaning training data before fine-tuning.
- **Phase 7 (Evals)** → The test set you create here is evaluated using your Phase 7 eval platform.

> **Next up: File 04 — Training with Unsloth/TRL.** Now that your data is clean, you'll actually run training. But the training loop has its own traps — gradient accumulation, learning rate scheduling, checkpointing, resuming from failures, monitoring loss curves in real time.
