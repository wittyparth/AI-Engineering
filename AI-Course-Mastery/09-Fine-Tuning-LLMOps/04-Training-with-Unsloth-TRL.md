# 04 — Training with Unsloth & TRL: The Training Loop

## 🎯 Purpose & Goals

> 🛑 STOP. Before we write training scripts, you need to understand the biggest failure mode in fine-tuning:

**Training is where your data, your model, and your hyperparameters COLLIDE.**

And when they collide badly, you get:
- Loss diverging to infinity (LR too high)
- Training that's 10x slower than it should be (wrong config)
- OOM at epoch 3 (memory leak from gradient accumulation)
- A checkpoint that's WORSE than the base model (too many epochs)
- A model that can't generate anything (wrong chat template in training)

Every tutorial shows a clean training script that "just works." In reality, your FIRST training run will fail. Your SECOND might work but produce bad results. Your THIRD might look good but have a subtle bug that wastes hours.

The goal of this module is NOT to show you the perfect training script. It's to teach you how to DEBUG training failures so you can get to the perfect script faster.

---

### 🤔 Discovery Question 1: The Loss Spike Mystery

Your training is going well:

```
Step  100: loss=1.85
Step  200: loss=1.62
Step  300: loss=1.48
Step  400: loss=1.35
Step  500: loss=8.24  ← WHAT?!
Step  600: loss=1.28
Step  700: loss=1.15
Step  800: loss=1.08
```

At step 500, loss spiked from 1.35 to 8.24. Then at step 600, it's back to normal (1.28).

**🤔 Before reading on, answer these:**

1. What could cause a SINGLE step to have 6x higher loss while neighboring steps are normal? (Hint: Think about the data — what if step 500 processed a BATCH of exceptionally hard examples? What if step 500 had a corrupted example?)

2. The loss RECOVERED after the spike. If it was a learning rate issue (LR too high), the loss would STAY high or diverge. If it was a data corruption issue, it would happen at the SAME step every epoch. What does the "spike + recovery" pattern tell you about the root cause?

3. **The hardest question:** You have two options:
   - **Ignore the spike**: It's just one bad batch, the model recovered
   - **Investigate the spike**: It might indicate data issues, gradient instability, or numerical problems
   
   If you investigate every spike, you spend hours debugging harmless noise. If you ignore spikes, you might miss a real problem that slowly degrades your model. **How do you distinguish "benign spike" from "dangerous anomaly"?** What would you look at to decide?

---

### 🤔 Discovery Question 2: The Resumption Paradox

You're training a model. At step 1000, you save a checkpoint (loss=0.85). Your training CRASHES at step 1500 (power outage, OOM, etc.).

You resume from the step-1000 checkpoint. You continue training.

```
Step 1000 (original): loss=0.85  ← Checkpoint saved here
Step 1000 (resumed):  loss=0.92  ← HIGHER than original!
Step 1100 (resumed):  loss=0.88
Step 1200 (resumed):  loss=0.82  ← Finally below original
```

**🤔 Before reading on, answer these:**

1. You resumed from the EXACT same checkpoint. But the loss at step 1000 is HIGHER (0.92 vs 0.85). How is this possible if you loaded the same weights? (Hint: What state is NOT preserved in a checkpoint besides weights? Think about optimizer state, learning rate scheduler, batch order.)

2. If you only save model WEIGHTS (not optimizer state), the optimizer "forgets" its momentum and adaptive learning rates. AdamW adapts per-parameter learning rates based on gradient history. Loading only weights means the optimizer restarts with ZERO history — effectively losing the learning rate adaptation. What happens to loss when the optimizer restarts?

3. **The hardest question:** You want to save checkpoints that can be RESUMED perfectly, without any loss spike. What state do you need to save besides model weights? (Optimizer state, scheduler state, random seed, data ordinal — anything else?) **What's the minimum set of state needed for PERFECT resumption?**

---

### 🤔 Discovery Question 3: The Silent OOM

You train a 7B model with LoRA. Epoch 1 works fine. Epoch 2 crashes with OOM halfway through.

Your configuration:
- Batch size: 4
- Gradient accumulation: 8
- Max length: 2048
- LoRA rank: 16
- Base model: 4-bit quantized

**🤔 Before reading on, answer these:**

1. If epoch 1 worked fine, why does epoch 2 OOM? The data is the same size, the model is the same size. What CHANGED between epoch 1 and epoch 2? (Hint: Think about the cache. What caches grow during training? Gradients? Activation memory? Optimizer states?)

2. Gradient accumulation: batch_size=4, grad_accum=8 means "effective batch size" = 32. But the memory for activations is based on batch_size=4 (the micro-batch). At epoch 2, if the cache isn't cleared properly, activation memory from epoch 1 accumulates. But that shouldn't happen with proper framework code. What ELSE could grow?

