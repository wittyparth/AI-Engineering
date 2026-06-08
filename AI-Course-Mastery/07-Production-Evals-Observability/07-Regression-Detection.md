# 07 — Regression Detection: Finding Problems Before Users Do

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about regression detection algorithms, you need to understand the fundamental challenge: distinguishing SIGNAL from NOISE in eval metrics.

You have an eval pipeline (File 05) that runs every day. It produces a faithfulness score. Here's the last 30 days:

```
Day:    1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30
Score: 92 93 91 90 92 94 91 90 88 91 92 90 89 91 93 92 90 88 87 86 85 84 83 82 81 80 79 78 77 76
```

**Look at this sequence carefully.** When did the regression start?

If you said "Day 19" — you see the downward trend from 87 onward. But on Day 19, the score was 87 — that's within the normal range of 88-94. You couldn't have confidently said "regression" on Day 19.

If you said "Day 24" — the score is 82, well below the normal range. But by Day 24, you've lost 10 points. Users have been getting worse answers for 5 days.

If you said "Day 14" — the score is 89, but looking backward, you can see it was the START of the decline. But on Day 14, with only the data up to that point, 89 looks like normal variation.

**The problem:** By the time a regression is VISIBLE (Day 24), it's already BEEN happening for significant time. By the time it's STATISTICALLY SIGNIFICANT, users are already noticing.

---

### 🤔 Discovery Question 1: The Gradual vs. Sudden Problem

Two regression patterns, both bad, both need DIFFERENT detection strategies:

**Sudden shift (bad deploy):**
```
Day:   1  2  3  4  5  6  7  8  9 10
Score: 92 93 91 92 94 91 90 92 91 93
                          ↑ DEPLOY
Day:  11 12 13 14 15 16 17 18 19 20
Score: 82 83 81 80 84 82 81 83 80 82
```

A deploy on Day 10 caused an INSTANT drop from ~92 to ~82. Clear signal. Easy to detect.

**Gradual drift (slow decay):**
```
Day:   1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
Score: 92 93 91 92 94 91 90 92 91 93 92 90 89 87 85
                              Wait, is that a trend?
```

No single deploy. The system is slowly degrading — maybe a data drift, maybe a model API change, maybe user behavior shifting. Each day's score is within normal range. But the 30-day trend is clearly down.

**🤔 Before reading on, answer these:**

1. For SUDDEN shifts: What detection method would catch this on Day 11 (the first day after the drop)? What if Day 11's score of 82 is WITHIN the normal range of your metric (which has variance of ±5)? How would you know it's a shift and not noise?

2. For GRADUAL drift: What detection method would catch the trend on Day 13 (when it first goes below 90)? What if you can't detect it until Day 18 (when it's clearly a trend but already 5 points down)? Is there a way to catch it EARLIER by looking at per-category scores instead of the aggregate?

3. **The hardest part:** What if BOTH patterns happen simultaneously? A sudden shift AND a gradual drift? Your system got a new model (sudden: quality changes immediately) AND user behavior is shifting (gradual: quality drifts over time). How do your detection algorithms distinguish between the two, and more importantly, how do you decide which to fix first?

---

### 🤔 Discovery Question 2: The Correlated Changes Problem

Your team makes TWO changes at the same time:
1. Updates the system prompt to be more concise
2. Changes the embedding model from text-embedding-3-small to text-embedding-3-large

The next eval run shows: faithfulness dropped from 0.92 to 0.86.

**🤔 Before reading on, answer these:**

1. Which change caused the regression? You can't tell — they were deployed together. Your pipeline catches the regression but can't DIAGNOSE the cause. Your team reverts BOTH changes. You lost the improvements from BOTH. How would you structure deployments to AVOID this problem?

2. Your PM says: "Just revert both, then try them one at a time." That's correct but SLOW. Each deploy cycle takes 2 hours (build + eval pipeline + staging). Reverting + redeploying both separately = 4+ hours. Is there a FASTER way to determine the cause? What diagnostic information could you collect DURING a deploy that would help isolate the cause without needing a second deploy?

3. You discover that Change 1 (prompt) improved relevance but hurt faithfulness. Change 2 (embedding) improved faithfulness but hurt retrieval recall. The combined effect is a net negative on faithfulness. But individually, EACH change improved something. How would your regression detection system account for this — where a change that improves one metric while hurting another might still be net-positive overall?

---

### 🤔 Discovery Question 3: The "Statistically Significant but Meaningless" Problem

Your eval pipeline runs 500 test cases per day. You detect a regression: faithfulness dropped from 0.920 to 0.912 — an 0.8% drop. With 500 samples, this is STATISTICALLY SIGNIFICANT (p < 0.01).

Your team investigates for 2 hours. Root cause: a prompt change that made answers slightly less verbose. The answers are still correct, still faithful. They're just shorter. The LLM judge penalized brevity (verbosity bias from File 02).

