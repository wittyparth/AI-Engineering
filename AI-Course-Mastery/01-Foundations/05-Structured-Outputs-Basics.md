# Structured Outputs — Getting Reliable Data from LLMs

## 🎯 Purpose & Goals

> **🛑 STOP. Here's a problem:**
>
> You ask the model: *"Extract the name, email, and age from: 'Rahul Sharma is 28 years old and his email is rahul@example.com'"*
>
> The model returns: *"The person mentioned is Rahul Sharma, who is 28 years old. His email address is rahul@example.com."*
>
> **That's a SENTENCE, not structured data.** Your app needs `{"name": "Rahul Sharma", "age": 28, "email": "rahul@example.com"}`.
>
> **How would you solve this? Think about it before reading on.**
>
> *(Hint: Can you make the model return JSON reliably? What if it returns invalid JSON? What if it misses a field?)*

---

**By the end of this file, you will:**
- Never parse LLM output with string manipulation again (it's 2025, not 2023)
- Know 3 different approaches to getting structured data from LLMs
- Have a production-ready extraction pipeline that handles failures gracefully
- Understand why structured outputs are the foundation of AI agents

**⏱ Time Budget:** 1 hour 45 min

---

## 📖 The Problem: LLMs Return Text, You Need Data

```python
# THE NAIVE APPROACH (what beginners do — painful and fragile):

response = "The person mentioned is Rahul Sharma, who is 28 years old. His email is rahul@example.com."

# Try to parse with string manipulation...
import re
name_match = re.search(r"mentioned is (\w+ \w+)", response)
age_match = re.search(r"(\d+) years old", response)
email_match = re.search(r"email is ([\w.@]+)", response)

# WHAT COULD GO WRONG?
# - Model says "The individual" instead of "The person mentioned is"
# - Model says "aged 28" instead of "28 years old"
# - Model includes "I estimate" before the age
# - Model outputs in Hindi
# - Model adds extra text at the end

# THIS IS A MAINTENANCE NIGHTMARE.
# Every model update can break your parsing.
# Every prompt change can break your parsing.
```

> **🤔 YOUR TURN:** Before reading the solution, think of 3 approaches you could use to solve this. Write them down.

---

## 📖 The 3 Approaches (Ranked by Reliability)

### Approach 1: JSON Mode (Good — Forces Valid JSON)

```python
from openai import OpenAI
import json

client = OpenAI()

# 🛑 PAUSE: Before reading the code, how would YOU write a function
# that asks the model for JSON and ensures it gets JSON back?

# Think about:
# 1. How to specify the JSON schema in the prompt
# 2. How to force the model to ONLY return JSON
# 3. How to parse and validate the result
# 4. What happens when the model returns invalid JSON

# --- NOW READ THE SOLUTION ---

def extract_person_json_mode(text: str) -> dict | None:
    """Ask the model to return JSON, force JSON mode"""
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """Extract person information from text.
Return a JSON object with these fields (and ONLY these fields):
- name: string (full name)
- age: number (integer, null if not mentioned)
- email: string (null if not mentioned)
- occupation: string (null if not mentioned)

Example: {"name": "Rahul Sharma", "age": 28, "email": "rahul@example.com", "occupation": "Software Engineer"}

Return ONLY the JSON. No explanation. No other text."""
            },
            {"role": "user", "content": text}
        ],
        response_format={"type": "json_object"},  # 🔑 Forces valid JSON!
        temperature=0.0
    )
    
    content = response.choices[0].message.content
    
    try:
        return json.loads(content)
    except json.JSONDecodeError as e:
        print(f"Invalid JSON: {e}")
        print(f"Raw content: {content}")
        return None

# Test it
result = extract_person_json_mode("Rahul Sharma is 28 years old and his email is rahul@example.com")
print(result)
# {'name': 'Rahul Sharma', 'age': 28, 'email': 'rahul@example.com', 'occupation': None}

# ✅ Benefit: Always returns valid JSON
# ❌ Drawback: Schema defined in prompt text — model might add/remove fields
```

**🔍 Debug Challenge:** What happens if you call `extract_person_json_mode("I love ice cream")` (no person mentioned)?

<details>
<summary>Think then click</summary>
The model will hallucinate a person! It might return:
```json
{"name": "John Doe", "age": null, "email": null, "occupation": null}
```
The model is FORCED to return JSON — it can't say "no person found."
You need to handle this case: add an "error" field or use Approach 2.
</details>

### Approach 2: Structured Outputs / Strict Mode (Better — Guaranteed Schema)

```python
from pydantic import BaseModel
from typing import Optional

# 🛑 PAUSE: How would YOU define a schema that the model MUST follow?
# Think about: field types, validation, optional fields

# --- NOW READ ---

class Person(BaseModel):
    """Pydantic model defines the exact schema"""
    name: Optional[str] = None  # Optional because text might mention no one
    age: Optional[int] = None
    email: Optional[str] = None
    occupation: Optional[str] = None
    confidence: float = 0.0  # How confident is the extraction?
    error: Optional[str] = None  # If no person found

def extract_person_structured(text: str) -> Person:
    """
    OpenAI Structured Outputs — the model is GUARANTEED to match the schema.
    If it can't fill a field, it sets it to null (not hallucinate).
    """
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "Extract person information from text. Set fields to null if not mentioned. If no person is mentioned, set error field."
            },
            {"role": "user", "content": text}
        ],
        response_format=Person,  # 🔑 Pass Pydantic model directly!
        temperature=0.0
    )
    
    # response.choices[0].message.parsed is ALREADY a Person object!
    person = response.choices[0].message.parsed
    return person

# Test with person
person = extract_person_structured("Rahul Sharma, 28, rahul@example.com, works at Google")
print(f"Name: {person.name}")       # Rahul Sharma
print(f"Age: {person.age}")          # 28
print(f"Error: {person.error}")      # None

# Test with NO person
person = extract_person_structured("I love ice cream")
print(f"Name: {person.name}")       # None
print(f"Error: {person.error}")      # "No person mentioned in text"

# ✅ UNLIKE JSON Mode: The model CANNOT add/remove/rename fields
# ✅ Validation is automatic — age is int, confidence is float
# ✅ Null handling is explicit — model sets null, doesn't hallucinate
```

### Approach 3: Instructor Library (Best — Multi-Provider)

```python
import instructor
from anthropic import Anthropic
from openai import OpenAI

# 🛑 PAUSE: What if you need structured outputs from Claude or Gemini?
# They don't support OpenAI's structured output API.
# How would you handle this?

# --- NOW READ ---
# Instructor library works with ANY provider!

# Patch Anthropic to support structured outputs
anthropic_client = instructor.from_anthropic(Anthropic())

def extract_with_claude(text: str) -> Person:
    """Same Pydantic model, works with Claude!"""
    person = anthropic_client.messages.create(
        model="claude-3-5-sonnet-20241022",
        response_model=Person,  # Same Pydantic model
        max_tokens=1000,
        messages=[{"role": "user", "content": f"Extract person from: {text}"}],
    )
    return person

# Same API, different provider:
person = extract_with_claude("Priya Patel, 32, priya@company.in, Senior Engineer at Microsoft")
print(f"Name: {person.name}")  # Priya Patel
print(f"Age: {person.age}")     # 32
```

---

## 📖 Production Patterns You'll Actually Use

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from enum import Enum
import instructor

class Sentiment(str, Enum):
    POSITIVE = "POSITIVE"
    NEGATIVE = "NEGATIVE" 
    NEUTRAL = "NEUTRAL"
    MIXED = "MIXED"

class ProductReview(BaseModel):
    """Real production schema — used in thousands of companies"""
    sentiment: Sentiment
    confidence: float = Field(ge=0.0, le=1.0)
    key_phrases: List[str] = Field(default_factory=list, max_length=5)
    pros: List[str] = Field(default_factory=list)
    cons: List[str] = Field(default_factory=list)
    rating: Optional[int] = Field(None, ge=1, le=5)
    summary: str = Field(max_length=200)
    suggested_response: Optional[str] = None
    escalation_required: bool = False
    category: str = "general"

client_instructor = instructor.from_openai(OpenAI())

def analyze_review(review_text: str) -> ProductReview:
    """Production-grade review analysis"""
    return client_instructor.chat.completions.create(
        model="gpt-4o-mini",
        response_model=ProductReview,
        messages=[{
            "role": "user",
            "content": f"""Analyze this customer review.

Review: {review_text}

Extract the structured information."""
        }],
        temperature=0.0
    )

# Real examples:
reviews = [
    "The product is amazing! Fast delivery and great quality. Will buy again.",
    "Terrible customer service. They took 2 weeks to respond and then blamed me.",
    "It's okay for the price. Not great, not terrible.",
]

for review in reviews:
    result = analyze_review(review)
    print(f"\nSentiment: {result.sentiment.value} ({result.confidence:.0%})")
    print(f"Pros: {result.pros}")
    print(f"Cons: {result.cons}")
    print(f"Escalate: {result.escalation_required}")
```

---

## 🧪 Design Challenge: Build an Invoice Extractor

**Time: 30 minutes**

**THE SCENARIO:** Your company receives 1,000 invoices/week as emails. You need to extract structured data from each one.

**YOUR TASK:** Design the Pydantic model and extraction function.

**Think about:**
- What fields does an invoice have?
- Some fields are required, some optional
- What if the invoice is in a different currency?
- What if the invoice is handwritten/scanned?
- What if the text mentions multiple line items?

```python
# 🛑 TRY THIS FIRST before looking at my solution:

from pydantic import BaseModel, Field
from typing import List, Optional
from enum import Enum
from datetime import date

# Design your Invoice model here:
class Invoice(BaseModel):
    """An invoice extracted from text"""
    # YOUR FIELDS HERE
    pass

# Design your extraction function:
def extract_invoice(text: str) -> Invoice:
    """Extract invoice details from free text"""
    # YOUR CODE HERE
    pass
```

<details>
<summary>👀 See production solution after you've designed yours</summary>

```python
from pydantic import BaseModel, Field, field_validator
from typing import List, Optional
from enum import Enum
from datetime import date

class Currency(str, Enum):
    USD = "USD"
    INR = "INR"
    EUR = "EUR" 
    GBP = "GBP"
    UNKNOWN = "UNKNOWN"

class LineItem(BaseModel):
    description: str
    quantity: int = Field(default=1, ge=1)
    unit_price: float = Field(ge=0)
    total_price: float = Field(ge=0)
    
    @field_validator('total_price')
    def validate_total(cls, v, info):
        """Sanity check: total should roughly equal qty × unit_price"""
        qty = info.data.get('quantity', 1)
        unit = info.data.get('unit_price', 0)
        expected = qty * unit
        if v > expected * 1.1:  # Allow 10% rounding
            print(f"Warning: total ({v}) differs from qty×unit ({expected})")
        return v

class Invoice(BaseModel):
    vendor_name: str
    vendor_address: Optional[str] = None
    invoice_number: str
    invoice_date: date
    due_date: Optional[date] = None
    currency: Currency = Currency.INR
    line_items: List[LineItem] = Field(default_factory=list)
    subtotal: Optional[float] = None
    tax_amount: Optional[float] = None
    total_amount: float
    confidence: float = Field(default=0.0, ge=0.0, le=1.0)
    extraction_notes: Optional[str] = None
```

**Compare:**
- Did you remember validation?
- Did you handle missing optional fields?
- Did you include a confidence score?
- Did you think about line items vs summary fields?

**The gaps show where to improve.** If you got all of these, you're ahead of 90% of engineers.
</details>

---

## 📖 Cross-Reference

Structured outputs connect to:
- **Phase 2 (Prompt Engineering):** Structured outputs ARE a prompting technique
- **Phase 6 (Agents):** Function calling IS structured outputs for tool calls
- **Phase 7 (Evals):** Eval frameworks use structured outputs for scoring
- **Phase 9 (Fine-tuning):** Training data needs structured formats

---

## 🚦 Gate Check

- [ ] I can explain why naive parsing is fragile
- [ ] I know the difference between JSON Mode and Structured Outputs
- [ ] I can build a Pydantic model for extraction
- [ ] I've designed my own invoice extractor
- [ ] I understand how Instructor works cross-provider

---

**→ Continue to `06-Python-AI-Patterns.md`**
