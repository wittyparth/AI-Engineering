# ⛩️ GATE 09 — Sculptor's Forge (Fine-Tuning)

**Rank Requirement:** A-
**Theme:** Fine-tuning LLMs — data preparation, training, evaluation, and deployment of custom models
**Total Days:** 7 Dungeons + 1 Boss Battle
**Core Skills:** Dataset preparation, LoRA, QLoRA, training recipes, evaluation, distillation

---

## Dungeon 1 — Fine-Tuning vs RAG

**Day 1 | Concept:** When to fine-tune vs when to use RAG. They're complementary, not competing.

| Aspect | RAG | Fine-Tuning |
|--------|-----|-------------|
| Knowledge source | External documents | Model weights |
| Update cost | Re-index (cheap) | Re-train (expensive) |
| New knowledge | Instant | Takes hours/days |
| Behavior change | Prompt engineering | Deep model change |
| Tone/styling | Limited | Full control |
| Hallucination | Lower (grounded) | May retain pre-training |
| Compute cost | Low (inference only) | High (training + inference) |

**When to Fine-Tune:**
- You need consistent tone/structure (e.g., always output JSON)
- You need to teach the model *how* to do something
- You have limited context window for examples
- You want lower latency (no retrieval step)

**When to RAG:**
- Knowledge changes frequently
- You need citation sources
- You don't have training data
- You need to ground in specific documents

**🗡️ Dungeon Mastery:** Given 3 scenarios (customer support bot, code assistant, medical Q&A), decide: RAG, fine-tune, or both. Justify each choice.

**💻 Code — Decision Framework:**
```python
class FineTuneOrRAG:
    @staticmethod
    def evaluate(requirements: dict) -> dict:
        scores = {"rag": 0, "fine_tune": 0}

        # Knowledge change frequency
        if requirements.get("knowledge_change", "monthly") in ["daily", "weekly"]:
            scores["rag"] += 3
        else:
            scores["fine_tune"] += 2

        # Output format strictness
        if requirements.get("strict_format", False):
            scores["fine_tune"] += 3

        # Training data availability
        if requirements.get("training_samples", 0) >= 500:
            scores["fine_tune"] += 2

        # Latency requirements
        if requirements.get("max_latency_ms", 1000) < 500:
            scores["fine_tune"] += 1  # No retrieval step

        # Citation requirement
        if requirements.get("require_citations", False):
            scores["rag"] += 3

        best = "rag" if scores["rag"] > scores["fine_tune"] else "fine_tune"
        if abs(scores["rag"] - scores["fine_tune"]) <= 2:
            best = "both"

        return {"scores": scores, "recommendation": best}
```

**👹 Boss Lesson:** Fine-tuning changes the model. RAG changes the context. Most production systems need both.

---

## Dungeon 2 — Dataset Preparation

**Day 2 | Concept:** Fine-tuning quality is 90% data quality. Garbage in, garbage out — with training data, this is literal.

**Dataset Format (ChatML):**
```json
{"messages": [
  {"role": "system", "content": "You are a helpful assistant."},
  {"role": "user", "content": "What is RAG?"},
  {"role": "assistant", "content": "RAG stands for Retrieval-Augmented Generation..."}
]}
```

**Data Quality Rules:**
1. **No duplicates** — Deduplicate aggressively
2. **No contradictions** — Same Q should not have different A
3. **Balanced distribution** — Don't over-represent one topic
4. **Correct format** — Every example must be parseable
5. **Edge cases** — Include refusals, edge cases, hard questions
6. **Human verified** — Never use purely synthetic data without review

**🗡️ Dungeon Mastery:** Take 20 raw Q&A pairs. Clean them: remove duplicates, fix formatting, balance categories, add 3 edge cases. Output in ChatML format.

