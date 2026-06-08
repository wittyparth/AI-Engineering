# API Integration Patterns — Talking to Every Provider

## 🎯 Purpose & Goals

> **🛑 STOP. Design problem:**
>
> You need to build a system that can use OpenAI, Anthropic, OR local Ollama models — switchable with ONE config change.
>
> The APIs are different:
> - OpenAI: `client.chat.completions.create(messages=[...])`
> - Anthropic: `client.messages.create(messages=[...], system="...")`
> - Ollama: `client.chat.completions.create()` (OpenAI-compatible, different base_url)
>
> **Sketch a design that abstracts over all three. What's your interface? What patterns would you use?**
>
> *Think about: Strategy Pattern? Adapter Pattern? Protocol classes?*

---

**By the end of this file, you will:**
- Write code that works with ALL major LLM providers
- Build a provider abstraction layer (swap providers with one line)
- Handle provider-specific quirks (system prompts, streaming, auth)
- Have a working multi-provider client

**⏱ Time Budget:** 2 hours

---

## 📖 Provider Comparison

> **🤔 YOUR TURN:** Before reading the code, list the key differences between OpenAI, Anthropic, and Ollama APIs:

| Aspect | OpenAI | Anthropic | Ollama |
|--------|--------|-----------|--------|
| System prompt | | | |
| Message format | | | |
| Streaming | | | |
| Auth | | | |
| Base URL | | | |

<details>
<summary>👀 Full comparison</summary>

| Aspect | OpenAI | Anthropic | Ollama |
|--------|--------|-----------|--------|
| System prompt | Part of messages list with role="system" | Separate `system` parameter | Same as OpenAI |
| Message format | `[{"role": "user", "content": "hi"}]` | `[{"role": "user", "content": "hi"}]` | Same as OpenAI |
| Streaming | `stream=True`, iterate chunks | `.messages.stream()`, `with stream as s:` | Same as OpenAI |
| Auth | API key in header | API key (x-api-key header) | No auth (localhost) |
| Base URL | `https://api.openai.com/v1` | `https://api.anthropic.com/v1` | `http://localhost:11434/v1` |

**Key insight:** Ollama is OpenAI-API compatible! You can use the `openai` Python package with a different `base_url`.
</details>

---

## 📖 Building the Provider Abstraction

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import AsyncGenerator, List, Optional
import os

# 🛑 PAUSE: Design the interface FIRST.
# What methods should EVERY provider have?
# What should the input/output look like?

@dataclass
class LLMResponse:
    """Standardized response from any provider"""
    content: str
    model: str
    provider: str
    prompt_tokens: int
    completion_tokens: int
    latency_ms: float
    
    @property
    def total_cost(self) -> float:
        """Calculate cost based on provider pricing"""
        pricing = {
            "openai": {"gpt-4o-mini": (0.15, 0.60), "gpt-4o": (2.50, 10.00)},
            "anthropic": {"claude-3-haiku": (0.25, 1.25), "claude-3-5-sonnet": (3.00, 15.00)},
            "ollama": {"default": (0.0, 0.0)},  # Free!
        }
        provider_pricing = pricing.get(self.provider, {})
        model_pricing = provider_pricing.get(self.model, provider_pricing.get("default", (0, 0)))
        
        input_cost = (self.prompt_tokens / 1_000_000) * model_pricing[0]
        output_cost = (self.completion_tokens / 1_000_000) * model_pricing[1]
        return round(input_cost + output_cost, 6)


# Now design the abstract interface:
class LLMProvider(ABC):
    """Abstract interface that ALL providers must implement"""
    
    @abstractmethod
    async def complete(self, messages: list[dict], temperature: float = 0.0, max_tokens: int = 1000) -> LLMResponse:
        """Send messages and get a complete response"""
        pass
    
    @abstractmethod
    async def stream(self, messages: list[dict], temperature: float = 0.0) -> AsyncGenerator[str, None]:
        """Send messages and stream the response token by token"""
        pass
    
    @property
    @abstractmethod
    def provider_name(self) -> str:
        pass


# 🛑 YOUR TURN: Implement the OpenAI provider.
# You know the API. Try to implement it before reading mine.

class OpenAIProvider(LLMProvider):
    """Write your implementation here"""
    pass


<details>
<summary>👀 Full implementation</summary>