3. **The hardest question:** "Out of memory" errors are the MOST common training failure. But they're also the MOST preventable. **What's your systematic approach to estimating memory BEFORE training starts?** How do you know if a configuration (model size, batch size, max length, LoRA rank) will fit in your GPU before you waste 2 hours finding out at epoch 2?

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 30 min |
| Training Loop Architecture | 25 min |
| SFTTrainer vs Trainer | 15 min |
| Hyperparameters Deep-Dive | 30 min |
| Gradient Accumulation & Memory | 20 min |
| Code: Training Script (BAD) | 30 min |
| Code: Fixed Training Script | 40 min |
| Code: Training Monitor | 25 min |
| Code: Checkpoint & Resume | 20 min |
| Distributed Training Basics | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 50 min |
| Gate Check | 15 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Training Loop — What Actually Happens

```
For each epoch:
    For each batch:
        1. Forward pass: input → model → output
        2. Compute loss: output vs target
        3. Backward pass: loss → gradients
        4. Optimizer step: gradients → weight update
        5. Log metrics: loss, learning rate, memory
        6. [Optional] Evaluation: run eval set
        7. [Optional] Save checkpoint
```

Each step has MEMORY implications:
- **Forward pass**: Stores activations (all intermediate values) for backward pass
- **Backward pass**: Computes gradients (same size as model weights)
- **Optimizer step**: Stores optimizer state (momentum, variance — 2x model size for AdamW)

```
MEMORY BREAKDOWN (7B model, LoRA r=16):

Component           │ Full FT     │ LoRA        │ QLoRA (4-bit)
────────────────────┼─────────────┼─────────────┼──────────────
Model weights       │ 28 GB       │ 28 GB       │ 3.5 GB
LoRA adapters       │ 0           │ 66 MB       │ 66 MB
Gradients           │ 28 GB       │ 66 MB       │ 66 MB
Optimizer states    │ 56 GB       │ 132 MB      │ 132 MB
Activations (bs=4)  │ ~8 GB       │ ~8 GB       │ ~8 GB
────────────────────┼─────────────┼─────────────┼──────────────
Total               │ ~120 GB     │ ~36 GB      │ ~12 GB
                    │ (2-4× A100) │ (1× A100)   │ (1× RTX 4090)
```

This is WHY LoRA exists: full FT needs 120GB (multiple GPUs), QLoRA needs 12GB (one consumer GPU).

### 2. Gradient Accumulation — What It Actually Does

Gradient accumulation lets you simulate a LARGER batch size than your GPU can fit:

```
Without accumulation (batch_size=4):
  Step 1: Process 4 examples → Update weights
  → 4 examples per update

With accumulation (batch_size=4, grad_accum=8):
  Step 1: Process 4 examples → Accumulate gradients (don't update yet)
  Step 2: Process 4 examples → Accumulate gradients
  ...
  Step 8: Process 4 examples → UPDATE weights (using 32 examples total)
  → 32 "effective" examples per update, 4 "micro-batch" examples at a time
```

**The critical insight:** Gradient accumulation does NOT reduce memory per example. It still processes batch_size=4 at a time. It just delays the weight UPDATE until N batches have been processed.

### 3. Learning Rate Schedulers

The learning rate doesn't stay constant during training:

```
Constant LR:
  lr = 2e-4 throughout
  → Good for short training, risk of not converging

Cosine with warmup:
  Step 0-20: Warmup (lr: 0 → 2e-4)
  Step 20-500: Cosine decay (lr: 2e-4 → 2e-5)
  → Most common, good balance

Linear with warmup:
  Step 0-20: Warmup (lr: 0 → 2e-4)
  Step 20-500: Linear decay (lr: 2e-4 → 0)
  → Simple, works well for fine-tuning
```

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Training Script