**🤔 Before reading on, answer these:**

1. A statistically significant 0.8% drop that took 2 hours to investigate and turned out to be irrelevant. How many of these false alarms can your team absorb before they start ignoring regression alerts?

2. How would you set a PRACTICAL SIGNIFICANCE threshold — not just "is this drop statistically significant?" but "is this drop LARGE ENOUGH to care about?" What determines the threshold? (Think about: user impact, cost impact, frequency, and reversibility.)

3. Your teammate proposes: "Let's just raise the alert threshold to 2% drop. That filters out the noise." But what if the next REAL regression is a 1.5% drop? You'd miss it. How do you set a threshold that filters NOISE without filtering SIGNAL?

---

> **Regression detection is not about finding every change. It's about finding CHANGES THAT MATTER, quickly enough to act, without overwhelming the team with false alarms.**

By the end of this module, you will:
- Implement statistical regression detection (CUSUM, moving average, Z-score)
- Distinguish gradual drift from sudden shifts
- Design deployment strategies that isolate change causes
- Build automated rollback triggers
- Set practical significance thresholds that balance sensitivity and noise

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Discovery Questions (above) | 25 min |
| Statistical Process Control for Metrics | 20 min |
| CUSUM for Shift Detection | 20 min |
| Moving Average for Trend Detection | 15 min |
| Deployment Strategies for Isolation | 20 min |
| Code: CUSUM Shift Detector | 30 min |
| Code: Moving Average Trend Detector | 20 min |
| Code: Deployment Isolation Analysis | 25 min |
| Code: Automated Rollback Trigger | 25 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 10 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. Statistical Process Control for Eval Metrics

The core idea: every metric has a "natural range" of variation due to noise (LLM randomness, user behavior, API variance). A regression is when the metric moves OUTSIDE this natural range.

```
UPPER CONTROL LIMIT (UCL): μ + 3σ   ← Score above this = unexpectedly good
MEAN (μ): historical average          ← Expected score
LOWER CONTROL LIMIT (LCL): μ - 3σ   ← Score below this = regression!

If 3 consecutive data points are outside 2σ → WARNING
If 1 data point is outside 3σ → ALERT
If 8 consecutive points are on same side of mean → TREND DETECTED
```

These are standard SPC rules from manufacturing. They apply DIRECTLY to AI eval metrics.

The challenge: LLM eval metrics are NOISIER than manufacturing measurements. A 3σ threshold might generate 1 false positive per 370 data points. If you measure 500 cases per day, that's 1.35 false positives per day — too many.

**🔧 Fix:** Use 3.5σ instead of 3σ for noisier metrics. Or use a wider control limit AND require 2 consecutive data points outside the limit.

### 2. CUSUM for Shift Detection

CUSUM (Cumulative Sum) detects SMALL persistent shifts that individual data points can't reveal.

```
For each data point:
  S_positive = max(0, S_previous + (x_t - μ - k))
  S_negative = max(0, S_previous - (x_t - μ + k))
  
  Where:
  - μ = target mean
  - k = allowable slack (usually 0.5σ)
  - h = decision threshold (usually 4-5σ)
  
  If S_positive > h → UPWARD shift detected
  If S_negative > h → DOWNWARD shift detected
```

CUSUM is excellent at detecting a 1-2% shift within 5-10 data points, where a simple threshold would need 20-30 data points.

---

## 💻 Code Examples

### Example 1: CUSUM Shift Detector

