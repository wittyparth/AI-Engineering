# 05 — Evaluation & Comparison: Proving Your Fine-Tuning Worked

## 🎯 Purpose & Goals

> 🛑 STOP. Before you evaluate your fine-tuned model, you need to understand the single most dangerous assumption in AI engineering:

**"Lower loss = better model."**

This is WRONG. And believing it has led teams to ship models that are objectively worse than their base.

Here's why: Loss measures how well the model predicts the training data. But "predicting training data" is NOT the same as "performing well on your task." A model can have PERFECT loss (memorized the training set) and still be useless — because it can't generalize, because it learned spurious correlations, because it forgot how to do everything else.

The evaluation phase is where fine-tuning either PROVES its value or gets exposed as a waste of time.

**The goal of this module: Build an evaluation framework that catches regressions before they reach production.**

---

### 🤔 Discovery Question 1: The Baseline Problem

You fine-tune a model on legal summarization. Before training, you evaluate the BASE model:

| Metric | Base Model Score |
|--------|-----------------|
| ROUGE-L | 0.32 |
| BERTScore | 0.78 |
| Human rating (1-5) | 3.1 |

After fine-tuning, you evaluate your FT model:

| Metric | Base Model | FT Model | Change |
|--------|-----------|----------|--------|
| ROUGE-L | 0.32 | 0.47 | **+47% 🎉** |
| BERTScore | 0.78 | 0.85 | **+9% 🎉** |
| Human rating (1-5) | 3.1 | 4.2 | **+35% 🎉** |

Everything looks great! You ship the model.

**Two weeks later, users complain: "The summaries are full of factual errors. They SOUND right but the details are wrong."**

**🤔 Before reading on, answer these:**

1. ROUGE-L, BERTScore, and human ratings ALL improved. But the model is producing factual errors. How is this possible? What are these metrics actually measuring, and what are they MISSING? (Hint: ROUGE measures n-gram overlap with reference summaries — it rewards FLUENT text that resembles the reference, even if the facts are wrong. BERTScore measures embedding similarity — same issue. Human raters might be evaluating WRITING quality, not factuality.)

2. Your baseline evaluation was: feed the base model the same prompts, compare outputs. But the base model was NEVER trained on legal summarization — it was a GENERAL model. Is a general-purpose model the RIGHT baseline? Or should you compare against a PROMPT-ENGINEERED version of the base model (few-shot examples, instructions)? What if prompt-engineered base model gets 0.44 ROUGE-L? Is your +47% actually only +7%?

3. **The hardest question:** You need to evaluate FACTUAL ACCURACY, not just textual similarity. But verifying facts in legal summaries requires a LAWYER to review each output. You have 1,000 test examples. A lawyer costs $200/hour and reviews 10 summaries per hour. That's $20,000 just for evaluation. **How do you build a cost-effective evaluation that catches factual errors without going bankrupt?** Could you use LLM-as-judge? Could you create a fact-checking pipeline? What are the tradeoffs?

---

### 🤔 Discovery Question 2: The Metric That Lies

You track 5 metrics during evaluation:

```
┌────────────────────┬──────────┬──────────┬──────────┐
│ Metric             │ Baseline │ FT Model │ Δ        │
├────────────────────┼──────────┼──────────┼──────────┤
│ Exact Match        │ 28.3%    │ 41.2%    │ +12.9%   │
│ F1 Score           │ 0.52     │ 0.63     │ +0.11    │
│ ROUGE-L            │ 0.35     │ 0.44     │ +0.09    │
│ Perplexity         │ 4.2      │ 2.8      │ -1.4     │
│ Response Length    │ 142      │ 187      │ +45      │
└────────────────────┴──────────┴──────────┴──────────┘
```

All metrics improved. The model is clearly better, right?

But here's the hidden problem: Your FT model was trained on dataset where reference answers are AVERAGE 200 tokens. Your base model naturally generates 142-token responses. The FT model learned to generate LONGER responses (187 tokens). And ROUGE-L naturally increases with response length — longer responses have more n-gram overlap with the reference purely by chance.

**🤔 Before reading on, answer these:**

1. Response length increased by 32%. ROUGE-L increased by 26%. Is the ROUGE-L improvement REAL (better quality) or just a side-effect of longer responses? How would you DECOUPLE length from quality in your evaluation? (Hint: Length-normalized ROUGE. Or evaluate on a SUBSET where FT and base generate similar-length responses.)

2. Perplexity dropped from 4.2 to 2.8 — a 33% improvement! But perplexity measures how "surprised" the model is by the test data. If your test data was DRAWN FROM THE SAME DISTRIBUTION as your training data, lower perplexity might just mean "the model memorized the training distribution better." How do you ensure your test set is TRULY independent from your training set? What constitutes contamination?

3. **The hardest question:** You notice that 3 of your 5 metrics correlate strongly with response length. Your FT model generates longer responses, so those metrics improve. But users prefer CONCISE answers. A user satisfaction survey shows: base model: 4.1/5, FT model: 3.7/5. **All automated metrics say "improvement" but users say "worse." Which do you trust? How do you build an evaluation that PREDICTS user satisfaction instead of just measuring textual similarity?**

---

### 🤔 Discovery Question 3: The Regression That Went Unnoticed

Your legal summarization FT model scores:

| Metric | Base | FT | Δ |
|--------|------|----|----|
| Legal summarization | 0.32 | 0.47 | +47% |
| General QA | 0.78 | 0.77 | -1% |
| Code generation | 0.65 | 0.42 | **-35%** |
| Sentiment analysis | 0.81 | 0.80 | -1% |
| Safety (toxic output) | 0.2% | 3.1% | **+15x** |

**🤔 Before reading on, answer these:**

1. You tested 5 capabilities. Legal summarization improved dramatically. Code generation dropped 35%. Safety toxicity rate increased 15x. If you ONLY tested legal summarization (the task you fine-tuned for), you'd think the model is amazing. **What's your responsibility as an AI engineer to test for regressions?** Where do you draw the line between "thorough testing" and "impractical over-testing"?

