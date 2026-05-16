# Drill: Dockerize an LLM Service

**Time**: 30 min  **Difficulty**: Easy

## Task

Take a simple LLM API service and create a production-ready Docker setup.

## What to Build

```dockerfile
# Stage 1: Dependencies
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dirs -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Requirements

1. Multi-stage build
2. Non-root user
3. Health check
4. `.dockerignore` (no __pycache__, .git, .venv, tests)
5. Docker Compose with the LLM service + redis

## Verify

```bash
docker build -t llm-service .
docker run -p 8000:8000 llm-service
curl http://localhost:8000/health
# Should return 200
```
