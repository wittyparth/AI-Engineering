# Structured Outputs

## The Problem

LLMs output text. Production systems need JSON, typed data, and predictable schemas.

## Three Strategies

### 1. Prompt Engineering (Fragile)
```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": """
        Return JSON: {"name": str, "age": int, "skills": list[str]}
    """}],
)
```
Fails ~5-15% of the time on complex schemas.

### 2. Function/Tool Calling (Better)
```python
tools = [{
    "type": "function",
    "function": {
        "name": "extract_user",
        "parameters": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "age": {"type": "integer"},
                "skills": {"type": "array", "items": {"type": "string"}}
            },
            "required": ["name", "age"]
        }
    }
}]
```
Much more reliable. Fails <1% on simple schemas.

### 3. Structured Output / Constrained Decoding (Best)
- **OpenAI**: `response_format={ "type": "json_object" }` with `strict: true`
- **Anthropic**: Tool use with structured tools
- **Open-source**: `outlines`, `lm-format-enforcer`, `guidance`
- **Local**: vLLM supports guided decoding via JSON schema

## 🔴 Senior: The Reliability Hierarchy

```
Constrained Decoding > Function Calling > Prompt Engineering
100% valid JSON          ~99% valid          ~85-95% valid
```

If you MUST parse the output (no human validation), use constrained decoding. Period.

## Validation + Retry

```python
from pydantic import BaseModel, ValidationError

class UserProfile(BaseModel):
    name: str
    age: int
    skills: list[str]

def extract_with_retry(text: str, max_retries: int = 3) -> UserProfile:
    for attempt in range(max_retries):
        try:
            response = get_structured_output(text, UserProfile)
            return UserProfile.model_validate(json.loads(response))
        except (ValidationError, json.JSONDecodeError) as e:
            if attempt == max_retries - 1:
                raise
            continue
```

## Drill: Structured Output Reliability

Build a test harness that:
1. Sends 100 different extraction prompts
2. Tries all 3 strategies (prompt, function calling, constrained)
3. Measures: validity rate, accuracy, latency
4. Reports the failure modes for each strategy
5. Implements automatic retry + schema validation