2. Safety toxicity increased 15x. This is the SAFETY REGRESSION that Phase 8 guardrails are designed to catch. But what if your Phase 8 guardrails were calibrated for the BASE model's output distribution? The FT model produces different kinds of toxic text — more confident, more contextually embedded, harder to detect. **How do you ensure your guardrails work for BOTH base and FT models?**

3. **The hardest question:** You have limited evaluation budget (time and money). You can't test every capability for every model version. **Design a REGRESSION TEST SUITE — a minimal set of evaluations that catches the most damaging regressions before shipping. What capabilities must be tested? What metrics matter? How few evals can you get away with before the risk becomes unacceptable?**

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 1 hr |
| Code Examples (2) | 1.5 hr |
| Drills (3) | 1 hr |
| Gate Check | 30 min |
| **Total** | **~4.5 hr** |

---

## 📖 Concept Deep-Dive

### 1. The Evaluation Hierarchy for Fine-Tuning

Not all evaluations are equal. For fine-tuning, you need a FOUR-LEVEL hierarchy:

```
Level 1: Training Metrics
├── Training loss
├── Evaluation loss  
├── Gradient norms
└── Learning rate

Level 2: Task-Specific Metrics
├── Exact Match (classification/extraction)
├── ROUGE/ROUGE-L (summarization)
├── BLEU (translation)
├── F1 Score (information extraction)
├── BERTScore (semantic similarity)
└── Perplexity (language modeling)

Level 3: Behavioral Evaluation
├── Human evaluation (quality, factuality)
├── LLM-as-judge (multi-dimension scoring)
├── Adversarial testing (edge cases)
├── Safety evaluation (toxic output rate)
└── Capability regression (unrelated tasks)

Level 4: Production Evaluation
├── A/B test results (user metrics)
├── Latency/cost benchmarks
├── Error analysis (what kinds of mistakes?)
└── Drift monitoring over time
```

**Senior Engineer Thinking:**
- Junior engineers evaluate at Level 2 only (metrics)
- Senior engineers build Level 3 evaluation BEFORE they start training
- Staff engineers connect Level 4 evaluation back to Level 1 — "this loss pattern in training predicts this production behavior"

### 2. Why Single Metrics Are Dangerous

Every metric has a blind spot:

| Metric | What It Measures | Blind Spot |
|--------|-----------------|------------|
| Loss | Prediction accuracy on training distribution | Memorization ≠ generalization |
| ROUGE | N-gram overlap with reference | Rewards longer/textbook answers |
| BLEU | Precision of n-gram matches | Punishes valid alternative phrasings |
| BERTScore | Embedding similarity | Semantically similar ≠ factually correct |
| Perplexity | Model confidence | Confidently wrong gets low perplexity |
| Exact Match | Exact string match | Too strict for generation tasks |

**The Rule of Three:** No single metric should make a ship/no-ship decision. You need at least THREE independent signals:
1. One automated metric (e.g., ROUGE-L)
2. One quality metric (e.g., LLM-as-judge or human eval)
3. One safety/regression metric (e.g., toxicity check, capability regression)

If all three agree → ship.
If they disagree → investigate before shipping.

### 3. Statistical Significance in Model Comparison

You evaluate both models on 500 test examples. FT model scores 0.47 ROUGE-L, base scores 0.32. Is the 0.15 difference REAL or just noise?

**Without statistical significance testing, you don't know.**

Key concepts:
- **Confidence intervals**: A 95% CI of [0.43, 0.51] for FT means you're 95% confident the true score is between 0.43 and 0.51
- **Overlapping CIs**: If base CI is [0.29, 0.35] and FT CI is [0.43, 0.51], no overlap → significant
- **Paired evaluation**: Same test examples, different models → use paired tests (more sensitive)
- **Bootstrapping**: Resample your evaluation data 1000x to estimate score distributions without parametric assumptions

**Common mistake**: Evaluating on 10 examples, seeing 90% vs 80%, declaring victory. With 10 examples, a 10% difference is NOT statistically significant. You need ~385 examples per model to detect a 5% difference at 95% confidence.

### 4. The Regression Test Suite

Every fine-tuned model needs a regression test suite covering:

1. **Primary task performance**: Did the FT model improve on the target task?
2. **Capability retention**: Did the FT model maintain performance on general capabilities?
3. **Safety baseline**: Did the FT model regress on safety? (Toxicity, bias, harmful content)
4. **Edge case robustness**: Does the FT model handle adversarial inputs?
5. **Behavioral consistency**: Does the FT model maintain the same output format/tone?

**The Minimum Viable Regression Suite:**
- 200 primary task examples (with human-verified ground truth)
- 100 general capability examples (from a public benchmark like MMLU or HELM)
- 100 safety examples (toxic prompts, jailbreak attempts, edge cases)
- 50 adversarial examples (format variations, misspellings, unusual inputs)

**Cost estimate**: ~450 LLM eval calls × $0.01 = $4.50 per evaluation run. Run after every training run.

### 5. LLM-as-Judge for Fine-Tuning Evaluation

Using LLMs to evaluate LLM outputs is controversial but practical. The key is a STRUCTURED evaluation rubric:

```
Evaluation Rubric for Legal Summarization:

1. ACCURACY (1-5): Are ALL facts in the summary present in the source document?
2. COMPLETENESS (1-5): Does the summary include all important points?
3. CONCISENESS (1-5): Is the summary appropriately brief?
4. CLARITY (1-5): Is the summary easy to understand?
5. LEGAL PRECISION (1-5): Does the summary use correct legal terminology?
```

**The trap**: Using the same LLM that you're trying to replace as the judge. If you're fine-tuning for legal summarization, don't use GPT-4 to evaluate your GPT-3.5 FT model — GPT-4 has its own biases. Use a DIFFERENT model family (Claude, Gemini) or a specialized evaluation model.

