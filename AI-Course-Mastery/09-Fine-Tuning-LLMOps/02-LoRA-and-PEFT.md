# 02 — LoRA & Parameter-Efficient Fine-Tuning

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about rank and alpha and adapter placement, you need to understand WHY LoRA exists.

Full fine-tuning updates ALL model parameters. For a 7B model, that's 7 billion parameters × 4 bytes × 3 states (weights, gradients, optimizer) = 84GB of GPU memory. You need an A100 (80GB) or H100 just to START.

But here's the surprising truth: **The changes made during fine-tuning are LOW-RANK.** You don't need to update all 7B parameters. The meaningful changes happen in a much smaller subspace.

LoRA exploits this. Instead of updating the full weight matrix W (size d×k), LoRA learns two smaller matrices A (size d×r) and B (size r×k), where r << min(d, k). The update is:

```
W' = W + ΔW = W + AB
```

Where:
- W is the PRE-TRAINED weight (frozen, never updated)
- A and B are TRAINABLE low-rank matrices
- r is the RANK (typically 8-64 vs. d=4096 for a 7B model)

**The parameter savings:**
- Full fine-tuning: Update 7B parameters
- LoRA (r=16): Update ~33M parameters (0.5% of full FT)
- Memory: 84GB → 16GB for a 7B model

But — and this is the critical insight — **LoRA doesn't TRAIN on fewer examples. It doesn't CONVERGE faster. It doesn't produce better results.** It just makes training CHEAPER.

The question isn't "Does LoRA work?" (it does). The question is: **What do you LOSE when you restrict fine-tuning to a low-rank subspace?** And more importantly, how do you choose r (rank), α (alpha), and target modules to MINIMIZE what you lose?

---

### 🤔 Discovery Question 1: The Rank Paradox

You fine-tune a model on a legal summarization task. You try different LoRA ranks:

| Rank | Trainable Parameters | Test Accuracy | Training Time |
|------|---------------------|---------------|---------------|
| 1    | 2M                  | 72%           | 20 min        |
| 2    | 4M                  | 78%           | 22 min        |
| 4    | 8M                  | 84%           | 25 min        |
| 8    | 16M                 | 88%           | 30 min        |
| 16   | 33M                 | 89%           | 40 min        |
| 32   | 66M                 | 89%           | 60 min        |
| 64   | 132M                | 87%           | 90 min        |
| 128  | 265M                | 85%           | 150 min       |

**🤔 Before reading on, answer these:**

1. Rank 16 → Rank 32: Double the parameters, SAME accuracy (89%). What is happening to the extra 33M parameters? Are they learning anything useful?

2. Rank 64 → Rank 128: MORE parameters, LOWER accuracy (87% → 85%). How is this possible? How can adding MORE trainable parameters make the model WORSE? (Hint: Think about what happens when you have too many degrees of freedom for a fixed-size dataset.)

3. **The hardest question:** Rank 8 gives 88% accuracy with 16M parameters in 30 minutes. Rank 32 gives 89% accuracy with 66M parameters in 60 minutes. The 1% improvement costs 4x more parameters and 2x more time. **Is it worth it?** How do you DECIDE the optimal rank for your specific task, dataset size, and budget?

---

### 🤔 Discovery Question 2: The Where-to-Adapt Problem

You apply LoRA to a 7B model. Your friend applies LoRA to the SAME model with the SAME rank. Same dataset, same epochs, same learning rate.

Your config: LoRA on ALL attention layers (query, key, value, output) = 32 adapters
Your friend's config: LoRA on query and value only = 16 adapters

Your accuracy: 82%
Your friend's accuracy: 87%

**🤔 Before reading on, answer these:**

