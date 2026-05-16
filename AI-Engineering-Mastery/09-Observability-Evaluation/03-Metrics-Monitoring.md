# Metrics & Monitoring

## Key Metrics

### Performance Metrics
```python
# Latency
rag_latency = Histogram("rag_latency_seconds", "RAG end-to-end latency",
    buckets=[0.1, 0.25, 0.5, 1, 2.5, 5, 10],
    labelnames=["model", "provider"])
# p50, p95, p99 latency per model/provider

# Throughput
rag_requests = Counter("rag_requests_total", "Total RAG requests",
    labelnames=["status", "model"])

# Errors
rag_errors = Counter("rag_errors_total", "RAG errors",
    labelnames=["error_type", "component"])
```

### Quality Metrics
```python
# User feedback
feedback_score = Histogram("rag_feedback_score", "User feedback (thumbs up=1, down=0)")

# Retrieval quality
retrieval_score = Histogram("rag_retrieval_score",
    "Retrieval relevance score distribution")

# Cost
cost_per_query = Histogram("rag_cost_per_query",
    "Cost per query in USD",
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5],
    labelnames=["model"])
```

## Prometheus + Grafana

```python
from prometheus_client import start_http_server, Histogram, Counter, Gauge
from prometheus_fastapi_instrumentator import Instrumentator

# Auto-instrument FastAPI
Instrumentator().instrument(app).expose(app)

# Custom metrics
RAG_LATENCY = Histogram("rag_latency_seconds", "RAG latency",
    buckets=(0.1, 0.5, 1.0, 2.0, 5.0, 10.0))

@app.post("/rag")
async def rag_query(request: Request):
    start = time.time()
    response = await handle_rag(request)
    RAG_LATENCY.observe(time.time() - start)
    return response
```

## Grafana Dashboard

```python
dashboard = {
    "panels": [
        {"title": "RAG Latency p50/p95/p99", "type": "graph", "target": "rag_latency_seconds"},
        {"title": "Requests per Minute", "type": "stat", "target": "rate(rag_requests_total[1m])"},
        {"title": "Error Rate by Component", "type": "pie", "target": "rag_errors_total"},
        {"title": "Cost per Query", "type": "gauge", "target": "rag_cost_per_query"},
        {"title": "Cache Hit Rate", "type": "stat", "target": "cache_hit_ratio"},
        {"title": "Token Usage by Model", "type": "bar", "target": "token_usage_total"},
        {"title": "User Feedback Score (7d avg)", "type": "stat", "target": "avg(rag_feedback_score[7d])"},
    ]
}
```

## 🔴 Senior: Alert Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| p95 latency | > 3s | > 5s |
| Error rate | > 1% | > 5% |
| Cache hit rate | < 20% | < 10% |
| Daily cost | > 80% budget | > 100% budget |
| User feedback avg | < 0.7 | < 0.5 |
| Health check failures | 1 instance | 3+ instances |
