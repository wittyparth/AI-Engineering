# AI Engineering Learning Paths: Research Summary

> Research from Reddit (r/LocalLLaMA, r/MachineLearning, r/LangChain), Discord communities, and industry analysis on the best way to learn AI engineering in 2025-2026.

## The Consensus: Learn by Building, Not by Watching

The single strongest signal across every community: **build projects, don't collect courses.** The people who get hired as AI Engineers are the ones who can ship, not the ones with the most certificates.

### Recommended Learning Sequence (Community-Verified)

1. **Python fundamentals** — enough to not struggle with syntax while learning AI concepts
2. **LLM APIs** — raw API calls before any framework (feel the boilerplate)
3. **Structured outputs** — Pydantic + function calling (you need this every day)
4. **RAG from scratch** — build it manually before touching LangChain
5. **Evaluation** — learn to measure before optimizing
6. **Agent loops** — build ReAct manually before LangGraph
7. **Production deployment** — FastAPI + Docker + cloud

## Top-Rated Courses (Community-Agreed)

### Beginner
| Course | Why It's Recommended |
|---|---|
| **FastAPI Official Tutorial** | Free, excellent, teaches the API layer you'll use daily |
| **Building Systems with the ChatGPT API** (DeepLearning.AI) | Short (2hr), teaches production API patterns |
| **Prompt Engineering for ChatGPT** (DeepLearning.AI) | Short (1hr), systematic approach to prompting |
| **LangChain for LLM App Development** (DeepLearning.AI) | Good intro, but take AFTER building manually |

### Intermediate
| Course | Why It's Recommended |
|---|---|
| **Building and Evaluating Advanced RAG** (DeepLearning.AI) | Production RAG patterns with evaluation |
| **Weaviate Academy** | Best free resource for vector database concepts |
| **RAG From Scratch** (LangChain YouTube series) | Watch after building manual RAG |
| **Hamel Husain's Field Guide to Evaluating AI Systems** | Essential reading for production eval |

### Advanced
| Course | Why It's Recommended |
|---|---|
| **AI Agents in LangGraph** (DeepLearning.AI) | Best intro to agent state machines |
| **Functions, Tools and Agents with LangChain** (DeepLearning.AI) | Tool use patterns |
| **MCP: Build Rich-Context AI Apps** (DeepLearning.AI) | MCP mastery |
| **Multi AI Agent Systems with CrewAI** (DeepLearning.AI) | Multi-agent patterns |

### Expert
| Course | Why It's Recommended |
|---|---|
| **UC Berkeley: Large Language Model Agents** | Research-level agent design |
| **Full Stack Deep Learning: LLM Bootcamp** | Free recordings, production LLM system design |
| **Hugging Face Agents Course** | Multi-framework agent building |
| **Hugging Face MCP Course** | Free, MCP mastery |

## Books the Community Agrees On

| Book | Author | When to Read |
|---|---|---|
| **AI Engineering** | Chip Huyen | After building a few projects — connects theory to production patterns |
| **Designing Machine Learning Systems** | Chip Huyen | When thinking about system design |
| **Build a Large Language Model (from Scratch)** | Sebastian Raschka | If you want deep transformer understanding |
| **Natural Language Processing with Transformers** | Tunstall, von Werra, Wolf | After basic RAG — Hugging Face ecosystem |

## Red Flags (What Communities Warn Against)

- **"AI Engineer" certification programs** — most are cash grabs
- **Dragging through years of ML theory** before building anything
- **Starting with LangChain** without understanding what it abstracts
- **Watching 50 hours of video** without writing code
