# 🧪 Self-Tests — All Phases

## How To Use Self-Tests

1. **Closed book** — no notes, no files, no AI assistance
2. **Write your answers** in full sentences (typing helps retention)
3. **Score yourself honestly** — 1 point per question where your answer was substantially correct
4. **If you score <7/10** for any phase, re-study that phase before advancing
5. **Re-take tests** during phase-end reviews, milestone review, and final sweep

---

## Phase 0 — Engineering Mindset (10 Questions)

1. What is the primary difference between a junior AI engineer and a senior AI engineer?
2. Explain the difference between production thinking and academic thinking in AI.
3. What is the Evaluation-First Philosophy? Why is it non-negotiable?
4. What are the 5 decision criteria in the Decision Framework?
5. Why is cost awareness an engineering discipline and not a business concern?
6. What is the Build Loop? Walk through its stages.
7. Give an example where academic thinking would lead to a poor production outcome.
8. What's the first question a senior AI engineer asks when starting a new project?
9. How do you measure whether you're improving as an AI engineer?
10. "Ship or sink" — what does this mean in practice? When would you NOT ship?

**Answer Key:** Re-read the file for any question you got wrong.

---

## Phase 1 — Foundations (10 Questions)

1. Explain autoregressive generation in one paragraph (no notes).
2. What is BPE tokenization and why does it create variable-length tokens?
3. How does the attention mechanism work? Draw the diagram from memory.
4. Compare GPT-4, Claude 3, and Gemini — when would you use each?
5. What's the difference between JSON mode and function calling for structured outputs?
6. Describe the retry+fallback pattern for LLM API calls.
7. How does SSE streaming work at the protocol level?
8. When would you use WebSockets instead of SSE for streaming?
9. Name 3 production considerations when building a multi-provider gateway.
10. What is the circuit breaker pattern and why does it matter for LLM APIs?

---

## Phase 2 — Prompt Engineering (10 Questions)

1. What is context engineering and why is it more important than prompt phrasing?
2. Compare zero-shot, few-shot, and chain-of-thought — when does each fail?
3. What makes a good system prompt? Give 3 principles.
4. How do you systematically evaluate a prompt? What metrics matter?
5. What is DSPy and how does it optimize prompts differently than manual tuning?
6. Name 3 common prompt injection patterns and how to defend against each.
7. What's the difference between tree-of-thought and chain-of-thought?
8. How would you version prompts in a production system?
9. When should you NOT use DSPy?
10. What is the red-teaming methodology for prompts?

---

## Phase 3 — Embeddings & Vector DBs (10 Questions)

1. What is an embedding and how does it represent meaning?
2. Walk through the architecture of semantic search from scratch.
3. Compare ChromaDB, Qdrant, and pgvector — pick one for a production app and justify.
4. What is cosine similarity and when would you NOT use it?
5. How does hybrid search work? Why would you need it?
6. What's the difference between IVFFlat and HNSW indexes?
7. How do you choose an embedding model? What tradeoffs exist?
8. What metadata filtering strategies work with vector search?
9. How does chunk size affect search quality?
10. When would pgvector be a bad choice?

---

## Phase 4 — RAG Foundations (10 Questions)

1. What problem does RAG solve? When would you NOT use RAG?
2. Walk through a complete RAG pipeline step by step.
3. Compare 5 chunking strategies — when would you use each?
4. What's the difference between recall@k, MRR, and NDCG for retrieval evaluation?
5. What is the "lost-in-the-middle" problem and how do you mitigate it?
6. Name 5 RAG failure modes and a detection strategy for each.
7. How does RAGAS evaluate RAG quality? Walk through faithfulness vs answer relevance.
8. What caching strategies work for RAG in production?
9. How do you handle "no relevant context found" gracefully?
10. What's the cost breakdown of a single RAG query in production?

---

## Phase 5 — Advanced RAG (10 Questions)

1. How does HyDE work? When would it REDUCE retrieval quality?
2. What problem does contextual retrieval solve? How is it different from better chunking?
3. Explain the bi-encoder → cross-encoder two-stage retrieval architecture.
4. What makes RAG "agentic"? When would you avoid agentic RAG?
5. How does multimodal RAG work? What embedding strategy does it need?
6. What is GraphRAG and what types of questions is it best for?
7. How do you run an ablation study comparing 4 retrieval techniques?
8. What metrics should you track when comparing RAG techniques?
9. When is reranking NOT worth the added latency?
10. What cost overhead do you incur with agentic RAG loops?

