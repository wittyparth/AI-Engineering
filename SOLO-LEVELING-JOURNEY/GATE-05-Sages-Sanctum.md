# ⛩️ GATE 05 — Sage's Sanctum (Advanced RAG)

**Rank Requirement:** C+
**Theme:** Production RAG patterns — routing, indexing strategies, agents, and multi-modal retrieval
**Total Days:** 7 Dungeons + 1 Boss Battle
**Core Skills:** Advanced indexing, RAG routing, agentic RAG, multi-modal RAG, chat memory

---

## Dungeon 1 — Indexing Strategies

**Day 1 | Concept:** How you structure your index determines what kinds of queries work. One-size-fits-all indexing is a lie.

**Indexing Strategies:**
| Strategy | Structure | Best For |
|----------|-----------|----------|
| Flat | One index, all chunks | Simple Q&A over homogeneous docs |
| Hierarchical | Summary → Chunks → Sentences | Deep reasoning, multi-step |
| Metadata-filtered | Index per category tag | Filtered search ("only PDFs from 2024") |
| Multi-representation | Same doc embedded N ways | Different query types need different angles |
| Temporal | Time-bucketed indices | News, evolving docs, versioned content |

**🗡️ Dungeon Mastery:** Take a collection of documents about Python, FastAPI, and Docker. Build a metadata-filtered index with "topic" as the filter field. Query for only FastAPI content. Verify filtering works.

**💻 Code — Hierarchical Index:**
```python
class HierarchicalIndex:
    """Two-level index: summaries at top, chunks at bottom."""

    def __init__(self, vector_db, llm):
        self.db = vector_db
        self.llm = llm

    def index_document(self, doc_id: str, sections: list[dict]):
        """sections: [{heading, text, subsections}]"""
        # Level 1: Embed section summaries
        summaries = []
        summary_ids = []
        for i, section in enumerate(sections):
            summary = self._summarize(section["text"])
            summaries.append(summary)
            summary_ids.append(f"{doc_id}:summary:{i}")

        # Level 2: Embed individual chunks
        chunks = []
        chunk_ids = []
        for i, section in enumerate(sections):
            chunk_texts = self._chunk_text(section["text"])
            for j, text in enumerate(chunk_texts):
                chunks.append(text)
                chunk_ids.append(f"{doc_id}:chunk:{i}:{j}")

        # Store summaries (top level)
        self.db.add(documents=summaries, ids=summary_ids,
                    metadatas=[{"level": "summary", "doc_id": doc_id}] * len(summaries))

        # Store chunks (bottom level)
        self.db.add(documents=chunks, ids=chunk_ids,
                    metadatas=[{"level": "chunk", "doc_id": doc_id}] * len(chunks))

    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        # First, find relevant summaries
        summary_results = self.db.query(
            query_texts=[query],
            n_results=top_k,
            where={"level": "summary"},
        )

        # Then drill into chunks from those docs
        doc_ids = [m["doc_id"] for m in summary_results["metadatas"][0]]
        chunk_results = self.db.query(
            query_texts=[query],
            n_results=top_k * 2,
            where={"$and": [{"level": "chunk"}, {"doc_id": {"$in": doc_ids}}]},
        )
        return chunk_results['documents'][0][:top_k]

    def _summarize(self, text: str) -> str:
        prompt = f"Summarize this in 2-3 sentences:\n{text[:2000]}"
        return self.llm.generate(prompt)

    def _chunk_text(self, text: str, size: int = 500, overlap: int = 25):
        words = text.split()
        chunks = []
        for i in range(0, len(words), size - overlap):
            chunk = " ".join(words[i:i + size])
            chunks.append(chunk)
        return chunks
```

**👹 Boss Lesson:** One index to rule them all is a fantasy. Match your indexing strategy to your query patterns.

---

## Dungeon 2 — RAG Routing

**Day 2 | Concept:** Not all queries should go through RAG. Route queries to the right handler: summarization, direct LLM, RAG, or tool call.

