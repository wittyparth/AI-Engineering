# Python AI Patterns — The Code You'll Write Every Day

## 🎯 Purpose & Goals

> **🛑 STOP. Here's a problem:**
>
> You need to call an AI API 1,000 times with different prompts. Each call takes 2 seconds. If you do it one-by-one, that's 33 minutes.
>
> **How would you make it faster? Write pseudocode for your approach.**
>
> *Think about: concurrency, error handling, rate limits, progress tracking*

---

**By the end of this file, you will:**
- Write async Python that properly handles concurrent API calls
- Build production-grade error handling (not just try/except)
- Write streaming code that feels fast to users
- Know the exact Python patterns that appear in EVERY AI project

**⏱ Time Budget:** 2 hours

---

## 📖 Pattern 1: Async Everything

> **🤔 YOUR TURN:** Look at the code below. It works but it's SLOW. 
> After reading it, write the async version. Time yourself — 5 minutes.

```python
# BUGGY/SLOW CODE — What's wrong here?
from openai import OpenAI
import time

client = OpenAI()

def summarize(texts: list[str]) -> list[str]:
    """Summarize multiple texts — this takes forever"""
    results = []
    for text in texts:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": f"Summarize: {text[:500]}"}],
            max_tokens=100
        )
        results.append(response.choices[0].message.content)
    return results

# If each call takes 2 seconds, 10 texts = 20 seconds
# 100 texts = 200 seconds (3+ minutes!)
```

**Why is this slow?**
- Each `client.chat.completions.create()` is a NETWORK call
- Network calls are I/O bound, not CPU bound
- While waiting for the response, your CPU is doing NOTHING
- With async, you can send ALL requests simultaneously

**Now write the fast version:**

```python
# YOUR ASYNC VERSION:
from openai import AsyncOpenAI
import asyncio

client = AsyncOpenAI()

async def summarize_async(texts: list[str]) -> list[str]:
    """Write your async implementation here"""
    pass

# Your code goes here...
```

<details>
<summary>👀 See production solution after you've written yours</summary>

```python
from openai import AsyncOpenAI
import asyncio
from typing import List

client = AsyncOpenAI()

async def summarize_one(text: str) -> str:
    """Summarize a single text"""
    response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"Summarize in 2 sentences: {text[:500]}"}],
        max_tokens=100,
        temperature=0.0
    )
    return response.choices[0].message.content

async def summarize_many(texts: List[str]) -> List[str]:
    """
    All calls happen CONCURRENTLY.
    10 texts = ~2 seconds total (was 20 seconds).
    100 texts = ~3-4 seconds total (was 200 seconds).
    """
    tasks = [summarize_one(text) for text in texts]
    # 🔑 asyncio.gather runs all tasks concurrently
    results = await asyncio.gather(*tasks)
    return results

# With progress tracking:
async def summarize_with_progress(texts: List[str]) -> List[str]:
    """Show progress as tasks complete"""
    tasks = [summarize_one(text) for text in texts]
    
    results = [None] * len(texts)
    for i, task in enumerate(asyncio.as_completed(tasks)):
        result = await task
        results[i] = result
        print(f"✅ Completed {i+1}/{len(texts)}")
    
    return results

# Run it
results = asyncio.run(summarize_many(["Text 1...", "Text 2...", "Text 3..."]))
```

**Compare your solution:**
- Did you use `asyncio.gather` or did you do individual awaits?
- Did you handle errors? (What if one task fails?)
- Did you add progress tracking?

**Key insight:** `asyncio.gather` runs all tasks in parallel. Individual `await` calls run sequentially. The difference between 2 seconds and 20 seconds.
</details>

---

## 📖 Pattern 2: Streaming (Don't Make Users Wait)

> **🤔 YOUR TURN:** Here's a non-streaming call. How would you make it stream?

```python
# SLOW UX — User stares at spinner for 5 seconds
def ask_question(question: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": question}],
        max_tokens=500
    )
    return response.choices[0].message.content

# User sees nothing for 5 seconds, then all text appears at once
# This feels SLOW and BAD even if it's fast
```

**Write the streaming version:**

```python
# YOUR STREAMING VERSION:
def ask_question_streaming(question: str):
    """Write streaming version here"""
    pass
```

<details>
<summary>👀 Click after writing yours</summary>

```python
from openai import OpenAI

client = OpenAI()

def ask_question_streaming(question: str) -> str:
    """
    Streaming shows text AS IT'S GENERATED.
    First token appears in ~500ms.
    User reads while rest generates.
    Feels 10x faster even if same total time.
    """
    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": question}],
        stream=True,  # 🔑 This is the magic parameter
        max_tokens=500
    )
    
    full_response = ""
    for chunk in stream:
        if chunk.choices[0].delta.content is not None:
            token = chunk.choices[0].delta.content
            print(token, end="", flush=True)  # Show immediately
            full_response += token
    
    print()  # Newline at end
    return full_response

# Async streaming for FastAPI:
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI

app = FastAPI()
async_client = AsyncOpenAI()

async def generate_tokens(prompt: str):
    """Async generator — yields tokens as they arrive"""
    stream = await async_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

@app.get("/chat")
async def chat(prompt: str):
    """FastAPI endpoint that streams the response"""
    return StreamingResponse(
        generate_tokens(prompt),
        media_type="text/plain"
    )
```
</details>

