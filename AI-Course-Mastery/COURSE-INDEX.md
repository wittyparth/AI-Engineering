# 📚 AI Engineering Mastery — Course Index

**Total:** 11 Phases → ~90+ topic files → 11 portfolio projects  
**Time:** ~8 months at 2–3 hrs/day  
**Stack:** FastAPI · OpenAI · Anthropic · LiteLLM · LangGraph · Pydantic AI · LlamaIndex · DSPy · Qdrant · Langfuse · Docker · AWS

---

## Phase 0: Engineering Mindset
*Install the mental models of a senior AI engineer before writing code.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 0.1 | `00-Module-Overview.md` | Phase goals, time budget, daily schedule, deliverables |
| 0.2 | `01-Senior-vs-Junior-Mindset.md` | $70K vs $250K thinking patterns, real examples, framework ≠ seniority |
| 0.3 | `02-Production-vs-Academic-Thinking.md` | The 99% trap, benchmark lying, "works on my machine" gap |
| 0.4 | `03-Evaluation-First-Philosophy.md` | If you can't measure it you can't ship it, eval pipeline, cost of NOT evaluating |
| 0.5 | `04-Decision-Framework.md` | When to build raw vs use frameworks, framework maturity model |
| 0.6 | `05-Cost-Awareness-Culture.md` | Token economics, model cascade pattern, cost tracking |
| 0.7 | `06-The-Build-Loop.md` | Problem → Build Raw → Framework → Productionize → Evaluate → Debug |

---

## Phase 1: Foundations — LLM Gateway
*How LLMs work + Build a Multi-Provider LLM Gateway.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 1.1 | `00-Module-Overview.md` | Phase goals, 10-day schedule, build path |
| 1.2 | `01-How-LLMs-Work.md` | Token prediction, training pipeline, what LLMs CAN'T do |
| 1.3 | `02-Tokenizers-Deep-Dive.md` | BPE/WordPiece, token ≠ word, model-specific tokenizers, token budgeting |
| 1.4 | `03-Context-Windows-Attention.md` | Attention mechanism, lost-in-the-middle, positional encoding, context budgeting |
| 1.5 | `04-Model-Families-Tradeoffs.md` | Frontier vs open-source, model router pattern, decision framework |
| 1.6 | `05-Structured-Outputs-Basics.md` | Prompt → function calling → constrained decoding, validation + retry |
| 1.7 | `06-Python-AI-Patterns.md` | Async, retry with exponential backoff, rate limiting, streaming patterns |
| 1.8 | `07-API-Integration-Patterns.md` | Unified client, retry strategies, rate limiting, streaming patterns |
| 1.9 | `08-Streaming-SSE-WebSockets.md` | SSE vs WebSockets, token-level streaming, backpressure, connection management |
| 1.10 | `09-AI-Production-Framework-Landscape.md` | Agent/RAG/Gateway/Serving/Prompt/Observability framework categories |
| 💪 | `Drill-Build-CLI-Chat.md` | Build a CLI chat client that streams responses |
| 💪 | `Drill-Tokenizer-Explorer.md` | Explore tokenization patterns, build cost table |
| 🏗️ | `Project-Cornerstone-Multi-Provider-LLM-Gateway.md` | Multi-provider gateway with LiteLLM, streaming, cost tracking |

---

## Phase 2: Prompt Engineering — Systematic & Scientific
*Engineer prompts like code — with metrics, versioning, and automated optimization.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 2.1 | `00-Module-Overview.md` | Phase goals, 14-day schedule, prompt engineering hierarchy |
| 2.2 | `01-Context-Engineering.md` | Prompt anatomy, why context matters more than instructions |
| 2.3 | `02-Techniques-Deep-Dive.md` | Zero-shot, few-shot, CoT, structured output, when each works |
| 2.4 | `03-System-Prompts-Personas.md` | System prompt design, persona engineering, behavioral guardrails |
| 2.5 | `04-Advanced-Techniques.md` | Self-consistency, ToT, reflexion, generated knowledge prompting |
| 2.6 | `05-Red-Teaming-Antipatterns.md` | 9 prompt attack patterns, failure modes, overfitting, bias |
| 2.7 | `06-Evaluation-AB-Testing.md` | Unit tests, regression suites, A/B testing, statistical significance |
| 2.8 | `07-DSPy-Optimization.md` | Signatures, modules, optimizers (BootstrapFewShot, MIPROv2), assertions |
| 2.9 | `08-Specialized-Prompt-Patterns.md` | Code generation, reasoning, creative writing, analysis prompts |
| 2.10 | `09-Production-Prompt-Management.md` | Template registry, versioning, deployment strategy, rollback |
| 🏗️ | `10-Project-Prompt-Engineering-System.md` | Injection-resistant prompt system with DSPy compilation pipeline |