**Router Logic:**
```python
if query_needs_facts(query):
    → RAG (retrieve + generate)
elif query_needs_summary(query):
    → MapReduce (summarize all docs)
elif query_is_simple(query):
    → Direct LLM (no retrieval)
elif query_needs_computation(query):
    → Tool call (calculator, code interpreter)
```

**🗡️ Dungeon Mastery:** Build a query router that classifies 10 test queries into one of 4 handlers. Measure classification accuracy.

**💻 Code — Query Router:**
```python
from enum import Enum

class RouteType(Enum):
    RAG = "rag"
    SUMMARY = "summary"
    DIRECT = "direct"
    TOOL = "tool"

class QueryRouter:
    def __init__(self, llm=None):
        self.llm = llm

    def classify(self, query: str) -> RouteType:
        """Classify query into route type."""
        prompt = f"""Classify this user query into one of:

- rag: Needs factual information from documents (e.g., "What does section 3 say about X?")
- summary: Needs overview of many documents (e.g., "Summarize the key points")
- direct: Simple chat or opinion (e.g., "Hello", "What do you think about X?")
- tool: Needs computation or real-time data (e.g., "Calculate 2+2", "What's the weather?")

Query: {query}
Classification (one word):"""

        result = self.llm.generate(prompt).strip().lower()

        for route in RouteType:
            if route.value in result:
                return route
        return RouteType.RAG  # Default to RAG

class RouterPipeline:
    def __init__(self, router: QueryRouter, rag_pipeline, summarizer, direct_llm, tool_handler):
        self.router = router
        self.handlers = {
            RouteType.RAG: rag_pipeline.query,
            RouteType.SUMMARY: summarizer.summarize,
            RouteType.DIRECT: direct_llm.chat,
            RouteType.TOOL: tool_handler.execute,
        }

    def route_and_execute(self, query: str) -> dict:
        route = self.router.classify(query)
        handler = self.handlers[route]
        result = handler(query)
        return {"route": route.value, "result": result}
```

**👹 Boss Lesson:** RAG is not a default. It's a choice. Route wisely — retrieval overhead costs latency and tokens.

---

## Dungeon 3 — Self-RAG & Corrective RAG (CRAG)

**Day 3 | Concept:** Advanced RAG patterns where the LLM checks its own retrieval quality and decides whether to retrieve, refine, or regenerate.

**Self-RAG Flow:**
1. LLM decides: should I retrieve for this query?
2. If yes → retrieve → LLM evaluates if each chunk is relevant
3. Generate answer from relevant chunks only
4. LLM evaluates if the answer is supported by the chunks

**CRAG Flow:**
1. Retrieve
2. Evaluate relevance: Is any chunk relevant?
3. If YES → Generate with relevant chunks
4. If NO → Try query rewrite + re-retrieve, or fall back to web search
5. If PARTIALLY → Keep relevant, discard irrelevant, generate

**🗡️ Dungeon Mastery:** Implement CRAG. Give it a query where no chunks are relevant. It should fall back to web search or gracefully say "I don't know."

**💻 Code — Self-RAG:**
```python
class SelfRAG:
    def __init__(self, retriever, llm):
        self.retriever = retriever
        self.llm = llm

    def _should_retrieve(self, query: str) -> bool:
        prompt = f"""Does answering this question require looking up factual information?
Reply YES or NO only.

Question: {query}"""
        return self.llm.generate(prompt).strip().upper() == "YES"

    def _is_relevant(self, chunk: str, query: str) -> bool:
        prompt = f"""Is this chunk relevant to answering the question?
Reply RELEVANT or NOT_RELEVANT only.

Chunk: {chunk[:500]}
Question: {query}"""
        return "RELEVANT" in self.llm.generate(prompt).upper()

    def _is_supported(self, answer: str, chunks: list[str]) -> bool:
        context = "\n".join(chunks)
        prompt = f"""Is every claim in the answer supported by the context?
Reply SUPPORTED or UNSUPPORTED only.

Context: {context[:2000]}
Answer: {answer}"""
        return "SUPPORTED" in self.llm.generate(prompt).upper()

    def query(self, query: str) -> dict:
        if not self._should_retrieve(query):
            return {"mode": "direct", "answer": self.llm.generate(query)}

        # Retrieve and filter
        chunks = self.retriever.retrieve(query)
        relevant_chunks = [c for c in chunks if self._is_relevant(c, query)]

        if not relevant_chunks:
            return {"mode": "no_context", "answer": "I don't have information about that."}

        # Generate
        context = "\n".join(relevant_chunks)
        answer = self.llm.generate(f"Context:\n{context}\n\nQ: {query}\nA:")

        # Self-check
        supported = self._is_supported(answer, relevant_chunks)
        return {
            "mode": "rag",
            "answer": answer,
            "chunks_used": len(relevant_chunks),
            "supported": supported,
        }
```

