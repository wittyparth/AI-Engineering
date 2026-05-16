# Llama Stack: Standardized Model Serving

## The Problem

Every model server has a different API surface:
- vLLM: OpenAI-compatible chat + completions endpoints
- TGI: HuggingFace-specific extensions
- Ollama: Custom API with model management
- AWS Bedrock: Boto3 SDK with different auth
- Together / Fireworks: REST APIs with different rate limits

Switching between them means rewriting integration code.

## Llama Stack Solution

Llama Stack provides a **standardized, OpenAI-compatible API** across any inference backend — local, cloud, or on-premises. Think of it as a universal adapter layer for model serving.

```
Your Application
      ↓  (OpenAI-compatible API)
Llama Stack Server
      ↓  (provider abstraction)
vLLM | TGI | Ollama | Bedrock | Fireworks | Together
```

## Quick Start

```bash
# Install
pip install llama-stack llama-stack-client

# Start server with vLLM
llama stack run --distribution meta-reference \
    --model meta-llama/Llama-3.2-8B-Instruct \
    --port 8080
```

## Configuration

```yaml
# config.yaml
models:
  - model_id: "Llama-3.2-8B-Instruct"
    provider: "vllm"
    provider_config:
      model: "meta-llama/Llama-3.2-8B-Instruct"
      tensor_parallel_size: 1
      gpu_memory_utilization: 0.9
      max_model_len: 8192

  - model_id: "Llama-3.2-90B-Vision"
    provider: "fireworks"
    provider_config:
      api_key: ${FIREWORKS_API_KEY}

inference:
  port: 8080
  batch_size: 8
  max_retries: 3
```

## Client Usage

```python
from llama_stack_client import LlamaStackClient

# The SAME client works for ANY backend
client = LlamaStackClient(base_url="http://localhost:8080")

# Chat completion (OpenAI-compatible)
response = client.chat_completions.create(
    model_id="Llama-3.2-8B-Instruct",
    messages=[{"role": "user", "content": "Hello"}],
    temperature=0.7,
    max_tokens=500,
)

# Streaming
stream = client.chat_completions.create(
    model_id="Llama-3.2-8B-Instruct",
    messages=[{"role": "user", "content": "Tell me a story"}],
    stream=True,
)
for chunk in stream:
    print(chunk.delta.text, end="")

# Batch inference
responses = client.batch_inference(
    model_id="Llama-3.2-8B-Instruct",
    input_batch=[
        {"messages": [{"role": "user", "content": "Q1"}]},
        {"messages": [{"role": "user", "content": "Q2"}]},
    ],
)
```

## Multi-Provider Routing

```yaml
# multi-provider.yaml
models:
  - model_id: "llama-chat"
    provider: "vllm"          # Primary: local GPU
    provider_config:
      model: "meta-llama/Llama-3.2-8B-Instruct"
      tensor_parallel_size: 2

  - model_id: "llama-chat-fallback"
    provider: "fireworks"      # Fallback: cloud API
    provider_config:
      api_key: ${FIREWORKS_API_KEY}

routing:
  strategy: "fallback"         # Try primary, fallback on failure
  primary: "llama-chat"
  fallback: "llama-chat-fallback"
  timeout_ms: 30000
```

## Llama Stack vs vLLM vs TGI

| Feature | Llama Stack | vLLM | TGI |
|---------|------------|------|-----|
| API standard | OpenAI-compatible + extensions | OpenAI-compatible | OpenAI-compatible |
| Multi-provider | ✅ vLLM, TGI, Ollama, Bedrock, Fireworks | ❌ vLLM only | ❌ TGI only |
| Multi-model routing | ✅ Built-in | ❌ Manual | ❌ Manual |
| Batch inference | ✅ Native | ✅ Via API | ✅ Via API |
| Quantization | ✅ FP8, AWQ, GPTQ | ✅ FP8, AWQ, GPTQ | ✅ AWQ, GPTQ |
| Vision models | ✅ Native support | ⚠️ LLaVA only | ✅ IDEFICS, LLaVA |
| Tool calling | ✅ Standardized | ✅ OpenAI format | ⚠️ Limited |
| Ease of setup | Medium (more config) | Low | Low |

## Production Deployment

```dockerfile
# Dockerfile.llama-stack
FROM nvidia/cuda:12.4-runtime-ubuntu22.04

RUN pip install llama-stack llama-stack-client vllm

COPY config.yaml /etc/llama-stack/config.yaml

EXPOSE 8080

CMD ["llama", "stack", "run", \
     "--distribution", "meta-reference", \
     "--config", "/etc/llama-stack/config.yaml"]
```

```yaml
# docker-compose.yml
services:
  llama-stack:
    build:
      context: .
      dockerfile: Dockerfile.llama-stack
    ports:
      - "8080:8080"
    volumes:
      - ~/.cache/huggingface:/root/.cache/huggingface
      - ./config.yaml:/etc/llama-stack/config.yaml
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 2
              capabilities: [gpu]
```

## 🔴 Senior: When to Use Llama Stack

**Use it when:**
- You need to serve models across multiple backends (local GPU + cloud fallback)
- You want vendor independence at the serving layer
- You're building a multi-model platform (chat, vision, code models)
- You need to switch providers without touching application code

**Don't use it when:**
- You serve a single model on a single backend (vLLM is simpler)
- You need maximum per-request throughput (vLLM direct is faster)
- You're prototyping locally (Ollama is simpler)
- Your team lacks DevOps/MLOps support (more moving parts)

## Drill: Multi-Backend Model Serving

1. Set up Llama Stack with vLLM backend serving Llama 3.2 8B
2. Add a Fireworks/Together fallback for the same model
3. Run a batch of 100 requests, measure:
   - Average latency (should be ~same)
   - P95 latency
   - Throughput (requests/sec)
   - Failover time when vLLM is killed
4. Compare: direct vLLM vs Llama Stack on vLLM (latency overhead)
5. Swap the backend to TGI — what changes in your client code?