```python
# BAD: Training script with 8+ bugs

from transformers import TrainingArguments, AutoModelForCausalLM, AutoTokenizer
from datasets import Dataset
from peft import LoraConfig, get_peft_model
import torch

# Bug 1: No CUDA check
# If CUDA is not available, this silently runs on CPU (50x slower)
# The script will take 3 days instead of 1 hour and you won't know why
device = "cuda" if torch.cuda.is_available() else "cpu"
print(f"Using device: {device}")

# Load data (assuming it's pre-formatted — Bug 2: No data validation)
import json
with open("train_data.json") as f:
    raw_data = json.load(f)

dataset = Dataset.from_list(raw_data)

# Bug 3: Tokenization without max_length
# If any example is 10,000 tokens, it will OOM or train on extremely long sequences
def tokenize_fn(examples):
    return tokenizer(
        examples["text"],
        truncation=True,
        padding=True,
        # max_length=512 is MISSING
        # Without max_length, sequences vary wildly in length
        # This causes: (a) OOM on long sequences, (b) inefficient batching
    )

tokenized = dataset.map(tokenize_fn, batched=True)

# Load model (Bug 4: No quantization for memory)
model = AutoModelForCausalLM.from_pretrained(
    "mistralai/Mistral-7B-v0.1",
    torch_dtype=torch.float16,
    # load_in_4bit=True is MISSING
    # Full FP16 7B = 14GB just for weights
    # With optimizer states = 42GB total
    # Most GPUs can't handle this
)

# LoRA config
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
)
model = get_peft_model(model, lora_config)

# Bug 5: Learning rate too high for LoRA
# LoRA typically needs 1e-4 to 2e-4, but with a 7B model
# and small dataset, 5e-5 is safer
# 5e-4 runs the risk of DIVERGENCE
training_args = TrainingArguments(
    output_dir="./output",
    learning_rate=5e-4,  # Too high!
    per_device_train_batch_size=1,
    num_train_epochs=3,
    logging_steps=10,
    save_steps=500,
    # Bug 6: No evaluation strategy
    # eval_strategy="steps" is MISSING
    # Can't detect overfitting during training
    # Bug 7: No gradient checkpointing
    # gradient_checkpointing=True is MISSING
    # Uses MORE memory than necessary
    # Bug 8: No mixed precision
    # fp16=True is MISSING
    # Training is 2x slower
)

# Bug 9: No data collator — may cause padding issues
from transformers import DataCollatorForSeq2Seq
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized,
    # data_collator=DataCollatorForSeq2Seq(tokenizer) is MISSING
)

# Bug 10: No error handling for training failures
# If training crashes at epoch 2 (OOM), you lose ALL progress
# No checkpoint resumption, no graceful error handling
try:
    trainer.train()
except Exception as e:
    print(f"Training failed: {e}")
    # Bug 11: No cleanup
    # GPU memory might not be freed if training crashes
    # Next training run will start with fragmented memory
```

### 🔍 Find The 11 Bugs

**🤔 Before reading the fixed version:**