**Better approach**: Build a CALIBRATION set — 20 examples with known human scores. If your LLM judge scores match human scores within ±0.5, trust it. If not, fix your rubric.

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Evaluation Script (Full of Bugs)

```python
"""
WHAT NOT TO DO — An evaluation script with 10 hidden bugs.

Read this code. Find the bugs. THEN look at the fixed version.
"""

import json
import numpy as np
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer
from rouge_score import rouge_scorer
from bert_score import BERTScorer
import torch

# BUG 1: No seed set — results are non-deterministic
# BUG 2: No configuration — all settings hardcoded

def evaluate_model(model_path, test_file):
    """Evaluate a fine-tuned model on a test set."""
    
    # BUG 3: Loading model without specifying quantization
    model = AutoModelForCausalLM.from_pretrained(model_path)
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    
    # BUG 4: No padding token set for base model
    # tokenizer.pad_token = tokenizer.eos_token
    
    # Load test data
    with open(test_file, 'r') as f:
        test_data = [json.loads(line) for line in f]
    
    # BUG 5: Using entire test set as evaluation
    # No train/eval split — but this IS the test set. No validation set either.
    
    # Initialize scorers
    rouge = rouge_scorer.RougeScorer(['rougeL'], use_stemmer=True)
    bert = BERTScorer(lang='en', rescale_with_baseline=True)
    
    predictions = []
    references = []
    
    for i, example in enumerate(test_data):
        prompt = example['prompt']
        reference = example['completion']
        
        # BUG 6: No generation parameters
        # Using defaults means different models use different generation settings
        # Base model might need different temperature than FT model
        inputs = tokenizer(prompt, return_tensors='pt')
        with torch.no_grad():
            outputs = model.generate(**inputs, max_new_tokens=256)
        
        prediction = tokenizer.decode(outputs[0], skip_special_tokens=True)
        
        # BUG 7: Not removing the prompt from the generated output
        # prediction includes prompt text, so ROUGE is comparing
        # "prompt + answer" against "reference" — wrong!
        
        predictions.append(prediction)
        references.append(reference)
    
    # BUG 8: No length normalization
    rouge_scores = [rouge.score(ref, pred)['rougeL'].fmeasure 
                    for ref, pred in zip(references, predictions)]
    
    # BUG 9: BERTScore on entire output including prompt
    _, _, bert_f1 = bert.score(predictions, references)
    
    # BUG 10: Reporting mean without confidence intervals
    # "ROUGE-L: 0.47" — could be 0.47 ± 0.20 (huge variance)
    print(f"ROUGE-L: {np.mean(rouge_scores):.4f}")
    print(f"BERTScore F1: {bert_f1.mean().item():.4f}")
    
    # BUG 11: No per-example analysis
    # What kinds of examples does the model fail on?
    # Are failures random or systematic?
    
    return rouge_scores, bert_f1
```

#### 🔍 Find The Bugs

1. **No seed set** → Non-deterministic generation. Run twice, get different results.
2. **No configuration** → Compare scenarios impossible.
3. **No quantization** → Loading 7B model in full precision on 16GB GPU? OOM.
4. **No padding token** → Batch generation will fail or produce wrong results.
5. **No validation split** → You're evaluating on everything. No held-out set for hyperparameter tuning.
6. **Default generation params** → Different model families need different generation configs. Your comparison is contaminated.
7. **Prompt included in output** → ROUGE score is artificially inflated by prompt overlap.
8. **No length normalization** → Longer predictions get higher ROUGE naturally.
9. **BERTScore on prompt-included text** → Embedding similarity captures prompt similarity, not answer quality.
10. **No confidence intervals** → "ROUGE-L: 0.47" is useless without knowing variance.
11. **No per-example analysis** → Where does it fail? Is there a pattern?

---

### Example 2: Production Evaluation Framework (Fixed)

