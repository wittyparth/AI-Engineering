# 01 — Fine-Tuning Fundamentals: What Actually Happens During Training

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about loss functions and gradient descent, I need you to understand something critical:

**Fine-tuning is NOT "teaching the model new information."**

If you think fine-tuning works by "teaching the model facts from your data," every decision you make will be wrong. You'll overtrain, use wrong data formats, and wonder why the model gets worse.

Here's what fine-tuning ACTUALLY does:

**Fine-tuning ADJUSTS the model's BEHAVIOR — not its knowledge.**

The model already knows the facts (it was pre-trained on trillions of tokens). Fine-tuning doesn't teach it new facts — it teaches it WHEN and HOW to use what it already knows.

Think of it this way:
- **Pre-training**: The model reads the entire internet and learns language, facts, patterns
- **Fine-tuning**: The model practices your specific TASK so it gets better at that task

The pre-trained model KNOWS how to summarize legal documents. It read millions of legal documents during training. But it doesn't know that YOU want summaries in a specific format, at a specific length, with specific emphasis. Fine-tuning aligns its existing knowledge with your specific requirements.

This distinction matters because:
- If you think FT teaches new facts → you'll try to fine-tune on RARE information (it won't work well)
- If you think FT changes behavior → you'll focus on FORMAT, STYLE, and TASK structure (it works)
- If you think FT adds knowledge → you'll be surprised when the model can't recall facts from training data
- If you think FT shapes behavior → you'll understand why the model needs MANY examples of the SAME pattern

---

### 🤔 Discovery Question 1: The Loss Paradox

You fine-tune a model. You track training loss. This is what you see:

```
Epoch 1: loss=1.85
Epoch 2: loss=1.42
Epoch 3: loss=1.12
Epoch 4: loss=0.89
Epoch 5: loss=0.71
Epoch 6: loss=0.58
Epoch 7: loss=0.49
Epoch 8: loss=0.42  ← looks great!
```

Loss went down every epoch. The model learned. Right?

You evaluate the model on your test set after Epoch 8. It performs WORSE than the base model before training.

**🤔 Before reading on, answer these:**