```python
"""
cusum_detector.py

CUSUM (Cumulative Sum) for detecting small persistent shifts in eval metrics.

Why CUSUM over simple threshold?
- A 1% drop in faithfulness might be invisible in a single data point
  (noise is ±3%, so 1% is within normal range)
- But if every day for 10 days the score is 0.5% below the mean,
  CUSUM accumulates this signal and flags it as a shift
- CUSUM detects shifts 2-3x faster than simple threshold methods
"""

from dataclasses import dataclass
from typing import Optional
import statistics
import math


@dataclass
class CUSUMConfig:
    target_mean: float       # The expected value
    target_std: float        # Expected standard deviation
    slack_k: float = 0.5     # Allowable slack (in std units)
    threshold_h: float = 5.0 # Decision threshold (in std units)
    
    @property
    def k_value(self) -> float:
        return self.slack_k * self.target_std
    
    @property
    def h_value(self) -> float:
        return self.threshold_h * self.target_std


@dataclass
class ShiftDetection:
    shift_detected: bool
    direction: str  # "up" | "down" | "none"
    cumulative_positive: float
    cumulative_negative: float
    days_since_shift: int
    estimated_shift_size: float  # How much did the mean change?


class CUSUMDetector:
    """
    CUSUM-based shift detection for eval metrics.
    
    Detects when a metric has shifted PERSISTENTLY (not just a spike).
    Ideal for detecting: model changes, prompt changes, data drift.
    """
    
    def __init__(self, config: CUSUMConfig):
        self.config = config
        self.values: list[float] = []
        self.cum_positive: list[float] = []
        self.cum_negative: list[float] = []
        self.mean = config.target_mean
        self.std = config.target_std
    
    @classmethod
    def from_history(cls, history: list[float], slack_k: float = 0.5, threshold_h: float = 5.0):
        """Initialize CUSUM from historical data (compute mean/std from history)."""
        mean = statistics.mean(history)
        std = statistics.stdev(history) if len(history) > 1 else 0.1
        config = CUSUMConfig(
            target_mean=mean,
            target_std=std,
            slack_k=slack_k,
            threshold_h=threshold_h,
        )
        detector = cls(config)
        detector.values = list(history)
        return detector
    
    def add_point(self, value: float) -> ShiftDetection:
        """Add a new data point and check for shift."""
        self.values.append(value)
        
        if len(self.values) == 1:
            self.cum_positive.append(0.0)
            self.cum_negative.append(0.0)
            return ShiftDetection(
                shift_detected=False,
                direction="none",
                cumulative_positive=0.0,
                cumulative_negative=0.0,
                days_since_shift=0,
                estimated_shift_size=0.0,
            )
        
        # CUSUM update
        k = self.config.k_value
        prev_sp = self.cum_positive[-1]
        prev_sn = self.cum_negative[-1]
        
        sp = max(0, prev_sp + (value - self.mean - k))
        sn = max(0, prev_sn - (value - self.mean + k))
        
        self.cum_positive.append(sp)
        self.cum_negative.append(sn)
        
        h = self.config.h_value
        
        # Detect shift
        if sp > h:
            # Upward shift detected
            shift_size = self._estimate_shift_since(len(self.values) - 1)
            return ShiftDetection(
                shift_detected=True,
                direction="up",
                cumulative_positive=sp,
                cumulative_negative=sn,
                days_since_shift=self._days_since_last_reset(),
                estimated_shift_size=shift_size,
            )
        elif sn > h:
            # Downward shift detected (REGRESSION!)
            shift_size = self._estimate_shift_since(len(self.values) - 1)
            return ShiftDetection(
                shift_detected=True,
                direction="down",
                cumulative_positive=sp,
                cumulative_negative=sn,
                days_since_shift=self._days_since_last_reset(),
                estimated_shift_size=shift_size,
            )
        
        return ShiftDetection(
            shift_detected=False,
            direction="none",
            cumulative_positive=sp,
            cumulative_negative=sn,
            days_since_shift=0,
            estimated_shift_size=0.0,
        )
    
    def _estimate_shift_since(self, idx: int) -> float:
        """Estimate how much the mean has shifted since a given point."""
        if idx >= len(self.values):
            return 0.0
        recent = self.values[idx:]
        return statistics.mean(recent) - self.mean
    
    def _days_since_last_reset(self) -> int:
        """How many data points since the CUSUM was last near zero."""
        for i in range(len(self.cum_positive) - 1, -1, -1):
            if abs(self.cum_positive[i]) < self.config.k_value * 2 and \
               abs(self.cum_negative[i]) < self.config.k_value * 2:
                return len(self.cum_positive) - 1 - i
        return len(self.cum_positive)


# === Simulated example ===
if __name__ == "__main__":
    # Normal operation: mean = 0.92, std = 0.02
    history = [0.92 + random.gauss(0, 0.02) for _ in range(20)]
    detector = CUSUMDetector.from_history(history)
    
    # After deploy: mean shifts down to 0.88
    for day in range(1, 15):
        if day <= 10:
            value = 0.88 + random.gauss(0, 0.02)  # Post-deploy
        else:
            value = 0.92 + random.gauss(0, 0.02)  # Reverted!
        
        result = detector.add_point(value)
        
        if result.shift_detected:
            print(f"Day {day}: ⚠️ {'DOWNWARD' if result.direction == 'down' else 'UPWARD'} SHIFT "
                  f"detected! (shift ≈ {result.estimated_shift_size:.3f})")
```

**🤔 Try it yourself:** Run this simulation. On which day does CUSUM detect the shift? Now compare with a simple threshold method (alert if score < 0.88). Which detects faster? What's the false positive rate of each method?

---

### Example 2: Moving Average Trend Detector

For GRADUAL drifts (not sudden shifts), CUSUM isn't as effective. Moving averages work better.