1. Which bugs are FATAL (training won't work at all)?
2. Which bugs are PERFORMANCE (training is slower/more expensive than necessary)?
3. Which bugs are QUALITY (training works but produces a worse model)?
4. What's the FIRST error this script would produce when run?

---

### Example 2: Production Training Script (Fixed)

```python
"""
Production Training Script with Unsloth/TRL

What this does RIGHT:
1. Memory-efficient (QLoRA + gradient checkpointing)
2. Proper evaluation during training
3. Learning rate scheduling with warmup
4. Gradient accumulation for effective batch size
5. Proper checkpointing with optimizer state
6. Resumption from checkpoints
7. Mixed precision (2x faster)
8. Data collator for proper padding
9. Memory estimation before training
10. Graceful error handling
"""

import torch
import gc
import json
import os
from typing import Optional

from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    Trainer,
    DataCollatorForSeq2Seq,
    EarlyStoppingCallback,
)
from datasets import Dataset, DatasetDict
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer


def estimate_memory(model_size_b: float = 7.0,
                    batch_size: int = 4,
                    max_length: int = 512,
                    use_qlora: bool = True,
                    lora_rank: int = 16) -> dict:
    """
    Estimate GPU memory BEFORE training starts.
    
    This prevents the "epoch 2 OOM" problem by telling you
    upfront if your configuration will fit.
    """
    # Base model (quantized if QLoRA)
    base_memory = model_size_b * 0.5 if use_qlora else model_size_b * 2
    
    # LoRA adapter weights (tiny)
    lora_params = 2 * (model_size_b / 28) * lora_rank * 4  # Approximate
    adapter_memory = lora_params * 2 / 1024  # MB → GB
    
    # Optimizer states (2x adapter for AdamW momentum + variance)
    optimizer_memory = adapter_memory * 2
    
    # Activations (depends on batch size and sequence length)
    activation_per_token = 2 * model_size_b  # 2 bytes per param per token
    activation_memory = (
        activation_per_token * batch_size * max_length / 1024 / 1024 / 1024
    )
    
    total = base_memory + adapter_memory + optimizer_memory + activation_memory + 0.5
    
    return {
        "base_model_gb": round(base_memory, 1),
        "adapter_gb": round(adapter_memory, 2),
        "optimizer_gb": round(optimizer_memory, 2),
        "activations_gb": round(activation_memory, 1),
        "overhead_gb": 0.5,
        "total_estimated_gb": round(total, 1),
        "recommended_gpu": (
            "RTX 4090 (24GB)" if total < 20
            else "A100 (40GB)" if total < 35
            else "A100 (80GB) or H100"
        ),
    }


def create_training_args(
    output_dir: str = "./output",
    num_epochs: int = 3,
    learning_rate: float = 2e-4,
    batch_size: int = 4,
    grad_accum: int = 8,
    max_length: int = 512,
    warmup_steps: int = 20,
    logging_steps: int = 10,
    eval_steps: Optional[int] = None,
    save_steps: int = 200,
    early_stop_patience: int = 3,
) -> TrainingArguments:
    """
    Create training arguments with proper defaults.
    
    Key parameter explanations:
    - learning_rate=2e-4: Standard for LoRA (10x higher than full FT)
    - grad_accum=8: Effective batch size = batch_size × grad_accum
    - warmup_steps=20: Gradually increase LR to prevent early divergence
    - fp16=True: Mixed precision for 2x training speed
    - gradient_checkpointing=True: Trade compute for memory
    """
    # Calculate eval frequency if not specified
    if eval_steps is None:
        # Evaluate 10 times per epoch
        eval_steps = max(10, 200 // grad_accum)
    
    return TrainingArguments(
        output_dir=output_dir,
        
        # Training
        num_train_epochs=num_epochs,
        per_device_train_batch_size=batch_size,
        per_device_eval_batch_size=batch_size,
        gradient_accumulation_steps=grad_accum,
        learning_rate=learning_rate,
        warmup_steps=warmup_steps,
        
        # Precision
        fp16=True,  # 2x speed, half memory
        # bf16=True,  # Use if GPU supports bfloat16 (A100, H100)
        
        # Memory
        gradient_checkpointing=True,
        gradient_checkpointing_kwargs={"use_reentrant": False},
        optim="adamw_torch",  # or "adamw_8bit" for more memory savings
        
        # Evaluation
        eval_strategy="steps",
        eval_steps=eval_steps,
        load_best_model_at_end=True,
        metric_for_best_model="eval_loss",
        greater_is_better=False,
        
        # Logging & saving
        logging_steps=logging_steps,
        save_strategy="steps",
        save_steps=save_steps,
        save_total_limit=3,  # Keep only 3 best checkpoints
        save_only_model=False,  # Save optimizer state too (for resumption)
        
        # Reporting
        report_to="none",  # Set to "wandb" for experiment tracking
        
        # Data
        dataloader_num_workers=2,
        ddp_find_unused_parameters=False if torch.cuda.device_count() > 1 else None,
        
        # Stability
        max_grad_norm=0.3,  # Gradient clipping to prevent divergence
    )


def setup_model_for_training(
    model_name: str = "unsloth/phi-2",
    lora_rank: int = 16,
    lora_alpha: float = 32,
    target_modules: list[str] = None,
    use_qlora: bool = True,
):
    """
    Set up model for efficient training.
    
    QLoRA: 4-bit quantized base + LoRA adapters
    - Base model: 4-bit (NF4) → 3.5GB for 7B model
    - Adapters: FP16 LoRA → 66MB for rank 16
    - Total: ~12GB → fits on RTX 4090
    """
    if target_modules is None:
        target_modules = ["q_proj", "v_proj"]
    
    # Memory estimation
    mem_estimate = estimate_memory(
        model_size_b=7.0,
        batch_size=4,
        max_length=512,
        use_qlora=use_qlora,
        lora_rank=lora_rank,
    )
    
    print("MEMORY ESTIMATE:")
    for key, value in mem_estimate.items():
        print(f"  {key}: {value}")
    print(f"  Recommended GPU: {mem_estimate['recommended_gpu']}")
    
    # Load tokenizer
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token
    
    # Load model (quantized)
    load_kwargs = {
        "device_map": "auto",
        "trust_remote_code": True,
    }
    
    if use_qlora:
        load_kwargs["load_in_4bit"] = True
        load_kwargs["bnb_4bit_compute_dtype"] = torch.float16
        load_kwargs["bnb_4bit_quant_type"] = "nf4"
        load_kwargs["bnb_4bit_use_double_quant"] = True
    else:
        load_kwargs["torch_dtype"] = torch.float16
    
    model = AutoModelForCausalLM.from_pretrained(model_name, **load_kwargs)
    
    # Prepare model for k-bit training (if quantized)
    if use_qlora:
        model = prepare_model_for_kbit_training(model)
    
    # LoRA configuration
    lora_config = LoraConfig(
        r=lora_rank,
        lora_alpha=lora_alpha,
        target_modules=target_modules,
        lora_dropout=0.0,  # No dropout with QLoRA (can destabilize)
        bias="none",
        task_type="CAUSAL_LM",
    )
    
    model = get_peft_model(model, lora_config)
    
    # Print trainable parameters
    trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
    total = sum(p.numel() for p in model.parameters())
    print(f"\nTrainable: {trainable:,} ({trainable/total*100:.2f}%)")
    print(f"Total:     {total:,}")
    
    return model, tokenizer


def train_with_resume(
    model,
    tokenizer,
    train_dataset,
    eval_dataset,
    training_args: TrainingArguments,
    resume_from: Optional[str] = None,
):
    """
    Train with checkpoint resumption support.
    
    If training crashes, you can resume from the LAST checkpoint:
    train_with_resume(..., resume_from="./output/checkpoint-500")
    
    The checkpoint saves:
    - Model weights
    - Optimizer state (AdamW momentum + variance)
    - Scheduler state (current learning rate)
    - Random state (reproducibility)
    """
    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=eval_dataset,
        data_collator=DataCollatorForSeq2Seq(
            tokenizer,
            pad_to_multiple_of=8,
        ),
        callbacks=[
            EarlyStoppingCallback(
                early_stopping_patience=3,
            ),
        ],
    )
    
    try:
        if resume_from:
            print(f"Resuming from checkpoint: {resume_from}")
            trainer.train(resume_from_checkpoint=resume_from)
        else:
            trainer.train()
    except Exception as e:
        print(f"\n⚠️ TRAINING INTERRUPTED: {e}")
        print(f"Checkpoint saved at: {training_args.output_dir}")
        print("To resume, pass resume_from='<last-checkpoint>'")
        
        # Save whatever we have
        trainer.save_model()
        raise
    
    # Save final model
    trainer.save_model()
    tokenizer.save_pretrained(training_args.output_dir)
    
    # Final evaluation
    eval_result = trainer.evaluate()
    print(f"\nFinal eval_loss: {eval_result['eval_loss']:.4f}")
    
    return trainer


def main():
    """Main training function with all safeguards."""
    
    # 1. Check CUDA
    if not torch.cuda.is_available():
        print("⚠️ CUDA NOT AVAILABLE — training will be EXTREMELY slow on CPU")
        response = input("Continue on CPU? (y/n): ")
        if response.lower() != 'y':
            return
    
    # 2. Estimate memory
    mem = estimate_memory()
    print(f"\nEstimated memory: {mem['total_estimated_gb']}GB")
    print(f"Available GPU: {torch.cuda.get_device_properties(0).total_mem / 1024**3:.0f}GB")
    
    if mem['total_estimated_gb'] > torch.cuda.get_device_properties(0).total_mem / 1024**3 * 0.9:
        print("⚠️ Estimated memory exceeds 90% of GPU memory — RISKY")
        print("   Consider: smaller batch size, shorter max_length, or higher quantization")
        response = input("Continue anyway? (y/n): ")
        if response.lower() != 'y':
            return
    
    # 3. Setup
    model, tokenizer = setup_model_for_training()
    
    # 4. Load data
    with open("train_data.json") as f:
        data = json.load(f)
    
    # 5. Split
    from sklearn.model_selection import train_test_split
    train_data, eval_data = train_test_split(data, test_size=0.1, random_state=42)
    
    train_dataset = Dataset.from_list(train_data)
    eval_dataset = Dataset.from_list(eval_data)
    
    # 6. Training args
    training_args = create_training_args(
        output_dir="./fine-tuned-model",
        num_epochs=3,
        learning_rate=2e-4,
    )
    
    # 7. Train
    trainer = train_with_resume(model, tokenizer, train_dataset, eval_dataset, training_args)
    
    print("\n✅ TRAINING COMPLETE")
    print(f"Model saved to: {training_args.output_dir}")


if __name__ == "__main__":
    main()
```


### Example 3: Training Monitor — Real-Time Dashboard

```python
"""
Training Monitor — Watch Training in Real Time

This is a separate SCRIPT that reads training logs and
provides real-time insights. Run it alongside training.

Usage:
  # In terminal 1: Run training
  python train.py
  
  # In terminal 2: Run monitor
  python monitor.py --log-dir ./output
"""

import time
import json
import os
import sys
from pathlib import Path


class TrainingMonitor:
    """
    Monitor training progress in real time.
    
    Reads the training logs written by Trainer and provides:
    - Current loss, learning rate, speed
    - Loss trend (improving, plateaued, diverging)
    - ETA to completion
    - Memory usage
    """
    
    def __init__(self, log_dir: str):
        self.log_dir = Path(log_dir)
        self.log_file = self._find_log_file()
        
        self.losses = []
        self.eval_losses = []
        self.learning_rates = []
        self.step_times = []
        self.start_time = time.time()
    
    def _find_log_file(self) -> Path:
        """Find the training log file (trainer_log.jsonl or similar)."""
        patterns = ["trainer_log.jsonl", "training_logs.jsonl", "logs/*.jsonl"]
        for pattern in patterns:
            files = list(self.log_dir.glob(pattern))
            if files:
                return sorted(files)[-1]
        raise FileNotFoundError(f"No training logs found in {self.log_dir}")
    
    def poll(self):
        """Read latest log entries and print status."""
        if not self.log_file.exists():
            print("Waiting for log file...")
            return False
        
        with open(self.log_file) as f:
            lines = f.readlines()
        
        for line in lines:
            try:
                entry = json.loads(line)
                
                if "loss" in entry:
                    self.losses.append(entry["loss"])
                    self.learning_rates.append(entry.get("learning_rate", 0))
                    self.step_times.append(entry.get("time", 0))
                
                if "eval_loss" in entry:
                    self.eval_losses.append(entry["eval_loss"])
            except json.JSONDecodeError:
                continue
        
        if not self.losses:
            return False
        
        self._display_status()
        return True
    
    def _display_status(self):
        """Print current training status."""
        current_loss = self.losses[-1]
        current_lr = self.learning_rates[-1]
        
        # Loss trend
        if len(self.losses) >= 10:
            recent = self.losses[-10:]
            if recent[-1] < recent[0] * 0.95:
                trend = "↓ IMPROVING"
            elif recent[-1] > recent[0] * 1.05:
                trend = "↑ DIVERGING ⚠️"
            else:
                trend = "→ PLATEAUED"
        else:
            trend = "? TOO EARLY"
        
        # Speed
        if len(self.step_times) >= 10:
            recent_times = self.step_times[-10:]
            avg_step_time = (recent_times[-1] - recent_times[0]) / len(recent_times)
            steps_per_sec = 1 / max(avg_step_time, 0.001)
        else:
            steps_per_sec = 0
        
        # Eval loss comparison
        eval_info = ""
        if self.eval_losses:
            eval_info = f"| Eval: {self.eval_losses[-1]:.4f}"
        
        # Overfitting warning
        if len(self.eval_losses) >= 3:
            recent_eval = self.eval_losses[-3:]
            if recent_eval[-1] > recent_eval[0]:
                eval_info += " ⚠️ OVERFITTING!"
        
        print(
            f"\rStep {len(self.losses)}: "
            f"loss={current_loss:.4f} "
            f"{eval_info} "
            f"lr={current_lr:.2e} "
            f"{trend} "
            f"speed={steps_per_sec:.1f} steps/s "
            f"elapsed={time.time() - self.start_time:.0f}s",
            end="",
            flush=True
        )
    
    def run(self, poll_interval: float = 2.0):
        """Run the monitor continuously."""
        print(f"Monitoring training in: {self.log_file}")
        print("─" * 60)
        
        try:
            while True:
                if not self.poll():
                    time.sleep(poll_interval)
                    continue
                
                # Check if training completed
                if self._check_completion():
                    print("\n\nTraining completed!")
                    break
                
                time.sleep(poll_interval)
                
        except KeyboardInterrupt:
            print("\n\nMonitor stopped.")
    
    def _check_completion(self) -> bool:
        """Check if training has completed."""
        done_file = self.log_dir / "training_complete.txt"
        return done_file.exists()


if __name__ == "__main__":
    log_dir = sys.argv[1] if len(sys.argv) > 1 else "./output"
    monitor = TrainingMonitor(log_dir)
    monitor.run()
```

---

### 🔍 Understanding Training Dynamics

Before moving on, here are 3 critical training patterns to recognize:

**Pattern 1: Healthy Training**
```
Loss starts high (2.0-3.0), decreases smoothly, plateaus at low value (0.5-1.0)
Eval loss closely follows training loss (gap < 0.2)
Learning rate follows the schedule (warmup → decay)
→ Keep training, this is working
```

**Pattern 2: Overfitting (most common)**
```
Training loss keeps decreasing (0.5 → 0.1 → 0.02)
Eval loss DECREASES then INCREASES (1.2 → 1.0 → 1.3 → 1.8)
Gap between train/eval widens
→ STOP TRAINING. Use earlier checkpoint.
→ Fix: Fewer epochs, more data, lower rank, higher dropout
```

**Pattern 3: Divergence (LR too high)**
```
Loss starts normal (2.0), then SPIKES (2.0 → 5.0 → 15.0 → NaN)
Learning rate stays high
→ TRAINING IS FAILING. Stop and restart with lower LR.
→ Fix: Lower learning rate, increase warmup steps
```

---

## ✅ Good Output Examples

### What a Healthy Training Run Looks Like

```
TRAINING RUN — LEGAL SUMMARIZATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Configuration:
  Model:              Llama-3.2-3B (QLoRA 4-bit)
  LoRA rank:          16
  Batch size:         4 (eff. batch: 32 with grad_accum=8)
  Max length:         1024
  Learning rate:      2e-4 (cosine, 20-step warmup)
  Epochs:             5 (early stopped at epoch 4)
  GPU:                RTX 4090 (24GB)
  Training time:      47 minutes

Loss Curves:
Step   Train Loss  Eval Loss  LR         Notes
   0     2.31       2.28       0.0e+0    Warmup
  20     1.85       1.82       2.0e-4    Warmup complete
  50     1.42       1.45       1.9e-4    Both decreasing
 100     1.12       1.18       1.7e-4    Gap stable (~0.06)
 150     0.89       1.02       1.4e-4    Good progress
 200     0.71       0.95       1.0e-4    Eval still decreasing
 250     0.58       0.92       0.6e-4    Gap widening ⚠️
 300     0.49       0.98       0.3e-4    Eval WORSENING! → STOP

Best checkpoint: Step 200 (eval_loss=0.95)
  → This is the checkpoint you deploy, NOT the final one

Memory:
  Peak: 18.2 GB / 24 GB (76%)
  → Comfortable, room for larger batch

Speed:
  Training: 8.2 steps/sec
  Total: 47 min for 4 epochs (304 steps)
```

### What a Failed Run Looks Like

```
TRAINING RUN — DISASTER
━━━━━━━━━━━━━━━━━━━━━━━

Configuration:
  Model:              Mistral-7B (FP16, no QLoRA!)  ← Problem 1
  Batch size:         8  ← Too large for FP16 7B
  Learning rate:      5e-4  ← Too high for LoRA
  GPU:                RTX 4090 (24GB)

Problems:
  ❌ OOM at epoch 2 (7B FP16 = 14GB + optimizer = 42GB + activations)
  ❌ Loss diverged at step 50 (LR 5e-4 too high)
  ❌ Checkpoint at step 100 was corrupted (power interruption)
  
Wasted: 2 hours, $1.50 of compute
Fix: Use QLoRA (4-bit), LR=2e-4, batch_size=4, enable gradient checkpointing
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Training Without Monitoring

```python
# BAD: Set and forget
trainer.train()
# No real-time monitoring
# No idea if loss is diverging
# No idea when overfitting starts
# If training crashes at 3AM, you find out at 9AM

# GOOD: Active monitoring
trainer.train()
# Have TrainingMonitor running in another terminal
# Set up Slack/email alerts for loss spikes
# Check every 30 minutes during first run
```

### Antipattern 2: Optimizing the Wrong Thing

```python
# BAD: Optimize for training loss
# "My training loss went from 2.0 to 0.01 — amazing!"
# Training loss near 0 = memorization, not learning

# GOOD: Optimize for eval loss
# "My eval loss went from 2.0 to 1.0 — model is generalizing"
# Eval loss improving = actual learning
```

### Common Training Failures:

| Failure | Symptom | Cause | Fix |
|---------|---------|-------|-----|
| **OOM** | CUDA OOM at some point | Memory exceeded | QLoRA, smaller batch, shorter max_length |
| **NaN loss** | Loss becomes NaN | Learning rate too high, numerical instability | Lower LR, gradient clipping, FP32 for unstable layers |
| **Loss divergence** | Loss increases over time | LR too high, bad initialization | Lower LR, increase warmup, check data |
| **No learning** | Loss barely decreases | LR too low, wrong format, frozen layers | Higher LR, check format, unfreeze layers |
| **Slow training** | 1 step/sec on A100 | No mixed precision, no gradient checkpointing | Enable fp16, gradient_checkpointing |
| **Resume failure** | Resume loss higher than saved | Missing optimizer state | Save full checkpoint, not just weights |

---

## 🧪 Drills & Challenges

### Drill 1: Diagnose Training Logs (30 min)

Given these training logs, identify what went wrong:

```python
# Log A
logs_a = [
    {"step": 0, "loss": 2.5, "eval_loss": 2.5, "lr": 0.0002},
    {"step": 10, "loss": 8.3, "eval_loss": None, "lr": 0.0002},
    {"step": 20, "loss": 15.7, "eval_loss": None, "lr": 0.0002},
    {"step": 30, "loss": NaN, "eval_loss": None, "lr": 0.0002},
]

# Log B
logs_b = [
    {"step": 0, "loss": 2.5, "eval_loss": 2.5, "lr": 2e-4},
    {"step": 50, "loss": 2.1, "eval_loss": 2.2, "lr": 1.9e-4},
    {"step": 100, "loss": 1.8, "eval_loss": 1.9, "lr": 1.7e-4},
    {"step": 150, "loss": 0.9, "eval_loss": 1.2, "lr": 1.4e-4},
    {"step": 200, "loss": 0.4, "eval_loss": 1.6, "lr": 1.0e-4},
    {"step": 250, "loss": 0.1, "eval_loss": 2.1, "lr": 0.6e-4},
]

# Log C
logs_c = [
    {"step": 0, "loss": 2.5, "eval_loss": 2.5, "lr": 2e-5},
    {"step": 50, "loss": 2.4, "eval_loss": 2.5, "lr": 1.9e-5},
    {"step": 100, "loss": 2.3, "eval_loss": 2.4, "lr": 1.7e-5},
    {"step": 150, "loss": 2.2, "eval_loss": 2.3, "lr": 1.4e-5},
    {"step": 200, "loss": 2.1, "eval_loss": 2.2, "lr": 1.0e-5},
]
```

For each log, answer:
1. What's the problem?
2. What's the root cause?
3. How would you fix it?

---

### Drill 2: Build a Memory Calculator (30 min)

```python
def calculate_training_memory(
    model_params_b: float = 7.0,
    batch_size: int = 4,
    max_length: int = 512,
    quantization: str = "4bit",  # "none", "8bit", "4bit"
    lora: bool = True,
    lora_rank: int = 16,
    grad_accum: int = 8,
    mixed_precision: bool = True,
    gradient_checkpointing: bool = True,
) -> dict:
    """
    Calculate GPU memory usage for a training configuration.
    
    Components:
    - Model weights (depends on quantization)
    - LoRA adapters (if lora=True)
    - Gradients (same size as trainable parameters)
    - Optimizer states (2x trainable for AdamW)
    - Activations (depends on batch_size, max_length, grad_ckpt)
    - Overhead (~0.5GB for CUDA contexts)
    
    Returns breakdown and total.
    """
    # Your implementation here
    pass
```

**🤔 Questions:**
1. How much memory does gradient checkpointing SAVE? (Hint: it trades compute for memory — activations are recomputed, not stored)
2. Why does a LARGER batch_size increase memory ENOUGH to matter? Where does the extra memory go?
3. For a 7B model with QLoRA, batch_size=4, max_length=1024: what's YOUR estimated memory? Does it fit on a 24GB GPU?

---

### Drill 3: Design the Resume Strategy (45 min)

Your training has been running for 6 hours. At step 2000 (of 3000), the GPU crashes (power failure).

Design a checkpoint and resume strategy that MINIMIZES lost progress:

```python
class CheckpointStrategy:
    """
    Design the checkpoint strategy for crash recovery.
    
    Constraints:
    - Saving a checkpoint takes 30 seconds
    - Each checkpoint is 4GB (with optimizer state)
    - You want to lose AT MOST 10 minutes of training on crash
    
    Questions:
    1. How often should you save?
    2. What state should each checkpoint contain?
    3. How many checkpoints to keep?
    4. What's the total storage needed?
    """
    
    def __init__(self, 
                 total_steps: int,
                 save_time_seconds: float = 30,
                 max_loss_minutes: float = 10):
        # Your implementation
        pass
    
    def recommend_save_frequency(self) -> int:
        """Calculate how many steps between saves."""
        pass
    
    def estimate_storage(self) -> dict:
        """Estimate total storage needed."""
        pass
```

**🤔 Questions:**
1. If you save every 50 steps (save_time=30s), you spend 0.6s per step just saving — that's ~30 minutes of total save time over the entire training. Is this acceptable?
2. If you DON'T save optimizer state, checkpoints are 10x smaller and 10x faster to save. But resumption has a loss spike. When would you skip optimizer state?
3. What if you save to CLOUD storage (S3) instead of local disk? How does network latency change your strategy?

---

## 🚦 Gate Check

Before moving to File 05 (Evaluation & Comparison), verify you can:

1. **Set up a training run** — Write a complete training script with: QLoRA model loading, proper tokenization, train/eval split, gradient accumulation, mixed precision, evaluation during training, early stopping, checkpointing with optimizer state

2. **Estimate memory before training** — Given model size, quantization, batch size, max length, and LoRA rank, estimate total GPU memory and recommend a compatible GPU

3. **Read training logs** — Given a training log, identify: overfitting (eval loss rising), divergence (loss spiking), underfitting (loss barely decreasing), optimal stopping point

4. **Handle training crashes** — Given a crash at step N, design a resumption strategy that loses minimal progress. Include: save frequency, state to save, resumption procedure

5. **Debug common failures** — Diagnose and fix: OOM, NaN loss, loss divergence, no learning, slow training

6. **Choose hyperparameters** — Given:
   - Model: 7B, QLoRA 4-bit
   - Dataset: 5,000 examples, max 1024 tokens
   - GPU: RTX 4090 (24GB)
   
   Recommend: batch_size, grad_accum, learning_rate, warmup_steps, epochs, save_steps

---

## 📚 Resources

### Training Frameworks
- **[Unsloth](https://github.com/unslothai/unsloth)** — 2x faster LoRA training, optimized kernels
- **[TRL (Transformer Reinforcement Learning)](https://huggingface.co/docs/trl)** — SFTTrainer for supervised fine-tuning
- **[HuggingFace SFTTrainer Docs](https://huggingface.co/docs/trl/main/en/sft_trainer)** — The standard fine-tuning trainer

### Debugging Training
- **[CUDA Memory Debugging](https://pytorch.org/docs/stable/notes/cuda.html#memory-management)** — Understanding GPU memory usage
- **[Training Stability](https://arxiv.org/abs/2304.13025)** — Preventing loss divergence and NaN

### Phase Cross-References
- **Phase 7 (Evals)** → Your Phase 7 eval platform evaluates the TRAINED model. Run eval BEFORE training (baseline) and AFTER training (improvement).
- **Phase 8 (Guardrails)** → Fine-tuned models need GUARDRAIL testing. Your Phase 8 guardrails catch regressions introduced by training.

> **Next up: File 05 — Evaluation & Comparison.** You've trained a model. Now PROVE it's better than the base. And prove it didn't get WORSE at things the base model was good at. This is where your Phase 7 eval platform comes in.
