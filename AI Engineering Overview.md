# 🧠 The Complete AI Engineering Course
### From Software Engineer to Production AI Builder

**Designed for:** Partha — Frontend/Backend/Cloud Engineer  
**Goal:** Build production-grade AI applications and land an AI Engineering role or ship real products  
**Philosophy:** Learn by building. Every concept has a project. Every project ships.  
**Standard:** Adrian Cantrill-level depth, but for AI Engineering  
**Duration:** ~6–8 months at 2 hours/day | Flexible — go faster or slower  

---

> *"The engineers extracting real value from AI aren't those with the most innovative demos — they're the ones doing the less glamorous engineering work: building evaluation pipelines, implementing guardrails, designing for uncertainty, and treating LLM systems with the same rigour they'd apply to any critical infrastructure."*
> — ZenML, analysis of 1,200 production LLM deployments (2025)

---

## 📋 How This Course Works

This course is designed like a senior engineer mentoring a junior engineer. You won't just read — you'll build, break, fix, and ship. Each module has:

- 📖 **Concept Deep-Dives** — The theory you need to understand, not just copy
- 🎯 **Coding Challenges** — Hands-on tasks you must complete before moving on
- 🏗️ **Mini-Projects** — Small, focused builds that reinforce each concept
- 🚀 **Portfolio Projects** — Real-world products you can showcase
- 🧪 **Knowledge Checks** — Quizzes and self-assessment gates
- 🔁 **Revision Checkpoints** — Spaced repetition reviews built in
- 📹 **Curated Resources** — Best YouTube videos, GitHub repos, papers, blogs
- 🚦 **"Can you do this?" Tests** — Before advancing, prove you can

**Ground Rules from your senior engineer:**
1. Never just copy-paste code. Type it. Break it. Fix it.
2. Every project must be pushed to GitHub with a README
3. If you can't explain it to someone in plain English, you don't understand it yet
4. Build in public — document your journey on Twitter/LinkedIn/blog
5. The goal is not to finish fast. The goal is to build deeply.

---

## 🗺️ Course Map

```
PHASE 0: FOUNDATIONS (Weeks 1–3)
  Module 01 → AI Engineering Landscape
  Module 02 → Python for AI Engineering
  Module 03 → The Transformer & LLM Internals

PHASE 1: WORKING WITH LLMs (Weeks 4–8)
  Module 04 → Prompt Engineering (The Real Kind)
  Module 05 → LLM APIs & SDKs — OpenAI, Anthropic, Google
  Module 06 → Structured Outputs & Function Calling
  Module 07 → Embeddings & Semantic Search

PHASE 2: RAG SYSTEMS (Weeks 9–13)
  Module 08 → Vector Databases Deep Dive
  Module 09 → Building RAG — Naive to Advanced
  Module 10 → Chunking, Indexing & Retrieval Strategies
  Module 11 → Multimodal RAG

PHASE 3: AI AGENTS (Weeks 14–19)
  Module 12 → Agent Foundations — Tools, Memory, Planning
  Module 13 → Building Agents with LangChain & LangGraph
  Module 14 → Multi-Agent Systems
  Module 15 → MCP — Model Context Protocol

PHASE 4: PRODUCTION AI (Weeks 20–26)
  Module 16 → LLM Evaluation & Evals as Unit Tests
  Module 17 → Observability, Tracing & Monitoring
  Module 18 → Guardrails, Safety & Security
  Module 19 → Fine-Tuning & LoRA

PHASE 5: SHIPPING & SCALE (Weeks 27–32)
  Module 20 → LLMOps & CI/CD for AI
  Module 21 → Cost Optimization & Latency
  Module 22 → Full-Stack AI Application Patterns

CAPSTONE
  Module 23 → Capstone — Build & Launch a Real AI Product
```

---

## 🛠️ Your Tech Stack

By the end of this course, you will be fluent in:

**Core AI:**
- OpenAI API, Anthropic Claude API, Google Gemini API
- HuggingFace Transformers & Hub
- Ollama (local LLMs — Llama, Mistral, Phi)
- LangChain, LangGraph, LlamaIndex

**Databases & Search:**
- Pinecone, Weaviate, Chroma, pgvector
- FAISS for local vector search
- Redis for caching

**Observability & Evals:**
- Langfuse (open-source, self-hostable)
- DeepEval, Ragas
- LangSmith

**Infrastructure:**
- FastAPI for AI backends
- Vercel / Railway for deployment
- Docker for containerization
- AWS Bedrock (leverages your cloud knowledge)

**Frontend (you already know this):**
- Next.js + streaming responses
- Vercel AI SDK

---

---

# PHASE 0: FOUNDATIONS

---

## Module 01 — The AI Engineering Landscape

**Time:** ~1 week | **Difficulty:** ⭐☆☆☆☆

### What This Module Is About

Before you write a single line of code, you need to understand the terrain. AI Engineering is a new discipline — it's not Data Science, it's not ML Research, and it's not traditional Software Engineering. Understanding where it sits, what the job actually requires, and what the production reality looks like will save you months of building in the wrong direction.

Think of this like an AWS exam intro module — you learn the "why" before the "how".

---

### 1.1 — What Is an AI Engineer?

An AI Engineer is the bridge between raw AI capabilities (models, APIs, research) and real products that users actually use. Here's how it differs from the roles you might confuse it with:

| Role | Focus | Builds What |
|------|--------|-------------|
| **ML Engineer** | Training models, pipelines, MLOps | Custom models from scratch |
| **Data Scientist** | Analysis, statistics, model evaluation | Insights, notebooks |
| **AI Researcher** | Novel architectures, papers | New techniques |
| **AI Engineer** ← You | Applying models to real products | User-facing AI features |
| **Full-Stack Dev** | End-to-end web products | Traditional web apps |

**The AI Engineer's job is:**
1. Take powerful pre-trained models (GPT-4o, Claude 3.5, Gemini) 
2. Wrap them with the right prompting, context, retrieval, and tooling
3. Build reliable, observable, production-grade systems
4. Ship them to real users

This is perfect for you. You already know how to build products. Now you're adding AI as a capability layer.

---

### 1.2 — The AI Stack (Know This Deeply)

```
┌─────────────────────────────────────────┐
│           USER INTERFACE                │
│   (Chat UI, API, Voice, Email, etc.)   │
├─────────────────────────────────────────┤
│         APPLICATION LAYER               │
│   (Your code: FastAPI, Next.js, etc.)  │
├─────────────────────────────────────────┤
│         AI ORCHESTRATION                │
│   (LangChain, LlamaIndex, custom)      │
├─────────────────────────────────────────┤
│  CONTEXT MANAGEMENT                     │
│  ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │  RAG /   │ │  Memory  │ │ Tools/ │  │
│  │ Retrieval│ │          │ │ Agents │  │
│  └──────────┘ └──────────┘ └────────┘  │
├─────────────────────────────────────────┤
│           LLM LAYER                     │
│  OpenAI | Anthropic | Google | Local   │
├─────────────────────────────────────────┤
│         OBSERVABILITY & EVALS           │
│      (Langfuse, LangSmith, Ragas)      │
└─────────────────────────────────────────┘
```

Every module in this course is one or more layers of this stack. By the end, you'll understand all of them.

---

### 1.3 — How LLMs Actually Work (The Mental Model You Need)

You don't need to understand the math. You need the right mental model.

**The key insight:** An LLM is a *next-token predictor*. It takes a sequence of tokens (chunks of text) and predicts the most likely next token, again and again, until it's done.

**What this means for you as an engineer:**
- The model has no memory between conversations — everything must be in the context window
- The model doesn't "know" things — it generates *likely continuations* of text
- This is why hallucinations happen — the model generates what seems likely, not what's true
- This is why context engineering matters — garbage in, garbage out
- This is why RAG works — you give it real facts to continue from

**Token Mental Model:**
- ~1 token ≈ ~0.75 words in English
- GPT-4o: 128k context window = ~96,000 words = ~200 pages of text
- Claude 3.5 Sonnet: 200k context window
- Every token costs money — context engineering = cost engineering

**The Jagged Frontier:**
> AI models are "jagged" — brilliant at hard tasks, brittle at seemingly simple ones. The skill of an AI engineer is understanding where the jaggedness is for your use case.

---

### 1.4 — The Production Reality (Read This Carefully)

Based on analysis of 1,200 production LLM deployments (ZenML, 2025):

**What separates demo apps from production apps:**
1. **Evaluation pipelines** — "Evals are the new unit tests"
2. **Guardrails** — Constraining model behavior before it reaches users
3. **Observability** — Full tracing of every model call
4. **Context engineering** — Not just prompt engineering, but what you put in the window
5. **Software engineering fundamentals** — Tests, CI/CD, proper architecture

**The 80/20 problem:**
> Getting to 80% quality takes 20% of the time. Getting to 95%+ takes the remaining 80% of time. That final stretch is pure engineering — evaluation, guardrails, iteration.

This course teaches you that final 20% of work that most "AI tutorials" skip.

---

### 📚 Required Reading & Watching

