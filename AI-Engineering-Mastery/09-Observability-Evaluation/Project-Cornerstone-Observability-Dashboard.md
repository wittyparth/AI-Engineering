# Cornerstone Project: Observability Dashboard

**Time**: 2-3 days  **Difficulty**: Medium  **Portfolio**: Yes

## The Problem

Your AI system is running in production. You have NO idea:
- What queries are users sending?
- How fast are responses?
- How much does it cost?
- Which queries are failing?
- Is quality improving or degrading?

Build an observability system that answers ALL of these.

## Requirements

### Must Have
- [ ] Langfuse tracing on every AI request (or OpenTelemetry)
- [ ] Custom metrics exposed (Prometheus format)
- [ ] Prometheus + Grafana stack configured
- [ ] Latency histograms (p50, p95, p99) by endpoint
- [ ] Error rate tracking by error type
- [ ] Token usage and cost tracking
- [ ] User feedback collection (thumbs up/down)
- [ ] Dashboard showing: latency, errors, cost, throughput, quality

### Should Have
- [ ] Cache hit/miss ratio tracking
- [ ] Model usage breakdown (% per model)
- [ ] Top-failed queries dashboard
- [ ] Query analytics (popular topics, zero-result queries)
- [ ] Alerting (Slack/email when error rate spikes)
- [ ] Automatic regression detection in eval pipeline

### Nice to Have
- [ ] Custom eval metrics in Grafana
- [ ] Cost forecast (projected monthly cost)
- [ ] Session-level traces (follow one user's experience)
- [ ] RAG-specific metrics (chunk relevance, reranking lift)

## Architecture

```
AI Service → OpenTelemetry SDK → Collector → Prometheus
                            ↘ Langfuse (trace viewer)
                                              ↓
User Feedback → PostgreSQL → Grafana (dashboards)
```

## Metrics to Export

```python
# Latency
latency = Histogram("ai_latency_seconds", [...],
    labelnames=["endpoint", "model", "provider"])

# Errors
errors = Counter("ai_errors_total", [...],
    labelnames=["endpoint", "error_type"])

# Cost
cost = Histogram("ai_cost_per_query", [...],
    labelnames=["model", "provider"])

# Quality (from evals)
quality = Gauge("ai_quality_score", [...],
    labelnames=["metric_name"])

# User feedback
feedback = Counter("ai_feedback_total", [...],
    labelnames=["endpoint", "positive"])
```

## Dashboard Sections

1. **Overview**: RPM, error rate, p95 latency, cost today
2. **Latency Breakdown**: p50/p95/p99 by endpoint
3. **Cost Analysis**: Cost by model, by user, forecast
4. **Error Analysis**: Error rate by type, top error messages
5. **Quality Trends**: Eval metrics over time (7d, 30d)
6. **User Feedback**: Thumbs up/down rate, trend
7. **Top Queries**: Most frequent, most expensive, most failed

## Drill: Incident Simulation

Simulate these incidents and verify your observability catches them:
1. Vector DB goes down → expect retrieval errors
2. Prompt change causes quality drop → expect eval metric regression
3. Traffic spike → expect latency increase + auto-scaling
4. Cache failure → expect increased cost + latency

## Design Decisions

1. **Trace backend**: Langfuse vs SigNoz vs Datadog vs Grafana Tempo?
2. **Metrics backend**: Prometheus vs Datadog vs CloudWatch?
3. **Dashboard**: Grafana vs Datadog vs custom?
4. **Alerting**: Grafana Alerting vs PagerDuty vs OpsGenie?
5. **Cost tracking**: Custom vs Langfuse built-in?
6. **Eval integration**: Automated vs manual vs hybrid?

## Reflection

1. What was the most useful metric you added?
2. What alert would have saved you from a real incident?
3. How did you distinguish "noise" from "signal" in your metrics?
4. What would you need to charge customers per-query?