**👹 Boss Lesson:** Self-RAG is RAG with a quality control department. Never blindly trust retriever output — validate every step.

---

## Dungeon 4 — Agentic RAG

**Day 4 | Concept:** Give the RAG system agency. The agent decides: search here, then search there, then synthesize across multiple sources.

**Agentic RAG Tools:**
```
1. search_docs(query) → Retrieve from vector DB
2. search_web(query) → Search the internet
3. search_code(query) → Search codebase
4. search_news(query) → Recent news
5. calculate(expression) → Math
6. extract(query, text) → Extract structured data
```

**🗡️ Dungeon Mastery:** Build an agent with 3 tools (search docs, search code, calculate). Give it a query that requires using all 3 tools sequentially. Trace the tool calls.

**💻 Code — Agentic RAG:**
```python
import json

class AgenticRAG:
    def __init__(self, llm, tools: dict):
        self.llm = llm
        self.tools = tools  # {"tool_name": callable}
        self.max_steps = 10

    def query(self, user_input: str) -> str:
        messages = [
            {"role": "system", "content": self._system_prompt()},
            {"role": "user", "content": user_input},
        ]

        for step in range(self.max_steps):
            response = self.llm.chat(messages)
            messages.append({"role": "assistant", "content": response})

            # Check if done (no tool call)
            if "[DONE]" in response:
                return response.replace("[DONE]", "").strip()

            # Parse tool call
            tool_call = self._parse_tool_call(response)
            if not tool_call:
                continue

            # Execute tool
            tool_name = tool_call["tool"]
            args = tool_call["args"]
            if tool_name in self.tools:
                result = self.tools[tool_name](**args)
                messages.append({
                    "role": "tool",
                    "content": json.dumps({"result": result, "tool": tool_name}),
                })

        return "Max steps reached. Here's what I found so far."

    def _system_prompt(self) -> str:
        return """You are an AI assistant with tools. Available tools:

1. search_docs(query: str) — Search documentation
2. search_code(query: str) — Search codebase  
3. calculate(expr: str) — Evaluate math expression

To use a tool, respond with:
TOOL: tool_name
ARGS: {"param": "value"}

When done, respond with:
[DONE] Final answer here."""

    def _parse_tool_call(self, text: str) -> dict | None:
        lines = text.strip().split("\n")
        tool_line = None
        args_line = None
        for i, line in enumerate(lines):
            if line.startswith("TOOL:"):
                tool_line = line.replace("TOOL:", "").strip()
                if i + 1 < len(lines) and lines[i+1].startswith("ARGS:"):
                    args_str = lines[i+1].replace("ARGS:", "").strip()
                    try:
                        args_line = json.loads(args_str)
                    except json.JSONDecodeError:
                        args_line = {}
        if tool_line:
            return {"tool": tool_line, "args": args_line or {}}
        return None
```

**👹 Boss Lesson:** Agentic RAG is the most powerful pattern but also the most dangerous. More agency = more capability + more failure modes.

---

## Dungeon 5 — Multi-Modal RAG

**Day 5 | Concept:** Text-only RAG misses images, tables, charts, and diagrams. Multi-modal RAG indexes and retrieves across all modalities.

**Multi-Modal Approaches:**
| Approach | How | Best For |
|----------|-----|----------|
| Caption-based | Generate text captions, search those | Simple, works with text-only DB |
| Multi-modal embeddings | CLIP embeddings for both text and images | True cross-modal search |
| Image-caption hybrid | Store both image and caption separately | Trade-off approach |
| Vision-Language RAG | GPT-4V/LLaVA reads retrieved images | Best quality, most expensive |