1. You have MORE adapters (32 vs 16), more trainable parameters, more capacity to learn. But you got WORSE results. Why would applying LoRA to MORE layers hurt performance? (Hint: What happens when you modify too many layers simultaneously? Does the model's internal structure get disrupted?)

2. Different layers play different roles in the model:
   - Early layers: Learn basic syntax and surface patterns
   - Middle layers: Learn semantic representations
   - Later layers: Learn task-specific behaviors
   
   If your task is to change OUTPUT FORMAT (e.g., JSON → Markdown), which layers would you target? If your task is to learn DOMAIN-SPECIFIC VOCABULARY, which layers? If BOTH, do you need different adapters?

3. **The hardest question:** Your friend targets query and value projections and gets 87%. You target ALL projections and get 82%. But what if you targeted ONLY the MLP layers (not attention)? Or ONLY the output layer? Or ONLY layer 24-32 (last 8 layers)? **How do you find the OPTIMAL set of layers to adapt without trying every combination?** Is there a way to MEASURE which layers matter most for your task?

---

### 🤔 Discovery Question 3: The Merging Mystery

You fine-tune with LoRA. The adapter file is 66MB (small!). You deploy with the adapter SEPARATE from the base model (the inference server loads base model + adapter).

Your friend MERGES the adapter into the base model:
```
W_merged = W_base + W_adapter
```

They save the merged model as a single file. They deploy it without needing a separate adapter.

Your friend reports: "The merged model gets 84% accuracy. The unmerged model got 89% accuracy."

**🤔 Before reading on, answer these:**

1. Merging is a mathematical identity: W + AB = merged. If A and B are trained correctly, W + AB should be IDENTICAL to running the adapter separately. Why would the merged model perform DIFFERENTLY? (Hint: Quantization. The base model is usually quantized to 4-bit or 8-bit. What happens when you add adapter weights to quantized weights?)

2. When you merge, the adapter weights are converted to the same precision as the base model. If the base model is in 4-bit (NF4) and adapter weights are in FP16, the merge loses precision. How much does this affect performance? Is the accuracy drop WORTH the deployment simplicity?

3. **The hardest question:** You have MULTIPLE adapters for different tasks (legal, medical, code). Each adapter is 66MB. The base model is 4GB (quantized). If you run adapters SEPARATELY, you need 4GB + 66MB per adapter = ~4GB + small overhead. If you MERGE each adapter into the base model, you need 4GB PER ADAPTER. With 10 adapters, that's 4GB (separate) vs 40GB (merged). **Is merging worth the space cost? When would you merge vs. not merge?**

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 30 min |
| LoRA Intuition & Matrix Math | 30 min |
| Rank, Alpha, and Scaling | 25 min |
| Target Modules: Where to Adapt | 20 min |
| Code: LoRA from Scratch (Understanding) | 35 min |
| Code: LoRA with PEFT Library (Production) | 30 min |
| Code: Adapter Merging & Quantization | 25 min |
| Multi-Adapter Routing | 20 min |
| Other PEFT Methods (DoRA, QLoRA, AdaLoRA) | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 50 min |
| Gate Check | 15 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. LoRA Intuition — Why Low-Rank Works

Imagine you have a matrix W that maps "concept space" to "output space."

During pre-training, W learned to capture ALL concepts: grammar, facts, reasoning, style, tone, domain knowledge — everything.

When you fine-tune, you only want to ADJUST a small subset of these concepts. Maybe your task requires more "legal reasoning" and less "creative writing." You want to gently push W in the direction of legal reasoning.

The key insight from the LoRA paper: **Fine-tuning changes are LOW-RANK.** You don't need to modify W in all directions — just a few key directions (the rank r).

```
Visual intuition:

W (pre-trained, 4096×4096)
┌──────────────────────────────────────────────┐
│                                              │
│     Contains ALL knowledge (general)         │
│                                              │
└──────────────────────────────────────────────┘

ΔW = A×B (the LoRA update, rank r=8)
┌──────────┐  ┌──────────┐
│  A       │  │     B    │
│  (4096×8)│  │   (8×4096)│
│          │  │          │
└──────────┘  └──────────┘

A×B captures the MOST IMPORTANT directions of change
using only 2 × 4096 × 8 = 65,536 parameters
instead of 4096 × 4096 = 16,777,216 parameters

That's 256x fewer parameters!
```

### 2. Rank, Alpha, and Scaling

**Rank (r):** The dimension of the low-rank subspace.
- r=1: Only change in 1 direction — very constrained, may not capture task
- r=8: Good starting point for most tasks
- r=16: More capacity, useful for complex tasks
- r=64: Rarely needed, risk of overfitting

**Alpha (α):** The scaling factor for LoRA updates.
- The actual update is: `ΔW = α/r × AB`
- Higher α = stronger adaptation
- Rule of thumb: Set α = 2r (e.g., r=8, α=16)
- α acts like a LEARNING RATE for the adapter — higher = more aggressive adaptation

**Why α and r matter together:**
- If you double r AND double α, the adapter has more parameters but the same SCALE of update
- If you keep α fixed and increase r, the adapter has more parameters with WEAKER individual updates
- If you keep r fixed and increase α, the adapter has the same parameters with STRONGER updates

### 3. Target Modules — Where to Adapt

Different layers in a transformer do different things:

```
Layer Type    │ Role                               │ Adapt for...
──────────────┼────────────────────────────────────┼────────────────
Embedding     │ Token → vector                     │ New vocabulary (rare)
Early Attn    │ Syntax, surface patterns           │ Format changes
Mid Attn      │ Semantic relationships             │ Domain adaptation
Late Attn     │ Task-specific reasoning             │ Task learning
MLP layers    │ Knowledge storage & retrieval       │ Fact learning
Output layer  │ Vocabulary distribution             │ Style/tone changes
```

**Common target patterns:**

| Target | When | Why |
|--------|------|-----|
| `q_proj, v_proj` | Most tasks (default) | Balances capacity and efficiency |
| `q_proj, k_proj, v_proj, o_proj` | Complex tasks | Maximum attention adaptation |
| All linear layers | Maximum adaptation | Risk of overfitting |
| Only last N layers | Subtle style changes | Preserves most base knowledge |

---

## 💻 CODE EXAMPLES

### Example 1: LoRA from Scratch — Understanding the Math

Before using the PEFT library, let's understand what LoRA actually does to the weights:

```python
"""
LoRA from Scratch — The Math Behind the Magic

This is NOT production code. This is UNDERSTANDING code.
Read it to understand WHAT LoRA does.
"""

import torch
import torch.nn as nn
import math


class LoRALayer(nn.Module):
    """
    A single LoRA adapter layer.
    
    Wraps a pre-trained weight matrix W with low-rank updates.
    W stays FROZEN. A and B are trainable.
    
    Forward:
        output = W(x) + α/r × B(A(x))
        
    Where:
        W: pre-trained weight (frozen, d×k)
        A: low-rank matrix (trainable, d×r) — initialized with random Gaussian
        B: low-rank matrix (trainable, r×k) — initialized with zeros
        r: rank (how many directions to change)
        α: alpha (how strongly to apply the change)
    """
    
    def __init__(self, original_weight: torch.Tensor, 
                 rank: int = 8, alpha: float = 16):
        super().__init__()
        
        # Store original weight and freeze it
        self.original_weight = original_weight.detach().clone()
        self.original_weight.requires_grad = False  # FROZEN!
        
        d, k = original_weight.shape  # d=in_features, k=out_features
        
        # LoRA matrices
        # A: projects from d → r (down projection)
        self.A = nn.Parameter(torch.randn(d, rank) * 0.01)
        
        # B: projects from r → k (up projection)
        # Initialize to ZERO — so the adapter starts as a no-op
        self.B = nn.Parameter(torch.zeros(rank, k))
        
        self.alpha = alpha
        self.rank = rank
        self.scaling = alpha / rank
    
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        """
        Forward pass:
        1. Compute W(x) — the original pre-trained output
        2. Compute ΔW(x) = α/r × B(A(x)) — the low-rank update
        3. Return W(x) + ΔW(x) — the adapted output
        """
        # Original output (frozen)
        base_output = torch.matmul(x, self.original_weight.T)
        
        # LoRA update (trainable)
        # x is (batch, d), A is (d, r), B is (r, k)
        lora_output = torch.matmul(
            torch.matmul(x, self.A),  # xA: (batch, r)
            self.B                     # xAB: (batch, k)
        ) * self.scaling
        
        return base_output + lora_output


# DEMONSTRATION: LoRA vs Full Fine-Tuning

def demonstrate_lora_vs_full():
    """
    Show the parameter savings of LoRA vs full fine-tuning.
    
    For a single linear layer in a 7B model:
    - Input: 4096 dimensions
    - Output: 4096 dimensions
    - Weight matrix: 4096 × 4096 = 16.8M parameters
    """
    d, k = 4096, 4096  # Typical size for a 7B model layer
    
    # Full fine-tuning: update ALL parameters
    full_params = d * k  # 16,777,216
    full_memory = full_params * 4 * 3  # weights + gradients + optimizer (float32)
    # 16.8M × 4 × 3 = 201MB for ONE layer
    
    print("LoRA PARAMETER SAVINGS:")
    print(f"{'Rank':>8} | {'LoRA Params':>15} | {'vs Full':>10} | {'Memory':>10}")
    print("-" * 50)
    
    for rank in [1, 2, 4, 8, 16, 32, 64, 128]:
        lora_params = 2 * d * rank  # A(d×r) + B(r×k) = d×r + r×k = 2dr (when d=k)
        lora_memory = lora_params * 4 * 3  # float32
        ratio = lora_params / full_params * 100
        
        print(f"{rank:>8} | {lora_params:>15,} | {ratio:>9.4f}% | {lora_memory/1024/1024:>8.1f}MB")
    
    print(f"\nFull FT: {full_params:,} params, {full_memory/1024/1024:.0f}MB per layer")
    print(f"7B model has ~32 layers → Full FT: {full_memory*32/1024:.0f}GB for one layer type")
    print(f"7B model with LoRA (r=8): {2*d*8*32:,} params total")


demonstrate_lora_vs_full()
```

### 🔍 Understanding What You Just Read

**🤔 Before reading on, answer these:**

1. In the LoRALayer, B is initialized to ZEROS. Why? What would happen if B was initialized randomly like A?

2. The forward pass does: `base_output + lora_output * (α/r)`. If you set α = 0, what happens? If you set r = 4096 (same as the full weight dimension), what changes? Is it still "low-rank"?

3. During training, W is frozen. Only A and B are updated. But the GRADIENTS flow through W to reach A and B. What do the gradients through W look like? Are they computed even though W doesn't change?

---

### Example 2: LoRA with PEFT Library (Production Version)

Now let's see the ACTUAL production code. This uses HuggingFace's PEFT library:

```python
"""
LoRA Fine-Tuning with PEFT Library — Production Code

What to notice:
- The PEFT library handles ALL the LoRA math for you
- You specify: r, alpha, target_modules, and it injects adapters
- The base model stays frozen automatically
- Saving saves ONLY the adapter (tiny!)
"""

import torch
from transformers import (
    AutoModelForCausalLM, 
    AutoTokenizer, 
    TrainingArguments,
    Trainer,
    DataCollatorForSeq2Seq,
)
from peft import (
    LoraConfig, 
    get_peft_model, 
    TaskType,
    PeftModel,
)
from datasets import Dataset, DatasetDict


def setup_lora_model(
    base_model_name: str = "microsoft/phi-2",
    rank: int = 8,
    alpha: float = 16,
    target_modules: list[str] = None,
    dropout: float = 0.05,
) -> tuple:
    """
    Set up a model with LoRA adapters.
    
    This is the PRODUCTION approach.
    Default parameters are STARTING POINTS, not RULES.
    """
    if target_modules is None:
        # Default: adapt query and value projections
        # This is the most common pattern
        target_modules = ["q_proj", "v_proj"]
    
    # Load base model (quantized for memory efficiency)
    model = AutoModelForCausalLM.from_pretrained(
        base_model_name,
        torch_dtype=torch.float16,
        device_map="auto",
        # For larger models, add:
        # load_in_4bit=True,  # QLoRA — quantizes base model
    )
    
    tokenizer = AutoTokenizer.from_pretrained(base_model_name)
    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token
    
    # LoRA configuration
    lora_config = LoraConfig(
        task_type=TaskType.CAUSAL_LM,
        r=rank,                              # Rank of the update matrices
        lora_alpha=alpha,                     # Scaling factor
        lora_dropout=dropout,                 # Dropout for regularization
        target_modules=target_modules,        # Which modules to adapt
        bias="none",                          # Don't train bias terms
        # init_lora_weights="gaussian",       # How to initialize A (default: Gaussian)
    )
    
    # Wrap model with LoRA
    lora_model = get_peft_model(model, lora_config)
    
    # Print parameter breakdown
    trainable_params = sum(p.numel() for p in lora_model.parameters() if p.requires_grad)
    total_params = sum(p.numel() for p in lora_model.parameters())
    print(f"Trainable params: {trainable_params:,} ({trainable_params/total_params*100:.2f}%)")
    print(f"Total params: {total_params:,}")
    
    return lora_model, tokenizer


# HOWEVER — before you use this, ask yourself:
# Are these default target_modules RIGHT for YOUR task?

# 🤔 Think about this:
# For a FORMAT change (output JSON → output Markdown):
#   Which layers matter most for format? 
#   → Late layers (output distribution), early layers (surface patterns)
#   → Try: ["q_proj", "v_proj"] or ["q_proj", "v_proj", "o_proj"]
#
# For DOMAIN KNOWLEDGE (general chat → legal reasoning):
#   Which layers matter most for knowledge?
#   → Middle layers (semantic representation), MLP layers (knowledge storage)
#   → Try: ["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"]
#
# For STYLE change (formal → casual):
#   Which layers matter most for style?
#   → Late layers (output distribution), not early layers
#   → Try: Only target the last 8 layers' projections


def train_lora(
    lora_model,
    tokenizer,
    train_data: list[dict],
    eval_data: list[dict],
    output_dir: str = "./lora-adapter",
    num_epochs: int = 3,
    learning_rate: float = 2e-4,  # NOTE: 10x higher than full FT!
    batch_size: int = 4,
):
    """
    Train a LoRA model.
    
    Key differences from full fine-tuning:
    - Learning rate is 10x HIGHER (2e-4 vs 2e-5)
    - Training is MUCH faster (fewer parameters to update)
    - Memory is MUCH lower (no optimizer states for base model)
    """
    # Format data
    def format_example(example):
        return {
            "text": f"<|user|>\n{example['instruction']}\n<|assistant|>\n{example['response']}"
        }
    
    train_formatted = [format_example(e) for e in train_data]
    eval_formatted = [format_example(e) for e in eval_data]
    
    def tokenize_fn(examples):
        result = tokenizer(
            examples["text"],
            truncation=True,
            padding="max_length",
            max_length=512,
        )
        result["labels"] = result["input_ids"].copy()
        return result
    
    train_dataset = Dataset.from_list(train_formatted).map(tokenize_fn, batched=True)
    eval_dataset = Dataset.from_list(eval_formatted).map(tokenize_fn, batched=True)
    
    training_args = TrainingArguments(
        output_dir=output_dir,
        num_train_epochs=num_epochs,
        per_device_train_batch_size=batch_size,
        per_device_eval_batch_size=batch_size,
        learning_rate=learning_rate,
        warmup_steps=10,
        logging_steps=5,
        eval_strategy="steps",
        eval_steps=20,
        save_strategy="steps",
        save_steps=50,
        load_best_model_at_end=True,
        metric_for_best_model="eval_loss",
        fp16=True,
        gradient_checkpointing=True,
        save_total_limit=3,  # Keep only last 3 checkpoints
    )
    
    trainer = Trainer(
        model=lora_model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=eval_dataset,
        data_collator=DataCollatorForSeq2Seq(tokenizer, pad_to_multiple_of=8),
    )
    
    # Train
    trainer.train()
    
    # Save ONLY the LoRA adapter (not the base model)
    # This saves ~66MB instead of ~4GB
    lora_model.save_pretrained(output_dir)
    tokenizer.save_pretrained(output_dir)
    
    print(f"LoRA adapter saved to {output_dir}")
    print(f"Adapter size: ~{sum(p.numel() for p in lora_model.parameters() if p.requires_grad) * 4 / 1024 / 1024:.0f}MB")
    
    return trainer
```

### 🔍 Critical Questions About This Code

**🤔 Before running this, think about:**

1. **The learning rate**: LoRA uses 2e-4, which is 10x higher than full fine-tuning's 2e-5. Why? Because LoRA updates fewer parameters, each parameter needs to change MORE to have the same effect. But what happens if you use 2e-4 with a LARGE rank (r=64)? Is that the same as 2e-5 with small rank? (No — the interaction between rank and LR is complex.)

2. **The target_modules default**: Most tutorials say "use ['q_proj', 'v_proj']". But this is based on experiments with SPECIFIC models (Llama) on SPECIFIC tasks (summarization). For YOUR model (Phi-2, Mistral, Gemma) on YOUR task, the optimal modules might be DIFFERENT. How do you discover the right modules for your case?

3. **Saving only the adapter**: The code saves only the adapter weights (66MB), not the base model (4GB). To INFER, you need: base model + adapter. What happens if the base model gets updated (new version from HuggingFace)? Your adapter might not be compatible. How do you handle base model versioning?

---

### Example 3: Merging Adapters — When and How

```python
"""
Adapter Merging — Combining Base Model + LoRA Adapter

When you merge:
- W_merged = W_base + (α/r) × AB
- The result is a SINGLE weight matrix
- You can save and load it like a regular model

When to MERGE:
✅ Deployment simplicity (single model file)
✅ Faster inference (no adapter computation overhead)
✅ Compatibility (works with any inference server)

When NOT to merge:
❌ You need to switch adapters frequently
❌ You use quantization (merging loses precision)
❌ You want to keep base model unmodified
"""

from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer


def merge_and_save(
    base_model_name: str = "microsoft/phi-2",
    adapter_path: str = "./lora-adapter",
    output_path: str = "./merged-model",
):
    """
    Merge LoRA adapter into base model and save as single file.
    
    ⚠️ WARNING: Merging is IRREVERSIBLE.
    Once merged, you can't separate the adapter from the base model.
    Always keep the original adapter file separately!
    """
    # Load base model
    base_model = AutoModelForCausalLM.from_pretrained(
        base_model_name,
        torch_dtype=torch.float16,
        device_map="auto",
    )
    
    # Load adapter
    model = PeftModel.from_pretrained(base_model, adapter_path)
    
    # Merge adapter weights into base model
    merged_model = model.merge_and_unload()
    # After this: merged_model is a STANDARD model (no LoRA)
    # The adapter weights are PERMANENTLY folded into the base weights
    
    # Save merged model
    merged_model.save_pretrained(output_path)
    tokenizer = AutoTokenizer.from_pretrained(base_model_name)
    tokenizer.save_pretrained(output_path)
    
    print(f"Merged model saved to {output_path}")
    print(f"Model size: {sum(p.numel() for p in merged_model.parameters()) * 4 / 1024 / 1024 / 1024:.1f}GB")
    
    return merged_model


# ⚠️ WHAT CAN GO WRONG WITH MERGING?

# Problem 1: Quantization + Merging = Precision Loss
# If base model is in 4-bit and adapter is in FP16:
# merge() converts adapter weights to 4-bit → precision loss
# Result: merged model ≈ 4-bit (slightly worse than unmerged)
# Solution: Keep adapter separate when using quantization

# Problem 2: Floating Point Accumulation
# Adapter weights have small values (trained with high precision)
# Adding them to base weights can cause rounding errors
# Especially noticeable in FP16 (half precision)
# Result: Merged model may produce slightly different outputs

# Problem 3: Adapter Sharing
# If you merge adapter A into model → save as "phi-2-legal"
# Now you can't use adapter B (medical) with the same base
# You'd need a separate merged model for each adapter
# Solution: Keep adapters separate if you have > 2-3 adapters


def demonstrate_merge_impact(base_model, adapter_path, test_inputs: list[str]):
    """
    Compare unmerged vs merged model outputs.
    
    If there's a SIGNIFICANT difference between unmerged
    and merged outputs, the merge damaged the model.
    """
    from peft import PeftModel
    
    # Unmerged: base + adapter (separate)
    unmerged = PeftModel.from_pretrained(
        AutoModelForCausalLM.from_pretrained(base_model, device_map="auto"),
        adapter_path,
    )
    
    # Merged: single model
    merged_base = AutoModelForCausalLM.from_pretrained(base_model, device_map="auto")
    merged = PeftModel.from_pretrained(merged_base, adapter_path)
    merged = merged.merge_and_unload()
    
    # Compare outputs
    differences = 0
    for test_input in test_inputs:
        # Get unmerged output
        unmerged_output = generate(unmerged, test_input)
        
        # Get merged output
        merged_output = generate(merged, test_input)
        
        # Compare
        if unmerged_output != merged_output:
            differences += 1
            print(f"DIFFERENCE for: '{test_input[:30]}...'")
            print(f"  Unmerged: {unmerged_output[:50]}")
            print(f"  Merged:   {merged_output[:50]}")
    
    print(f"\n{len(test_inputs)} tests: {differences} differences found")
    if differences == 0:
        print("✅ Merge is lossless (no quality degradation)")
    else:
        print(f"⚠️ {differences/len(test_inputs)*100:.0f}% of outputs changed after merge")
```

---

### Example 4: Multi-Adapter Routing (Production Pattern)

```python
"""
Multi-Adapter Routing — One Base Model, Many Adapters

Production pattern for serving multiple fine-tuned models
from a SINGLE base model.

Benefits:
- One base model in GPU memory (4GB quantized)
- Many adapters (66MB each) loaded on demand
- Total memory: 4GB + N × 66MB
- vs. N merged models: N × 4GB

Use cases:
- Multi-tenant SaaS (each client gets a custom adapter)
- Multi-task system (legal, medical, code in one service)
- A/B testing (different adapter versions)
"""

from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel
import time


class MultiAdapterRouter:
    """
    Route requests to different LoRA adapters dynamically.
    
    Architecture:
    - Load base model ONCE into GPU memory
    - Load adapters ON DEMAND (they're tiny: 66MB)
    - Switch adapters without reloading base model
    
    Adapter switching cost: ~10-50ms (vs 5-30s to reload base model)
    """
    
    def __init__(self, base_model_name: str = "microsoft/phi-2"):
        self.base_model_name = base_model_name
        
        # Load base model once
        print(f"Loading base model: {base_model_name}")
        self.base_model = AutoModelForCausalLM.from_pretrained(
            base_model_name,
            torch_dtype=torch.float16,
            device_map="auto",
        )
        self.tokenizer = AutoTokenizer.from_pretrained(base_model_name)
        if self.tokenizer.pad_token is None:
            self.tokenizer.pad_token = self.tokenizer.eos_token
        
        # Adapter cache
        self.adapters: dict[str, PeftModel] = {}
        self.current_adapter: str = "none"
        self.load_times: dict[str, float] = {}
    
    def load_adapter(self, adapter_name: str, adapter_path: str):
        """Load a LoRA adapter (cached for reuse)."""
        if adapter_name in self.adapters:
            return
        
        start = time.time()
        print(f"  Loading adapter: {adapter_name}")
        
        # Load adapter on top of base model
        model = PeftModel.from_pretrained(self.base_model, adapter_path)
        self.adapters[adapter_name] = model
        
        elapsed = time.time() - start
        self.load_times[adapter_name] = elapsed
        print(f"  Loaded in {elapsed*1000:.0f}ms")
    
    def switch_adapter(self, adapter_name: str):
        """Switch to a different adapter."""
        if adapter_name not in self.adapters:
            raise ValueError(f"Adapter {adapter_name} not loaded")
        
        if self.current_adapter == adapter_name:
            return  # Already using this adapter
        
        # Disable current adapter
        if self.current_adapter in self.adapters:
            self.adapters[self.current_adapter].disable_adapters()
        
        # Enable target adapter
        self.adapters[adapter_name].enable_adapters()
        self.current_adapter = adapter_name
    
    def generate(self, prompt: str, adapter_name: str) -> str:
        """Generate text using a specific adapter."""
        self.switch_adapter(adapter_name)
        
        inputs = self.tokenizer(prompt, return_tensors="pt").to(self.base_model.device)
        
        model = self.adapters[adapter_name]
        outputs = model.generate(
            **inputs,
            max_new_tokens=256,
            temperature=0.7,
        )
        
        return self.tokenizer.decode(outputs[0], skip_special_tokens=True)
    
    def get_stats(self) -> dict:
        return {
            "base_model": self.base_model_name,
            "loaded_adapters": list(self.adapters.keys()),
            "current_adapter": self.current_adapter,
            "load_times": self.load_times,
            "adapter_counts": {
                "loaded": len(self.adapters),
                "memory_per_adapter_mb": 66,  # Approximate
            }
        }


# DEMONSTRATION

router = MultiAdapterRouter("microsoft/phi-2")

# Load adapters (simulated paths)
router.load_adapter("legal", "./adapters/phi2-legal-summary")
router.load_adapter("medical", "./adapters/phi2-medical-qa")
router.load_adapter("code", "./adapters/phi2-code-assist")

# Route requests dynamically
for query, adapter in [
    ("Summarize this contract: ...", "legal"),
    ("What are symptoms of diabetes?", "medical"),
    ("Write a Python function to sort a list", "code"),
]:
    response = router.generate(query, adapter)
    print(f"[{adapter}] {response[:100]}...")

stats = router.get_stats()
print(f"\nStatistics: {stats}")
```

**🤔 Design questions:**
1. What happens when TWO users request DIFFERENT adapters simultaneously? Is the adapter switch THREAD-SAFE?
2. If you have 20 adapters, each 66MB, that's 1.3GB for adapters + 4GB for base = 5.3GB total. VS 20 merged models = 80GB. When does the memory savings stop being worth the complexity?
3. What if an adapter was trained on an OLDER version of the base model? Will it still work?

---

## ✅ Good Output Examples

### What a Well-Tuned LoRA Configuration Looks Like

```
LORA CONFIGURATION — LEGAL SUMMARIZATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Configuration:
  Model:              Llama-3.2-3B
  Rank (r):           16
  Alpha (α):          32  (α=2r)
  Dropout:            0.05
  Target modules:     q_proj, v_proj
  Bias:               none
  Optimizer:          AdamW (lr=2e-4)
  Batch size:         8
  Gradient steps:     500

Training Results:
  Train loss:         1.12 → 0.45
  Eval loss:          1.18 → 0.52
  Perplexity:         3.25 → 1.68
  Training time:      35 min (on RTX 4090)
  Max memory:         14.2 GB

Parameter Breakdown:
  Base model:         3.0B (frozen)
  LoRA adapters:      6.6M (trainable, 0.22% of total)
  Adapter file size:  26 MB

Performance:
  Before FT:          72% accuracy, 2.1s latency
  After FT:           91% accuracy, 2.1s latency (same!)
  Cost:               $0.50 training, $0.002/inference

Comparison:
  Full FT equivalent: $15 training, 48GB memory, 3h time
  LoRA savings:       97% cheaper, 71% less memory, 83% faster
```

### Multi-Adapter System Performance

```
MULTI-ADAPTER ROUTING — 10 ADAPTERS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Setup:
  Base model:         Mistral-7B (quantized 4-bit)
  Adaptérs:           10 (legal, medical, code, finance, ...)
  Memory base:        4.2 GB (quantized)
  Memory per adapter: 0 MB (loaded on demand)
  Total memory:       4.2 GB + 0.3 GB overhead = 4.5 GB

vs. 10 merged models: 42 GB (10×)

Adapter switching latency:
  Cold load:          120ms (from disk)
  Warm switch:        2ms (already in memory)
  
Request routing:
  User → Router → Detect intent → Load/Switch adapter → Generate
  Overhead: ~5ms per request (amortized)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Rank Too High for Dataset Size

```python
# BAD: Rank 64 with 200 examples
# More parameters than data points → model memorizes noise
lora_config = LoraConfig(r=64, lora_alpha=128, ...)
# 64M trainable parameters, 200 examples → 320,000 params per example!
# → Overfitting guaranteed

# GOOD: Match rank to dataset size
# Rule of thumb: rank = min(8, sqrt(dataset_size / 100))
# 200 examples → rank 1-2
# 1,000 examples → rank 4-8
# 10,000 examples → rank 8-16
lora_config = LoraConfig(r=4, lora_alpha=8, ...)
```

### Antipattern 2: Same Target Modules for Every Task

```python
# BAD: Copy-paste from tutorial
# Every tutorial uses ["q_proj", "v_proj"] (from Llama paper)
# But YOUR task might need different modules!
lora_config = LoraConfig(target_modules=["q_proj", "v_proj"])
# Works for Llama on summarization, may fail for Phi-2 on code

# GOOD: Match target modules to task
# Style/tone change → only late layers
lora_config = LoraConfig(
    target_modules=["q_proj", "v_proj"],
    layers_to_transform=list(range(24, 32)),  # Last 8 layers only
)
# Knowledge intensive → all attention + MLP
lora_config = LoraConfig(
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj", 
                     "gate_proj", "up_proj", "down_proj"],
)
```

### Antipattern 3: Merging with Quantization

```python
# BAD: Load in 4-bit → Adapter in FP16 → Merge → Precision loss
model = AutoModel.from_pretrained(..., load_in_4bit=True)
model = PeftModel.from_pretrained(model, adapter_path)
merged = model.merge_and_unload()  # Converts FP16 adapter to 4-bit → LOSES quality

