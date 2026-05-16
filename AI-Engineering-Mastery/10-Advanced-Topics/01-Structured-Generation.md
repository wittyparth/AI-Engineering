# Structured Generation

## The Problem

Function calling + validation is 99% reliable. But 99% is not enough for:
- Automated data extraction (1 error in 100 = bad day at scale)
- API calls (one bad tool call = API error)
- Compliance (must match exact schema)

Two approaches dominate: **constrained decoding** (Outlines) and **structured prompting** (Instructor).

---

## Approach 1: Constrained Decoding (Outlines)

Outlines modifies the decoding process to only generate tokens that produce valid output according to a schema.

```python
from outlines import generate, models
from pydantic import BaseModel

class Invoice(BaseModel):
    invoice_number: str
    date: str
    vendor: str
    amount: float
    line_items: list[dict]
    tax_rate: float = 0.0

model = models.transformers("meta-llama/Llama-3.2-7B")
generator = generate.json(model, Invoice)

# This is GUARANTEED to output valid JSON matching Invoice schema
result = generator("Extract invoice data: Invoice #INV-2024-001 from Acme Corp...")
print(result)  # Invoice(invoice_number='INV-2024-001', ...)
```

### How It Works

Instead of generating free text, the decoding process is constrained:
- At each token position, only tokens that produce valid JSON (matching schema) are allowed
- Impossible to produce invalid JSON
- Impossible to miss required fields

---

## Approach 2: Structured Prompting (Instructor)

Instructor patches the API client to retry with validation errors as feedback until valid output is produced.

```python
import instructor
from openai import OpenAI
from pydantic import BaseModel

class Invoice(BaseModel):
    invoice_number: str
    date: str
    vendor: str
    amount: float
    line_items: list[dict]
    tax_rate: float = 0.0

# Patch OpenAI client — now it returns Pydantic models
client = instructor.from_openai(OpenAI())

# Automatically retries with error feedback if output is invalid
invoice = client.chat.completions.create(
    model="gpt-4o",
    response_model=Invoice,
    messages=[{"role": "user", "content": "Extract invoice data from email..."}],
    max_retries=3,  # Retry with validation errors if schema doesn't match
)
print(invoice.invoice_number)  # Typed Pydantic model
```

### Key Features

```python
# Streaming partial models
from instructor import Partial

stream = client.chat.completions.create_partial(
    model="gpt-4o",
    response_model=Invoice,
    messages=[{"role": "user", "content": "Extract..."}],
    stream=True,
)
for partial_invoice in stream:
    print(partial_invoice.amount)  # Updates as tokens arrive
```

```python
# Custom validation + retry
from pydantic import field_validator

class ValidatedInvoice(BaseModel):
    invoice_number: str
    amount: float

    @field_validator("amount")
    @classmethod
    def amount_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError("Amount must be positive")
        return v

# Instructor retries with the validation error as LLM context
invoice = client.chat.completions.create(
    model="gpt-4o",
    response_model=ValidatedInvoice,
    messages=[{"role": "user", "content": "Extract invoice with amount -50"}],
    max_retries=3,
)
```

---

## Outlines vs Instructor

| Aspect | Outlines | Instructor |
|--------|----------|------------|
| Approach | Constrained decoding (modifies token generation) | Structured prompting + retry with validation feedback |
| Provider support | Local models (transformers, vLLM, exLlama) | OpenAI, Anthropic, Gemini, Cohere, any LiteLLM provider |
| Guarantee | 100% schema compliance (constraint at token level) | High (retry-based, depends on model capability) |
| Latency | 10-30% slower per token | Same as base + retry overhead (0-3 extra calls) |
| Framework | Standalone library | Patches existing clients (OpenAI, etc.) |
| API models | Limited (OpenAI function calling is separate) | Full support (GPT-4, Claude, Gemini function calling) |
| Custom validation | Must implement in post-processing | Built-in Pydantic validators with auto-retry |
| Streaming | Partial JSON parsing | Partial model streaming |

## 🔴 Senior: Choosing Your Approach

**Use Outlines when:**
- You serve local models (Llama, Mistral, etc.)
- 100% schema compliance is mandatory (healthcare, finance compliance)
- You control the inference stack (self-hosted vLLM, transformers)
- Latency is not the primary concern

**Use Instructor when:**
- You use API-based models (GPT-4, Claude, Gemini)
- You want built-in Pydantic validation with auto-retry
- You need streaming partial models
- You want function calling + structured output together

**Use both when:**
- You're building a multi-provider system (API models + local models)
- Outlines for local, Instructor for API — same Pydantic models

## Use Cases

| Use Case | Recommended Approach | Why |
|----------|-------------------|-----|
| API integration | Instructor | Function calling + Pydantic validation |
| Healthcare data extraction | Outlines | 100% compliance requirement |
| Self-hosted document parser | Outlines | Local model, no API dependency |
| Multi-provider pipeline | Both | Instructor for API, Outlines for local fallback |
| Real-time UI updates | Instructor Partial | Streaming partial models |
| Database query generation | Outlines | Must produce valid SQL every time |

## Drill: Compare Structured Generation Approaches

1. Build an invoice extraction pipeline using raw function calling (baseline)
2. Implement the same pipeline with Instructor (structured prompting + retry)
3. Implement with Outlines (constrained decoding)
4. Test on 100 documents, measure:
   - Schema compliance rate (raw: ~99%, Instructor: ~99.8%+, Outlines: 100%)
   - Average latency per call
   - P99 latency
   - Cost per 1000 documents
5. Report: which approach wins for your use case and why
