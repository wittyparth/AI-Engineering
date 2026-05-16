# A/B Testing for AI

## The Problem

"Did this change improve the system?" — in traditional software, you can measure clicks, conversions, etc. In AI, quality is subjective and multi-dimensional.

## A/B Test Framework

```python
class ABTest:
    def __init__(self, control, variant, metric: str, min_sample: int = 100):
        self.control = control
        self.variant = variant
        self.metric = metric
        self.min_sample = min_sample
        self.results = {"control": [], "variant": []}

    async def run(self, test_cases: list[dict]) -> dict:
        for case in test_cases:
            # Route to both (offline A/B)
            c_result = await self.control(case)
            v_result = await self.variant(case)

            self.results["control"].append(self._score(c_result, case))
            self.results["variant"].append(self._score(v_result, case))

        return self._analyze()

    def _analyze(self) -> dict:
        import scipy.stats as stats

        control_scores = self.results["control"]
        variant_scores = self.results["variant"]

        t_stat, p_value = stats.ttest_ind(control_scores, variant_scores)

        return {
            "control_mean": mean(control_scores),
            "variant_mean": mean(variant_scores),
            "improvement": (mean(variant_scores) - mean(control_scores)) / mean(control_scores),
            "p_value": p_value,
            "significant": p_value < 0.05,
            "sample_size": len(control_scores),
        }
```

## Online A/B Testing

```python
class OnlineABTest:
    """Route traffic between control and variant in production."""

    def __init__(self, control, variant, traffic_split: float = 0.5):
        self.control = control
        self.variant = variant
        self.split = traffic_split  # % of traffic to variant
        self.metrics = {"control": {"requests": 0, "thumbs_up": 0},
                        "variant": {"requests": 0, "thumbs_up": 0}}

    async def route(self, request):
        if random.random() < self.split:
            response = await self.variant(request)
            self.metrics["variant"]["requests"] += 1
            variant_used = True
        else:
            response = await self.control(request)
            self.metrics["control"]["requests"] += 1
            variant_used = False

        return response, variant_used

    async def track_feedback(self, variant_used: bool, positive: bool):
        group = "variant" if variant_used else "control"
        self.metrics[group]["thumbs_up"] += 1 if positive else 0

    def get_results(self) -> dict:
        def satisfaction(m):
            return m["thumbs_up"] / m["requests"] if m["requests"] > 0 else 0

        return {
            "control_satisfaction": satisfaction(self.metrics["control"]),
            "variant_satisfaction": satisfaction(self.metrics["variant"]),
            "control_requests": self.metrics["control"]["requests"],
            "variant_requests": self.metrics["variant"]["requests"],
        }
```

## 🔴 Senior: A/B Pitfalls

| Pitfall | Problem | Fix |
|---------|---------|-----|
| Small sample size | Results not statistically significant | Wait for N > 100 per group |
| Multiple comparisons | Testing 10 metrics → 1 will be "significant" by chance | Bonferroni correction |
| Novelty effect | Users prefer new thing because it's new | Run for 7+ days |
| Selection bias | Different users in different groups | Random assignment |
| Metric dilution | Aggregating across very different query types | Segment by query type |
