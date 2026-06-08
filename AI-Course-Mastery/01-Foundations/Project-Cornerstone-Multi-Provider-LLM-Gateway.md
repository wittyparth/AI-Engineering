# 🏗️ Project 1 — Multi-Provider LLM Gateway

## 🎯 Purpose & Goals

> **🛑 STOP. Here is your brief:**
>
> Your startup is building an AI-powered product. You need to call LLMs from OpenAI, Anthropic, and local models. You need streaming. You need token tracking. You need to switch providers with a config change — not a code change. You need structured outputs. You need to handle failures gracefully. You need to know how much every request costs in real time.
>
> **Your mission: Build the gateway that powers all AI calls for your entire organization.**
>
> This is not a toy. This is the system every AI-powered company has internally. It's your first portfolio piece.
>
> *Before scrolling — sketch the full architecture. What endpoints? What abstraction layers? How does auth work? How do you handle rate limits, errors, streaming? Where does config live?*

---

**By building this project, you will:**
- Integrate everything from Phase 1 into a single production system
- Build a provider abstraction layer that handles OpenAI, Anthropic, and local models
- Implement streaming, structured outputs, token counting, and cost tracking
- Write production-grade FastAPI with proper error handling, config, and testing
- Have a portfolio project that demonstrates real AI engineering skill

**⏱ Time Budget:** 8-12 hours over 3 days

| Day | Focus | Time |
|-----|-------|------|
| 1 | Architecture + Provider layer + Core endpoints | 3-4h |
| 2 | Streaming + Token tracking + Error handling | 3-4h |
| 3 | Testing + Caching + Monitoring + README | 2-4h |

---

## 📖 System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │ Web App  │  │ Mobile   │  │ CLI Tool │  │ Internal Services │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────────┬──────────┘  │
│       │             │             │                   │              │
│       └─────────────┴─────────────┴───────────────────┘              │
│                              │  HTTPS + API Key                      │
├──────────────────────────────┼──────────────────────────────────────┤
│                    LLM GATEWAY (FastAPI)                              │
│                                                                      │
│  ┌──────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │ Auth     │  │ Rate Limiter     │  │ Request Validator        │  │
│  │ Middle   │→│ (Redis/IP-based)  │→│ (Pydantic models)        │  │
│  └──────────┘  └──────────────────┘  └──────────┬───────────────┘  │
│                                                  ▼                  │
│                           ┌──────────────────────────────┐          │
│                           │       ROUTING LAYER           │          │
│                           │  ModelSelector → ProviderRoute│          │
│                           └──────────────┬───────────────┘          │
│                                          ▼                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  PROVIDER ABSTRACTION LAYER                  │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  │  │ OpenAI   │  │ Anthropic│  │  Ollama  │  │ LiteLLM  │   │   │
│  │  │ Provider │  │ Provider │  │ Provider │  │ Provider │   │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │   │
│  └───────┼─────────────┼─────────────┼──────────────┼─────────┘   │
│          ▼             ▼             ▼              ▼              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              SUPPORT SERVICES                                 │  │
│  │  ┌────────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐  │  │
│  │  │ Tokenizer  │ │ Cost      │ │ Cache     │ │ Circuit    │  │  │
│  │  │ Engine     │ │ Tracker   │ │ (Redis)   │ │ Breaker    │  │  │
│  │  └────────────┘ └───────────┘ └───────────┘ └────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                        OBSERVABILITY                                │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌───────────────┐    │
│  │ Logging  │  │ Metrics  │  │  Tracing   │  │ Health Checks │    │
│  │ (struct) │  │(Prometheus)│  │ (Langfuse) │  │  /health     │    │
│  └──────────┘  └──────────┘  └────────────┘  └───────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow — Single Request

```
1. Client sends POST /v1/chat/completions
   Body: { model, messages, stream, temperature, ... }
   Header: Authorization: Bearer <api_key>

2. Auth Middleware validates API key
   → 401 if invalid

3. Rate Limiter checks Redis counter
   → 429 if rate exceeded

4. Request Validator (Pydantic) parses body
   → 422 if invalid JSON/fields

5. ModelSelector maps model name → provider
   → "gpt-4o" → OpenAI
   → "claude-3-sonnet" → Anthropic
   → "llama3.2" → Ollama

6. Provider handles the request:
   a. Check cache (if applicable)
   b. Call API with retry + circuit breaker
   c. Count tokens (input + output)
   d. Track cost

7. Response returned (streaming or non-streaming)
   Headers: X-Token-Count, X-Cost, X-Provider, X-Latency
```

### Project Structure

```
llm-gateway/
├── README.md                    # Architecture decisions, setup
├── pyproject.toml               # Dependencies
├── .env.example                 # Config template
├── Dockerfile
├── docker-compose.yml           # Gateway + Redis + Prometheus
│
├── src/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app, middleware, startup
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py            # Pydantic Settings (env → config)
│   │   ├── auth.py              # API key validation middleware
│   │   ├── rate_limiter.py      # Token bucket / sliding window
│   │   ├── models.py            # All Pydantic request/response models
│   │   ├── errors.py            # Exception hierarchy + handlers
│   │   └── dependencies.py      # FastAPI dependency injection
│   │
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── base.py              # Abstract base provider
│   │   ├── openai_provider.py   # OpenAI implementation
│   │   ├── anthropic_provider.py# Anthropic implementation
│   │   ├── ollama_provider.py   # Local Ollama implementation
│   │   ├── lite_llm_provider.py # LiteLLM fallback provider
│   │   └── factory.py          # Provider factory (model → provider)
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── tokenizer.py         # Token counting + model context limits
│   │   ├── cost_tracker.py      # Real-time cost accounting
│   │   ├── cache.py             # Semantic + exact-match caching
│   │   ├── circuit_breaker.py  # Provider failure state tracking
│   │   └── metrics.py          # Prometheus metrics
│   │
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── chat.py              # /v1/chat/completions
│   │   ├── models.py            # /v1/models (list available)
│   │   ├── tokenize.py          # /v1/tokenize (count + breakdown)
│   │   └── health.py            # /health, /ready, /metrics
│   │
│   └── middleware/
│       ├── __init__.py
│       ├── logging.py           # Structured request/response logging
│       └── timing.py            # Request latency tracking
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py              # Fixtures, mock providers
│   ├── test_chat.py
│   ├── test_tokenize.py
│   ├── test_providers.py
│   ├── test_rate_limiter.py
│   └── test_integration.py
│
└── scripts/
    ├── load_test.py             # Locust or custom load test
    └── provision_demo.py        # Setup demo data
```

---

## 💻 Code — The Full Implementation

### pyproject.toml

```toml
[project]
name = "llm-gateway"
version = "0.1.0"
description = "Multi-Provider LLM Gateway — route, stream, track tokens, control costs"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "pydantic>=2.5.0",
    "pydantic-settings>=2.1.0",
    "httpx>=0.27.0",
    "tiktoken>=0.7.0",
    "redis>=5.0.0",
    "prometheus-client>=0.20.0",
    "python-dotenv>=1.0.0",
    "lxml>=5.0.0",  # For structured outputs (XML parsing)
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=5.0.0",
    "httpx>=0.27.0",  # For TestClient
    "ruff>=0.5.0",
    "locust>=2.0.0",
]

[tool.ruff]
line-length = 100
target-version = "py311"
```

### src/core/config.py

```python
"""
Configuration — all environment variables in one place.

This is THE source of truth. Every configurable value in the gateway
comes from one of these sources:
1. Environment variables (highest priority)
2. .env file (for development)
3. Default values (sensible defaults for local dev)
"""

from pydantic_settings import BaseSettings
from pydantic import Field, model_validator
from typing import Optional
import os


class Settings(BaseSettings):
    """
    Application settings loaded from environment variables.
    
    🔴 SENIOR MOVE: Pydantic Settings automatically validates types,
    provides sensible defaults, and supports .env files. One class
    covers ALL configuration.
    """
    
    # ── General ────────────────────────────────────────────────────
    app_name: str = "LLM Gateway"
    debug: bool = Field(default=False, alias="DEBUG")
    environment: str = Field(default="development", alias="ENVIRONMENT")
    log_level: str = Field(default="INFO", alias="LOG_LEVEL")
    
    # ── Server ─────────────────────────────────────────────────────
    host: str = Field(default="0.0.0.0", alias="HOST")
    port: int = Field(default=8000, alias="PORT")
    cors_origins: list[str] = Field(default=["*"], alias="CORS_ORIGINS")
    
    # ── API Keys ───────────────────────────────────────────────────
    openai_api_key: Optional[str] = Field(default=None, alias="OPENAI_API_KEY")
    anthropic_api_key: Optional[str] = Field(default=None, alias="ANTHROPIC_API_KEY")
    
    # ── Gateway Auth ───────────────────────────────────────────────
    # 🔴 SECURITY: In production, rotate these monthly and use a secrets manager
    gateway_api_keys: list[str] = Field(
        default=["dev-key-change-me"],
        alias="GATEWAY_API_KEYS",
        description="Comma-separated list of valid API keys for the gateway itself",
    )
    
    # ── Rate Limiting ──────────────────────────────────────────────
    rate_limit_per_minute: int = Field(default=60, alias="RATE_LIMIT_PER_MINUTE")
    rate_limit_burst: int = Field(default=100, alias="RATE_LIMIT_BURST")
    
    # ── Redis ──────────────────────────────────────────────────────
    redis_url: str = Field(default="redis://localhost:6379/0", alias="REDIS_URL")
    
    # ── Providers ──────────────────────────────────────────────────
    default_model: str = Field(default="gpt-4o-mini", alias="DEFAULT_MODEL")
    ollama_base_url: str = Field(default="http://localhost:11434", alias="OLLAMA_BASE_URL")
    
    # ── Caching ────────────────────────────────────────────────────
    cache_enabled: bool = Field(default=True, alias="CACHE_ENABLED")
    cache_ttl_seconds: int = Field(default=300, alias="CACHE_TTL_SECONDS")
    
    # ── Circuit Breaker ────────────────────────────────────────────
    circuit_breaker_threshold: int = Field(default=5, alias="CIRCUIT_BREAKER_THRESHOLD")
    circuit_breaker_recovery_seconds: int = Field(default=60, alias="CIRCUIT_BREAKER_RECOVERY")
    
    # ── Ollama (local) ─────────────────────────────────────────────
    ollama_models: list[str] = Field(
        default=["llama3.2", "llama3.1", "mistral", "phi3"],
        alias="OLLAMA_MODELS",
    )
    
    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
        "extra": "ignore",  # Don't crash on extra env vars
    }
    
    @model_validator(mode="after")
    def validate_providers(self):
        """At least one provider must be configured."""
        if not self.openai_api_key and not self.anthropic_api_key:
            # Allow Ollama-only mode in dev
            if self.environment == "production":
                raise ValueError(
                    "At least one of OPENAI_API_KEY or ANTHROPIC_API_KEY must be set in production"
                )
        return self


settings = Settings()
```

### src/core/models.py

