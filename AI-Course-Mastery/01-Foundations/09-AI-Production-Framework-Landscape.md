# AI Production Framework Landscape — Know Your Tools

## 🎯 Purpose & Goals

> **🛑 STOP. Here's a problem:**
>
> A teammate says: "We need to build a RAG system. Let's use LangChain."
> Another says: "No, use LlamaIndex."
> Another says: "Just build it from scratch."
>
> **Who's right? What would YOU decide?**
>
> *More importantly: What questions would you ask to make this decision?*

---

**By the end of this file, you will:**
- Know the major AI frameworks and what each is best for
- Understand WHY each framework exists (what problem it solves)
- Be able to make an informed build-vs-framework decision
- Know which framework to start with for each use case

**⏱ Time Budget:** 1.5 hours

---

## 📖 The Framework Zoo

> **🤔 YOUR TURN:** Before I describe them, list EVERY AI framework/tool you've heard of. Then group them by what they do.

| What It Does | Frameworks |
|--------------|------------|
| Provider abstraction | ___ |
| RAG / Data pipelines | ___ |
| AI Agents | ___ |
| Prompt optimization | ___ |
| Observability | ___ |
| Evaluation | ___ |
| Structured outputs | ___ |
| Vector search | ___ |

<details>
<summary>👀 Full landscape</summary>

| What It Does | Frameworks |
|--------------|------------|
| Provider abstraction | LiteLLM, Portkey |
| RAG / Data pipelines | LangChain, LlamaIndex |
| AI Agents | LangGraph, Pydantic AI, CrewAI, AutoGen |
| Prompt optimization | DSPy |
| Observability | Langfuse, LangSmith, Weights & Biases |
| Evaluation | RAGAS, DeepEval, custom |
| Structured outputs | Instructor, Outlines |
| Vector search | ChromaDB, Qdrant, Pinecone, pgvector, Weaviate, FAISS |
</details>

---

## 📖 Decision Framework: When to Use What

```python
"""
THE KEY INSIGHT: Frameworks SOLVE PAIN POINTS.
Don't ask "which framework is best?" Ask "what pain am I feeling?"

PAIN: I keep rewriting OpenAI→Anthropic→Ollama adapters
  → USE: LiteLLM (provider abstraction)

PAIN: My RAG pipeline is becoming complex (chunking, embedding, retrieval, reranking)
  → USE: LangChain (if you want general-purpose) or LlamaIndex (if data-centric)

PAIN: My agents need state management, branching, persistence
  → USE: LangGraph (complex) or Pydantic AI (simpler)

PAIN: My prompts feel like guesswork, no systematic improvement
  → USE: DSPy (optimizes prompts programmatically)

PAIN: I can't debug why my AI system gave a bad answer
  → USE: Langfuse (traces every call, shows inputs/outputs)

PAIN: I can't measure if my RAG system is actually good
  → USE: RAGAS (RAG-specific evaluation metrics)
"""
```

---

## 📖 Deep Dive: The 3 You'll Use Most

### 1. LangChain — The Generalist

**Best for:** Rapid prototyping, RAG pipelines, chain composition  
**Worst for:** Fine-grained control, production at scale (over-abstraction)

```python
# What LangChain does well:
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema import StrOutputParser

# Chain = compose operations declaratively
prompt = ChatPromptTemplate.from_template("Tell me a {adjective} story about {topic}")
model = ChatOpenAI(model="gpt-4o-mini")
output_parser = StrOutputParser()

chain = prompt | model | output_parser  # 🔑 Pipe syntax
result = chain.invoke({"adjective": "funny", "topic": "a robot"})
```

**When to reach for LangChain:**
- You need to try 5 different RAG approaches in 2 hours
- You want to add tools/agents to your app quickly
- You're prototyping and don't care about optimization yet

**When NOT to:**
- You need maximum performance (LangChain adds 50-200ms overhead)
- You need to debug at the HTTP request level
- Your use case is simple (raw API is cleaner)

### 2. LlamaIndex — The Data Specialist

**Best for:** Data ingestion, indexing, complex RAG  
**Worst for:** General-purpose agent building

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core import Settings

# Load documents, chunk, embed, index — in 3 lines
documents = SimpleDirectoryReader("./data").load_data()
index = VectorStoreIndex.from_documents(documents)

# Query with RAG built in
query_engine = index.as_query_engine()
response = query_engine.query("What is the return policy?")
```

**When to reach for LlamaIndex:**
- You're building data-heavy RAG (100K+ documents)
- You need advanced indexing (hierarchical, knowledge graphs)
- You want built-in evaluation tools

### 3. Pydantic AI — The Agent Builder

**Best for:** Production agents with type safety  
**Worst for:** Simple chatbots (over-engineered)

```python
from pydantic_ai import Agent, RunContext
from pydantic import BaseModel

class WeatherResult(BaseModel):
    temperature: float
    conditions: str
    city: str

agent = Agent(
    'openai:gpt-4o-mini',
    result_type=WeatherResult,  # 🔑 Type-safe outputs
    system_prompt="Get weather data for any city.",
)

@agent.tool
def get_weather(ctx: RunContext, city: str) -> dict:
    """Get current weather for a city"""
    # Your API call here
    return {"temp": 28, "conditions": "sunny"}

result = agent.run_sync("What's the weather in Mumbai?")
print(f"{result.city}: {result.temperature}°C")  # Type-safe!
```

---

## 🧪 Decision Drill: Choose the Right Tool

**Scenario 1:** Build a FAQ chatbot for a SaaS company. 500 product docs. Need to answer questions accurately.

**What framework/tools would you use?**

<details>
<summary>Senior Answer</summary>
**Start raw** — just use OpenAI embeddings + chromadb + direct API calls.
**Why?** It's simple, you understand every line, and 500 docs doesn't need a framework.
**Add framework later** — only add LangChain when you need chunking strategies or chain composition.
</details>

**Scenario 2:** You need to build a multi-agent system where one agent researches, one writes code, and one reviews the code.

**What framework/tools would you use?**

<details>
<summary>Senior Answer</summary>
**LangGraph** — this is exactly what LangGraph is built for: stateful, multi-agent workflows with conditional routing.
**Alternative:** Pydantic AI if the agents are simpler.
**NOT:** Raw API calls — you'd be writing a graph state machine from scratch.
</details>

**Scenario 3:** Simple classification: given customer emails, categorize into 5 buckets. 10K emails/day.

**What framework/tools would you use?**

<details>
<summary>Senior Answer</summary>
**Just OpenAI API + Pydantic validation.**
No framework needed. A single API call with structured outputs. Adding LangChain here would be over-engineering.
</details>

---

## 📖 The Build Raw Rule (From Phase 0)

**Golden Rule:** For ANY framework, ask: "Can I build this without the framework in under 50 lines?"

If YES → Build raw first, add framework later if the pain justifies it.
If NO → Use the framework (you'd be writing it yourself anyway).

```python
# Examples:
# ❌ Build a RAG system (50+ lines) → LangChain is helpful
# ✅ Classify text (10 lines) → Just use the API
# ❌ Multi-agent system (200+ lines) → LangGraph is valuable
# ✅ Single completion (5 lines) → Raw API call is cleaner
```

---

## 🚦 Gate Check

- [ ] I can explain what each major framework is best for
- [ ] I know when to use LangChain vs LlamaIndex vs raw API
- [ ] I know when to use LangGraph vs Pydantic AI
- [ ] I can apply the Build Raw Rule to any new problem
- [ ] I'm not attached to any single framework — I use the right tool

---

**→ Now do the drills: `Drill-Build-CLI-Chat.md`**