# GOOD: Keep adapter separate when using quantization
model = AutoModel.from_pretrained(..., load_in_4bit=True)
model = PeftModel.from_pretrained(model, adapter_path)
# DON'T merge — serve with adapter on top
# Quality preserved, slight latency overhead from adapter computation

# OR: Load in FP16 for merging, then quantize AFTER merge
model = AutoModel.from_pretrained(...)  # FP16
model = PeftModel.from_pretrained(model, adapter_path)
merged = model.merge_and_unload()  # FP16 merge (lossless)
quantized = quantize(merged)  # Now quantize (single step, less lossy)
```

### Common Failure Modes:

| Failure Mode | Symptom | Root Cause | Fix |
|---|---|---|---|
| **Adapter does nothing** | FT model = base model | α too low, LR too low, rank too low | Increase α/rank/LR, check gradients |
| **Adapter ruins model** | Output becomes gibberish | α too high, LR too high, trained too long | Lower α/LR/epochs, check eval loss |
| **Merge degrades quality** | Merged model worse than unmerged | Quantization + merge precision loss | Keep adapter separate, or FP16 merge |
| **Multi-adapter interference** | Adapter A affects performance on B | Adapters share base model weights | Unload adapter A before loading B |
| **Adapter incompatible** | Error loading adapter | Base model version mismatch | Track base model hash, pin versions |
| **OOM with multiple adapters** | GPU memory full | Too many adapters loaded simultaneously | Load/unload on demand, store on CPU |

---

## 🧪 Drills & Challenges

### Drill 1: Find the Optimal Rank (45 min)

You have a dataset of 5,000 examples for a code generation task. You want to find the OPTIMAL LoRA rank.

```python
# Design an experiment to find the best rank