**💻 Code — Dataset Tools:**
```python
import json
import hashlib

class FineTuneDataset:
    def __init__(self):
        self.examples = []

    def add_example(self, system: str, user: str, assistant: str):
        self.examples.append({
            "messages": [
                {"role": "system", "content": system},
                {"role": "user", "content": user},
                {"role": "assistant", "content": assistant},
            ]
        })

    def add_completion(self, prompt: str, completion: str):
        """For base model fine-tuning (no chat template)."""
        self.examples.append({
            "prompt": prompt,
            "completion": completion,
        })

    def deduplicate(self):
        seen = set()
        deduped = []
        for ex in self.examples:
            key = json.dumps(ex, sort_keys=True)
            h = hashlib.md5(key.encode()).hexdigest()
            if h not in seen:
                seen.add(h)
                deduped.append(ex)
        self.examples = deduped
        return len(deduped)

    def balance(self, categories: dict[str, list[int]]):
        """categories = {"topic_name": [indices]}"""
        max_count = max(len(v) for v in categories.values())
        balanced = []
        for topic, indices in categories.items():
            count = 0
            for idx in indices:
                if count >= max_count:
                    break
                balanced.append(self.examples[idx])
                count += 1
        self.examples = balanced

    def train_test_split(self, test_ratio: float = 0.1):
        import random
        random.shuffle(self.examples)
        split = int(len(self.examples) * (1 - test_ratio))
        return self.examples[:split], self.examples[split:]

    def save(self, path: str):
        with open(path, "w", encoding="utf-8") as f:
            for ex in self.examples:
                f.write(json.dumps(ex, ensure_ascii=False) + "\n")

    @staticmethod
    def load(path: str) -> "FineTuneDataset":
        ds = FineTuneDataset()
        with open(path, "r", encoding="utf-8") as f:
            for line in f:
                if line.strip():
                    ds.examples.append(json.loads(line))
        return ds

    def stats(self) -> dict:
        if not self.examples:
            return {"count": 0}
        avg_len = sum(len(ex["messages"][-1]["content"]) for ex in self.examples) / len(self.examples)
        return {
            "count": len(self.examples),
            "avg_response_length": avg_len,
            "format": "chat" if "messages" in self.examples[0] else "completion",
        }
```

**👹 Boss Lesson:** Spend 80% of your fine-tuning effort on data. The model trains itself; you can't train bad data into good results.

---

## Dungeon 3 — LoRA & QLoRA

**Day 3 | Concept:** Full fine-tuning is expensive. LoRA (Low-Rank Adaptation) trains tiny adapter matrices instead of all weights. QLoRA adds quantization for even lower memory.

**Memory Comparison (Llama 3 8B):**
| Method | VRAM Required | Speed | Quality |
|--------|--------------|-------|---------|
| Full fine-tune | ~80 GB | Slow | Best |
| LoRA (rank=16) | ~24 GB | Fast | 95% of full |
| QLoRA (4-bit) | ~12 GB | Medium | 93% of full |

**LoRA Parameters:**
- **rank (r):** 8-64. Higher = more expressive, more memory
- **alpha:** Scaling factor. Typically 2x rank
- **target_modules:** Which layers to adapt (q_proj, v_proj, etc.)
- **dropout:** 0.05-0.1 for regularization

**🗡️ Dungeon Mastery:** Calculate memory requirements: QLoRA fine-tune a 7B model on a 20K dataset with rank=16. How much VRAM? How long at batch_size=4?

**💻 Code — LoRA Config:**
```python
# Using Hugging Face PEFT:
"""
from peft import LoraConfig, get_peft_model, TaskType

lora_config = LoraConfig(
    r=16,                    # Rank
    lora_alpha=32,           # Scaling (typically 2*r)
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type=TaskType.CAUSAL_LM,
)

# QLoRA (add 4-bit quantization):
from transformers import BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype="float16",
    bnb_4bit_use_double_quant=True,
)
"""

# Memory calculator:
class MemoryCalculator:
    @staticmethod
    def estimate_vram(model_size_b: float, method: str = "qlora", batch_size: int = 4, gradient_accum: int = 4) -> dict:
        """Estimate VRAM requirements for fine-tuning."""
        base = {
            "full": model_size_b * 4,  # Full fine-tune: 4x model size (AdamW)
            "lora": model_size_b * 2,  # LoRA: ~2x model size
            "qlora": model_size_b * 0.5 + 2,  # QLoRA: ~0.5x + overhead
        }

        method_memory = base.get(method, base["qlora"])

        # Batch processing overhead
        per_sample = model_size_b * 2 / batch_size  # Rough estimate
        total = method_memory + per_sample + 2  # +2GB overhead

        return {
            "method": method,
            "model_size_gb": model_size_b * 2,  # FP16 weights
            "training_vram_gb": round(total, 1),
            "batch_size": batch_size,
            "gradient_accumulation": gradient_accum,
            "effective_batch": batch_size * gradient_accum,
        }

# Example:
# calc = MemoryCalculator()
# print(calc.estimate_vram(7, "qlora", 4, 4))
```

**👹 Boss Lesson:** LoRA/QLoRA are the democratization of fine-tuning. A $30 Colab subscription can fine-tune a 7B model.

---

## Dungeon 4 — Training Recipes