---

## Phase 3: Embeddings & Vector Databases
*Build a semantic search engine from scratch through production.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 3.1 | `00-Module-Overview.md` | Phase goals, build path, search quality philosophy |
| 3.2 | `01-Embeddings-Deep-Dive.md` | What embeddings are, how they're created, quality dimensions |
| 3.3 | `02-Semantic-Search-From-Scratch.md` | Cosine similarity, Euclidean distance, dot product, brute force → IVF |
| 3.4 | `03-Embedding-Model-Selection.md` | Model landscape, benchmark on YOUR data, the embedding trap |
| 3.5 | `04-ChromaDB.md` | Setup, collections, CRUD, filtering, local development workflow |
| 3.6 | `05-Qdrant-Pinecone.md` | Production vector DBs — HNSW, quantization, filtering, scaling |
| 3.7 | `06-pgvector.md` | PostgreSQL as vector DB, hybrid search, indexing |
| 3.8 | `07-Advanced-Vector-Search.md` | Hybrid search (BM25 + dense), RRF fusion, cold start problem |
| 💪 | `08-Drills-Embeddings.md` | Build vector search from scratch + embedding benchmark |
| 🏗️ | `09-Project-Knowledge-Search-Engine.md` | Production semantic search with hybrid search + Qdrant |

---

## Phase 4: RAG Foundations
*Make LLMs answer from YOUR data — production retrieval-augmented generation.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 4.1 | `00-Module-Overview.md` | Phase goals, 3-implementation approach (raw/LangChain/LlamaIndex) |
| 4.2 | `01-What-Is-RAG.md` | The RAG pipeline, the RAG tax, when to use which framework |
| 4.3 | `02-Chunking-Strategies.md` | Recursive → semantic → document-aware chunking, chunking for different content |
| 4.4 | `03-Retrieval-Deep-Dive.md` | Embedding → similarity search → top-K, ingestion failure modes |
| 4.5 | `04-Context-Integration.md` | Prompt construction, from-scratch implementation, vanilla RAG trap |
| 4.6 | `05-RAG-Failure-Modes.md` | 5 common RAG failures — retrieval miss, bad chunks, hallucination, missing context |
| 4.7 | `06-Evaluating-RAG.md` | RAGAS (faithfulness, relevancy, context precision/recall), LLM-as-judge |
| 4.8 | `07-Production-RAG.md` | Reranking, hybrid search, query transformation, caching, streaming |
| 💪 | `08-Drills-RAG.md` | Build RAG without frameworks + chunking optimization + LangChain RAG |
| 🏗️ | `09-Project-Customer-Support-Bot.md` | Full RAG system with reranking, query rewriting, RAGAS evaluation |

---

## Phase 5: Advanced RAG
*Beyond naive retrieval — HyDE, agentic RAG, GraphRAG, multimodal.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 5.1 | `00-Module-Overview.md` | Phase goals, 5 problems solved (vocab mismatch, multi-hop, etc.) |
| 5.2 | `01-HyDE-Query-Transformation.md` | HyDE, multi-query, query decomposition, strategy comparison |
| 5.3 | `02-Contextual-Retrieval.md` | Context-aware chunking, parent-child retrieval, contextual embedding |
| 5.4 | `03-Reranking-Deep-Dive.md` | Cross-encoder reranking, Cohere Rerank, performance vs second retrieval |
| 5.5 | `04-Agentic-RAG.md` | Agent decides WHEN/HOW to retrieve, agentic RAG loop with reflexion |
| 5.6 | `05-Multimodal-RAG.md` | Multimodal embeddings, image captioning + text RAG, multimodal LLM, cost |
| 5.7 | `06-GraphRAG.md` | Knowledge graph construction, entity extraction, relationship mapping, query patterns |
| 5.8 | `07-Advanced-Evaluation.md` | Multi-hop accuracy, retrieval precision, cache hit rate, faithfulness |
| 💪 | `08-Advanced-Drills.md` | Advanced RAG patterns, agentic retrieval implementation |
| 🏗️ | `09-Project-Advanced-RAG-Engine.md` | Agentic research assistant with HyDE, self-RAG, semantic caching |