**Watch first (in order):**
- 🎥 [Andrej Karpathy — "Intro to Large Language Models"](https://www.youtube.com/watch?v=zjkBMFhNj_g) *(1 hour — essential mental model)*
- 🎥 [Andrej Karpathy — "Let's build GPT"](https://www.youtube.com/watch?v=kCc8FmEb1nY) *(3 hours — optional but deeply educational)*
- 🎥 [3Blue1Brown — "Attention in transformers"](https://www.youtube.com/watch?v=eMlx5fFNoYc) *(27 min — intuition for transformers)*

**Read:**
- 📄 [roadmap.sh/ai-engineer](https://roadmap.sh/ai-engineer) — Map the full landscape
- 📄 [What Is an AI Engineer? — Chip Huyen](https://huyenchip.com/2024/04/23/what-ai-engineers-should-know.html)
- 📄 [ZenML — What 1,200 Production Deployments Reveal About LLMOps](https://www.zenml.io/blog/what-1200-production-deployments-reveal-about-llmops-in-2025)

**GitHub Repos to Star:**
- ⭐ [awesome-llm-apps](https://github.com/Shubhamsaboo/awesome-llm-apps)
- ⭐ [llm-course by mlabonne](https://github.com/mlabonne/llm-course)
- ⭐ [LangChain](https://github.com/langchain-ai/langchain)

---

### 🧪 Module 01 Knowledge Check

Answer these in writing in your learning journal. If you can't answer, re-read the section:

1. What is the difference between an AI Engineer, ML Engineer, and Data Scientist?
2. Explain what a context window is and why it matters for cost
3. Why do LLMs hallucinate? (Explain using the next-token prediction model)
4. What is the "jagged frontier" concept?
5. What separates a demo from a production AI application? Name 3 things.
6. Sketch the AI stack from memory (user interface → LLM layer)

**Grading yourself:** If you can answer all 6 in 10 minutes without looking — proceed. If not — re-read.

---

### 🚦 Module 01 Gate — Can You Do This?

**Before moving to Module 02, complete this task:**

Write a 500-word blog post (not for publishing, just for you) titled:
*"What I'm building and why I'm learning AI Engineering"*

Include:
- What kind of products you want to build
- One specific problem you want to solve with AI
- Your understanding of what an AI Engineer does vs an ML Engineer

Push it to a private GitHub repo called `ai-engineering-journey`. This repo will collect all your work throughout this course.

---
---

## Module 02 — Python for AI Engineering

**Time:** ~1 week | **Difficulty:** ⭐⭐☆☆☆

> *You already know programming. This module is specifically about the Python patterns that appear repeatedly in AI engineering — async, streaming, data validation, environment management. Skip what you know, deeply learn what you don't.*

---

### 2.1 — Setting Up Your AI Engineering Environment

**Never use global Python installs.** Here's the professional setup:

```bash
# Install uv (the modern Python package manager — faster than pip)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create a project
uv init my-ai-project
cd my-ai-project

# Create and activate virtual environment
uv venv
source .venv/bin/activate  # Mac/Linux
.venv\Scripts\activate     # Windows

# Install packages fast
uv add openai anthropic langchain langchain-openai python-dotenv
```

**Environment variables — NEVER hardcode API keys:**
```bash
# .env file (never commit this)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
```

```python
# Always load this way
from dotenv import load_dotenv
import os

load_dotenv()

api_key = os.getenv("OPENAI_API_KEY")
if not api_key:
    raise ValueError("OPENAI_API_KEY not found. Check your .env file.")
```

Add this to your `.gitignore`:
```
.env
.venv/
__pycache__/
*.pyc
```

---

### 2.2 — Async Python (Critical for AI Engineering)

AI API calls take time. Blocking on them = slow applications. Async = doing multiple things at once.

**The problem without async:**
```python
import time

def slow_api_call(query):
    time.sleep(2)  # Simulates API call
    return f"Result for: {query}"

# Sequential — takes 6 seconds total
results = []
for q in ["query1", "query2", "query3"]:
    results.append(slow_api_call(q))
```

**The solution with async:**
```python
import asyncio
import httpx

async def call_api(client, query):
    # Non-blocking API call
    response = await client.post("https://api.example.com", json={"q": query})
    return response.json()

async def main():
    async with httpx.AsyncClient() as client:
        # All 3 calls happen concurrently — takes ~2 seconds total
        results = await asyncio.gather(
            call_api(client, "query1"),
            call_api(client, "query2"),
            call_api(client, "query3"),
        )
    return results

asyncio.run(main())
```

**Real-world pattern — OpenAI async:**
```python
from openai import AsyncOpenAI
import asyncio

client = AsyncOpenAI()

async def get_completion(prompt: str) -> str:
    response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

async def batch_completions(prompts: list[str]) -> list[str]:
    tasks = [get_completion(p) for p in prompts]
    return await asyncio.gather(*tasks)

# Run it
results = asyncio.run(batch_completions(["What is Paris?", "What is Tokyo?"]))
```

---

### 2.3 — Streaming Responses (Essential for Good UX)

Nobody wants to stare at a loading spinner for 10 seconds. Streaming shows text as it's generated — this is how ChatGPT works.

```python
from openai import OpenAI

client = OpenAI()

def stream_response(prompt: str):
    """Stream tokens as they're generated"""
    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True  # This is the key
    )
    
    full_response = ""
    for chunk in stream:
        if chunk.choices[0].delta.content is not None:
            token = chunk.choices[0].delta.content
            print(token, end="", flush=True)  # Print without newline
            full_response += token
    
    print()  # Final newline
    return full_response

stream_response("Explain quantum computing in simple terms")
```

**Async streaming (for FastAPI):**
```python
from openai import AsyncOpenAI
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()
client = AsyncOpenAI()

async def generate_tokens(prompt: str):
    stream = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

@app.get("/stream")
async def stream_endpoint(prompt: str):
    return StreamingResponse(
        generate_tokens(prompt),
        media_type="text/plain"
    )
```

---

### 2.4 — Pydantic — Data Validation (You'll Use This Constantly)

LLMs output text. You often need structured data. Pydantic is your validator.

```python
from pydantic import BaseModel, Field, validator
from typing import Optional, List

class ExtractedPerson(BaseModel):
    name: str
    age: Optional[int] = None
    email: Optional[str] = None
    skills: List[str] = Field(default_factory=list)
    
    @validator('email')
    def email_must_be_valid(cls, v):
        if v and '@' not in v:
            raise ValueError('Invalid email format')
        return v
    
    @validator('age')
    def age_must_be_reasonable(cls, v):
        if v and (v < 0 or v > 150):
            raise ValueError('Age must be between 0 and 150')
        return v

# Validate LLM output
raw_output = {"name": "Rahul", "age": 28, "email": "rahul@example.com", "skills": ["Python", "React"]}

try:
    person = ExtractedPerson(**raw_output)
    print(person.name)  # Rahul
    print(person.model_dump())  # Full dict
except ValueError as e:
    print(f"Validation failed: {e}")
```

**Why this matters:** LLMs make mistakes. Pydantic catches them before they crash your app.

---

### 2.5 — Error Handling for AI Systems

AI API calls fail. Rate limits hit. Models return unexpected output. You need robust error handling.

```python
import time
from openai import OpenAI, RateLimitError, APIError
from typing import Optional

client = OpenAI()

def robust_completion(
    prompt: str, 
    max_retries: int = 3,
    base_delay: float = 1.0
) -> Optional[str]:
    """
    Make an API call with exponential backoff retry logic.
    This is production-grade error handling.
    """
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{"role": "user", "content": prompt}],
                timeout=30  # Always set timeouts!
            )
            return response.choices[0].message.content
            
        except RateLimitError:
            if attempt < max_retries - 1:
                wait_time = base_delay * (2 ** attempt)  # Exponential backoff
                print(f"Rate limit hit. Waiting {wait_time}s before retry {attempt + 1}")
                time.sleep(wait_time)
            else:
                raise
                
        except APIError as e:
            print(f"API error on attempt {attempt + 1}: {e}")
            if attempt < max_retries - 1:
                time.sleep(base_delay)
            else:
                raise
                
    return None
```

---

### 2.6 — Type Hints (Write Them Always)

AI engineering code gets complex fast. Type hints make it readable and catch bugs early.

```python
from typing import Any, Dict, List, Optional, Union
from dataclasses import dataclass

@dataclass
class Message:
    role: str  # "system", "user", or "assistant"
    content: str

@dataclass  
class ConversationHistory:
    messages: List[Message]
    
    def add_message(self, role: str, content: str) -> None:
        self.messages.append(Message(role=role, content=content))
    
    def to_api_format(self) -> List[Dict[str, str]]:
        return [{"role": m.role, "content": m.content} for m in self.messages]
    
    def get_last_n_messages(self, n: int) -> List[Message]:
        return self.messages[-n:]

# Usage
history = ConversationHistory(messages=[])
history.add_message("system", "You are a helpful assistant")
history.add_message("user", "Hello!")
print(history.to_api_format())
```

---

### 📚 Resources

**Watch:**
- 🎥 [Corey Schafer — Python async/await](https://www.youtube.com/watch?v=t5Bo1Je9EmE)
- 🎥 [ArjanCodes — Pydantic v2 Complete Tutorial](https://www.youtube.com/watch?v=yj-wSRJwrrc)
- 🎥 [FastAPI Tutorial — Official](https://fastapi.tiangolo.com/tutorial/)

**Read:**
- 📄 [Real Python — Async IO in Python](https://realpython.com/async-io-python/)
- 📄 [Pydantic Documentation](https://docs.pydantic.dev/latest/)

---

### 🎯 Coding Challenges

Complete these before moving on. They should take 2–4 hours total:

**Challenge 1 — Async Batch Processor:**
Write an async function that takes a list of 10 prompts and sends them all concurrently to the OpenAI API, returning all results. Add a simple progress counter.

**Challenge 2 — Streaming Chat:**
Build a simple CLI chat application that streams responses character by character. Maintain conversation history across turns.

**Challenge 3 — Pydantic LLM Output Validator:**
Write a function that asks GPT-4o-mini to extract person details from a paragraph of text, validates the output with Pydantic, and handles validation errors gracefully.

**Challenge 4 — Retry Wrapper:**
Take the `robust_completion` function above and add: logging of each attempt, a configurable timeout, and a fallback response if all retries fail.

---

### 🏗️ Mini-Project: AI-Powered CLI Tool

**What you'll build:** A CLI tool called `aiask` that:
1. Takes a question as a command-line argument
2. Streams the response to the terminal
3. Maintains session history in a JSON file
4. Supports `--reset` flag to clear history
5. Shows token count after each response

```bash
# Usage
aiask "What is the capital of France?"
aiask "What about its population?"  # Uses history
aiask --reset
```

**Requirements:**
- Use `argparse` or `typer` for CLI
- Streaming must work (no loading bar, just token-by-token output)
- History stored in `~/.aiask_history.json`
- Token counting using `tiktoken`
- Proper error handling

**Push to:** `ai-engineering-journey/projects/01-aiask-cli/`

---

### 🧪 Module 02 Knowledge Check

1. What is the difference between `async def` and `def`? When would you use each?
2. Explain what `await asyncio.gather()` does in one sentence
3. What does Pydantic do, and why is it important for AI systems?
4. What is exponential backoff and why do we use it?
5. Show the code to stream an OpenAI response and print each token

---

### 🚦 Module 02 Gate

Before moving on, your `aiask` CLI tool must:
- ✅ Work end-to-end (ask a question, get a streamed answer)
- ✅ Maintain conversation history
- ✅ Have proper error handling (test it by using a wrong API key)
- ✅ Be pushed to GitHub with a README

---
---

## Module 03 — The Transformer & LLM Internals (What You Must Know)

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐☆☆

> *You don't need to implement a transformer from scratch for production AI engineering. But you absolutely need to understand what's happening inside, because it directly affects every architectural decision you'll make.*

This module is your "how does the engine work" module. After this, you'll understand:
- Why longer prompts cost more
- Why temperature matters and how to set it
- Why models hallucinate
- What embeddings actually are
- Why fine-tuning works

---

### 3.1 — Tokens: The Atomic Unit of LLMs

Everything starts with tokenization. LLMs don't read words — they read tokens.

**What is a token?**
- A token is a chunk of characters — roughly 0.75 words in English
- Common words like "the", "is", "of" are usually single tokens
- Rare words get split: "tokenization" → ["token", "ization"]
- Numbers, code, and non-English text often use more tokens per word

**Why this matters:**
```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

text = "The quick brown fox jumps over the lazy dog"
tokens = enc.encode(text)
print(f"Text: {text}")
print(f"Token count: {len(tokens)}")   # ~10 tokens
print(f"Token IDs: {tokens}")
print(f"Decoded tokens: {[enc.decode([t]) for t in tokens]}")
```

**Cost implications:**
- GPT-4o input: ~$0.0025 per 1K tokens ($2.50 per 1M)
- GPT-4o output: ~$0.01 per 1K tokens ($10 per 1M)
- A 10,000 word document ≈ ~13,000 tokens ≈ $0.033 per query
- Scale to 1,000 users/day: $33/day just in retrieval context

**This is why context engineering matters more than prompt engineering.**

---

### 3.2 — The Attention Mechanism (The Heart of Everything)

The transformer's breakthrough is **attention** — the ability for every token to "look at" every other token and decide how relevant they are to each other.

**The intuition:**
Imagine reading the sentence: *"The animal didn't cross the street because it was too tired."*

What does "it" refer to? The animal, or the street? You as a human know it's "the animal" because you attend to the right word.

Attention lets the model do this — mathematically.

**What you need to know (without the math):**
- **Self-attention:** Each token computes how much it should "attend to" every other token
- **Multi-head attention:** The model does this multiple times in parallel with different "views"
- **Why deeper = smarter:** More layers = more complex patterns can be learned
- **Context window = attention limit:** The model can only attend to tokens within the window

**The implications for engineering:**
1. The beginning and end of context get more attention than the middle (the "lost in the middle" problem)
2. Longer context = quadratically more computation = costs more
3. Order matters — put the most important context first or last, not in the middle

---

### 3.3 — Temperature & Sampling (Controlling Output)

After the model computes token probabilities, it chooses the next token. **Temperature** controls how it chooses.

```python
from openai import OpenAI
client = OpenAI()

def compare_temperatures(prompt: str):
    temperatures = [0.0, 0.7, 1.5]
    
    for temp in temperatures:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=temp,
            max_tokens=100
        )
        print(f"\n--- Temperature: {temp} ---")
        print(response.choices[0].message.content)

compare_temperatures("Tell me a one-sentence story about a robot")
```

**Temperature guide:**
| Temperature | Behavior | Use For |
|-------------|----------|---------|
| `0.0` | Deterministic (almost always the same output) | Extraction, classification, factual QA |
| `0.3–0.7` | Balanced — coherent but varied | Most chatbots, assistants |
| `0.8–1.2` | More creative, more random | Creative writing, brainstorming |
| `> 1.5` | Increasingly chaotic | Rarely useful in production |

**Other sampling parameters:**
- `top_p` (nucleus sampling): Only sample from the top P% probability tokens. Use `temperature` OR `top_p`, not both.
- `max_tokens`: Hard limit on output length. Always set this in production.
- `stop`: Stop generating when these sequences appear. Useful for structured outputs.
- `presence_penalty` / `frequency_penalty`: Reduce repetition.

---

### 3.4 — Embeddings (The Geometric Representation of Meaning)

Embeddings are vectors (lists of numbers) that represent text in a multi-dimensional space where **similar meanings are geometrically close**.

```
"King" ≈ [0.23, -0.11, 0.87, ..., 0.03]  (1536 numbers for OpenAI)
"Queen" ≈ [0.19, -0.09, 0.91, ..., 0.01]  (very close!)
"Apple" ≈ [-0.45, 0.78, -0.12, ..., 0.67]  (far away)
```

**The famous analogy:**
```
King - Man + Woman ≈ Queen
```

This works because embeddings capture semantic relationships geometrically.

**Creating embeddings:**
```python
from openai import OpenAI
import numpy as np

client = OpenAI()

def get_embedding(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def cosine_similarity(a: list[float], b: list[float]) -> float:
    """Measure similarity between two embeddings (1 = identical, 0 = unrelated)"""
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Compare semantic similarity
e1 = get_embedding("I love programming in Python")
e2 = get_embedding("I enjoy coding with Python")
e3 = get_embedding("I hate mangoes")

print(f"Similar sentences: {cosine_similarity(e1, e2):.3f}")  # ~0.95
print(f"Different sentences: {cosine_similarity(e1, e3):.3f}")  # ~0.3
```

**Why embeddings matter for AI Engineering:**
- The entire RAG (Retrieval Augmented Generation) system is built on embeddings
- You embed your documents, embed the user's question, find the closest documents
- This is semantic search — not keyword search

---

### 3.5 — The Context Window: Your Most Important Resource

The context window is the total amount of text the model can "see" at once — both the input and the output.

**What goes in the context window:**
```
[System Prompt] + [Conversation History] + [Retrieved Documents (RAG)] + [User Message]
       ↓
[Model's Generated Response]
       ↓
Total must be < Context Window Size
```

**Context window sizes (2025):**
| Model | Context Window | Practical Use |
|-------|---------------|--------------|
| GPT-4o | 128k tokens | ~96k words |
| Claude 3.5 Sonnet | 200k tokens | ~150k words |
| Gemini 1.5 Pro | 1M tokens | ~750k words |
| GPT-4o-mini | 128k tokens | Cheap, fast |
| Llama 3.2 (local) | 128k tokens | Free, private |

**The "Lost in the Middle" Problem:**
Research shows that models pay less attention to content in the middle of a long context. Put the most important information at the beginning or end of your context.

---

### 3.6 — Why Hallucinations Happen (And What To Do About It)

**Root cause:** LLMs predict the *most likely next token*, not the *most true next token*. If the training data contained wrong information stated confidently, the model learned to state it confidently too.

**Types of hallucinations:**
1. **Factual hallucination** — Inventing facts ("The Eiffel Tower was built in 1799")
2. **Source hallucination** — Citing papers/books that don't exist
3. **Reasoning hallucination** — Confident but wrong math or logic
4. **Sycophantic hallucination** — Agreeing with the user even when they're wrong

**How AI Engineers handle this:**
1. **RAG** — Give the model the actual facts, ask it to answer from those
2. **Structured outputs** — Force the model to cite its sources
3. **Self-consistency** — Ask the same question multiple ways, check if answers agree
4. **Confidence scoring** — Ask the model to rate its own confidence
5. **Evals** — Test for hallucinations in CI/CD (covered in Module 16)

---

### 📚 Resources

**Watch:**
- 🎥 [3Blue1Brown — "But what is a GPT?"](https://www.youtube.com/watch?v=wjZofJX0v4M) *(27 min — best visual explanation)*
- 🎥 [Andrej Karpathy — "Let's build GPT from scratch"](https://www.youtube.com/watch?v=kCc8FmEb1nY) *(3.5 hrs — optional deep dive)*
- 🎥 [Computerphile — Attention in Transformers](https://www.youtube.com/watch?v=yGTUuEx3GkA)

**Read:**
- 📄 [The Illustrated Transformer — Jay Alammar](https://jalammar.github.io/illustrated-transformer/) *(Read this. Visualize everything.)*
- 📄 [The Lost in the Middle Problem — Stanford paper](https://arxiv.org/abs/2307.03172)
- 📄 [OpenAI Tokenizer](https://platform.openai.com/tokenizer) *(Play with this tool)*

---

### 🎯 Coding Challenges

**Challenge 1 — Token Counter:**
Build a function that takes a conversation history (list of messages) and returns:
- Total token count
- Estimated cost for GPT-4o
- Warning if total > 100k tokens

**Challenge 2 — Temperature Explorer:**
Build a small script that sends the same creative prompt 5 times at temperature 0.0, 0.5, and 1.0 and saves all outputs to compare. Write observations about what changed.

**Challenge 3 — Semantic Similarity Search:**
- Embed these 10 sentences about programming languages
- Given a new query sentence, find the 3 most similar
- Use cosine similarity (no vector database yet — just numpy)

**Challenge 4 — Context Window Stress Test:**
- Build a conversation that gets progressively longer
- Track when the model starts "forgetting" things from early in the conversation
- Document your observations

---

### 🚦 Module 03 Gate

Before moving to Phase 1, you should be able to explain:
- What a token is and calculate the token count + cost for a given text
- What temperature does and when to set it low vs high
- What embeddings are and why cosine similarity works
- Why LLMs hallucinate and 3 strategies to mitigate it
- What "lost in the middle" means and how to design around it

**If you can teach this to someone else — proceed.**

---
---

# PHASE 1: WORKING WITH LLMs

---

## Module 04 — Prompt Engineering (The Real Kind)

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐☆☆

> *Most "prompt engineering" tutorials teach you tricks. This module teaches you the science — why certain patterns work, when to use each technique, and how to evaluate if your prompt is actually better.*

---

### 4.1 — Prompt Engineering Is Context Engineering

The biggest misconception: prompt engineering is about writing clever prompts.

The reality: **prompt engineering is about giving the model exactly the right context to produce the output you need.**

Think of it like this: You're not telling the model what to do. You're setting up a situation where the right output is the most probable next sequence of tokens.

**The Anatomy of a Great Prompt:**

```python
def build_prompt(
    task_description: str,      # What you want done
    context: str,               # Background information
    examples: list[dict],       # Few-shot examples
    constraints: list[str],     # Rules and limitations
    output_format: str,         # Exactly what you want back
    user_input: str             # The actual query
) -> list[dict]:
    
    system_message = f"""
{task_description}

CONTEXT:
{context}

CONSTRAINTS:
{chr(10).join(f"- {c}" for c in constraints)}

OUTPUT FORMAT:
{output_format}
    """.strip()
    
    messages = [{"role": "system", "content": system_message}]
    
    # Add few-shot examples
    for example in examples:
        messages.append({"role": "user", "content": example["input"]})
        messages.append({"role": "assistant", "content": example["output"]})
    
    # Add actual query
    messages.append({"role": "user", "content": user_input})
    
    return messages
```

---

### 4.2 — The Core Techniques (With When to Use Each)

#### Zero-Shot Prompting
No examples. Just the task. Use when the task is straightforward and well-understood.

```python
messages = [
    {"role": "system", "content": "You are a sentiment classifier. Classify the sentiment as POSITIVE, NEGATIVE, or NEUTRAL."},
    {"role": "user", "content": "This product is absolutely terrible. Complete waste of money."}
]
# Works well because "classify sentiment" is a common pattern in training data
```

#### Few-Shot Prompting
Provide 2–5 examples of input→output pairs. Use when zero-shot gives inconsistent results.

```python
messages = [
    {"role": "system", "content": "Extract the company name and funding amount from news snippets."},
    
    # Example 1
    {"role": "user", "content": "Acme Corp announced a $50M Series B round led by Sequoia."},
    {"role": "assistant", "content": '{"company": "Acme Corp", "amount": "$50M", "round": "Series B"}'},
    
    # Example 2
    {"role": "user", "content": "TechStartup raised $12 million from Y Combinator."},
    {"role": "assistant", "content": '{"company": "TechStartup", "amount": "$12M", "round": null}'},
    
    # Actual query
    {"role": "user", "content": "Bangalore-based Fintech startup MoenyFlow secured $8M in seed funding from Lightspeed India."},
]
# Now the model knows exactly what format you expect
```

#### Chain-of-Thought (CoT) Prompting
Tell the model to "think step by step" before answering. Use for reasoning tasks.

```python
messages = [
    {
        "role": "system", 
        "content": """You are a math tutor. When solving problems:
1. First, identify what we're solving for
2. List the given information
3. Show each calculation step
4. State the final answer clearly

Always think through the problem before giving the answer."""
    },
    {
        "role": "user",
        "content": "A train travels from Mumbai to Pune, a distance of 150km. It travels at 60km/h for the first hour, then speeds up to 90km/h. How long does the total journey take?"
    }
]
# Without CoT, models often get multi-step math wrong
# With CoT, they reason step by step and get it right
```

#### Role-Based Prompting
Give the model a specific persona. Use to shift the style, expertise level, or perspective.

```python
messages = [
    {
        "role": "system",
        "content": """You are a senior backend engineer at a Series B startup with 10 years of experience in distributed systems. You give direct, opinionated advice. You don't sugarcoat — if someone's approach is wrong, you say so. You ask clarifying questions before diving into solutions."""
    },
    {"role": "user", "content": "Should I use Redis or Memcached for my caching layer?"}
]
```

#### The "Think before you answer" Pattern
A powerful trick: ask the model to think privately, then answer. This uses "scratchpad" reasoning.

```python
messages = [
    {
        "role": "system",
        "content": """When answering complex questions:
1. First, use <thinking> tags to reason through the problem
2. Then provide your final answer after </thinking>

Your thinking is private — use it to explore, question, and work through the problem honestly.
Your final answer should be clear and confident."""
    },
    {"role": "user", "content": "Is it better to use a monorepo or separate repos for a 5-person startup?"}
]
```

---

### 4.3 — System Prompts: The Foundation

The system prompt is where you establish the model's role, constraints, and behavior. This is the most important part of your prompt.

**Great system prompt structure:**
```
1. ROLE: Who the model is and its expertise
2. TASK: What it's here to do
3. CONTEXT: Background it needs to know
4. CONSTRAINTS: What it must not do / limits
5. OUTPUT FORMAT: Exactly what you want back
6. TONE: How it should communicate
```

**Example — Customer Support Bot:**
```python
system_prompt = """
## Role
You are Priya, a customer support specialist for CloudStore, India's leading e-commerce platform. You have deep knowledge of CloudStore's policies, products, and systems.

## Your Goal
Help customers resolve their issues quickly and leave them feeling heard and valued.

## What You Know
- Delivery typically takes 3-5 business days
- Returns are accepted within 30 days with receipt
- Customer service hours: 9am–6pm IST, Monday–Saturday
- Escalation email: support@cloudstore.in

## Constraints
- Never promise refunds or replacements — you can "initiate a review"
- Never speculate about other companies' products
- If you don't know something, say "Let me check that for you" and ask for relevant details
- Keep responses under 150 words unless the issue requires more

## Output Format
Always end your response with:
- Next steps for the customer (if any)
- A reference number format: CS-[timestamp last 6 digits]

## Tone
Professional but warm. Like a helpful friend who works at CloudStore, not a formal corporate robot.
"""
```

---

### 4.4 — Prompt Anti-Patterns (What to Avoid)

These are common mistakes that produce bad results:

**❌ Vague instructions:**
```
"Be helpful and answer questions"
```

**✅ Specific instructions:**
```
"Answer questions about our product catalog. If the question is not about our products, politely redirect to our product pages. Always recommend at most 3 products."
```

---

**❌ Telling the model what NOT to do:**
```
"Don't give medical advice"
"Don't be rude"
"Don't make things up"
```

**✅ Tell it what TO do instead:**
```
"If medical questions arise, recommend consulting a qualified doctor"
"Always maintain a respectful, professional tone"
"Only state facts you're certain of; say 'I'm not sure' when uncertain"
```

---

**❌ Leaving output format to chance:**
```
"Analyze this customer feedback"
```

**✅ Define exact output format:**
```
"Analyze this customer feedback and return a JSON object with:
- sentiment: POSITIVE | NEGATIVE | NEUTRAL
- key_topics: list of 1-3 main topics mentioned
- urgency: LOW | MEDIUM | HIGH
- suggested_response: a 1-sentence suggested reply"
```

---

**❌ Not testing with edge cases:**
```
# Tested only with polite, clear customer messages
# Deployed to production
# Customer sends: "GIVE ME MY MONEY BACK NOW YOU SCAMMERS"
# Model panics and gives weird response
```

**✅ Red team your prompts:**
Test with: offensive inputs, jailbreak attempts, off-topic questions, empty inputs, very long inputs, inputs in other languages.

---

### 4.5 — Evaluating Prompts (The Part Everyone Skips)

Prompt engineering without evaluation is guesswork. You need to measure if prompt B is actually better than prompt A.

**Simple A/B prompt testing framework:**
```python
import json
from openai import OpenAI
from typing import Callable

client = OpenAI()

def evaluate_prompt(
    prompt_a: str,
    prompt_b: str,
    test_cases: list[dict],
    evaluator_fn: Callable[[str, str], float]  # Returns score 0-1
) -> dict:
    """
    Compare two prompts on a set of test cases.
    evaluator_fn takes (expected_output, actual_output) and returns a score.
    """
    scores_a, scores_b = [], []
    
    for case in test_cases:
        # Test Prompt A
        response_a = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": prompt_a},
                {"role": "user", "content": case["input"]}
            ]
        )
        score_a = evaluator_fn(case["expected"], response_a.choices[0].message.content)
        scores_a.append(score_a)
        
        # Test Prompt B
        response_b = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": prompt_b},
                {"role": "user", "content": case["input"]}
            ]
        )
        score_b = evaluator_fn(case["expected"], response_b.choices[0].message.content)
        scores_b.append(score_b)
    
    return {
        "prompt_a_avg_score": sum(scores_a) / len(scores_a),
        "prompt_b_avg_score": sum(scores_b) / len(scores_b),
        "winner": "A" if sum(scores_a) > sum(scores_b) else "B",
        "individual_scores": list(zip(scores_a, scores_b))
    }

# Example: Evaluate sentiment classification prompts
def exact_match_evaluator(expected: str, actual: str) -> float:
    return 1.0 if expected.strip().upper() in actual.upper() else 0.0
```

This is the foundation of the eval systems you'll build in Module 16.

---

### 📚 Resources

**Watch:**
- 🎥 [OpenAI DevDay — Prompt Engineering Deep Dive](https://www.youtube.com/watch?v=ahnGLM-RC1Y)
- 🎥 [Anthropic — Claude Prompt Engineering Guide](https://www.youtube.com/watch?v=1FWRkWwnqRg)
- 🎥 [DeepLearning.AI — ChatGPT Prompt Engineering for Developers](https://learn.deeplearning.ai/courses/chatgpt-prompt-eng-for-developers)

**Read:**
- 📄 [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- 📄 [Anthropic Prompt Engineering Documentation](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- 📄 [Prompt Injection Attacks — Simon Willison](https://simonwillison.net/2023/Apr/14/worst-that-could-happen/)

**GitHub:**
- ⭐ [Awesome Prompt Engineering](https://github.com/promptslab/Awesome-Prompt-Engineering)
- ⭐ [OpenAI Cookbook](https://github.com/openai/openai-cookbook)

---

### 🎯 Coding Challenges

**Challenge 1 — Prompt Comparison:**
Build a script that tests 3 different system prompts for a customer support bot on 10 test cases. Automatically scores them and picks the winner.

**Challenge 2 — CoT vs Direct:**
For a set of 5 math word problems, compare:
- Direct answer (no CoT)
- "Think step by step" CoT
- Structured CoT (with numbered steps)
Measure accuracy for each.

**Challenge 3 — Jailbreak Defense:**
Write a prompt for a children's homework helper. Then try 10 different jailbreak attempts. Log which ones succeed. Improve your system prompt to handle them.

**Challenge 4 — Format Controller:**
Write a prompt that consistently outputs valid JSON for a product review analyzer. The output must have specific fields. Test with 20 reviews. How often does it fail format? Fix the failures.

---

### 🏗️ Portfolio Project 01 — AI Content Studio

**What you'll build:** A web tool that helps content creators write better blog posts.

**Features:**
1. **Tone analyzer** — Analyzes a draft for tone (formal/casual/technical)
2. **Headline generator** — Generates 5 headline variations from a blog outline
3. **SEO optimizer** — Suggests keyword-improved rewrites of paragraphs
4. **Readability checker** — Flags complex sentences and suggests simplifications

**Tech:**
- FastAPI backend with prompt templates
- Next.js frontend with streaming
- Each feature uses a different prompting technique (CoT, few-shot, role-based)

**Evaluation component:**
- Each feature has a `test_prompt.py` that runs 20 test cases
- Build a simple eval dashboard showing accuracy per feature

**Push to:** `ai-engineering-journey/projects/02-ai-content-studio/`

This is a real-world product you could sell. Make it great.

---

### 🧪 Module 04 Knowledge Check

1. What is the difference between zero-shot and few-shot prompting? Give an example of when you'd use each.
2. Why is Chain-of-Thought prompting effective for reasoning tasks?
3. What are 3 anti-patterns in prompt engineering?
4. How would you measure if prompt B is better than prompt A?
5. What is prompt injection and how would you defend against it?
6. Build a system prompt for an AI that helps junior developers debug Python code. Include all 6 elements of a great system prompt.

---

### 🚦 Module 04 Gate

Before moving to Module 05:
- ✅ AI Content Studio is functional with at least 2 features working end-to-end
- ✅ Each feature has a test script with at least 10 test cases
- ✅ You can explain why you chose each prompting technique for each feature
- ✅ You have written prompts and tested for at least 5 edge cases / failure modes

---

## 🔁 Phase 0 Revision Checkpoint

Before moving deeper, take 30 minutes to review:

1. **Flashcard Review:** Create flashcards for: token, embedding, temperature, context window, few-shot, CoT, hallucination. Review using Anki or any flashcard tool.

2. **Explain-It-Back:** Record yourself (even just audio) explaining: "What is an LLM and why do hallucinations happen?" in under 2 minutes.

3. **Code Review:** Look at your `aiask` CLI and Content Studio. What would you improve now that you know more?

4. **Learning Journal:** Write 3 things that surprised you and 2 things you're still confused about.

**If you're confused about something — don't proceed. Go back and fix the gap. This is the most important advice in this entire course.**

---

*[End of Part 1 — Modules 01–04]*
*Next: Module 05 — LLM APIs & SDKs | Module 06 — Structured Outputs | Module 07 — Embeddings & Semantic Search*
---

# PHASE 1 (CONTINUED): WORKING WITH LLMs

---

## Module 05 — LLM APIs & SDKs: OpenAI, Anthropic, Google

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐☆☆

> *Every major LLM provider has slightly different APIs, pricing, strengths, and failure modes. An AI Engineer knows them all and picks the right one for the job — not just defaults to GPT-4.*

---

### 5.1 — The Provider Landscape (2025)

| Provider | Model | Best For | Context | Pricing |
|----------|-------|----------|---------|---------|
| **OpenAI** | GPT-4o | General purpose, function calling | 128k | $$$ |
| **OpenAI** | GPT-4o-mini | Fast, cheap tasks | 128k | $ |
| **Anthropic** | Claude 3.5 Sonnet | Long documents, coding, safety | 200k | $$ |
| **Anthropic** | Claude 3 Haiku | Fast, cheap | 200k | $ |
| **Google** | Gemini 1.5 Pro | Massive context, multimodal | 1M | $$ |
| **Google** | Gemini 1.5 Flash | Fast cheap multimodal | 1M | $ |
| **Meta/Local** | Llama 3.2 via Ollama | Free, private, no internet | 128k | Free |
| **Mistral** | Mistral Large | European data residency | 32k | $$ |

**When to pick which:**
- Fastest, cheapest for simple tasks → GPT-4o-mini or Gemini Flash
- Best for very long documents → Claude 3.5 Sonnet (200k context)
- Best when you need 1M token context → Gemini 1.5 Pro
- Privacy-critical (no data leaving your infra) → Llama via Ollama
- Best reasoning → GPT-4o, Claude 3.5 Sonnet, Gemini 1.5 Pro (roughly equal)

---

### 5.2 — OpenAI SDK — Production Patterns

**Basic setup with proper configuration:**
```python
from openai import OpenAI, AsyncOpenAI
import os

# Always use a configured client — not ad-hoc calls
client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY"),
    timeout=30.0,        # Always set timeouts
    max_retries=3,       # Built-in retry logic
)

# The canonical completion call with all important parameters
def complete(
    messages: list[dict],
    model: str = "gpt-4o-mini",
    temperature: float = 0.0,
    max_tokens: int = 1000,
    response_format: dict | None = None,  # For JSON mode
) -> str:
    kwargs = dict(
        model=model,
        messages=messages,
        temperature=temperature,
        max_tokens=max_tokens,
    )
    if response_format:
        kwargs["response_format"] = response_format
    
    response = client.chat.completions.create(**kwargs)
    return response.choices[0].message.content
```

**Vision / Multimodal with GPT-4o:**
```python
import base64
from pathlib import Path

def analyze_image(image_path: str, question: str) -> str:
    # Read and encode image
    image_data = base64.b64encode(Path(image_path).read_bytes()).decode('utf-8')
    
    # Detect file type
    extension = Path(image_path).suffix.lower()
    media_type_map = {'.jpg': 'jpeg', '.jpeg': 'jpeg', '.png': 'png', '.webp': 'webp'}
    media_type = f"image/{media_type_map.get(extension, 'jpeg')}"
    
    response = client.chat.completions.create(
        model="gpt-4o",  # Must use 4o for vision
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:{media_type};base64,{image_data}",
                        "detail": "high"  # "low" for cheaper, faster; "high" for detailed analysis
                    }
                },
                {"type": "text", "text": question}
            ]
        }],
        max_tokens=500
    )
    return response.choices[0].message.content

# Usage
result = analyze_image("receipt.jpg", "What is the total amount on this receipt?")
```

**Batch API — for offline processing:**
```python
import json

# Use Batch API for large-scale processing at 50% cost reduction
def create_batch_job(prompts: list[str]) -> str:
    requests = []
    for i, prompt in enumerate(prompts):
        requests.append({
            "custom_id": f"request-{i}",
            "method": "POST",
            "url": "/v1/chat/completions",
            "body": {
                "model": "gpt-4o-mini",
                "messages": [{"role": "user", "content": prompt}],
                "max_tokens": 500
            }
        })
    
    # Write to JSONL file
    with open("batch_requests.jsonl", "w") as f:
        for req in requests:
            f.write(json.dumps(req) + "\n")
    
    # Upload and create batch
    with open("batch_requests.jsonl", "rb") as f:
        batch_file = client.files.create(file=f, purpose="batch")
    
    batch = client.batches.create(
        input_file_id=batch_file.id,
        endpoint="/v1/chat/completions",
        completion_window="24h"
    )
    return batch.id
```

---

### 5.3 — Anthropic Claude SDK

Claude is particularly good at: following complex instructions, long-document analysis, safe outputs, and coding.

```python
import anthropic
import os

client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

# Basic completion
def claude_complete(
    user_message: str,
    system_prompt: str = "",
    model: str = "claude-3-5-sonnet-20241022",
    max_tokens: int = 1000,
    temperature: float = 0.0
) -> str:
    message = client.messages.create(
        model=model,
        max_tokens=max_tokens,
        temperature=temperature,
        system=system_prompt,  # Note: separate from messages in Claude SDK
        messages=[{"role": "user", "content": user_message}]
    )
    return message.content[0].text

# Streaming with Claude
def claude_stream(user_message: str, system_prompt: str = ""):
    with client.messages.stream(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1000,
        system=system_prompt,
        messages=[{"role": "user", "content": user_message}]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
        print()
        
        # Get full message after streaming
        return stream.get_final_message()

# Document analysis with PDF (Claude's strength)
def analyze_pdf(pdf_path: str, question: str) -> str:
    import base64
    pdf_data = base64.standard_b64encode(open(pdf_path, "rb").read()).decode("utf-8")
    
    message = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {
                        "type": "base64",
                        "media_type": "application/pdf",
                        "data": pdf_data,
                    }
                },
                {"type": "text", "text": question}
            ]
        }]
    )
    return message.content[0].text
```

---

### 5.4 — Google Gemini SDK

```python
import google.generativeai as genai
import os

genai.configure(api_key=os.getenv("GOOGLE_API_KEY"))

# Create model instance
model = genai.GenerativeModel(
    model_name="gemini-1.5-flash",
    system_instruction="You are a helpful assistant.",
    generation_config=genai.GenerationConfig(
        temperature=0.0,
        max_output_tokens=1000,
    )
)

# Basic completion
def gemini_complete(prompt: str) -> str:
    response = model.generate_content(prompt)
    return response.text

# Streaming
def gemini_stream(prompt: str):
    for chunk in model.generate_content(prompt, stream=True):
        print(chunk.text, end="", flush=True)

# Multi-turn chat (Gemini manages history internally)
chat = model.start_chat(history=[])

def gemini_chat(user_message: str) -> str:
    response = chat.send_message(user_message)
    return response.text

# Gemini's superpower: 1M context window
# You can literally pass a 700-page book as context
def analyze_large_document(document_text: str, question: str) -> str:
    prompt = f"""
Here is a document to analyze:

<document>
{document_text}
</document>

Question: {question}

Provide a comprehensive answer based only on the document.
    """
    response = model.generate_content(prompt)
    return response.text
```

---

### 5.5 — Local LLMs with Ollama (Zero Cost, Full Privacy)

Ollama lets you run open-source models locally. Essential for: development (save API costs), privacy-critical data, and offline use.

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull models (like docker pull)
ollama pull llama3.2          # Meta's Llama 3.2 (3B)
ollama pull mistral           # Mistral 7B
ollama pull phi3              # Microsoft Phi-3 (small but capable)
ollama pull nomic-embed-text  # Great embedding model

# Run interactively
ollama run llama3.2
```

```python
from openai import OpenAI  # Ollama is OpenAI-API compatible!

# Point OpenAI client at local Ollama
local_client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama",  # Required but not used
)

def local_complete(prompt: str, model: str = "llama3.2") -> str:
    response = local_client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0
    )
    return response.choices[0].message.content

# This works the same as OpenAI — you can swap providers easily
# Great pattern: use local for dev, OpenAI for production
```

---

### 5.6 — The Provider Abstraction Pattern

Professional AI apps don't hardcode a single provider. They abstract it.

```python
from enum import Enum
from typing import Protocol
from openai import OpenAI
import anthropic

class Provider(Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    LOCAL = "local"

class LLMProvider(Protocol):
    def complete(self, messages: list[dict], **kwargs) -> str: ...
    def stream(self, messages: list[dict], **kwargs): ...

class OpenAIProvider:
    def __init__(self, model: str = "gpt-4o-mini"):
        self.client = OpenAI()
        self.model = model
    
    def complete(self, messages: list[dict], **kwargs) -> str:
        response = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            **kwargs
        )
        return response.choices[0].message.content
    
    def stream(self, messages: list[dict], **kwargs):
        stream = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            stream=True,
            **kwargs
        )
        for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

class AnthropicProvider:
    def __init__(self, model: str = "claude-3-5-sonnet-20241022"):
        self.client = anthropic.Anthropic()
        self.model = model
    
    def complete(self, messages: list[dict], **kwargs) -> str:
        # Extract system message if present
        system = ""
        filtered = []
        for m in messages:
            if m["role"] == "system":
                system = m["content"]
            else:
                filtered.append(m)
        
        response = self.client.messages.create(
            model=self.model,
            system=system,
            messages=filtered,
            max_tokens=kwargs.get("max_tokens", 1000),
        )
        return response.content[0].text

def get_provider(provider: Provider, **kwargs) -> LLMProvider:
    if provider == Provider.OPENAI:
        return OpenAIProvider(**kwargs)
    elif provider == Provider.ANTHROPIC:
        return AnthropicProvider(**kwargs)
    elif provider == Provider.LOCAL:
        return OpenAIProvider(
            model=kwargs.get("model", "llama3.2"),
            # Override base_url in __init__
        )

# Usage — swap providers with one line
llm = get_provider(Provider.OPENAI)
# llm = get_provider(Provider.ANTHROPIC)  # Switch easily
result = llm.complete([{"role": "user", "content": "Hello!"}])
```

---

### 📚 Resources

**Watch:**
- 🎥 [OpenAI API — Full Tutorial 2025](https://www.youtube.com/watch?v=OB99E7Y1cMA)
- 🎥 [Build with Claude — Anthropic Developer Tutorials](https://www.youtube.com/@anthropic-ai)
- 🎥 [Ollama — Full Guide to Local LLMs](https://www.youtube.com/watch?v=Lb3DvK5A-pA)

**Read:**
- 📄 [OpenAI Platform Docs](https://platform.openai.com/docs)
- 📄 [Anthropic API Docs](https://docs.anthropic.com)
- 📄 [Google AI Studio](https://aistudio.google.com)

---

### 🎯 Coding Challenges

**Challenge 1 — Provider Comparison:**
Send the same 10 prompts to GPT-4o-mini, Claude 3 Haiku, and Gemini Flash. Compare: response quality, latency, token count, cost estimate.

**Challenge 2 — Multi-Provider Fallback:**
Build a function that tries Provider A, falls back to Provider B if it fails, then Provider C. Log each attempt.

**Challenge 3 — Local Dev Setup:**
Get Ollama running with Llama 3.2. Build the same chat interface from Module 02's mini-project but using Ollama. Verify it works offline.

**Challenge 4 — Vision Task:**
Build a receipt parser that takes a photo of a receipt and returns structured JSON with items and prices. Compare GPT-4o vs Gemini 1.5 Flash.

---

### 🏗️ Mini-Project: Universal LLM Gateway

Build a simple API gateway (FastAPI) that:
- Accepts standard OpenAI-format requests
- Routes to the cheapest available provider for simple tasks
- Routes to the most capable provider for complex tasks
- Falls back automatically on failure
- Logs every request with provider, latency, tokens, cost estimate

This is a simplified version of what Portkey and LiteLLM do.

---

## Module 06 — Structured Outputs & Function Calling

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐☆☆

> *The moment you need an LLM to return reliable, parseable data — not just text — you need structured outputs. This is essential for any production AI feature.*

---

### 6.1 — The Problem: LLMs Return Text, You Need Data

```python
# Without structured outputs — fragile
response = "The sentiment is POSITIVE with a confidence of 0.87"
# Now you have to parse this string... 
# What if it says "The sentiment is Positive" (capital)?
# What if it adds an explanation before the sentiment?
# This is a maintenance nightmare.

# With structured outputs — reliable
response = '{"sentiment": "POSITIVE", "confidence": 0.87}'
import json
data = json.loads(response)  # Always works
```

---

### 6.2 — JSON Mode

The simplest structured output — force the model to always return valid JSON:

```python
from openai import OpenAI
from pydantic import BaseModel
import json

client = OpenAI()

class SentimentResult(BaseModel):
    sentiment: str  # POSITIVE | NEGATIVE | NEUTRAL
    confidence: float
    key_phrases: list[str]
    summary: str

def analyze_sentiment(text: str) -> SentimentResult:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": f"""Analyze sentiment and return a JSON object with:
- sentiment: "POSITIVE", "NEGATIVE", or "NEUTRAL"
- confidence: float between 0 and 1
- key_phrases: list of phrases that determined the sentiment (max 3)
- summary: one-sentence summary of the sentiment

JSON only. No other text."""
            },
            {"role": "user", "content": text}
        ],
        response_format={"type": "json_object"},  # Forces valid JSON
        temperature=0.0
    )
    
    raw = json.loads(response.choices[0].message.content)
    return SentimentResult(**raw)  # Validates with Pydantic

result = analyze_sentiment("This product is amazing! Works perfectly and arrived fast.")
print(result.sentiment)     # POSITIVE
print(result.confidence)    # 0.95
print(result.key_phrases)   # ['amazing', 'works perfectly', 'arrived fast']
```

---

### 6.3 — Structured Outputs (OpenAI's Strict Mode)

Even better than JSON mode — the model is guaranteed to match your exact schema:

```python
from pydantic import BaseModel
from typing import List, Optional

class Article(BaseModel):
    title: str
    summary: str
    tags: List[str]
    estimated_read_time_minutes: int
    difficulty: str  # "BEGINNER" | "INTERMEDIATE" | "ADVANCED"
    key_takeaways: List[str]
    is_technical: bool

def analyze_article(article_text: str) -> Article:
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Analyze the provided article and extract structured information."},
            {"role": "user", "content": article_text}
        ],
        response_format=Article,  # Pass Pydantic model directly!
        temperature=0.0
    )
    
    # response.choices[0].message.parsed is already an Article object!
    return response.choices[0].message.parsed

article = analyze_article("... long article text ...")
print(article.title)          # Already typed and validated
print(article.difficulty)     # "INTERMEDIATE"
print(article.is_technical)   # True
```

---

### 6.4 — Function Calling (The Foundation of Agents)

Function calling lets the LLM tell your application to call a function with specific arguments. This is how AI agents interact with real systems.

```python
import json
from openai import OpenAI

client = OpenAI()

# Define functions the model can call
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a city",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "City name, e.g. 'Mumbai'"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit"
                    }
                },
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_products",
            "description": "Search for products in the catalog",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "max_price": {"type": "number"},
                    "category": {"type": "string"}
                },
                "required": ["query"]
            }
        }
    }
]

# Your actual functions (these do real work)
def get_weather(city: str, unit: str = "celsius") -> dict:
    # In reality: call a weather API
    return {"city": city, "temp": 28, "unit": unit, "condition": "Sunny"}

def search_products(query: str, max_price: float = None, category: str = None) -> list:
    # In reality: query your database
    return [{"name": "Product A", "price": 499, "category": "Electronics"}]

# The agentic loop
def agent_loop(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]
    
    while True:
        # Ask the model what to do
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools,
            tool_choice="auto"  # Let model decide when to call functions
        )
        
        message = response.choices[0].message
        messages.append(message)  # Add assistant's response to history
        
        # Did the model decide to call a function?
        if message.tool_calls:
            for tool_call in message.tool_calls:
                func_name = tool_call.function.name
                func_args = json.loads(tool_call.function.arguments)
                
                print(f"[Calling: {func_name}({func_args})]")
                
                # Execute the function
                if func_name == "get_weather":
                    result = get_weather(**func_args)
                elif func_name == "search_products":
                    result = search_products(**func_args)
                else:
                    result = {"error": "Unknown function"}
                
                # Add function result to messages
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result)
                })
        else:
            # Model finished — return final answer
            return message.content

result = agent_loop("What's the weather in Mumbai? Also find me electronics under ₹1000.")
print(result)
```

---

### 6.5 — Instructor: The Production Library for Structured Outputs

The `instructor` library makes structured outputs much cleaner in production:

```python
import instructor
from anthropic import Anthropic
from openai import OpenAI
from pydantic import BaseModel, Field
from typing import List, Literal

# Works with both OpenAI and Anthropic!
openai_client = instructor.from_openai(OpenAI())
anthropic_client = instructor.from_anthropic(Anthropic())

class ExtractedCompany(BaseModel):
    name: str
    industry: str
    funding_amount: str = Field(description="e.g. '$50M', '$1.2B'")
    funding_round: Literal["Seed", "Series A", "Series B", "Series C", "IPO", "Unknown"]
    investors: List[str] = Field(default_factory=list)
    headquarters: str = "Unknown"

# Clean, type-safe extraction
def extract_company_info(text: str) -> ExtractedCompany:
    return openai_client.chat.completions.create(
        model="gpt-4o-mini",
        response_model=ExtractedCompany,  # instructor magic
        messages=[{"role": "user", "content": f"Extract company funding info from: {text}"}],
        temperature=0.0
    )

company = extract_company_info("""
Bengaluru-based fintech startup Razorpay raised $375 million in Series F funding 
co-led by Lone Pine Capital and Alkeon Capital, with participation from Tiger Global.
""")

print(company.name)             # Razorpay
print(company.funding_amount)   # $375M
print(company.funding_round)    # Series F -> Closest: "Series C" or custom
print(company.investors)        # ['Lone Pine Capital', 'Alkeon Capital', 'Tiger Global']
```

---

### 🎯 Coding Challenges

**Challenge 1 — Invoice Parser:**
Build an invoice extractor that takes any invoice (text or image) and returns:
- Vendor name, date, total amount
- Line items (description, quantity, unit price)
- Tax amount, payment terms
Use Pydantic validation to ensure amounts are numbers, dates are parseable.

**Challenge 2 — Multi-Step Tool Use:**
Build a research assistant that:
1. Can search Wikipedia (use `wikipedia` python package)
2. Can fetch a URL (use `httpx`)
3. Can calculate date differences
4. Answers questions using these tools

**Challenge 3 — Schema Validation:**
Take 100 product descriptions. Extract structured data (name, price, category, features). Measure: how many fail Pydantic validation? Build a retry mechanism for failures.

---

### 🏗️ Portfolio Project 02 — AI Document Intelligence

**What you'll build:** A document analysis SaaS (think: Docsend meets AI)

**Features:**
1. **Upload any document** (PDF, DOCX, image) — parse and extract text
2. **Smart summary** — structured summary with key points, action items, decisions
3. **Entity extraction** — people, organizations, dates, amounts mentioned
4. **Q&A mode** — ask questions, get answers with citations
5. **Data export** — download all extractions as structured JSON/CSV

**Tech:**
- FastAPI + PostgreSQL backend
- Next.js frontend with file upload
- OpenAI + Claude (use Claude for documents, GPT for structured extraction)
- `instructor` for structured outputs
- `pdfplumber` + `python-docx` for document parsing

**What makes it portfolio-worthy:**
- Production error handling (corrupt files, too-large files, unsupported formats)
- Cost tracking per document
- Comparison mode: compare 2 documents, show similarities/differences

**Push to:** `ai-engineering-journey/projects/03-doc-intelligence/`

---

## Module 07 — Embeddings & Semantic Search

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐☆☆

> *Embeddings are the foundation of RAG, semantic search, recommendation systems, and duplicate detection. Master this and you'll use it in every AI project you build.*

---

### 7.1 — What Embeddings Really Are

An embedding converts text into a vector — a list of numbers that represents the *meaning* of the text in a high-dimensional space.

```python
# text-embedding-3-small produces 1536-dimensional vectors
# That means each piece of text becomes a list of 1536 floats
# Similar meaning → similar direction in 1536-dimensional space

from openai import OpenAI
import numpy as np

client = OpenAI()

def embed(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
        dimensions=512  # You can reduce dimensions for cheaper storage
    )
    return response.data[0].embedding

# Cosine similarity: measures angle between vectors (0=unrelated, 1=identical)
def similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

# Try it:
e1 = embed("Machine learning algorithms for image classification")
e2 = embed("Deep learning techniques for recognizing pictures")
e3 = embed("How to make biryani")

print(f"ML vs DL: {similarity(e1, e2):.3f}")  # ~0.92 — very similar!
print(f"ML vs Biryani: {similarity(e1, e3):.3f}")  # ~0.2 — very different
```

---

### 7.2 — Embedding Models Comparison

| Model | Dimensions | Context | Best For |
|-------|-----------|---------|---------|
| `text-embedding-3-small` | 1536 | 8191 tokens | General purpose, cheap |
| `text-embedding-3-large` | 3072 | 8191 tokens | Higher accuracy |
| `text-embedding-ada-002` | 1536 | 8191 | Legacy (avoid for new projects) |
| `nomic-embed-text` (local) | 768 | 8192 | Free, good quality |
| `all-MiniLM-L6-v2` (local) | 384 | 256 | Ultra-fast, smaller |

**Cost consideration:**
- `text-embedding-3-small`: ~$0.00002 per 1K tokens (essentially free)
- Embed a 1M token dataset: $0.02

---

### 7.3 — Building Semantic Search from Scratch

Before using vector databases, understand what they're doing internally:

```python
import numpy as np
from openai import OpenAI
import pickle
from pathlib import Path

client = OpenAI()

class SimpleVectorStore:
    """
    A simple in-memory vector store — this is what Chroma/Pinecone do, but simpler.
    Shows you exactly what's happening under the hood.
    """
    
    def __init__(self):
        self.documents: list[str] = []
        self.embeddings: list[list[float]] = []
        self.metadata: list[dict] = []
    
    def add_documents(self, documents: list[str], metadata: list[dict] = None):
        """Embed and store documents"""
        print(f"Embedding {len(documents)} documents...")
        
        # Batch embedding — more efficient than one-by-one
        response = client.embeddings.create(
            model="text-embedding-3-small",
            input=documents
        )
        
        new_embeddings = [item.embedding for item in response.data]
        
        self.documents.extend(documents)
        self.embeddings.extend(new_embeddings)
        self.metadata.extend(metadata or [{} for _ in documents])
        
        print(f"Store now has {len(self.documents)} documents")
    
    def search(self, query: str, top_k: int = 5) -> list[dict]:
        """Find most similar documents to the query"""
        # Embed the query
        response = client.embeddings.create(
            model="text-embedding-3-small",
            input=query
        )
        query_embedding = np.array(response.data[0].embedding)
        
        # Calculate similarity with all stored documents
        similarities = []
        for i, doc_embedding in enumerate(self.embeddings):
            doc_vec = np.array(doc_embedding)
            sim = np.dot(query_embedding, doc_vec) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(doc_vec)
            )
            similarities.append((i, float(sim)))
        
        # Sort by similarity, take top_k
        similarities.sort(key=lambda x: x[1], reverse=True)
        
        results = []
        for idx, score in similarities[:top_k]:
            results.append({
                "document": self.documents[idx],
                "score": score,
                "metadata": self.metadata[idx]
            })
        
        return results
    
    def save(self, path: str):
        with open(path, 'wb') as f:
            pickle.dump({"docs": self.documents, "embeddings": self.embeddings, "meta": self.metadata}, f)
    
    def load(self, path: str):
        with open(path, 'rb') as f:
            data = pickle.load(f)
            self.documents = data["docs"]
            self.embeddings = data["embeddings"]
            self.metadata = data["meta"]

# Example usage
store = SimpleVectorStore()

# Index some documents
docs = [
    "Python is a high-level programming language known for readability",
    "JavaScript is the language of the web, running in browsers",
    "Machine learning uses data to train models to make predictions",
    "Neural networks are inspired by the human brain's structure",
    "FastAPI is a modern Python web framework for building APIs",
    "React is a JavaScript library for building user interfaces",
    "Embeddings are vector representations of text meaning",
    "The Indian Premier League is a cricket tournament held annually"
]

store.add_documents(docs)

# Search
results = store.search("How do I build a web API?", top_k=3)
for r in results:
    print(f"Score: {r['score']:.3f} | {r['document']}")
```

---

### 7.4 — Advanced Embedding Techniques

#### Sentence Window Embedding
Don't embed individual sentences — embed windows of sentences for more context:

```python
def create_sentence_windows(text: str, window_size: int = 3) -> list[dict]:
    """Create overlapping windows of sentences for better context"""
    sentences = text.split('. ')
    windows = []
    
    for i in range(len(sentences)):
        start = max(0, i - window_size // 2)
        end = min(len(sentences), i + window_size // 2 + 1)
        
        window_text = '. '.join(sentences[start:end])
        windows.append({
            "text": window_text,           # Full window for embedding
            "original": sentences[i],      # The actual sentence (for display)
            "sentence_idx": i
        })
    
    return windows
```

#### Embedding for Duplicate Detection
```python
def find_duplicates(texts: list[str], threshold: float = 0.95) -> list[tuple]:
    """Find near-duplicate texts using embedding similarity"""
    embeddings = []
    response = client.embeddings.create(model="text-embedding-3-small", input=texts)
    embeddings = [item.embedding for item in response.data]
    
    duplicates = []
    for i in range(len(texts)):
        for j in range(i + 1, len(texts)):
            sim = similarity(embeddings[i], embeddings[j])
            if sim > threshold:
                duplicates.append((i, j, sim, texts[i], texts[j]))
    
    return duplicates
```

---

### 🎯 Coding Challenges

**Challenge 1 — Job Search Engine:**
Embed 50 job descriptions. Build a semantic search that takes a candidate's skills/bio and finds the top 5 matching jobs. 

**Challenge 2 — Duplicate Detector:**
Take 200 customer support tickets. Find which ones are asking the same question (semantic duplicates). Group them together.

**Challenge 3 — Embedding Visualization:**
Embed 20 sentences across 5 topics (sports, cooking, programming, movies, travel). Use PCA or t-SNE to reduce to 2D and plot them. Verify that same-topic sentences cluster together.

**Challenge 4 — Hybrid Search:**
Implement a search that combines:
- Keyword search (BM25 — use `rank_bm25` library)
- Semantic search (embeddings)
- Merge results with a weighted score

---

### 🏗️ Mini-Project: Personal Knowledge Search Engine

**What you'll build:** A tool that indexes all your notes/docs/PDFs and makes them semantically searchable.

Features:
1. Watch a folder for files (PDF, MD, TXT)
2. Automatically embed new/changed files
3. CLI and web interface for search
4. Show top 5 results with highlighted matching section
5. "Ask a question" mode — uses the top results as context to answer

**Tech:** Python, OpenAI embeddings, your SimpleVectorStore (not a real vector DB yet — that's next module)

---

### 🧪 Phase 1 Revision Checkpoint

Before entering Phase 2, review:

**Concept Review Questions:**
1. What is the difference between the OpenAI API and Anthropic API for multi-turn conversation history?
2. When would you use Claude over GPT-4o? Give a real scenario.
3. What is the difference between JSON Mode and Structured Outputs (Strict mode)?
4. Why does function calling matter for agents?
5. What is cosine similarity and why use it (not Euclidean distance) for embeddings?

**Practical Review:**
- Can you build a streaming API endpoint from memory?
- Can you write a function call definition for a database query function?
- Can you explain why similar texts have similar embeddings?

---
---

# PHASE 2: RAG SYSTEMS

---

## Module 08 — Vector Databases Deep Dive

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐☆☆

> *Your SimpleVectorStore from Module 07 works for 1,000 docs. Vector databases work for 100 million docs with millisecond query time. Understanding why matters for choosing the right tool.*

---

### 8.1 — Why Vector Databases Exist

**The problem with naive search:**
- Linear search through 1M embeddings = ~1 second per query
- Scale to 10M documents = ~10 seconds
- This doesn't work for production

**The solution: Approximate Nearest Neighbor (ANN)**

Vector databases use index structures (HNSW, IVF, PQ) that find the *approximately* closest vectors in milliseconds by organizing the vector space intelligently.

**HNSW (Hierarchical Navigable Small World) — the most common:**
- Builds a multi-layer graph of vectors
- High layers = coarse search (fast, low accuracy)
- Low layers = fine search (slower, high accuracy)
- Query traverses from top layer down
- Result: 99%+ recall at < 10ms for millions of vectors

---

### 8.2 — ChromaDB (Start Here — Local & Easy)

Chroma is the best choice for development and smaller production systems:

```python
import chromadb
from openai import OpenAI

openai_client = OpenAI()

# Initialize Chroma (persistent storage)
chroma_client = chromadb.PersistentClient(path="./chroma_db")

# Create a collection (like a table)
collection = chroma_client.get_or_create_collection(
    name="company_docs",
    metadata={"hnsw:space": "cosine"}  # Use cosine similarity
)

def embed_texts(texts: list[str]) -> list[list[float]]:
    response = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    return [item.embedding for item in response.data]

# Add documents
def add_documents(texts: list[str], ids: list[str], metadatas: list[dict] = None):
    embeddings = embed_texts(texts)
    collection.add(
        documents=texts,
        embeddings=embeddings,
        ids=ids,
        metadatas=metadatas or [{} for _ in texts]
    )

# Query
def search(query: str, top_k: int = 5, filter_dict: dict = None) -> list[dict]:
    query_embedding = embed_texts([query])[0]
    
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k,
        where=filter_dict  # Metadata filtering: {"source": "manual"}
    )
    
    output = []
    for i in range(len(results['ids'][0])):
        output.append({
            "id": results['ids'][0][i],
            "document": results['documents'][0][i],
            "score": 1 - results['distances'][0][i],  # Convert distance to similarity
            "metadata": results['metadatas'][0][i]
        })
    
    return output

# Example: Index product documentation
add_documents(
    texts=[
        "Our return policy allows returns within 30 days of purchase",
        "Shipping takes 3-5 business days for standard delivery",
        "Premium members get free shipping on all orders",
    ],
    ids=["doc-001", "doc-002", "doc-003"],
    metadatas=[
        {"source": "returns_policy", "updated": "2024-01"},
        {"source": "shipping_info", "updated": "2024-01"},
        {"source": "premium_benefits", "updated": "2024-01"},
    ]
)

# Search with metadata filter
results = search("How long does shipping take?", filter_dict={"source": "shipping_info"})
```

---

### 8.3 — Pinecone (Production-Grade Cloud Vector DB)

For production systems with millions of vectors:

```python
from pinecone import Pinecone, ServerlessSpec

# Initialize
pc = Pinecone(api_key=os.getenv("PINECONE_API_KEY"))

# Create index
if "my-index" not in pc.list_indexes().names():
    pc.create_index(
        name="my-index",
        dimension=1536,  # Must match your embedding model!
        metric="cosine",
        spec=ServerlessSpec(cloud="aws", region="us-east-1")
    )

index = pc.Index("my-index")

# Upsert vectors (create or update)
def upsert_documents(texts: list[str], ids: list[str], metadatas: list[dict]):
    embeddings = embed_texts(texts)
    
    vectors = []
    for i, (text, embedding, metadata) in enumerate(zip(texts, embeddings, metadatas)):
        vectors.append({
            "id": ids[i],
            "values": embedding,
            "metadata": {**metadata, "text": text}  # Store text in metadata!
        })
    
    # Upsert in batches of 100 (Pinecone limit)
    batch_size = 100
    for i in range(0, len(vectors), batch_size):
        index.upsert(vectors=vectors[i:i+batch_size])

# Query with namespace (great for multi-tenant apps)
def search_pinecone(query: str, top_k: int = 5, namespace: str = "default") -> list[dict]:
    query_embedding = embed_texts([query])[0]
    
    results = index.query(
        vector=query_embedding,
        top_k=top_k,
        namespace=namespace,
        include_metadata=True
    )
    
    return [
        {
            "id": match.id,
            "score": match.score,
            "text": match.metadata.get("text", ""),
            "metadata": match.metadata
        }
        for match in results.matches
    ]
```

---

### 8.4 — pgvector — Vectors in PostgreSQL

If you already have a PostgreSQL database, add vector search without a new service:

```sql
-- Enable the extension
CREATE EXTENSION vector;

-- Create a table with vector column
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536),  -- Must match your model
    source VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create HNSW index for fast search
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Search: Find most similar documents
SELECT 
    id,
    content,
    source,
    1 - (embedding <=> '[0.1, 0.2, ...]'::vector) as similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

```python
import psycopg2
import numpy as np

def search_pgvector(query_embedding: list[float], top_k: int = 5) -> list[dict]:
    conn = psycopg2.connect(os.getenv("DATABASE_URL"))
    cursor = conn.cursor()
    
    # Convert to PostgreSQL vector format
    embedding_str = "[" + ",".join(map(str, query_embedding)) + "]"
    
    cursor.execute("""
        SELECT id, content, source,
               1 - (embedding <=> %s::vector) as similarity
        FROM documents
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (embedding_str, embedding_str, top_k))
    
    results = []
    for row in cursor.fetchall():
        results.append({
            "id": row[0],
            "content": row[1], 
            "source": row[2],
            "similarity": float(row[3])
        })
    
    conn.close()
    return results
```

**When to use what:**
| Use Case | Recommended |
|----------|-------------|
| Development / prototyping | ChromaDB (local) |
| Production, < 1M vectors, simple | ChromaDB (cloud) or pgvector |
| Production, > 1M vectors | Pinecone or Weaviate |
| Already have PostgreSQL | pgvector |
| Multi-tenant SaaS | Pinecone (namespaces) |

---

### 🎯 Coding Challenges

**Challenge 1 — Migration:**
Take your SimpleVectorStore from Module 07. Migrate all documents to ChromaDB. Verify search results are the same.

**Challenge 2 — Metadata Filtering:**
Index 100 product reviews with metadata: product_id, rating, date. Build a search that finds relevant reviews only for a specific product above a certain rating.

**Challenge 3 — pgvector Setup:**
Set up PostgreSQL locally with pgvector. Index the same documents. Compare query performance vs ChromaDB.

---
---

## Module 09 — Building RAG: Naive to Advanced

**Time:** ~2 weeks | **Difficulty:** ⭐⭐⭐⭐☆

> *RAG (Retrieval-Augmented Generation) is the most important pattern in production AI. Master it completely.*

---

### 9.1 — Naive RAG: The Foundation

The simplest possible RAG pipeline:

```
Document → Chunk → Embed → Store
Query → Embed → Search → Retrieve → Generate Answer
```

```python
from openai import OpenAI
import chromadb

client = OpenAI()
chroma = chromadb.PersistentClient(path="./rag_db")
collection = chroma.get_or_create_collection("knowledge_base", metadata={"hnsw:space": "cosine"})

def naive_rag(query: str, top_k: int = 3) -> str:
    """The simplest possible RAG pipeline"""
    
    # Step 1: Embed the query
    query_embedding = client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    ).data[0].embedding
    
    # Step 2: Retrieve relevant chunks
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k
    )
    
    retrieved_chunks = results['documents'][0]
    
    # Step 3: Build context from retrieved chunks
    context = "\n\n---\n\n".join(retrieved_chunks)
    
    # Step 4: Generate answer
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """You are a helpful assistant. Answer the question using ONLY the provided context.
If the context doesn't contain the answer, say "I don't have information about that."
Always cite which part of the context you used."""
            },
            {
                "role": "user",
                "content": f"""Context:
{context}

Question: {query}"""
            }
        ],
        temperature=0.0
    )
    
    return response.choices[0].message.content
```

---

### 9.2 — Advanced RAG Techniques

#### HyDE — Hypothetical Document Embeddings

The problem: "What's the refund policy?" and the actual policy text don't look similar when embedded, because one is a question and one is a statement.

The solution: Generate a hypothetical answer first, embed that, use it to search.

```python
def hyde_retrieval(query: str, top_k: int = 5) -> list[str]:
    """
    HyDE: Generate a hypothetical answer, use it to retrieve real documents
    Often dramatically improves retrieval quality
    """
    # Step 1: Generate hypothetical answer
    hypothetical = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "Generate a concise, factual-sounding answer to the question. Write as if you're answering from a company's knowledge base. Don't say 'I' — write in third person."
            },
            {"role": "user", "content": query}
        ],
        temperature=0.0,
        max_tokens=150
    ).choices[0].message.content
    
    print(f"[HyDE] Hypothetical answer: {hypothetical[:100]}...")
    
    # Step 2: Embed the hypothetical answer (not the query!)
    embedding = client.embeddings.create(
        model="text-embedding-3-small",
        input=hypothetical  # This is the key change
    ).data[0].embedding
    
    # Step 3: Search using the hypothetical answer's embedding
    results = collection.query(query_embeddings=[embedding], n_results=top_k)
    return results['documents'][0]
```

#### Contextual Retrieval (Anthropic's Technique)

When chunking, add context about where each chunk comes from within the document:

```python
def add_context_to_chunk(document: str, chunk: str, document_title: str) -> str:
    """
    Add surrounding context to each chunk before embedding.
    Anthropic found this improves retrieval by 49%.
    """
    context_prompt = f"""
<document_title>{document_title}</document_title>
<document_summary>
Please give a short context explaining what this chunk is about in relation to the larger document.
Keep it to 1-2 sentences.
</document_summary>

<document>
{document[:2000]}  # First 2000 chars for context
...
</document>

<chunk>
{chunk}
</chunk>

Provide context for this chunk in 1-2 sentences:
"""
    
    context = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": context_prompt}],
        max_tokens=100,
        temperature=0.0
    ).choices[0].message.content
    
    return f"{context}\n\n{chunk}"

# Usage: before storing each chunk, add context
enhanced_chunk = add_context_to_chunk(
    document=full_document_text,
    chunk="Returns must be initiated within 30 days.",
    document_title="Customer Return Policy 2024"
)
# enhanced_chunk: "This chunk describes the time limit for initiating returns in the company's return policy.\n\nReturns must be initiated within 30 days."
```

#### Multi-Query Retrieval

One query might miss relevant documents. Generate multiple queries and combine results:

```python
def multi_query_retrieval(query: str, top_k: int = 5) -> list[str]:
    """Generate multiple query variations and combine retrieved documents"""
    
    # Generate 3 query variations
    variations_response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """Generate 3 different versions of the given query that might retrieve different relevant documents.
Return ONLY the 3 queries, one per line, no numbering."""
            },
            {"role": "user", "content": query}
        ],
        temperature=0.7
    )
    
    variations = variations_response.choices[0].message.content.strip().split('\n')
    variations = [query] + [v for v in variations if v.strip()][:3]  # Original + 3 more
    
    # Retrieve for each variation
    all_docs = {}
    for variation in variations:
        embedding = client.embeddings.create(
            model="text-embedding-3-small",
            input=variation
        ).data[0].embedding
        
        results = collection.query(query_embeddings=[embedding], n_results=top_k)
        
        for doc, meta in zip(results['documents'][0], results['metadatas'][0]):
            doc_id = meta.get('id', doc[:50])  # Use ID or first 50 chars as key
            if doc_id not in all_docs:
                all_docs[doc_id] = {"doc": doc, "count": 0}
            all_docs[doc_id]["count"] += 1  # Count how many queries retrieved this
    
    # Return docs that appeared in multiple queries (reciprocal rank fusion idea)
    sorted_docs = sorted(all_docs.values(), key=lambda x: x["count"], reverse=True)
    return [item["doc"] for item in sorted_docs[:top_k]]
```

#### Reranking — Post-Retrieval Improvement

After vector search, re-rank results for precision:

```python
import cohere

co = cohere.Client(os.getenv("COHERE_API_KEY"))

def rerank_results(query: str, documents: list[str], top_n: int = 3) -> list[dict]:
    """
    Use Cohere's reranker to re-order retrieved documents.
    Rerankers use cross-encoders which are much more accurate than bi-encoders.
    """
    response = co.rerank(
        query=query,
        documents=documents,
        model="rerank-english-v3.0",
        top_n=top_n
    )
    
    return [
        {
            "document": documents[result.index],
            "relevance_score": result.relevance_score,
            "original_rank": result.index
        }
        for result in response.results
    ]

# Full advanced RAG pipeline
def advanced_rag(query: str) -> dict:
    # 1. Multi-query retrieval (wider net)
    raw_docs = multi_query_retrieval(query, top_k=10)
    
    # 2. Reranking (more precise)
    reranked = rerank_results(query, raw_docs, top_n=3)
    
    # 3. Build context with citations
    context_parts = []
    for i, item in enumerate(reranked, 1):
        context_parts.append(f"[Source {i}]: {item['document']}")
    
    context = "\n\n".join(context_parts)
    
    # 4. Generate answer with citations
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """Answer using ONLY the provided sources. 
Cite sources as [Source 1], [Source 2] etc. when you use information from them.
If the sources don't contain the answer, say so clearly."""
            },
            {"role": "user", "content": f"Sources:\n{context}\n\nQuestion: {query}"}
        ],
        temperature=0.0
    )
    
    return {
        "answer": response.choices[0].message.content,
        "sources": [item["document"] for item in reranked],
        "relevance_scores": [item["relevance_score"] for item in reranked]
    }
```

---

### 9.3 — RAG Failure Modes & How to Debug Them

**Failure 1: Retrieval Failure — Wrong chunks retrieved**
- Symptom: "I don't have information about that" when you know the document has it
- Cause: Query and document embeddings are semantically distant
- Fix: HyDE, multi-query, better chunking, contextual embeddings

**Failure 2: Lost in the Middle — Correct chunks retrieved but answer is wrong**
- Symptom: Model ignores relevant content in the middle of context
- Cause: The "lost in the middle" attention problem
- Fix: Put most relevant chunks first/last, not in the middle

**Failure 3: Context Overflow — Too much context confuses the model**
- Symptom: Rambling, unfocused answers despite good retrieval
- Cause: Too many chunks, model can't identify what's relevant
- Fix: Reranking to reduce to top 3-5, shorter chunks

**Failure 4: Hallucination Despite RAG**
- Symptom: Model answers confidently but ignores the provided context
- Cause: Model relies on parametric (training) knowledge
- Fix: Stronger system prompt ("ONLY use the provided context"), faithfulness eval

---

### 🏗️ Portfolio Project 03 — Intelligent Customer Support Bot

**What you'll build:** A production-ready customer support bot for a fictional e-commerce company.

**System Design:**
```
Company Docs (PDFs, FAQs, Policy Docs)
        ↓
   Document Processing
   (chunking, contextual embeddings)
        ↓
   ChromaDB + pgvector
        ↓
Advanced RAG Pipeline (HyDE + Reranking)
        ↓
   Answer Generation
   (with citations, confidence score)
        ↓
   User Interface
   (streaming chat, source display)
```

**Features:**
1. Index a company's knowledge base (provide sample docs: return policy, shipping, product catalog)
2. Multi-turn conversation with memory
3. Answer with cited sources
4. "I don't know" detection (when confidence is low)
5. Escalation trigger (when bot can't answer → human handoff simulation)
6. Admin dashboard: view queries, retrieved chunks, answer quality

**Evaluation (required):**
- Build a test set of 20 question-answer pairs
- Measure retrieval recall (was the right chunk in the top 5?)
- Measure answer faithfulness (did the model use the retrieved context?)
- Compare naive RAG vs your advanced RAG pipeline

**Tech:** FastAPI + Next.js + ChromaDB + Cohere Reranker

**Push to:** `ai-engineering-journey/projects/04-support-bot/`

---

## Module 10 — Chunking, Indexing & Retrieval Strategies

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐⭐☆**

> *How you chunk your documents has more impact on RAG quality than almost any other decision. Most tutorials ignore this entirely.*

---

### 10.1 — Chunking Strategies

**Fixed-size chunking (baseline):**
```python
def fixed_size_chunk(text: str, chunk_size: int = 512, overlap: int = 50) -> list[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap  # Overlap for continuity
    return chunks
```

**Recursive character chunking (LangChain's approach):**
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,       # Target chunk size in characters
    chunk_overlap=200,     # Overlap between chunks
    length_function=len,
    separators=["\n\n", "\n", ". ", " ", ""]  # Try to split on these in order
)

chunks = splitter.split_text(document_text)
```

**Semantic chunking (split on topic changes, not character count):**
```python
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

def semantic_chunk(sentences: list[str], threshold: float = 0.7) -> list[str]:
    """
    Split text when semantic similarity drops between consecutive sentences.
    Groups related sentences together regardless of character count.
    """
    if not sentences:
        return []
    
    # Embed all sentences
    embeddings = embed_texts(sentences)
    
    # Find split points where similarity drops
    chunks = []
    current_chunk = [sentences[0]]
    
    for i in range(1, len(sentences)):
        sim = cosine_similarity([embeddings[i-1]], [embeddings[i]])[0][0]
        
        if sim < threshold:  # Topic change detected!
            chunks.append(" ".join(current_chunk))
            current_chunk = [sentences[i]]
        else:
            current_chunk.append(sentences[i])
    
    chunks.append(" ".join(current_chunk))  # Don't forget last chunk
    return chunks
```

**When to use what:**

| Strategy | Best For | Trade-off |
|----------|----------|-----------|
| Fixed-size | Simple docs, quick setup | May split mid-sentence |
| Recursive character | General purpose | Needs tuning per doc type |
| Semantic | Complex technical docs | Slower (requires embedding) |
| Sentence window | Dense information | More API calls |
| Document structure (headers) | Well-structured docs | Requires parsing |

---

### 10.2 — Hybrid Search (Keyword + Semantic)

Pure semantic search misses exact keyword matches. Pure keyword search misses paraphrased content. Combine both:

```python
from rank_bm25 import BM25Okapi
import numpy as np

class HybridSearch:
    def __init__(self, documents: list[str]):
        self.documents = documents
        self.embeddings = embed_texts(documents)
        
        # BM25 for keyword search
        tokenized = [doc.lower().split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized)
    
    def search(self, query: str, top_k: int = 5, alpha: float = 0.5) -> list[dict]:
        """
        alpha = weight for semantic search (0.0 = pure BM25, 1.0 = pure semantic)
        """
        # Semantic search scores
        query_embedding = embed_texts([query])[0]
        semantic_scores = []
        for doc_embedding in self.embeddings:
            sim = np.dot(query_embedding, doc_embedding) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(doc_embedding)
            )
            semantic_scores.append(float(sim))
        
        # BM25 scores
        bm25_scores = self.bm25.get_scores(query.lower().split())
        
        # Normalize scores to [0, 1]
        def normalize(scores):
            min_s, max_s = min(scores), max(scores)
            if max_s == min_s:
                return [0.5] * len(scores)
            return [(s - min_s) / (max_s - min_s) for s in scores]
        
        sem_norm = normalize(semantic_scores)
        bm25_norm = normalize(bm25_scores.tolist())
        
        # Combine with weighted sum
        combined = [
            alpha * s + (1 - alpha) * b 
            for s, b in zip(sem_norm, bm25_norm)
        ]
        
        # Return top_k
        ranked = sorted(enumerate(combined), key=lambda x: x[1], reverse=True)
        
        return [
            {
                "document": self.documents[idx],
                "score": score,
                "semantic_score": sem_norm[idx],
                "bm25_score": bm25_norm[idx]
            }
            for idx, score in ranked[:top_k]
        ]
```

---

### 📚 Resources for Phase 2

**Watch:**
- 🎥 [DeepLearning.AI — Building and Evaluating Advanced RAG](https://learn.deeplearning.ai/courses/building-evaluating-advanced-rag)
- 🎥 [Jerry Liu (LlamaIndex) — Advanced RAG Techniques](https://www.youtube.com/watch?v=TRjq7t2Ms5I)
- 🎥 [AI Jason — RAG from Scratch Series](https://www.youtube.com/playlist?list=PLfaIDFEXuae2LXbO1_PKyVJiQ23ZztA0x)

**Read:**
- 📄 [Anthropic — Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)
- 📄 [Pinecone — RAG Guide](https://www.pinecone.io/learn/retrieval-augmented-generation/)
- 📄 [LangChain — RAG from Scratch](https://github.com/langchain-ai/rag-from-scratch)

**GitHub:**
- ⭐ [RAG Techniques by NirDiamant](https://github.com/NirDiamant/RAG_Techniques)
- ⭐ [LangChain RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/)

---

## 🔁 Phase 2 Revision Checkpoint

Before entering Phase 3 (Agents), review:

**Build a mini demo:**
Implement a complete RAG pipeline in under 50 lines of code from memory. No looking up documentation. If you can't — you need more practice.

**Explain to yourself:**
1. What is the difference between naive RAG and advanced RAG? Name 3 specific improvements.
2. What is HyDE and when would you use it?
3. What is reranking and why can't the vector search already do this?
4. What is hybrid search combining?
5. What chunking strategy would you use for a legal contract? Why?

---

*[End of Part 2 — Modules 05–10]*
*Next: Module 11 — Multimodal RAG | Module 12–15 — AI Agents | Module 16–19 — Production AI*
---

## Module 11 — Multimodal RAG

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐⭐☆

> *The real world isn't just text. PDFs have tables, charts, images. Slack has screenshots. Customer service has photos. Multimodal RAG handles all of it.*

---

### 11.1 — The Multimodal Challenge

Standard RAG only handles text. But real documents contain:
- **Tables** — financial reports, comparison tables, data sheets
- **Charts/Graphs** — visualizations that don't translate well to text
- **Images** — diagrams, screenshots, product photos
- **Mixed layouts** — text wrapped around figures

**The three approaches:**

| Approach | How | Best For |
|----------|-----|---------|
| **Extract text from images** (OCR + captioning) | GPT-4V describes images, stored as text | Most use cases |
| **Embed images directly** | Multi-modal embedding models (CLIP) | Image similarity search |
| **ColPali** | Modern approach — embeds document pages as images | Complex PDF layouts |

---

### 11.2 — Document Intelligence Pipeline

```python
import fitz  # PyMuPDF - best PDF parser
import base64
from pathlib import Path
from openai import OpenAI

client = OpenAI()

def process_pdf_multimodal(pdf_path: str) -> list[dict]:
    """
    Extract text AND analyze images from a PDF.
    Returns list of chunks with both text and image context.
    """
    doc = fitz.open(pdf_path)
    all_chunks = []
    
    for page_num in range(len(doc)):
        page = doc[page_num]
        
        # Extract text from page
        text = page.get_text()
        
        # Extract images from page
        image_list = page.get_images()
        image_descriptions = []
        
        for img_index, img in enumerate(image_list):
            xref = img[0]
            pix = fitz.Pixmap(doc, xref)
            
            if pix.n < 5:  # GRAY or RGB (not CMYK)
                img_data = pix.tobytes("jpeg")
                img_base64 = base64.b64encode(img_data).decode("utf-8")
                
                # Use GPT-4V to describe the image
                description = describe_image(img_base64)
                image_descriptions.append(f"[Image {img_index+1}: {description}]")
        
        # Combine text with image descriptions
        page_content = text
        if image_descriptions:
            page_content += "\n\n" + "\n".join(image_descriptions)
        
        # Chunk the page content
        if page_content.strip():
            all_chunks.append({
                "content": page_content,
                "metadata": {
                    "page": page_num + 1,
                    "source": pdf_path,
                    "has_images": len(image_list) > 0
                }
            })
    
    return all_chunks

def describe_image(image_base64: str) -> str:
    """Use GPT-4o to describe an image in context of a document"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": {"url": f"data:image/jpeg;base64,{image_base64}"}
                },
                {
                    "type": "text",
                    "text": """Describe this image from a document in 1-3 sentences. Focus on:
- What type of content it shows (chart, diagram, photo, table, etc.)
- The key information or data it conveys
- Any important numbers, labels, or trends visible

Be concise and factual. This description will be used for search and retrieval."""
                }
            ]
        }],
        max_tokens=200
    )
    return response.choices[0].message.content
```

---

### 🎯 Module 11 Challenge

**Build:** Process a real annual report PDF (find any public company's annual report).
- Extract all text + image descriptions
- Index everything in ChromaDB
- Build a Q&A interface
- Test: "What was the revenue growth?" and "Describe the main chart on page 5"

---
---

# PHASE 3: AI AGENTS

---

## Module 12 — Agent Foundations: Tools, Memory, Planning

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐⭐☆

> *Agents are the most exciting part of AI engineering — and the most dangerous to get wrong in production. This module teaches you to build them right.*

---

### 12.1 — What Is an Agent?

An agent is an LLM that can:
1. **Reason** — think about what needs to be done
2. **Use tools** — call functions to interact with the world
3. **Remember** — maintain state across multiple steps
4. **Plan** — break complex tasks into subtasks
5. **Loop** — keep going until the task is complete

**The ReAct Pattern (Reason + Act):**
```
Thought: What do I need to do to answer this?
Action: [call a tool]
Observation: [tool result]
Thought: What does this tell me? What do I do next?
Action: [call another tool if needed, or give final answer]
```

---

### 12.2 — The Agent Loop: Deep Implementation

```python
from openai import OpenAI
import json
from typing import Callable, Any
from dataclasses import dataclass

client = OpenAI()

@dataclass
class Tool:
    name: str
    description: str
    parameters: dict
    function: Callable

class Agent:
    def __init__(self, system_prompt: str, tools: list[Tool], model: str = "gpt-4o-mini"):
        self.system_prompt = system_prompt
        self.tools = {t.name: t for t in tools}
        self.model = model
        self.messages = [{"role": "system", "content": system_prompt}]
        self.max_iterations = 10  # Safety limit!
    
    def _tools_schema(self) -> list[dict]:
        """Convert tools to OpenAI function schema format"""
        return [
            {
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.parameters
                }
            }
            for tool in self.tools.values()
        ]
    
    def run(self, user_message: str) -> str:
        """Execute the agent loop"""
        self.messages.append({"role": "user", "content": user_message})
        
        iteration = 0
        while iteration < self.max_iterations:
            iteration += 1
            print(f"\n[Agent Iteration {iteration}]")
            
            # Ask the model what to do
            response = client.chat.completions.create(
                model=self.model,
                messages=self.messages,
                tools=self._tools_schema(),
                tool_choice="auto"
            )
            
            message = response.choices[0].message
            self.messages.append(message)
            
            # Check if agent wants to call tools
            if not message.tool_calls:
                # No tool calls = agent is done
                print(f"[Agent] Final answer reached after {iteration} iterations")
                return message.content
            
            # Execute each tool call
            for tool_call in message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                
                print(f"[Tool] Calling: {tool_name}({tool_args})")
                
                if tool_name in self.tools:
                    try:
                        result = self.tools[tool_name].function(**tool_args)
                        result_str = json.dumps(result) if not isinstance(result, str) else result
                    except Exception as e:
                        result_str = f"Error executing {tool_name}: {str(e)}"
                else:
                    result_str = f"Unknown tool: {tool_name}"
                
                print(f"[Tool] Result: {result_str[:100]}...")
                
                # Add tool result to messages
                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result_str
                })
        
        return "Max iterations reached. Please try a simpler query."

# Define real tools
import httpx
import datetime

def search_web(query: str) -> str:
    """Real web search using DuckDuckGo"""
    # Use DuckDuckGo API (no key required)
    r = httpx.get(f"https://api.duckduckgo.com/?q={query}&format=json&no_html=1")
    data = r.json()
    abstract = data.get("AbstractText", "")
    related = [r.get("Text", "") for r in data.get("RelatedTopics", [])[:3] if "Text" in r]
    return f"Abstract: {abstract}\nRelated: {', '.join(related)}"

def get_current_time() -> str:
    """Get current date and time"""
    return datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S IST")

def calculate(expression: str) -> str:
    """Safely evaluate a math expression"""
    try:
        result = eval(expression, {"__builtins__": {}}, {"abs": abs, "round": round, "min": min, "max": max})
        return str(result)
    except Exception as e:
        return f"Error: {e}"

# Create and run the agent
research_agent = Agent(
    system_prompt="""You are a research assistant that helps answer complex questions.
You have access to web search, time, and calculation tools.
Always verify your answers and cite your sources.
When you have enough information, give a clear, direct answer.""",
    tools=[
        Tool("search_web", "Search the web for current information", 
             {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]},
             search_web),
        Tool("get_current_time", "Get the current date and time",
             {"type": "object", "properties": {}},
             get_current_time),
        Tool("calculate", "Calculate a mathematical expression",
             {"type": "object", "properties": {"expression": {"type": "string"}}, "required": ["expression"]},
             calculate),
    ]
)

answer = research_agent.run("What is today's date, and how many days until 2026?")
print(f"\nFinal Answer:\n{answer}")
```

---

### 12.3 — Memory Systems for Agents

Agents need memory. Four types:

```python
from collections import deque
import json
from datetime import datetime

class AgentMemory:
    def __init__(self, max_short_term: int = 20):
        # 1. Working memory — current conversation (limited size)
        self.working_memory: deque = deque(maxlen=max_short_term)
        
        # 2. Episodic memory — past conversations (searchable)
        self.episodic_memory: list[dict] = []
        
        # 3. Semantic memory — facts learned (always available)
        self.semantic_memory: dict = {}
        
        # 4. Procedural memory — how to do things (in system prompt)
    
    def add_to_working(self, role: str, content: str):
        self.working_memory.append({"role": role, "content": content, "time": datetime.now().isoformat()})
    
    def save_episode(self, summary: str, key_facts: list[str]):
        """Save current conversation as an episode"""
        self.episodic_memory.append({
            "time": datetime.now().isoformat(),
            "summary": summary,
            "key_facts": key_facts
        })
    
    def remember_fact(self, key: str, value: str):
        """Store a semantic fact"""
        self.semantic_memory[key] = {"value": value, "learned_at": datetime.now().isoformat()}
    
    def get_relevant_memories(self, query: str) -> str:
        """Build a memory context string for the agent"""
        memory_parts = []
        
        # Include semantic facts
        if self.semantic_memory:
            facts = [f"- {k}: {v['value']}" for k, v in self.semantic_memory.items()]
            memory_parts.append("Known facts:\n" + "\n".join(facts))
        
        # Include recent episodes (simplified — in production use embedding search)
        if self.episodic_memory:
            recent = self.episodic_memory[-3:]  # Last 3 episodes
            episodes = [f"- {e['summary']}" for e in recent]
            memory_parts.append("Recent interactions:\n" + "\n".join(episodes))
        
        return "\n\n".join(memory_parts)
    
    def get_working_messages(self) -> list[dict]:
        return [{"role": m["role"], "content": m["content"]} for m in self.working_memory]
```

---

### 📚 Agent Resources

**Watch:**
- 🎥 [Harrison Chase (LangChain) — Agents Deep Dive](https://www.youtube.com/watch?v=DWUdGFCU2Rc)
- 🎥 [DeepLearning.AI — AI Agents in LangGraph](https://learn.deeplearning.ai/courses/ai-agents-in-langgraph)
- 🎥 [Andrej Karpathy — Software 2.0 and Agents](https://www.youtube.com/watch?v=y57wwucbXR8)

**Read:**
- 📄 [Lilian Weng — LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/)
- 📄 [ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629)

---

## Module 13 — Building Agents with LangChain & LangGraph

**Time:** ~2 weeks | **Difficulty:** ⭐⭐⭐⭐☆

> *LangGraph is the production framework for agents. It gives you stateful, controllable, observable agents — the difference between a demo and a product.*

---

### 13.1 — Why LangGraph Over Simple Agent Loops?

Your hand-built agent from Module 12 has problems:
- No state persistence between runs
- No way to pause and wait for human input
- No conditional logic in the flow
- No easy way to run steps in parallel
- Hard to debug what happened

LangGraph solves all of this with a **graph-based state machine**:
- Each node = a step (LLM call, tool use, etc.)
- Each edge = a transition (conditional or unconditional)
- State flows through the graph
- You can checkpoint, pause, resume, and observe

---

### 13.2 — LangGraph Fundamentals

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
import operator

# 1. Define State — the information flowing through your graph
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]  # Messages accumulate
    current_plan: str
    tools_used: Annotated[list, operator.add]
    final_answer: str
    iteration_count: int

# 2. Define Nodes — the steps in your workflow
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def plan_node(state: AgentState) -> AgentState:
    """Create a plan for answering the query"""
    messages = state["messages"]
    
    planning_prompt = SystemMessage(content="""Given the user's question, create a brief plan:
1. What information do I need?
2. Which tools will I use?
3. What will the final answer look like?

Be concise.""")
    
    response = llm.invoke([planning_prompt] + messages)
    
    return {
        "current_plan": response.content,
        "messages": [AIMessage(content=f"[Plan]: {response.content}")],
        "iteration_count": state.get("iteration_count", 0) + 1
    }

def execute_node(state: AgentState) -> AgentState:
    """Execute the plan using tools"""
    # ... tool execution logic
    return {"messages": [AIMessage(content="Executed tools")], "tools_used": ["web_search"]}

def synthesize_node(state: AgentState) -> AgentState:
    """Synthesize the final answer"""
    all_messages = state["messages"]
    
    response = llm.invoke(all_messages + [
        HumanMessage(content="Based on everything gathered, provide the final comprehensive answer.")
    ])
    
    return {"final_answer": response.content}

def should_continue(state: AgentState) -> str:
    """Conditional edge — decide what to do next"""
    if state.get("final_answer"):
        return "end"
    elif state.get("iteration_count", 0) >= 3:
        return "synthesize"  # Force synthesis after 3 iterations
    else:
        return "execute"

# 3. Build the Graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("plan", plan_node)
workflow.add_node("execute", execute_node)
workflow.add_node("synthesize", synthesize_node)

# Add edges
workflow.set_entry_point("plan")
workflow.add_conditional_edges("plan", should_continue)
workflow.add_conditional_edges("execute", should_continue)
workflow.add_edge("synthesize", END)

# 4. Compile with memory (enables persistence/checkpointing)
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)

# 5. Run
config = {"configurable": {"thread_id": "user-123"}}  # Each user gets a thread

result = app.invoke(
    {
        "messages": [HumanMessage(content="What are the top 3 AI companies by valuation in 2025?")],
        "current_plan": "",
        "tools_used": [],
        "final_answer": "",
        "iteration_count": 0
    },
    config=config
)

print(result["final_answer"])
```

---

### 🏗️ Portfolio Project 04 — AI Research Agent

**What you'll build:** A deep research agent that can research any topic and produce a comprehensive report.

**How it works:**
1. User provides a topic: "Impact of AI on Indian fintech sector"
2. Agent plans: "I need to search for: recent AI fintech news, Indian fintech landscape, regulatory environment, specific companies"
3. Agent searches web for each subtopic in parallel
4. Agent synthesizes into a structured report
5. User can ask follow-up questions (memory persists)
6. Export report as PDF

**Architecture:**
```
User Input → Planning Node → [Parallel Search Nodes] → Synthesis Node → Report Generation
                                    ↑
                              Human Review Node (optional approval before synthesis)
```

**Production features:**
- LangGraph for stateful workflow
- Langfuse for tracing every step
- Export to PDF/Markdown
- "Interrupt for human review" pattern
- Citation management

**Tech:** LangGraph + LangSmith/Langfuse + FastAPI + Next.js

**Push to:** `ai-engineering-journey/projects/05-research-agent/`

---

## Module 14 — Multi-Agent Systems

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐⭐⭐

> *One agent is powerful. Multiple specialized agents working together can tackle problems too complex for any single agent.*

---

### 14.1 — Why Multi-Agent?

A single agent trying to write code, test it, debug it, and write documentation is trying to do too many things at once. Each task requires a different "mindset."

Multi-agent systems have:
- **Specialization** — each agent is good at one thing
- **Parallelism** — agents can work simultaneously
- **Checks** — agents can review each other's work
- **Scale** — can handle much longer, more complex tasks

---

### 14.2 — Supervisor Pattern

```python
from langgraph.graph import StateGraph
from langchain_openai import ChatOpenAI
import operator
from typing import TypedDict, Annotated

llm = ChatOpenAI(model="gpt-4o")

# Worker agents
def researcher_agent(task: str) -> str:
    """Specialized in gathering information"""
    # ... implementation

def coder_agent(task: str) -> str:
    """Specialized in writing code"""
    # ... implementation

def reviewer_agent(code: str) -> str:
    """Specialized in reviewing and critiquing code"""
    # ... implementation

def writer_agent(task: str, content: str) -> str:
    """Specialized in writing clear documentation"""
    # ... implementation

# Supervisor decides who does what
class SupervisorState(TypedDict):
    task: str
    team_output: Annotated[list, operator.add]
    next_agent: str
    final_result: str

AGENTS = ["researcher", "coder", "reviewer", "writer", "FINISH"]

def supervisor(state: SupervisorState) -> SupervisorState:
    """Supervisor LLM that orchestrates the team"""
    from langchain_core.messages import SystemMessage, HumanMessage
    
    context = "\n".join(state.get("team_output", []))
    
    response = llm.invoke([
        SystemMessage(content=f"""You are a supervisor managing a software development team.
Available agents: {', '.join(AGENTS)}

Current task: {state['task']}

Work done so far:
{context if context else "Nothing yet."}

Who should work next? Choose FINISH when the task is complete.
Respond with ONLY the agent name."""),
        HumanMessage(content="Who should work next?")
    ])
    
    return {"next_agent": response.content.strip()}

def route_to_agent(state: SupervisorState) -> str:
    return state["next_agent"]

# Build supervisor graph
workflow = StateGraph(SupervisorState)
workflow.add_node("supervisor", supervisor)
workflow.add_node("researcher", lambda s: {"team_output": [researcher_agent(s["task"])]})
workflow.add_node("coder", lambda s: {"team_output": [coder_agent(s["task"])]})
workflow.add_node("reviewer", lambda s: {"team_output": [reviewer_agent(s["team_output"][-1])]})
workflow.add_node("writer", lambda s: {"team_output": [writer_agent(s["task"], "\n".join(s["team_output"]))]})

workflow.set_entry_point("supervisor")
workflow.add_conditional_edges("supervisor", route_to_agent, {
    "researcher": "researcher",
    "coder": "coder", 
    "reviewer": "reviewer",
    "writer": "writer",
    "FINISH": "__end__"
})

# All agents route back to supervisor
for agent in ["researcher", "coder", "reviewer", "writer"]:
    workflow.add_edge(agent, "supervisor")

app = workflow.compile()
```

---

### 📚 Multi-Agent Resources

**Watch:**
- 🎥 [DeepLearning.AI — Multi AI Agent Systems with crewAI](https://learn.deeplearning.ai/courses/multi-ai-agent-systems-with-crewai)
- 🎥 [LangGraph Multi-Agent Collaboration](https://www.youtube.com/watch?v=hvAPnpSfSGo)

**Read:**
- 📄 [LangGraph Multi-Agent Tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/)
- 📄 [CrewAI Documentation](https://docs.crewai.com)

---

## Module 15 — MCP: Model Context Protocol

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐⭐☆

> *MCP is the USB standard for AI tools. It lets AI models connect to any data source or service through a standard protocol. This is rapidly becoming the industry standard for agent tooling.*

---

### 15.1 — What Is MCP?

MCP (Model Context Protocol) standardizes how AI models connect to external systems. Instead of writing custom tool code for every agent, you write an MCP server once and any MCP-compatible AI can use it.

**Without MCP:**
```python
# Custom tool for every agent
def query_database(query: str) -> str:
    conn = get_db_connection()
    return conn.execute(query).fetchall()

# Repeat for every project, every agent
```

**With MCP:**
```python
# MCP server: write once, use from any AI
# Claude Desktop, Cursor, your custom agent — all can use the same server
```

---

### 15.2 — Building an MCP Server

```python
# Install: pip install mcp
from mcp.server import Server
from mcp.server.stdio import stdio_server
import mcp.types as types
import json
import sqlite3

# Create the MCP server
server = Server("my-database-server")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    """Tell clients what tools this server offers"""
    return [
        types.Tool(
            name="query_products",
            description="Query the products database",
            inputSchema={
                "type": "object",
                "properties": {
                    "sql": {
                        "type": "string",
                        "description": "SQL query to execute (SELECT only)"
                    }
                },
                "required": ["sql"]
            }
        ),
        types.Tool(
            name="get_product",
            description="Get details of a specific product by ID",
            inputSchema={
                "type": "object",
                "properties": {
                    "product_id": {"type": "integer"}
                },
                "required": ["product_id"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    """Handle tool calls from AI clients"""
    
    if name == "query_products":
        sql = arguments["sql"]
        
        # Safety check — only allow SELECT
        if not sql.strip().upper().startswith("SELECT"):
            return [types.TextContent(type="text", text="Error: Only SELECT queries allowed")]
        
        conn = sqlite3.connect("products.db")
        cursor = conn.cursor()
        
        try:
            cursor.execute(sql)
            rows = cursor.fetchall()
            columns = [desc[0] for desc in cursor.description]
            result = [dict(zip(columns, row)) for row in rows]
            return [types.TextContent(type="text", text=json.dumps(result, indent=2))]
        except Exception as e:
            return [types.TextContent(type="text", text=f"Error: {str(e)}")]
        finally:
            conn.close()
    
    elif name == "get_product":
        product_id = arguments["product_id"]
        conn = sqlite3.connect("products.db")
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM products WHERE id = ?", (product_id,))
        row = cursor.fetchone()
        conn.close()
        
        if row:
            columns = [desc[0] for desc in cursor.description]
            return [types.TextContent(type="text", text=json.dumps(dict(zip(columns, row)), indent=2))]
        else:
            return [types.TextContent(type="text", text="Product not found")]

# Run the server
if __name__ == "__main__":
    import asyncio
    asyncio.run(stdio_server(server))
```

**MCP resources config (`~/.config/claude/claude_desktop_config.json`):**
```json
{
  "mcpServers": {
    "my-database": {
      "command": "python",
      "args": ["/path/to/my_mcp_server.py"]
    }
  }
}
```

---

### 🏗️ Mini-Project: MCP Server for Your Portfolio

Build an MCP server that exposes:
1. Your GitHub repositories (list, get README, get file)
2. Your personal knowledge base (search your notes)
3. Your calendar availability

Once built, Claude Desktop and any MCP-compatible agent can use these tools.

---
---

# PHASE 4: PRODUCTION AI

---

## Module 16 — LLM Evaluation: Evals as Unit Tests

**Time:** ~2 weeks | **Difficulty:** ⭐⭐⭐⭐☆

> *"Evals are the new unit tests" — this is the most important engineering discipline in AI that most tutorials skip entirely. After this module, you will never ship an AI feature without evals.*

---

### 16.1 — Why Evals Matter

Without evals:
- You changed the system prompt — is it better or worse? You don't know.
- You upgraded the model — did quality improve? You don't know.
- Your RAG pipeline changed — is retrieval better? You don't know.
- A customer complains — can you reproduce the failure? Maybe not.

With evals:
- Every code change is automatically tested against a golden dataset
- Model upgrades are objectively compared
- You can confidently say "this version is 23% better at X"

---

### 16.2 — Building an Eval Framework

```python
from dataclasses import dataclass, field
from typing import Callable
from openai import OpenAI
import json

client = OpenAI()

@dataclass
class EvalCase:
    """A single test case"""
    input: str                    # The query to test
    expected_output: str = ""     # Optional: expected answer
    expected_contains: list[str] = field(default_factory=list)  # Must contain these
    expected_not_contains: list[str] = field(default_factory=list)  # Must NOT contain
    metadata: dict = field(default_factory=dict)

@dataclass
class EvalResult:
    case: EvalCase
    actual_output: str
    scores: dict[str, float]      # Multiple metrics
    passed: bool
    reasoning: str = ""

class LLMEvaluator:
    """Use an LLM to judge the quality of another LLM's output"""
    
    def __init__(self, judge_model: str = "gpt-4o"):
        self.client = OpenAI()
        self.judge_model = judge_model
    
    def evaluate_faithfulness(self, context: str, answer: str) -> dict:
        """Is the answer faithful to the provided context? (Crucial for RAG)"""
        response = self.client.chat.completions.create(
            model=self.judge_model,
            messages=[{
                "role": "user",
                "content": f"""Evaluate if this answer is faithful to the provided context.
An answer is faithful if it only makes claims supported by the context.

Context:
{context}

Answer:
{answer}

Score from 0.0 to 1.0 where:
1.0 = Completely faithful, every claim is supported by context
0.5 = Mostly faithful but has some unsupported claims
0.0 = Makes claims not in the context (hallucination)

Respond as JSON: {{"score": float, "reasoning": "brief explanation", "unsupported_claims": ["list"]}}"""
            }],
            response_format={"type": "json_object"},
            temperature=0.0
        )
        return json.loads(response.choices[0].message.content)
    
    def evaluate_relevance(self, question: str, answer: str) -> dict:
        """Does the answer actually address the question?"""
        response = self.client.chat.completions.create(
            model=self.judge_model,
            messages=[{
                "role": "user",
                "content": f"""Is this answer relevant to the question?

Question: {question}
Answer: {answer}

Score 0.0 to 1.0. JSON: {{"score": float, "reasoning": "explanation"}}"""
            }],
            response_format={"type": "json_object"},
            temperature=0.0
        )
        return json.loads(response.choices[0].message.content)
    
    def evaluate_completeness(self, question: str, answer: str, expected: str = "") -> dict:
        """Does the answer fully address all aspects of the question?"""
        comparison = f"\nExpected answer: {expected}" if expected else ""
        
        response = self.client.chat.completions.create(
            model=self.judge_model,
            messages=[{
                "role": "user",
                "content": f"""How complete is this answer to the question?

Question: {question}
Answer: {answer}{comparison}

Score 0.0 to 1.0. JSON: {{"score": float, "reasoning": "explanation", "missing_aspects": ["list"]}}"""
            }],
            response_format={"type": "json_object"},
            temperature=0.0
        )
        return json.loads(response.choices[0].message.content)

class EvalSuite:
    def __init__(self, name: str, system_under_test: Callable[[str], str]):
        self.name = name
        self.sut = system_under_test  # The function being tested
        self.evaluator = LLMEvaluator()
        self.cases: list[EvalCase] = []
        self.results: list[EvalResult] = []
    
    def add_case(self, case: EvalCase):
        self.cases.append(case)
    
    def run(self, verbose: bool = True) -> dict:
        """Run all test cases and return aggregate metrics"""
        self.results = []
        
        for i, case in enumerate(self.cases):
            if verbose:
                print(f"[{i+1}/{len(self.cases)}] Testing: {case.input[:60]}...")
            
            # Run the system under test
            actual = self.sut(case.input)
            
            # Run evaluations
            scores = {}
            
            # Rule-based checks
            contains_score = 1.0
            for expected in case.expected_contains:
                if expected.lower() not in actual.lower():
                    contains_score = 0.0
                    break
            scores["contains_expected"] = contains_score
            
            not_contains_score = 1.0
            for bad in case.expected_not_contains:
                if bad.lower() in actual.lower():
                    not_contains_score = 0.0
                    break
            scores["not_contains_bad"] = not_contains_score
            
            # LLM-based evaluation
            relevance = self.evaluator.evaluate_relevance(case.input, actual)
            scores["relevance"] = relevance["score"]
            
            # Overall pass/fail
            passed = all(s >= 0.7 for s in scores.values())
            
            result = EvalResult(
                case=case,
                actual_output=actual,
                scores=scores,
                passed=passed
            )
            self.results.append(result)
            
            if verbose:
                status = "✅ PASS" if passed else "❌ FAIL"
                avg_score = sum(scores.values()) / len(scores)
                print(f"  {status} | Avg Score: {avg_score:.2f} | {scores}")
        
        # Aggregate metrics
        pass_rate = sum(1 for r in self.results if r.passed) / len(self.results)
        avg_scores = {}
        for metric in self.results[0].scores.keys():
            avg_scores[metric] = sum(r.scores[metric] for r in self.results) / len(self.results)
        
        summary = {
            "suite_name": self.name,
            "total_cases": len(self.results),
            "pass_rate": pass_rate,
            "passed": int(pass_rate * len(self.results)),
            "failed": int((1 - pass_rate) * len(self.results)),
            "avg_scores": avg_scores
        }
        
        print(f"\n{'='*60}")
        print(f"Eval Suite: {self.name}")
        print(f"Pass Rate: {pass_rate:.1%} ({summary['passed']}/{summary['total_cases']})")
        print(f"Scores: {avg_scores}")
        
        return summary

# Usage
def my_chatbot(question: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "You are a customer support agent for a tech company."},
            {"role": "user", "content": question}
        ]
    )
    return response.choices[0].message.content

# Create eval suite
suite = EvalSuite("Customer Support Bot", my_chatbot)

# Add test cases
suite.add_case(EvalCase(
    input="What is your return policy?",
    expected_contains=["return", "day"],
    expected_not_contains=["I don't know", "I cannot", "sorry I can't"],
))

suite.add_case(EvalCase(
    input="My order hasn't arrived after 2 weeks",
    expected_contains=["apologize", "investigate", "track"],
))

# Run evals
results = suite.run()
```

---

### 16.3 — RAGAS — RAG-Specific Evaluation

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall, context_precision
from datasets import Dataset

# Prepare your RAG test cases
test_data = {
    "question": [
        "What is the return policy?",
        "How long does shipping take?"
    ],
    "answer": [
        "Returns are accepted within 30 days.",  # Your RAG's answer
        "Standard shipping takes 3-5 business days."
    ],
    "contexts": [
        ["Our return policy allows returns within 30 days of purchase with receipt."],
        ["We offer standard shipping (3-5 days) and express shipping (1-2 days)."]
    ],
    "ground_truth": [
        "Returns are accepted within 30 days of purchase.",
        "Standard shipping takes 3-5 business days."
    ]
}

dataset = Dataset.from_dict(test_data)

# Run RAGAS evaluation
results = evaluate(
    dataset,
    metrics=[
        faithfulness,         # Does answer stick to context?
        answer_relevancy,     # Does answer address the question?
        context_recall,       # Did retrieval find the right documents?
        context_precision,    # Are retrieved docs actually relevant?
    ]
)

print(results)
```

---

### 16.4 — CI/CD for AI — Evals in Your Pipeline

```yaml
# .github/workflows/ai-evals.yml
name: AI Quality Gates

on:
  pull_request:
    paths: ['src/**', 'prompts/**']  # Run on any AI-related changes

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run AI Evals
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: python run_evals.py --threshold 0.85
      
      - name: Upload eval results
        uses: actions/upload-artifact@v4
        with:
          name: eval-results
          path: eval_results.json
```

```python
# run_evals.py
import sys
import json

def main():
    threshold = float(sys.argv[sys.argv.index("--threshold") + 1]) if "--threshold" in sys.argv else 0.8
    
    # Run your eval suite
    results = suite.run(verbose=True)
    
    # Save results
    with open("eval_results.json", "w") as f:
        json.dump(results, f, indent=2)
    
    # Fail CI if below threshold
    if results["pass_rate"] < threshold:
        print(f"\n❌ CI FAILED: Pass rate {results['pass_rate']:.1%} < threshold {threshold:.1%}")
        sys.exit(1)
    else:
        print(f"\n✅ CI PASSED: Pass rate {results['pass_rate']:.1%} >= threshold {threshold:.1%}")
        sys.exit(0)

main()
```

---

## Module 17 — Observability, Tracing & Monitoring

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐⭐☆

> *You can't improve what you can't see. Observability is non-negotiable in production AI. This is what separates demo-quality from production-quality systems.*

---

### 17.1 — What AI Observability Means

Traditional APM tells you: CPU usage, memory, p95 latency.

AI Observability tells you:
- Which prompts triggered which model calls?
- What context was sent to the model?
- What was retrieved from the vector database?
- Did the model call any tools? With what arguments?
- Did the answer pass quality thresholds?
- How much did this request cost?

---

### 17.2 — Langfuse — Open-Source AI Observability

Langfuse is the production standard for open-source AI observability. Self-hostable, free, and excellent.

```bash
# Run Langfuse locally with Docker
docker compose up -d

# Or use cloud version at cloud.langfuse.com
```

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context
from openai import OpenAI

# Initialize
langfuse = Langfuse(
    public_key=os.getenv("LANGFUSE_PUBLIC_KEY"),
    secret_key=os.getenv("LANGFUSE_SECRET_KEY"),
    host=os.getenv("LANGFUSE_HOST", "https://cloud.langfuse.com")
)

client = OpenAI()

# The @observe decorator automatically traces function calls
@observe()
def retrieve_context(query: str) -> list[str]:
    """Traced retrieval step"""
    results = collection.query(...)
    
    # Add custom metadata to the trace
    langfuse_context.update_current_observation(
        input=query,
        output=results,
        metadata={"top_k": 5, "similarity_threshold": 0.7}
    )
    
    return results

@observe()
def generate_answer(query: str, context: list[str]) -> str:
    """Traced generation step"""
    context_str = "\n\n".join(context)
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Answer using the provided context."},
            {"role": "user", "content": f"Context: {context_str}\n\nQuestion: {query}"}
        ]
    )
    
    answer = response.choices[0].message.content
    
    # Log token usage and cost
    langfuse_context.update_current_observation(
        usage={
            "input": response.usage.prompt_tokens,
            "output": response.usage.completion_tokens,
        }
    )
    
    return answer

@observe(name="rag-pipeline")  # Name appears in Langfuse dashboard
def rag_pipeline(user_query: str) -> str:
    """Full RAG pipeline with automatic tracing"""
    # Both child functions are automatically traced as spans
    context = retrieve_context(user_query)
    answer = generate_answer(user_query, context)
    
    # Add user feedback after the fact (update the trace)
    langfuse_context.update_current_trace(
        user_id="user-123",
        session_id="session-456",
        tags=["rag", "customer-support"]
    )
    
    return answer

# Run — everything is traced automatically
result = rag_pipeline("What is your return policy?")
```

**What you'll see in Langfuse:**
- Full trace tree: pipeline → retrieval → generation
- Input/output at each step
- Token usage and cost per request
- Latency breakdown
- You can score traces manually for quality
- Filter by user, session, time, model

---

### 17.3 — Custom Metrics & Alerting

```python
from langfuse import Langfuse

langfuse = Langfuse()

def score_and_alert(trace_id: str, answer: str, query: str):
    """Evaluate quality and create alerts for poor responses"""
    
    # Run quick quality check
    is_relevant = "i don't know" not in answer.lower()
    is_complete = len(answer.split()) > 20  # Very basic completeness check
    
    # Score the trace in Langfuse (shows in dashboard)
    langfuse.score(
        trace_id=trace_id,
        name="relevance",
        value=1.0 if is_relevant else 0.0,
        comment=f"Answer {'is' if is_relevant else 'is NOT'} relevant"
    )
    
    langfuse.score(
        trace_id=trace_id,
        name="completeness",
        value=1.0 if is_complete else 0.5,
    )
    
    # Alert on low quality (in production: send to Slack/PagerDuty)
    if not is_relevant or not is_complete:
        print(f"⚠️ LOW QUALITY RESPONSE DETECTED")
        print(f"Query: {query}")
        print(f"Answer: {answer[:200]}")
        # In production: post to Slack webhook
```

---

## Module 18 — Guardrails, Safety & Security

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐⭐☆

> *Guardrails are the difference between a responsible AI product and a liability. This module is not optional.*

---

### 18.1 — Types of Guardrails

| Guardrail Type | What It Does | Implementation |
|----------------|-------------|----------------|
| **Input filtering** | Block bad inputs before they reach the LLM | Rule-based + classifier |
| **Output filtering** | Block bad outputs before they reach the user | Rule-based + LLM judge |
| **Topic boundaries** | Keep the model on-topic | System prompt + classifier |
| **PII protection** | Prevent personal data leakage | Regex + NER model |
| **Prompt injection defense** | Prevent jailbreaks | Input sanitization + prompting |

---

### 18.2 — Building a Guardrail Layer

```python
import re
from openai import OpenAI
from typing import Optional

client = OpenAI()

class GuardrailViolation(Exception):
    def __init__(self, message: str, violation_type: str):
        super().__init__(message)
        self.violation_type = violation_type

class InputGuardrails:
    """Pre-process and validate user inputs"""
    
    # Patterns that indicate prompt injection attempts
    INJECTION_PATTERNS = [
        r"ignore (previous|all|your) (instructions|system prompt)",
        r"pretend you are",
        r"act as (if you are|a|an) (?!assistant)",
        r"disregard (your|all) (rules|constraints|guidelines)",
        r"(jailbreak|DAN|do anything now)",
        r"your (real|true) (personality|self|purpose) is",
    ]
    
    # Simple PII patterns (use a proper NER library in production)
    PII_PATTERNS = {
        "credit_card": r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b",
        "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
        "indian_aadhar": r"\b\d{4}\s\d{4}\s\d{4}\b",
        "phone": r"\b(\+91|0)?[789]\d{9}\b",
    }
    
    def check_injection(self, text: str) -> Optional[str]:
        """Check for prompt injection attempts"""
        text_lower = text.lower()
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, text_lower, re.IGNORECASE):
                return f"Potential prompt injection detected: '{pattern}'"
        return None
    
    def check_pii(self, text: str) -> dict[str, list[str]]:
        """Detect PII in input"""
        found_pii = {}
        for pii_type, pattern in self.PII_PATTERNS.items():
            matches = re.findall(pattern, text)
            if matches:
                found_pii[pii_type] = matches
        return found_pii
    
    def redact_pii(self, text: str) -> str:
        """Replace PII with redacted placeholders"""
        for pii_type, pattern in self.PII_PATTERNS.items():
            placeholder = f"[REDACTED_{pii_type.upper()}]"
            text = re.sub(pattern, placeholder, text)
        return text
    
    def validate(self, user_input: str, allow_pii: bool = False) -> str:
        """Run all input guardrails"""
        
        # 1. Check for empty input
        if not user_input.strip():
            raise GuardrailViolation("Empty input", "EMPTY_INPUT")
        
        # 2. Check length
        if len(user_input) > 10000:
            raise GuardrailViolation("Input too long", "TOO_LONG")
        
        # 3. Check for injection
        injection_issue = self.check_injection(user_input)
        if injection_issue:
            raise GuardrailViolation(injection_issue, "PROMPT_INJECTION")
        
        # 4. Handle PII
        pii_found = self.check_pii(user_input)
        if pii_found and not allow_pii:
            user_input = self.redact_pii(user_input)
            print(f"[Guardrail] PII redacted: {list(pii_found.keys())}")
        
        return user_input

class OutputGuardrails:
    """Validate AI outputs before serving to users"""
    
    def __init__(self, allowed_topics: list[str] = None):
        self.allowed_topics = allowed_topics
    
    def check_harmful_content(self, output: str) -> Optional[str]:
        """Use OpenAI's moderation API to check for harmful content"""
        response = client.moderations.create(input=output)
        result = response.results[0]
        
        if result.flagged:
            categories = [k for k, v in result.categories.model_dump().items() if v]
            return f"Output flagged for: {', '.join(categories)}"
        return None
    
    def check_topic_relevance(self, user_query: str, ai_output: str) -> float:
        """Use LLM to check if output is on-topic"""
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"""Is this AI response on-topic for the allowed topics?

Allowed topics: {', '.join(self.allowed_topics or ['anything'])}
User query: {user_query}
AI response: {ai_output[:500]}

Rate relevance: 1.0 = completely on-topic, 0.0 = completely off-topic
JSON: {{"score": float, "reasoning": "brief"}}"""
            }],
            response_format={"type": "json_object"},
            temperature=0.0
        )
        import json
        result = json.loads(response.choices[0].message.content)
        return result["score"]
    
    def validate(self, user_query: str, ai_output: str) -> str:
        """Run all output guardrails"""
        
        # 1. Harmful content check
        harmful = self.check_harmful_content(ai_output)
        if harmful:
            raise GuardrailViolation(f"Output contains harmful content: {harmful}", "HARMFUL_CONTENT")
        
        # 2. Topic relevance check (if configured)
        if self.allowed_topics:
            relevance = self.check_topic_relevance(user_query, ai_output)
            if relevance < 0.5:
                return "I can only help with topics related to our products and services. Could you rephrase your question?"
        
        # 3. Redact any PII that leaked into output
        pii_guardrail = InputGuardrails()
        return pii_guardrail.redact_pii(ai_output)

# Putting it all together
class SafeAIService:
    def __init__(self, allowed_topics: list[str] = None):
        self.input_guard = InputGuardrails()
        self.output_guard = OutputGuardrails(allowed_topics)
    
    def answer(self, user_query: str) -> str:
        # 1. Validate and clean input
        try:
            safe_input = self.input_guard.validate(user_query)
        except GuardrailViolation as e:
            return f"I can't process that request: {e.violation_type}"
        
        # 2. Get AI response
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a helpful customer support agent."},
                {"role": "user", "content": safe_input}
            ]
        )
        raw_output = response.choices[0].message.content
        
        # 3. Validate output
        try:
            safe_output = self.output_guard.validate(user_query, raw_output)
        except GuardrailViolation as e:
            return "I encountered an issue generating a safe response. Please try again."
        
        return safe_output
```

---

## Module 19 — Fine-Tuning & LoRA

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐⭐⭐

> *Fine-tuning is misunderstood. It's NOT the first thing you should do. After this module, you'll know exactly when to fine-tune and how.*

---

### 19.1 — When to Fine-Tune (And When NOT To)

**The decision tree:**
```
Is your task failing with prompting alone?
  No → Keep using prompting. You don't need fine-tuning.
  Yes ↓
Is the failure due to missing knowledge?
  Yes → Use RAG. Fine-tuning doesn't add reliable new facts.
  No ↓
Is the failure due to wrong style, format, or behavior?
  Yes → Fine-tuning is appropriate.
  No ↓
Can you solve it with better few-shot examples?
  Yes → Add examples to prompt first (cheaper).
  No → Fine-tune.
```

**Good reasons to fine-tune:**
- You need a specific output format the model struggles to follow consistently
- You need a specific persona or tone maintained exactly
- You're making hundreds of calls and want to reduce prompt length (cost savings)
- You need the model to understand domain-specific language (medical, legal)
- You want speed — fine-tuned smaller models can beat large models at specific tasks

**Bad reasons to fine-tune:**
- "To make it smarter" — doesn't work
- "To add recent knowledge" — use RAG
- "Because prompting failed once" — try better prompting first

---

### 19.2 — OpenAI Fine-Tuning

```python
import json
from openai import OpenAI

client = OpenAI()

# Step 1: Prepare training data (JSONL format)
training_data = [
    {
        "messages": [
            {"role": "system", "content": "You extract product information in JSON format."},
            {"role": "user", "content": "Apple iPhone 15 Pro Max 256GB Natural Titanium - ₹1,59,900"},
            {"role": "assistant", "content": '{"brand": "Apple", "model": "iPhone 15 Pro Max", "storage": "256GB", "color": "Natural Titanium", "price": 159900, "currency": "INR"}'}
        ]
    },
    # ... 50+ more examples (minimum recommended: 50-100)
]

# Write to JSONL
with open("training_data.jsonl", "w") as f:
    for example in training_data:
        f.write(json.dumps(example) + "\n")

# Step 2: Upload training file
training_file = client.files.create(
    file=open("training_data.jsonl", "rb"),
    purpose="fine-tune"
)
print(f"Training file uploaded: {training_file.id}")

# Step 3: Create fine-tuning job
job = client.fine_tuning.jobs.create(
    training_file=training_file.id,
    model="gpt-4o-mini",  # Fine-tune the cheaper model
    hyperparameters={
        "n_epochs": 3,          # Usually 3-5 is enough
        "batch_size": "auto",
        "learning_rate_multiplier": "auto"
    },
    suffix="product-extractor"  # Your model will be: ft:gpt-4o-mini:your-org:product-extractor:xyz
)

print(f"Fine-tuning job created: {job.id}")

# Step 4: Monitor progress
import time
while True:
    job = client.fine_tuning.jobs.retrieve(job.id)
    print(f"Status: {job.status}")
    
    if job.status in ["succeeded", "failed"]:
        break
    
    time.sleep(60)  # Check every minute

if job.status == "succeeded":
    model_id = job.fine_tuned_model
    print(f"Fine-tuned model: {model_id}")
    
    # Test the fine-tuned model
    response = client.chat.completions.create(
        model=model_id,
        messages=[
            {"role": "system", "content": "You extract product information in JSON format."},
            {"role": "user", "content": "Samsung Galaxy S24 Ultra 512GB Titanium Black - ₹1,34,999"}
        ]
    )
    print(response.choices[0].message.content)
```

---

### 19.3 — LoRA Fine-Tuning with HuggingFace (Open-Source)

For open-source models, LoRA (Low-Rank Adaptation) is the standard approach:

```python
# Install: pip install transformers peft datasets trl accelerate bitsandbytes

from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, TaskType
from trl import SFTTrainer, SFTConfig
from datasets import Dataset
import torch

# Load base model with 4-bit quantization (fits in less GPU memory)
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_quant_type="nf4",
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.2-3B-Instruct",
    quantization_config=bnb_config,
    device_map="auto"
)

tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.2-3B-Instruct")
tokenizer.pad_token = tokenizer.eos_token

# LoRA configuration — only train a tiny fraction of parameters
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,              # Rank — higher = more capacity but more memory
    lora_alpha=32,     # Scaling factor
    lora_dropout=0.1,
    target_modules=["q_proj", "v_proj"]  # Which layers to adapt
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 4,194,304 || all params: 3,215,653,888 || trainable%: 0.1305

# Prepare dataset
def format_instruction(example):
    return f"""### Instruction:
{example['instruction']}

### Input:
{example['input']}

### Response:
{example['output']}"""

# Fine-tune with SFTTrainer
trainer = SFTTrainer(
    model=model,
    train_dataset=your_dataset,
    peft_config=lora_config,
    args=SFTConfig(
        output_dir="./fine-tuned-model",
        num_train_epochs=3,
        per_device_train_batch_size=4,
        learning_rate=2e-4,
        logging_steps=10,
        save_strategy="epoch",
    ),
    formatting_func=format_instruction,
)

trainer.train()
trainer.save_model("./fine-tuned-model")

# Push to HuggingFace Hub for deployment
model.push_to_hub("your-username/your-fine-tuned-model")
```

---

### 📚 Phase 4 Resources

**Watch:**
- 🎥 [Hamel Husain — Evaluating LLMs: A Practical Guide](https://www.youtube.com/watch?v=kMIZlqGP2_w)
- 🎥 [DeepLearning.AI — Evaluating and Debugging Generative AI](https://learn.deeplearning.ai/courses/evaluating-debugging-generative-ai)
- 🎥 [Langfuse — Getting Started](https://www.youtube.com/watch?v=2E8iTvGo5Hs)
- 🎥 [Maxime Labonne — Fine-Tune Llama 3.2](https://www.youtube.com/watch?v=pK6GA0sJMaM)

**Read:**
- 📄 [Hamel Husain — Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/)
- 📄 [Langfuse Documentation](https://langfuse.com/docs)
- 📄 [RAGAS Documentation](https://docs.ragas.io)
- 📄 [Guardrails AI Documentation](https://www.guardrailsai.com/docs)

**GitHub:**
- ⭐ [RAGAS — RAG Evaluation](https://github.com/explodinggradients/ragas)
- ⭐ [DeepEval — LLM Testing](https://github.com/confident-ai/deepeval)
- ⭐ [Langfuse](https://github.com/langfuse/langfuse)

---

### 🏗️ Portfolio Project 05 — Production RAG Platform

**This is your Phase 4 capstone project.** It brings together evals, observability, and guardrails into one production-grade system.

**What you'll build:** A production-grade document Q&A platform

**Production requirements (non-negotiable):**
1. **Evals:** RAGAS scores run automatically on every code change via CI/CD
2. **Observability:** Every request traced in Langfuse with costs
3. **Guardrails:** Input validation, output quality check, topic boundary enforcement
4. **Monitoring dashboard:** Real-time view of quality metrics
5. **A/B testing:** Compare two RAG strategies on the same traffic

**Architecture:**
```
User → Guardrails → RAG Pipeline → Guardrails → User
           ↓              ↓             ↓
       Rejected?    Langfuse Trace   Quality Score
           ↓              ↓             ↓
         Return       Dashboard      Alert if Low
```

**Tech:** FastAPI + Next.js + ChromaDB + Langfuse + GitHub Actions

**Push to:** `ai-engineering-journey/projects/06-production-rag-platform/`

---

## 🔁 Phase 3 & 4 Revision Checkpoint

**Teach-back test:**
Without looking at notes, explain in 5 minutes:
1. What is the ReAct pattern?
2. What is a LangGraph node vs an edge?
3. What is the difference between evals and observability?
4. Name 4 types of guardrails and give an example of each
5. When should you fine-tune vs use RAG?

**Code test:**
From memory, write:
- A simple 2-tool agent that can calculate and get current time
- A RAGAS evaluation setup for faithfulness and answer relevancy
- A function that detects prompt injection using regex

---

*[End of Part 3 — Modules 11–19]*
*Next: Module 20 — LLMOps | Module 21 — Cost Optimization | Module 22 — Full-Stack Patterns | Module 23 — Capstone*
---

# PHASE 5: SHIPPING & SCALE

---

## Module 20 — LLMOps & CI/CD for AI

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐⭐☆

> *LLMOps is DevOps for AI systems. It's the operational layer that lets you ship confidently and iterate quickly without breaking production.*

---

### 20.1 — The LLMOps Stack

```
Development          Testing              Production
─────────────        ─────────────        ─────────────
Local Ollama    →    Eval Suite      →    OpenAI/Anthropic
Jupyter/VSCode  →    GitHub Actions  →    Langfuse (observability)
Langfuse Dev    →    Staging env     →    Langfuse Prod
Prompt dev      →    A/B test        →    Feature flags
```

---

### 20.2 — Prompt Version Management

Never hardcode prompts. Version control them like code:

```python
# prompts/customer_support_v2.txt (tracked in git)
"""
## Role
You are a helpful customer support agent for ShopIndia...
[... full prompt ...]
"""

# prompts/__init__.py
from pathlib import Path
import os

PROMPTS_DIR = Path(__file__).parent

def load_prompt(name: str, version: str = "latest") -> str:
    """Load a prompt by name and version"""
    if version == "latest":
        # Find highest version
        files = list(PROMPTS_DIR.glob(f"{name}_v*.txt"))
        if not files:
            raise FileNotFoundError(f"No prompt found for {name}")
        latest = sorted(files, key=lambda f: int(f.stem.split('_v')[-1]))[-1]
        return latest.read_text()
    else:
        return (PROMPTS_DIR / f"{name}_v{version}.txt").read_text()

# Usage
system_prompt = load_prompt("customer_support")  # Always gets latest
system_prompt = load_prompt("customer_support", "2")  # Specific version
```

---

### 20.3 — Deployment Patterns for AI Applications

**Pattern 1: API-First (Most Common)**
```python
# FastAPI app with proper AI endpoint patterns
from fastapi import FastAPI, HTTPException, BackgroundTasks
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import asyncio

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    session_id: str
    user_id: str

class ChatResponse(BaseModel):
    answer: str
    session_id: str
    trace_id: str
    cost_estimate_usd: float

@app.post("/chat")
async def chat(request: ChatRequest):
    try:
        # Your AI pipeline here
        result = await run_rag_pipeline(
            query=request.message,
            session_id=request.session_id,
            user_id=request.user_id
        )
        return ChatResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat/stream")  
async def chat_stream(request: ChatRequest):
    async def generate():
        async for token in stream_rag_pipeline(request.message):
            yield f"data: {token}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")

# Health check — important for production
@app.get("/health")
async def health():
    return {
        "status": "healthy",
        "model": os.getenv("LLM_MODEL", "gpt-4o-mini"),
        "version": os.getenv("APP_VERSION", "unknown")
    }
```

**Docker deployment:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Non-root user for security
RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

---

### 20.4 — Feature Flags for AI

Feature flags let you gradually roll out AI changes without full deployments:

```python
import os
import random
from enum import Enum

class AIFeature(Enum):
    NEW_RAG_PIPELINE = "new_rag_pipeline"
    RERANKING_ENABLED = "reranking_enabled"
    GPT4O_MODEL = "gpt4o_model"
    HYDE_RETRIEVAL = "hyde_retrieval"

class FeatureFlags:
    def __init__(self):
        # In production: fetch from LaunchDarkly, Split, or a config service
        self.flags = {
            AIFeature.NEW_RAG_PIPELINE: float(os.getenv("FLAG_NEW_RAG", "0.0")),
            AIFeature.RERANKING_ENABLED: float(os.getenv("FLAG_RERANKING", "1.0")),
            AIFeature.GPT4O_MODEL: float(os.getenv("FLAG_GPT4O", "0.1")),  # 10% rollout
            AIFeature.HYDE_RETRIEVAL: float(os.getenv("FLAG_HYDE", "0.5")),
        }
    
    def is_enabled(self, feature: AIFeature, user_id: str) -> bool:
        """Deterministic flag evaluation based on user_id"""
        rollout = self.flags.get(feature, 0.0)
        if rollout >= 1.0:
            return True
        if rollout <= 0.0:
            return False
        
        # Hash user_id to get consistent result per user
        user_hash = hash(f"{user_id}:{feature.value}") % 100
        return user_hash < (rollout * 100)

flags = FeatureFlags()

async def smart_rag_pipeline(query: str, user_id: str) -> str:
    """Use feature flags to run different pipeline variations"""
    
    model = "gpt-4o" if flags.is_enabled(AIFeature.GPT4O_MODEL, user_id) else "gpt-4o-mini"
    use_reranking = flags.is_enabled(AIFeature.RERANKING_ENABLED, user_id)
    use_hyde = flags.is_enabled(AIFeature.HYDE_RETRIEVAL, user_id)
    
    # Log which flags were used (crucial for A/B analysis)
    print(f"Pipeline: model={model}, reranking={use_reranking}, hyde={use_hyde}")
    
    # Run pipeline with selected features
    # ...
```

---

## Module 21 — Cost Optimization & Latency

**Time:** ~1 week | **Difficulty:** ⭐⭐⭐⭐☆

> *A demo doesn't care about cost. A production system serving 10,000 users/day absolutely does. This module teaches you to build AI systems that are fast and affordable.*

---

### 21.1 — The Cost Optimization Playbook

**Costs in a typical RAG system:**
- Embedding queries: ~$0.00002/1k tokens (negligible)
- Input context (question + retrieved docs): $0.0025-0.01/1k tokens
- Output generation: $0.005-0.02/1k tokens
- Reranking (Cohere): $0.001/1k tokens

**At scale: 10,000 queries/day with 5,000-token average context:**
- Without optimization: ~$250/day
- With optimization: ~$25/day (10x reduction possible)

---

### 21.2 — Caching Strategies

```python
import hashlib
import json
import redis
from typing import Optional
from functools import wraps

# Redis for distributed caching
cache = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cache_key(prefix: str, *args, **kwargs) -> str:
    """Generate deterministic cache key"""
    content = json.dumps({"args": args, "kwargs": kwargs}, sort_keys=True)
    content_hash = hashlib.md5(content.encode()).hexdigest()
    return f"{prefix}:{content_hash}"

def cached_embedding(text: str, model: str = "text-embedding-3-small") -> list[float]:
    """Cache embeddings — same text always produces same embedding"""
    key = cache_key("embed", text, model)
    
    cached = cache.get(key)
    if cached:
        return json.loads(cached)
    
    # Compute embedding
    response = client.embeddings.create(model=model, input=text)
    embedding = response.data[0].embedding
    
    # Cache for 7 days (embeddings don't change)
    cache.setex(key, 7 * 24 * 3600, json.dumps(embedding))
    
    return embedding

def cached_llm_response(
    messages: list[dict], 
    model: str,
    ttl_seconds: int = 3600  # 1 hour default
) -> Optional[str]:
    """Cache LLM responses for identical inputs"""
    key = cache_key("llm", messages, model)
    
    cached = cache.get(key)
    if cached:
        print("[Cache HIT] Returning cached LLM response")
        return cached
    
    # Get fresh response
    response = client.chat.completions.create(model=model, messages=messages)
    result = response.choices[0].message.content
    
    # Cache only for temperature=0 responses (deterministic)
    cache.setex(key, ttl_seconds, result)
    print("[Cache MISS] Response cached")
    
    return result

# Semantic caching — cache based on similar questions, not exact matches
class SemanticCache:
    """Cache responses for semantically similar queries"""
    
    def __init__(self, similarity_threshold: float = 0.95):
        self.threshold = similarity_threshold
        self.cached_queries: list[dict] = []  # In production: use vector DB
    
    def get(self, query: str) -> Optional[str]:
        """Find cached response for a similar query"""
        if not self.cached_queries:
            return None
        
        query_embedding = cached_embedding(query)
        
        # Find most similar cached query
        best_match = None
        best_similarity = 0
        
        for item in self.cached_queries:
            sim = cosine_similarity(query_embedding, item["embedding"])
            if sim > best_similarity:
                best_similarity = sim
                best_match = item
        
        if best_similarity >= self.threshold:
            print(f"[Semantic Cache HIT] Similarity: {best_similarity:.3f}")
            return best_match["response"]
        
        return None
    
    def store(self, query: str, response: str):
        """Store a query-response pair"""
        self.cached_queries.append({
            "query": query,
            "embedding": cached_embedding(query),
            "response": response
        })

semantic_cache = SemanticCache(similarity_threshold=0.92)
```

---

### 21.3 — Model Selection for Cost Efficiency

```python
def select_model(query: str, context_length: int) -> str:
    """
    Dynamically select the most cost-effective model for the task.
    You don't always need GPT-4o!
    """
    
    # Simple classification of query complexity
    complexity_indicators = {
        "high": ["analyze", "compare", "synthesize", "evaluate", "design", "architect", "debug"],
        "low": ["what is", "define", "list", "when", "who", "translate", "summarize"]
    }
    
    query_lower = query.lower()
    
    is_complex = any(word in query_lower for word in complexity_indicators["high"])
    is_simple = any(phrase in query_lower for phrase in complexity_indicators["low"])
    
    # Long context = need more capable model
    needs_long_context = context_length > 50000
    
    if needs_long_context or is_complex:
        return "gpt-4o"           # Most capable, most expensive
    elif is_simple:
        return "gpt-4o-mini"      # 10x cheaper, great for simple tasks
    else:
        return "gpt-4o-mini"      # Default to cheaper when uncertain

# Cost tracking
class CostTracker:
    PRICING = {
        "gpt-4o": {"input": 0.0025, "output": 0.01},        # per 1k tokens
        "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
        "claude-3-5-sonnet": {"input": 0.003, "output": 0.015},
        "text-embedding-3-small": {"input": 0.00002, "output": 0},
    }
    
    def __init__(self):
        self.daily_cost = 0.0
        self.calls = 0
    
    def log_call(self, model: str, input_tokens: int, output_tokens: int) -> float:
        pricing = self.PRICING.get(model, {"input": 0.01, "output": 0.03})
        cost = (input_tokens / 1000 * pricing["input"]) + (output_tokens / 1000 * pricing["output"])
        self.daily_cost += cost
        self.calls += 1
        return cost
    
    def report(self) -> dict:
        return {
            "daily_cost_usd": round(self.daily_cost, 4),
            "total_calls": self.calls,
            "avg_cost_per_call_usd": round(self.daily_cost / max(1, self.calls), 6),
            "monthly_estimate_usd": round(self.daily_cost * 30, 2)
        }

tracker = CostTracker()
```

---

### 21.4 — Latency Optimization

```python
import asyncio
from typing import Any

# Pattern 1: Parallel retrieval + pre-processing
async def optimized_rag_pipeline(query: str) -> str:
    """Run retrieval, embedding, and pre-processing in parallel"""
    
    # These can run simultaneously!
    embedding_task = asyncio.create_task(
        get_embedding_async(query)
    )
    
    # While embedding, start pre-processing query
    # (classify intent, extract entities, etc.)
    intent_task = asyncio.create_task(
        classify_intent_async(query)
    )
    
    # Wait for both
    embedding, intent = await asyncio.gather(embedding_task, intent_task)
    
    # Now search with the embedding
    results = await search_async(embedding, intent)
    
    # Generate answer (can't be parallelized — needs search results)
    answer = await generate_async(query, results)
    
    return answer

# Pattern 2: Streaming to hide latency
async def streaming_rag(query: str):
    """
    Start streaming immediately — users perceive this as faster
    even if total time is similar.
    """
    # Retrieve context (fast — ~200ms)
    context = await retrieve_fast(query)
    
    # Stream generation immediately
    stream = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[...],
        stream=True
    )
    
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

# Pattern 3: Prompt compression for long contexts
def compress_context(chunks: list[str], max_tokens: int = 2000) -> str:
    """Compress retrieved context when it's too long"""
    
    # Simple approach: truncate to max_tokens
    combined = "\n\n".join(chunks)
    
    if count_tokens(combined) <= max_tokens:
        return combined
    
    # Compress using LLM (spend tokens to save tokens in main call)
    response = client.chat.completions.create(
        model="gpt-4o-mini",  # Use cheap model for compression
        messages=[{
            "role": "user",
            "content": f"Summarize this in under 500 words, preserving all key facts:\n\n{combined[:4000]}"
        }],
        max_tokens=700
    )
    return response.choices[0].message.content
```

---

## Module 22 — Full-Stack AI Application Patterns

**Time:** ~1.5 weeks | **Difficulty:** ⭐⭐⭐⭐☆

> *You know how to build the AI backend. This module connects it to real frontend experiences — streaming UIs, chat interfaces, tool invocation UX, and more.*

---

### 22.1 — The Vercel AI SDK (Best Way to Build AI UIs)

```typescript
// Next.js 15 + Vercel AI SDK

// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText, tool } from 'ai';
import { z } from 'zod';

export const runtime = 'edge';  // Edge runtime for low latency

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: openai('gpt-4o-mini'),
    system: 'You are a helpful assistant.',
    messages,
    tools: {
      // Define tools the model can call
      getWeather: tool({
        description: 'Get current weather for a city',
        parameters: z.object({
          city: z.string().describe('City name'),
        }),
        execute: async ({ city }) => {
          // Call your weather API
          return { city, temperature: 28, condition: 'Sunny' };
        },
      }),
      searchDatabase: tool({
        description: 'Search the product database',
        parameters: z.object({
          query: z.string(),
          limit: z.number().default(5),
        }),
        execute: async ({ query, limit }) => {
          // Call your backend
          const res = await fetch(`/api/products/search?q=${query}&limit=${limit}`);
          return res.json();
        },
      }),
    },
    maxSteps: 5,  // Allow multi-step tool use
    onFinish: async ({ usage, finishReason }) => {
      // Log to your observability platform
      console.log('Tokens used:', usage);
    },
  });

  return result.toDataStreamResponse();
}
```

```tsx
// app/chat/page.tsx
'use client';

import { useChat } from 'ai/react';
import { useState } from 'react';

export default function ChatPage() {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
    error,
    stop,
  } = useChat({
    api: '/api/chat',
    onError: (err) => console.error('Chat error:', err),
  });

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      {/* Messages */}
      <div className="flex-1 overflow-y-auto space-y-4 mb-4">
        {messages.map((message) => (
          <div
            key={message.id}
            className={`flex ${message.role === 'user' ? 'justify-end' : 'justify-start'}`}
          >
            <div
              className={`max-w-[80%] rounded-lg px-4 py-2 ${
                message.role === 'user'
                  ? 'bg-blue-500 text-white'
                  : 'bg-gray-100 text-gray-900'
              }`}
            >
              {/* Show tool invocations */}
              {message.toolInvocations?.map((inv) => (
                <div key={inv.toolCallId} className="text-xs opacity-70 mb-1">
                  🔧 {inv.toolName}({JSON.stringify(inv.args)})
                </div>
              ))}
              <div className="whitespace-pre-wrap">{message.content}</div>
            </div>
          </div>
        ))}
        
        {isLoading && (
          <div className="flex justify-start">
            <div className="bg-gray-100 rounded-lg px-4 py-2 animate-pulse">
              Thinking...
            </div>
          </div>
        )}
      </div>

      {/* Input */}
      <form onSubmit={handleSubmit} className="flex gap-2">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask anything..."
          className="flex-1 border rounded-lg px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500"
          disabled={isLoading}
        />
        {isLoading ? (
          <button
            type="button"
            onClick={stop}
            className="bg-red-500 text-white rounded-lg px-4 py-2 hover:bg-red-600"
          >
            Stop
          </button>
        ) : (
          <button
            type="submit"
            className="bg-blue-500 text-white rounded-lg px-4 py-2 hover:bg-blue-600 disabled:opacity-50"
            disabled={!input.trim()}
          >
            Send
          </button>
        )}
      </form>
    </div>
  );
}
```

---

### 22.2 — RAG UI Patterns

**Show your sources — users trust cited answers more:**
```tsx
interface Source {
  id: string;
  content: string;
  score: number;
  metadata: { source: string; page?: number };
}

interface AnswerWithSources {
  answer: string;
  sources: Source[];
}

function AnswerCard({ answer, sources }: AnswerWithSources) {
  const [showSources, setShowSources] = useState(false);
  
  return (
    <div className="border rounded-lg p-4">
      <div className="prose max-w-none">{answer}</div>
      
      {sources.length > 0 && (
        <div className="mt-3">
          <button
            onClick={() => setShowSources(!showSources)}
            className="text-sm text-blue-500 hover:underline"
          >
            {showSources ? 'Hide' : 'Show'} {sources.length} sources
          </button>
          
          {showSources && (
            <div className="mt-2 space-y-2">
              {sources.map((source, i) => (
                <div key={source.id} className="bg-gray-50 rounded p-2 text-sm">
                  <div className="font-medium text-gray-700">
                    [{i + 1}] {source.metadata.source}
                    {source.metadata.page && ` (p.${source.metadata.page})`}
                  </div>
                  <div className="text-gray-600 mt-1">{source.content.slice(0, 200)}...</div>
                  <div className="text-xs text-gray-400 mt-1">
                    Relevance: {(source.score * 100).toFixed(0)}%
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

---

### 📚 Phase 5 Resources

**Watch:**
- 🎥 [Vercel AI SDK — Full Tutorial](https://www.youtube.com/watch?v=j3jURNlWE_I)
- 🎥 [LLMOps — MLflow for LLMs](https://www.youtube.com/watch?v=8EhP7fCsBXk)
- 🎥 [AI Cost Optimization Strategies](https://www.youtube.com/watch?v=mcRzOPe6tN4)

**Read:**
- 📄 [Vercel AI SDK Documentation](https://sdk.vercel.ai/docs)
- 📄 [LiteLLM — Universal LLM Proxy](https://docs.litellm.ai)

---
---

# CAPSTONE

---

## Module 23 — Capstone: Build & Launch a Real AI Product

**Time:** ~6 weeks | **Difficulty:** ⭐⭐⭐⭐⭐

> *This is where everything comes together. You're not building a tutorial project — you're building something you'd be proud to show to a hiring manager or early users.*

---

### 23.1 — Choosing Your Capstone Project

Pick one that excites you. Here are 5 strong options with increasing complexity:

---

**Option A: AI-Powered Study Coach (India-relevant)**

A study assistant for competitive exam preparation (UPSC, JEE, NEET, CAT, GATE).

```
Core features:
- Upload your study materials (notes, textbooks, previous papers)
- AI explains concepts with multiple approaches until you understand
- Auto-generates practice questions from your materials
- Identifies weak areas based on your answers
- Creates personalized revision schedules
- Voice mode (text-to-speech answers)

Production complexity:
- Multi-tenant: each student has isolated vector store
- RAG: student's own notes + general knowledge base
- Fine-tuned model: exam-specific question format
- Evals: question quality, explanation accuracy
- Analytics: progress tracking, weak area detection
```

**Tech:** FastAPI + Next.js + Pinecone + OpenAI + Langfuse + PostgreSQL

---

**Option B: LegalEase — AI Contract Analyzer (B2B SaaS potential)**

AI-powered legal document analysis for Indian SMBs and freelancers.

```
Core features:
- Upload any contract (NDA, employment, service agreement)
- Plain-English summary of key terms
- Risk flagging: clauses unfavorable to user
- Comparison: compare two versions of a contract
- "Ask a question about this contract" Q&A
- Export annotated PDF with highlights

Production complexity:
- Long document handling (100+ page contracts)
- Multimodal: scanned PDF support
- Structured extraction: key dates, parties, obligations
- Citation: every answer cites the exact clause
- Guardrails: no legal advice, recommend lawyer for high-risk clauses
- Evals: extraction accuracy, risk detection F1 score
```

**Tech:** FastAPI + Next.js + pgvector + Claude (200k context) + instructor

---

**Option C: TechRecruit — AI Technical Interview Platform**

AI platform that conducts mock technical interviews and gives detailed feedback.

```
Core features:
- Conduct live mock coding interviews with AI interviewer
- Real-time code execution (sandboxed)
- AI asks follow-up questions based on your answers
- Evaluates: code quality, explanation, problem-solving approach
- Detailed feedback report after each session
- Company-specific interview prep (Google, Amazon, etc.)
- Track progress over time

Production complexity:
- Multi-turn conversation with memory across interview
- Code execution sandbox (Docker)
- RAG: company-specific interview patterns
- Multi-agent: one agent interviews, one evaluates
- Evals: question quality, feedback quality, problem difficulty calibration
```

**Tech:** LangGraph + Next.js + Docker (code execution) + Pinecone + Langfuse

---

**Option D: InsightBot — Analytics & BI AI Assistant**

Natural language interface to business data — "Ask your database questions in plain English."

```
Core features:
- Connect to PostgreSQL/MySQL/BigQuery
- Ask questions in natural language
- AI generates and executes SQL (safely)
- Visualize results automatically
- "Why did revenue drop last month?" — multi-step analysis
- Dashboard builder: save favorite queries as cards
- Anomaly detection: AI alerts on unusual patterns

Production complexity:
- Text-to-SQL with validation and safety guardrails
- Multi-step reasoning for complex analytics questions
- RAG: database schema + business context
- Agent: can run multiple queries to answer one question
- Evals: SQL correctness, query safety, answer accuracy
```

**Tech:** LangGraph + Next.js + FastAPI + pgvector + Langfuse

---

**Option E: ContentPilot — AI-Powered Social Media Manager (India market)**

AI tool for Indian content creators to manage their social media.

```
Core features:
- Generate content ideas based on trending topics in India
- Write posts in multiple Indian languages (Hindi, Bengali, etc.)
- Repurpose one long video into thread, carousel, shorts script
- Optimal posting time suggestions
- Brand voice consistency check
- Competitor analysis
- Auto-reply suggestions for comments

Production complexity:
- Multilingual: Hindi, Bengali, Tamil, English
- Multi-modal: analyze images, generate captions
- Fine-tuned model: brand voice adaptation
- Evals: content quality, brand consistency, engagement prediction
- Scheduler: background job processing
```

**Tech:** FastAPI + Next.js + OpenAI + Langfuse + Celery + Redis

---

### 23.2 — Capstone Production Requirements

Regardless of which project you pick, these are non-negotiable:

**Architecture (30% of grade):**
- [ ] Clear system design document (diagram + explanation)
- [ ] Modular, maintainable codebase
- [ ] Proper error handling throughout
- [ ] Environment-based configuration
- [ ] Docker containerization

**AI Engineering (30% of grade):**
- [ ] Advanced RAG (not naive chunking + retrieval)
- [ ] Structured outputs with validation
- [ ] At least one agentic component (tool use or multi-step)
- [ ] Provider abstraction (easy to swap models)

**Production Quality (25% of grade):**
- [ ] Eval suite with 30+ test cases running in CI/CD
- [ ] Langfuse integration (full tracing)
- [ ] Input + output guardrails
- [ ] Cost tracking per request
- [ ] Basic monitoring dashboard

**Portfolio Presentation (15% of grade):**
- [ ] Deployed and accessible URL (Vercel + Railway/Render)
- [ ] README with: what it is, how it works, architecture diagram, local setup
- [ ] Loom video walkthrough (3-5 minutes)
- [ ] Blog post on your journey building it (LinkedIn or dev.to)

---

### 23.3 — 6-Week Capstone Timeline

**Week 1 — Architecture & Setup**
- [ ] Choose project
- [ ] Write detailed system design (draw architecture diagram)
- [ ] Set up all infrastructure (DB, vector store, observability)
- [ ] Build core data ingestion pipeline
- [ ] Write first 10 eval test cases

**Week 2 — Core AI Pipeline**
- [ ] Build RAG pipeline (chunking, embedding, retrieval)
- [ ] Build agent workflow if applicable
- [ ] Write all prompt templates and version them
- [ ] Connect Langfuse — trace first requests
- [ ] Run RAGAS evals on retrieval quality

**Week 3 — Application Layer**
- [ ] Build FastAPI backend with all endpoints
- [ ] Build Next.js frontend with streaming
- [ ] Add authentication (Clerk or NextAuth)
- [ ] Add guardrails
- [ ] Write 20 more eval test cases

**Week 4 — Production Hardening**
- [ ] Set up GitHub Actions with eval gates
- [ ] Add caching for embeddings and common queries
- [ ] Cost tracking implementation
- [ ] Handle edge cases and error states
- [ ] Conduct red-teaming session (try to break it)

**Week 5 — Polish & Deploy**
- [ ] UI polish — make it look professional
- [ ] Deploy to production (Vercel + Railway)
- [ ] Load testing with Locust (simulate 100 concurrent users)
- [ ] Fix performance bottlenecks
- [ ] Final eval suite run — all passing?

**Week 6 — Document & Showcase**
- [ ] Write comprehensive README
- [ ] Record Loom walkthrough
- [ ] Write LinkedIn post / blog post
- [ ] Create system design document for portfolio
- [ ] Update your resume with this project
- [ ] Share in AI communities for feedback

---

### 23.4 — Documentation Template

Every portfolio project needs this README:

```markdown
# [Project Name] — [One-line description]

[![CI/CD](badge-url)](workflow-url)
[![Deploy](badge-url)](live-url)

> One-paragraph description of what this is and who it's for.

## Live Demo
[app.yourdomain.com](https://app.yourdomain.com) | [Loom walkthrough](https://loom.com/...)

## Problem Solved
[2-3 sentences on the problem and who faces it]

## How It Works
[Architecture diagram — Excalidraw or Mermaid]

[3-4 paragraph explanation of the AI engineering decisions you made]

## Tech Stack
- **LLM:** OpenAI GPT-4o-mini + Claude 3.5 Sonnet
- **Embeddings:** text-embedding-3-small
- **Vector DB:** Pinecone
- **Orchestration:** LangGraph
- **Observability:** Langfuse
- **Backend:** FastAPI + PostgreSQL
- **Frontend:** Next.js 15 + Vercel AI SDK

## Key AI Engineering Decisions
1. **Why I chose Claude for documents**: 200k context window handles full contracts
2. **Chunking strategy**: Semantic chunking improved retrieval recall from 67% to 89%
3. **HyDE for retrieval**: Reduced "I don't know" responses by 40%

## Eval Results
| Metric | Score |
|--------|-------|
| Faithfulness | 0.91 |
| Answer Relevancy | 0.87 |
| Context Recall | 0.83 |
| Pass Rate (30 cases) | 93.3% |

## Local Setup
[Instructions]

## Lessons Learned
[3-5 honest lessons from building this]
```

---

---

# POST-COURSE: WHAT'S NEXT

---

## Continuing Your Journey

You've completed the course. Here's what comes next:

### Stay Current (AI moves fast)

**Subscribe to:**
- 📧 [The Batch — Andrew Ng's AI Newsletter](https://www.deeplearning.ai/the-batch/)
- 📧 [AI Tidbits by Andrej Karpathy](https://tidbits.ai)
- 📧 [The AI Engineer — Weekly newsletter](https://www.theaiengineer.news/)
- 📧 [Simon Willison's Weblog](https://simonwillison.net)

**Follow on Twitter/X:**
- @karpathy (Andrej Karpathy)
- @swyx (Shawn Wang — AI Engineer)
- @hwchase17 (Harrison Chase — LangChain)
- @alexandr_wang (Scale AI)

**Communities:**
- [AI Engineer Discord](https://discord.gg/aiengineer)
- [LangChain Discord](https://discord.gg/langchain)
- [Hugging Face Discord](https://discord.gg/huggingface)
- [LocalLLaMA Reddit](https://reddit.com/r/LocalLLaMA)

---

### Your Portfolio at Course Completion

After completing this course, you should have:

1. **aiask CLI** — Streaming command-line AI assistant
2. **AI Content Studio** — Prompt engineering showcase
3. **Document Intelligence** — Structured output extraction
4. **Intelligent Support Bot** — Full RAG with evaluation
5. **Research Agent** — LangGraph multi-step agent
6. **Production RAG Platform** — Evals + observability + guardrails
7. **Capstone Project** — Your flagship product

**That's 7 real projects. Any senior AI engineer would respect this portfolio.**

---

### The Learning Journal Review

Go back through every entry in your learning journal. Write a reflection:
- What was the hardest concept to understand?
- What project taught you the most?
- What would you build next?
- What do you still feel uncertain about?
- What advice would you give yourself at the start?

---

### Career Paths

**Option 1: AI Engineer at a company**
Your portfolio demonstrates:
- RAG systems (95% of AI jobs need this)
- Agent development (fast-growing demand)
- Production discipline (evals, observability, guardrails)
- Full-stack delivery (frontend + backend + AI)

**Option 2: Build your own product**
Your capstone could become your startup:
- All the infrastructure is built
- You understand the tech deeply
- You know the costs and how to optimize
- You can iterate quickly

**Option 3: Freelance AI Engineering**
Companies are desperately hiring for:
- "Build us a RAG system on our documents"
- "Add an AI assistant to our product"
- "Evaluate and improve our existing AI features"

---

## Final Words from Your Senior Engineer

You started this course as a frontend/backend developer who wanted to learn AI. If you've done the work — actually built everything, not just read — you're now an AI Engineer.

Here's what I want you to know:

**The tools will change. The fundamentals won't.**

RAG is just retrieval + generation. Agents are just LLMs + tools + loops. Evals are just tests. These patterns will survive even as the specific libraries evolve.

**The "last 20%" is what defines you.**

Anyone can build a demo. The engineers who ship real products are the ones who build the eval pipelines, add the guardrails, add the observability, and do the boring work of making it reliable. That's the 20% most tutorials skip and the 20% that gets you hired or makes your product succeed.

**Build in public.**

Document your journey. Write about what confused you. Share your projects. The AI community is genuinely helpful and loves seeing real work. Your imperfect projects showing real learning are more valuable than a polished tutorial clone.

**Keep building.**

The best way to stay sharp is to keep shipping. After this course: build one AI-powered project per month. Even a weekend hack counts.

I'm proud of the work you've done. Go build something great.

---

## 📊 Course Completion Checklist

Track your progress:

### Phase 0 — Foundations
- [ ] Module 01: Landscape understood
- [ ] Module 02: `aiask` CLI built and deployed
- [ ] Module 03: Transformer internals understood

### Phase 1 — Working with LLMs
- [ ] Module 04: AI Content Studio built
- [ ] Module 05: Multi-provider experience
- [ ] Module 06: Document Intelligence built
- [ ] Module 07: Semantic search working

### Phase 2 — RAG Systems
- [ ] Module 08: Vector DB setup (local + cloud)
- [ ] Module 09: Support Bot with advanced RAG
- [ ] Module 10: Hybrid search implemented
- [ ] Module 11: Multimodal document processed

### Phase 3 — AI Agents
- [ ] Module 12: Agent loop from scratch
- [ ] Module 13: Research Agent with LangGraph
- [ ] Module 14: Multi-agent system working
- [ ] Module 15: MCP server built

### Phase 4 — Production AI
- [ ] Module 16: Eval suite with CI/CD
- [ ] Module 17: Langfuse tracing all requests
- [ ] Module 18: Guardrails on input and output
- [ ] Module 19: Fine-tuning experiment done

### Phase 5 — Shipping & Scale
- [ ] Module 20: Feature flags implemented
- [ ] Module 21: Caching + cost tracking
- [ ] Module 22: Full-stack AI app deployed

### Capstone
- [ ] Project chosen and scoped
- [ ] Architecture designed
- [ ] All core features built
- [ ] 30+ evals passing in CI/CD
- [ ] Deployed to production
- [ ] README + Loom recorded
- [ ] Blog post published

---

*This course was designed with the depth, rigor, and production focus of the best engineering courses available. The goal is not to finish — it's to genuinely understand and build. Take your time. Build everything. Ship everything. You've got this.*

---

**END OF COURSE**

*AI Engineering Course v1.0 — Designed for Partha*  
*Last updated: 2025*  
*Total duration: ~6-8 months at 2 hours/day*  
*Total projects: 7 portfolio projects + 1 capstone*