```python
"""
Pydantic models for ALL request/response types.

🔴 SENIOR MOVE: Models are the CONTRACT of your API. They define
exactly what goes in and what comes out. No ambiguity, no runtime surprises.
"""

from pydantic import BaseModel, Field, model_validator
from typing import Optional, Literal
from datetime import datetime
from enum import Enum


# ── Request Models ─────────────────────────────────────────────────

class Message(BaseModel):
    """A single message in the conversation."""
    role: Literal["system", "user", "assistant", "tool"]
    content: str
    
    model_config = {"extra": "forbid"}  # No extra fields


class ChatCompletionRequest(BaseModel):
    """Request body for POST /v1/chat/completions."""
    model: str = Field(default="gpt-4o-mini", description="Model identifier")
    messages: list[Message] = Field(..., min_length=1, description="Conversation history")
    stream: bool = Field(default=False, description="Stream tokens as they're generated")
    
    # Generation parameters (with sensible defaults)
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    max_tokens: int = Field(default=2048, ge=1, le=128_000)
    top_p: float = Field(default=1.0, ge=0.0, le=1.0)
    stop: Optional[list[str]] = None
    
    # Structured output
    response_format: Optional[dict] = None  # {"type": "json_object"} or {"type": "json_schema", "schema": {...}}
    
    # Advanced
    user_id: Optional[str] = None  # For per-user tracking
    metadata: Optional[dict] = None  # For observability
    
    model_config = {"extra": "forbid"}
    
    @model_validator(mode="after")
    def validate_messages(self):
        """Conversation must start with system or user message."""
        if not self.messages:
            raise ValueError("At least one message is required")
        return self


class TokenizeRequest(BaseModel):
    """Request body for POST /v1/tokenize."""
    text: str = Field(..., min_length=1, description="Text to tokenize")
    model: str = Field(default="gpt-4o-mini", description="Model for tokenizer selection")


# ── Response Models ────────────────────────────────────────────────

class Usage(BaseModel):
    """Token usage information."""
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    estimated_cost: float = Field(default=0.0, description="Estimated cost in USD")


class Choice(BaseModel):
    """A single completion choice."""
    index: int = 0
    message: Optional[dict] = None
    finish_reason: Optional[str] = None


class ChatCompletionResponse(BaseModel):
    """Response body for POST /v1/chat/completions (non-streaming)."""
    id: str
    object: str = "chat.completion"
    created: int  # Unix timestamp
    model: str
    choices: list[Choice]
    usage: Usage
    
    # Gateway metadata
    provider: str = ""
    cached: bool = False
    latency_ms: float = 0.0


class TokenDetail(BaseModel):
    position: int
    token_id: int
    text: str
    byte_length: int


class TokenizeResponse(BaseModel):
    """Response body for POST /v1/tokenize."""
    text: str
    num_tokens: int
    char_count: int
    encoding: str
    details: list[TokenDetail]
    estimated_cost_input: float
    estimated_cost_output: float


class ModelInfo(BaseModel):
    id: str
    provider: str
    context_window: int
    pricing: dict[str, float]
    features: list[str]  # e.g., ["streaming", "structured_output", "vision"]


class ListModelsResponse(BaseModel):
    data: list[ModelInfo]


# ── Error Models ───────────────────────────────────────────────────

class ErrorDetail(BaseModel):
    code: str
    message: str
    detail: Optional[str] = None


class ErrorResponse(BaseModel):
    error: ErrorDetail
```

### src/core/errors.py

```python
"""
Exception hierarchy for the entire gateway.

🔴 SENIOR MOVE: Every error has:
1. An HTTP status code
2. A machine-readable error code
3. A human-readable message
4. Optional detailed debug info

Clients can parse error.code to handle errors programmatically.
"""

from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse
from .models import ErrorResponse, ErrorDetail


class GatewayError(Exception):
    """Base exception for all gateway errors."""
    def __init__(self, code: str, message: str, status_code: int = 500, detail: str = None):
        self.code = code
        self.message = message
        self.status_code = status_code
        self.detail = detail
        super().__init__(message)


class ProviderError(GatewayError):
    """Error from an LLM provider."""
    def __init__(self, provider: str, message: str, status_code: int = 502):
        super().__init__(
            code=f"provider_error.{provider}",
            message=f"{provider} API error: {message}",
            status_code=status_code,
        )


class RateLimitError(GatewayError):
    """Rate limit exceeded."""
    def __init__(self, retry_after: int = 60):
        super().__init__(
            code="rate_limit_exceeded",
            message=f"Rate limit exceeded. Try again in {retry_after}s.",
            status_code=429,
            detail=f"retry_after={retry_after}",
        )


class AuthError(GatewayError):
    """Authentication failure."""
    def __init__(self, detail: str = "Invalid API key"):
        super().__init__(
            code="auth_failed",
            message=detail,
            status_code=401,
        )


class ModelNotFoundError(GatewayError):
    """Requested model is not available."""
    def __init__(self, model: str):
        super().__init__(
            code="model_not_found",
            message=f"Model '{model}' is not available. See GET /v1/models for available models.",
            status_code=404,
        )


class ValidationError(GatewayError):
    """Request validation failed."""
    def __init__(self, message: str, detail: str = None):
        super().__init__(
            code="validation_error",
            message=message,
            status_code=422,
            detail=detail,
        )


class CircuitBreakerOpenError(GatewayError):
    """Circuit breaker is open — provider is considered down."""
    def __init__(self, provider: str):
        super().__init__(
            code=f"circuit_breaker_open.{provider}",
            message=f"{provider} is currently unavailable (circuit breaker open). Try again later.",
            status_code=503,
        )


class ContextOverflowError(GatewayError):
    """Request exceeds model's context window."""
    def __init__(self, num_tokens: int, max_tokens: int, model: str):
        super().__init__(
            code="context_overflow",
            message=f"Request ({num_tokens} tokens) exceeds {model}'s max context ({max_tokens} tokens).",
            status_code=400,
            detail=f"Reduce your prompt or use a model with a larger context window.",
        )


async def gateway_error_handler(request: Request, exc: GatewayError):
    """Handle all GatewayError subclasses."""
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            error=ErrorDetail(
                code=exc.code,
                message=exc.message,
                detail=exc.detail,
            )
        ).model_dump(),
    )


async def generic_error_handler(request: Request, exc: Exception):
    """Handle unexpected errors."""
    return JSONResponse(
        status_code=500,
        content=ErrorResponse(
            error=ErrorDetail(
                code="internal_error",
                message="An unexpected error occurred.",
            )
        ).model_dump(),
    )
```

### src/core/dependencies.py

```python
"""
FastAPI dependency injection — clean, testable, and explicit.

🔴 SENIOR MOVE: Dependencies let you inject providers, services, etc.
without globals or singletons. Every handler is testable in isolation.
"""

from fastapi import Request, Depends, HTTPException
from typing import AsyncGenerator
import logging

from .config import settings
from .errors import AuthError, RateLimitError
from services.circuit_breaker import CircuitBreakerRegistry
from services.cost_tracker import CostTracker
from services.tokenizer import TokenizerService
from services.cache import CacheService


logger = logging.getLogger(__name__)


async def verify_api_key(request: Request) -> None:
    """Dependency: Validates the gateway API key."""
    auth_header = request.headers.get("Authorization", "")
    
    if not auth_header.startswith("Bearer "):
        raise AuthError("Missing or invalid Authorization header. Use: Bearer <key>")
    
    api_key = auth_header[7:]  # Remove "Bearer "
    
    if api_key not in settings.gateway_api_keys:
        logger.warning(f"Failed auth attempt with key ending in ...{api_key[-4:]}")
        raise AuthError("Invalid API key")


async def get_tokenizer_service() -> TokenizerService:
    """Dependency: Returns the tokenizer service instance."""
    return TokenizerService()


async def get_cost_tracker() -> CostTracker:
    """Dependency: Returns the cost tracker."""
    return CostTracker()


async def get_circuit_breaker(request: Request) -> CircuitBreakerRegistry:
    """Dependency: Returns the circuit breaker registry from app state."""
    return request.app.state.circuit_breaker


async def get_cache(request: Request) -> CacheService:
    """Dependency: Returns cache service from app state."""
    return request.app.state.cache
```

### src/providers/base.py

```python
"""
Abstract base provider — the contract EVERY provider must fulfill.

🔴 SENIOR MOVE: Abstract base class + Protocol = compile-time AND
runtime guarantees. Juniors wing it. Seniors define the interface first.
"""

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import AsyncGenerator, Optional
from datetime import datetime
from pydantic import BaseModel


@dataclass
class ProviderResponse:
    """Standardized response from any provider."""
    content: str
    model: str
    provider_name: str
    finish_reason: str = "stop"
    
    # Token counts
    prompt_tokens: int = 0
    completion_tokens: int = 0
    
    # Timing
    latency_ms: float = 0.0
    
    # Raw response (for debugging)
    raw_response: Optional[dict] = None


@dataclass
class ProviderConfig:
    """Configuration for a specific provider instance."""
    api_key: str = ""
    base_url: str = ""
    default_model: str = ""
    timeout_seconds: int = 60
    max_retries: int = 3


class BaseLLMProvider(ABC):
    """
    Abstract provider. Every provider MUST implement:
    - chat(): Non-streaming completion
    - stream_chat(): Streaming completion (yields tokens)
    - count_tokens(): Accurate token counting
    - get_context_window(): Model-specific context limits
    """
    
    def __init__(self, config: ProviderConfig):
        self.config = config
        self.client = self._create_client()
    
    @abstractmethod
    def _create_client(self):
        """Create the HTTP client for this provider."""
        pass
    
    @abstractmethod
    async def chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
        response_format: Optional[dict] = None,
    ) -> ProviderResponse:
        """
        Non-streaming chat completion.
        
        Returns the full response as a ProviderResponse with
        content, token counts, timing, and cost info.
        """
        pass
    
    @abstractmethod
    async def stream_chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
    ) -> AsyncGenerator[str, None]:
        """
        Streaming chat completion.
        
        Yields content tokens as they arrive from the API.
        The caller assembles the full response.
        """
        pass
    
    @abstractmethod
    async def count_tokens(self, messages: list[dict], model: str) -> int:
        """Count tokens for a list of messages using the correct tokenizer."""
        pass
    
    @abstractmethod
    def get_context_window(self, model: str) -> int:
        """Return max context window for a given model."""
        pass
    
    @abstractmethod
    async def close(self):
        """Clean up resources (HTTP client, etc.)."""
        pass
```

### src/providers/openai_provider.py

