# Docker Containerization

## Multi-Stage Build

```dockerfile
# Stage 1: Build
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dirs -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY ./src ./src
COPY ./models ./models
ENV PATH=/root/.local/bin:$PATH

# Non-root user for security
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Docker Compose for AI Stack

```yaml
version: "3.8"
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on:
      - qdrant
      - redis
      - ollama
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  qdrant:
    image: qdrant/qdrant:latest
    volumes:
      - qdrant_data:/qdrant/storage
    ports: ["6333:6333"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes:
      - redis_data:/data

  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

volumes:
  qdrant_data:
  redis_data:
  ollama_data:
```

## 🔴 Senior: Production Docker Patterns

### 1. Health Checks
```python
@app.get("/health")
async def health():
    return {
        "status": "ok",
        "model_loaded": model.is_loaded,
        "vector_db_connected": qdrant_client.is_connected,
        "redis_connected": redis_client.ping(),
        "uptime": time.time() - start_time,
    }
```

### 2. Graceful Shutdown
```python
@app.on_event("shutdown")
async def shutdown():
    logger.info("Shutting down...")
    await qdrant_client.close()
    await redis_client.close()
    await httpx_client.aclose()
    logger.info("Shutdown complete")
```

### 3. Resource Limits
```yaml
services:
  api:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
```

## Drill: Dockerize Your Phase 4 RAG Project

1. Multi-stage build for the API
2. Docker Compose with all services
3. Health checks on every service
4. Graceful shutdown handling
5. Non-root user
6. `.dockerignore` to exclude unnecessary files
7. Verify: `docker compose up --build` starts everything cleanly