```python
"""
Production Evaluation Framework for Fine-Tuning Comparison.

This is NOT a toy script. This is what a real eval pipeline looks like.
Features:
- Deterministic evaluation (seed + config)
- Multiple metrics with confidence intervals
- Paired comparison (same examples for both models)
- Per-example error analysis
- Regression test suite
- Statistical significance testing
"""

import json
import random
import numpy as np
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field, asdict
from pathlib import Path
import logging
from tqdm import tqdm

import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from rouge_score import rouge_scorer
from bert_score import BERTScorer
from scipy import stats

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ──────────────────────────────────────────────
# CONFIGURATION
# ──────────────────────────────────────────────

@dataclass
class EvalConfig:
    """All evaluation settings in one place."""
    seed: int = 42
    batch_size: int = 8
    max_new_tokens: int = 256
    temperature: float = 0.1  # Low temperature for deterministic eval
    top_p: float = 0.9
    num_beams: int = 1  # No beam search for speed
    quantization: str = "4bit"  # "4bit" | "8bit" | "none"
    metrics: List[str] = field(default_factory=lambda: [
        "rougeL", "bertscore", "exact_match", "f1"
    ])
    confidence_level: float = 0.95
    num_bootstrap_samples: int = 1000
    
    def __post_init__(self):
        assert self.temperature <= 0.2, (
            f"Evaluation temperature should be low for deterministic comparison. "
            f"Got {self.temperature}. Use 0.1 or lower."
        )


@dataclass
class GenerationParams:
    """Consistent generation parameters across all models."""
    max_new_tokens: int = 256
    temperature: float = 0.1
    top_p: float = 0.9
    do_sample: bool = True
    num_beams: int = 1
    pad_token_id: Optional[int] = None
    eos_token_id: Optional[int] = None
    
    def to_dict(self) -> Dict:
        return asdict(self)


# ──────────────────────────────────────────────
# MODEL LOADING
# ──────────────────────────────────────────────

def load_model_for_eval(model_path: str, config: EvalConfig):
    """Load model with consistent settings for evaluation."""
    
    torch.manual_seed(config.seed)
    random.seed(config.seed)
    np.random.seed(config.seed)
    
    quantization_config = None
    if config.quantization == "4bit":
        quantization_config = BitsAndBytesConfig(
            load_in_4bit=True,
            bnb_4bit_compute_dtype=torch.float16,
            bnb_4bit_quant_type="nf4",
        )
    elif config.quantization == "8bit":
        quantization_config = BitsAndBytesConfig(
            load_in_8bit=True,
        )
    
    model = AutoModelForCausalLM.from_pretrained(
        model_path,
        quantization_config=quantization_config,
        device_map="auto",
        torch_dtype=torch.float16,
    )
    
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    
    # CRITICAL: Set padding token for batch generation
    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token
        model.config.pad_token_id = model.config.eos_token_id
    
    model.eval()
    return model, tokenizer


# ──────────────────────────────────────────────
# METRICS
# ──────────────────────────────────────────────

class MetricsCalculator:
    """Calculate multiple evaluation metrics with proper handling."""
    
    def __init__(self):
        self.rouge = rouge_scorer.RougeScorer(
            ['rougeL'], use_stemmer=True
        )
        self.bert = BERTScorer(
            lang='en', rescale_with_baseline=True
        )
    
    def extract_answer(self, generated_text: str, prompt: str) -> str:
        """Extract just the generated part, removing the prompt."""
        if generated_text.startswith(prompt):
            return generated_text[len(prompt):].strip()
        return generated_text.strip()
    
    def compute_rougeL(self, prediction: str, reference: str) -> float:
        """Compute ROUGE-L with length-aware analysis."""
        scores = self.rouge.score(reference, prediction)
        return scores['rougeL'].fmeasure
    
    def compute_bertscore(self, prediction: str, reference: str) -> float:
        """Compute BERTScore for a single pair."""
        _, _, f1 = self.bert.score([prediction], [reference])
        return f1.item()
    
    def compute_exact_match(self, prediction: str, reference: str) -> float:
        """Exact match after normalization."""
        pred_norm = prediction.lower().strip()
        ref_norm = reference.lower().strip()
        return 1.0 if pred_norm == ref_norm else 0.0
    
    def compute_f1(self, prediction: str, reference: str) -> float:
        """Token-level F1 score."""
        pred_tokens = set(prediction.lower().split())
        ref_tokens = set(reference.lower().split())
        
        if not pred_tokens and not ref_tokens:
            return 1.0
        if not pred_tokens or not ref_tokens:
            return 0.0
        
        intersection = pred_tokens & ref_tokens
        precision = len(intersection) / len(pred_tokens)
        recall = len(intersection) / len(ref_tokens)
        
        if precision + recall == 0:
            return 0.0
        return 2 * precision * recall / (precision + recall)


# ──────────────────────────────────────────────
# EVALUATION RUNNER
# ──────────────────────────────────────────────

class EvalRunner:
    """Run evaluation for one model and produce structured results."""
    
    def __init__(self, config: EvalConfig):
        self.config = config
        self.metrics = MetricsCalculator()
    
    def generate_batch(
        self,
        model,
        tokenizer,
        prompts: List[str],
        gen_params: GenerationParams,
    ) -> List[str]:
        """Generate responses for a batch of prompts."""
        inputs = tokenizer(
            prompts,
            return_tensors="pt",
            padding=True,
            truncation=True,
            max_length=2048,
        ).to(model.device)
        
        with torch.no_grad():
            outputs = model.generate(
                **inputs,
                **gen_params.to_dict(),
            )
        
        predictions = []
        for i, output in enumerate(outputs):
            prompt_len = inputs['attention_mask'][i].sum().item()
            generated = tokenizer.decode(
                output[prompt_len:], skip_special_tokens=True
            )
            predictions.append(generated.strip())
        
        return predictions
    
    def evaluate_model(
        self,
        model_path: str,
        test_examples: List[Dict],
        model_name: str = "unknown",
    ) -> Dict:
        """Evaluate a model on test examples."""
        
        logger.info(f"Loading model: {model_path}")
        model, tokenizer = load_model_for_eval(model_path, self.config)
        
        gen_params = GenerationParams(
            max_new_tokens=self.config.max_new_tokens,
            temperature=self.config.temperature,
            top_p=self.config.top_p,
            do_sample=True,
            num_beams=self.config.num_beams,
        )
        
        prompts = [ex['prompt'] for ex in test_examples]
        references = [ex['completion'] for ex in test_examples]
        
        all_metrics = {
            'rougeL': [],
            'bertscore': [],
            'exact_match': [],
            'f1': [],
        }
        per_example_results = []
        
        # Process in batches
        for i in range(0, len(prompts), self.config.batch_size):
            batch_prompts = prompts[i:i + self.config.batch_size]
            batch_refs = references[i:i + self.config.batch_size]
            
            predictions = self.generate_batch(
                model, tokenizer, batch_prompts, gen_params
            )
            
            for pred, ref, prompt in zip(
                predictions, batch_refs, batch_prompts
            ):
                # Extract just the answer (remove prompt)
                answer = self.metrics.extract_answer(pred, prompt)
                
                result = {
                    'prompt': prompt[:100] + '...' if len(prompt) > 100 else prompt,
                    'reference': ref,
                    'prediction': answer,
                }
                
                for metric in self.config.metrics:
                    if metric == 'rougeL':
                        score = self.metrics.compute_rougeL(answer, ref)
                    elif metric == 'bertscore':
                        score = self.metrics.compute_bertscore(answer, ref)
                    elif metric == 'exact_match':
                        score = self.metrics.compute_exact_match(answer, ref)
                    elif metric == 'f1':
                        score = self.metrics.compute_f1(answer, ref)
                    else:
                        continue
                    
                    all_metrics[metric].append(score)
                    result[f'{metric}_score'] = score
                
                per_example_results.append(result)
        
        # Compute summary statistics with confidence intervals
        summary = self._compute_summary(all_metrics)
        summary['model_name'] = model_name
        summary['num_examples'] = len(test_examples)
        summary['per_example'] = per_example_results
        
        return summary
    
    def _compute_summary(self, all_metrics: Dict) -> Dict:
        """Compute mean, std, and confidence intervals for all metrics."""
        summary = {}
        
        for metric_name, scores in all_metrics.items():
            if not scores:
                continue
            
            scores = np.array(scores)
            mean = np.mean(scores)
            std = np.std(scores)
            
            # Bootstrap confidence intervals
            ci_lower, ci_upper = self._bootstrap_ci(
                scores, self.config.confidence_level
            )
            
            summary[metric_name] = {
                'mean': float(mean),
                'std': float(std),
                'ci_lower': float(ci_lower),
                'ci_upper': float(ci_upper),
                'scores': [float(s) for s in scores],  # All individual scores
            }
        
        return summary
    
    def _bootstrap_ci(
        self, scores: np.ndarray, confidence: float = 0.95
    ) -> Tuple[float, float]:
        """Compute confidence interval via bootstrap resampling."""
        n = len(scores)
        means = []
        
        for _ in range(self.config.num_bootstrap_samples):
            sample = np.random.choice(scores, size=n, replace=True)
            means.append(np.mean(sample))
        
        alpha = (1 - confidence) / 2
        lower = np.percentile(means, alpha * 100)
        upper = np.percentile(means, (1 - alpha) * 100)
        
        return lower, upper


# ──────────────────────────────────────────────
# COMPARISON
# ──────────────────────────────────────────────

class ModelComparator:
    """Compare two models and determine if differences are significant."""
    
    def __init__(self, config: EvalConfig):
        self.config = config
    
    def compare(
        self, base_results: Dict, ft_results: Dict
    ) -> Dict:
        """Compare base and fine-tuned model results."""
        
        comparison = {
            'base_model': base_results['model_name'],
            'ft_model': ft_results['model_name'],
            'num_examples': base_results['num_examples'],
            'metrics': {},
            'overall_verdict': None,
        }
        
        all_verdicts = []
        
        for metric_name in self.config.metrics:
            if metric_name not in base_results or metric_name not in ft_results:
                continue
            
            base = base_results[metric_name]
            ft = ft_results[metric_name]
            
            diff = ft['mean'] - base['mean']
            diff_pct = (diff / base['mean'] * 100) if base['mean'] != 0 else 0
            
            # Paired t-test for statistical significance
            base_scores = np.array(base['scores'])
            ft_scores = np.array(ft['scores'])
            
            t_stat, p_value = stats.ttest_rel(ft_scores, base_scores)
            
            # Effect size (Cohen's d)
            pooled_std = np.sqrt(
                (base['std'] ** 2 + ft['std'] ** 2) / 2
            )
            cohens_d = diff / pooled_std if pooled_std > 0 else 0
            
            # Determine if improvement is significant
            is_significant = p_value < 0.05
            ci_overlap = (
                ft['ci_lower'] <= base['ci_upper'] and
                base['ci_lower'] <= ft['ci_upper']
            )
            
            if diff > 0 and is_significant:
                verdict = "IMPROVED 🟢"
            elif diff > 0 and not is_significant:
                verdict = "IMPROVED (not significant) 🟡"
            elif diff < 0 and is_significant:
                verdict = "REGRESSED 🔴"
            elif diff < 0 and not is_significant:
                verdict = "REGRESSED (not significant) 🟡"
            else:
                verdict = "NO CHANGE ⚪"
            
            all_verdicts.append(verdict)
            
            comparison['metrics'][metric_name] = {
                'base_mean': base['mean'],
                'ft_mean': ft['mean'],
                'difference': diff,
                'difference_pct': diff_pct,
                't_statistic': float(t_stat),
                'p_value': float(p_value),
                'effect_size_cohens_d': float(cohens_d),
                'ci_overlap': bool(ci_overlap),
                'is_significant': bool(is_significant),
                'verdict': verdict,
            }
        
        # Overall verdict
        improvements = sum(1 for v in all_verdicts if 'IMPROVED' in v)
        regressions = sum(1 for v in all_verdicts if 'REGRESSED' in v)
        
        if regressions == 0 and improvements >= 1:
            comparison['overall_verdict'] = "PASS ✅"
        elif regressions >= improvements:
            comparison['overall_verdict'] = "FAIL ❌"
        else:
            comparison['overall_verdict'] = "REVIEW REQUIRED ⚠️"
        
        return comparison


# ──────────────────────────────────────────────
# REGRESSION TEST SUITE
# ──────────────────────────────────────────────

class RegressionTestSuite:
    """Minimal set of evaluations to catch regressions."""
    
    def __init__(self, config: EvalConfig):
        self.config = config
        self.runner = EvalRunner(config)
    
    def test_capability_retention(
        self,
        model_path: str,
        benchmark_examples: List[Dict],
        capability_name: str,
    ) -> Dict:
        """Test if a specific capability is retained after fine-tuning."""
        
        logger.info(f"Testing capability: {capability_name}")
        
        results = self.runner.evaluate_model(
            model_path, benchmark_examples, f"ft_{capability_name}"
        )
        results['capability'] = capability_name
        return results
    
    def safety_evaluation(
        self,
        model_path: str,
        harmful_prompts: List[str],
        safe_threshold: float = 0.02,  # Max 2% harmful responses
    ) -> Dict:
        """Evaluate safety regressions."""
        
        logger.info("Running safety evaluation")
        
        # This would require a toxicity classifier
        # For now, we flag examples for manual review
        model, tokenizer = load_model_for_eval(model_path, self.config)
        gen_params = GenerationParams(
            max_new_tokens=128, temperature=0.1
        )
        
        responses = self.runner.generate_batch(
            model, tokenizer, harmful_prompts, gen_params
        )
        
        safety_results = {
            'num_prompts': len(harmful_prompts),
            'responses': list(zip(harmful_prompts, responses)),
            'requires_review': True,
            'warning': (
                "Safety evaluation requires a toxicity classifier. "
                "Flagged for manual review."
            ),
        }
        
        return safety_results
    
    def run_full_suite(
        self,
        model_path: str,
        primary_task_data: List[Dict],
        capability_benchmarks: Dict[str, List[Dict]],
        safety_data: List[str],
    ) -> Dict:
        """Run the full regression test suite."""
        
        suite_results = {
            'model_path': model_path,
            'primary_task': None,
            'capability_tests': {},
            'safety': None,
        }
        
        # 1. Primary task evaluation
        logger.info("Running primary task evaluation")
        suite_results['primary_task'] = self.runner.evaluate_model(
            model_path, primary_task_data
        )
        
        # 2. Capability retention tests
        for cap_name, cap_data in capability_benchmarks.items():
            suite_results['capability_tests'][cap_name] = \
                self.test_capability_retention(model_path, cap_data, cap_name)
        
        # 3. Safety evaluation
        suite_results['safety'] = self.safety_evaluation(
            model_path, safety_data
        )
        
        return suite_results


# ──────────────────────────────────────────────
# USAGE EXAMPLE
# ──────────────────────────────────────────────

if __name__ == "__main__":
    """
    Usage:
        python eval_framework.py --base meta-llama/Llama-2-7b-hf --ft ./ft-legal-model
    
    This script:
    1. Loads both models with identical settings
    2. Evaluates both on the same test set
    3. Computes multiple metrics with confidence intervals
    4. Performs statistical significance testing
    5. Identifies per-example failures
    6. Generates a comparison report
    """
    
    import argparse
    
    parser = argparse.ArgumentParser()
    parser.add_argument("--base", required=True, help="Base model path")
    parser.add_argument("--ft", required=True, help="FT model path")
    parser.add_argument("--test", required=True, help="Test data (JSONL)")
    parser.add_argument("--output", default="./eval_report.json")
    args = parser.parse_args()
    
    # Load test data
    with open(args.test, 'r') as f:
        test_data = [json.loads(line) for line in f]
    
    # Shuffle with fixed seed for deterministic split
    random.seed(42)
    random.shuffle(test_data)
    
    # Split: 80% eval, 20% deep analysis
    eval_data = test_data[:int(0.8 * len(test_data))]
    analysis_data = test_data[int(0.8 * len(test_data)):]
    
    config = EvalConfig(
        seed=42,
        batch_size=4,
        max_new_tokens=256,
        temperature=0.1,
        num_bootstrap_samples=1000,
    )
    
    runner = EvalRunner(config)
    comparator = ModelComparator(config)
    
    # Evaluate both models
    print("Evaluating base model...")
    base_results = runner.evaluate_model(args.base, eval_data, "base")
    
    print("Evaluating fine-tuned model...")
    ft_results = runner.evaluate_model(args.ft, eval_data, "fine-tuned")
    
    # Compare
    print("Comparing models...")
    comparison = comparator.compare(base_results, ft_results)
    
    # Print results
    print(f"\n{'='*60}")
    print(f"EVALUATION REPORT")
    print(f"{'='*60}")
    print(f"Base model: {comparison['base_model']}")
    print(f"FT model: {comparison['ft_model']}")
    print(f"Examples: {comparison['num_examples']}")
    print(f"Overall: {comparison['overall_verdict']}")
    print(f"{'='*60}\n")
    
    for metric, data in comparison['metrics'].items():
        print(f"{metric.upper()}:")
        print(f"  Base: {data['base_mean']:.4f}")
        print(f"  FT:   {data['ft_mean']:.4f}")
        print(f"  Diff: {data['difference']:+.4f} ({data['difference_pct']:+.1f}%)")
        print(f"  p-value: {data['p_value']:.4f}")
        print(f"  Cohen's d: {data['effect_size_cohens_d']:.3f}")
        print(f"  Verdict: {data['verdict']}")
        print()
    
    # Deep analysis: examples where FT is worse
    print(f"\n{'='*60}")
    print("DEEP ANALYSIS: Where FT model regressed most")
    print(f"{'='*60}")
    
    # Find examples with biggest score drops
    ft_preds = {r['prompt']: r for r in ft_results['per_example']}
    base_preds = {r['prompt']: r for r in base_results['per_example']}
    
    regressions = []
    for prompt in ft_preds:
        if prompt not in base_preds:
            continue
        ft_score = ft_preds[prompt].get('rougeL_score', 0)
        base_score = base_preds[prompt].get('rougeL_score', 0)
        if ft_score < base_score - 0.1:  # More than 0.1 drop
            regressions.append({
                'prompt': prompt,
                'base_score': base_score,
                'ft_score': ft_score,
                'drop': base_score - ft_score,
            })
    
    regressions.sort(key=lambda x: x['drop'], reverse=True)
    print(f"Found {len(regressions)} regressed examples (drop > 0.1)")
    for r in regressions[:5]:
        print(f"\n  Prompt: {r['prompt'][:80]}...")
        print(f"  Base: {r['base_score']:.3f} → FT: {r['ft_score']:.3f} (drop: {r['drop']:.3f})")
    
    # Save report
    report = {
        'comparison': comparison,
        'base_results': base_results,
        'ft_results': ft_results,
        'regressions': regressions[:20],
    }
    
    with open(args.output, 'w') as f:
        json.dump(report, f, indent=2, default=str)
    
    print(f"\nReport saved to {args.output}")
```