```python
"""
trend_detector.py

Detects GRADUAL trends in eval metrics using moving averages.
Catches slow degradation that CUSUM might miss.
"""

from dataclasses import dataclass
from typing import Optional
import statistics


@dataclass
class TrendResult:
    trend_detected: bool
    trend_direction: str  # "improving" | "degrading" | "stable"
    short_term_avg: float  # Last 7 days
    long_term_avg: float   # Last 30 days
    slope: float           # Points per day
    days_to_critical: Optional[float]  # If trend continues, when do we hit threshold?


class MovingAverageTrendDetector:
    """
    Detects gradual trends by comparing short-term vs long-term averages.
    
    Key insight: if short-term avg (7 days) is significantly different
    from long-term avg (30 days), a trend is in progress.
    
    Why this works for gradual drift:
    - Each day's score looks normal (within ±2σ)
    - But the 7-day average drifts away from the 30-day average
    - This detects drifts 2-3 weeks EARLIER than waiting for individual
      data points to cross a threshold
    """
    
    def __init__(self, min_history: int = 30):
        self.values: list[float] = []
        self.min_history = min_history
    
    def add_point(self, value: float, critical_threshold: Optional[float] = None) -> TrendResult:
        """Add a data point and check for trends."""
        self.values.append(value)
        
        if len(self.values) < self.min_history:
            return TrendResult(
                trend_detected=False,
                trend_direction="stable",
                short_term_avg=0.0,
                long_term_avg=0.0,
                slope=0.0,
                days_to_critical=None,
            )
        
        short_term = self.values[-7:]  # Last 7 days
        long_term = self.values[-30:]  # Last 30 days
        
        short_avg = statistics.mean(short_term)
        long_avg = statistics.mean(long_term)
        
        # Compute slope over last 14 days (linear regression)
        recent = self.values[-14:]
        n = len(recent)
        x_mean = (n - 1) / 2
        y_mean = statistics.mean(recent)
        
        numerator = sum((i - x_mean) * (v - y_mean) for i, v in enumerate(recent))
        denominator = sum((i - x_mean) ** 2 for i in range(n))
        slope = numerator / denominator if denominator != 0 else 0.0
        
        # Normalize slope: as fraction of long-term mean
        normalized_slope = slope / long_avg if long_avg != 0 else 0
        
        # Detect trend
        trend_detected = False
        direction = "stable"
        
        # A trend is significant if the short-term avg deviates
        # from the long-term avg by more than 1% of the long-term avg
        # OR the normalized slope exceeds 0.001 per day
        deviation = (short_avg - long_avg) / long_avg
        
        if deviation < -0.01:
            trend_detected = True
            direction = "degrading"
        elif deviation > 0.01:
            trend_detected = True
            direction = "improving"
        
        # If trend is degrading, estimate when we hit critical threshold
        days_to_critical = None
        if direction == "degrading" and critical_threshold is not None and slope < 0:
            current = short_avg
            if slope < 0:
                days_to_critical = (current - critical_threshold) / abs(slope)
        
        return TrendResult(
            trend_detected=trend_detected,
            trend_direction=direction,
            short_term_avg=short_avg,
            long_term_avg=long_avg,
            slope=normalized_slope,
            days_to_critical=days_to_critical,
        )


# === Usage ===
if __name__ == "__main__":
    detector = MovingAverageTrendDetector(min_history=30)
    
    # Simulate 60 days: 30 stable, 30 gradual decline
    for day in range(1, 61):
        if day <= 30:
            value = 0.92  # Stable
        else:
            value = 0.92 - (day - 30) * 0.003  # Gradual decline
        
        # Add noise
        import random
        value += random.gauss(0, 0.01)
        value = max(0, min(1, value))
        
        result = detector.add_point(value, critical_threshold=0.85)
        
        if result.trend_detected:
            print(f"Day {day}: ⚠️ {result.trend_direction} trend detected! "
                  f"7d avg: {result.short_term_avg:.3f}, "
                  f"30d avg: {result.long_term_avg:.3f}, "
                  f"slope: {result.slope:.5f}")
            if result.days_to_critical:
                print(f"  → At this rate, critical in {result.days_to_critical:.1f} days")
```

**🤔 Try this:** On which day does the trend detector first trigger? How early is that compared to when a simple threshold (alert if score < 0.88) would trigger? How many false positives does the moving average method generate from the noise?

---

### Example 3: Deployment Isolation Strategy

The best regression detection is PREVENTION — structure deployments so regressions are easy to diagnose.