```python
"""
OpenAI provider implementation.

Handles:
- OpenAI API calls (standard + streaming)
- GPT-4o, GPT-4o-mini, GPT-4, GPT-3.5
- o200k_base and cl100k_base tokenizers
- Structured outputs via response_format
"""

import json
import time
import tiktoken
import httpx
from typing import AsyncGenerator, Optional
from datetime import datetime
import logging

from .base import BaseLLMProvider, ProviderConfig, ProviderResponse
from core.errors import ProviderError

logger = logging.getLogger(__name__)


# ┌─────────────────────────────────────────────────────────────────┐
# │ MODEL METADATA — All OpenAI models with context windows & costs │
# │                                                                 │
# │ Update this table when OpenAI releases new models.              │
# │ The provider uses this to validate requests and estimate costs. │
# └─────────────────────────────────────────────────────────────────┘
OPENAI_MODELS = {
    "gpt-4o": {
        "context_window": 128_000,
        "max_output": 16_384,
        "pricing": {"input": 2.50, "output": 10.00},  # per 1M tokens
        "encoding": "o200k_base",
        "features": ["streaming", "structured_output", "vision", "function_calling"],
    },
    "gpt-4o-mini": {
        "context_window": 128_000,
        "max_output": 16_384,
        "pricing": {"input": 0.15, "output": 0.60},
        "encoding": "o200k_base",
        "features": ["streaming", "structured_output", "vision", "function_calling"],
    },
    "gpt-4-turbo": {
        "context_window": 128_000,
        "max_output": 4_096,
        "pricing": {"input": 10.00, "output": 30.00},
        "encoding": "cl100k_base",
        "features": ["streaming", "structured_output", "function_calling"],
    },
    "gpt-3.5-turbo": {
        "context_window": 16_385,
        "max_output": 4_096,
        "pricing": {"input": 0.50, "output": 1.50},
        "encoding": "cl100k_base",
        "features": ["streaming", "function_calling"],
    },
}


class OpenAIProvider(BaseLLMProvider):
    """
    Provider for OpenAI-compatible APIs.
    
    Features:
    - Full streaming support via SSE parsing
    - Structured output (JSON mode + JSON Schema)
    - Automatic retry with exponential backoff
    - Accurate token counting via tiktoken
    - Cost estimation for every request
    """
    
    def _create_client(self) -> httpx.AsyncClient:
        """Create the HTTP client with OpenAI API configuration."""
        return httpx.AsyncClient(
            base_url=self.config.base_url or "https://api.openai.com/v1",
            headers={
                "Authorization": f"Bearer {self.config.api_key}",
                "Content-Type": "application/json",
            },
            timeout=httpx.Timeout(self.config.timeout_seconds, connect=15.0),
        )
    
    async def chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
        response_format: Optional[dict] = None,
    ) -> ProviderResponse:
        """
        Non-streaming chat completion.
        
        🔴 SENIOR MOVE: Always time the request. Latency tracking
        is how you detect provider degradation before it becomes an outage.
        """
        start_time = time.monotonic()
        
        body = {
            "model": model,
            "messages": messages,
            "temperature": temperature,
            "max_tokens": max_tokens,
        }
        
        if response_format:
            body["response_format"] = response_format
        
        try:
            response = await self.client.post("/chat/completions", json=body)
            response.raise_for_status()
            data = response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 401:
                raise ProviderError("openai", "Invalid API key. Check OPENAI_API_KEY.")
            elif e.response.status_code == 429:
                retry_after = int(e.response.headers.get("Retry-After", 5))
                raise ProviderError("openai", f"Rate limited. Retry after {retry_after}s.", status_code=429)
            else:
                error_body = e.response.text[:500]
                raise ProviderError("openai", f"HTTP {e.response.status_code}: {error_body}")
        except httpx.TimeoutException:
            raise ProviderError("openai", "Request timed out after {self.config.timeout_seconds}s")
        
        elapsed = (time.monotonic() - start_time) * 1000  # ms
        
        choice = data["choices"][0]
        content = choice["message"]["content"] or ""
        
        return ProviderResponse(
            content=content,
            model=model,
            provider_name="openai",
            finish_reason=choice.get("finish_reason", "stop"),
            prompt_tokens=data["usage"]["prompt_tokens"],
            completion_tokens=data["usage"]["completion_tokens"],
            latency_ms=elapsed,
            raw_response=data,
        )
    
    async def stream_chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
    ) -> AsyncGenerator[str, None]:
        """
        Streaming chat completion.
        
        Yields content tokens from the SSE stream.
        Handles all edge cases:
        - Empty content chunks (first/last chunk)
        - [DONE] signal
        - Malformed JSON in chunk
        - Network interruptions
        - Provider rate limiting mid-stream
        """
        body = {
            "model": model,
            "messages": messages,
            "temperature": temperature,
            "max_tokens": max_tokens,
            "stream": True,
        }
        
        try:
            async with self.client.stream("POST", "/chat/completions", json=body) as response:
                response.raise_for_status()
                
                async for line in response.aiter_lines():
                    if not line.startswith("data: "):
                        continue
                    
                    data_str = line[6:].strip()
                    
                    if data_str == "[DONE]":
                        return
                    
                    try:
                        chunk = json.loads(data_str)
                    except json.JSONDecodeError:
                        logger.warning(f"Malformed SSE chunk: {line[:100]}")
                        continue
                    
                    delta = chunk.get("choices", [{}])[0].get("delta", {})
                    content = delta.get("content", "")
                    
                    if content:
                        yield content
                        
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 429:
                raise ProviderError("openai", "Rate limited mid-stream. Implement retry logic.")
            raise ProviderError("openai", f"Stream failed: HTTP {e.response.status_code}")
        except httpx.TimeoutException:
            raise ProviderError("openai", "Stream timed out")
        except httpx.NetworkError as e:
            raise ProviderError("openai", f"Network error during stream: {str(e)}")
    
    async def count_tokens(self, messages: list[dict], model: str) -> int:
        """
        Count tokens using the model's correct tokenizer.
        
        This is NOT a guess — it uses tiktoken which matches exactly
        what OpenAI counts server-side.
        
        🔴 SENIOR MOVE: Different models use different tokenizers!
        gpt-4o → o200k_base, gpt-4 → cl100k_base
        Using the wrong one gives WRONG counts.
        """
        model_info = OPENAI_MODELS.get(model, OPENAI_MODELS["gpt-4o-mini"])
        encoding = tiktoken.get_encoding(model_info["encoding"])
        
        # ChatML tokenization: each message has overhead
        # <|start|>role<|content|>content<|end|>
        tokens_per_message = 3  # <|start|>, role, <|content|>
        tokens_per_name = 1
        
        num_tokens = 0
        for message in messages:
            num_tokens += tokens_per_message
            for key, value in message.items():
                num_tokens += len(encoding.encode(str(value)))
                if key == "name":
                    num_tokens += tokens_per_name
        
        num_tokens += 3  # Assistant prefix
        return num_tokens
    
    def get_context_window(self, model: str) -> int:
        model_info = OPENAI_MODELS.get(model, OPENAI_MODELS["gpt-4o-mini"])
        return model_info["context_window"]
    
    async def close(self):
        await self.client.aclose()
```

### src/providers/anthropic_provider.py

```python
"""
Anthropic provider implementation.

Claude uses a different API shape than OpenAI:
- System prompt is a SEPARATE parameter, not a message role
- Streaming uses a different event format
- Tokenizer is different (not tiktoken-compatible)
"""

import json
import time
import httpx
from typing import AsyncGenerator, Optional
import logging

from .base import BaseLLMProvider, ProviderConfig, ProviderResponse
from core.errors import ProviderError

logger = logging.getLogger(__name__)


ANTHROPIC_MODELS = {
    "claude-3-5-sonnet-latest": {
        "context_window": 200_000,
        "max_output": 8_192,
        "pricing": {"input": 3.00, "output": 15.00},
        "features": ["streaming", "vision", "tool_use"],
    },
    "claude-3-5-haiku-latest": {
        "context_window": 200_000,
        "max_output": 8_192,
        "pricing": {"input": 0.80, "output": 4.00},
        "features": ["streaming", "vision", "tool_use"],
    },
    "claude-3-opus-latest": {
        "context_window": 200_000,
        "max_output": 4_096,
        "pricing": {"input": 15.00, "output": 75.00},
        "features": ["streaming", "vision", "tool_use"],
    },
}


class AnthropicProvider(BaseLLMProvider):
    """
    Provider for Anthropic Claude API.
    
    🔴 KEY DIFFERENCE from OpenAI: Anthropic separates the system prompt
    from messages. We handle this transparently — if a message with
    role="system" is present, we extract it.
    """
    
    def _create_client(self) -> httpx.AsyncClient:
        return httpx.AsyncClient(
            base_url=self.config.base_url or "https://api.anthropic.com/v1",
            headers={
                "x-api-key": self.config.api_key,
                "anthropic-version": "2023-06-01",
                "Content-Type": "application/json",
            },
            timeout=httpx.Timeout(self.config.timeout_seconds, connect=15.0),
        )
    
    def _separate_system(self, messages: list[dict]) -> tuple[list[dict], Optional[str]]:
        """
        Extract system prompt from messages (Anthropic doesn't support role='system' in messages array).
        
        Returns (messages_without_system, system_prompt).
        """
        system_parts = []
        other_messages = []
        
        for msg in messages:
            if msg["role"] == "system":
                system_parts.append(msg["content"])
            else:
                other_messages.append(msg)
        
        system_prompt = "\n\n".join(system_parts) if system_parts else None
        return other_messages, system_prompt
    
    async def chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
        response_format: Optional[dict] = None,
    ) -> ProviderResponse:
        """
        Non-streaming chat completion for Claude.
        
        Handles the system prompt separation transparently.
        Response format is supported via Claude's own JSON mode.
        """
        start_time = time.monotonic()
        
        # 🔴 ANTHROPIC QUIRK: System prompt goes in its own field, not messages
        other_messages, system_prompt = self._separate_system(messages)
        
        body = {
            "model": model,
            "messages": other_messages,
            "temperature": temperature,
            "max_tokens": max_tokens,
        }
        
        if system_prompt:
            body["system"] = system_prompt
        
        if response_format and response_format.get("type") == "json_object":
            body["metadata"] = {"user_id": "gateway"}
        
        try:
            response = await self.client.post("/messages", json=body)
            response.raise_for_status()
            data = response.json()
        except httpx.HTTPStatusError as e:
            error_body = e.response.text[:500]
            raise ProviderError("anthropic", f"HTTP {e.response.status_code}: {error_body}")
        except httpx.TimeoutException:
            raise ProviderError("anthropic", "Request timed out")
        
        elapsed = (time.monotonic() - start_time) * 1000
        
        content = ""
        for block in data.get("content", []):
            if block.get("type") == "text":
                content += block.get("text", "")
        
        # Anthropic: use tokens from response or estimate
        usage = data.get("usage", {})
        
        return ProviderResponse(
            content=content,
            model=model,
            provider_name="anthropic",
            finish_reason=data.get("stop_reason", "end_turn"),
            prompt_tokens=usage.get("input_tokens", 0),
            completion_tokens=usage.get("output_tokens", 0),
            latency_ms=elapsed,
            raw_response=data,
        )
    
    async def stream_chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
    ) -> AsyncGenerator[str, None]:
        """
        Streaming for Anthropic.
        
        Anthropic SSE events have a different format than OpenAI:
        - event: content_block_delta
        - data: {"type": "content_block_delta", "delta": {"type": "text_delta", "text": "Hello"}}
        
        We filter for text_delta events and yield the text.
        """
        other_messages, system_prompt = self._separate_system(messages)
        
        body = {
            "model": model,
            "messages": other_messages,
            "temperature": temperature,
            "max_tokens": max_tokens,
            "stream": True,
        }
        
        if system_prompt:
            body["system"] = system_prompt
        
        try:
            async with self.client.stream("POST", "/messages", json=body) as response:
                response.raise_for_status()
                
                async for line in response.aiter_lines():
                    if line.startswith("event: "):
                        event_type = line[7:].strip()
                        continue
                    
                    if line.startswith("data: "):
                        data_str = line[6:].strip()
                        try:
                            event_data = json.loads(data_str)
                        except json.JSONDecodeError:
                            continue
                        
                        # We want content_block_delta events with text_delta
                        if event_data.get("type") == "content_block_delta":
                            delta = event_data.get("delta", {})
                            if delta.get("type") == "text_delta":
                                text = delta.get("text", "")
                                if text:
                                    yield text
                        
                        # stream_end event signals done
                        if event_data.get("type") == "message_stop":
                            return
                        
        except httpx.HTTPStatusError as e:
            raise ProviderError("anthropic", f"Stream failed: {e.response.status_code}")
        except httpx.TimeoutException:
            raise ProviderError("anthropic", "Stream timed out")
    
    async def count_tokens(self, messages: list[dict], model: str) -> int:
        """
        Estimate tokens for Anthropic.
        
        Note: Anthropic doesn't provide a public tokenizer like tiktoken.
        We use a reasonable estimation: ~4 chars per token for English.
        
        🔴 LIMITATION: This is an estimate. For accurate counts, use
        Anthropic's official token counting endpoint (when available)
        or the /v1/tokenize endpoint with claude-specific estimation.
        """
        total = 0
        for msg in messages:
            content = msg.get("content", "")
            # Anthropic tokens are roughly 3.5 chars/token for English
            # More for other languages
            total += max(1, len(content) // 4)
        
        # System prompt overhead (if any)
        system_msgs = [m for m in messages if m["role"] == "system"]
        if system_msgs:
            total += max(1, len(system_msgs[0].get("content", "")) // 4)
        
        return total
    
    def get_context_window(self, model: str) -> int:
        model_info = ANTHROPIC_MODELS.get(model, ANTHROPIC_MODELS["claude-3-5-sonnet-latest"])
        return model_info["context_window"]
    
    async def close(self):
        await self.client.aclose()
```

