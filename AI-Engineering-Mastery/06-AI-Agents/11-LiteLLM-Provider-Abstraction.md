# LiteLLM: Provider Abstraction

## The Problem

Every LLM provider has a different API:
```python
# OpenAI
openai.ChatCompletion.create(model="gpt-4o", messages=[...])

# Anthropic
anthropic.messages.create(model="claude-4", messages=[...])

# Google
genai.GenerativeModel("gemini-2.5-pro").generate_content(...)

# Local
ollama.chat(model="llama3.2", messages=[...])
```

## LiteLLM Solution

```python
from litellm import completion

# Same API for ALL providers
response = completion(
    model="gpt-4o",  # or "claude-4", "gemini/gemini-2.5-pro", "ollama/llama3.2"
    messages=[{"role": "user", "content": "Hello"}],
    temperature=0.7,
    max_tokens=1000,
)
```

## Provider Routing

```python
from litellm import Router

router = Router(model_list=[
    {
        "model_name": "gpt-4o",
        "litellm_params": {"model": "openai/gpt-4o", "api_key": os.getenv("OPENAI_API_KEY")},
        "tpm": 100000,  # Tokens per minute limit
        "rpm": 1000,    # Requests per minute limit
    },
    {
        "model_name": "claude-4",
        "litellm_params": {"model": "anthropic/claude-sonnet-4-20250514", "api_key": os.getenv("ANTHROPIC_API_KEY")},
        "tpm": 50000,
        "rpm": 500,
    },
    {
        "model_name": "local-llama",
        "litellm_params": {"model": "ollama/llama3.2", "api_key": "fake"},
        "tpm": 999999,
        "rpm": 9999,
    },
])

# Smart routing with fallback
response = await router.acompletion(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
    fallbacks=["claude-4", "local-llama"],  # Auto fallback on failure
)

# Cost tracking built-in
print(f"Cost: ${response._cost:.6f}")
print(f"Provider: {response._response_ms}")
```

## LiteLLM Proxy (Production)

Run a proxy server that handles all providers:

```bash
litellm --model gpt-4o --model claude-4 --model ollama/llama3.2 \
    --port 8000 --detailed_debug
```

```python
# Your code talks to ONE endpoint
import openai
client = openai.OpenAI(base_url="http://localhost:8000")
response = client.chat.completions.create(
    model="gpt-4o",  # LiteLLM routes to the actual provider
    messages=[...],
)
```

## Pydantic AI + LiteLLM

Pydantic AI can use LiteLLM as its model backend:

```python
from pydantic_ai import Agent
from pydantic_ai.models.litellm import LiteLLMModel

agent = Agent(
    LiteLLMModel("gpt-4o"),
    system_prompt="You are helpful.",
)

# You can swap models by changing one string
# "gpt-4o" → "claude-sonnet-4" → "ollama/llama3.2"
```

## 🔴 Senior: Framework Composition

The pro stack in 2026:

```
Pydantic AI (agent framework)
    ↓
LiteLLM (provider abstraction + routing + cost tracking)
    ↓
OpenAI / Anthropic / Ollama / vLLM (actual models)
```

Each layer handles ONE concern:
- Pydantic AI: agent logic, tools, structured output
- LiteLLM: provider switching, rate limits, failover, cost
- Raw APIs/models: actual inference

You can swap ANY layer without changing the others.

## Drill: LiteLLM Gateway

Build a FastAPI endpoint that uses LiteLLM Router to:
1. Route simple queries to GPT-4o-mini ($0.15/M tokens)
2. Route complex queries to GPT-4o ($2.50/M tokens)
3. Fallback to Claude 4 if GPT-4o fails
4. Log cost per query
5. Report which model was used per query