**🗡️ Dungeon Mastery:** Take a PDF with text and images. Index the text as chunks and the images as CLIP embeddings (or text captions). Query for something described only in the image.

**💻 Code — Caption-Based Multi-Modal RAG:**
```python
class MultiModalRAG:
    def __init__(self, vector_db, caption_model, embedder):
        self.db = vector_db
        self.caption_model = caption_model
        self.embedder = embedder

    def index_image(self, image_path: str, image_id: str):
        """Generate caption + embed image for retrieval."""
        # Generate text caption
        caption = self._generate_caption(image_path)

        # Store the caption for text-based retrieval
        self.db.add(
            documents=[f"[IMAGE: {image_id}] {caption}"],
            ids=[f"img_{image_id}"],
            metadatas=[{"type": "image", "path": image_path, "caption": caption}],
        )

    def index_text(self, text: str, text_id: str):
        self.db.add(
            documents=[text],
            ids=[f"text_{text_id}"],
            metadatas=[{"type": "text"}],
        )

    def query(self, text_query: str, top_k: int = 3) -> list[dict]:
        results = self.db.query(query_texts=[text_query], n_results=top_k)
        return [
            {"content": doc, "type": meta.get("type"), "path": meta.get("path")}
            for doc, meta in zip(results['documents'][0], results['metadatas'][0])
        ]

    def _generate_caption(self, image_path: str) -> str:
        # Use BLIP or similar captioning model
        prompt = f"Describe this image in 1-2 sentences"
        # In production: use a vision model
        return f"[Auto-generated caption for {image_path}]"
```

**👹 Boss Lesson:** Multi-modal RAG is where the field is heading. If your RAG system only sees text, it's blind to half the information.

---

## Dungeon 6 — Chat Memory & RAG

**Day 6 | Concept:** Single-turn RAG is easy. Multi-turn RAG requires memory — you need to handle follow-ups, clarifications, and conversation context.

**Memory Strategies:**
| Strategy | How | Quality |
|----------|-----|---------|
| Concat last N | Inject last N Q&A pairs verbatim | Simple, works |
| Summarize history | LLM summarizes conversation so far | Good for long conversations |
| Hybrid | Recent turns verbatim + older summarized | Best for production |
| Semantic memory | Retrieve relevant past exchanges | When you need cross-session |

**💻 Code — RAG with Memory:**
```python
from collections import deque

class ConversationalRAG:
    def __init__(self, rag_pipeline, llm, max_history: int = 5):
        self.rag = rag_pipeline
        self.llm = llm
        self.history = deque(maxlen=max_history)

    def add_turn(self, user: str, assistant: str):
        self.history.append({"user": user, "assistant": assistant})

    def _build_context(self, query: str) -> str:
        # Recent history verbatim
        history_text = ""
        for turn in self.history:
            history_text += f"User: {turn['user']}\nAssistant: {turn['assistant']}\n"

        # Retrieve relevant docs
        docs = self.rag.retrieve(query)
        docs_text = "\n".join(docs)

        return f"""Conversation History:
{history_text}

Relevant Documents:
{docs_text}

Current Question: {query}

Answer (consider history + documents):"""

    def query(self, user_input: str) -> str:
        context = self._build_context(user_input)
        response = self.llm.generate(context)
        self.add_turn(user_input, response)
        return response


# Contextual compression — compress retrieved chunks with conversation context:
class ContextualCompressor:
    def compress(self, query: str, chunks: list[str], llm) -> list[str]:
        compressed = []
        for chunk in chunks:
            prompt = f"""Given the conversation context, extract only the parts of this chunk that are relevant.

Question: {query}
Chunk: {chunk}

Relevant parts:"""
            relevant = llm.generate(prompt)
            if relevant.strip():
                compressed.append(relevant)
        return compressed or chunks  # Fallback to original
```