**Day 4 | Concept:** Training hyperparameters dramatically affect fine-tuning quality. The right recipe = difference between amazing and broken.

**Key Hyperparameters:**
| Parameter | Range | Effect |
|-----------|-------|--------|
| Learning rate | 1e-5 to 5e-5 | Too high = loss spikes, too low = no learning |
| Epochs | 1-5 | More epochs = overfitting risk |
| Batch size | 4-64 | Larger = stable but more memory |
| Warmup steps | 10-100 | Prevents early loss spikes |
| Weight decay | 0.01-0.1 | Prevents overfitting |
| LR scheduler | cosine, linear | Cosine = better convergence |

**Overfitting Signs:**
- Training loss ↓ but eval loss ↑
- Model memorizes training examples
- Output becomes repetitive
- General knowledge decreases (catastrophic forgetting)

**🗡️ Dungeon Mastery:** Simulate a training run. Given loss curves (train loss = 0.1, eval loss = 2.3), diagnose overfitting. Propose fixes.

**💻 Code — Training Config:**
```python
from dataclasses import dataclass

@dataclass
class TrainingRecipe:
    """Recommended defaults for QLoRA fine-tuning."""
    learning_rate: float = 2e-4       # Common for LoRA
    num_epochs: int = 3
    batch_size: int = 4
    gradient_accumulation_steps: int = 4
    warmup_steps: int = 20
    weight_decay: float = 0.01
    logging_steps: int = 10
    save_steps: int = 200
    max_grad_norm: float = 0.3
    lr_scheduler: str = "cosine"
    optim: str = "paged_adamw_8bit"
    max_seq_length: int = 2048

    def effective_batch_size(self) -> int:
        return self.batch_size * self.gradient_accumulation_steps

    def total_steps(self, num_samples: int) -> int:
        samples_per_step = self.effective_batch_size()
        steps_per_epoch = num_samples / samples_per_step
        return int(steps_per_epoch * self.num_epochs)

    def to_dict(self) -> dict:
        return {k: str(v) if isinstance(v, float) else v
                for k, v in self.__dict__.items()}


# Using Hugging Face Trainer:
"""
from transformers import TrainingArguments

args = TrainingArguments(
    output_dir="./lora-model",
    num_train_epochs=recipe.num_epochs,
    per_device_train_batch_size=recipe.batch_size,
    gradient_accumulation_steps=recipe.gradient_accumulation_steps,
    warmup_steps=recipe.warmup_steps,
    learning_rate=recipe.learning_rate,
    weight_decay=recipe.weight_decay,
    logging_steps=recipe.logging_steps,
    save_steps=recipe.save_steps,
    max_grad_norm=recipe.max_grad_norm,
    lr_scheduler_type=recipe.lr_scheduler,
    optim=recipe.optim,
    fp16=True,
    report_to="none",
)
"""
```

**👹 Boss Lesson:** Start with known-good recipes. Tuning hyperparameters from scratch takes weeks of compute. Learn from community benchmarks.

---

## Dungeon 5 — Evaluating Fine-Tuned Models

**Day 5 | Concept:** A fine-tuned model that scores well on the training task but lost general capabilities is a FAILURE.

**Evaluation Axes:**
| Axis | What to Test | Method |
|------|-------------|--------|
| Task performance | Does it perform the trained task? | Eval dataset |
| General knowledge | Did it forget what it knew? | MMLU, HellaSwag |
| Format compliance | Does it output the right format? | Format validator |
| Hallucination rate | Does it make up facts? | Factuality check |
| Toxicity change | Did fine-tuning make it more toxic? | Toxicity benchmark |

**Catastrophic Forgetting:**
```python
# Compare pre vs post fine-tuning:
# Pre-fine-tune: "What is the capital of France?" → "Paris" ✅
# Post-fine-tune: "What is the capital of France?" → "I can help format JSON data..."
# This is forgetting. You need to mix general data into training.
```

**🗡️ Dungeon Mastery:** Fine-tune a model (simulate) on a specific format task. Test it on 5 general knowledge questions. If it forgot 2+, add 30% general data and retrain.