---

### 🔍 Critical Analysis Questions

1. **Generation parameters**: The config uses temperature=0.1 and do_sample=True. But some evaluations suggest using temperature=0 (greedy) for deterministic comparison. What's the tradeoff? (Greedy: deterministic always, but may produce repetitive text. Low temp: slight variation, but more natural. Which is better for fair comparison?)

2. **The bootstrap method** resamples WITH replacement from the original scores. But what if the scores have STRUCTURE — for example, early test examples are easier than later ones? How would temporal structure in your test set affect bootstrap estimates?

3. **Paired vs. unpaired tests**: The comparator uses `ttest_rel` (paired t-test) because the same examples are used for both models. But what if some examples are missing for one model (OOM during generation)? How would you handle incomplete pairs?

---

## ✅ Good Output Examples

### What a Good Evaluation Report Looks Like

```
====================================================================
EVALUATION REPORT
====================================================================
Base model: meta-llama/Llama-2-7b-hf
FT model: ./ft-legal-model
Examples: 500
Overall: PASS ✅
====================================================================

ROUGEL:
  Base: 0.3245 ± 0.0210 (95% CI: [0.3120, 0.3370])
  FT:   0.4712 ± 0.0198 (95% CI: [0.4598, 0.4826])
  Diff: +0.1467 (+45.2%)
  p-value: <0.0001
  Cohen's d: 0.89 (LARGE effect)
  Verdict: IMPROVED 🟢

BERtsCORE:
  Base: 0.7823 ± 0.0156 (95% CI: [0.7740, 0.7906])
  FT:   0.8512 ± 0.0142 (95% CI: [0.8440, 0.8584])
  Diff: +0.0689 (+8.8%)
  p-value: <0.0001
  Cohen's d: 0.52 (MEDIUM effect)
  Verdict: IMPROVED 🟢

EXACT_MATCH:
  Base: 0.2830 ± 0.0202 (95% CI: [0.2720, 0.2940])
  FT:   0.4120 ± 0.0221 (95% CI: [0.4000, 0.4240])
  Diff: +0.1290 (+45.6%)
  p-value: <0.0001
  Cohen's d: 0.64 (MEDIUM effect)
  Verdict: IMPROVED 🟢

SAFETY EVALUATION:
  Base model toxic output rate: 0.2%
  FT model toxic output rate: 3.1%
  Change: +15.5x 🔴
  Verdict: FLAGGED FOR REVIEW ⚠️

CAPABILITY RETENTION:
  General QA: -1.2% (NOT SIGNIFICANT) 🟡
  Code generation: -35.4% (SIGNIFICANT) 🔴
  Sentiment analysis: -0.8% (NOT SIGNIFICANT) 🟢
  Named entity recognition: -2.1% (NOT SIGNIFICANT) 🟢

REGRESSION ANALYSIS:
  Found 23 examples where FT score < base score - 0.1
  Common pattern: Multi-document summaries where FT model
  incorrectly merges facts across documents
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Comparing On the Same Distribution You Trained On

**Problem**: You evaluate on test data drawn from the SAME distribution as your training data. The FT model memorized the training distribution, so it naturally scores higher. But it hasn't actually learned to generalize — it just memorized patterns.

**Fix**: Hold out a test set from a DIFFERENT time period, different source, or different domain. If the FT model still scores higher on OUT-OF-DISTRIBUTION data, the improvement is real.

### Antipattern 2: Cherry-Picking Metrics

**Problem**: Your primary task improved (+47% ROUGE-L!). But code generation dropped 35%. You report only the ROUGE-L improvement.

**Fix**: Pre-commit to a set of metrics BEFORE evaluating. Write them down in your eval config. Report ALL metrics, not just the good ones.

### Antipattern 3: Ignoring Statistical Significance

**Problem**: You evaluate on 100 examples. FT model scores 0.47, base scores 0.44. You declare victory. But with 100 examples and high variance, this difference might not be significant.

**Fix**: Always report confidence intervals. If CIs overlap, the difference might be noise. Run more eval examples until CIs separate.

### Antipattern 4: The "One Good Example" Fallacy

**Problem**: You find one example where your FT model produces a brilliant response and the base model fails. You show this example to stakeholders. Everyone is impressed.

**Fix**: The plural of "anecdote" is not "data." Always show DISTRIBUTIONS, not highlights. The average matters more than the best case.

### Common Failure Modes

| Failure | Symptom | Fix |
|---------|---------|-----|
| Data contamination | FT model scores suspiciously high on test set | Use out-of-distribution test set |
| Metric hacking | Metrics improve but quality doesn't | Add human eval or LLM-as-judge |
| Length confound | Longer responses score higher | Length-normalize or group by length |
| Over-evaluation | Model is tested too much → overfits to eval set | Lock eval set, don't iterate based on it |
| Under-evaluation | Too few test examples → high variance | Minimum 200 examples per evaluation |
| No regression testing | FT model loses general capabilities | Add capability benchmark suite |

---

## 🧪 Drills & Challenges

### Drill 1: Build the Comparison Report Generator (45 min)

**Goal**: Extend the evaluation framework to generate a COMPARISON REPORT with visualizations.

**Task**: Add a `generate_report()` method that:
1. Takes comparison results (from `ModelComparator`)
2. Generates a markdown report with:
   - Summary table (all metrics with CIs)
   - Per-metric bar charts with error bars
   - Per-example score distribution histograms
   - Top-10 best/worst examples for FT model
   - Statistical significance table
   - Regression analysis section

**Example output format:**
```markdown
# Evaluation Report: Fine-Tuned Legal Model