def find_optimal_rank(
    base_model: str,
    train_data: list[dict],
    eval_data: list[dict],
    ranks_to_try: list[int] = [1, 2, 4, 8, 16, 32, 64],
) -> dict:
    """
    Run LoRA fine-tuning with different ranks and return results.
    
    For each rank, track:
    - Final eval loss
    - Training time
    - GPU memory used
    - Model size after training
    
    Return analysis with recommendation.
    """
    results = {}
    
    for rank in ranks_to_try:
        print(f"\n=== Testing rank {rank} ===")
        
        # Setup LoRA with this rank
        # Train
        # Evaluate
        # Record metrics
        
        results[rank] = {
            "eval_loss": None,  # Fill in
            "train_time_min": None,
            "memory_gb": None,
            "model_size_mb": None,
        }
    
    return results


# 🤔 After running:
# 1. Plot eval_loss vs rank. What shape do you see?
# 2. At what rank does the loss plateau?
# 3. Is there a rank where loss starts INCREASING (overfitting)?
# 4. What's your recommended rank given: 5K examples, code gen task?
```

**Expected findings:**
- Rank 1-4: Underfitting (high loss)
- Rank 8-16: Sweet spot (lowest loss, good efficiency)
- Rank 32-64: Plateau or overfitting (no improvement, maybe worse)
- Recommended: r=8 for speed, r=16 for max quality

---

### Drill 2: Find the Right Target Modules (45 min)

You have 5 different tasks. For each, WHICH layers would you target with LoRA?

```python
tasks = [
    {
        "name": "style_transfer",
        "description": "Change formal writing to casual, friendly tone",
        "change_type": "surface",  # Surface-level change
    },
    {
        "name": "domain_adaptation",
        "description": "Adapt general model to legal terminology and reasoning",
        "change_type": "semantic",  # Deeper semantic change
    },
    {
        "name": "format_change",
        "description": "Change output from paragraphs to bullet points",
        "change_type": "structural",  # Output structure change
    },
    {
        "name": "knowledge_update",
        "description": "Teach the model about new product features (not in training data)",
        "change_type": "factual",  # SHOULD NOT be done via FT!
    },
    {
        "name": "instruction_following",
        "description": "Make model better at following complex multi-step instructions",
        "change_type": "reasoning",  # Reasoning pattern change
    },
]
```

**For EACH task, answer:**
1. Would you use LoRA for this? Or is there a BETTER approach (prompting, RAG)?
2. Which target_modules would you choose? (q_proj only? all attention?)
3. Would you target ALL layers or specific layer ranges?
4. What rank would you use? Why?
5. What's the RISK of this approach? (Forgetting, overfitting, etc.)

---

### Drill 3: Debug the Broken LoRA Config (30 min)

```python
# BROKEN LORA CONFIG — find and fix the 5 bugs

