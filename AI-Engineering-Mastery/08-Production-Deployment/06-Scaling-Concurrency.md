# Scaling & Concurrency

## Vertical vs Horizontal Scaling

| Strategy | What | When |
|----------|------|------|
| Vertical | Bigger instance (more RAM/CPU/GPU) | Model doesn't fit, model requires GPU |
| Horizontal | More instances | Stateless API, RAG pipeline |
| Hybrid | Bigger GPUs + more API instances | Full stack (GPU + CPU services) |

## Auto-Scaling Strategy

```python
class AutoScaler:
    """Scale based on queue depth and latency."""

    def should_scale_up(self, metrics: dict) -> bool:
        # Scale up if:
        # - Queue depth > 100 AND p95 latency > 2s
        # - CPU > 80% for 5+ minutes
        return (metrics["queue_depth"] > 100 and metrics["p95_latency"] > 2000) \
            or metrics["cpu_avg_5min"] > 80

    def should_scale_down(self, metrics: dict) -> bool:
        # Scale down if:
        # - Queue depth < 10 AND p95 latency < 500ms
        # - CPU < 30% for 10+ minutes
        return metrics["queue_depth"] < 10 and metrics["p95_latency"] < 500 \
            and metrics["cpu_avg_10min"] < 30
```

## Async Processing for Heavy Workloads

```python
from celery import Celery

celery_app = Celery("ai-tasks", broker="redis://redis:6379/0")

@celery_app.task
def process_document_task(doc_id: str):
    """Background task for document ingestion."""
    doc = fetch_document(doc_id)
    chunks = chunk_document(doc)
    embeddings = embed_chunks(chunks)
    store_embeddings(embeddings)
    return {"status": "done", "chunks": len(chunks)}

# API endpoint just enqueues the task
@app.post("/documents/ingest")
async def ingest_document(doc_id: str):
    task = process_document_task.delay(doc_id)
    return {"task_id": task.id, "status": "queued"}
```

## Connection Pooling

```python
class DatabasePool:
    def __init__(self, dsn: str, min_size: int = 5, max_size: int = 20):
        self.pool = await asyncpg.create_pool(
            dsn,
            min_size=min_size,
            max_size=max_size,
            command_timeout=30,
        )

    async def execute(self, query: str, *args):
        async with self.pool.acquire() as conn:
            return await conn.execute(query, *args)
```

## 🔴 Senior: Where Bottlenecks Actually Happen

```
Most people optimize: Model inference (GPU)
Real bottleneck:       Tokenization → Embedding → Vector Search → LLM → Output Validation

Profile before you optimize! These are the usual suspects:
1. Network calls to vector DB (latency, not compute)
2. Serialization/deserialization of large payloads
3. Token counting on every request
4. Synchronous logging (always use async telemetry)
5. Missing connection pooling (opens/closes connections per request)
```
