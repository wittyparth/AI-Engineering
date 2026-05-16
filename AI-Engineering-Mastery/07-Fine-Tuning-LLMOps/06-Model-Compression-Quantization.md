# Model Compression & Quantization

## Why Quantization

- GPT-4o 70B in FP16: 140GB GPU memory → Needs 2x A100 (80GB)
- GPT-4o 70B in INT4: 35GB GPU memory → Fits on 1x A100
- 4x less memory, similar quality, faster inference

## Quantization Formats

| Format | Bits | Memory Reduction | Quality Loss | Best For |
|--------|------|-----------------|-------------|----------|
| FP16 | 16 | 0% | None | Baseline |
| INT8 | 8 | 50% | <1% | Balanced |
| INT4 (GPTQ) | 4 | 75% | 1-3% | GPU inference |
| INT4 (GGUF) | 4 | 75% | 1-3% | CPU inference |
| NF4 (QLoRA) | 4 | 75% | 1-3% | Training |
| AWQ | 4 | 75% | 1-2% | Best quality 4-bit |
| 2-bit | 2 | 87% | 5-10% | Edge devices |

## Quantization with GPTQ

```python
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

quantize_config = BaseQuantizeConfig(
    bits=4,
    group_size=128,
    desc_act=False,
)

model = AutoGPTQForCausalLM.from_pretrained(
    "meta-llama/Llama-3.2-7B",
    quantize_config=quantize_config,
)

model.quantize(tokenizer, calibration_data)
model.save_quantized("./model-4bit-gptq")
```

## Quantization with GGUF (llama.cpp)

```bash
# Convert to GGUF format
python convert.py ./model/ --outfile model.gguf

# Quantize to Q4_K_M
./quantize model.gguf model-Q4_K_M.gguf q4_K_M
```

## Serving Quantized Models

```python
# vLLM with AWQ
from vllm import LLM, SamplingParams

model = LLM(
    model="casperhansen/llama-3.2-7b-awq",
    quantization="AWQ",
    dtype="half",
    max_model_len=8192,
)
```

## 🔴 Senior: When NOT to Quantize

| Scenario | Reason |
|----------|--------|
| Quality-critical applications | Every bit of precision matters |
| Fine-tuning (not inference) | Quantization degrades gradient quality |
| Few-shot sensitive tasks | Quantization can change few-shot behavior |
| Deterministic outputs needed | Quantized models have more variance |

For most production RAG/agents, INT4 is fine (1-3% quality loss for 75% cost savings).

## Drill: Quality Loss Measurement

1. Take a model in FP16
2. Quantize to INT4, INT8, and GGUF Q4_K_M
3. Run 100 prompts through all versions
4. Measure: quality score, latency, memory
5. Report: "At what quality loss threshold does it break my use case?"