## Summary
| Metric | Base | FT | Δ | p-value | Significant? |
|--------|------|----|----|---------|--------------|
| ROUGE-L | 0.3245 | 0.4712 | +45.2% | <0.001 | ✅ |
| BERTScore | 0.7823 | 0.8512 | +8.8% | <0.001 | ✅ |

## Score Distributions
[Histogram images would go here]

## Regression Analysis
23 examples where FT score < base score - 0.1
...
```

**Evaluation criteria:**
- Report is complete (all metrics, CIs, verdicts)
- Visualizations are clear (use matplotlib or plotly)
- Report highlights regressions prominently
- Report includes actionable insights (what to fix)

---

### Drill 2: Detect Data Contamination (45 min)

**Goal**: Build a detector that identifies if your test set overlaps with training data.

**Background**: Data contamination happens when test examples appear (or are very similar to) training examples. This inflates evaluation scores because the model has effectively "seen" the test set during training.

**Task**: Build a contamination detector that:
1. Takes a training file and test file (both JSONL)
2. For each test example, finds the most similar training example
3. Flags test examples with >0.9 similarity to any training example
4. Reports: how many test examples are contaminated, at what similarity thresholds

**Hints:**
- Use `difflib.SequenceMatcher` for string similarity (fast, heuristic)
- Or use embeddings + cosine similarity for semantic similarity (more accurate)
- Contaminated test scores should be REPORTED SEPARATELY (not aggregated)

**Questions to answer:**
1. If 20% of your test set is contaminated, and contaminated examples score 15% higher, how much is your "true" score inflated?
2. At what similarity threshold should you flag contamination? 0.9? 0.8? 0.7?
3. What if a training example is 95% similar to a test example but the KEY DETAIL is different? Is that contamination?

---

### Drill 3: Multi-Metric Decision Framework (45 min)

**Goal**: Build a decision framework that produces a single SHIP/NO-SHIP decision from multiple metrics.

**Background**: You have 5 metrics. Some improved, some regressed. How do you decide if the model is ready to ship?

**Task**: Implement a weighted decision framework:

```python
# Metrics from evaluation
metrics = {
    "rougeL": {"score": 0.47, "baseline": 0.32, "threshold": 0.35},
    "bertscore": {"score": 0.85, "baseline": 0.78, "threshold": 0.80},
    "code_gen": {"score": 0.42, "baseline": 0.65, "threshold": 0.55},
    "safety": {"score": 0.031, "baseline": 0.002, "threshold": 0.01},
    "latency": {"score": 2.3, "baseline": 1.8, "threshold": 2.5},
}

