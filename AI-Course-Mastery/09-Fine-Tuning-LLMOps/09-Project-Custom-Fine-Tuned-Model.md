# 09 — Project: Custom Fine-Tuned Model (Portfolio Capstone)

## 🎯 Purpose & Goals

> 🛑 STOP. Before you start this project, you need to understand what makes this DIFFERENT from the previous 8 files:

**Every file so far taught you a SKILL. This file teaches you a PROCESS.**

Fine-tuning in isolation is easy. Fine-tuning for a REAL use case — with data you collected, evaluation you designed, and a deployment that real users will hit — is HARD. This project is where you prove you can do the hard thing.

**Project Philosophy:**
- You choose the use case (I'll give you 4 options + templates)
- You make the design decisions (I'll give you the questions, not the answers)
- You execute the full pipeline (data → train → eval → deploy → document)
- The output is a PORTFOLIO-README.md that demonstrates senior-level thinking

**The goal of this module: Build a complete, portfolio-ready fine-tuning project that proves you can ship custom AI models to production.**

---

### 🤔 Discovery Scenario: The Portfolio Paradox

You have two options for your portfolio project:

**Option A**: Fine-tune a model on a POPULAR dataset (legal summarization, medical Q&A, code generation).
- Pros: Clear benchmarks, easy to explain, recruiters recognize the domain
- Cons: Hundreds of other candidates have the SAME project. Yours doesn't stand out.

**Option B**: Fine-tune a model on your PERSONAL data or a NICHE use case you care about.
- Pros: Unique story, demonstrates initiative, shows domain expertise
- Cons: Harder to evaluate (no standard benchmark), harder to explain, you have to DEFEND your design decisions

**🤔 Before reading on, answer these:**

1. In a job interview, you show Option A. The interviewer says: "This is the same as every other legal summarization project. What did YOU actually learn?" How do you answer? (Notice: if your only answer is about the TECHNIQUE — "I used LoRA with rank 16" — that's not differentiating. The interviewer wants to see your DECISION-MAKING.)

2. You pick Option B. The interviewer says: "Your test set only has 100 examples because you couldn't collect more. How do I know your model actually improved?" How do you defend a project with LIMITED data? What EVALUATION methodology makes small-dataset projects credible?

3. **The hardest question:** Your portfolio project will be seen by 3 types of people:
   - Recruiters (30 seconds): Read the title and skim the results
   - Technical screeners (5 minutes): Check the approach and code quality  
   - Hiring managers (15 minutes): Evaluate the decision-making and tradeoffs
   
   **How do you structure a README that serves ALL THREE audiences without being 10 pages long?** What's in the first 30 seconds vs. what's in the appendices?

---

## ⏱ Time Budget

| Phase | Time |
|-------|------|
| Choose use case & design | 1 hr |
| Data collection & preparation | 3 hr |
| Training (multiple runs) | 3 hr |
| Evaluation & comparison | 2 hr |
| Deployment | 1 hr |
| README & documentation | 1 hr |
| **Total** | **~11 hr** |

---

## 📋 Project Options

Choose ONE of the following four use cases. Each has different challenges:

### Option 1: Email Response Generator

**Use case**: Fine-tune a model to generate professional email responses in YOUR writing style.

**Dataset**: Your own sent emails (export from Gmail/Outlook).
- Collect 500-2,000 sent emails
- Format: `[Email thread context] → [Your response]`
- Challenge: Privacy — you need to strip PII (use Phase 8 PII redactor!)

**Why this is a good project:**
- Extremely personal — you can DEMO it with your own emails
- Obvious value — everyone hates writing emails
- Limited data — shows you can fine-tune with small datasets
- Easy to evaluate — "Does this sound like me?" is intuitive

**Key challenges:**
- Privacy concerns (showing real emails in portfolio?)
- Writing style is subjective (how do you evaluate "sounds like me"?)
- Email threading (long context windows needed)

### Option 2: Domain-Specific Q&A (Your Expertise)

**Use case**: Fine-tune a model on YOUR area of expertise — something you know deeply that a general model doesn't handle well.

**Dataset**: 
- You write 200-500 Q&A pairs from your knowledge
- Or extract from your company's internal docs (with permission!)
- Or use a niche technical domain you understand

**Examples:**
- "Kubernetes networking debugging" if you're a DevOps engineer
- "FastAPI anti-patterns" if you're a Python backend engineer
- "React performance optimization" if you're a frontend engineer

**Why this is a good project:**
- You are the domain EXPERT — you can judge quality
- It's unique — no one else has YOUR project
- Shows deep domain knowledge + AI skills = senior hybrid engineer

**Key challenges:**
- Writing 500 high-quality Q&A pairs takes time
- Small dataset requires careful training (regularization, early stopping)
- Evaluation requires MANUAL review (you're the expert)

### Option 3: Structured Data Extraction (Technical)

**Use case**: Fine-tune a model to extract structured data from UNSTRUCTURED text.

**Dataset**: 
- 1,000-5,000 text → JSON pairs
- Use invoices, receipts, medical notes, legal documents
- Or synthetic data from public datasets (e.g., extract events from news articles)

**Format Example:**
```
Input: "Meeting with Acme Corp on June 15th at 2pm to discuss Q3 budget. 
        Attendees: John (CFO), Sarah (VP Eng). Location: Zoom."
        
Output: {
  "event_type": "meeting",
  "organization": "Acme Corp",
  "date": "2024-06-15",
  "time": "14:00",
  "attendees": ["John", "Sarah"],
  "topics": ["Q3 budget"],
  "location": "Zoom"
}
```

**Why this is a good project:**
- Clear, measurable evaluation (exact match, F1 on extraction)
- Obvious business value (automating data entry)
- Tests function-calling + structured output skills

**Key challenges:**
- Creating high-quality annotations is time-consuming
- Some fields are harder to extract than others (date normalization!)
- Requires careful evaluation of PARTIAL correctness

### Option 4: Code Review Assistant (Best for Your Background)

**Use case**: Fine-tune a model to review code and suggest improvements — in YOUR preferred coding style.

**Dataset**: 
- Collect your own PR reviews (from GitHub/GitLab)
- Or write 200-500 code snippets with "what's wrong and how to fix it"
- Use multiple languages you work with (Python, TypeScript, etc.)

**Format:**
```
Input:
```python
def process_data(items):
    result = []
    for i in range(len(items)):
        if items[i] > 0:
            result.append(items[i] * 2)
    return result
``

Output:
## Issues Found
1. **Inefficient loop**: Use list comprehension instead of manual indexing
2. **Type hints missing**: Add function signature type hints
3. **Naming**: `items` is too generic for production code

## Suggested Fix:
```python
def process_data(items: list[float]) -> list[float]:
    \"\"\"Filter and transform positive values.\"\"\"
    return [item * 2 for item in items if item > 0]
```
```

**Why this is a good project:**
- You're an experienced developer — you know good code from bad
- Extremely valuable — every team needs better code review
- Tests function-calling (format output)
- Connects to everything you've learned (prompt engineering, evals, deployment)

**Key challenges:**
- Code review is SUBJECTIVE (your style vs. others')
- Multiple languages increase data requirements
- Evaluation needs HUMAN review (no automated metric captures good code review)

---

## 🏗️ Project Pipeline

### Phase 1: Design & Planning (1 hour)

Before writing any code, answer these design questions:

1. **Which use case?** Choose one from above or propose your own.
2. **What's the success criteria?** How will you PROVE the model improved?
3. **What's the baseline?** Base model? Prompt-engineered base model? RAG?
4. **What's the minimal viable quality?** At what score would you say "this is worth deploying"?
5. **What data do you need?** Quantity, format, source, privacy considerations.
6. **What's the training approach?** Full FT, LoRA, QLoRA? Rank? Target modules?
7. **How will you evaluate?** Metrics, test set, human eval, LLM-as-judge?
8. **How will you deploy?** vLLM? Ollama? API wrapper?

**Deliverable**: Write down your answers. This is your project plan.

**🤔 Design Questions (must answer before coding):**

1. Your dataset has 500 examples. A LoRA rank of 32 has 8M trainable parameters. A rank of 8 has 2M. With 500 examples, which rank is MORE likely to overfit? Why? How do you choose the right rank for your dataset size?

2. Your evaluation shows the FT model scoring 0.45 ROUGE-L vs. base at 0.32. But you spent 8 hours collecting data, 3 hours training, and $5 on GPU credits. If the base model with a 3-shot prompt scores 0.41 ROUGE-L, was the fine-tuning worth it? DEFINE your threshold for "worth it" based on effort, cost, and quality.

3. You deploy your model on a T4 GPU (16GB). Your 7B model at 4-bit uses 5GB. But at 10 concurrent users, it OOMs because the KV cache fills the remaining 11GB. You can't afford a bigger GPU. **How do you fit your model + serving within 16GB?** (Hint: Reduce max_model_len, reduce max_num_seqs, use prefix caching, or quantize more aggressively — each has tradeoffs.)

### Phase 2: Data Collection & Preparation (3 hours)

#### Step 1: Collect or generate your dataset

For each use case:

**Email Generator** → Export from Gmail:
```python
import json
from googleapiclient.discovery import build

def export_emails(service, max_results=2000):
    """Export sent emails for fine-tuning dataset."""
    # Your code here — use Gmail API to fetch sent messages
    # Format: prompt (email thread) → completion (your response)
    pass
```

**Domain Q&A** → Write Q&A pairs:
```json
[
  {
    "prompt": "What's the difference between a ConfigMap and a Secret in Kubernetes?",
    "completion": "Both store configuration data, but Secrets are base64-encoded and intended for sensitive data. ConfigMaps store plaintext configuration. Secrets also have optional encryption at rest and more restrictive RBAC defaults. For production, use Secrets for credentials, ConfigMaps for everything else."
  },
  ...
]
```

**Structured Extraction** → Annotate text→JSON pairs:
```json
[
  {
    "prompt": "Patient: John Doe, DOB: 1985-03-22, Diagnosis: Type 2 Diabetes, Medication: Metformin 500mg twice daily, Next appointment: 2024-07-15 with Dr. Smith",
    "completion": "{\"patient_name\": \"John Doe\", \"dob\": \"1985-03-22\", \"diagnosis\": \"Type 2 Diabetes\", \"medication\": \"Metformin 500mg\", \"dosage\": \"twice daily\", \"next_appointment\": \"2024-07-15\", \"physician\": \"Dr. Smith\"}"
  },
  ...
]
```

**Code Review** → Create buggy→fixed code pairs:
```json
[
  {
    "prompt": "Review this Python code:\n\ndef calc(a, b):\n    return a * (b - a)\n",
    "completion": "## Issues\n1. **Unclear naming**: `a`, `b` don't describe the parameters\n2. **No docstring**: Purpose of the function is unclear\n3. **No type hints**: Input/output types not specified\n\n## Fixed\n\ndef calculate_product(base: float, adjustment: float) -> float:\n    \"\"\"Calculate product with adjustment factor.\"\"\"\n    return base * (adjustment - base)\n"
  },
  ...
]
```

#### Step 2: Clean and validate

**🤔 Two critical questions to answer HERE (not later):**

1. **Label consistency**: You have 500 Q&A pairs written over 3 days. Day 1 answers are detailed (3 paragraphs). Day 3 answers are concise (2 sentences). If you train on inconsistent verbosity, the model will produce MID-LENGTH answers — which NEITHER style prefers. **How do you normalize your dataset so the model learns the style you WANT, not the average of all your moods?**

2. **Train/val/test split**: You have 500 examples. Should you use 80/10/10 (400 train, 50 val, 50 test)? Or 70/15/15? Or k-fold cross-validation? With only 50 test examples, your evaluation has WIDE confidence intervals (a 2-example difference changes the score by 4%). **What split strategy gives you RELIABLE evaluation without wasting too many training examples?**

#### Step 3: Format for training

```python
"""
Convert your dataset to the format expected by SFTTrainer.
"""
from datasets import Dataset
from transformers import AutoTokenizer
import json

def prepare_dataset(
    data_file: str,
    model_name: str = "mistralai/Mistral-7B-Instruct-v0.2",
    output_file: str = "formatted_dataset.jsonl",
):
    """Format raw data for fine-tuning."""
    
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    tokenizer.pad_token = tokenizer.eos_token
    
    with open(data_file, 'r') as f:
        examples = [json.loads(line) for line in f]
    
    formatted = []
    for ex in examples:
        messages = [
            {"role": "user", "content": ex["prompt"]},
            {"role": "assistant", "content": ex["completion"]},
        ]
        
        formatted_text = tokenizer.apply_chat_template(
            messages, tokenize=False, add_generation_prompt=False
        )
        
        formatted.append({"text": formatted_text})
    
    dataset = Dataset.from_list(formatted)
    dataset.save_to_disk(f"formatted_{output_file}")
    
    # Also save train/val/test splits
    splits = dataset.train_test_split(test_size=0.1, seed=42)
    val_test = splits['test'].train_test_split(test_size=0.5, seed=42)
    
    splits['train'].save_to_disk("data/train")
    val_test['train'].save_to_disk("data/val")
    val_test['test'].save_to_disk("data/test")
    
    print(f"Train: {len(splits['train'])} examples")
    print(f"Val: {len(val_test['train'])} examples")
    print(f"Test: {len(val_test['test'])} examples")
    
    return splits['train'], val_test['train'], val_test['test']
```

### Phase 3: Training (3 hours)

**🤔 Critical training decisions:**

1. **Base model selection**: For most custom datasets, Mistral-7B-Instruct or Llama-3-8B-Instruct is the right choice. But what if your use case is function-calling? Should you start from a BASE model, an INSTRUCT model, or a function-calling-specific model (like Gorilla)? **What are the tradeoffs of each starting point for the 4 use cases?**

2. **Training iterations**: You'll likely need 3-5 training runs:
   - Run 1: Baseline LoRA config (rank=8, LR=2e-4, 3 epochs) — see if training works
   - Run 2: Adjust rank/LR based on Run 1 eval loss curve
   - Run 3: Add regularization if overfitting, or more epochs if underfitting
   - Run 4: Fine-tune hyperparameters based on validation set
   - Run 5: Final train on train+val, evaluate on test (ONLY ONCE)

   **How many runs do you actually need?** Each run costs $1-5 in GPU credits. With 10 runs, you've spent $50 and 5 hours just waiting for training. With 2 runs, you might miss the optimal config. **What's the MINIMUM number of training runs to get a reliable result?**

3. **When to stop training**: You're monitoring validation loss every 50 steps. It goes down for 200 steps, then STAYS FLAT for 100 steps, then goes UP. The "early stopping" heuristic says stop when val loss increases for N consecutive evaluations. **What should N be for your dataset size?** (Hint: With 500 examples, val loss is noisy. A single-point increase might be noise. With 5,000 examples, val loss is smoother. A smaller N is appropriate.)

#### Training Script Template

```python
#!/usr/bin/env python3
"""
Template training script — customize for your project.

Usage:
    python train.py --data ./data/train --model mistralai/Mistral-7B-Instruct-v0.2
"""

import torch
from datasets import load_from_disk
from transformers import (
    AutoTokenizer, AutoModelForCausalLM,
    BitsAndBytesConfig, TrainingArguments
)
from trl import SFTTrainer
from peft import LoraConfig, get_peft_model
import argparse

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--data", required=True, help="Training data directory")
    parser.add_argument("--model", default="mistralai/Mistral-7B-Instruct-v0.2")
    parser.add_argument("--output", default="./ft-output")
    parser.add_argument("--lr", type=float, default=2e-4)
    parser.add_argument("--epochs", type=int, default=3)
    parser.add_argument("--rank", type=int, default=16)
    parser.add_argument("--batch-size", type=int, default=4)
    parser.add_argument("--grad-accum", type=int, default=4)
    args = parser.parse_args()
    
    # Load datasets
    train_data = load_from_disk(f"{args.data}/train")
    val_data = load_from_disk(f"{args.data}/val")
    
    # Load tokenizer
    tokenizer = AutoTokenizer.from_pretrained(args.model)
    tokenizer.pad_token = tokenizer.eos_token
    
    # Quantization
    bnb_config = BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_compute_dtype=torch.float16,
        bnb_4bit_quant_type="nf4",
    )
    
    # Load model
    model = AutoModelForCausalLM.from_pretrained(
        args.model,
        quantization_config=bnb_config,
        device_map="auto",
        torch_dtype=torch.float16,
    )
    
    # LoRA config
    lora_config = LoraConfig(
        r=args.rank,
        lora_alpha=args.rank * 2,
        target_modules=["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"],
        lora_dropout=0.05,
        bias="none",
        task_type="CAUSAL_LM",
    )
    model = get_peft_model(model, lora_config)
    model.print_trainable_parameters()
    
    # Training args
    training_args = TrainingArguments(
        output_dir=args.output,
        per_device_train_batch_size=args.batch_size,
        per_device_eval_batch_size=args.batch_size,
        gradient_accumulation_steps=args.grad_accum,
        learning_rate=args.lr,
        lr_scheduler_type="cosine",
        warmup_ratio=0.03,
        num_train_epochs=args.epochs,
        logging_steps=10,
        eval_steps=50,
        save_steps=500,
        evaluation_strategy="steps",
        save_total_limit=2,
        load_best_model_at_end=True,
        metric_for_best_model="eval_loss",
        fp16=True,
        max_grad_norm=0.3,
        report_to="wandb",
        run_name=f"ft-{args.model.split('/')[-1]}-r{args.rank}",
    )
    
    # Trainer
    trainer = SFTTrainer(
        model=model,
        args=training_args,
        train_dataset=train_data,
        eval_dataset=val_data,
        tokenizer=tokenizer,
        dataset_text_field="text",
        max_seq_length=2048,
    )
    
    # Train
    trainer.train()
    
    # Save
    trainer.save_model(f"{args.output}/final")
    tokenizer.save_pretrained(f"{args.output}/final")
    
    print(f"Model saved to {args.output}/final")

if __name__ == "__main__":
    main()
```

### Phase 4: Evaluation & Comparison (2 hours)

```python
"""
Template evaluation script — compare FT model vs. base model.

This uses the evaluation framework from File 05.
"""

import json
import numpy as np
from eval_framework import EvalConfig, EvalRunner, ModelComparator

def evaluate_models(
    base_model: str,
    ft_model: str,
    test_file: str,
    output_file: str = "eval_results.json",
):
    """Evaluate and compare base vs. FT model on your test set."""
    
    with open(test_file, 'r') as f:
        test_data = [json.loads(line) for line in f]
    
    # Configure evaluation
    config = EvalConfig(
        seed=42,
        batch_size=4,
        max_new_tokens=256,
        temperature=0.1,
    )
    
    runner = EvalRunner(config)
    comparator = ModelComparator(config)
    
    # Evaluate both
    print(f"Evaluating base model: {base_model}")
    base_results = runner.evaluate_model(base_model, test_data, "base")
    
    print(f"Evaluating FT model: {ft_model}")
    ft_results = runner.evaluate_model(ft_model, test_data, "fine-tuned")
    
    # Compare
    comparison = comparator.compare(base_results, ft_results)
    
    # Save
    with open(output_file, 'w') as f:
        json.dump({
            "comparison": {k: v for k, v in comparison.items() if k != 'metrics' or k != 'per_example'},
            "base_metrics": base_results,
            "ft_metrics": ft_results,
        }, f, indent=2, default=str)
    
    # Print summary
    print(f"\n{'='*60}")
    print(f"EVALUATION RESULTS")
    print(f"{'='*60}")
    print(f"Overall: {comparison['overall_verdict']}")
    
    for metric, data in comparison['metrics'].items():
        print(f"\n{metric.upper()}:")
        print(f"  Base: {data['base_mean']:.4f}")
        print(f"  FT:   {data['ft_mean']:.4f}")
        print(f"  Δ:    {data['difference']:+.4f} ({data['difference_pct']:+.1f}%)")
        print(f"  Sig:  {'✅' if data['is_significant'] else '❌'} (p={data['p_value']:.4f})")
    
    return comparison
```

**🤔 Evaluation questions you must answer:**

1. Your task is EMAIL GENERATION. What metric do you use? ROUGE-L? That measures n-gram overlap with your reference email — but there's no SINGLE "correct" email response. Many valid responses exist. **How do you evaluate a task with MULTIPLE valid outputs?**

2. Your FT model scores 0.52 ROUGE-L. Your prompt-engineered base model scores 0.48. The difference is statistically significant (p=0.03). But the FT model ALSO takes 2x longer to generate responses (1.8s vs. 0.9s). **At what latency difference is the quality improvement NOT worth it?** Define your threshold BEFORE evaluating.

3. Your test set has 50 examples. You run evaluation 3 times (different random seeds → different sampling). Results: Run 1: 0.52, Run 2: 0.49, Run 3: 0.54. The mean is 0.517 with std of 0.025. Your base model is 0.48. The difference (0.037) is within 2 std deviations. **Is your FT model actually better, or is the result just noise?** How many test examples do you need to be confident?

### Phase 5: Deployment (1 hour)

```python
"""
Template deployment script — serve your fine-tuned model.
"""

# 1. Convert to a format suitable for inference
# If trained with LoRA, merge the adapter weights:
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer

base_model = AutoModelForCausalLM.from_pretrained("mistralai/Mistral-7B-Instruct-v0.2")
ft_model = PeftModel.from_pretrained(base_model, "./ft-output/final")
merged_model = ft_model.merge_and_unload()
merged_model.save_pretrained("./ft-merged-model")
tokenizer = AutoTokenizer.from_pretrained("./ft-output/final")
tokenizer.save_pretrained("./ft-merged-model")
print("Merged model saved to ./ft-merged-model")

# 2. Serve with vLLM
# docker run --gpus all -p 8000:8000 \\
#   -v ./ft-merged-model:/model \\
#   vllm/vllm-openai:latest \\
#   --model /model \\
#   --quantization awq \\
#   --max-model-len 4096

# 3. Test inference
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="placeholder",
)

response = client.chat.completions.create(
    model="default",
    messages=[
        {"role": "user", "content": "Your test prompt here"}
    ],
    max_tokens=200,
)
print(response.choices[0].message.content)
```

**🤔 Deployment decision questions:**

1. Your model was fine-tuned with 4-bit QLoRA. You want to deploy. Should you:
   - Option A: Deploy the QLoRA adapter on top of the base model (requires downloading base + adapter at inference time)
   - Option B: Merge LoRA weights into base model, then quantize to 4-bit for deployment
   - Option C: Keep in FP16 (full precision) for maximum quality, deploy on larger GPU
   
   Each has tradeoffs in quality, latency, and deployment complexity. Which do you choose for your use case?

2. Your use case is CODE REVIEW. The model needs 2,048 tokens of context (code snippets). But deployment GPU has 16GB, and max_model_len=8192 consumes too much KV cache. **What's the minimum max_model_len that covers 95% of your use case?** Check your data to answer this. (Hint: Plot the distribution of code snippet lengths in your test set. The 95th percentile is your answer.)

3. You deploy with vLLM. After 24 hours, you notice the KV cache is growing steadily but NEVER shrinking. Your max_num_seqs is 256 but average concurrent users is 12. **Why isn't the KV cache being evicted?** (Hint: Long-lived connections. If users keep connections open, vLLM keeps their KV cache. You need to check: idle timeout settings, connection handling in your API layer, and whether sessions are properly closed.)

### Phase 6: README & Documentation (1 hour)

This is the most important deliverable. Your README is what recruiters and hiring managers will see.

**Mandatory Sections:**

```markdown
# Project: [Use Case] Fine-Tuned Model

## Overview
<!-- 3 sentences max. What, why, result. -->
I fine-tuned [model] on [dataset description] to [capability]. 
The result: [metric] improved from [base] to [FT], a [X]% improvement.
Deployed at [link if applicable].

## Architecture
<!-- Architecture diagram (ASCII or image) showing: -->
<!-- Data → Train → Eval → Deploy pipeline -->

## Dataset
- Source: [where data came from]
- Size: [N] examples
- Split: [train/val/test] = [N1/N2/N3]
- Cleaning: [deduplication, quality filtering, normalization]
- Format: [chat template, function-calling format]
- **Challenge**: [e.g., "Only 500 examples available — explain how you handled small data"]

## Training
- Base model: [model name]
- Technique: [LoRA / QLoRA / Full FT]
- Hyperparameters:
  - Rank: [N]
  - Learning rate: [value]
  - Batch size: [effective batch size]
  - Epochs: [N]
  - Hardware: [GPU type, hours]
  - Cost: [$X.XX]
- **Key decisions**:
  - Why [rank]? [Brief reasoning]
  - Why [LR]? [Brief reasoning]
  - Why [base model]? [Brief reasoning]

## Results
| Metric | Base Model | FT Model | Change |
|--------|-----------|----------|--------|
| [Metric 1] | [base] | [ft] | [+X%] |
| [Metric 2] | [base] | [ft] | [+X%] |
| Latency (p50) | [base] | [ft] | [+X%] |

**Statistical significance**: [explain]

### Qualitative Examples
<!-- Show 3 comparison examples: base vs FT side by side -->
<!-- Include 2 where FT is better AND 1 where base is better -->

## Deployment
- Serving framework: [vLLM / TGI / Ollama]
- Quantization: [AWQ / GPTQ / none]
- GPU: [type, count]
- Cost: [$X/hr, ~$X/month at expected load]
- Latency: [p50/p95/p99]

## What I Learned
<!-- 3-5 bullet points. Show SENIOR thinking, not technique. -->
- [Learning about tradeoffs]
- [Learning about evaluation]
- [Learning about data quality]
- [Learning about deployment]

## Links
- Code: [GitHub repo]
- Model: [HuggingFace link if uploaded]
- Demo: [link if deployed]
```

---

## ✅ Good Output Examples

### What an A+ Portfolio README Looks Like

```markdown
# Project: Kubernetes Troubleshooting Assistant

## Overview
I fine-tuned Mistral-7B-Instruct on 800 real Kubernetes debugging scenarios to 
create an assistant that diagnoses cluster issues. The FT model achieves 73% 
accuracy on diagnosis (vs. 54% for the base model with few-shot prompting) and 
generates actionable fix steps 2x faster than consulting docs.

## Architecture
```
K8s Debugging Cases (800)
        │
        ▼
   Format: User query → Diagnoses + Fix steps
        │
        ├── Base: Mistral-7B-Instruct → Prompt engineer (5-shot) → 54% accuracy
        └── FT: LoRA rank=16 → Fine-tuned → 73% accuracy (+19pp ↑)
        │
        ▼
   vLLM serving (AWQ 4-bit, T4 GPU)
        │
        ▼
   Slack bot interface
```

## Dataset
- **Source**: 3 years of my personal K8s debugging notes + Stack Overflow K8s questions
- **Size**: 800 scenarios (650 train, 75 val, 75 test)
- **Challenge**: Many scenarios have similar symptoms but different root causes 
  (e.g., "pod crash loop" could be OOM, liveness probe failure, or config error).
  I ensured each diagnosis clearly distinguishes between similar-presenting issues.

## Training
- **Approach**: QLoRA (4-bit base, rank=16, alpha=32)
- **Hardware**: 1x RTX 4090 (24GB), 45 minutes per epoch
- **Cost**: ~$12 total (RunPod rental)
- **Key decision**: I tested rank={8, 16, 32} and found rank=16 gave the best 
  accuracy-to-overfit ratio. Rank=8 underfit (67%), rank=32 overfit (val loss 
  increased after epoch 2 despite train loss still dropping).

## Results
| Metric | Base (5-shot) | FT Model | Δ |
|--------|--------------|----------|----|
| Diagnosis Accuracy | 54% | 73% | +35% |
| Fix Step Quality (LLM-judge) | 6.2/10 | 8.1/10 | +31% |
| Latency p50 | 1.2s | 1.3s | +8% (acceptable) |

### Where FT Excels
```
User: "Pods are crash looping. describe pod shows OOMKilled"
Base: "Check resource limits and increase memory" (too generic)
FT:  "OOMKilled indicates the pod exceeded its memory limit. 
      First: `kubectl top pod <pod>` to see actual usage.
      If actual > limit: increase memory limit in deployment.
      If actual << limit: check for memory leak in the app.
      Also verify: `kubectl describe node` for node pressure." 
      ✓ Specific, actionable, prioritized
```

## Deployment
- Serving: vLLM with AWQ 4-bit on T4 GPU ($0.50/hr)
- Integration: Slack bot — users text, bot responds with diagnosis
- Scale: Handles ~50 requests/day with p99 latency of 3.2s

## What I Learned
1. **Small data requires careful regularization**. With 650 training examples, 
   a single overfitting epoch ruined the model. Early stopping on val loss 
   was essential — I couldn't rely on fixed epoch count.
2. **K8s problems have similar-sounding symptoms with different causes**. 
   My first dataset had too many "pod crash loop" entries with similar answers. 
   I had to REDUCE similarity in the dataset to force the model to learn 
   differential diagnosis.
3. **Domain experts make better evaluators than metrics**. ROUGE-L scored my 
   model 0.52 vs base 0.48 — a small difference. But human evaluation showed 
   the FT model's diagnoses were much more SPECIFIC and actionable. 
   Automated metrics missed the most important quality dimension.
4. **Deployment cost surprised me**. At $0.50/hr for a T4, I expected $12/day 
   for always-on. But at 50 requests/day, it's actually $2/request in GPU 
   cost — terrible value. I switched to serverless (Replicate) which costs 
   ~$0.05/request at this volume. A/B test your cost assumptions.
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: The "Perfect" Project That Doesn't Exist

**Problem**: You spend 3 weeks collecting the PERFECT dataset, 2 weeks training the PERFECT model, 1 week writing the PERFECT README. The project is amazing — but it never actually ships.

**Fix**: Set a START DATE and a SHIP DATE. The project ships on the ship date, even if it's not perfect. Imperfect but real > perfect but imaginary.

### Antipattern 2: Training Without a Baseline

**Problem**: You train the FT model, get great results, and declare victory. But you never compared against a prompt-engineered base model. The "great results" might be achievable with 3 lines of prompt engineering.

**Fix**: ESTABLISH THE BASELINE BEFORE TRAINING. Run the base model (with and without few-shot prompting) on your test set. If the base model with prompting is within 10% of your FT model, reconsider whether FT is worth the effort.

### Antipattern 3: The "I Forgot to Save the Config" Problem

**Problem**: You run 5 experiments. Each one uses a slightly different config. You find the best one. But you didn't save the config — you just remember "it was rank 16 or 32, with LR around 2e-4." Now you can't reproduce your own best result.

**Fix**: LOG EVERYTHING. Use a config file (not command-line args). Save the config alongside the model weights. Use the model registry from File 07.

### Antipattern 4: Cherry-Picking Results

**Problem**: Out of 10 metrics, your FT model improved on 6, stayed same on 2, and regressed on 2. Your README shows only the 6 improvements.

**Fix**: REPORT ALL METRICS. A senior engineer who sees ONLY improvements is suspicious. Seeing regressions alongside improvements shows you're thorough and honest.

### Common Project Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| Data contamination | FT model suspiciously high test scores | Use out-of-distribution test set |
| Overfitting | High train accuracy, low val accuracy | Early stopping, lower rank, more data |
| Catastrophic forgetting | FT model loses general capabilities | Add capability regression test to eval |
| Runtime error | Model works in notebook but breaks in deployment | Containerize early, test in serving environment |
| Wrong base model | FT model doesn't improve on target task | Check if the base model can do the task at all with prompting |

---

## 🚦 Gate Check

Before marking this project as COMPLETE, verify:

### Deliverables

1. **✅ 1 fine-tuned model** — Saved in a format ready for deployment (merged + quantized or adapter weights + base model)

2. **✅ 1 evaluation report** — Comparing FT model vs. baseline (base model with prompting) with: multiple metrics, confidence intervals, statistical significance, per-example analysis

3. **✅ 1 deployment** — Model is running (even if just on your laptop) and can accept requests via API

4. **✅ 1 GitHub repo** — Contains:
   - `README.md` (following the template above)
   - `data/` — Dataset (or clear instructions to reproduce it)
   - `train.py` — Training script
   - `eval.py` — Evaluation script
   - `deploy/` — Deployment configuration
   - `requirements.txt` — Dependencies

### Self-Assessment Questions

Answer these honestly in your README:

1. **Would I use this model myself?** If not, why would anyone else?
2. **What's the WEAKEST part of this project?** (Data quality? Evaluation? Deployment?) Be honest — every project has weaknesses.
3. **If I had 10x more data, how would my approach change?** (If it wouldn't change, your approach is robust. If it would change completely, you compromised your design for small data — be transparent about this.)
4. **What's the most important lesson I learned from this project that I'll apply to the NEXT fine-tuning project?**
5. **Is this project WORTH putting in my portfolio?** (If yes, why? If no, what would make it worth it?)

---

## 📚 Resources

### Reference Projects
- **[HuggingFace Fine-Tuning Examples](https://huggingface.co/docs/peft/en/developer_guides/fine-tune)** — Official PEFT fine-tuning examples
- **[Unsloth Examples](https://github.com/unslothai/unsloth?tab=readme-ov-file#-how-to-install---)** — Example notebooks for various models
- **[OpenAI Fine-Tuning Guide](https://platform.openai.com/docs/guides/fine-tuning)** — If using hosted fine-tuning

### Portfolio Inspiration
- **[Chip Huyen's ML Portfolio Tips](https://huyenchip.com/2023/01/28/how-to-build-a-data-science-portfolio.html)** — What makes a portfolio stand out
- **[Andrej Karpathy's Projects](https://github.com/karpathy)** — Minimal but impactful project repos

### Dataset Sources
- **[HuggingFace Datasets](https://huggingface.co/datasets)** — Thousands of public datasets
- **[Kaggle Datasets](https://www.kaggle.com/datasets)** — Competition and community datasets
- Your OWN data (emails, code reviews, domain knowledge) — Most differentiated option

### Phase Cross-References
- **ALL of Phase 9** → This project IS Phase 9. Every file (00-08) feeds into this.
- **Phase 7 (Evals)** → Use your Phase 7 eval platform for continuous evaluation after deployment.
- **Phase 8 (Guardrails)** → Deploy your Phase 8 guardrails in front of your FT model.
- **Phase 10 (Infrastructure)** → Migrate your deployment to the Phase 10 production infrastructure.
- **Phase 11 (Capstone)** → This model could be a COMPONENT of your capstone project.

---

> **Congratulations! You've completed Phase 9: Fine-Tuning & LLMOps. 🎉**
>
> You've gone from "I've never fine-tuned a model" to:
> - Understanding when to fine-tune (and when NOT to)
> - Building LoRA adapters from scratch
> - Preparing high-quality training datasets
> - Running training with proper monitoring and debugging
> - Evaluating FT models against baselines with statistical rigor
> - Fine-tuning for specialized tasks (function calling)
> - Managing models with a registry and experiment tracking
> - Deploying models to production with vLLM
> - Shipping a complete, portfolio-worthy fine-tuning project
>
> **Next up: Phase 10 — Deployment & Scale.** You have a working model. Now make it RESILIENT, COST-EFFECTIVE, and READY FOR REAL USERS at scale.
