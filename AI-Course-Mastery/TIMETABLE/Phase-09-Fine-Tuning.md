# Phase 9 — Fine-Tuning & LLMOps (Days 107-117)

**Goal:** Master fine-tuning fundamentals, LoRA/PEFT, dataset preparation, training with Unsloth, function calling fine-tuning, model registry — build a fine-tuned custom model.
**Files:** 9 concept + 1 project (10 files total)
**Total days:** 11 study days + 1 phase-end review

---

### Day 107 — Fine-Tuning Fundamentals

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + Fine-Tuning Fundamentals | `00-Module-Overview.md`, `01-Fine-Tuning-Fundamentals.md` |
| 🌤️ Afternoon | Code: compare base model vs fine-tuned on a sample | Load a base model, analyze outputs, identify areas for improvement |
| 🌙 Evening | Flashback | Write: "When should you fine-tune vs prompt-engineer vs RAG?" |

**Key Concepts:** Full fine-tuning vs PEFT, when to fine-tune, fine-tuning risks (catastrophic forgetting, overfitting)
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Decision framework (FT vs RAG vs prompting), FT failure modes

---

### Day 108 — LoRA & PEFT (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | LoRA & PEFT deep dive | `02-LoRA-and-PEFT.md` (theory: rank, alpha, adapter placement) |
| 🌤️ Afternoon | Code: implement LoRA fine-tuning | Load base model, apply LoRA, train on a small dataset, compare outputs |
| 🌙 Evening | Flashback | Write: "How does LoRA work technically? What do rank and alpha control?" |

**Key Concepts:** Low-rank adaptation, adapter matrices, rank selection, LoRA vs QLoRA, adapter merging
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** LoRA hyperparameters, rank-alpha relationship, adapter placement strategy

---

### Day 109 — Dataset Preparation (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Dataset Preparation | `03-Dataset-Preparation.md` (format, quality, augmentation) |
| 🌤️ Afternoon | Code: build dataset prep pipeline | Load raw data → clean → format → split → validate → push to hub |
| 🌙 Evening | Flashback | Write: "What makes a high-quality fine-tuning dataset? What ruins one?" |

**Key Concepts:** Dataset formats (ChatML, ShareGPT, Alpaca), quality filtering, deduplication, train/val/test split
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Dataset quality metrics, format conversion, data leakage prevention

---

### Day 110 — Training with Unsloth & TRL

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Training with Unsloth & TRL | `04-Training-with-Unsloth-TRL.md` |
| 🌤️ Afternoon | Code: run a full training job | Unsloth setup, SFTTrainer, training loop, checkpointing |
| 🌙 Evening | Flashback | Write: "Walk through the training process from data to saved adapter" |

**Key Concepts:** Unsloth optimizations, SFTTrainer, training args, loss monitoring, checkpointing
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Training hyperparameters, GPU memory management, training monitoring

---

### Day 111 — Evaluation & Comparison

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Evaluation & Comparison | `05-Evaluation-and-Comparison.md` |
| 🌤️ Afternoon | Code: compare base vs fine-tuned systematically | Side-by-side evals on a test set, quantitative comparison |
| 🌙 Evening | Flashback | Write: "How do you prove fine-tuning actually improved the model?" |

**Key Concepts:** Pre/post FT evaluation, task-specific metrics, human evaluation, cost-benefit analysis
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** FT eval methodology, when FT fails to improve, cost of FT vs benefit

---

### Day 112 — Fine-Tuning for Function Calling

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Fine-Tuning for Function Calling | `06-Fine-Tuning-for-Function-Calling.md` |
| 🌤️ Afternoon | Code: FT a model for tool use | Prepare tool-use dataset, fine-tune, evaluate tool call accuracy |
| 🌙 Evening | Flashback | Write: "How is fine-tuning for function calling different from general FT?" |

**Key Concepts:** Function calling format, tool-use datasets, accuracy evaluation, structured output FT
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** FC-specific FT techniques, evaluation methodology for tool calls

---

### Day 113 — LLMOps & Model Registry

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | LLMOps & Model Registry | `07-LLMOps-Model-Registry.md` |
| 🌤️ Afternoon | Code: build model registry | Versioning, metadata tracking, deployment tagging, A/B deployment |
| 🌙 Evening | Flashback | Write: "What would an LLMOps platform need to support? Walk through the lifecycle." |

**Key Concepts:** Model registry, versioning, lineage tracking, model lifecycle management
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Model registry vs experiment tracker, model promotion workflow

---

### Day 114 — Deployment of Fine-Tuned Models

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Deployment of Fine-Tuned Models | `08-Deployment-of-Fine-Tuned-Models.md` |
| 🌤️ Afternoon | Code: deploy FT model with vLLM | Adapter merging, model serving, API endpoint for FT model |
| 🌙 Evening | Flashback | Write: "How do you deploy a fine-tuned model to production?" |

**Key Concepts:** Adapter merging, vLLM serving, GPU requirements, scaling FT models
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Merged vs unmerged adapter, quantization for deployment, cost of serving FT models

---

### Day 115-116 — Project: Custom Fine-Tuned Model (2 days)

**All days = 🏗️ Project** — `09-Project-Custom-Fine-Tuned-Model.md`

#### Day 115 — Project: Dataset + Training

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + create dataset | Spec → choose task → create/prepare dataset |
| 🌤️ Afternoon | Run fine-tuning job | Full training with Unsloth, monitor loss, save adapter |
| 🌙 Evening | Flashback + commit | Training results, loss curves → git commit |

**Checklist:** Spec read ☐ | Dataset prepared ☐ | Training completed ☐ | GitHub updated ☐

#### Day 116 — Project: Eval + Deployment + Gate Check

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Evaluate + compare | Base vs FT comparison with quantitative metrics |
| 🌤️ Afternoon | Deploy + Gate Check | 🎯 vLLM deployment, test API, docs, gate checks |
| 🌙 Evening | Final commit + reflection | Was the fine-tuning worth the effort? By how much did it improve? |

**🎯 Gate Check — Can you:**
- [ ] Prepare a high-quality fine-tuning dataset?
- [ ] Run LoRA fine-tuning with Unsloth?
- [ ] Evaluate and compare base vs fine-tuned model quantitatively?
- [ ] Fine-tune for function calling?
- [ ] Deploy a fine-tuned model with vLLM?
- [ ] Manage model versions in a registry?
- [ ] Explain when fine-tuning is the wrong approach?
- [ ] Calculate cost vs benefit of fine-tuning?

---

### Day 117 — Phase 9 Review

**🔄 Phase-End + Weekly Review combined**

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read ALL 10 files. Re-type LoRA training code from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 9. Re-read weak sections. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself. |

**🧪 Self-Test Questions:**
1. When should you fine-tune vs use RAG vs improve prompting?
2. How does LoRA work technically? What controls its capacity?
3. What makes a high-quality fine-tuning dataset?
4. Walk through the training process from data to saved model
5. How do you evaluate whether fine-tuning improved the model?
6. How is fine-tuning for function calling different?
7. What is a model registry and why do you need it?
8. How do you deploy a fine-tuned model to production?
9. What are the cost implications of fine-tuning and serving?
10. What are the biggest risks of fine-tuning and how do you mitigate them?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