```python
import time
from openai import AsyncOpenAI

class OpenAIProvider(LLMProvider):
    def __init__(self, model: str = "gpt-4o-mini"):
        self.client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.model = model
    
    @property
    def provider_name(self) -> str:
        return "openai"
    
    async def complete(self, messages: list[dict], temperature: float = 0.0, max_tokens: int = 1000) -> LLMResponse:
        start = time.time()
        
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            temperature=temperature,
            max_tokens=max_tokens,
        )
        
        latency = (time.time() - start) * 1000
        
        return LLMResponse(
            content=response.choices[0].message.content,
            model=self.model,
            provider="openai",
            prompt_tokens=response.usage.prompt_tokens,
            completion_tokens=response.usage.completion_tokens,
            latency_ms=latency,
        )
    
    async def stream(self, messages: list[dict], temperature: float = 0.0) -> AsyncGenerator[str, None]:
        stream = await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            temperature=temperature,
            stream=True,
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

class AnthropicProvider(LLMProvider):
    def __init__(self, model: str = "claude-3-haiku"):
        import anthropic
        self.client = anthropic.AsyncAnthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
        self.model = model
    
    @property
    def provider_name(self) -> str:
        return "anthropic"
    
    async def complete(self, messages: list[dict], temperature: float = 0.0, max_tokens: int = 1000) -> LLMResponse:
        start = time.time()
        
        # Anthropic separates system prompt
        system = ""
        filtered_messages = []
        for msg in messages:
            if msg["role"] == "system":
                system = msg["content"]
            else:
                filtered_messages.append(msg)
        
        response = await self.client.messages.create(
            model=self.model,
            system=system or None,
            messages=filtered_messages,
            max_tokens=max_tokens,
            temperature=temperature,
        )
        
        latency = (time.time() - start) * 1000
        
        return LLMResponse(
            content=response.content[0].text,
            model=self.model,
            provider="anthropic",
            prompt_tokens=response.usage.input_tokens,
            completion_tokens=response.usage.output_tokens,
            latency_ms=latency,
        )
    
    async def stream(self, messages: list[dict], temperature: float = 0.0) -> AsyncGenerator[str, None]:
        system = ""
        filtered = []
        for msg in messages:
            if msg["role"] == "system":
                system = msg["content"]
            else:
                filtered.append(msg)
        
        async with self.client.messages.stream(
            model=self.model,
            system=system or None,
            messages=filtered,
            max_tokens=1000,
            temperature=temperature,
        ) as stream:
            async for text in stream.text_stream:
                yield text

class OllamaProvider(LLMProvider):
    def __init__(self, model: str = "llama3.2"):
        # Ollama uses OpenAI-compatible API!
        from openai import AsyncOpenAI
        self.client = AsyncOpenAI(
            base_url="http://localhost:11434/v1",
            api_key="ollama",  # Required but not used
        )
        self.model = model
    
    @property
    def provider_name(self) -> str:
        return "ollama"
    
    async def complete(self, messages: list[dict], temperature: float = 0.0, max_tokens: int = 1000) -> LLMResponse:
        start = time.time()
        
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            temperature=temperature,
            max_tokens=max_tokens,
        )
        
        latency = (time.time() - start) * 1000
        
        return LLMResponse(
            content=response.choices[0].message.content,
            model=self.model,
            provider="ollama",
            prompt_tokens=response.usage.prompt_tokens if response.usage else 0,
            completion_tokens=response.usage.completion_tokens if response.usage else 0,
            latency_ms=latency,
        )
    
    async def stream(self, messages: list[dict], temperature: float = 0.0) -> AsyncGenerator[str, None]:
        stream = await self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            temperature=temperature,
            stream=True,
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

# 🎯 The Factory — one function to rule them all:
class LLMFactory:
    """Create the right provider based on config"""
    
    providers = {
        "openai": OpenAIProvider,
        "anthropic": AnthropicProvider,
        "ollama": OllamaProvider,
    }
    
    @staticmethod
    def create(provider_name: str, model: str = None) -> LLMProvider:
        provider_class = LLMFactory.providers.get(provider_name)
        if not provider_class:
            raise ValueError(f"Unknown provider: {provider_name}. Available: {list(LLMFactory.providers.keys())}")
        
        return provider_class(model=model) if model else provider_class()

# USAGE — swap providers with ONE LINE:
async def main():
    # Change this ONE string to switch providers:
    provider = LLMFactory.create("openai", "gpt-4o-mini")
    # provider = LLMFactory.create("anthropic", "claude-3-haiku")
    # provider = LLMFactory.create("ollama", "llama3.2")
    
    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is the capital of France?"}
    ]
    
    response = await provider.complete(messages)
    print(f"[{response.provider}/{response.model}] ${response.total_cost:.6f}")
    print(response.content)

asyncio.run(main())
```
</details>

---

## 📖 Debug Challenge: Provider API Differences

```python
# BUGGY CODE — This works with OpenAI but fails with Anthropic. Why?

from openai import OpenAI
import anthropic

openai_client = OpenAI()
anthropic_client = anthropic.Anthropic()

messages = [
    {"role": "system", "content": "You are a translator. Translate to French."},
    {"role": "user", "content": "Hello, how are you?"},
]

# Works fine:
openai_response = openai_client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages
)

# CRASHES! Why?
anthropic_response = anthropic_client.messages.create(
    model="claude-3-haiku",
    messages=messages  # <-- Same messages, different format expected!
)
```

**🔍 Find the bug and fix it.**

<details>
<summary>Click after thinking</summary>

**Bug:** Anthropic expects system prompt as a SEPARATE parameter, not in the messages list.

**Fix:**
```python
# Separate system from messages
system_prompt = "You are a translator. Translate to French."
filtered_messages = [{"role": "user", "content": "Hello, how are you?"}]

anthropic_response = anthropic_client.messages.create(
    model="claude-3-haiku",
    system=system_prompt,  # 🔑 Separate parameter
    messages=filtered_messages,
    max_tokens=500
)
```

**Senior Insight:** This is THE most common bug when switching providers. Always wrap provider calls in an abstraction layer (like we built above) so you fix this ONCE.
</details>

---

## 📖 LiteLLM — The "Don't Build It Yourself" Option

Before you build your own provider abstraction, know that **LiteLLM** already does this:

```python
# pip install litellm
from litellm import completion

# Same function, any provider:
response = completion(
    model="gpt-4o-mini",          # or "claude-3-haiku" or "ollama/llama3.2"
    messages=[{"role": "user", "content": "Hello!"}],
)

# Cost tracking built in:
from litellm import cost_per_token
print(f"Cost: ${response._cost:.6f}")
```

**When to build vs buy your abstraction:**
- **Build:** If you need custom behavior (special error handling, metrics, fallbacks)
- **Use LiteLLM:** If you just need multi-provider calls and cost tracking

---

## 🚦 Gate Check

- [ ] I can explain the key API differences between OpenAI, Anthropic, and Ollama
- [ ] I've implemented at least 2 providers using the abstract interface
- [ ] I can swap providers with a config change
- [ ] I understand why provider abstraction matters
- [ ] I know when to use LiteLLM vs build my own

---

**→ Continue to `08-Streaming-SSE-WebSockets.md`**
