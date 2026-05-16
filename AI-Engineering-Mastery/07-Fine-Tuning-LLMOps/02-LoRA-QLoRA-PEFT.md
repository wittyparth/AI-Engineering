# LoRA / QLoRA / PEFT

## Why Parameter-Efficient Fine-Tuning

Full fine-tuning of a 70B model costs ~$500K in compute. LoRA costs ~$100. The difference is that LoRA trains a tiny set of parameters instead of all of them.

## How LoRA Works

Instead of modifying the original weights W, LoRA learns a low-rank decomposition:

```
W' = W + BA
    ↑     ↑  ↑
    orig  │  small matrix (r=8-64)
          small matrix
```

- Original W: 4096 × 4096 = 16.8M parameters (full rank)
- LoRA B, A: 4096 × 8 + 8 × 4096 = 65K parameters (0.4%)

## Implementation

```python
from peft import LoraConfig, get_peft_model, TaskType

lora_config = LoraConfig(
    r=16,  # Rank (higher = more capacity, more memory)
    lora_alpha=32,  # Scaling factor
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],  # Which layers to adapt
    lora_dropout=0.1,
    bias="none",
    task_type=TaskType.CAUSAL_LM,
)

model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.2-3B")
model = get_peft_model(model, lora_config)
```

## QLoRA: LoRA + Quantization

QLoRA = 4-bit quantized base model + LoRA adapters on top. Run a 70B on a single GPU.

```python
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.2-70B",
    quantization_config=BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_use_double_quant=True,
        bnb_4bit_quant_type="nf4",
        bnb_4bit_compute_dtype=torch.bfloat16,
    ),
    device_map="auto",
)
model = get_peft_model(model, lora_config)
```

## Training Script

```python
from transformers import TrainingArguments, Trainer

training_args = TrainingArguments(
    output_dir="./llama-lora-finetuned",
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    num_train_epochs=3,
    logging_steps=10,
    save_strategy="epoch",
    fp16=True,
    report_to="wandb",  # Log to Weights & Biases
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dataset,
    data_collator=data_collator,
)
trainer.train()

# Save only the LoRA adapters (tiny!)
model.save_pretrained("./lora-adapters")
```

## 🔴 Senior: LoRA Configuration Guide

| r (Rank) | Capacity | Memory | When |
|----------|----------|--------|------|
| 8 | Low | Minimal | Small dataset, similar task |
| 16 | Medium | Low | Default for most tasks |
| 32 | High | Medium | Complex behavior change |
| 64 | Very high | High | Major task change |

**Rule of thumb**: Start with r=16. Only increase if the fine-tuned model isn't learning the target behavior.

## Drill: Fine-tune LLM with LoRA

1. Take a small base model (Llama 3.2 3B or Mistral 7B)
2. Prepare a dataset (1000+ examples)
3. Fine-tune with LoRA (r=16)
4. Evaluate: did the behavior change as expected?
5. Compare before and after on 20 test prompts
