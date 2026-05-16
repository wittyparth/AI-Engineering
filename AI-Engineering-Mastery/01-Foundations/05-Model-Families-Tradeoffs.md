# Model Families & Tradeoffs

## Provider Landscape (2026)

### Frontier Models
| Provider | Flagship | Best For | Cost (per M tokens) |
|----------|----------|----------|-------------------|
| OpenAI | GPT-4o / o3 | General, coding, structured output | $2.50-$10 input |
| Anthropic | Claude 4 Sonnet | Long context, safety, reasoning | $3-$15 input |
| Google | Gemini 2.5 Pro | Multimodal, 2M context, cost | $1.25-$5 input |

### Open-Source Models
| Model | Size | Best For | Run Locally? |
|-------|------|----------|-------------|
| Llama 3.3 | 70B | General, coding | Via vLLM/Ollama |
| Mistral | 7B-123B | Multilingual, efficient | Yes |
| Qwen 2.5 | 7B-72B | Coding, math | Yes |
| DeepSeek V3 | 671B MoE | Coding, reasoning | Not really |
| Phi-3/Vision | 3.8B-14B | Edge, low-resource | Yes |

## Decision Framework

```
Question: Which model should I use?

If latency critical (<500ms) → GPT-4o-mini or Llama 3.2 3B
If long context needed (>100K) → Claude 4 or Gemini 2.5
If cost sensitive at scale → Open-source via vLLM
If structured output reliability → GPT-4o (strict mode)
If multimodal needed → Gemini 2.5 or GPT-4o
If privacy required → Self-hosted Llama/Mistral
If reasoning-heavy → o3-mini or Claude 4
```

🔴 **Senior**: Model selection is a business decision, not a technical one. The best model is the cheapest one that passes your eval suite. Build a model router early.

## Model Router Pattern

```python
class ModelRouter:
    def __init__(self):
        self.routes = {
            "fast": "gpt-4o-mini",         # simple queries
            "reasoning": "o3-mini",         # complex reasoning
            "long_context": "claude-4",     # document analysis
            "cheap": "llama-3.2:3b",        # high-volume, simple
        }

    def route(self, query: str, context_size: int, complexity: str) -> str:
        if context_size > 50000:
            return self.routes["long_context"]
        if complexity == "high":
            return self.routes["reasoning"]
        if self._is_simple(query):
            return self.routes["cheap"]
        return self.routes["fast"]
```

## Drill: Model Comparison Framework

Build a script that:
1. Sends the same 10 prompts to 4 different models
2. Measures: latency, token usage, cost, output quality
3. Generates a comparison table
4. Recommends which model for which task based on your thresholds