**💻 Code — Model Evaluation:**
```python
class FineTuneEvaluator:
    def __init__(self, base_model_fn, fine_tuned_fn):
        self.base_model = base_model_fn
        self.fine_tuned = fine_tuned_fn

    def evaluate_task_performance(self, eval_dataset: list[dict]) -> dict:
        """Test fine-tuned model on target task."""
        correct = 0
        for example in eval_dataset:
            response = self.fine_tuned(example["input"])
            if example["expected"].strip().lower() in response.strip().lower():
                correct += 1
        accuracy = correct / len(eval_dataset)
        return {"task_accuracy": accuracy, "total": len(eval_dataset), "correct": correct}

    def evaluate_general_knowledge(self, benchmark: list[dict]) -> dict:
        """Compare base vs fine-tuned on general knowledge."""
        results = {"base": [], "fine_tuned": []}
        for example in benchmark:
            base_answer = self.base_model(example["question"])
            ft_answer = self.fine_tuned(example["question"])

            base_correct = example["answer"].strip().lower() in base_answer.strip().lower()
            ft_correct = example["answer"].strip().lower() in ft_answer.strip().lower()

            results["base"].append(base_correct)
            results["fine_tuned"].append(ft_correct)

        return {
            "base_accuracy": sum(results["base"]) / len(results["base"]),
            "fine_tuned_accuracy": sum(results["fine_tuned"]) / len(results["fine_tuned"]),
            "forgetting_score": max(0, sum(results["base"]) / len(results["base"]) - sum(results["fine_tuned"]) / len(results["fine_tuned"])),
        }

    def compare(self, eval_dataset: list[dict], benchmark: list[dict]) -> dict:
        task = self.evaluate_task_performance(eval_dataset)
        general = self.evaluate_general_knowledge(benchmark)
        return {
            "task_performance": task,
            "general_knowledge": general,
            "verdict": "PASS" if task["task_accuracy"] >= 0.8 and general.get("forgetting_score", 0) < 0.2 else "REVIEW",
        }
```

**👹 Boss Lesson:** Always evaluate a fine-tuned model on BOTH the target task AND general capability. Task-only eval hides catastrophic forgetting.

---

## Dungeon 6 — Model Distillation

**Day 6 | Concept:** Take a large, expensive model and train a smaller one to mimic it. Distillation = compression without catastrophic quality loss.

**How Distillation Works:**
```
Large Teacher (GPT-4) generates outputs → Small Student (Llama 3B) trains on those outputs

Key insight: Student learns from teacher's PROBABILITY DISTRIBUTION, not just its answers.
The "soft targets" contain information about why the teacher chose that answer.
```

**When to Distill:**
- Serving costs too high
- Latency too high
- Need local/offline model
- Rate limits on API model

**🗡️ Dungeon Mastery:** Take a teacher model (simulate: GPT-4 responses) and generate training data for a student. Create 20 Q&A pairs. The student should learn the teacher's style, not just its answers.

**💻 Code — Distillation Tools:**
```python
class ModelDistiller:
    def __init__(self, teacher_fn, student_model):
        self.teacher = teacher_fn
        self.student = student_model

    def generate_training_data(self, prompts: list[str], system_prompt: str = "") -> list[dict]:
        """Generate student training data from teacher outputs."""
        training_data = []
        for prompt in prompts:
            teacher_output = self.teacher(prompt, system_prompt=system_prompt)
            training_data.append({
                "messages": [
                    {"role": "system", "content": system_prompt} if system_prompt else None,
                    {"role": "user", "content": prompt},
                    {"role": "assistant", "content": teacher_output},
                ]
            })
        return [ex for ex in training_data if ex["messages"][0] is not None]


class DistillationPipeline:
    """End-to-end distillation workflow."""

    @staticmethod
    def create_curriculum(source_domain: str, target_domain: str, num_examples: int = 1000) -> list[str]:
        """Generate a curriculum of prompts from easy to hard."""
        # In practice: generate prompts across difficulty levels
        return [f"Example {i}: {source_domain} → {target_domain}" for i in range(num_examples)]

    @staticmethod
    def quality_check(teacher_output: str, student_output: str) -> dict:
        """Compare teacher vs student output quality."""
        # Compare length, format, key concepts
        teacher_len = len(teacher_output)
        student_len = len(student_output)
        length_ratio = min(teacher_len, student_len) / max(teacher_len, student_len) if max(teacher_len, student_len) > 0 else 0
        return {
            "length_match": length_ratio,
            "teacher_length": teacher_len,
            "student_length": student_len,
        }
```

**👹 Boss Lesson:** Distillation is how you deploy GPT-4 quality at GPT-3.5 prices. But it never surpasses the teacher — it only approximates.

---

## Dungeon 7 — Deployment of Fine-Tuned Models

**Day 7 | Concept:** A fine-tuned model in a notebook is a science experiment. A deployed model is an engineering product.