1. How is it possible that loss goes down every epoch but the model gets WORSE? What is loss actually measuring? (Hint: loss measures prediction error on TRAINING data. What's different between training data and test data?)

2. At Epoch 3, the loss was 1.12. At Epoch 8, the loss was 0.42. But the Epoch 3 model performs BETTER on the test set. What happened between Epoch 3 and Epoch 8? The model kept learning — but WHAT did it learn?

3. **The hardest question:** You have two checkpoints: Epoch 3 (loss=1.12, better test performance) and Epoch 8 (loss=0.42, worse test performance). If you ONLY look at training loss, you'd pick Epoch 8. If you ONLY look at test performance, you'd pick Epoch 3. **In production, you don't have a test set — you have real users. How do you decide which checkpoint to deploy?** What metrics would you use beyond loss?

---

### 🤔 Discovery Question 2: The Double Descent

You're fine-tuning a small model (125M parameters) on a classification task.

You train with increasing amounts of data:

```
100 examples:   test accuracy = 72%
500 examples:   test accuracy = 78%
1,000 examples: test accuracy = 76%  ← went DOWN!
2,000 examples: test accuracy = 74%  ← went DOWN further!
5,000 examples: test accuracy = 81%  ← went UP again!
10,000 examples: test accuracy = 85%
```

**🤔 Before reading on, answer these:**

1. Accuracy went UP (72% → 78%), then DOWN (78% → 76% → 74%), then UP AGAIN (74% → 81% → 85%). This pattern is called "double descent." What do you think causes accuracy to DECREASE as you add MORE data? (Hint: Think about what happens when a small model tries to memorize too many patterns with a fixed parameter count.)

2. The model is small (125M parameters). With 1,000-2,000 examples, it's trying to memorize specific examples rather than learning general patterns. But with 5,000+ examples, it starts to generalize again. Why does MORE data eventually help, even for a small model? (Hint: More data means the "signal" patterns appear more consistently than "noise" patterns — the model can't memorize everything, so it's forced to generalize.)

3. **The hardest question:** When you hit the "valley" of double descent (74% accuracy at 2,000 examples), your instinct is to STOP adding data or GET A BIGGER MODEL. But if you push through to 5,000+ examples, accuracy RECOVERS. **How do you know if you're in the descent valley vs. your data is actually bad?** What experiment would distinguish "need more data" from "data quality is the problem"?

---

### 🤔 Discovery Question 3: The Test Set Contamination

Your fine-tuned model scores 94% on your held-out test set. You deploy to production.

In production, accuracy is 63%.

You investigate. You find that your test set was created by splitting your collected data: 90% training, 10% test. The split was RANDOM.

**🤔 Before reading on, answer these:**

1. Random splitting means similar examples ended up in BOTH training and test sets. If you have 10 examples of "password reset" questions, 9 go to training and 1 goes to test. The model "memorizes" the pattern from the 9 training examples and performs well on the nearly-identical test example. In production, users ask password reset questions with DIFFERENT phrasing — and the model fails. How should you have split the data differently?

2. What if the contamination is MORE subtle? Your training data and test data were collected from the same source at the same time. They share the SAME biases, same language patterns, same topics. Your test accuracy reflects "how well the model reproduces patterns from this specific data source" — NOT "how well the model generalizes to new data." How do you build a test set that truly measures generalization?

3. **The hardest question:** Your production data is FUNDAMENTALLY different from your training data. Users phrase questions differently in real support interactions vs. how your team wrote the training examples. The model learned the TEAM's language patterns, not the USERS' language patterns. **What's the solution?** You can't collect production data before you launch (you don't have users yet). You can't test what you haven't seen. How do you build a fine-tuning dataset that generalizes to UNSEEN production distributions?

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 30 min |
| What Fine-Tuning Actually Does | 25 min |
| Loss, Overfitting, Generalization | 30 min |
| Catastrophic Forgetting | 20 min |
| Code: Fine-Tuning from Scratch (BAD) | 35 min |
| Code: Fixing the Fine-Tuning (GOOD) | 40 min |
| Code: Training Monitor | 25 min |
| Data Leakage & Test Set Design | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 50 min |
| Gate Check | 15 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. What Fine-Tuning Actually Changes

**Fine-tuning adjusts WEIGHTS, not knowledge.**

Think of the model as having:
- **Pre-trained weights**: Learned from trillions of tokens. Contains general language understanding, facts, reasoning patterns.
- **Fine-tuned weights**: Adjusted from your training data. Doesn't ADD new knowledge — it REPRIORITIZES existing knowledge.

When you fine-tune:
- Weights that are USEFUL for your task get STRONGER
- Weights that are NOT useful for your task get WEAKER
- Some weights that were useful for OTHER tasks get overwritten → catastrophic forgetting

The key metric: **How many examples does it take to change behavior vs. teach facts?**

| Goal | Examples Needed | Works? |
|------|----------------|--------|
| Change response format (JSON → Markdown) | 100-500 | ✅ Excellent |
| Change writing style (formal → casual) | 200-1,000 | ✅ Good |
| Teach domain terminology ("dispense" vs "give") | 500-2,000 | ✅ Good |
| Teach new facts ("our CEO is Alice") | 5,000+ | ❌ Poor (better in system prompt) |
| Teach rare knowledge (niche scientific concepts) | 20,000+ | ❌ Very Poor (RAG is better) |

### 2. Loss Curves and What They Tell You

The training loss curve is your WINDOW into what's happening during training:

```
GOOD TRAINING CURVE:
Loss
│
│   ╲
│    ╲
│     ╲╲
│       ╲╲
│         ╲╲__  ← Loss decreases smoothly, plateaus
│           ╲╲_
│              ╲_____
└──────────────────────────→ Steps

UNDERFITTING (not enough training):
Loss
│
│   ╲
│    ╲
│     ╲
│      ╲
│       ╲
│        ╲  ← Still going down when training ended
│         ╲     → Need more epochs or higher LR
└──────────────────────────→ Steps

OVEFITTING (too much training):
Loss
│
│   ╲
│    ╲╲
│      ╲╲___
│        ╲  ╲
│         ╲   ╲
│          ╲    ╲  ← Training loss still going down
│           ╲     ╲    but test loss would go UP!
└──────────────────────────→ Steps
```

**The critical insight:** Training loss tells you how well the model fits TRAINING data. It does NOT tell you how well it generalizes. You need a SEPARATE evaluation set (validation set) to measure generalization.

### 3. Learning Rate and Its Effects

Learning rate is the MOST important hyperparameter. Small changes have DRAMATIC effects:

| Learning Rate | Effect | When to Use |
|--------------|--------|-------------|
| Too high (5e-5) | Loss diverges, model forgets everything | Never (dataset too small) |
| High (2e-5) | Fast convergence, risk of forgetting | Large datasets (>10K) |
| Moderate (1e-5) | Steady learning, good balance | Most cases |
| Low (5e-6) | Slow, safe, preserves base knowledge | Small datasets (<1K) |
| Too low (1e-6) | Barely learns, wastes compute | Never (just use prompting) |

**The rule of thumb:** For LoRA fine-tuning (File 02), use 2e-4 as a starting point (yes, 10x higher than full fine-tuning — LoRA is more stable).

---

## 💻 CODE EXAMPLES

### Example 1: The Naive Fine-Tuning Script (What NOT To Do — Full of Bugs)

```python
# BAD: Naive fine-tuning with every common mistake

import torch
from transformers import (
    AutoModelForCausalLM, 
    AutoTokenizer, 
    Trainer, 
    TrainingArguments
)
from datasets import Dataset

# PROBLEM 1: No evaluation set
# If you don't evaluate during training, you can't detect overfitting

# Load model and tokenizer
model_name = "microsoft/phi-2"  # Small model for demo
model = AutoModelForCausalLM.from_pretrained(model_name)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# PROBLEM 2: No padding token
# Many models don't have a padding token set by default
# This causes silent errors in batch processing
# tokenizer.pad_token = tokenizer.eos_token  ← MISSING!

# PROBLEM 3: No max_length — will blow up on long sequences

# Load data
data = [
    {"instruction": "What is Python?", "response": "Python is a programming language."},
    {"instruction": "What is ML?", "response": "Machine learning is a subset of AI."},
    # In reality, you'd have thousands of examples
]

def format_example(example):
    # PROBLEM 4: No consistent chat template
    # Different models expect different formats
    # Phi-2 expects a specific format, not this ad-hoc one
    return {
        "text": f"Question: {example['instruction']}\nAnswer: {example['response']}"
    }

formatted = [format_example(e) for e in data]

# PROBLEM 5: No train/test split
# Using ALL data for training means no way to detect overfitting
dataset = Dataset.from_list(formatted)

def tokenize_fn(examples):
    return tokenizer(
        examples["text"],
        truncation=True,
        padding=True,
        # PROBLEM 6: No max_length
        # Without max_length, sequences can be very long
        # This causes OOM or very slow training
        # max_length=512,  ← MISSING!
    )

tokenized = dataset.map(tokenize_fn, batched=True)

# PROBLEM 7: Training for too many epochs with small data
# With only 2 examples, 10 epochs = 20 total training steps
# The model will MEMORIZE these 2 examples perfectly
# But it won't generalize to ANY new input
training_args = TrainingArguments(
    output_dir="./phi2-finetuned",
    num_train_epochs=10,  # Way too many for 2 examples!
    per_device_train_batch_size=1,
    learning_rate=5e-5,  # High LR + small data = forgetting
    save_steps=100,       # Won't save for 100 steps (only have 20)
    logging_steps=1,
    # PROBLEM 8: No evaluation strategy
    # Without eval, you can't detect overfitting
    # evaluation_strategy="steps",  ← MISSING!
    # eval_steps=5,                 ← MISSING!
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized,
    # PROBLEM 9: No eval dataset
    # eval_dataset=eval_tokenized,  ← MISSING!
)

# PROBLEM 10: No gradient checkpointing
# Full model fine-tuning uses ALL memory
# Most GPUs will OOM for models > 1B parameters
trainer.train()

# PROBLEM 11: No evaluation after training
# The script ends without checking if the model actually learned anything useful
# Was the training successful? We DON'T KNOW.

print("Training complete!")
```

### 🔍 Find The Bugs

Before reading the fixed version, answer these:

1. **The evaluation bug**: This script trains for 10 epochs on 2 examples and NEVER evaluates. After training, the model can perfectly reproduce the 2 training examples. But it's WORSE at everything else. How many bugs contribute to this overfitting problem? List them.

2. **The data formatting bug**: Different base models expect different chat templates. Phi-2 uses a specific format. Mistral uses another. Llama uses another. If you use the WRONG format, the model learns "wrong patterns" — it produces outputs that don't match the expected structure. Why does wrong format make the model perform WORSE even though the content is the same?

3. **The silent padding bug**: The tokenizer doesn't have a `pad_token` set. PyTorch uses padding for batch processing. If padding tokens aren't set correctly, the model attends to PADDING as if it's real text. What does the model learn from attending to padding tokens?

4. **The no-evaluation bug**: This script runs for 10 epochs and saves the FINAL checkpoint. But the best checkpoint is USUALLY before the final epoch (early stopping). Without validation loss tracking, how do you know which checkpoint to use?

---

### Example 2: Fixed Fine-Tuning Script

```python
"""
Production Fine-Tuning Script — Fixed

What this DOES right:
1. Train/eval split to detect overfitting
2. Proper tokenization with max_length
3. Evaluation during training
4. Early stopping
5. Learning rate appropriate for dataset size
6. Padding token set correctly
7. Consistent chat template for the model
8. Save best checkpoint, not last checkpoint
9. Evaluation after training
10. Memory-efficient training (LoRA — see File 02)
"""

import torch
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    Trainer,
    TrainingArguments,
    EarlyStoppingCallback,
)
from datasets import Dataset, DatasetDict
import numpy as np


def prepare_dataset(data: list[dict], tokenizer, 
                    max_length: int = 512,
                    test_split: float = 0.1) -> DatasetDict:
    """
    Prepare dataset with proper train/test split.
    
    Key decisions:
    - test_split=0.1: 10% held out for evaluation
    - max_length=512: Prevents OOM, consistent sequence length
    """
    import random
    random.shuffle(data)
    
    split_idx = int(len(data) * (1 - test_split))
    train_data = data[:split_idx]
    eval_data = data[split_idx:]
    
    def format_chat(example):
        """Format using the model's expected chat template."""
        # For Phi-2 / Mistral / Llama format:
        return {
            "text": f"<|user|>\n{example['instruction']}\n<|assistant|>\n{example['response']}"
        }
    
    train_formatted = [format_chat(e) for e in train_data]
    eval_formatted = [format_chat(e) for e in eval_data]
    
    train_dataset = Dataset.from_list(train_formatted)
    eval_dataset = Dataset.from_list(eval_formatted)
    
    def tokenize_fn(examples):
        """Tokenize with proper padding and truncation."""
        result = tokenizer(
            examples["text"],
            truncation=True,
            padding="max_length",
            max_length=max_length,
        )
        # For causal LM training, labels = input_ids
        result["labels"] = result["input_ids"].copy()
        return result
    
    tokenized_train = train_dataset.map(tokenize_fn, batched=True)
    tokenized_eval = eval_dataset.map(tokenize_fn, batched=True)
    
    return DatasetDict({
        "train": tokenized_train,
        "eval": tokenized_eval,
    })


def compute_metrics(eval_pred):
    """
    Compute evaluation metrics during training.
    
    This helps you detect overfitting EARLY.
    Rising eval loss → stop training.
    """
    predictions, labels = eval_pred
    # Simple metric: perplexity (lower is better)
    loss = torch.nn.CrossEntropyLoss()(
        torch.tensor(predictions).view(-1, predictions.shape[-1]),
        torch.tensor(labels).view(-1),
    )
    return {
        "eval_loss": loss.item(),
        "perplexity": np.exp(loss.item()),
    }


def train_model(
    model_name: str = "microsoft/phi-2",
    train_data: list[dict] = None,
    output_dir: str = "./fine-tuned-model",
    num_epochs: int = 5,
    learning_rate: float = 2e-5,
    batch_size: int = 4,
    max_length: int = 512,
):
    """
    Fine-tune a model with proper monitoring.
    
    Key parameters explained:
    - learning_rate=2e-5: Good starting point for full fine-tuning
    - num_epochs=5: Enough for most tasks, early stopping prevents overfitting
    - batch_size=4: Depends on GPU memory, adjust as needed
    - max_length=512: Balances context length vs. memory usage
    """
    # Load tokenizer and set padding token
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token
    
    # Prepare dataset
    dataset = prepare_dataset(train_data, tokenizer, max_length)
    
    # Load model
    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        torch_dtype=torch.float16,  # Half precision for memory efficiency
        device_map="auto",           # Auto-distribute across GPUs
    )
    
    # Training arguments with evaluation
    training_args = TrainingArguments(
        output_dir=output_dir,
        num_train_epochs=num_epochs,
        per_device_train_batch_size=batch_size,
        per_device_eval_batch_size=batch_size,
        learning_rate=learning_rate,
        warmup_steps=10,
        logging_steps=5,
        eval_strategy="steps",
        eval_steps=max(1, len(dataset["train"]) // batch_size // 5),
        # Evaluate roughly 5 times per epoch
        save_strategy="steps",
        save_steps=max(1, len(dataset["train"]) // batch_size // 2),
        load_best_model_at_end=True,  # Save BEST checkpoint, not last
        metric_for_best_model="eval_loss",
        greater_is_better=False,  # Lower loss is better
        report_to="none",  # Or "wandb" for experiment tracking
        fp16=True,  # Mixed precision training (2x speed)
        gradient_checkpointing=True,  # Memory efficient
    )
    
    # Trainer with evaluation
    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=dataset["train"],
        eval_dataset=dataset["eval"],
        compute_metrics=compute_metrics,
        callbacks=[EarlyStoppingCallback(
            early_stopping_patience=3,  # Stop if eval loss doesn't improve for 3 checks
        )],
    )
    
    # Train
    print(f"Starting training: {len(dataset['train'])} train, "
          f"{len(dataset['eval'])} eval examples")
    print(f"Parameters: epochs={num_epochs}, lr={learning_rate}, "
          f"batch={batch_size}, max_len={max_length}")
    
    trainer.train()
    
    # Save the best model (not the last one!)
    trainer.save_model()
    tokenizer.save_pretrained(output_dir)
    
    # Final evaluation
    eval_result = trainer.evaluate()
    print(f"\nFinal evaluation:")
    print(f"  Eval loss: {eval_result['eval_loss']:.4f}")
    print(f"  Perplexity: {eval_result['perplexity']:.2f}")
    
    return trainer


# Usage — but WAIT. Before you run this, ask yourself:
# Is fine-tuning the RIGHT approach for this task?
# (See File 00 — when to fine-tune vs. prompt engineer vs. RAG)
#
# If you're still reading: this code is structurally correct,
# but it does FULL fine-tuning. For production, you'd use
# LoRA (File 02) instead of updating ALL model weights.
# Full fine-tuning of a 3B model costs ~$100/run on a single GPU.
# LoRA fine-tuning costs ~$5/run.
```

### 🔍 What's Still Not Great About This "Fixed" Version?

Even the "fixed" version has issues. Can you spot them?

1. **Full fine-tuning is expensive**: This updates ALL model weights. For a 3B model, that's 3B parameters × 4 bytes = 12GB of optimizer states alone. You need an A100 for anything > 3B. What's the ALTERNATIVE that achieves similar results for 1% of the cost? (Hint: File 02)

2. **Single metric optimization**: It optimizes for eval_loss, but lower eval loss doesn't always mean better task performance. If the model memorizes the eval set patterns better but fails on novel patterns, eval_loss goes down but actual performance goes down. What ADDITIONAL metrics would you track?

3. **No baseline comparison**: After training, it evaluates the fine-tuned model but doesn't compare to the BASE model. If the base model scores eval_loss=2.5 and the fine-tuned model scores eval_loss=1.8, that's improvement. But what if the base model was already at eval_loss=1.9 on this data? Was fine-tuning worth it?

---

### Example 3: Training Monitor — Watch Your Training in Real Time

```python
"""
Training Monitor — Detect Problems EARLY

This is NOT a training script. This is a MONITORING script
that helps you INTERPRET what's happening during training.

Run this analysis on your training logs to detect:
1. Overfitting (eval loss starts going up)
2. Learning rate issues (loss diverges or plateaus too high)
3. Data issues (loss drops too fast = data too simple / memorizing)
4. When to stop (early stopping point)
"""

def analyze_training_logs(log_file: str) -> dict:
    """
    Analyze training logs and provide recommendations.
    
    Read this BEFORE your next training run to understand
    what went wrong with the previous run.
    """
    import json
    
    with open(log_file) as f:
        logs = [json.loads(line) for line in f if line.strip()]
    
    # Extract loss values
    train_steps = []
    train_losses = []
    eval_steps = []
    eval_losses = []
    
    for log in logs:
        if "loss" in log and "eval_loss" not in log:
            train_steps.append(log.get("step", 0))
            train_losses.append(log["loss"])
        if "eval_loss" in log:
            eval_steps.append(log.get("step", 0))
            eval_losses.append(log["eval_loss"])
    
    analysis = {
        "status": "unknown",
        "warnings": [],
        "recommendations": [],
    }
    
    # Check 1: Did training converge?
    if train_losses:
        final_loss = train_losses[-1]
        initial_loss = train_losses[0]
        improvement = (initial_loss - final_loss) / initial_loss * 100
        
        if improvement < 5:
            analysis["warnings"].append(
                f"Loss only improved {improvement:.1f}% — model barely learned. "
                f"Increase learning rate or check data quality."
            )
        elif improvement > 90:
            analysis["warnings"].append(
                f"Loss improved {improvement:.1f}% — SUSPICIOUSLY high. "
                f"The model may be MEMORIZING the training data. "
                f"Check for data leakage or too many epochs."
            )
    
    # Check 2: Is overfitting happening?
    if len(eval_losses) >= 3:
        recent_eval = eval_losses[-3:]  # Last 3 eval checks
        if recent_eval[-1] > recent_eval[0]:
            # Eval loss is RISING
            analysis["warnings"].append(
                f"EVAL LOSS RISING: {recent_eval[0]:.4f} → {recent_eval[-1]:.4f}. "
                f"OVERFITTING detected! Train fewer epochs or increase dropout."
            )
            # Find best checkpoint
            best_idx = eval_losses.index(min(eval_losses))
            analysis["recommendations"].append(
                f"Best checkpoint at step {eval_steps[best_idx]} "
                f"(eval_loss={eval_losses[best_idx]:.4f}). "
                f"Use this checkpoint, NOT the final one."
            )
    
    # Check 3: What's the gap between train and eval loss?
    if train_losses and eval_losses:
        # Compare last train loss to last eval loss
        last_train = train_losses[-1]
        last_eval = eval_losses[-1]
        gap = last_eval - last_train
        
        if gap > 0.5:
            analysis["warnings"].append(
                f"Large train/eval gap (train={last_train:.4f}, eval={last_eval:.4f}, "
                f"gap={gap:.4f}). The model is MEMORIZING not GENERALIZING. "
                f"Add regularization or reduce model capacity."
            )
    
    # Determine overall status
    if any("OVERFITTING" in w for w in analysis["warnings"]):
        analysis["status"] = "OVERFITTING"
    elif any("SUSPICIOUSLY" in w for w in analysis["warnings"]):
        analysis["status"] = "SUSPICIOUS"
    elif not analysis["warnings"]:
        analysis["status"] = "HEALTHY"
    else:
        analysis["status"] = "WARNINGS"
    
    return analysis


# Example usage — but the real value is:
# What do YOU see in YOUR training logs?

# Before reading the analysis output below, look at these two
# training runs and tell me which one is better:

run_a = {
    "train_loss": [2.1, 1.8, 1.5, 1.2, 0.9, 0.6, 0.3, 0.1],
    "eval_loss": [2.1, 1.9, 1.8, 1.7, 1.9, 2.1, 2.3, 2.5],
}

run_b = {
    "train_loss": [2.1, 1.9, 1.7, 1.6, 1.5, 1.5, 1.4, 1.4],
    "eval_loss": [2.1, 1.9, 1.8, 1.8, 1.7, 1.7, 1.7, 1.7],
}
```

### 🔍 Interpret These Training Runs

**🤔 Before reading the answer:**

1. Run A: Training loss goes from 2.1 → 0.1 (amazing!). But eval loss goes from 2.1 → 2.5 (getting WORSE!). Run B: Training loss goes from 2.1 → 1.4 (modest). Eval loss goes from 2.1 → 1.7 (improving!). **Which model would you deploy? Why?**

2. Run A's best checkpoint is at step 3 (eval_loss=1.7). Run B's best checkpoint is at step 7 (eval_loss=1.7). They have the SAME best eval loss. But Run A reaches it at step 3 while Run B needs step 7. **Is Run A actually "better" because it reaches the same performance faster? What's the catch?**

3. **The hardest question:** Run A's best eval loss is 1.7 (step 3). Run B's best eval loss is also 1.7 (step 7). But after deployment:
   - Run A's model performs at 72% accuracy in production
   - Run B's model performs at 84% accuracy in production
   
   Both have the SAME eval loss. Why is production performance DIFFERENT? What does eval loss measure that's DIFFERENT from production accuracy?

---

## ✅ Good Output Examples

### What a Healthy Fine-Tuning Run Looks Like

```
TRAINING LOG — HEALTHY RUN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Configuration:
  Model:         Llama-3.2-1B
  Dataset:       5,000 examples (train), 500 (eval)
  Epochs:        5 (early stopped at epoch 4)
  Learning rate: 2e-5
  Batch size:    8
  Max length:    1024

Loss Curves:
Epoch  Train Loss  Eval Loss  Perplexity  Notes
  1      1.85        1.82       6.17      Learning
  2      1.42        1.45       4.26      Both improving
  3      1.12        1.18       3.25      Gap stable (~0.06)
  4      0.89        1.02       2.77      Gap widening slightly ⚠️
 [5]     0.71        1.05       2.86      Eval loss went up! → STOP

Best checkpoint: Epoch 3 (eval_loss=1.18, perplexity=3.25)

Analysis:
  ✅ Train loss decreasing healthily (not too fast, not too slow)
  ✅ Eval loss decreasing initially (model generalizing)
  ⚠️ Eval loss minimum at epoch 3, then rising (overfitting starts)
  ✅ Stopped at epoch 4 (early stopping caught the overfitting)
  ✅ Train/eval gap reasonable (~0.06 at best checkpoint)

Performance:
  Before FT: accuracy=72%, eval_loss=2.31
  After FT:  accuracy=89%, eval_loss=1.18
  Improvement: +17% accuracy, -48% eval_loss ✅
```

### What a Failed Run Looks Like

```
TRAINING LOG — OVERFITTING RUN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Configuration:
  Model:         Llama-3.2-1B
  Dataset:       500 examples (train), 50 (eval)  ← TOO SMALL!
  Epochs:        20  ← TOO MANY!
  Learning rate: 5e-5  ← TOO HIGH!

Loss Curves:
Epoch  Train Loss  Eval Loss  Perplexity  Notes
  1      1.21        1.52       4.57      Already diverging!
  2      0.45        1.89       6.62      Train loss drops, eval rises
  3      0.12        2.34      10.38      Memorization!
  4      0.03        2.89      18.01      Model is WORSE than before
 ...
 20      0.001       5.23     187.23     Complete failure

Issues Found:
  ❌ Train loss near 0 (model memorized all 500 examples)
  ❌ Eval loss INCREASING (model got worse at generalization)
  ❌ Gap between train/eval: 5.23 (massive overfitting)
  ❌ Too many epochs (20 for 500 examples = catastrophic)
  ❌ Learning rate too high (divergence from epoch 1)

Wasted: ~$8 of compute, 45 minutes of time
Lesson: Train/eval split + early stopping would have caught this at epoch 2
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: No Evaluation During Training

```python
# BAD: Blind training
trainer = Trainer(
    model=model,
    args=training_args,  # No evaluation strategy
    train_dataset=train_data,
    # No eval_dataset!
)
trainer.train()
# After 5 hours: no idea if model is good or garbage

# GOOD: Evaluate during training
trainer = Trainer(
    model=model,
    args=TrainingArguments(
        eval_strategy="steps",
        eval_steps=50,
        load_best_model_at_end=True,  # KEY: use best checkpoint
    ),
    train_dataset=train_data,
    eval_dataset=eval_data,  # KEY: separate eval set
)
trainer.train()
# After 5 hours: best checkpoint saved, overfitting detected early
```

### Antipattern 2: Training on the Wrong Thing

```python
# BAD: Fine-tuning on Q&A pairs
# The model learns to reproduce answers, not to answer questions
data = [
    {"input": "What is Python?", "output": "Python is a language..."},
    # Model memorizes the answers, not the Q&A format
]

# GOOD: Fine-tuning on conversations with proper format
# The model learns the DIALOGUE pattern
data = [
    {"messages": [
        {"role": "user", "content": "What is Python?"},
        {"role": "assistant", "content": "Python is a language..."},
    ]},
    # Model learns the USER→ASSISTANT turn-taking pattern
]
```

### Antipattern 3: Assuming More Data = Better Model

```python
# Reality check from Discovery Question 2:
# 1,000 examples of bad data < 100 examples of perfect data
# 
# Common data quality issues:
# - Inconsistent labels (different reviewers gave different answers)
# - Wrong format (data doesn't match model's expected format)
# - Noisy data (includes irrelevant or incorrect examples)
# - Biased data (over-represents certain patterns)
#
# Before adding MORE data:
# 1. Evaluate current data quality
# 2. Check label consistency
# 3. Balance your dataset
# 4. THEN add more (high-quality) data
```

### Common Failure Modes:

| Failure Mode | Symptom | Root Cause | Fix |
|---|---|---|---|
| **Loss decreases, model worsens** | Training loss down, eval loss up | Overfitting | Fewer epochs, more data, LoRA |
| **Model forgets everything** | Base model was better at everything | Learning rate too high, too many epochs | Lower LR, fewer epochs, EWC |
| **Model can't generate** | Produces gibberish or repeats | Wrong chat template, bad tokenization | Check template, check padding token |
| **No improvement over base** | FT model = base model performance | Too little data, too low LR, wrong format | More data, higher LR, verify format |
| **Great on eval, bad in prod** | 94% test, 63% production | Data leakage, distribution shift | Better data splitting, domain adaptation |
| **OOM during training** | GPU runs out of memory | Sequence too long, batch too large | Gradient checkpointing, smaller batch, LoRA |

---

## 🧪 Drills & Challenges

### Drill 1: Diagnose Two Training Runs (30 min)

Below are the training logs from TWO fine-tuning runs. Both use the SAME model and data. One is healthy, one has problems.

**Your task:** Identify which run is which, diagnose the problems, and recommend fixes.

```python
# Run A — Logs
logs_a = [
    {"step": 0, "loss": 2.45},
    {"step": 10, "loss": 2.01},
    {"step": 20, "loss": 1.82, "eval_loss": 1.85},
    {"step": 30, "loss": 1.65},
    {"step": 40, "loss": 1.51, "eval_loss": 1.58},
    {"step": 50, "loss": 1.38},
    {"step": 60, "loss": 1.27, "eval_loss": 1.35},
    {"step": 70, "loss": 1.18},
    {"step": 80, "loss": 1.09, "eval_loss": 1.20},
]

# Run B — Logs
logs_b = [
    {"step": 0, "loss": 2.45},
    {"step": 10, "loss": 2.03},
    {"step": 20, "loss": 1.12, "eval_loss": 1.80},
    {"step": 30, "loss": 0.54},
    {"step": 40, "loss": 0.21, "eval_loss": 2.34},
    {"step": 50, "loss": 0.09},
    {"step": 60, "loss": 0.04, "eval_loss": 3.12},
    {"step": 70, "loss": 0.02},
    {"step": 80, "loss": 0.01, "eval_loss": 4.21},
]

# 🤔 Questions:
# 1. Which run is healthy? Which is overfitting? How can you tell?
# 2. In Run B, at what step should training have stopped?
# 3. What's the ROOT CAUSE of Run B's problem?
# 4. What specific changes would you make to Run B's configuration?
```

**Expected analysis:**
- Run A: Healthy (train loss and eval loss both decreasing, gap stable ~0.10-0.12)
- Run B: Overfitting (train loss near 0, eval loss INCREASING, gap widening from 0.68 to 4.20)
- Best stopping point for B: Step 20 (eval_loss=1.80, lowest)
- Root cause: Too many epochs, possibly too high LR for the dataset size

---

### Drill 2: Build an Early Stopping Detector (30 min)

Write a function that monitors training logs and recommends when to stop:

```python
def recommend_stop(logs: list[dict], patience: int = 3) -> dict:
    """
    Analyze training logs and recommend when to stop training.
    
    Args:
        logs: List of training log entries (each with "step", "loss",
              and optionally "eval_loss")
        patience: Number of eval checks without improvement before stopping
    
    Returns:
        {
            "should_stop": bool,
            "best_step": int,
            "best_eval_loss": float,
            "reason": str,
        }
    """
    # Your implementation here
    pass


# Test with these logs:
test_logs = [
    {"step": 0, "loss": 2.5, "eval_loss": 2.5},
    {"step": 10, "loss": 2.1, "eval_loss": 2.2},
    {"step": 20, "loss": 1.8, "eval_loss": 1.9},
    {"step": 30, "loss": 1.5, "eval_loss": 1.7},
    {"step": 40, "loss": 1.3, "eval_loss": 1.6},
    {"step": 50, "loss": 1.1, "eval_loss": 1.7},  # Eval went UP!
    {"step": 60, "loss": 0.9, "eval_loss": 1.8},  # Eval still going up
    {"step": 70, "loss": 0.7, "eval_loss": 1.9},  # Clear overfitting
]

result = recommend_stop(test_logs, patience=2)
print(f"Should stop: {result['should_stop']}")
print(f"Best step: {result['best_step']} (eval_loss={result['best_eval_loss']})")
```

**🤔 Design questions:**
1. Should early stopping look for ANY increase in eval loss, or only statistically significant increases? What counts as "significant"?
2. If eval loss FLUCTUATES (1.7 → 1.6 → 1.7 → 1.6), is that overfitting or noise? How does your algorithm distinguish?
3. Should you ever restart training from the best checkpoint instead of stopping completely?

---

### Drill 3: Design a Test Set Strategy (45 min)

You're fine-tuning a model for a **legal document classification** system. Your dataset:
- 50,000 legal documents labeled with 15 categories (contract, will, subpoena, etc.)
- Each document was labeled by a paralegal
- Documents span 20 years of legal history
- 40% are contracts, 5% are subpoenas (imbalanced)

**Your task:** Design the data splitting strategy:

```python
def split_legal_dataset(documents: list[dict], 
                         metadata: dict) -> dict:
    """
    Split a legal document dataset for fine-tuning.
    
    MUST avoid:
    1. Data leakage (same document type in train AND test)
    2. Temporal leakage (future documents leaking to past)
    3. Lawyer leakage (same paralegal labeling train AND test)
    4. Imbalanced splits (all subpoenas in test, none in train)
    
    Returns:
        {"train": [...], "eval": [...], "test": [...]}
    """
    # Your implementation here
    pass
```

**🤔 Questions to answer:**
1. If the SAME paralegal labeled both training and test documents, and that paralegal has a consistent style, the model will learn that style — not general legal patterns. How do you split by LABELER to measure true generalization?

2. Legal LANGUAGE changes over time. A document from 2000 uses different phrasing than one from 2020. If your training data is from 2000-2015 and your test data is from 2015-2020, the test measures "temporal generalization." But if you RANDOMLY split, both train and test have mixed time periods — the test doesn't measure temporal generalization at all. How do you design a temporal split?

3. Subpoenas are 5% of the data (2,500 documents). If you split 80/10/10, subpoenas split as 2,000/250/250. But 250 subpoenas might not be enough for a reliable test set. How do you handle IMBALANCED classes in your splitting strategy?

---

### Drill 4: The "Catastrophic Forgetting" Experiment (60 min)

Design an experiment to measure catastrophic forgetting:

```python
"""
Experiment: Measure what the model FORGOT after fine-tuning.

Setup:
1. Take a base model (e.g., Llama-3.2-1B)
2. Evaluate it on 3 benchmarks:
   - MMLU (general knowledge)
   - HumanEval (code generation)
   - GSM8K (math reasoning)
3. Fine-tune on legal summarization task
4. Re-evaluate on all 3 benchmarks
5. Measure what was lost

Your task: Write the evaluation harness
"""
import random

# Simulated benchmark scores for the BASE model
base_scores = {
    "mmlu": 0.72,       # General knowledge
    "humaneval": 0.68,  # Code generation
    "gsm8k": 0.65,      # Math reasoning
    "legal_summary": 0.45,  # Before FT — not good
}

# Simulated benchmark scores for the FINE-TUNED model
ft_scores = {
    "mmlu": 0.67,       # -5% 😰
    "humaneval": 0.61,  # -7% 😱
    "gsm8k": 0.63,      # -2% 😕
    "legal_summary": 0.87,  # +42% 🎉
}


def measure_forgetting(base: dict, fine_tuned: dict) -> dict:
    """
    Measure what was forgotten during fine-tuning.
    
    Returns per-task change and calculates "forgetting score"
    """
    # Your implementation here
    pass


def should_deploy(forgetting_report: dict, threshold: float = 0.10) -> tuple[bool, str]:
    """
    Decision: Should this fine-tuned model be deployed?
    
    Rule: If ANY benchmark regressed more than `threshold` (10%),
    the model should NOT be deployed without mitigation.
    """
    # Your implementation here
    pass
```

**🤔 Questions to answer:**
1. The model lost 7% on code generation (HumanEval). But if your application doesn't generate code, does this matter? Should "forgetting" only be measured on RELEVANT capabilities?
2. What if the forgetting is uneven across SUBGROUPS? The model got worse at math for non-English speakers but stayed the same for English speakers. How would you detect this?
3. What MITIGATIONS could reduce forgetting? (Hint: EWC, multi-task learning, replay buffers, LoRA)

---

## 🚦 Gate Check

Before moving to File 02 (LoRA & PEFT), verify you can:

1. **Read a training loss curve** — Given a sequence of training and eval losses, identify: healthy training, overfitting, underfitting, learning rate too high, data quality issues

2. **Design a train/eval/test split** — Given a dataset with temporal structure, labeler information, and class imbalance, design a splitting strategy that measures TRUE generalization (not just in-distribution performance)

3. **Identify catastrophic forgetting** — Given before/after benchmark scores, quantify what was forgotten, decide if the fine-tuning was worth it, and propose mitigations

4. **Explain the double descent phenomenon** — Given a plot of test accuracy vs. training data size, identify the double descent regions and explain WHY accuracy decreases before increasing again

5. **Choose appropriate hyperparameters** — Given:
   - Dataset size: 500 examples
   - Model size: 7B parameters
   - Task: Change output format (JSON → Markdown)
   
   Recommend: learning rate, number of epochs, batch size, and explain WHY each choice

6. **Build an early stopping detector** — Implement a function that reads training logs and recommends when to stop, with configurable patience and minimum improvement thresholds

7. **Calculate training cost** — Given:
   - Model: 7B parameters
   - GPU: A10G ($1.50/hr)
   - Dataset: 10,000 examples
   - Max length: 1,024 tokens
   - Batch size: 4
   - Epochs: 3
   
   Estimate: total training time, total GPU cost, and whether LoRA (File 02) would reduce this

---

## 📚 Resources

### Foundational Papers
- **[Fine-Tuning as Forcing](https://arxiv.org/abs/2305.17328)** — Understanding what fine-tuning actually does to model representations
- **[Catastrophic Forgetting in LLMs](https://arxiv.org/abs/2308.16171)** — Why fine-tuning causes forgetting and how to measure it
- **[Double Descent in Neural Networks](https://arxiv.org/abs/1912.02292)** — The phenomenon where more data temporarily makes models worse
- **[Data Leakage in ML](https://arxiv.org/abs/2107.03374)** — How data leakage inflates evaluation metrics

### Practical Guides
- **[HuggingFace Fine-Tuning Tutorial](https://huggingface.co/docs/transformers/training)** — Official HF training guide
- **[Fine-Tuning Best Practices](https://docs.ray.io/en/latest/ray-air/examples/gptj_finetuning.html)** — Anyscale's production fine-tuning guide

### Phase Cross-References
- **Phase 4 (RAG)** → RAG vs. Fine-tuning: RAG adds knowledge, FT changes behavior. Different tools.
- **Phase 7 (Evals)** → Your Phase 7 eval platform evaluates fine-tuned models. Use it BEFORE and AFTER fine-tuning to measure improvement AND regression.
- **Phase 8 (Guardrails)** → Fine-tuning can introduce safety regressions. Your guardrails MUST catch them. Always re-run guardrail evaluation after fine-tuning.

> **Next up: File 02 — LoRA & Parameter-Efficient Fine-Tuning.** Full fine-tuning is expensive. LoRA achieves similar results for 1% of the cost. But LoRA has its own traps — rank selection, adapter placement, learning rate scaling. You'll discover why by building it wrong first.
