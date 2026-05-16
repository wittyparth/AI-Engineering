# Drill: Structured Output Reliability

**Time**: 30 min  **Difficulty**: Easy

## Task

Build a test harness comparing structured output strategies:

```python
import json
from pydantic import BaseModel, ValidationError

# Schema
class Product(BaseModel):
    name: str
    price: float
    category: str
    in_stock: bool
    tags: list[str]

# Strategy 1: Prompt engineering
def extract_via_prompt(text: str) -> str:
    return llm.generate(f"""
    Extract product info as JSON:
    {text}
    Return JSON only.
    """)

# Strategy 2: Function calling
def extract_via_function(text: str) -> str:
    return llm.generate_with_tools(text, tools=[product_tool])

# Strategy 3: Constrained decoding (OpenAI strict mode)
def extract_via_structured(text: str) -> str:
    return llm.generate_structured(text, response_format=Product)

# Test
test_cases = [
    "The new MacBook Pro costs $2499 and is in stock. Category: electronics.",
    "Running shoes: $89.99, in stock, category: sports. Features: lightweight, durable.",
    # ... more cases including edge cases
]

results = {s: {"valid": 0, "invalid": 0, "errors": []} for s in strategies}
for text in test_cases:
    for name, strategy in strategies.items():
        try:
            response = strategy(text)
            Product.model_validate(json.loads(response))
            results[name]["valid"] += 1
        except (json.JSONDecodeError, ValidationError) as e:
            results[name]["invalid"] += 1
            results[name]["errors"].append(str(e))

print(json.dumps(results, indent=2))
```

## Report
- Which strategy had 100% valid JSON rate?
- Which had the best accuracy?
- Latency comparison?
- What errors did each strategy produce?
