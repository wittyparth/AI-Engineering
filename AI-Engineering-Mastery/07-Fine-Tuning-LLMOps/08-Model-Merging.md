# Model Merging

## What Is Model Merging?

Combining weights from multiple fine-tuned models into one model without additional training. Like ensemble learning but in weight space.

## Why Merge

| Reason | Example |
|--------|---------|
| Combine capabilities | Model that's good at code + good at safety |
| Reduce cost | One merged model instead of two separate ones |
| Improve quality | Merging sometimes produces better-than-either results |

## Merging Techniques

### 1. Linear (SLERP)
```python
import torch

def slerp_merge(model_a: dict, model_b: dict, t: float = 0.5) -> dict:
    """Spherical linear interpolation between two model checkpoints."""
    merged = {}
    for key in model_a:
        if key in model_b:
            # Simple linear interpolation
            merged[key] = (1 - t) * model_a[key] + t * model_b[key]
    return merged
```

### 2. TIES (Trim, Elect Sign, Merge)
1. **Trim**: Remove small-magnitude changes (keep top 20%)
2. **Elect Sign**: For each parameter, choose majority sign direction
3. **Merge**: Average remaining parameters, resolving sign conflicts

```python
def ties_merge(base: dict, finetune_a: dict, finetune_b: dict) -> dict:
    merged = {}
    for key in base:
        # 1. Compute task vectors
        vec_a = finetune_a[key] - base[key]
        vec_b = finetune_b[key] - base[key]

        # 2. Trim: zero out small changes
        mask_a = torch.abs(vec_a) > torch.quantile(torch.abs(vec_a), 0.8)
        vec_a = vec_a * mask_a
        # 3. Elect sign
        # 4. Average
        merged[key] = base[key] + (vec_a + vec_b) / 2
    return merged
```

### 3. DARE (Drop And REscale)
Randomly drop 90-99% of delta parameters, rescale the rest.

## When to Use Each

| Technique | Best For |
|-----------|----------|
| SLERP | Similar models, combining capabilities |
| TIES | Different models, resolving conflicts |
| DARE | When one model dominates the other |

## 🔴 Senior: Merging Risks

1. **Catastrophic forgetting**: Merge can cause both specializations to degrade
2. **Unexplained behavior**: Merged models can behave unpredictably
3. **Evaluation blind spots**: Merged model may fail on edge cases neither parent failed on
4. **Not a replacement for fine-tuning**: Merging is approximate at best

**Rule**: Only merge if merged model outperforms both parents on your eval suite.