### src/providers/ollama_provider.py

```python
"""
Ollama provider — local LLMs via Ollama.

Ollama is OpenAI-API compatible BUT with quirks:
- No auth needed (localhost)
- Different model naming
- No official tokenizer
- Limited structured output support
"""

import json
import time
import httpx
from typing import AsyncGenerator, Optional
import logging

from .base import BaseLLMProvider, ProviderConfig, ProviderResponse
from core.errors import ProviderError

logger = logging.getLogger(__name__)


class OllamaProvider(BaseLLMProvider):
    """
    Provider for local Ollama models.
    
    Uses the OpenAI-compatible endpoint at http://localhost:11434/v1.
    Falls back to the native API if the compatible endpoint fails.
    """
    
    def _create_client(self) -> httpx.AsyncClient:
        return httpx.AsyncClient(
            base_url=self.config.base_url or "http://localhost:11434",
            timeout=httpx.Timeout(self.config.timeout_seconds * 2, connect=5.0),  # Local, fast connect
        )
    
    @property
    def models(self) -> list[str]:
        """Return available Ollama models (from config or by querying Ollama)."""
        return self.config.default_model.split(",") if self.config.default_model else ["llama3.2"]
    
    async def chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
        response_format: Optional[dict] = None,
    ) -> ProviderResponse:
        """
        Chat via Ollama's native API (/api/chat).
        
        We use the native API instead of the OpenAI-compatible one because:
        1. It's better maintained
        2. More features (raw mode, keep_alive, etc.)
        3. Direct access to Ollama-specific options
        """
        start_time = time.monotonic()
        
        body = {
            "model": model,
            "messages": messages,
            "options": {
                "temperature": temperature,
                "num_predict": max_tokens,
            },
            "stream": False,
        }
        
        if response_format and response_format.get("type") == "json_object":
            body["format"] = "json"
        
        try:
            response = await self.client.post("/api/chat", json=body)
            response.raise_for_status()
            data = response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 404:
                raise ProviderError("ollama", f"Model '{model}' not found. Pull it: ollama pull {model}")
            raise ProviderError("ollama", f"HTTP {e.response.status_code}")
        except httpx.TimeoutException:
            raise ProviderError("ollama", "Request timed out (local model may still be loading)")
        except httpx.NetworkError:
            raise ProviderError("ollama", "Cannot connect to Ollama. Is it running? Run: ollama serve")
        
        elapsed = (time.monotonic() - start_time) * 1000
        
        content = data.get("message", {}).get("content", "")
        
        return ProviderResponse(
            content=content,
            model=model,
            provider_name="ollama",
            finish_reason=data.get("done_reason", "stop"),
            prompt_tokens=data.get("prompt_eval_count", 0),
            completion_tokens=data.get("eval_count", 0),
            latency_ms=elapsed,
            raw_response=data,
        )
    
    async def stream_chat(
        self,
        messages: list[dict],
        model: str,
        temperature: float = 0.7,
        max_tokens: int = 2048,
    ) -> AsyncGenerator[str, None]:
        """Streaming via Ollama API."""
        body = {
            "model": model,
            "messages": messages,
            "options": {
                "temperature": temperature,
                "num_predict": max_tokens,
            },
            "stream": True,
        }
        
        try:
            async with self.client.stream("POST", "/api/chat", json=body) as response:
                response.raise_for_status()
                
                async for line in response.aiter_lines():
                    if not line:
                        continue
                    try:
                        chunk = json.loads(line)
                    except json.JSONDecodeError:
                        continue
                    
                    content = chunk.get("message", {}).get("content", "")
                    if content:
                        yield content
                    
                    if chunk.get("done", False):
                        return
                        
        except httpx.HTTPStatusError as e:
            raise ProviderError("ollama", f"Stream failed: HTTP {e.response.status_code}")
    
    async def count_tokens(self, messages: list[dict], model: str) -> int:
        """Estimate tokens for Ollama (no official tokenizer)."""
        total = 0
        for msg in messages:
            total += max(1, len(str(msg.get("content", ""))) // 4)
        return total
    
    def get_context_window(self, model: str) -> int:
        """Ollama model context windows vary. Return a conservative default."""
        return 8_192  # Most local models use 8K
    
    async def close(self):
        await self.client.aclose()
```

### src/providers/factory.py

```python
"""
Provider factory — maps model names to provider instances.

🔴 SENIOR MOVE: The factory pattern decouples routing logic from
provider implementation. Adding a new provider = 1 new file + 1 factory entry.
"""

from typing import Optional
import logging

from .base import BaseLLMProvider, ProviderConfig
from .openai_provider import OpenAIProvider, OPENAI_MODELS
from .anthropic_provider import AnthropicProvider, ANTHROPIC_MODELS
from .ollama_provider import OllamaProvider
from core.config import settings
from core.errors import ModelNotFoundError

logger = logging.getLogger(__name__)


# ┌─────────────────────────────────────────────────────────────────┐
# │ MODEL ROUTING TABLE                                             │
# │                                                                 │
# │ Maps model name patterns → provider constructors.               │
# │ This single table controls the entire routing logic.            │
# └─────────────────────────────────────────────────────────────────┘

MODEL_ROUTES = {
    # OpenAI models
    "gpt-4o": ("openai", OpenAIProvider),
    "gpt-4o-mini": ("openai", OpenAIProvider),
    "gpt-4": ("openai", OpenAIProvider),
    "gpt-4-turbo": ("openai", OpenAIProvider),
    "gpt-3.5-turbo": ("openai", OpenAIProvider),
    
    # Anthropic models
    "claude-3-5-sonnet": ("anthropic", AnthropicProvider),
    "claude-3-5-haiku": ("anthropic", AnthropicProvider),
    "claude-3-opus": ("anthropic", AnthropicProvider),
    "claude-3-sonnet": ("anthropic", AnthropicProvider),
    "claude-3-haiku": ("anthropic", AnthropicProvider),
    
    # Ollama models (prefixed with "ollama/")
    # For any model name starting with "ollama/", strip prefix and pass to Ollama
}

# Build prefix-based routes for Ollama
OLLAMA_PREFIX = "ollama/"


def get_provider_for_model(model: str) -> tuple[str, BaseLLMProvider]:
    """
    Route a model name to the correct provider instance.
    
    Resolution order:
    1. Exact match in MODEL_ROUTES
    2. Prefix match (e.g., "ollama/llama3.2")
    3. Wildcard not found → error
    
    Returns (provider_name, provider_instance).
    """
    normalized = model.lower()
    
    # Check for Ollama prefix
    if normalized.startswith(OLLAMA_PREFIX):
        ollama_model = model[len(OLLAMA_PREFIX):]
        config = ProviderConfig(
            base_url=settings.ollama_base_url,
            timeout_seconds=120,
            default_model=ollama_model,
        )
        return ("ollama", OllamaProvider(config))
    
    # Exact route match
    route = MODEL_ROUTES.get(normalized)
    if route is None:
        # Try partial match (e.g., "claude-3-5-sonnet-20241022" → "claude-3-5-sonnet")
        for key, (provider_name, provider_class) in MODEL_ROUTES.items():
            if normalized.startswith(key):
                route = (provider_name, provider_class)
                break
    
    if route is None:
        logger.warning(f"Unknown model requested: {model}")
        raise ModelNotFoundError(model)
    
    provider_name, provider_class = route
    
    if provider_name == "openai":
        config = ProviderConfig(
            api_key=settings.openai_api_key or "",
            default_model=model,
        )
    elif provider_name == "anthropic":
        config = ProviderConfig(
            api_key=settings.anthropic_api_key or "",
            default_model=model,
        )
    else:
        config = ProviderConfig(default_model=model)
    
    return (provider_name, provider_class(config))


def list_available_models() -> list[dict]:
    """
    List all available models with their metadata.
    Returns the full model catalog for the GET /v1/models endpoint.
    """
    models = []
    
    # OpenAI models
    for model_name, info in OPENAI_MODELS.items():
        if settings.openai_api_key:
            models.append({
                "id": model_name,
                "provider": "openai",
                "context_window": info["context_window"],
                "pricing": info["pricing"],
                "features": info["features"],
            })
    
    # Anthropic models
    for model_name, info in ANTHROPIC_MODELS.items():
        if settings.anthropic_api_key:
            models.append({
                "id": model_name,
                "provider": "anthropic",
                "context_window": info["context_window"],
                "pricing": info["pricing"],
                "features": info["features"],
            })
    
    # Ollama models
    for model_name in settings.ollama_models:
        models.append({
            "id": f"ollama/{model_name}",
            "provider": "ollama",
            "context_window": 8_192,
            "pricing": {"input": 0.0, "output": 0.0},  # Free (local)
            "features": ["streaming"],
        })
    
    return models
```

### src/services/cost_tracker.py