---

## Phase 6 — AI Agents (10 Questions)

1. What is an AI agent? Walk through the perceive-think-act loop.
2. How does the ReAct loop work step by step?
3. What makes a good tool schema for LLM consumption? Give 3 principles.
4. Compare 4 memory types in agents — when would you use each?
5. How does LangGraph model agent state differently from a simple loop?
6. Compare supervisor vs swarm vs debate multi-agent architectures.
7. What problem does MCP solve? How is it different from custom tooling?
8. How do you evaluate an agent system? What metrics matter?
9. What are the 3 most common agent failure modes and how do you detect them?
10. When would you NOT build an agent system?

---

## Phase 7 — Production Evals & Observability (10 Questions)

1. What is eval-first development? How does it change the development workflow?
2. How does LLM-as-judge work? What biases exist and how do you mitigate them?
3. Name 5 metrics you'd track for an LLM system and why each matters.
4. What makes a good evaluation dataset? How do you avoid data leakage?
5. Walk through the architecture of an eval pipeline — from trigger to report.
6. What's the difference between observability and monitoring for LLM systems?
7. How do you detect a regression without golden data?
8. What would a custom eval framework's core abstractions be?
9. How do you integrate evals into CI/CD?
10. When is human evaluation better than LLM-as-judge?

---

## Phase 8 — Guardrails (10 Questions)

1. What are the 4 biggest security threats to an LLM application?
2. How does prompt injection work? Name 3 defense strategies.
3. What's the difference between input guardrails and output guardrails?
4. How do you detect and redact PII? Tradeoffs of different methods?
5. What content moderation categories matter for a general-purpose AI app?
6. Walk through guardrail pipeline architecture — from request to response.
7. How do you test guardrails? What metrics matter?
8. What's the difference between fail-closed and fail-open? When would you choose each?
9. How do you handle false positives in guardrails?
10. What's the latency budget for guardrails in a production system?

---

## Phase 9 — Fine-Tuning (10 Questions)

1. When should you fine-tune vs use RAG vs improve prompting?
2. How does LoRA work technically? What do rank and alpha control?
3. What makes a high-quality fine-tuning dataset?
4. Walk through the training process — from data to saved model.
5. How do you evaluate whether fine-tuning actually improved the model?
6. How is fine-tuning for function calling different from general FT?
7. What is a model registry and why do you need it?
8. How do you deploy a fine-tuned model to production?
9. What are the cost implications of fine-tuning and serving a FT model?
10. What are the biggest risks of fine-tuning and how do you mitigate them?

---

## Phase 10 — Deployment & Scale (10 Questions)

1. Draw the architecture of a production AI system from memory.
2. How does vLLM achieve high throughput? What is PagedAttention?
3. What caching strategies work for LLM APIs? When does caching NOT help?
4. What are the top 5 cost drivers in an AI system and how do you optimize each?
5. Where does latency come from? How do you profile and optimize?
6. What 5 metrics would you alert on? What are good SLO targets?
7. How does multi-provider failover work? Active-active vs active-passive?
8. What security threats are unique to AI systems vs traditional web apps?
9. How do you track cost per query?
10. What breaks first at scale? How do you design for scale from day 1?

---

## Phase 11 — Capstone Self-Evaluation

### Mid-Capstone Check (Day 142 — After Core Build)

1. Is my core AI feature working end-to-end?
2. Have I handled error states (API failure, empty results, rate limiting)?
3. Is my code structured in a way I could explain to another engineer?
4. What's the #1 thing I'd change if I started over?

### Pre-Launch Check (Day 152 — Before Deployment)

1. Have I tested with adversarial/edge case inputs?
2. Do I know my system's failure modes?
3. Is my monitoring set up to catch production issues?
4. Do I have a rollback plan?
5. Is my documentation clear enough for another engineer to deploy?

### Post-Launch Reflection

1. What was the hardest technical challenge?
2. What would I do differently next time?
3. What am I most proud of?
4. What skill from the course was most useful?
5. What do I want to learn next?