# Bug 1: ???
lora_config = LoraConfig(
    r=256,  # Is this too high? For a 200-example dataset?
    lora_alpha=512,
    lora_dropout=0.0,  # No dropout?
    target_modules=["embed_tokens"],  # Embedding layer?
    bias="all",  # Train all biases?
)

# Bug 2: ???
model = AutoModelForCausalLM.from_pretrained(
    "mistralai/Mistral-7B-v0.1",
    load_in_4bit=True,  # 4-bit base model
)
model = get_peft_model(model, lora_config)

# Bug 3: ???
training_args = TrainingArguments(
    learning_rate=2e-5,  # Same as full FT?
    num_train_epochs=20,  # For 200 examples?
    # ...
)

# Bug 4: ???
trainer.train()
merged = model.merge_and_unload()  # After 4-bit training?
```

**🤔 What are the 5 bugs? For each:**
1. What's wrong?
2. Why is it wrong?
3. What should it be instead?
4. What would happen if you ran it as-is?

---

## 🚦 Gate Check

Before moving to File 03 (Dataset Preparation), verify you can:

1. **Explain LoRA mathematically** — Write the forward pass formula and explain what each variable (W, A, B, r, α) means and how they interact

2. **Choose an appropriate rank** — Given:
   - Dataset size: 500 examples
   - Task: Format change (JSON → Markdown)
   - Model: 7B parameters
   
   Recommend r, α, and target_modules. Justify each choice.

3. **Decide merge vs. not merge** — Given:
   - You have 3 adapters (legal, medical, code)
   - Base model is quantized to 4-bit
   - Inference latency must be < 500ms
   - GPU memory: 16GB
   
   Should you merge or not? Why? What's the tradeoff?

4. **Design a multi-adapter system** — Given 20 clients, each with a custom adapter, and one GPU with 24GB memory:
   - Base model: 7B quantized (~4GB)
   - Each adapter: ~66MB
   - Design the loading/unloading strategy
   - Estimate: cold start time, switching latency, memory usage

5. **Identify overfitting in LoRA** — Given these training logs, identify which runs are overfitted and why:
   - Run A: r=64, 100 examples, 10 epochs, eval_loss=3.2
   - Run B: r=8, 10,000 examples, 3 epochs, eval_loss=1.1
   - Run C: r=16, 1,000 examples, 5 epochs, eval_loss=1.5

6. **Calculate LoRA savings** — Given:
   - Model: 7B parameters, 28 layers, each layer has q,k,v,o (4 matrices)
   - Compare FULL FT vs LoRA (r=16, target: q,v)
   - Parameter count savings
   - Memory savings (weights + optimizer states)
   - Training time savings (assume same batch size)

7. **Handle adapter incompatibility** — The base model was updated from v1.0 to v1.1. Your adapter was trained on v1.0. When you try to load it, you get errors. What are your options?

---

## 📚 Resources

### Foundational Papers
- **[LoRA: Low-Rank Adaptation of LLMs](https://arxiv.org/abs/2106.09685)** — The original paper (essential reading)
- **[QLoRA: Efficient Finetuning of Quantized LLMs](https://arxiv.org/abs/2305.14314)** — LoRA + 4-bit quantization
- **[DoRA: Weight-Decomposed Low-Rank Adaptation](https://arxiv.org/abs/2402.09353)** — Improved LoRA with magnitude/direction decoupling
- **[AdaLoRA: Adaptive Budget Allocation for PEFT](https://arxiv.org/abs/2303.10512)** — Automatically allocates rank across layers

### Practical Libraries
- **[PEFT Library](https://huggingface.co/docs/peft)** — HuggingFace's PEFT implementation (LoRA, DoRA, AdaLoRA, IA3, etc.)
- **[Unsloth](https://github.com/unslothai/unsloth)** — 2x faster LoRA training with optimized kernels

### Production Patterns
- **[Multi-Adapter Serving](https://www.anyscale.com/blog/llm-adapter-serving)** — How to serve multiple adapters efficiently
- **[LoRA Land](https://arxiv.org/abs/2405.00732)** — 310 fine-tuned LLMs with LoRA, analysis of rank/target selection

### Phase Cross-References
- **Phase 1 (Gateway)** → Your gateway (Phase 1) routes to models. With LoRA, it routes to ADAPTERS instead.
- **Phase 4 (RAG)** → LoRA can adapt the retrieval model too (embedding model fine-tuning). RAG + adapted LLM = powerful combo.
- **Phase 7 (Evals)** → Your eval platform evaluates base model vs. LoRA-adapted model. Compare before/after.
- **Phase 8 (Guardrails)** → LoRA can train SAFETY adapters that make models safer without full retraining.

> **Next up: File 03 — Dataset Preparation.** LoRA is only as good as your data. Bad data → bad model, regardless of rank or target modules. You'll learn why "more data" is often the WRONG goal, and how to build datasets that actually teach the model what you want.
