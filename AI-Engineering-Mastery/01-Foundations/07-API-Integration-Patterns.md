# API Integration Patterns

## The Unified Client

```python
class LLMClient:
    """Unified client across providers with retry, fallback, and telemetry."""

    def __init__(self, config: ProviderConfig):
        self.providers = {
            "openai": OpenAIClient(),
            "anthropic": AnthropicClient(),
            "ollama": OllamaClient(),
        }
        self.fallback_order = ["openai", "anthropic", "ollama"]
        self.telemetry = TelemetryClient()

    async def generate(
        self, prompt: str, provider: str = "openai", **kwargs
    ) -> StreamingResponse:
        for p in [provider] + [f for f in self.fallback_order if f != provider]:
            try:
                start = time.time()
                result = await self.providers[p].generate(prompt, **kwargs)
                self.telemetry.log(p, time.time() - start, result.usage)
                return result
            except RateLimitError:
                await asyncio.sleep(self._backoff(p))
                continue
            except ProviderError as e:
                logger.warning(f"{p} failed: {e}. Trying fallback.")
                continue
        raise AllProvidersExhausted("No provider succeeded")
```

## Retry Strategies

### Exponential Backoff
```python
def _backoff(self, attempt: int, base: float = 1.0) -> float:
    return base * (2 ** attempt) + random.uniform(0, 0.1)
```

### Jitter
Add randomness to prevent thundering herd:
```python
return base * (2 ** attempt) + random.uniform(0, min(1, base * 2 ** attempt))
```

## Rate Limiting

🔴 **Senior**: Rate limiting happens at the CLIENT, not just the server.

```python
class RateLimiter:
    def __init__(self, rpm: int = 100):
        self.rpm = rpm
        self.tokens = deque()

    async def acquire(self):
        now = time.time()
        # Remove tokens older than 1 minute
        while self.tokens and self.tokens[0] < now - 60:
            self.tokens.popleft()
        if len(self.tokens) >= self.rpm:
            sleep = self.tokens[0] + 60 - now
            await asyncio.sleep(sleep)
        self.tokens.append(now)
```

## Streaming Patterns

```python
@app.get("/chat/stream")
async def chat_stream(query: str):
    async def generate():
        async for chunk in llm_client.generate_stream(query):
            yield f"data: {json.dumps(chunk)}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

## Drill: Build the Error Tree

Create a decision tree/matrix showing which provider errors should:
- Retry (429, 500, timeout)
- Fallback (503, 403)
- Fail fast (400, 401, invalid request)
- Alert on-call (repeated 500s, auth failures)