---

## Phase 6: AI Agents
*From RAG to autonomous action — ReAct, tools, memory, multi-agent, MCP.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 6.1 | `00-Module-Overview.md` | Phase goals, agent vs RAG comparison, agent maturity model |
| 6.2 | `01-Agent-Foundations.md` | Agent architecture patterns — single, supervisor, orchestrator, swarm |
| 6.3 | `02-ReAct-Loop.md` | ReAct from scratch — observe → think → act, failure modes |
| 6.4 | `03-Tool-Design.md` | Tool schema, description engineering, parameter constraints, validation |
| 6.5 | `04-Memory-Systems.md` | Short-term, long-term (vector store), episodic memory, memory-augmented agents |
| 6.6 | `05-LangGraph-Deep-Dive.md` | State graphs, nodes, edges, checkpointing, human-in-the-loop |
| 6.7 | `06-Multi-Agent-Systems.md` | Communication patterns (direct, blackboard, queue), orchestrator-worker, handoff |
| 6.8 | `07-MCP-Protocol.md` | Model Context Protocol — servers, clients, tools, resources, prompts |
| 6.9 | `08-Agent-Evaluation.md` | Task completion, step-level evaluation, cost/efficiency, robustness testing |
| 🏗️ | `09-Project-Multi-Agent-Research-System.md` | Multi-agent research system with orchestrator, research, analysis, writing, fact-checking agents |

---

## Phase 7: Production Evals & Observability
*Know your AI actually works — continuously, measurably, at scale.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 7.1 | `00-Module-Overview.md` | Phase goals, proactive eval mindset, evals-first philosophy |
| 7.2 | `01-Evals-First-Mindset.md` | Evals before implementation, eval-driven development, the eval pyramid |
| 7.3 | `02-LLM-as-Judge.md` | Judge design, bias mitigation, pairwise scoring, rubric-based eval |
| 7.4 | `03-Metrics-That-Matter.md` | Latency (P50/P95/P99), throughput, error rates, cost, user feedback |
| 7.5 | `04-Evaluation-Datasets.md` | Golden datasets, synthetic data generation, versioning, curation |
| 7.6 | `05-Building-Eval-Pipelines.md` | CI/CD eval gates, regression detection, automated benchmarking |
| 7.7 | `06-Production-Observability.md` | OpenTelemetry, Langfuse tracing, spans, cost tracking per query |
| 7.8 | `07-Regression-Detection.md` | Drift detection, score drift, automated alerting, incident response |
| 💪 | `08-Custom-Eval-Framework.md` | Build your own evaluation framework with LLM-as-judge |
| 🏗️ | `09-Project-Eval-Platform.md` | AI observability dashboard with Langfuse + Grafana + alerting |

---

## Phase 8: Guardrails, Safety & Security
*Make your AI system safe by design — prevent harm, not just detect it.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 8.1 | `00-Module-Overview.md` | Phase goals, eval vs safety mindset, defense in depth |
| 8.2 | `01-Threat-Modeling.md` | Prompt injection (direct/indirect), data poisoning, model inversion/theft, supply chain |
| 8.3 | `02-Prompt-Injection.md` | 9 injection attack types, jailbreak patterns, multi-turn injection |
| 8.4 | `03-Input-Guardrails.md` | Classify and filter harmful inputs before they reach the LLM |
| 8.5 | `04-Output-Guardrails.md` | Filter unsafe outputs, content moderation, topic restrictions |
| 8.6 | `05-PII-Redaction.md` | PII detection, redaction strategies, masking, anonymization |
| 8.7 | `06-Content-Moderation.md` | Safety classification across harm categories, severity scoring |
| 8.8 | `07-Guardrails-Architecture.md` | Multi-layer defense, guardrail router, latency-aware routing |
| 💪 | `08-Testing-Guardrails.md` | Adversarial testing, red teaming, safety evaluation automation |
| 🏗️ | `09-Project-Safe-AI-Gateway.md` | Production guardrails system — input/output filtering, PII redaction, injection defense |

---