```python
"""
deployment_isolation.py

Strategies for isolating deployment changes so regressions are easy to diagnose.

The golden rule: ONE CHANGE per deploy.
But this is often impractical (you need to ship features).
This code helps you track WHICH changes correlate with WHICH metric shifts.
"""

from dataclasses import dataclass, field
from typing import Optional
from datetime import datetime
from collections import defaultdict


@dataclass
class DeploymentChange:
    """A single change deployed to the system."""
    change_id: str
    change_type: str  # "prompt" | "model" | "retrieval" | "config" | "infrastructure"
    description: str
    deployed_at: str
    deployed_by: str
    expected_impact: str  # "quality" | "latency" | "cost" | "safety"
    previous_value: Optional[str] = None  # What was the old value?
    new_value: Optional[str] = None


class DeploymentTracker:
    """
    Tracks every change deployed to the system.
    
    When a regression is detected, the tracker identifies
    WHICH change is most likely responsible.
    """
    
    def __init__(self):
        self.changes: list[DeploymentChange] = []
    
    def record_change(self, change: DeploymentChange):
        """Record a deployment change."""
        self.changes.append(change)
    
    def find_suspicious_changes(
        self,
        regression_time: str,
        metric_name: str,
        lookback_hours: int = 48,
    ) -> list[DeploymentChange]:
        """
        Find deployment changes that might have caused a regression.
        
        Strategy:
        1. Find ALL changes within the lookback window
        2. Filter by changes that AFFECT the regressed metric
        3. Rank by likelihood (how recent? how related?)
        """
        from datetime import datetime, timedelta
        
        regression_dt = datetime.fromisoformat(regression_time)
        lookback = timedelta(hours=lookback_hours)
        window_start = regression_dt - lookback
        
        suspicious = []
        
        for change in self.changes:
            change_dt = datetime.fromisoformat(change.deployed_at)
            
            if change_dt < window_start or change_dt > regression_dt:
                continue
            
            # Check if this change type affects this metric
            metric_impact = {
                "prompt": ["faithfulness", "relevance", "response_length", "user_satisfaction"],
                "model": ["faithfulness", "relevance", "latency", "cost", "response_length"],
                "retrieval": ["context_precision", "context_recall", "faithfulness"],
                "config": ["latency", "cost", "error_rate"],
                "infrastructure": ["latency", "error_rate", "cost"],
            }
            
            affected_metrics = metric_impact.get(change.change_type, [])
            if metric_name in affected_metrics:
                suspicious.append(change)
        
        # Sort by recency (most recent first)
        suspicious.sort(
            key=lambda c: c.deployed_at,
            reverse=True,
        )
        
        return suspicious
    
    def generate_isolation_recommendation(self, changes: list[DeploymentChange]) -> str:
        """
        Generate recommendation for isolating the cause.
        
        If multiple changes were deployed together, recommend
        a rollback strategy that isolates each change.
        """
        if len(changes) <= 1:
            return "Only one change in window. Likely cause identified."
        
        # Multiple changes — recommend isolation
        lines = [
            f"Found {len(changes)} changes in the regression window:",
        ]
        for c in changes:
            lines.append(f"  - {c.change_type}: {c.description} (by {c.deployed_by})")
        
        lines.append("")
        lines.append("RECOMMENDED ISOLATION STRATEGY:")
        lines.append("1. Roll back ALL changes to restore system health")
        lines.append("2. Re-deploy changes ONE AT A TIME in this order:")
        
        # Order by risk (infrastructure first, prompt last)
        order = {"infrastructure": 1, "retrieval": 2, "model": 3, "config": 4, "prompt": 5}
        sorted_changes = sorted(changes, key=lambda c: order.get(c.change_type, 99))
        
        for i, c in enumerate(sorted_changes, 1):
            lines.append(f"   {i}. {c.change_type}: {c.description}")
            lines.append(f"      → Run eval pipeline. Verify metrics before next deploy.")
        
        return "\n".join(lines)


# === Usage ===
if __name__ == "__main__":
    tracker = DeploymentTracker()
    
    # Record two changes deployed at the same time
    tracker.record_change(DeploymentChange(
        change_id="ch_001",
        change_type="prompt",
        description="Updated system prompt to be more concise",
        deployed_at="2025-05-15T10:30:00",
        deployed_by="alice",
        expected_impact="quality",
        previous_value="prompt_v4",
        new_value="prompt_v5",
    ))
    
    tracker.record_change(DeploymentChange(
        change_id="ch_002",
        change_type="model",
        description="Switched to gpt-4o-mini for cost savings",
        deployed_at="2025-05-15T10:35:00",
        deployed_by="bob",
        expected_impact="cost",
        previous_value="gpt-4o",
        new_value="gpt-4o-mini",
    ))
    
    # Regression detected
    suspicious = tracker.find_suspicious_changes(
        regression_time="2025-05-15T14:00:00",
        metric_name="faithfulness",
    )
    
    recommendation = tracker.generate_isolation_recommendation(suspicious)
    print(recommendation)
```

**🤔 The isolation dilemma:** The recommendation above says "deploy changes one at a time." But your PM says: "We can't do that. We have 10 changes per deploy and we ship 3 times a day. One-at-a-time deployment would take 30 deploys per day."

Your team needs to ship fast. But you also need to isolate regressions. Propose a COMPROMISE that:
- Allows fast shipping (multiple changes per deploy)
- Still enables root cause isolation when regressions happen
- Doesn't require manual testing of each change individually

---

### Example 4: Automated Rollback Trigger