# Weights for each metric (must sum to 1.0)
weights = {
    "rougeL": 0.25,
    "bertscore": 0.20,
    "code_gen": 0.20,
    "safety": 0.25,  # Safety gets high weight
    "latency": 0.10,
}
```

**Requirements:**
- Each metric gets a PASS/FAIL based on threshold
- Weighted score combines all metrics
- SHIP if weighted score > 0.7 AND no CRITICAL failures
- CRITICAL failure: any metric below HARD threshold (safety below 0.01, etc.)
- Report WHY a model failed (which metrics, how much below threshold)

**Questions to answer:**
1. How do you set thresholds? What if the base model doesn't meet them?
2. Should weights be fixed or task-dependent?
3. What if a metric that was weighted 0.05 fails catastrophically? Does the model still ship?

---

## 🚦 Gate Check

Before moving to File 06 (Function-Calling Fine-Tuning), verify you can:

1. **Design an evaluation** — Given a fine-tuning task (e.g., summarization, classification, generation), design an evaluation plan with: 3+ metrics, test set size, confidence level, regression tests

2. **Read an eval report** — Given this output:

```
ROUGE-L: Base 0.32 ± 0.02, FT 0.41 ± 0.03, p=0.04
BERTScore: Base 0.78 ± 0.01, FT 0.82 ± 0.02, p=0.02
Safety: Base 0.2%, FT 4.5%, p<0.001
Code: Base 0.65 ± 0.03, FT 0.44 ± 0.04, p<0.001
```

Answer: Does this model ship? Why? What's the biggest risk?

3. **Detect evaluation confounders** — Given a scenario where ROUGE-L improved 30% but human evaluators prefer the base model, identify the likely confounders (length, style, vocabulary overlap)

4. **Build a regression suite** — For a customer support fine-tuning project, specify:
   - 3 capabilities to test for regression
   - 1 safety dimension to evaluate
   - Minimum example counts for each
   - Decision rule for shipping

5. **Implement statistical comparison** — Given two arrays of scores (base and FT), implement: paired t-test, Cohen's d, bootstrap CI, and report whether the difference is significant AND practically meaningful

---

## 📚 Resources

### Evaluation Best Practices
- **[HELM Benchmark](https://crfm.stanford.edu/helm/latest/)** — Stanford's holistic evaluation framework
- **[Beyond the Imitation Game](https://arxiv.org/abs/2206.04615)** — BIG-bench, 204 tasks for LLM evaluation
- **[Evaluation Best Practices for LLMs](https://docs.google.com/document/d/1rHdvhnhJxJhJ3NqA7Y9PxJM9ZQ5Oqb4QH6Ox4MzLqVw/edit)** — Google's internal eval guidelines (public)

### Statistical Methods
- **[Bootstrap Confidence Intervals](https://arxiv.org/abs/math/0702842)** — Non-parametric CI estimation
- **[Effect Size Guidelines](https://www.psychologytoday.com/us/blog/evidence-based-living/202108/cohens-d-what-is-it-and-when-should-you-use-it)** — When is a difference "practically significant"?

### Phase Cross-References
- **Phase 7 (Evals)** → Your Phase 7 eval platform is the runtime for these comparisons. Wire your FT vs base evaluation into the Phase 7 platform for continuous monitoring.
- **Phase 4 (RAG)** → If you're fine-tuning for RAG, your evaluation must measure RETRIEVAL QUALITY (precision, recall, MRR) separately from GENERATION QUALITY (faithfulness, helpfulness).
- **Phase 8 (Guardrails)** → Every fine-tuned model needs guardrail regression testing. Fine-tuning can introduce safety regressions that your Phase 8 guardrails must catch.

> **Next up: File 06 — Fine-Tuning for Function Calling.** Standard fine-tuning improves text generation. But what if you want the model to USE TOOLS? Function-calling fine-tuning is a SPECIALIZED skill — the training data format, evaluation, and failure modes are fundamentally different from text generation FT.
