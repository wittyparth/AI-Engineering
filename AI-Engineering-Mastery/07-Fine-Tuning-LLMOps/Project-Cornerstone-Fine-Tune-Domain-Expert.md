# Cornerstone Project: Fine-Tune a Domain Expert

**Time**: 4-5 days  **Difficulty**: Hard  **Portfolio**: Yes

## The Problem

Pick a domain where a general-purpose LLM underperforms and fine-tune a model to be a domain expert. Options:
- **Technical documentation expert** (reads docs, answers how-to questions)
- **Code reviewer** (reviews PRs with domain-specific standards)
- **Medical Q&A** (answers medical questions from a knowledge base)
- **Legal document analyzer** (extracts clauses, identifies risks)
- **Your own domain** (whatever you're an expert in)

## Requirements

### Must Have
- [ ] Domain dataset (1000+ high-quality examples)
- [ ] LoRA fine-tuning with tracked experiment (WandB)
- [ ] Base model vs fine-tuned model comparison
- [ ] Quantitative evaluation (domain-specific metrics)
- [ ] Qualitative evaluation (blind A/B test with 20+ human raters)
- [ ] Quantized model export (GGUF or GPTQ)
- [ ] Model serving endpoint (FastAPI + vLLM)

### Should Have
- [ ] Synthetic data generation for hard cases
- [ ] Data filtering pipeline
- [ ] Training failure analysis
- [ ] Model merged with another expert model
- [ ] A/B testing in production

### Nice to Have
- [ ] RLHF/DPO alignment
- [ ] LORA adapter switching (swap domains without restart)
- [ ] Test-time compute (CoT for hard cases)

## Architecture

```
Documents → Synthetic Data Gen → Dataset → LoRA Training → Model Eval
                ↓                            ↓               ↓
           Data Filter                 WandB Logs      Comparison Report
                                                          ↓
                                                     Quantize → Deploy
```

## 🔴 Senior: Fine-Tuning Checklist

```
Before training:
  [ ] Is RAG insufficient for this task?
  [ ] Do I have 500+ high-quality examples?
  [ ] Is my evaluation methodology sound (no data leakage)?
  [ ] Do I have a clear "this is better" threshold?

During training:
  [ ] Loss is decreasing (not spiking/plateauing)
  [ ] Gradient norms are stable (< 10)
  [ ] Not overfitting (val loss < train loss, but not by much)

After training:
  [ ] Quantized model quality is within 2% of full precision
  [ ] Blind A/B test shows clear preference for fine-tuned
  [ ] Edge cases are handled (not worse than base model)
  [ ] Deployment infrastructure works end-to-end
```

## Evaluation Targets

| Metric | Target |
|--------|--------|
| Domain accuracy | >90% |
| Format adherence | >95% |
| Hallucination rate | <5% |
| Blind A/B win rate | >70% against base model |
| Inference latency | <2s (quantized) |
| Cost per query | <$0.01 (self-hosted quantized) |

## Design Decisions

1. **Base model selection**: Llama 3.2 8B vs Mistral 7B vs Qwen 7B?
2. **LoRA rank**: r=8, 16, 32, or 64?
3. **Data creation**: Synthetic only, human-only, or mixed?
4. **Evaluation method**: LLM-as-judge, human eval, or automated metrics?
5. **Quantization format**: GGUF (CPU) vs GPTQ (GPU) vs AWQ (best quality)?
6. **Serving framework**: vLLM, Ollama, TGI, or custom?

## Code Review Rubric

- Dataset quality is verified (sample inspection)
- Training is reproducible (seed, config)
- Evaluation methodology prevents data leakage
- Quantized model works and is within 2% quality
- Serving endpoint is production-ready (health, metrics, auth)
- Cost and performance are documented

## Reflection

1. Did fine-tuning actually improve quality? By how much?
2. What portion of the improvement came from data quality vs training?
3. Would RAG have achieved similar results with less effort?
4. What failure modes did you discover?
5. How would you maintain this model over time (as domain knowledge evolves)?