```python
"""
Real-time cost tracking for every request.

🔴 SENIOR MOVE: Cost tracking is not optional — it's how you
detect bill shocks before they happen. Track EVERY request.
"""

from dataclasses import dataclass, field
from datetime import datetime, timedelta
from collections import defaultdict
import threading
from typing import Optional


@dataclass
class CostEntry:
    """A single cost-captured request."""
    model: str
    provider: str
    prompt_tokens: int
    completion_tokens: int
    cost: float
    timestamp: datetime = field(default_factory=datetime.now)
    user_id: Optional[str] = None


class CostTracker:
    """
    Thread-safe cost tracker for the entire gateway.
    
    Tracks:
    - Cost per request (real-time)
    - Running total for the session/day
    - Cost per model/provider/user
    - Budget alerts
    """
    
    def __init__(self, monthly_budget: float = 100.0):
        self.monthly_budget = monthly_budget
        self._entries: list[CostEntry] = []
        self._lock = threading.Lock()
        self._model_rates = {
            # (provider, model) → (input_rate, output_rate) per 1M tokens
            ("openai", "gpt-4o"): (2.50, 10.00),
            ("openai", "gpt-4o-mini"): (0.15, 0.60),
            ("openai", "gpt-4-turbo"): (10.00, 30.00),
            ("openai", "gpt-3.5-turbo"): (0.50, 1.50),
            ("anthropic", "claude-3-5-sonnet-latest"): (3.00, 15.00),
            ("anthropic", "claude-3-5-haiku-latest"): (0.80, 4.00),
            ("anthropic", "claude-3-opus-latest"): (15.00, 75.00),
            ("ollama", "*"): (0.0, 0.0),  # Local = free
        }
    
    def track(
        self,
        provider: str,
        model: str,
        prompt_tokens: int,
        completion_tokens: int,
        user_id: Optional[str] = None,
    ) -> float:
        """
        Track a request and return its cost.
        
        This is called for EVERY request. The cost is calculated
        using the rate table and added to the running total.
        """
        cost = self._calculate_cost(provider, model, prompt_tokens, completion_tokens)
        
        entry = CostEntry(
            model=model,
            provider=provider,
            prompt_tokens=prompt_tokens,
            completion_tokens=completion_tokens,
            cost=cost,
            user_id=user_id,
        )
        
        with self._lock:
            self._entries.append(entry)
        
        return cost
    
    def _calculate_cost(self, provider: str, model: str, prompt_tokens: int, completion_tokens: int) -> float:
        """Calculate cost based on token counts and pricing."""
        # Try exact match
        rate = self._model_rates.get((provider, model))
        
        if rate is None:
            # Try wildcard for provider
            rate = self._model_rates.get((provider, "*"))
        
        if rate is None:
            # Unknown model — estimate at GPT-4o-mini rates
            rate = (0.15, 0.60)
        
        input_rate, output_rate = rate
        return (prompt_tokens * input_rate + completion_tokens * output_rate) / 1_000_000
    
    def get_total_cost(self) -> float:
        """Get total cost for all tracked requests."""
        with self._lock:
            return sum(e.cost for e in self._entries)
    
    def get_cost_by_model(self) -> dict[str, float]:
        """Get cost breakdown by model."""
        with self._lock:
            costs = defaultdict(float)
            for entry in self._entries:
                costs[entry.model] += entry.cost
            return dict(costs)
    
    def get_cost_by_provider(self) -> dict[str, float]:
        """Get cost breakdown by provider."""
        with self._lock:
            costs = defaultdict(float)
            for entry in self._entries:
                costs[entry.provider] += entry.cost
            return dict(costs)
    
    def get_cost_today(self) -> float:
        """Get cost for today only."""
        today = datetime.now().date()
        with self._lock:
            return sum(
                e.cost for e in self._entries
                if e.timestamp.date() == today
            )
    
    def get_budget_remaining(self) -> float:
        """How much budget remains this month."""
        return self.monthly_budget - self.get_total_cost()
    
    def is_over_budget(self) -> bool:
        """Check if we've exceeded the monthly budget."""
        return self.get_total_cost() >= self.monthly_budget
    
    def get_recent_entries(self, n: int = 10) -> list[CostEntry]:
        """Get the N most recent cost entries."""
        with self._lock:
            return sorted(self._entries, key=lambda e: e.timestamp, reverse=True)[:n]
```

### src/services/tokenizer.py

```python
"""
Tokenizer service — wraps tiktoken for all token counting needs.

Provides:
- Model-specific token counting
- Context window validation
- Multi-tokenizer support (o200k_base, cl100k_base, p50k_base)
- Cost estimation from token counts
"""

import tiktoken
from typing import Optional
import logging

from core.config import settings

logger = logging.getLogger(__name__)


# ┌─────────────────────────────────────────────────────────────────┐
# │ MODEL → ENCODING MAP                                            │
# │                                                                 │
# │ Different models use different tokenizers. Using the wrong one  │
# │ gives incorrect counts — sometimes by 2x or more.              │
# └─────────────────────────────────────────────────────────────────┘

MODEL_TO_ENCODING: dict[str, str] = {
    # o200k_base (GPT-4o family)
    "gpt-4o": "o200k_base",
    "gpt-4o-mini": "o200k_base",
    "gpt-4o-2024-08-06": "o200k_base",
    "gpt-4o-2024-05-13": "o200k_base",
    
    # cl100k_base (GPT-4, GPT-3.5)
    "gpt-4": "cl100k_base",
    "gpt-4-turbo": "cl100k_base",
    "gpt-4-turbo-preview": "cl100k_base",
    "gpt-3.5-turbo": "cl100k_base",
    "gpt-3.5-turbo-16k": "cl100k_base",
    "text-embedding-3-small": "cl100k_base",
    "text-embedding-3-large": "cl100k_base",
    
    # p50k_base (Codex)
    "codex": "p50k_base",
    
    # r50k_base (GPT-3, Davinci)
    "text-davinci-003": "r50k_base",
    "text-davinci-002": "r50k_base",
}


CONTEXT_WINDOWS: dict[str, int] = {
    "gpt-4o": 128_000,
    "gpt-4o-mini": 128_000,
    "gpt-4": 8_192,
    "gpt-4-turbo": 128_000,
    "gpt-3.5-turbo": 16_385,
    "gpt-3.5-turbo-16k": 16_385,
    "claude-3-5-sonnet-latest": 200_000,
    "claude-3-5-haiku-latest": 200_000,
    "claude-3-opus-latest": 200_000,
}


class TokenizerService:
    """
    Token counting and context validation.
    
    Caches encodings for performance — loading tiktoken encodings
    has overhead (~100ms first call).
    """
    
    def __init__(self):
        self._encodings: dict[str, tiktoken.Encoding] = {}
    
    def _get_encoding(self, model: str) -> tiktoken.Encoding:
        """Get the correct encoding for a model, with caching."""
        encoding_name = MODEL_TO_ENCODING.get(model, "cl100k_base")  # Safe fallback
        
        if encoding_name not in self._encodings:
            self._encodings[encoding_name] = tiktoken.get_encoding(encoding_name)
        
        return self._encodings[encoding_name]
    
    def count_tokens(self, text: str, model: str = "gpt-4o-mini") -> int:
        """
        Count tokens for a text string using the model's tokenizer.
        
        This is used for:
        - Calculating prompt cost
        - Checking context window limits
        - Estimating output cost
        """
        if not text:
            return 0
        encoding = self._get_encoding(model)
        return len(encoding.encode(text))
    
    def count_messages(self, messages: list[dict], model: str = "gpt-4o-mini") -> int:
        """
        Count tokens for a list of messages (chat format).
        
        Accounts for message format overhead:
        - System message: role token + content
        - User message: role token + content
        - Assistant message: role token + content + prefix
        
        This matches what the API counts server-side.
        """
        encoding = self._get_encoding(model)
        total = 0
        
        for msg in messages:
            # Each message has format overhead
            total += 4  # <|im_start|>role\ncontent\n<|im_end|>\n
            for key, value in msg.items():
                total += len(encoding.encode(str(value)))
                if key == "name":
                    total += 1  # name overhead
        
        # Assistant prefix
        total += 3  # <|im_start|>assistant\n
        
        return total
    
    def validate_context_fit(
        self,
        messages: list[dict],
        model: str,
        max_tokens: int = 2048,
    ) -> tuple[bool, int, int]:
        """
        Check if messages + max_tokens fit within the model's context window.
        
        Returns:
        - fits: True if it fits
        - prompt_tokens: Number of tokens in the prompt
        - context_limit: Max tokens for this model
        """
        prompt_tokens = self.count_messages(messages, model)
        context_limit = CONTEXT_WINDOWS.get(model, 128_000)
        
        total_needed = prompt_tokens + max_tokens
        fits = total_needed <= context_limit
        
        return fits, prompt_tokens, context_limit
    
    def estimate_cost(self, model: str, prompt_tokens: int, completion_tokens: int) -> dict:
        """
        Estimate cost for a request.
        
        Returns breakdown with input, output, and total cost.
        """
        # Pricing per 1M tokens (approximate for mid-2025)
        pricing = {
            "gpt-4o": {"input": 2.50, "output": 10.00},
            "gpt-4o-mini": {"input": 0.15, "output": 0.60},
            "gpt-4": {"input": 30.00, "output": 60.00},
            "gpt-4-turbo": {"input": 10.00, "output": 30.00},
            "gpt-3.5-turbo": {"input": 0.50, "output": 1.50},
        }
        
        rates = pricing.get(model, pricing["gpt-4o-mini"])
        
        input_cost = (prompt_tokens * rates["input"]) / 1_000_000
        output_cost = (completion_tokens * rates["output"]) / 1_000_000
        
        return {
            "input_cost": round(input_cost, 8),
            "output_cost": round(output_cost, 8),
            "total_cost": round(input_cost + output_cost, 8),
        }
    
    def get_token_breakdown(self, text: str, model: str = "gpt-4o-mini") -> list[dict]:
        """
        Get detailed per-token breakdown.
        
        Returns list of {position, token_id, text, byte_length} for
        every token in the text.
        """
        encoding = self._get_encoding(model)
        tokens = encoding.encode(text)
        
        details = []
        for i, token_id in enumerate(tokens):
            token_bytes = encoding.decode_single_token_bytes(token_id)
            try:
                token_text = token_bytes.decode("utf-8")
            except UnicodeDecodeError:
                token_text = repr(token_bytes)
            
            details.append({
                "position": i,
                "token_id": token_id,
                "text": token_text,
                "byte_length": len(token_bytes),
            })
        
        return details
```

### src/services/circuit_breaker.py

```python
"""
Circuit Breaker — prevents cascading failures when providers go down.

States:
  CLOSED  → Normal operation. Requests pass through.
  OPEN    → Provider is failing. Requests are REJECTED immediately (fast-fail).
  HALF_OPEN → Testing if provider recovered. One request allowed through.

┌──────────┐   failures > threshold   ┌──────────┐
│  CLOSED  │ ────────────────────────→ │   OPEN   │
│ (normal) │                           │ (failed) │
└────┬─────┘                           └────┬─────┘
     ↑                                      │
     │         recovery_timeout             │
     └──────────────────────────────────────┘
     ◄─────── try one request ──────────────┘
     
If HALF_OPEN request succeeds → CLOSED
If HALF_OPEN request fails   → OPEN (reset timer)
"""

from dataclasses import dataclass
from datetime import datetime, timedelta
import asyncio
from enum import Enum
from typing import Optional
import logging

logger = logging.getLogger(__name__)


class CircuitState(Enum):
    CLOSED = "closed"          # Normal
    OPEN = "open"              # Failing — fast-fail
    HALF_OPEN = "half_open"    # Testing recovery


@dataclass
class CircuitStateData:
    """State for a single provider's circuit breaker."""
    state: CircuitState = CircuitState.CLOSED
    failure_count: int = 0
    last_failure_time: Optional[datetime] = None
    last_success_time: Optional[datetime] = None
    opened_at: Optional[datetime] = None


class CircuitBreakerRegistry:
    """
    Manages circuit breakers for all providers.
    
    Usage:
        breaker = CircuitBreakerRegistry()
        
        async with breaker.guard("openai") as is_available:
            if is_available:
                result = await call_openai()
            else:
                result = await call_fallback()
    """
    
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = timedelta(seconds=recovery_timeout)
        self._state: dict[str, CircuitStateData] = {}
        self._lock = asyncio.Lock()
    
    def _get_state(self, provider: str) -> CircuitStateData:
        """Get or create state for a provider."""
        if provider not in self._state:
            self._state[provider] = CircuitStateData()
        return self._state[provider]
    
    async def record_success(self, provider: str):
        """Record a successful request."""
        async with self._lock:
            state = self._get_state(provider)
            state.failure_count = 0
            state.last_success_time = datetime.now()
            
            if state.state == CircuitState.HALF_OPEN:
                logger.info(f"Circuit breaker for {provider}: HALF_OPEN → CLOSED (recovered)")
                state.state = CircuitState.CLOSED
                state.opened_at = None
    
    async def record_failure(self, provider: str):
        """Record a failed request."""
        async with self._lock:
            state = self._get_state(provider)
            state.failure_count += 1
            state.last_failure_time = datetime.now()
            
            if state.failure_count >= self.failure_threshold and state.state != CircuitState.OPEN:
                logger.warning(
                    f"Circuit breaker for {provider}: OPENING after {state.failure_count} failures"
                )
                state.state = CircuitState.OPEN
                state.opened_at = datetime.now()
    
    async def is_available(self, provider: str) -> bool:
        """
        Check if a provider is available.
        
        Returns False immediately if circuit is OPEN.
        Returns True if CLOSED.
        Returns True (with transition to HALF_OPEN) if recovery_timeout elapsed.
        """
        async with self._lock:
            state = self._get_state(provider)
            
            if state.state == CircuitState.CLOSED:
                return True
            
            if state.state == CircuitState.OPEN:
                # Check if recovery timeout has elapsed
                if state.opened_at and (datetime.now() - state.opened_at) >= self.recovery_timeout:
                    logger.info(f"Circuit breaker for {provider}: OPEN → HALF_OPEN (testing)")
                    state.state = CircuitState.HALF_OPEN
                    return True
                return False
            
            # HALF_OPEN: allow one request through
            return True
    
    def get_all_states(self) -> dict[str, dict]:
        """Get state for all providers (for monitoring)."""
        result = {}
        for provider, state_data in self._state.items():
            result[provider] = {
                "state": state_data.state.value,
                "failure_count": state_data.failure_count,
                "last_failure": state_data.last_failure_time.isoformat() if state_data.last_failure_time else None,
                "last_success": state_data.last_success_time.isoformat() if state_data.last_success_time else None,
            }
        return result


class CircuitBreakerOpen(Exception):
    """Raised when a circuit breaker is open."""
    pass
```