```python
"""
rollback_trigger.py

Automated rollback when a regression is detected.
Balances speed of response vs. risk of false positive rollback.
"""

from dataclasses import dataclass
from typing import Optional
from enum import Enum


class RollbackDecision(Enum):
    NO_ROLLBACK = "no_rollback"
    ROLLBACK_SUGGESTED = "rollback_suggested"  # Human must approve
    AUTO_ROLLBACK = "auto_rollback"  # Proceed without human


@dataclass
class RollbackTriggerConfig:
    """
    Configuration for when to trigger automated rollback.
    
    Three levels of confidence:
    1. SUGGEST: 2 consecutive points below LCL (lower control limit)
    2. AUTO: 3+ consecutive points below LCL, or critical metric drop > 10%
    3. NEVER: For non-critical metrics (always suggest, never auto)
    """
    metric_name: str
    is_critical: bool  # Critical metrics can auto-rollback
    lower_control_limit: float
    consecutive_failures_required: int  # How many before we act
    auto_rollback_enabled: bool = True
    min_downtime_minutes: int = 5  # Don't rollback faster than this


class RollbackTrigger:
    """
    Decides when to roll back a deployment.
    
    The key design challenge: 
    - Too AGGRESSIVE → false positives → deploying takes forever
    - Too CONSERVATIVE → false negatives → users see bad system
    
    This implementation uses a GRADUAL response:
    1. First bad data point → WARNING (no action)
    2. Second consecutive → SUGGEST rollback (human decision)
    3. Third consecutive → AUTO rollback (if enabled for critical)
    """
    
    def __init__(self, config: RollbackTriggerConfig):
        self.config = config
        self.consecutive_violations = 0
        self.last_rollback_time: Optional[str] = None
    
    def evaluate(self, current_value: float, deploy_time: str) -> RollbackDecision:
        """Evaluate whether to roll back based on current metric value."""
        
        if current_value >= self.config.lower_control_limit:
            # Metric is healthy — reset counter
            self.consecutive_violations = 0
            return RollbackDecision.NO_ROLLBACK
        
        # Metric is below threshold
        self.consecutive_violations += 1
        
        if self.consecutive_violations >= self.config.consecutive_failures_required:
            # We have enough evidence
            
            if self.config.auto_rollback_enabled and self.config.is_critical:
                return RollbackDecision.AUTO_ROLLBACK
            else:
                return RollbackDecision.ROLLBACK_SUGGESTED
        
        # Not enough evidence yet
        return RollbackDecision.NO_ROLLBACK
    
    def execute_rollback(self, deploy_id: str):
        """Execute the rollback (in production, calls your deployment system)."""
        self.last_rollback_time = datetime.now().isoformat()
        print(f"🔄 Rolling back {deploy_id}...")
        # In production: trigger rollback in your CI/CD system


# === Usage: The Rollback Decision Matrix ===
"""
DECISION MATRIX:

                          │ Critical Metric     │ Non-Critical Metric
─────────────────────────┼────────────────────┼────────────────────
1 violation below LCL    │ WARN (no action)    │ WARN (no action)
2 consecutive violations │ SUGGEST rollback    │ SUGGEST rollback
3+ consecutive violations│ AUTO rollback       │ SUGGEST rollback  
5+ consecutive violations│ AUTO rollback       │ AUTO rollback
"""

# Example: faithfulness regression
config = RollbackTriggerConfig(
    metric_name="faithfulness",
    is_critical=True,
    lower_control_limit=0.85,
    consecutive_failures_required=3,
    auto_rollback_enabled=True,
)

trigger = RollbackTrigger(config)

# Simulate: deploy + 3 bad data points
for i in range(5):
    decision = trigger.evaluate(0.82, "2025-05-15T10:30:00")
    if decision == RollbackDecision.AUTO_ROLLBACK:
        print(f"Point {i+1}: AUTO-ROLLBACK triggered!")
        break
    elif decision == RollbackDecision.ROLLBACK_SUGGESTED:
        print(f"Point {i+1}: Rollback suggested (awaiting human approval)")
    else:
        print(f"Point {i+1}: No action (observing)")
```

**🤔 The rollback trust problem:** You set up auto-rollback for faithfulness < 0.85. It triggers. But the rollback itself takes 10 minutes. During those 10 minutes, users are still getting bad answers from the reverted system (which is also old and has its own issues).

Is auto-rollback worth it? Under what conditions does rolling back within 10 minutes save more user-experience loss than the 1% chance of rolling back a GOOD deploy (false positive)? How do you estimate this?

---

## ✅ Good Output Examples

### Good Regression Detection

```
=== REGRESSION DETECTION REPORT (2025-05-15) ===

METRIC: faithfulness

CURRENT STATUS: REGRESSION DETECTED ⚠️
  
  Short-term avg (7d):  0.87  
  Long-term avg (30d):  0.92  
  Deviation:            -5.4% below baseline
  
  CUSUM: Downward shift detected (cumulative: 4.2, threshold: 3.0)
  Trend: Degrading at -0.003/day → critical threshold in 6.7 days
  

SUSPECTED CAUSE:
  Change "ch_001" (prompt_v5) was deployed 4 hours before regression detected.
  Change type: prompt
  Previous: prompt_v4 → Current: prompt_v5
  
  Confidence: HIGH (only change in regression window)


RECOMMENDED ACTION:
  Rollback prompt to v4 (suggested)
  Keep model change (ch_002) — metrics unaffected
```