**Deployment Options:**
| Platform | Cost | Latency | Ease |
|----------|------|---------|------|
| Hugging Face Inference Endpoints | $$$ | 200ms | Easy |
| vLLM + TGI | $ (GPU) | 100ms | Medium |
| Ollama (local) | Free | 500ms | Easy |
| Llama.cpp (CPU) | Free | 1000ms | Hard |
| Modal / RunPod | $$ | 100ms | Medium |

**Production Checklist:**
- [ ] Model artifact saved and versioned
- [ ] Inference server configured (max tokens, temperature, stop sequences)
- [ ] Load testing done (RPS, latency p95)
- [ ] Monitoring: request count, latency, error rate
- [ ] Fallback: if fine-tuned model fails, fall back to base model
- [ ] A/B test: compare fine-tuned vs base model on live traffic

**🗡️ Dungeon Mastery:** Write a deployment script for a fine-tuned LoRA model. Include: load base model, apply LoRA weights, start inference server, health check.

**💻 Code — Model Deployment:**
```python
# Server setup using vLLM:
"""
# save_model.py
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

base_model_name = "meta-llama/Llama-3.1-8B"
lora_path = "./lora-checkpoint"

# Merge LoRA weights with base model
model = AutoModelForCausalLM.from_pretrained(
    base_model_name,
    torch_dtype=torch.float16,
    device_map="auto",
)
tokenizer = AutoTokenizer.from_pretrained(base_model_name)

model = PeftModel.from_pretrained(model, lora_path)
model = model.merge_and_unload()
model.save_pretrained("./merged-model")
tokenizer.save_pretrained("./merged-model")
"""

# Inference client:
class FineTunedModelClient:
    def __init__(self, endpoint_url: str, api_key: str = ""):
        self.endpoint = endpoint_url
        self.headers = {"Authorization": f"Bearer {api_key}"} if api_key else {}

    def generate(self, prompt: str, max_tokens: int = 512, temperature: float = 0.1) -> str:
        import requests
        payload = {
            "inputs": prompt,
            "parameters": {
                "max_new_tokens": max_tokens,
                "temperature": temperature,
                "do_sample": temperature > 0,
            },
        }
        resp = requests.post(
            f"{self.endpoint}/generate",
            json=payload,
            headers=self.headers,
        )
        return resp.json().get("generated_text", "")

    def health_check(self) -> bool:
        import requests
        try:
            resp = requests.get(f"{self.endpoint}/health", timeout=5)
            return resp.status_code == 200
        except:
            return False


# Simple benchmark:
class ModelBenchmark:
    def __init__(self, client: FineTunedModelClient):
        self.client = client

    def run(self, prompts: list[str], n_warmup: int = 3) -> dict:
        import time
        # Warmup
        for p in prompts[:n_warmup]:
            self.client.generate(p)

        # Benchmark
        latencies = []
        for p in prompts:
            start = time.time()
            self.client.generate(p)
            latencies.append((time.time() - start) * 1000)

        return {
            "avg_latency_ms": sum(latencies) / len(latencies),
            "p50_ms": sorted(latencies)[len(latencies) // 2],
            "p95_ms": sorted(latencies)[int(len(latencies) * 0.95)],
            "min_ms": min(latencies),
            "max_ms": max(latencies),
            "total_requests": len(latencies),
        }
```

**👹 Boss Lesson:** A fine-tuned model that's not deployed is just a flex. The real engineering is in serving it reliably under load.

---

## ⚔️ BOSS BATTLE: The Fine-Tuning Workshop

**Objective:** Design a complete fine-tuning pipeline that:
1. Creates a curated dataset of 100+ examples in ChatML format
2. Configures LoRA with appropriate hyperparameters
3. Trains the model with proper recipes (learning rate, epochs, warmup)
4. Evaluates on BOTH task accuracy and general knowledge retention
5. Saves a production-ready merged model
6. Deploys with a simple inference server

**🗡️ Victory Conditions:**
- Dataset is clean, deduplicated, and balanced
- Training recipe is justified (not random defaults)
- Eval shows ≥ 85% task accuracy
- Forgetting score < 15% (general knowledge preserved)
- Deployment script is production-ready
- Benchmark shows acceptable latency

**🏆 Rewards:**
- +500 XP
- A- Rank → A Rank
- Skill Unlock: **Forge Master** — Fine-tuning projects take half the iterations
- Title: **The Sculptor**

---

*"A base model is clay. Fine-tuning is the sculptor's hand. The quality depends on the sculptor, not the clay."*