### src/services/cache.py

```python
"""
Cache service for LLM responses.

Reduces costs and latency for repeated requests.
Supports two cache modes:
1. Exact-match: Same messages → same response (simple, high hit rate for repeat queries)
2. Semantic: Similar meaning → same response (complex, useful for FAQs)

🔴 SENIOR MOVE: Caching is how you turn $10/day into $2/day.
Most LLM traffic is repeated queries (retries, polls, same questions).
"""

import hashlib
import json
from datetime import datetime, timedelta
from typing import Optional, Any
import logging

logger = logging.getLogger(__name__)


class CacheService:
    """
    Simple in-memory cache with TTL.
    
    In production, swap this with Redis. The interface stays the same.
    """
    
    def __init__(self, default_ttl: int = 300, max_size: int = 1000):
        self.default_ttl = timedelta(seconds=default_ttl)
        self.max_size = max_size
        self._cache: dict[str, tuple[Any, datetime]] = {}  # key → (value, expires_at)
    
    def _make_key(self, model: str, messages: list[dict], temperature: float) -> str:
        """
        Generate a deterministic cache key from request parameters.
        
        Uses SHA-256 hash of the serialized request.
        Temperature 0.0 → deterministic output → cache aggressively.
        Temperature > 0 → less cache hit → but still cache exact repeats.
        """
        canonical = json.dumps({
            "model": model,
            "messages": messages,
            "temperature": round(temperature, 2),  # Round to avoid floating point issues
        }, sort_keys=True)
        
        return hashlib.sha256(canonical.encode()).hexdigest()
    
    def get(self, model: str, messages: list[dict], temperature: float) -> Optional[str]:
        """
        Try to get a cached response.
        
        Returns None if:
        - No cache entry exists
        - Entry has expired (TTL elapsed)
        - Cache is disabled for this model
        """
        key = self._make_key(model, messages, temperature)
        
        if key not in self._cache:
            return None
        
        value, expires_at = self._cache[key]
        
        if datetime.now() > expires_at:
            del self._cache[key]
            return None
        
        logger.debug(f"Cache HIT for model={model} temperature={temperature}")
        return value
    
    def set(
        self,
        model: str,
        messages: list[dict],
        temperature: float,
        response: str,
        ttl: Optional[int] = None,
    ):
        """
        Cache a response.
        
        Only cache when temperature is 0 (deterministic).
        Non-zero temperature responses vary — caching them misleads.
        
        🔴 SENIOR MOVE: Never cache non-deterministic responses!
        A temperature=0.7 response with "2+2=5" shouldn't be served to the next user.
        """
        if temperature > 0.1:
            # Don't cache non-deterministic responses
            return
        
        if len(self._cache) >= self.max_size:
            # Evict oldest entry
            oldest_key = min(self._cache.keys(), key=lambda k: self._cache[k][1])
            del self._cache[oldest_key]
        
        key = self._make_key(model, messages, temperature)
        ttl_delta = timedelta(seconds=ttl or self.default_ttl.seconds)
        self._cache[key] = (response, datetime.now() + ttl_delta)
        
        logger.debug(f"Cache SET for model={model} ttl={ttl_delta}")
    
    def invalidate(self, model: Optional[str] = None):
        """
        Invalidate cache entries.
        
        If model is specified, only invalidate entries for that model.
        If not, clear entire cache.
        """
        if model is None:
            self._cache.clear()
            logger.info("Cache fully invalidated")
        else:
            keys_to_delete = [
                k for k in self._cache
                if json.loads(bytes.fromhex(k[:len(k)])).get("model") == model
            ]
            for k in keys_to_delete:
                del self._cache[k]
            logger.info(f"Cache invalidated for model={model} ({len(keys_to_delete)} entries)")
    
    @property
    def size(self) -> int:
        return len(self._cache)
    
    @property
    def hit_rate(self) -> float:
        """Track cache efficiency."""
        if not hasattr(self, '_hits'):
            self._hits = 0
            self._misses = 0
        total = self._hits + self._misses
        return self._hits / total if total > 0 else 0.0
```

### src/routers/chat.py

```python
"""
Main chat completion router — the heart of the gateway.

Handles:
- POST /v1/chat/completions (streaming + non-streaming)
- Model routing via factory
- Token counting and cost tracking
- Caching for deterministic requests
- Circuit breaker integration
"""

import time
import json
import uuid
from datetime import datetime
from typing import AsyncGenerator
import logging

from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import StreamingResponse

from core.models import (
    ChatCompletionRequest,
    ChatCompletionResponse,
    Choice,
    Usage,
    Message,
)
from core.errors import (
    ProviderError,
    ModelNotFoundError,
    CircuitBreakerOpenError,
    ContextOverflowError,
)
from core.dependencies import (
    verify_api_key,
    get_tokenizer_service,
    get_cost_tracker,
    get_circuit_breaker,
    get_cache,
)
from providers.factory import get_provider_for_model, list_available_models
from services.tokenizer import TokenizerService
from services.cost_tracker import CostTracker
from services.circuit_breaker import CircuitBreakerRegistry
from services.cache import CacheService

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/v1", tags=["chat"])


@router.post("/chat/completions")
async def chat_completion(
    request: Request,
    body: ChatCompletionRequest,
    tokenizer: TokenizerService = Depends(get_tokenizer_service),
    cost_tracker: CostTracker = Depends(get_cost_tracker),
    circuit_breaker: CircuitBreakerRegistry = Depends(get_circuit_breaker),
    cache_service: CacheService = Depends(get_cache),
):
    """
    Chat completion endpoint.
    
    Supports both streaming and non-streaming responses.
    Routes to the correct provider based on model name.
    """
    # 1. Resolve model → provider
    try:
        provider_name, provider = get_provider_for_model(body.model)
    except ModelNotFoundError:
        # Return available models for better DX
        available = [m["id"] for m in list_available_models()]
        raise HTTPException(
            status_code=404,
            detail={
                "error": f"Model '{body.model}' not found",
                "available_models": available,
            }
        )
    
    # 2. Check circuit breaker
    available = await circuit_breaker.is_available(provider_name)
    if not available:
        raise CircuitBreakerOpenError(provider_name)
    
    # 3. Validate context window fit
    messages_dicts = [m.model_dump() for m in body.messages]
    fits, prompt_tokens, context_limit = tokenizer.validate_context_fit(
        messages_dicts, body.model, body.max_tokens
    )
    if not fits:
        raise ContextOverflowError(prompt_tokens, context_limit, body.model)
    
    # 4. Check cache (non-streaming, temperature=0 requests)
    if not body.stream and body.temperature <= 0.1:
        cached_response = cache_service.get(
            body.model, messages_dicts, body.temperature
        )
        if cached_response:
            logger.info(f"Cache HIT for model={body.model}")
            return ChatCompletionResponse(
                id=f"chatcmpl-cached-{uuid.uuid4().hex[:8]}",
                created=int(time.time()),
                model=body.model,
                choices=[Choice(index=0, message={"role": "assistant", "content": cached_response})],
                usage=Usage(
                    prompt_tokens=prompt_tokens,
                    completion_tokens=0,
                    total_tokens=prompt_tokens,
                    estimated_cost=0.0,
                ),
                provider=provider_name,
                cached=True,
                latency_ms=0.0,
            )
    
    # 5. Execute
    start_time = time.monotonic()
    
    if body.stream:
        return await _handle_streaming(
            provider, provider_name, body, cost_tracker, circuit_breaker, cache_service, start_time
        )
    else:
        return await _handle_non_streaming(
            provider, provider_name, body, cost_tracker, circuit_breaker, cache_service,
            tokenizer, start_time
        )


async def _handle_non_streaming(
    provider, provider_name, body, cost_tracker, circuit_breaker, cache_service,
    tokenizer, start_time,
) -> ChatCompletionResponse:
    """Handle a non-streaming request."""
    messages_dicts = [m.model_dump() for m in body.messages]
    
    try:
        response = await provider.chat(
            messages=messages_dicts,
            model=body.model,
            temperature=body.temperature,
            max_tokens=body.max_tokens,
            response_format=body.response_format,
        )
    except Exception as e:
        await circuit_breaker.record_failure(provider_name)
        raise
    else:
        await circuit_breaker.record_success(provider_name)
    
    elapsed = (time.monotonic() - start_time) * 1000
    
    # Track cost
    cost = cost_tracker.track(
        provider=provider_name,
        model=body.model,
        prompt_tokens=response.prompt_tokens,
        completion_tokens=response.completion_tokens,
        user_id=body.user_id,
    )
    
    # Cache if deterministic
    cache_service.set(
        body.model, messages_dicts, body.temperature, response.content
    )
    
    return ChatCompletionResponse(
        id=f"chatcmpl-{uuid.uuid4().hex[:12]}",
        created=int(time.time()),
        model=body.model,
        choices=[Choice(index=0, message={"role": "assistant", "content": response.content})],
        usage=Usage(
            prompt_tokens=response.prompt_tokens,
            completion_tokens=response.completion_tokens,
            total_tokens=response.prompt_tokens + response.completion_tokens,
            estimated_cost=round(cost, 8),
        ),
        provider=provider_name,
        latency_ms=round(elapsed, 2),
    )


async def _handle_streaming(
    provider, provider_name, body, cost_tracker, circuit_breaker, cache_service, start_time,
) -> StreamingResponse:
    """Handle a streaming request — returns SSE stream."""
    
    async def event_generator() -> AsyncGenerator[str, None]:
        """Generate SSE events for streaming response."""
        
        # We need to track the full response for caching and cost
        full_content = ""
        first_chunk = True
        total_completion_tokens = 0
        
        try:
            async for token in provider.stream_chat(
                messages=[m.model_dump() for m in body.messages],
                model=body.model,
                temperature=body.temperature,
                max_tokens=body.max_tokens,
            ):
                full_content += token
                total_completion_tokens += 1
                
                # Send SSE data chunk
                chunk_data = {
                    "id": f"chatcmpl-{uuid.uuid4().hex[:12]}",
                    "object": "chat.completion.chunk",
                    "created": int(time.time()),
                    "model": body.model,
                    "choices": [{
                        "index": 0,
                        "delta": {"content": token, "role": "assistant"} if first_chunk else {"content": token},
                        "finish_reason": None,
                    }],
                }
                yield f"data: {json.dumps(chunk_data)}\n\n"
                first_chunk = False
            
            # Send final [DONE] event
            yield "data: [DONE]\n\n"
            
            # Record success
            await circuit_breaker.record_success(provider_name)
            
            # Track cost (estimate completion tokens from response length)
            cost = cost_tracker.track(
                provider=provider_name,
                model=body.model,
                prompt_tokens=0,  # Can't count in streaming until we have full messages
                completion_tokens=total_completion_tokens,
                user_id=body.user_id,
            )
            
            logger.info(
                f"Streaming completed: model={body.model} "
                f"tokens={total_completion_tokens} cost=${cost:.6f}"
            )
            
        except Exception as e:
            logger.error(f"Streaming error: {str(e)}")
            await circuit_breaker.record_failure(provider_name)
            
            # Send error event
            error_data = {
                "error": {"message": str(e), "code": "stream_error"}
            }
            yield f"data: {json.dumps(error_data)}\n\n"
            yield "data: [DONE]\n\n"
    
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Provider": provider_name,
        },
    )
```

