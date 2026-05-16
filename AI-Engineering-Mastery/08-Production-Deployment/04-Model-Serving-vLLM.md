# Model Serving (vLLM)

## Why vLLM

| Feature | Benefit |
|---------|---------|
| PagedAttention | Near-zero memory waste from KV cache |
| Continuous batching | 10-20x higher throughput than naive serving |
| Quantization support | AWQ, GPTQ, FP8 |
| OpenAI-compatible API | Drop-in replacement for GPT API |
| Tensor parallelism | Multi-GPU inference |

## Basic vLLM Server

```python
from vllm import LLM, SamplingParams
from vllm.engine.arg_utils import AsyncEngineArgs
from vllm.entrypoints.openai.api_server import run_server

# Command line
# python -m vllm.entrypoints.openai.api_server \
#     --model meta-llama/Llama-3.2-8B \
#     --quantization awq \
#     --dtype half \
#     --max-model-len 8192 \
#     --gpu-memory-utilization 0.9 \
#     --tensor-parallel-size 2  # 2 GPUs
```

## Programmatic Usage

```python
from vllm import AsyncLLMEngine, AsyncEngineArgs, SamplingParams

class VLLMServer:
    def __init__(self, model_path: str):
        self.engine = AsyncLLMEngine.from_engine_args(
            AsyncEngineArgs(
                model=model_path,
                max_model_len=8192,
                gpu_memory_utilization=0.9,
                quantization="awq",
            )
        )

    async def generate(self, prompt: str, **kwargs) -> str:
        params = SamplingParams(
            temperature=kwargs.get("temperature", 0.7),
            max_tokens=kwargs.get("max_tokens", 1024),
            top_p=kwargs.get("top_p", 0.95),
        )

        full_response = ""
        async for result in self.engine.generate(prompt, params):
            full_response = result.outputs[0].text

        return full_response

    async def generate_stream(self, prompt: str, **kwargs):
        params = SamplingParams(
            temperature=0.7, max_tokens=1024,
        )
        async for result in self.engine.generate(prompt, params):
            yield result.outputs[0].text
```

## Deployment Architecture

```
                         ┌──────────────┐
                         │  FastAPI App  │
                         └──────┬───────┘
                                │
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
        ┌────────────┐  ┌────────────┐  ┌────────────┐
        │ vLLM Node1 │  │ vLLM Node2 │  │ vLLM Node3 │
        │ (Llama 8B) │  │ (Mistral)  │  │ (Qwen)     │
        └────────────┘  └────────────┘  └────────────┘
```

## 🔴 Senior: vLLM Configuration

| Setting | Recommendation | Why |
|---------|---------------|-----|
| max-model-len | 8192 | Balances memory vs context length |
| gpu-memory-utilization | 0.85-0.90 | Leave room for fragmentation |
| tensor-parallel-size | Number of GPUs | Scales almost linearly |
| quantization | AWQ (4-bit) | Best quality-to-speed ratio |
| max-num-seqs | 256 | Default is fine for most cases |
| enable-prefix-caching | True | Saves compute on system prompts |
