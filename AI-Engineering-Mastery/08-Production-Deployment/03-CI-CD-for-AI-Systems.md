# CI/CD for AI Systems

## Why AI CI/CD Is Different

Traditional CI/CD:
```
Code change → tests → deploy
```

AI CI/CD:
```
Code change → tests → eval on benchmark → compare with baseline → if improved → deploy
             ↘ Model update → eval on benchmark → compare → if improved → deploy
```

## GitHub Actions Pipeline

```yaml
name: AI Service CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run unit tests
        run: pytest tests/ -v

  eval:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Run evaluation suite
        run: python -m eval.run --dataset eval/datasets/regression.json
      - name: Compare with baseline
        run: python -m eval.compare --baseline baseline.json --current results.json
      - name: Check quality gate
        run: |
          python -c "
          import json
          current = json.load(open('results.json'))
          baseline = json.load(open('baseline.json'))
          if current['faithfulness'] < baseline['faithfulness'] * 0.95:
              print('Quality regression detected!')
              exit(1)
          print('Quality gate passed')
          "

  deploy:
    needs: eval
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker image
        run: docker build -t rag-api:${{ github.sha }} .
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --password-stdin
          docker tag rag-api:${{ github.sha }} $ECR_REPO:latest
          docker push $ECR_REPO:latest
      - name: Deploy to ECS
        run: |
          aws ecs update-service --cluster ai-cluster --service rag-api \
            --force-new-deployment
```

## Canary Deployments

```yaml
# Stage 1: Deploy to 5% of traffic
- name: Canary deploy (5%)
  run: aws ecs update-service --cluster ai-cluster --service rag-api-canary ...

# Stage 2: Monitor for 10 minutes
- name: Monitor canary
  run: sleep 600 && python -m scripts.check_canary_health

# Stage 3: Roll forward or rollback
- name: Promote to 100%
  if: success()
  run: aws ecs update-service --cluster ai-cluster --service rag-api ...
```

## Eval Gate

```python
class EvalGate:
    """Quality gate that prevents deployment if eval metrics regress."""

    def __init__(self, threshold: float = 0.95):
        self.threshold = threshold

    def should_deploy(self, current: dict, baseline: dict) -> bool:
        for metric in ["faithfulness", "answer_relevancy", "context_precision"]:
            if current[metric] < baseline[metric] * self.threshold:
                logger.error(f"Gate failed: {metric} dropped from "
                           f"{baseline[metric]} to {current[metric]}")
                return False
        logger.info("All quality gates passed")
        return True
```

## 🔴 Senior: Deployment Risks Specific to AI

1. **Prompt changes**: A prompt change can silently break everything. Always eval.
2. **Model updates**: New model version might behave differently. Staged rollout.
3. **Data drift**: User queries change over time. Monitor and re-eval periodically.
4. **Cost changes**: New model may be more/less expensive. Track per-query cost.
5. **Latency regressions**: New code may be slower. Latency monitoring in CI.