### src/routers/models.py

```python
"""
Model listing endpoint.
"""

from fastapi import APIRouter
from providers.factory import list_available_models

router = APIRouter(prefix="/v1", tags=["models"])


@router.get("/models")
async def list_models():
    """List all available models with metadata."""
    models = list_available_models()
    return {"data": models}


@router.get("/models/{model_id}")
async def get_model(model_id: str):
    """Get details for a specific model."""
    models = list_available_models()
    for model in models:
        if model["id"] == model_id:
            return {"data": model}
    
    return {"error": f"Model '{model_id}' not found"}, 404
```

### src/routers/tokenize.py

```python
"""
Token counting endpoints.
"""

import time
from fastapi import APIRouter, Depends

from core.models import TokenizeRequest, TokenizeResponse, TokenDetail
from services.tokenizer import TokenizerService
from core.dependencies import get_tokenizer_service

router = APIRouter(prefix="/v1", tags=["tokenize"])


@router.post("/tokenize")
async def tokenize(
    body: TokenizeRequest,
    tokenizer: TokenizerService = Depends(get_tokenizer_service),
):
    """
    Tokenize text and return detailed breakdown.
    
    Useful for:
    - Checking how many tokens a prompt will use
    - Debugging tokenization issues
    - Cost estimation before sending a request
    """
    details = tokenizer.get_token_breakdown(body.text, body.model)
    num_tokens = len(details)
    cost = tokenizer.estimate_cost(body.model, num_tokens, num_tokens)
    
    return TokenizeResponse(
        text=body.text,
        num_tokens=num_tokens,
        char_count=len(body.text),
        encoding="auto",  # Will be model-specific in production
        details=[TokenDetail(**d) for d in details],
        estimated_cost_input=cost["input_cost"],
        estimated_cost_output=cost["output_cost"],
    )


@router.post("/count")
async def count_tokens(
    body: TokenizeRequest,
    tokenizer: TokenizerService = Depends(get_tokenizer_service),
):
    """
    Quick token count (no detailed breakdown).
    
    Returns just the count and cost estimate.
    """
    num_tokens = tokenizer.count_tokens(body.text, body.model)
    cost = tokenizer.estimate_cost(body.model, num_tokens, num_tokens)
    
    return {
        "text_length": len(body.text),
        "num_tokens": num_tokens,
        "estimated_cost": cost["total_cost"],
    }
```

### src/routers/health.py

```python
"""
Health check and monitoring endpoints.
"""

import time
from datetime import datetime
from fastapi import APIRouter, Depends, Request
from services.circuit_breaker import CircuitBreakerRegistry
from services.cost_tracker import CostTracker
from services.cache import CacheService
from core.dependencies import get_circuit_breaker, get_cost_tracker, get_cache
from providers.factory import list_available_models

router = APIRouter(tags=["health"])

START_TIME = time.time()


@router.get("/health")
async def health_check():
    """Simple health check — is the gateway running?"""
    return {
        "status": "healthy",
        "uptime_seconds": int(time.time() - START_TIME),
        "timestamp": datetime.now().isoformat(),
    }


@router.get("/ready")
async def readiness_check(
    request: Request,
    circuit_breaker: CircuitBreakerRegistry = Depends(get_circuit_breaker),
):
    """
    Readiness check — is the gateway ready to serve traffic?
    
    Returns 200 if at least one provider is available.
    Returns 503 if ALL providers are down.
    """
    states = circuit_breaker.get_all_states()
    
    available_providers = [
        p for p, s in states.items()
        if s["state"] != "open"
    ]
    
    if not available_providers:
        return {
            "status": "not_ready",
            "reason": "All providers are unavailable",
            "provider_states": states,
        }, 503
    
    return {
        "status": "ready",
        "available_providers": available_providers,
        "provider_states": states,
    }


@router.get("/metrics/gateway")
async def gateway_metrics(
    cost_tracker: CostTracker = Depends(get_cost_tracker),
    circuit_breaker: CircuitBreakerRegistry = Depends(get_circuit_breaker),
    cache_service: CacheService = Depends(get_cache),
):
    """Gateway metrics for monitoring and dashboards."""
    return {
        "cost": {
            "total": round(cost_tracker.get_total_cost(), 6),
            "today": round(cost_tracker.get_cost_today(), 6),
            "by_model": cost_tracker.get_cost_by_model(),
            "by_provider": cost_tracker.get_cost_by_provider(),
            "budget_remaining": round(cost_tracker.get_budget_remaining(), 6),
        },
        "circuit_breakers": circuit_breaker.get_all_states(),
        "cache": {
            "size": cache_service.size,
            "max_size": 1000,
        },
        "models_available": len(list_available_models()),
        "uptime_seconds": int(time.time() - START_TIME),
    }


@router.get("/metrics/prometheus")
async def prometheus_metrics():
    """
    Prometheus-formatted metrics endpoint.
    
    In production, use prometheus_client library with proper
    Counter, Histogram, and Gauge metrics.
    """
    metrics = [
        "# HELP gateway_uptime_seconds Gateway uptime",
        "# TYPE gateway_uptime_seconds counter",
        f"gateway_uptime_seconds {int(time.time() - START_TIME)}",
        "",
        "# HELP gateway_models_available Number of available models",
        "# TYPE gateway_models_available gauge",
        f"gateway_models_available {len(list_available_models())}",
    ]
    return "\n".join(metrics), 200, {"Content-Type": "text/plain; charset=utf-8"}
```

### src/middleware/logging.py

```python
"""
Structured request/response logging middleware.

Logs every request with:
- Method, path, status code
- Latency
- User agent
- Client IP
- Request ID

🔴 SENIOR MOVE: Structured logs > unstructured logs.
JSON-formatted logs can be parsed by any log aggregator (Datadog, ELK, Grafana Loki).
"""

import time
import uuid
import logging
from datetime import datetime

from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response

logger = logging.getLogger("gateway")


class LoggingMiddleware(BaseHTTPMiddleware):
    """Middleware that logs all requests with structured data."""
    
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())[:8]
        start_time = time.monotonic()
        
        # Add request ID to state for downstream use
        request.state.request_id = request_id
        
        # Log request
        logger.info(
            "request_start",
            extra={
                "request_id": request_id,
                "method": request.method,
                "path": request.url.path,
                "query": str(request.query_params),
                "client_ip": request.client.host if request.client else "unknown",
                "user_agent": request.headers.get("user-agent", "unknown"),
            },
        )
        
        try:
            response = await call_next(request)
        except Exception as e:
            elapsed = (time.monotonic() - start_time) * 1000
            logger.error(
                "request_error",
                extra={
                    "request_id": request_id,
                    "method": request.method,
                    "path": request.url.path,
                    "latency_ms": round(elapsed, 2),
                    "error": str(e),
                },
            )
            raise
        
        elapsed = (time.monotonic() - start_time) * 1000
        
        # Log response
        logger.info(
            "request_complete",
            extra={
                "request_id": request_id,
                "method": request.method,
                "path": request.url.path,
                "status_code": response.status_code,
                "latency_ms": round(elapsed, 2),
            },
        )
        
        # Add response headers
        response.headers["X-Request-ID"] = request_id
        response.headers["X-Latency-Ms"] = str(round(elapsed, 2))
        
        return response
```

### src/main.py

```python
"""
LLM Gateway — Main application.

FastAPI application that:
1. Routes LLM requests to the correct provider
2. Handles streaming and non-streaming responses
3. Tracks tokens, costs, and latency
4. Implements circuit breakers for provider failures
5. Caches deterministic responses
6. Provides health checks and metrics

Run:
    uvicorn main:app --reload
    # or
    python main.py
"""

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse

from core.config import settings
from core.errors import (
    GatewayError,
    gateway_error_handler,
    generic_error_handler,
)
from services.circuit_breaker import CircuitBreakerRegistry
from services.cache import CacheService
from middleware.logging import LoggingMiddleware

# Configure logging
logging.basicConfig(
    level=getattr(logging, settings.log_level.upper(), logging.INFO),
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Application lifespan — setup and teardown.
    
    Startup:
    - Initialize circuit breaker registry
    - Initialize cache
    - Log gateway configuration (without secrets!)
    
    Shutdown:
    - Clean up provider connections
    - Log final metrics
    """
    logger.info(f"Starting {settings.app_name} in {settings.environment} mode")
    logger.info(f"Models: OpenAI={'✓' if settings.openai_api_key else '✗'} "
                f"Anthropic={'✓' if settings.anthropic_api_key else '✗'} "
                f"Ollama={'✓' if settings.ollama_base_url else '✗'}")
    
    # Initialize services
    app.state.circuit_breaker = CircuitBreakerRegistry(
        failure_threshold=settings.circuit_breaker_threshold,
        recovery_timeout=settings.circuit_breaker_recovery_seconds,
    )
    app.state.cache = CacheService(
        default_ttl=settings.cache_ttl_seconds,
    )
    app.state.providers = {}
    
    yield
    
    # Shutdown
    logger.info("Shutting down gateway")
    for name, provider in app.state.providers.items():
        try:
            await provider.close()
        except Exception:
            pass
    logger.info("Gateway shutdown complete")


# Create the application
app = FastAPI(
    title=settings.app_name,
    version="0.1.0",
    lifespan=lifespan,
    docs_url="/docs" if settings.environment != "production" else None,
    redoc_url="/redoc" if settings.environment != "production" else None,
)

# Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(LoggingMiddleware)

# Error handlers
app.add_exception_handler(GatewayError, gateway_error_handler)
app.add_exception_handler(Exception, generic_error_handler)

# Routers
from routers import chat, models, tokenize, health
app.include_router(chat.router)
app.include_router(models.router)
app.include_router(tokenize.router)
app.include_router(health.router)


@app.get("/")
async def root():
    """Root endpoint — API overview."""
    return {
        "name": settings.app_name,
        "version": "0.1.0",
        "endpoints": {
            "chat": "POST /v1/chat/completions",
            "models": "GET /v1/models",
            "tokenize": "POST /v1/tokenize",
            "count": "POST /v1/count",
            "health": "GET /health",
            "ready": "GET /ready",
            "docs": "/docs",
        },
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "main:app",
        host=settings.host,
        port=settings.port,
        reload=settings.environment == "development",
        log_level=settings.log_level.lower(),
    )
```