### Poor Regression Detection

```
=== ALERT ===

Faithfulness dropped to 0.87!
Triggering rollback...

(No context. No suspected cause. No trend analysis.
Just a single threshold breach. Likely false alarm.)
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: Single-Point Triggers

You trigger a rollback on a SINGLE data point below a threshold. Most of these will be false alarms due to normal variance.

**🔧 Fix:** Require MULTIPLE consecutive violations before acting. 2 for warnings, 3 for rollbacks. The cost of waiting 1-2 more data points is lower than the cost of false-positive rollbacks.

### ❌ Antipattern 2: Ignoring the Baseline Update Problem

Your control limits are computed from the first 30 days of data. Six months later, the system has improved legitimately — scores are now consistently higher. Your LCL (lower control limit) is now TOO LOW — it won't catch regressions that are within the old range but would have been detected with updated limits.

**🔧 Fix:** Periodically recompute control limits (every 30-60 days). BUT: exclude periods that had KNOWN regressions (don't let bad data lower your standards).

### ❌ Antipattern 3: Treating All Metrics Equally

You apply the same CUSUM configuration to faithfulness, latency, and cost. But these metrics have very different noise characteristics.

**🔧 Fix:** Each metric needs its own:
- Control limits (based on its historical variance)
- CUSUM slack (k) and threshold (h)
- Critical/non-critical classification
- Rollback policy

### ❌ Antipattern 4: The "Fix the Dashboard" Trap

A regression is detected. The team spends 2 days building a new dashboard widget to track the regressed metric. The original problem is never fixed.

**🔧 Fix:** A dashboard is not a fix. When a regression is detected:
1. Rollback or mitigate IMMEDIATELY (stop the bleeding)
2. Investigate root cause
3. Fix the root cause
4. THEN add better monitoring (if needed)

---

## 🧪 Drills & Challenges

### Drill 1: Choose the Detection Method (20 min)

For each scenario, choose: **CUSUM**, **Moving Average**, or **Simple Threshold**. Defend your choice.

**Scenario A:** You deployed a new prompt. You expect it might affect quality slightly. You want to detect ANY change within 2 days.

**Scenario B:** User behavior is slowly shifting (more mobile users, shorter queries). You want to know when the shift has accumulated to a meaningful quality drop.

**Scenario C:** Your LLM provider occasionally has degraded performance (higher latency, lower quality). You want to roll back to a fallback model within 5 minutes.

**Scenario D:** Your evaluation metric has high variance (±5 is common). You want to detect a 2% shift in the mean.

---

### Drill 2: The CUSUM vs. Threshold Simulation (30 min)

```python
# Generate synthetic data and compare CUSUM vs. simple threshold

import random
import statistics

# Parameters
normal_mean = 0.92
normal_std = 0.03
shift_size = 0.03  # Shift down by 3%
shift_day = 30
total_days = 60

# Generate data
data = []
for day in range(total_days):
    if day < shift_day:
        value = random.gauss(normal_mean, normal_std)
    else:
        value = random.gauss(normal_mean - shift_size, normal_std)
    data.append(max(0, min(1, value)))

# Implement TWO detection methods:
# Method 1: Simple threshold (alert if value < 0.87)
# Method 2: CUSUM with k=0.5, h=4.0

