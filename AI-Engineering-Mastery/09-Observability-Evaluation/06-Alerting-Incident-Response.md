# Alerting & Incident Response

## Alert Definition

```python
class AlertRule(BaseModel):
    name: str
    metric: str
    condition: str  # ">", "<", "=="
    threshold: float
    duration: str  # "5m", "10m"
    severity: str  # "warning", "critical"
    description: str

rules = [
    AlertRule(
        name="high_p95_latency",
        metric="rag_latency_p95",
        condition=">", threshold=5.0,
        duration="5m", severity="critical",
        description="p95 latency above 5s for 5 minutes",
    ),
    AlertRule(
        name="high_error_rate",
        metric="rag_error_rate",
        condition=">", threshold=0.05,
        duration="5m", severity="critical",
        description="Error rate above 5% for 5 minutes",
    ),
    AlertRule(
        name="low_cache_hit_rate",
        metric="cache_hit_ratio",
        condition="<", threshold=0.2,
        duration="30m", severity="warning",
        description="Cache hit rate below 20% for 30 minutes",
    ),
    AlertRule(
        name="approaching_cost_budget",
        metric="daily_cost",
        condition=">", threshold=0.8,
        duration="0m", severity="warning",
        description="Daily cost above 80% of budget",
    ),
]
```

## Incident Response Playbook

```yaml
incident: high_latency
severity: critical
symptoms:
  - p95 latency > 5s
  - User complaints about slow responses
  - Increased error rate

diagnosis:
  1. Check CloudWatch for CPU/memory spikes
  2. Check Langfuse for slow LLM calls
  3. Check vector DB latency
  4. Check if a new deployment happened recently
  5. Check if an upstream provider (OpenAI) has an outage

resolution:
  - If LLM provider issue: failover to alternate provider
  - If vector DB slow: reduce top-k, scale DB
  - If code issue: rollback to previous version
  - If traffic spike: trigger auto-scaling

post-mortem:
  - What broke?
  - Why wasn't it caught earlier?
  - What monitoring would have detected it sooner?
  - What's the fix to prevent recurrence?
```

## 🔴 Senior: On-Call for AI Systems

| Type of Alert | Response Time | Who |
|---------------|---------------|-----|
| All requests failing | 5 min | On-call engineer |
| p95 latency > 5s | 15 min | On-call engineer |
| Error rate > 5% | 15 min | On-call engineer |
| Cost spike > 20% above normal | 1 hour | On-call + lead |
| Quality score drop > 10% | 1 hour | ML engineer (next day) |
| Cache hit rate low | 1 day | Review and optimize |

**Note**: Not all alerts require immediate response. Design your on-call rotation accordingly.