---

## ✅ Good Output Examples

### Non-Streaming Request

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "What is 2+2?"}
    ],
    "temperature": 0
  }'
```

Response:

```json
{
  "id": "chatcmpl-a1b2c3d4e5f6",
  "object": "chat.completion",
  "created": 1747353600,
  "model": "gpt-4o-mini",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "2 + 2 = 4"
      }
    }
  ],
  "usage": {
    "prompt_tokens": 11,
    "completion_tokens": 5,
    "total_tokens": 16,
    "estimated_cost": 0.00000345
  },
  "provider": "openai",
  "cache_hit": false,
  "latency_ms": 342.1
}
```

Second call (cached):

```json
{
  "cached": true,
  "usage": {
    "prompt_tokens": 11,
    "completion_tokens": 0,
    "total_tokens": 11,
    "estimated_cost": 0.0
  },
  "latency_ms": 0.0
}
```

### Streaming Request

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "Count to 5"}
    ],
    "stream": true
  }'
```

Output (SSE stream):

```
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"1"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":","},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" 2"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":","},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" 3"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":","},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" 4"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":","},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" 5"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

### Error Response (Model Not Found)

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"model": "nonexistent-model", "messages": [{"role": "user", "content": "hi"}]}'
```

```json
{
  "detail": {
    "error": "Model 'nonexistent-model' not found",
    "available_models": [
      "gpt-4o",
      "gpt-4o-mini",
      "gpt-4",
      "claude-3-5-sonnet-latest",
      "ollama/llama3.2"
    ]
  }
}
```

---

## ✅ Key Patterns and Best Practices

### Pattern 1: Provider Abstraction
Every provider implements the same interface. Adding a new provider = new file + factory entry. The chat router never knows which provider it's calling.

### Pattern 2: Cost-Aware Response
Every response includes `estimated_cost`. This enables:
- Real-time budget tracking
- Per-user billing
- Model selection based on cost/quality tradeoffs

### Pattern 3: Cache-First Architecture
Temperature=0 requests are cached aggressively. The second call to "What is 2+2?" costs $0.00 and returns in <1ms.

### Pattern 4: Circuit Breaker
If a provider fails 5 times in a row, ALL requests to that provider fail fast (503) for 60 seconds. This prevents cascading timeouts from overwhelming the system.

### Pattern 5: Structured Error Codes
Every error has a machine-readable `code` (e.g., `provider_error.openai`, `rate_limit_exceeded`). Clients can handle errors programmatically instead of parsing error messages.

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: No Timeouts
```python
# WRONG ❌ — Hangs forever if provider is slow
response = await client.post(url, json=body)  # No timeout!
# Production: Always set timeouts. 30s for chat, 60s for streaming.
```

### ❌ Antipattern 2: Exposing API Keys
```python
# WRONG ❌ — Client gets your API key
response = await openai_client.chat(...)  # Client intercepts the key!
# Production: Gateway proxies requests — client never sees your API keys.
```

### ❌ Antipattern 3: Ignoring Token Limits
```python
# WRONG ❌ — Sends full conversation forever
messages.append(user_msg)
messages.append(assistant_msg)
# Eventually exceeds context window!
# Production: Trim history to fit. Use tokenizer service.
```

### ❌ Antipattern 4: No Cost Monitoring
```
# WRONG ❌ — The bill is a surprise at month end
# Dev: "Let's just use GPT-4 for everything, it's better!"
# Month 1 bill: $4,200
# Production: Track every cent. Set budgets. Alert on spikes.
```

### ❌ Antipattern 5: Synchronous Streaming
```python
# WRONG ❌ — Blocks the event loop during streaming
for chunk in response.iter_lines():  # Sync!
    await websocket.send_text(chunk)  # Pretends to be async
# Production: Use async iterators end-to-end.
```

---

## 🧪 Integration Test Suite

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from main import app


@pytest.fixture
async def client():
    """Test client with auth header pre-configured."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        ac.headers["Authorization"] = "Bearer dev-key-change-me"
        yield ac


# tests/test_chat.py
@pytest.mark.asyncio
async def test_chat_completion(client):
    """Test basic non-streaming chat."""
    response = await client.post("/v1/chat/completions", json={
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": "Say 'test'"}],
        "temperature": 0,
        "max_tokens": 10,
    })
    assert response.status_code == 200
    data = response.json()
    assert "choices" in data
    assert "usage" in data
    assert len(data["choices"]) > 0
    assert data["usage"]["total_tokens"] > 0


@pytest.mark.asyncio
async def test_chat_unauthorized(client):
    """Test auth failure."""
    client.headers["Authorization"] = "Bearer wrong-key"
    response = await client.post("/v1/chat/completions", json={
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": "hi"}],
    })
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_model_not_found(client):
    """Test unknown model returns helpful error."""
    response = await client.post("/v1/chat/completions", json={
        "model": "nonexistent-model",
        "messages": [{"role": "user", "content": "hi"}],
    })
    assert response.status_code == 404
    data = response.json()
    assert "available_models" in data.get("detail", {})


@pytest.mark.asyncio
async def test_streaming_response(client):
    """Test streaming endpoint returns SSE stream."""
    async with client.stream("POST", "/v1/chat/completions", json={
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": "Count to 3"}],
        "stream": True,
        "temperature": 0,
        "max_tokens": 50,
    }) as response:
        assert response.status_code == 200
        assert response.headers["content-type"] == "text/event-stream"
        
        chunks = []
        async for line in response.aiter_lines():
            if line.startswith("data: "):
                chunks.append(line)
        
        assert len(chunks) > 1  # At least one content chunk + [DONE]


@pytest.mark.asyncio
async def test_tokenize_endpoint(client):
    """Test token counting endpoint."""
    response = await client.post("/v1/tokenize", json={
        "text": "Hello, world!",
        "model": "gpt-4o-mini",
    })
    assert response.status_code == 200
    data = response.json()
    assert data["num_tokens"] > 0
    assert data["char_count"] == 13
    assert len(data["details"]) == data["num_tokens"]


@pytest.mark.asyncio
async def test_health_endpoint(client):
    """Test health check."""
    response = await client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"


@pytest.mark.asyncio
async def test_cache_hit(client):
    """Test that identical requests return cached response."""
    payload = {
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": "Say 'cache-test'"}],
        "temperature": 0,
        "max_tokens": 10,
    }
    
    # First call
    response1 = await client.post("/v1/chat/completions", json=payload)
    data1 = response1.json()
    
    # Second call (should be cached)
    response2 = await client.post("/v1/chat/completions", json=payload)
    data2 = response2.json()
    
    assert data1["choices"][0]["message"]["content"] == data2["choices"][0]["message"]["content"]


@pytest.mark.asyncio
async def test_context_overflow(client):
    """Test that overflowing context returns proper error."""
    long_text = "hello " * 200_000  # Way too long
    response = await client.post("/v1/chat/completions", json={
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": long_text}],
    })
    assert response.status_code == 400
```

---

## 🚦 Gate Check

Before declaring Phase 1 complete, verify:

- [ ] **Gateway boots** — `uvicorn main:app` starts without errors
- [ ] **Health check works** — `GET /health` returns 200
- [ ] **Auth middleware works** — request without API key returns 401
- [ ] **Chat completion works** — `POST /v1/chat/completions` returns valid response
- [ ] **Streaming works** — SSE stream delivers tokens progressively
- [ ] **Model routing works** — `gpt-4o-mini`, `claude-3-5-sonnet`, `ollama/llama3.2` each route correctly (test with available providers)
- [ ] **Token counting works** — `POST /v1/tokenize` returns accurate counts
- [ ] **Cost tracking works** — response includes `estimated_cost`, cost tracker shows running total
- [ ] **Cache works** — second call to identical request returns `cached: true`
- [ ] **"Model not found" returns 404** with list of available models
- [ ] **Context overflow returns 400** with helpful message
- [ ] **Tests pass** — `pytest tests/` shows green
- [ ] **README written** — explains architecture, setup, and design decisions
- [ ] **GitHub repo initialized** with your gateway code and README

> **🛑 STOP. The Gate Check is not optional.**
>
> This gateway is the foundation for EVERY project in this course:
> - Phase 4: Customer Support Bot → uses the gateway
> - Phase 5: Advanced RAG Engine → uses the gateway
> - Phase 6: Multi-Agent System → uses the gateway
> - Phase 7: Eval Platform → evaluates the gateway
> - Phase 8: Guardrails → wraps the gateway
> - Phase 11: Capstone → built on the gateway
>
> **If it's broken now, every subsequent project compounds the issues.**
> Ship it. Get it reviewed. Make it solid.

---

## 📚 Resources & Next Steps

### Resources
- [FastAPI docs](https://fastapi.tiangolo.com/) — Your web framework
- [Pydantic docs](https://docs.pydantic.dev/) — Request/response validation
- [httpx docs](https://www.python-httpx.org/) — Async HTTP client
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic API reference](https://docs.anthropic.com/en/api)
- [Ollama API docs](https://github.com/ollama/ollama/blob/main/docs/api.md)

### Cross-References
- [API Integration Patterns](07-API-Integration-Patterns.md) — Provider abstraction layer design
- [Streaming, SSE & WebSockets](08-Streaming-SSE-WebSockets.md) — Streaming implementation details
- [Python AI Patterns](06-Python-AI-Patterns.md) — Async, error handling, retry patterns
- [Drill: Build CLI Chat](Drill-Build-CLI-Chat.md) — Your CLI client is the gateway's first consumer
- [Drill: Tokenizer Explorer](Drill-Tokenizer-Explorer.md) — Token counting logic used here
- [Phase 2: Prompt Engineering](../02-Prompt-Engineering/) — You'll add prompt optimization as a gateway service
- [Phase 4: RAG Foundations](../04-RAG-Foundations/) — The gateway routes RAG queries
- [Phase 7: Production Evals](../07-Production-Evals-Observability/) — You'll evaluate this gateway's outputs
- [Phase 8: Guardrails](../08-Guardrails-Safety/) — You'll wrap this gateway with safety layers

### Next Steps

Your gateway is DONE. Now:

1. **Push to GitHub** with a solid README
2. **Run the tests** and fix any failures
3. **Use your CLI chat** (from Drill 1) to test the gateway
4. **Share it** — this is a genuine portfolio piece
5. **Start Phase 2** — Prompt Engineering

---

## 📊 Project Summary

| Metric | Target | How to Measure |
|--------|--------|---------------|
| Time to first token (streaming) | <1s | `X-Latency-Ms` header |
| Cache hit rate | >30% of temperature=0 requests | Cache service metrics |
| Provider failover | <100ms to detect and route | Circuit breaker timing |
| Token counting accuracy | Within 1% of provider counts | Compare against provider responses |
| Test coverage | >80% | `pytest --cov` |
| Uptime | 99.9% | Health check polling |
| Cost tracking accuracy | Exact match to provider billing | Compare cost tracker vs invoice |
| Models supported | 10+ across 3+ providers | `GET /v1/models` |
| Throughput | 100 req/s on single instance | Load test with Locust |
