# Phase 7: Fine-Tuning & LLMOps

**The Problem**: Base models don't know YOUR domain — legal jargon, medical terminology, company-specific formats. Prompting and RAG get you 80% there, but fine-tuning closes the gap. You also need to version, evaluate, and deploy fine-tuned models systematically.

**What You'll Build**: A fine-tuning pipeline that takes a base model and adapts it to your domain, with data management, training, evaluation, and deployment — exactly how companies like Cohere and Mistral customize models for enterprise clients.

**Difficulty**: Hard | **Time**: ~12 hours

## The Build Path

```
1. PROBLEM: "Base models don't understand my domain's language/format"
2. BUILD RAW: Understand when to fine-tune vs prompt vs RAG
3. BUILD RAW: LoRA/QLoRA — parameter-efficient fine-tuning from scratch
4. ADD TOOLS: Unsloth, Hugging Face TRL — production training libraries
5. PRODUCTIONIZE: Synthetic data generation for training data
6. PRODUCTIONIZE: Quantization (AWQ, GPTQ, FP8) for deployment
7. PRODUCTIONIZE: Model merging (merge fine-tuned weights with base)
8. EVALUATE: Before/after comparison on domain-specific benchmarks
9. DEBUG: Overfitting, catastrophic forgetting, data leakage
```

## Key Concepts

- When to fine-tune vs RAG vs prompt engineering (critical decision framework)
- LoRA/QLoRA — train adapters, not full models (costs $10, not $10K)
- Unsloth — 2x faster training, 50% less memory
- Synthetic data generation — create training data from LLMs
- Quantization — AWQ, GPTQ, GGUF, FP8
- Model merging — combine multiple fine-tuned adapters
- Evaluation — domain-specific benchmarks, before/after comparison

## The Project: Fine-Tune a Domain Expert

Fine-tune Llama 3.2 8B on a domain of your choice (legal, medical, code, customer support):
- Generate synthetic training data using GPT-4o
- Train with LoRA/QLoRA using Unsloth
- Evaluate against base model on domain-specific benchmarks
- Quantize and deploy with vLLM
- Compare: base model vs fine-tuned vs RAG-augmented base

## 🔴 Senior: Fine-Tuning is EXPENSIVE

A full fine-tuning run costs $10K+. LoRA costs $10. Always try LoRA first. Only do full fine-tuning when LoRA isn't enough. And always ask: "Can I solve this with prompting + RAG cheaper?"

## Gate Check

- [ ] Can explain when to fine-tune vs RAG vs prompt engineering
- [ ] LoRA fine-tuning completes in under 2 hours on a single GPU
- [ ] Fine-tuned model outperforms base model on domain benchmarks
- [ ] Quantized model runs 2x faster than unquantized
- [ ] Synthetic data pipeline generates valid, diverse training examples
