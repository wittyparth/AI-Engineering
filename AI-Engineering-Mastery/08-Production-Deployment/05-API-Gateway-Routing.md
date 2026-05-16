# API Gateway & Routing

## Model Routing

```python
class ModelRouter:
    def __init__(self, models: dict):
        self.models = models  # {"fast": ..., "quality": ..., "cheap": ...}

    def route(self, query: str, user_tier: str = "free") -> str:
        if self._requires_quality(query):
            return self.models["quality"]
        if user_tier == "free" or self._is_simple(query):
            return self.models["cheap"]
        return self.models["balanced"]

    def _requires_quality(self, query: str) -> bool:
        # Logic: complex queries need better models
        return len(query) > 200 or "?" == query.strip()[-1]
```

## Rate Limiting

```python
from fastapi import FastAPI, HTTPException
import time
from collections import defaultdict

class RateLimiter:
    def __init__(self):
        self.requests = defaultdict(list)

    async def check(self, user_id: str, max_requests: int = 100, window: int = 60):
        now = time.time()
        user_requests = self.requests[user_id]

        # Clean old requests
        user_requests = [r for r in user_requests if r > now - window]
        self.requests[user_id] = user_requests

        if len(user_requests) >= max_requests:
            raise HTTPException(status_code=429, detail="Rate limit exceeded")

        user_requests.append(now)
```

## API Gateway Pattern

```python
class AIGateway:
    def __init__(self):
        self.router = ModelRouter()
        self.limiter = RateLimiter()
        self.auth = AuthMiddleware()
        self.audit = AuditLogger()

    async def handle_request(self, request: Request) -> Response:
        # 1. Authenticate
        user = await self.auth.verify(request)

        # 2. Rate limit
        await self.limiter.check(user.id)

        # 3. Route to appropriate model
        model = self.router.route(request.query, user.tier)

        # 4. Add model context
        enriched = await self._enrich_context(request)

        # 5. Generate
        response = await model.generate(enriched)

        # 6. Audit
        await self.audit.log(user, request, response, model)

        return response
```

## 🔴 Senior: Production Considerations

| Concern | Implementation |
|---------|---------------|
| Auth | JWT or API keys, with tier limits |
| Rate limiting | Per-user + per-IP + global |
| Cost tracking | Per-query, per-user, per-model |
| Latency SLOs | p50 < 1s, p99 < 5s |
| Fallback | If primary model fails, try cheaper model |
| Graceful degradation | Return cached response or "under maintenance" |
| Request queuing | Redis queue for burst handling |