## Phase 9: Fine-Tuning & LLMOps
*Stop consuming models — become a CREATOR of them. LoRA, data prep, training, deployment.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 9.1 | `00-Module-Overview.md` | Phase goals, when NOT to fine-tune (most important skill), cost analysis |
| 9.2 | `01-Fine-Tuning-Fundamentals.md` | Fine-tuning vs RAG vs prompting decision matrix, common patterns |
| 9.3 | `02-LoRA-and-PEFT.md` | LoRA/QLoRA mechanics, rank selection, target modules, adapter merging |
| 9.4 | `03-Dataset-Preparation.md` | Chat template vs completion format, data quality checklist, augmentation |
| 9.5 | `04-Training-with-Unsloth-TRL.md` | Unsloth 2x speedup, Hugging Face TRL, SFT/DPO, LoRA configuration |
| 9.6 | `05-Evaluation-and-Comparison.md` | Before/after comparison, domain-specific evals, MT-Bench style evaluation |
| 9.7 | `06-Fine-Tuning-for-Function-Calling.md` | Tool-calling fine-tuning, structured output training data |
| 9.8 | `07-LLMOps-Model-Registry.md` | Model versioning, experiment tracking, evaluation database |
| 💪 | `08-Deployment-of-Fine-Tuned-Models.md` | vLLM serving, quantization (AWQ/GPTQ/GGUF), model merging |
| 🏗️ | `09-Project-Custom-Fine-Tuned-Model.md` | Fine-tune Llama 3.2 8B on a domain with LoRA + Unsloth + eval |

---

## Phase 10: Deployment & Scale
*Ship AI that survives production — infrastructure, caching, cost, latency, monitoring.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 10.1 | `00-Module-Overview.md` | Phase goals, production failure scenarios, cost trap |
| 10.2 | `01-Production-Infrastructure.md` | Docker multi-stage, docker-compose, health checks, graceful shutdown, resource limits |
| 10.3 | `02-Model-Serving-at-Scale.md` | vLLM PagedAttention, continuous batching, tensor parallelism, Llama Stack |
| 10.4 | `03-Caching-with-Redis.md` | Exact match caching, semantic caching, cache-aside pattern, invalidation |
| 10.5 | `04-Cost-Optimization.md` | Model cascades, prompt optimization, batching, cost tracking dashboard |
| 10.6 | `05-Latency-Optimization.md` | Speculative decoding, KV cache optimization, streaming, connection pooling |
| 10.7 | `06-Monitoring-and-Alerting.md` | Prometheus + Grafana, AI-specific metrics, alert thresholds, on-call playbook |
| 10.8 | `07-Multi-Provider-Failover.md` | Fallback routing, health checking, graceful degradation, cost-aware routing |
| 💪 | `08-Security-and-Compliance.md` | API security, auth, encryption, audit logging, compliance requirements |
| 🏗️ | `09-Project-Production-Infrastructure.md` | Deploy Phase 6's multi-agent system — Docker + AWS + CI/CD + monitoring |

---

## Phase 11: Capstone
*Build & launch a real AI product that integrates ALL 10 previous phases.*

| # | File | What You'll Learn |
|---|------|-------------------|
| 11.1 | `00-Capstone-Overview.md` | 4-week plan, 3 project options, deliverables, scoring rubric |
| 11.2 | `01-Project-Options-Details.md` | Deep dives on each option — required components, metrics, architecture, market positioning |

**Capstone Project Options:**
- **Option 1:** AI Customer Support System — multi-agent, RAG, sentiment, handoff
- **Option 2:** AI Code Review Assistant — GitHub App, diff analysis, security/bug/style agents
- **Option 3:** AI Document Intelligence Platform — multi-format ingestion, hybrid search, faithfulness validation

---

## Key to Symbols

| Symbol | Meaning |
|--------|---------|
| 📖 | Concept file (read + take notes) |
| 💪 | Drill (30-60 min focused exercise) |
| 🏗️ | Portfolio Project (ships to GitHub) |

## How to Use This Index

Each phase folder is numbered (`00-` through `10-`). Files within each phase are numbered sequentially. Read in order within each phase.

**Every module overview (`00-Module-Overview.md`) includes:**
- 🎯 Purpose & Goals
- ⏱ Time budget with daily schedule
- 📖 Concept deep-dive
- 💻 Production code examples
- ✅ Good output examples
- ❌ Antipatterns & failure modes
- 🧪 Drills & challenges
- 🚦 Gate check (must pass to advance)
- 📚 Resources