# Questions:
# 1. On which day does each method detect the shift?
# 2. How many false positives does each method generate BEFORE the shift?
# 3. If you run 100 simulations (different random seeds), what's the 
#    average detection delay for each method?
# 4. What's the false positive rate for each method?
```

**Your task:** Write the simulation code. Run it 100 times. Report the average detection delay and false positive rate for each method.

---

### Drill 3: The Multi-Metric Regression Diagnosis (25 min)

A regression is detected. These metrics all changed. For each, determine:
1. Is this likely the ROOT CAUSE or a SYMPTOM?
2. What would you investigate next?

```
Metric               Before    After    Change
Faithfulness         0.92      0.84     -0.08 ↓
Answer Length        185       98       -87   ↓
Response Time (p95)  3.2s      1.8s     -1.4s  ↓
Cost per Query       $0.042    $0.018   -$0.024 ↓
Retrieval Chunks     4.2       4.1      -0.1   ≈
```

**🤔 The pattern:** All the "bad" changes (faithfulness down) are correlated with "good" changes (faster, cheaper, shorter). What single root cause explains ALL of these changes simultaneously?

**Hint:** Think about what would make a system FASTER, CHEAPER, SHORTER, but LESS FAITHFUL. There are 3 possible root causes. List them. How would you distinguish between them?

---

### Drill 4: The False Positive Budget (20 min)

Your team can handle 2 false-positive regression alerts per week. After that, they start ignoring alerts.

Each deployment has a 5% chance of causing a REAL regression (detectable within 24 hours).

You deploy 10 times per week.

Your detection system has two configurable parameters:
- Sensitivity (how small a shift to detect) — higher sensitivity = more true positives + more false positives
- Specificity (how strict to be) — higher specificity = fewer false positives + fewer true positives

**Design your detection system to maximize TRUE POSITIVES while staying within the 2 false positives/week budget.**

What's your maximum detection rate (true positives detected / total regressions)? How would the answer change if you deployed 20 times per week?

---

### Drill 5: The Post-Regression Investigation (30 min)

A regression was detected and rolled back. The rollback fixed the metric. But you don't know WHY the regression happened (you rolled back the entire deploy, not a specific change).

Now you need to identify the root cause before re-deploying. The deploy package had 5 changes:

| Change | Type | Description |
|--------|------|-------------|
| A | Config | Updated temperature from 0.1 to 0.3 |
| B | Prompt | Added "be concise" to system prompt |
| C | Model | Switched from gpt-4o-mini to gpt-4o |
| D | Retrieval | Increased top_k from 3 to 5 |
| E | Infrastructure | Added Redis caching layer |

**Design a "binary search" re-deployment strategy:** what's the MINIMUM number of re-deploys needed to identify WHICH change caused the regression? How would you order the re-deploys to MINIMIZE total time (assuming each deploy + eval takes 30 minutes)?

---

## 🚦 Gate Check

Before moving to File 08, verify you can:

1. **☐** Implement CUSUM-based shift detection for eval metrics
2. **☐** Implement moving average trend detection for gradual drifts
3. **☐** Explain the difference between sudden shifts and gradual drifts (and why they need different detection)
4. **☐** Design deployment isolation strategies that make regression diagnosis easier
5. **☐** Build an automated rollback trigger with gradual escalation (warn → suggest → auto)
6. **☐** Distinguish root causes from symptoms in multi-metric regression patterns
7. **☐** Set practical significance thresholds that balance sensitivity and false positives

### 🛑 Stop and Think

**Question 1 — The "Crying Wolf" Problem**

Your team has been using CUSUM for 6 months. It detects 3-5 regressions per month. 2 of them are real (you roll back, metrics recover). 1-3 are false alarms (you investigate, find nothing, move on).

The team is getting tired of false alarms. The CTO asks: "Can we make this thing more accurate?"

You have two options:
- Option A: Tighten thresholds (fewer false alarms, might miss some real regressions)
- Option B: Add a SECOND detection method (only alert if BOTH methods agree)

Which do you choose? How does each option affect the true positive rate vs false positive rate?

**Question 2 — The Undetected Regression**

Your detection systems all PASS. But user feedback shows a clear decline. Something is wrong that your metrics didn't catch.

List 3 possible reasons why a regression might NOT be detected by your eval metrics. For each: what metric WOULD have caught it? What would you add to your detection suite to catch it next time?

**Question 3 — Cross-Phase Application**

Choose ONE concept from a previous phase. Design a regression detection strategy for it:

- Phase 2: Your prompt optimization system. How do you detect when a prompt change caused a regression?
- Phase 3: Your embedding model. How do you detect when embedding quality degraded?
- Phase 5: Your advanced RAG pipeline. How do you detect when the reranker started performing worse?
- Phase 6: Your multi-agent system. How do you detect when agent decision quality regressed?

For your chosen system, specify:
1. What metrics would you track for regression detection?
2. What detection method (CUSUM, moving average, threshold)?
3. What's the expected detection delay?
4. What's the rollback strategy?

---

## 📚 Resources

- **"Introduction to Statistical Process Control"** (Montgomery, 2019) — The textbook on SPC. Chapters on CUSUM and EWMA are directly applicable to AI eval metrics.
- **"CUSUM: A Tutorial"** (Hawkins & Olwell, 1998) — The definitive guide to CUSUM. The parameter selection guidelines (how to choose k and h) are essential for implementing Example 1 correctly.
- **"Detecting Concept Drift in ML Systems"** (Google, 2021) — Paper on detecting when input data distribution changes. The methods apply directly to eval metrics drift.
- **"Rollback Strategies for ML Systems"** (Anthropic, 2025) — Practical guide on when and how to roll back AI systems. Their "graduated response" model (warn → suggest → auto) is what Example 4 implements.
- **"Why Phase 6's Evaluation Module Matters for Regression Detection"** — Review Phase 6 File 08, specifically the section on "The Evaluation Hierarchy." The four levels of evaluation (atomic action → step-level → trajectory → task outcome) map to four levels of regression detection.

**Next Module:**
→ **08-Custom-Eval-Framework.md**: Building your own evaluation platform — dashboards, reporting, team-wide adoption, and integration with existing tools.