---

## 📖 Pattern 3: Error Handling That Actually Works

> **🤔 YOUR TURN:** What's wrong with this try/except?
> List at least 4 things that could go wrong that this doesn't handle.

```python
# BUGGY ERROR HANDLING:
def call_llm_safely(prompt: str) -> str:
    try:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}]
        )
        return response.choices[0].message.content
    except Exception as e:
        return f"Error: {e}"
```

**Write down the 4+ missing things before reading the solution:**

1. ___
2. ___
3. ___
4. ___
5. ___

<details>
<summary>🔍 What's missing</summary>

1. **No timeout** — request could hang indefinitely 
2. **No retry for rate limits** — 429 errors happen, need backoff
3. **No retry for transient errors** — 500 errors sometimes resolve
4. **Catches ALL exceptions** — including KeyboardInterrupt, SystemExit
5. **Returns error as string** — caller can't distinguish error from actual response
6. **No logging** — can't debug when it fails
7. **No fallback model** — if one model fails, try another
</details>

**Production version:**

```python
import time
import logging
from openai import APIError, RateLimitError, APITimeoutError
from typing import Optional

logger = logging.getLogger(__name__)

def robust_completion(
    prompt: str,
    model: str = "gpt-4o-mini",
    fallback_model: str = "claude-3-haiku",  # 🔑 Always have a fallback
    max_retries: int = 3,
    timeout: int = 30
) -> Optional[str]:
    """
    Production-grade completion.
    
    Features:
    - Timeout (NEVER hang indefinitely)
    - Exponential backoff for rate limits
    - Model fallback
    - Logging every attempt
    - Returns None on failure (caller handles gracefully)
    """
    
    models_to_try = [model, fallback_model]
    
    for current_model in models_to_try:
        for attempt in range(max_retries):
            try:
                response = client.chat.completions.create(
                    model=current_model,
                    messages=[{"role": "user", "content": prompt}],
                    timeout=timeout,  # 🔑 ALWAYS set timeout
                )
                
                logger.info(f"Success: model={current_model}, attempt={attempt+1}")
                return response.choices[0].message.content
                
            except RateLimitError:
                wait = 2 ** attempt  # Exponential: 1s, 2s, 4s
                logger.warning(f"Rate limited on {current_model}, waiting {wait}s")
                time.sleep(wait)
                
            except APITimeoutError:
                logger.warning(f"Timeout on {current_model}, attempt {attempt+1}")
                time.sleep(1)
                
            except APIError as e:
                if e.status_code >= 500:  # Server error — retry
                    logger.warning(f"Server error {e.status_code}, retrying")
                    time.sleep(2 ** attempt)
                else:
                    logger.error(f"Client error {e.status_code}: {e.message}")
                    break  # Don't retry client errors
                    
            except Exception as e:
                logger.error(f"Unexpected error: {e}", exc_info=True)
                break  # Don't retry unknown errors
    
    logger.error(f"All models and retries failed for prompt: {prompt[:100]}")
    return None  # 🔑 Return None — caller decides what to do
```

---

## 📖 Pattern 4: Config-Driven Architecture

> **🤔 YOUR TURN:** What's wrong with hardcoding model names and API keys?

```python
# BAD: Hardcoded everywhere
def analyze_sentiment(text):
    client = OpenAI(api_key="sk-...")  # 🔴 API key in code!
    response = client.chat.completions.create(
        model="gpt-4o",  # 🔴 Hardcoded model
        ...
    )
```

**Production pattern:**

```python
# config.py
from pydantic_settings import BaseSettings
from typing import Optional

class AIConfig(BaseSettings):
    """Environment-based configuration — NEVER hardcode"""
    
    # API Keys
    openai_api_key: str
    anthropic_api_key: Optional[str] = None
    
    # Default models
    fast_model: str = "gpt-4o-mini"
    powerful_model: str = "gpt-4o"
    embedding_model: str = "text-embedding-3-small"
    
    # Rate limits
    max_requests_per_minute: int = 500
    max_retries: int = 3
    request_timeout: int = 30
    
    # Cost controls
    monthly_budget_limit: float = 1000.0
    max_cost_per_query: float = 0.01
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

# Load once, use everywhere
config = AIConfig()

# usage.py
client = OpenAI(api_key=config.openai_api_key)

def analyze_sentiment(text: str) -> str:
    response = client.chat.completions.create(
        model=config.fast_model,  # Config-driven!
        messages=[{"role": "user", "content": f"Sentiment: {text}"}],
        timeout=config.request_timeout,
    )
    return response.choices[0].message.content
```

---

## 📖 Pattern 5: Pydantic Validation Everywhere

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional

class LLMResponse(BaseModel):
    """Every LLM response should be validated"""
    content: str
    model: str
    prompt_tokens: int = Field(ge=0)
    completion_tokens: int = Field(ge=0)
    latency_ms: float = Field(ge=0)
    cost: float = Field(ge=0)
    
    @property
    def total_tokens(self) -> int:
        return self.prompt_tokens + self.completion_tokens
    
    @field_validator('latency_ms')
    def warn_slow(cls, v):
        if v > 5000:
            print(f"⚠️ Slow response: {v:.0f}ms")
        return v

def trackable_completion(prompt: str, model: str = "gpt-4o-mini") -> LLMResponse:
    """Every call returns validated, tracked metrics"""
    import time
    
    start = time.time()
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
    elapsed = (time.time() - start) * 1000
    
    pricing = {"gpt-4o-mini": (0.15, 0.60), "gpt-4o": (2.50, 10.00)}
    input_rate, output_rate = pricing.get(model, (0.15, 0.60))
    
    cost = (response.usage.prompt_tokens / 1_000_000 * input_rate) + \
           (response.usage.completion_tokens / 1_000_000 * output_rate)
    
    return LLMResponse(
        content=response.choices[0].message.content,
        model=model,
        prompt_tokens=response.usage.prompt_tokens,
        completion_tokens=response.usage.completion_tokens,
        latency_ms=elapsed,
        cost=cost,
    )

# Now every call is auditable:
result = trackable_completion("What is Python?")
print(f"Tokens: {result.total_tokens}, Cost: ${result.cost:.6f}, Latency: {result.latency_ms:.0f}ms")
```

---

## 🧪 Final Design Challenge

**Time: 30 minutes**

**SCENARIO:** Build a function `batch_classify(items: list[str], categories: list[str]) -> list[dict]` that:

1. Classifies each item into one of the given categories
2. Runs ALL classifications concurrently (not sequentially)
3. Handles errors gracefully (if one fails, continue with others)
4. Tracks cost and latency for each classification
5. Shows progress as classifications complete

```python
# 🛑 WRITE YOUR SOLUTION FIRST:

async def batch_classify(items: list[str], categories: list[str]) -> list[dict]:
    """Classify multiple items concurrently"""
    # YOUR CODE HERE
    pass
```

<details>
<summary>👀 Production solution</summary>

```python
from openai import AsyncOpenAI
import asyncio
import time
from typing import List
from pydantic import BaseModel

client = AsyncOpenAI()

class Classification(BaseModel):
    item: str
    category: str
    confidence: float
    tokens_used: int
    cost: float
    latency_ms: float
    error: str | None = None

async def classify_one(item: str, categories: List[str]) -> Classification:
    """Classify a single item"""
    start = time.time()
    
    try:
        response = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"""Classify this item into ONE of these categories: {', '.join(categories)}

Item: {item}

Return your answer as: CATEGORY_NAME|CONFIDENCE_SCORE
Example: Electronics|0.95"""
            }],
            temperature=0.0,
            max_tokens=20,
            timeout=10
        )
        
        latency = (time.time() - start) * 1000
        content = response.choices[0].message.content
        
        # Parse response
        parts = content.split("|")
        category = parts[0].strip() if parts else "UNKNOWN"
        confidence = float(parts[1]) if len(parts) > 1 and parts[1].strip() else 0.0
        
        cost = (response.usage.prompt_tokens / 1_000_000 * 0.15) + \
               (response.usage.completion_tokens / 1_000_000 * 0.60)
        
        return Classification(
            item=item, category=category, confidence=confidence,
            tokens_used=response.usage.total_tokens, cost=cost,
            latency_ms=latency
        )
        
    except Exception as e:
        return Classification(
            item=item, category="ERROR", confidence=0.0,
            tokens_used=0, cost=0.0, latency_ms=(time.time()-start)*1000,
            error=str(e)
        )

async def batch_classify(items: List[str], categories: List[str]) -> List[Classification]:
    """Classify ALL items concurrently with progress"""
    tasks = [classify_one(item, categories) for item in items]
    
    results = []
    for i, task in enumerate(asyncio.as_completed(tasks)):
        result = await task
        results.append(result)
        status = "✅" if not result.error else "❌"
        print(f"{status} [{i+1}/{len(items)}] {result.item[:30]:30} → {result.category}")
    
    # Sort to match original order
    item_order = {item: i for i, item in enumerate(items)}
    results.sort(key=lambda r: item_order.get(r.item, 0))
    
    # Summary
    total_cost = sum(r.cost for r in results)
    avg_latency = sum(r.latency_ms for r in results) / len(results)
    errors = sum(1 for r in results if r.error)
    
    print(f"\n📊 Summary: {len(results)-errors}/{len(results)} succeeded")
    print(f"💰 Total cost: ${total_cost:.4f}")
    print(f"⚡ Avg latency: {avg_latency:.0f}ms")
    
    return results
```
</details>

---

## 🚦 Gate Check

- [ ] I can write async code that runs API calls concurrently
- [ ] I can implement streaming with proper token-by-token output
- [ ] I can build error handling with retries and fallbacks
- [ ] I use Pydantic to validate every LLM response
- [ ] I use config/env vars instead of hardcoding
- [ ] I've completed the batch_classify design challenge

---

**→ Continue to `07-API-Integration-Patterns.md`**