**🗡️ Dungeon Mastery:** Have a 3-turn conversation with a RAG system. First turn: "What is RAG?" Second: "How is it different from fine-tuning?" Third: "Can I use both?". Verify the third answer references both prior answers.

**👹 Boss Lesson:** Stateless RAG is fine for Q&A. Conversational RAG is what users actually want.

---

## Dungeon 7 — Production RAG Architecture

**Day 7 | Concept:** Move from notebook RAG to production RAG with monitoring, caching, rate limiting, and A/B testing.

**Production Architecture:**
```
User → Rate Limiter → Cache Check → Query Router → Retriever → Reranker → Generator → Guard → User

                                          ↓
                                    Monitor (latency, relevancy, faithfulness)
```

**Production Concerns:**
- **Caching:** Cache frequent queries (TTL-based invalidate)
- **Monitoring:** Track retrieval latency, generation latency, token usage
- **Guardrails:** Block toxic content, PII leakage
- **A/B Testing:** Compare chunk sizes, embedding models, reranking strategies
- **Feedback Loop:** User thumbs up/down → retrain reranker

**💻 Code — Production RAG Config:**
```python
from dataclasses import dataclass
from typing import Optional
import time
import hashlib

@dataclass
class RAGConfig:
    # Ingestion
    chunk_size: int = 500
    chunk_overlap: int = 50
    embedding_model: str = "all-MiniLM-L6-v2"

    # Retrieval
    top_k_retrieve: int = 20
    top_k_rerank: int = 3
    reranker_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"
    use_hybrid_search: bool = True
    dense_weight: float = 0.5

    # Generation
    model: str = "gpt-4o-mini"
    temperature: float = 0.1
    max_tokens: int = 1024
    citation_style: str = "numbered"

    # Production
    cache_ttl_seconds: int = 3600
    rate_limit_per_minute: int = 60
    enable_monitoring: bool = True
    fallback_behavior: str = "answer_without_context"


class ProductionRAG:
    def __init__(self, config: RAGConfig):
        self.config = config
        self.cache = {}
        self.metrics = {"latencies": [], "queries": 0}

    def query(self, query: str) -> dict:
        start = time.time()
        self.metrics["queries"] += 1

        # 1. Cache check
        cache_key = hashlib.md5(query.encode()).hexdigest()
        if cache_key in self.cache:
            entry = self.cache[cache_key]
            if time.time() - entry["timestamp"] < self.config.cache_ttl_seconds:
                return entry["result"]

        # 2. Query understanding
        # 3. Retrieve
        # 4. Rerank
        # 5. Generate
        # (filled in with actual pipeline calls)

        latency = time.time() - start
        self.metrics["latencies"].append(latency)

        result = {"answer": "...", "latency_ms": round(latency * 1000)}
        self.cache[cache_key] = {"result": result, "timestamp": time.time()}
        return result
```

**🗡️ Dungeon Mastery:** Add a simple monitoring endpoint to your RAG system. Track: queries per minute, average latency, cache hit rate, and top 5 frequent queries.

**👹 Boss Lesson:** Notebook RAG works on 10 docs. Production RAG works on 10M docs at 100ms latency. The gap is architecture.

---

## ⚔️ BOSS BATTLE: Advanced RAG Engine

**Objective:** Build an advanced RAG engine that combines 3+ techniques from this gate:
1. Must use a **smart index** (hierarchical or multi-representation)
2. Must use a **query router** (classify before handling)
3. Must use **self-RAG or CRAG** (self-evaluate retrieval quality)
4. Must handle **conversation memory** (at least 3 turns)
5. Must include **monitoring** (latency, cache hits, query log)

**🗡️ Victory Conditions:**
- Routes queries correctly across 3+ handlers
- Self-evaluates and rejects irrelevant chunks
- Maintains conversation context across turns
- Logs performance metrics
- Handles failure gracefully (no "I don't have info" when fallback exists)

**🏆 Rewards:**
- +500 XP
- C+ Rank → B- Rank
- Skill Unlock: **RAG Mastery** — Advanced RAG techniques cost 50% less mental effort
- Title: **The Sage**

---

*"A wise RAG system knows what it doesn't know — and knows what to do about it."*
