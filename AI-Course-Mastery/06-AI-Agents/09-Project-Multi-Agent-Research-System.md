# 09 — Project: Multi-Agent Research System

## 🎯 Project Overview

This is your Phase 6 capstone. You will build a **production-grade multi-agent research system** that decomposes complex questions, searches multiple sources, synthesizes findings, and produces well-cited answers.

This isn't a chatbot that answers from memory. This is an **agent orchestration system** — the kind used by companies building automated research assistants, competitive intelligence platforms, and knowledge synthesis tools.

By the end of this project, you'll have a portfolio piece that demonstrates:

- **Agent architecture**: Supervisor-worker decomposition, ReAct loop orchestration, state management
- **Tool design**: Search, retrieval, analysis, and citation tools with proper safety boundaries
- **Memory integration**: Semantic memory for cross-session learning, working memory for context
- **MCP standardization**: Tools exposed as standardized MCP servers for pluggability
- **Evaluation rigor**: Trajectory evaluation, decision quality metrics, safety guardrails
- **Production engineering**: Streaming, error handling, cost tracking, latency optimization

### Learning Objectives

1. Design and build a multi-agent system from scratch before reaching for frameworks
2. Implement the supervisor-worker pattern with dynamic task decomposition
3. Build a ReAct loop for each worker agent with proper tool boundaries
4. Integrate memory systems for both working context and cross-session learning
5. Expose tools via MCP for pluggable, language-agnostic integration
6. Evaluate the full system across decision quality, efficiency, and safety dimensions
7. Profile and optimize the system for production latency and cost

### 🛑 STOP. Before you start, answer these design questions.

**Question 1 — Decomposition Strategy**

*A user asks: "Compare the approaches to fine-tuning used by OpenAI, Anthropic, and Google in their latest models, and recommend which one I should use for my legal document analysis startup."*

This question requires:
- Understanding 3 different fine-tuning APIs
- Knowledge of legal NLP requirements
- A recommendation based on cost, quality, and compliance

Your supervisor agent needs to decompose this into subtasks. **How do you design the decomposition?** What subtasks do you create? Do you create 3 parallel search tasks (one per provider) + 1 analysis task? Or do you make it iterative — first understand legal NLP needs, then search for providers?

**🤔 The deeper question:** What happens when the decomposition is wrong? What if the search for "OpenAI fine-tuning" returns info about GPT-4o, but the user actually needs info about the fine-tuning API for smaller models? Does your system detect this and re-plan, or does it just return wrong information confidently?

---

**Question 2 — Information Conflict Resolution**

Your research agent finds these results:

```
Source A: "Python 3.14 will be released in October 2025"
Source B: "Python 3.14 release scheduled for October 2026"  
Source C: "Python 3.14 beta expected Q3 2025, stable release TBD"
```

Three sources, three different answers. Your agent needs to synthesize a coherent response.

**How does your analysis agent handle this?** Do you:
- A) Pick the most recent source (Source B) and ignore the others?
- B) Present all three with source credibility annotations?
- C) Try to find a fourth, more authoritative source to break the tie?
- D) Report the conflict and ask the user to decide?

**🤔 The deeper question:** What determines which approach is correct? Is it always the same strategy, or does it depend on the domain? How does your agent know which strategy to use for a given query? How do you evaluate whether the agent made the *right* conflict resolution decision?

---

**Question 3 — The Cost-Quality-Latency Triangle**

You're designing a system where:
- A simple query (e.g., "What is the capital of France?") needs 1 tool call, costs $0.01, takes 2 seconds
- A complex query (e.g., "Analyze the macroeconomic trends affecting cloud computing pricing in 2025-2026") might need 20+ tool calls, cost $0.50+, and take 60+ seconds

Your users expect fast answers. But your system can't know in advance whether a query is simple or complex.

**How do you design for this variance?** Do you always use the same full pipeline (15s minimum latency, high cost)? Do you try to classify queries upfront and route to different pipelines? Do you stream intermediate results so the user sees progress?

**🤔 The deeper question:** How does your evaluation framework account for the fact that some queries *should* cost $0.50 because they're genuinely hard, while others that cost $0.50 represent the agent getting lost in loops? What metric separates "thorough" from "inefficient"?

---

**Write down your answers to these three questions. They will shape every design decision in this project.**

---

## ⏱ Time Budget

| Milestone | Estimated Time |
|-----------|---------------|
| Setup & tool implementation | 3-4 hours |
| Milestone 1: Supervisor & single worker | 6-8 hours |
| Milestone 2: Multi-worker orchestration | 6-8 hours |
| Milestone 3: Memory & cross-session learning | 4-6 hours |
| Milestone 4: MCP integration | 4-6 hours |
| Milestone 5: Evaluation & optimization | 6-8 hours |
| Milestone 6: Production polish & streaming | 4-6 hours |
| Documentation & portfolio prep | 2-3 hours |
| **Total** | **35-50 hours** |

---

## 📋 Project Specification

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        SUPERVISOR AGENT                          │
│  Role: Task decomposition, worker routing, result synthesis      │
│  State: LangGraph StateGraph with checkpointer                   │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │  Search    │  │  Analysis  │  │  Citation  │  │ Quality  │  │
│  │  Worker   │  │  Worker   │  │  Worker   │  │  Checker │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
│         │               │               │               │       │
│         ▼               ▼               ▼               ▼       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    TOOL LAYER (MCP)                      │    │
│  │  search_web │ fetch_page │ extract_text │ calculate     │    │
│  │  query_db  │ search_cache │ verify_source │ format_cite │    │
│  └─────────────────────────────────────────────────────────┘    │
│         │               │               │               │       │
│         ▼               ▼               ▼               ▼       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 INFRASTRUCTURE LAYER                      │    │
│  │  Memory Store │ Trajectory Recorder │ Cost Tracker       │    │
│  │  Cache       │ Rate Limiter        │ Safety Monitor     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Responsibility | Key Concepts Used |
|-----------|---------------|-------------------|
| **Supervisor Agent** | Decompose tasks, route to workers, synthesize results | ReAct (02), Plan-then-execute (01) |
| **Search Worker** | Execute web searches, fetch pages, extract content | Tool design (03), ReAct loop (02) |
| **Analysis Worker** | Analyze and synthesize information from multiple sources | Memory (04), Tool use (03) |
| **Citation Worker** | Verify sources, format citations, check credibility | Tool design (03), Safety (08) |
| **Quality Checker** | Validate final answer for completeness and accuracy | Evaluation (08) |
| **Memory Store** | Cross-session learning, working context | Memory systems (04) |
| **MCP Server** | Standardized tool exposure | MCP protocol (07) |
| **LangGraph State** | State machine for multi-agent orchestration | LangGraph (05) |
| **Evaluation Harness** | Trajectory recording, decision eval, safety checks | Agent evaluation (08) |

### Tool Requirements

Your research system must include at least these tools (each connected through MCP):

| Tool | Input | Output | Worker(s) |
|------|-------|--------|-----------|
| `search_web` | query: string, num_results: int | list of {url, title, snippet} | Search, Supervisor |
| `fetch_page` | url: string | markdown content | Search, Analysis |
| `extract_text` | content: string, pattern: string | extracted information | Analysis |
| `verify_source` | url: string, claim: string | credibility score | Citation |
| `format_citation` | sources: list, style: string | formatted citations | Citation |
| `search_cache` | query: string | cached results or miss | All workers |
| `store_memory` | key: string, value: dict | confirmation | Supervisor |
| `retrieve_memory` | query: string | relevant memories | Supervisor |
| `calculate` | expression: string | numeric result | Analysis, Supervisor |

### Constraints & Requirements

1. **Build from scratch first**: Implement the agent loop yourself before using LangGraph for the orchestration layer
2. **All tools through MCP**: Every tool must be exposed as an MCP server (File 07 pattern)
3. **Trajectory recording**: Every agent run must be recorded for evaluation (File 08 pattern)
4. **Memory persistence**: Cross-session memory must survive restarts (File 04 pattern)
5. **Safety policies**: At minimum, implement rate limiting, tool whitelist, and output content filtering
6. **Cost tracking**: Every run must report token usage and estimated cost
7. **Streaming**: Intermediate results should be streamable (the supervisor's current plan, worker progress, etc.)
8. **Graceful degradation**: If a worker fails, the system should retry or re-route, not crash

---

## Milestone 0: Setup & Tool Implementation

### Step 1: Environment Setup

```bash
# Core dependencies
pip install openai anthropic httpx pydantic

# MCP dependencies
pip install mcp  # or your chosen MCP SDK

# LangGraph (for Milestone 1+)
pip install langgraph

# Evaluation & storage
pip install numpy pandas
```

### Step 2: Implement the Tool Layer

Build all the tools listed above as standalone functions first, then wrap them as MCP servers in Milestone 4.

**Focus on:**
- Proper error handling (network failures, rate limits, invalid inputs)
- Pydantic models for input/output validation
- Timeout handling (web requests must not hang forever)
- Rate limiting (don't hammer search engines)
- Logging (every tool call must be logged for evaluation)

**Example starting point:**

```python
"""
tools/search_web.py

Standalone web search tool. 
Later wrapped as MCP server in Milestone 4.
"""

from pydantic import BaseModel
from typing import Optional
import httpx
import time
import json


class SearchInput(BaseModel):
    query: str
    num_results: int = 5


class SearchResult(BaseModel):
    url: str
    title: str
    snippet: str
    source_credibility: Optional[float] = None


class SearchOutput(BaseModel):
    results: list[SearchResult]
    total_estimated: int
    search_time_ms: int


class SearchTool:
    """Web search tool using your chosen search API."""
    
    def __init__(self, api_key: str, rate_limit_per_minute: int = 30):
        self.api_key = api_key
        self.rate_limit = rate_limit_per_minute
        self._call_times: list[float] = []
    
    async def _check_rate_limit(self) -> None:
        """Ensure we don't exceed rate limits."""
        now = time.time()
        window_start = now - 60
        self._call_times = [t for t in self._call_times if t > window_start]
        if len(self._call_times) >= self.rate_limit:
            raise RuntimeError(f"Rate limit exceeded: {self.rate_limit}/minute")
    
    async def run(self, input_data: SearchInput) -> SearchOutput:
        """Execute the search."""
        await self._check_rate_limit()
        start = time.time()
        
        try:
            # Implementation depends on your search provider
            # (Tavily, SerpAPI, Bing API, or even a local index)
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    "https://api.your-search-provider.com/search",
                    headers={"Authorization": f"Bearer {self.api_key}"},
                    json={"query": input_data.query, "num": input_data.num_results},
                    timeout=15.0,
                )
                response.raise_for_status()
                data = response.json()
            
            self._call_times.append(time.time())
            
            results = [
                SearchResult(
                    url=r.get("url", ""),
                    title=r.get("title", ""),
                    snippet=r.get("snippet", ""),
                )
                for r in data.get("results", [])
            ]
            
            return SearchOutput(
                results=results,
                total_estimated=data.get("total", len(results)),
                search_time_ms=int((time.time() - start) * 1000),
            )
            
        except httpx.TimeoutException:
            raise RuntimeError(f"Search timed out for query: {input_data.query}")
        except httpx.HTTPStatusError as e:
            raise RuntimeError(f"Search API error: {e.response.status_code}")
```

### Step 3: Implement the Trajectory Recorder

Before you build agents, build the recording infrastructure (from File 08). You need this running from day one — you can't add evaluation retroactively.

```python
"""
infra/trajectory_recorder.py

Records every agent decision for evaluation.
Must be the first infrastructure built — data collection starts Day 1.
"""

from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from typing import Any, Optional
import json
import uuid
import os
import asyncio


@dataclass
class TrajectoryRecord:
    """Single atomic record of an agent's step."""
    step_type: str  # "thought" | "tool_call" | "tool_result" | "final"
    agent_name: str  # "supervisor" | "search_worker" | etc.
    content: dict[str, Any]
    token_count: int = 0
    latency_ms: int = 0
    timestamp: str = ""
    
    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now(timezone.utc).isoformat()


class AsyncTrajectoryRecorder:
    """
    Async trajectory recorder that never blocks the agent.
    Buffers in memory and flushes asynchronously.
    """
    
    def __init__(self, output_dir: str = "./trajectories/"):
        self.output_dir = output_dir
        os.makedirs(output_dir, exist_ok=True)
        self._buffer: dict[str, list[TrajectoryRecord]] = {}
        self._session_id: str = ""
    
    def start_session(self, task: str, metadata: Optional[dict] = None) -> str:
        """Start a new trajectory session. Returns session_id."""
        self._session_id = uuid.uuid4().hex[:12]
        self._buffer[self._session_id] = []
        
        # Record session start
        self.record(TrajectoryRecord(
            step_type="session_start",
            agent_name="system",
            content={"task": task, "metadata": metadata or {}},
        ))
        return self._session_id
    
    def record(self, record: TrajectoryRecord) -> None:
        """Add a record to the in-memory buffer."""
        if self._session_id not in self._buffer:
            self._buffer[self._session_id] = []
        self._buffer[self._session_id].append(record)
    
    async def flush(self, session_id: Optional[str] = None) -> None:
        """Write buffered records to disk asynchronously."""
        sid = session_id or self._session_id
        if sid not in self._buffer:
            return
        
        records = self._buffer[sid]
        if not records:
            return
        
        filepath = os.path.join(self.output_dir, f"{sid}.json")
        
        # Write to a temp file first, then atomically rename
        tmp_path = filepath + ".tmp"
        
        def _write():
            with open(tmp_path, "w") as f:
                json.dump({
                    "session_id": sid,
                    "records_count": len(records),
                    "records": [asdict(r) for r in records],
                }, f, indent=2)
            os.replace(tmp_path, filepath)
        
        # Run the blocking I/O in a thread pool
        loop = asyncio.get_event_loop()
        await loop.run_in_executor(None, _write)
        
        # Clear the buffer for this session
        self._buffer[sid] = []
    
    def get_session(self, session_id: str) -> list[dict]:
        """Retrieve a recorded session for evaluation."""
        filepath = os.path.join(self.output_dir, f"{session_id}.json")
        if not os.path.exists(filepath):
            return []
        with open(filepath) as f:
            data = json.load(f)
        return data.get("records", [])

```

---

## Milestone 1: Supervisor Agent with Single Worker

**Goal**: Build a supervisor agent that decomposes simple research questions and routes them to a search worker, then synthesizes the results.

### 🏗️ What to Build

1. A **ReAct loop** (File 02) for the supervisor that handles task decomposition
2. A **search worker** that takes a subtask and runs web searches
3. A **synthesis step** that combines search results into a coherent answer
4. Basic **state management** (your own simple state, not LangGraph yet)

### 🔧 Implementation Strategy

**Build your own state machine first.** Don't reach for LangGraph yet. A simple Python class with states and transitions:

```python
@dataclass
class ResearchState:
    task: str
    subtasks: list[str]
    subtask_results: dict[str, Any]
    current_step: str  # "decompose" | "route" | "execute" | "synthesize" | "complete"
    trajectory: AsyncTrajectoryRecorder
    final_answer: Optional[str] = None
    error: Optional[str] = None
```

**Your supervisor ReAct loop (simplified):**

```
1. RECEIVE: User query → "Compare Python 3.12 and 3.13 features"
2. THINK: "This requires finding features of both versions and comparing them"
3. DECOMPOSE: ["Find Python 3.12 key features", "Find Python 3.13 key features", "Compare features"]
4. ROUTE each subtask to search worker
5. COLLECT results from worker
6. SYNTHESIZE findings into comparison table
7. OUTPUT: Well-structured comparison with citations
```

### ✅ Success Criteria

- Supervisor correctly decomposes a question into at least 2 subtasks
- Search worker successfully executes searches for each subtask
- Supervisor synthesizes results into a coherent answer
- Trajectory recorder captures all steps
- Works end-to-end on 3 test queries

### ❌ Common Pitfalls

- **Over-decomposition**: Breaking "What's the weather in London?" into 10 subtasks. Your supervisor should know when a single search is enough.
- **Under-decomposition**: Trying to answer "What are the macroeconomic implications of AI on Southeast Asian labor markets?" with one search. Your supervisor should break this into manageable pieces.
- **Premature synthesis**: Combining results before all searches complete, leading to incomplete answers.

> **🤔 Question for this milestone:** Your supervisor decides on a decomposition plan BEFORE executing any subtasks. What happens when the first search result reveals that the decomposition was wrong? Should your system re-plan mid-execution, or commit to the original plan and hope the synthesis step catches issues? What's the tradeoff?

---

## Milestone 2: Multi-Worker Orchestration with LangGraph

**Goal**: Upgrade your orchestration to LangGraph (File 05) with multiple workers (search, analysis, citation) and conditional routing.

### 🏗️ What to Build

1. **LangGraph StateGraph** replacing your custom state machine
2. **3 specialized workers**: Search, Analysis, Citation
3. **Conditional routing** based on worker results
4. **Human-in-the-loop** checkpointing for approval before critical actions
5. **Error recovery**: If a worker fails, the supervisor re-routes or retries

### 🔧 Architecture

```python
from langgraph.graph import StateGraph, MessagesState
from typing import TypedDict, Literal


class AgentState(TypedDict):
    task: str
    subtasks: list[dict]  # [{"id": 1, "description": "...", "assigned_to": "search", "status": "pending"}]
    results: dict  # {subtask_id: result}
    worker_outputs: dict  # {worker_name: last_output}
    current_phase: Literal["decompose", "route", "execute", "verify", "synthesize", "complete"]
    errors: list[str]
    iteration_count: int


# Define nodes
def decompose_node(state: AgentState) -> dict:
    """LLM call to decompose task into subtasks."""
    ...

def route_node(state: AgentState) -> dict:
    """Assign each subtask to the right worker."""
    ...

def search_worker_node(state: AgentState) -> dict:
    """Execute search subtasks."""
    ...

def analysis_worker_node(state: AgentState) -> dict:
    """Analyze and synthesize search results."""
    ...

def citation_worker_node(state: AgentState) -> dict:
    """Verify and format sources."""
    ...

def synthesis_node(state: AgentState) -> dict:
    """Combine all worker outputs into final answer."""
    ...

def quality_check_node(state: AgentState) -> dict:
    """Verify final answer quality before output."""
    ...

# Define conditional routing
def after_search_router(state: AgentState) -> str:
    """Route based on what search found."""
    if state["errors"]:
        return "replan"  # Need supervisor to re-evaluate
    if all_search_done(state):
        return "analyze"
    return "wait"  # More search results pending

def after_analysis_router(state: AgentState) -> str:
    """Route based on analysis results."""
    if needs_citations(state):
        return "cite"
    if needs_more_info(state):
        return "research_more"  # Back to search
    return "synthesize"


# Build graph
graph = StateGraph(AgentState)

graph.add_node("decompose", decompose_node)
graph.add_node("route", route_node)
graph.add_node("search_worker", search_worker_node)
graph.add_node("analysis_worker", analysis_worker_node)
graph.add_node("citation_worker", citation_worker_node)
graph.add_node("synthesize", synthesis_node)
graph.add_node("quality_check", quality_check_node)

graph.add_edge("decompose", "route")
graph.add_conditional_edges("route", route_decision_router, {
    "search": "search_worker",
    "analyze": "analysis_worker",  # If subtasks already have data
})
graph.add_conditional_edges("search_worker", after_search_router, ...)
graph.add_conditional_edges("analysis_worker", after_analysis_router, ...)
graph.add_edge("citation_worker", "synthesize")
graph.add_edge("synthesize", "quality_check")

graph.set_entry_point("decompose")

# Add checkpointer for human-in-the-loop
from langgraph.checkpoint import MemorySaver
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)
```

### ✅ Success Criteria

- All 3 workers operational and independently testable
- Conditional routing works: search → analysis → citation → synthesize
- If a search returns no results, the supervisor re-plans (different query, different source)
- Human-in-the-loop pauses before citation worker (critical step — verify sources)
- Trajectory recorded with worker attribution (which worker did what)
- Works on 5 test queries of varying complexity

### ❌ Common Pitfalls

- **Cyclic loops**: Worker fails, re-routes to search, search fails, re-routes to same worker → infinite loop. Implement an `iteration_count` cap.
- **State explosion**: Not cleaning state between iterations, causing ever-growing context windows.
- **Worker confusion**: Sending a subtask to the wrong worker because the routing logic is too simplistic.

> **🤔 Question for this milestone:** In your LangGraph, the quality_check node runs AFTER synthesis. But what if synthesis produces a bad answer that quality_check rejects? Your graph sends it back to "decompose" for re-planning. How many re-plan cycles do you allow before giving up? What signals differentiate "the agent needs one more attempt" from "the agent is fundamentally unable to answer this question"?

---

## Milestone 3: Memory Integration

**Goal**: Add working memory (within a session) and semantic memory (across sessions) to your research system.

### 🏗️ What to Build

1. **Working memory** (File 04): Sliding window of recent context within a research session
2. **Semantic memory** (File 04): Vector-based storage of research findings that persists across sessions
3. **Memory consolidation**: Automatic extraction of key facts from completed research sessions
4. **Memory retrieval**: The supervisor queries memory before searching external sources

### 🔧 Architecture

```
┌────────────────────────────────────────────┐
│           MEMORY INTEGRATION LAYER          │
│                                            │
│  ┌────────────────┐  ┌──────────────────┐ │
│  │  WORKING       │  │  SEMANTIC        │ │
│  │  MEMORY        │  │  MEMORY          │ │
│  │                │  │                  │ │
│  │  - Current     │  │  - Vector store  │ │
│  │    subtask     │  │  - Key facts     │ │
│  │  - Recent      │  │  - Sources       │ │
│  │    results     │  │  - Cross-session │ │
│  │  - Active      │  │  - Conflict      │ │
│  │    context     │  │    resolution    │ │
│  └────────────────┘  └──────────────────┘ │
│                                            │
│  ┌────────────────────────────────────┐    │
│  │  CONSOLIDATION PIPELINE            │    │
│  │  (runs after each completed task)  │    │
│  │  1. Extract key facts from result  │    │
│  │  2. Cross-reference existing mems  │    │
│  │  3. Resolve contradictions         │    │
│  │  4. Store with source attribution  │    │
│  └────────────────────────────────────┘    │
└────────────────────────────────────────────┘
```

**Integration point**: The supervisor agent's ReAct loop gets an additional tool:

```python
# Before searching externally, check memory
async def supervisor_think(state: ResearchState) -> str:
    """Modified ReAct thought step that checks memory first."""
    
    # 1. Check semantic memory for relevant prior findings
    memories = await retrieve_memory(state.task)
    if memories:
        state.add_context("prior_knowledge", memories)
        return "I have prior knowledge about aspects of this query. "
               "Let me check if it's sufficient before searching externally."
    
    # 2. If no memory hit, proceed with full research pipeline
    return "No prior knowledge found. Proceeding with full research."
```

**Memory retrieval in the supervisor's tool set:**

```python
# Added to supervisor's tools:
MEMORY_TOOLS = [
    {
        "name": "retrieve_semantic_memory",
        "description": "Search past research findings for relevant information. "
                       "Call this FIRST before doing external searches.",
        "input_schema": {
            "query": "str - The information need",
            "min_relevance": "float (optional, default 0.7) - Minimum similarity threshold"
        }
    },
    {
        "name": "store_finding",
        "description": "Store a key finding for future cross-session retrieval.",
        "input_schema": {
            "key": "str - A unique identifier for this finding",
            "fact": "str - The finding itself",
            "source": "str - URL or reference",
            "confidence": "float (0.0-1.0) - How confident in this finding"
        }
    }
]
```

### ✅ Success Criteria

- Semantic memory survives restart (persistent storage)
- Supervisor checks memory before external search — and uses it when relevant
- After completing a research task, key facts are automatically extracted and stored
- Repeated queries about the same topic use cached results (cheaper, faster)
- Working memory maintains coherent context across multi-turn conversations

### ❌ Common Pitfalls

- **Memory pollution**: Storing low-quality findings because the worker made a bad search. Only consolidate results that passed quality check.
- **Over-reliance on memory**: The agent stops searching because it "remembers" an answer that's now outdated. Memory must have time-to-live or freshness annotations.
- **Context confusion**: Working memory from a previous, unrelated query leaks into the current context, causing hallucination.
- **Consolidation latency**: The consolidation pipeline runs synchronously, delaying the user's next query. This should be async/background.

> **🤔 Question for this milestone:** Your system has been running for a month. Semantic memory has grown to 10,000+ stored facts. When the supervisor retrieves memory, it gets 50+ potentially relevant results. Most are outdated or low quality. How do you design memory retrieval to surface only the most relevant, recent, and reliable memories? How does retrieval quality degrade as memory grows, and how do you measure that degradation?

---

## Milestone 4: MCP Integration

**Goal**: Wrap all tools as MCP servers (File 07) and connect them to your agents through a standardized protocol.

### 🏗️ What to Build

1. **MCP server** for each tool category (search server, analysis server, memory server, citation server)
2. **MCP client** in your agent that discovers and connects to available servers
3. **Tool discovery** — the agent dynamically discovers what tools are available at runtime
4. **Server registry** — a central registry of where each MCP server is running

### 🔧 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR AGENT                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │             MCP CLIENT                           │   │
│  │  - Discovers tools via list_tools()             │   │
│  │  - Calls tools via call_tool()                  │   │
│  │  - Routes responses back to agent              │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
           │                │                │
           ▼                ▼                ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  SEARCH MCP     │ │  MEMORY MCP     │ │  CITATION MCP   │
│  SERVER          │ │  SERVER          │ │  SERVER          │
│                 │ │                 │ │                 │
│  search_web()   │ │  store_mem()    │ │  verify_src()   │
│  fetch_page()   │ │  retrieve_mem() │ │  format_cite()  │
│  extract_text() │ │  consolidate()  │ │  check_fact()   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

**Example MCP Server** (using the MCP Python SDK):

```python
"""
mcp/servers/search_server.py

MCP server that wraps search tools.
Run as: python search_server.py
Connects to agent via stdio or TCP.
"""

from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio
import mcp.types as types


class SearchMCPServer:
    """MCP server exposing search tools."""
    
    def __init__(self):
        self.server = Server("search-server")
        self.server.tool = self._tool_registry
        self.server.call_tool = self._handle_call
        self._tools = {}
    
    def register_tool(self, name: str, description: str, 
                      input_schema: dict, handler):
        """Register a tool with the MCP server."""
        self._tools[name] = {
            "description": description,
            "input_schema": input_schema,
            "handler": handler,
        }
    
    async def _tool_registry(self) -> list[types.Tool]:
        """MCP discovery endpoint — returns available tools."""
        return [
            types.Tool(
                name=name,
                description=info["description"],
                inputSchema=info["input_schema"],
            )
            for name, info in self._tools.items()
        ]
    
    async def _handle_call(self, name: str, arguments: dict) -> list[types.TextContent]:
        """MCP tool execution endpoint."""
        if name not in self._tools:
            raise ValueError(f"Unknown tool: {name}")
        result = await self._tools[name]["handler"](arguments)
        return [types.TextContent(type="text", text=str(result))]
    
    async def run_stdio(self):
        """Run the server over stdio (for local development)."""
        async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
            await self.server.run(
                read_stream, write_stream,
                InitializationOptions(
                    server_name="search-server",
                    server_version="0.1.0",
                ),
            )


# Example: run the search server
if __name__ == "__main__":
    server = SearchMCPServer()
    
    # Register search tools
    search_tool = SearchTool(api_key=os.getenv("SEARCH_API_KEY"))
    
    server.register_tool(
        name="search_web",
        description="Search the web for information on a query",
        input_schema={
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"},
                "num_results": {"type": "integer", "default": 5},
            },
            "required": ["query"],
        },
        handler=lambda args: search_tool.run(SearchInput(**args)),
    )
    
    # Start server
    import asyncio
    asyncio.run(server.run_stdio())
```

### ✅ Success Criteria

- At least 3 MCP servers running (search, memory, citation)
- Agent discovers all tools via `list_tools` at connection time
- Adding a new tool doesn't require restarting the agent — just registering it on the server
- Each server can be tested independently (standalone run)
- All existing tools from Milestones 1-3 work through MCP without code changes

### ❌ Common Pitfalls

- **MCP over TCP without auth**: If your servers are on different machines, they need authentication. Don't expose search servers to the public internet.
- **Too many connections**: Each MCP server is a separate process. 5 servers = 5 processes. This is fine for development but needs containerization for production.
- **Serialization overhead**: MCP serializes/deserializes every call. For high-frequency tool calls (like a tight ReAct loop), this adds latency. Benchmark before and after MCP integration.

> **🤔 Question for this milestone:** Your MCP servers are running in separate Docker containers. The search server goes down. Your agent discovers this when it tries to call `search_web` and gets a connection refused error. What happens? Does the entire research task fail, or does your system degrade gracefully? How do you design agent-level resilience when infrastructure components fail?

---

## Milestone 5: Evaluation & Optimization

**Goal**: Build a comprehensive evaluation framework for your multi-agent system (File 08), run baselines, identify bottlenecks, and optimize.

### 🏗️ What to Build

1. **Evaluation dataset**: 20+ test queries with gold trajectories and success criteria
2. **Decision quality evaluation**: Per-worker tool selection and parameter accuracy
3. **Efficiency evaluation**: Cost per task, steps per task, loop detection
4. **Safety evaluation**: Tool whitelist, parameter bounds, policy checks
5. **Regression detection**: Compare evaluation results across runs
6. **Optimization**: Profile and fix the top 3 bottlenecks

### 🔧 Evaluation Dataset Format

Create a file `eval_dataset.json` with this structure:

```json
[
  {
    "task_id": "eval_001",
    "task": "What is the capital of France?",
    "difficulty": "easy",
    "expected_trajectory": [
      {"worker": "supervisor", "action": "decompose", "expected_subtasks": 0},
      {"worker": "supervisor", "action": "decide_no_search_needed"},
      {"worker": "supervisor", "action": "answer_from_knowledge"}
    ],
    "success_criteria": ["Paris"],
    "max_cost": 0.01,
    "max_steps": 3,
    "safety_rules": ["no_external_calls_needed"]
  },
  {
    "task_id": "eval_002", 
    "task": "Compare Python 3.12 and 3.13 typing features",
    "difficulty": "medium",
    "expected_trajectory": [
      {"worker": "supervisor", "action": "decompose", "expected_subtasks": 2},
      {"worker": "search_worker", "action": "search_web", "query_contains": "Python 3.12 typing"},
      {"worker": "search_worker", "action": "search_web", "query_contains": "Python 3.13 typing"},
      {"worker": "analysis_worker", "action": "analyze", "input_contains": "typing"},
      {"worker": "synthesize", "action": "combine"}
    ],
    "success_criteria": ["TypeVar", "override", "deprecated"],
    "max_cost": 0.15,
    "max_steps": 12,
    "safety_rules": ["no_excessive_external_calls"]
  },
  {
    "task_id": "eval_003",
    "task": "Analyze the impact of the 2024 US presidential election results on tech stocks",
    "difficulty": "hard",
    "expected_trajectory": [
      {"worker": "supervisor", "action": "decompose", "expected_subtasks": 3},
      {"worker": "search_worker", "query_contains": "2024 election"},
      {"worker": "search_worker", "query_contains": "tech stocks"},
      {"worker": "search_worker", "query_contains": "market reaction"},
      {"worker": "analysis_worker", "action": "cross_reference"},
      {"worker": "citation_worker", "action": "verify"},
      {"worker": "synthesize", "action": "combine"}
    ],
    "success_criteria": ["market", "sector", "policy"],
    "max_cost": 0.50,
    "max_steps": 25,
    "safety_rules": ["no_speculation_without_source", "must_cite_all_claims"]
  }
]
```

### Baseline Measurement

Before optimizing, run 3 trials of each test case and record:

```python
# baseline_run.py
eval_results = run_eval_pipeline(
    trajectory_dir="./trajectories/",
    eval_dataset="./eval_dataset.json",
    num_trials=3,  # Agents are non-deterministic
)

baseline_report = eval_results.generate_report()
print(baseline_report.summary())

# Save baseline for comparison
baseline_report.save("baseline_2025_05_01.json")
```

### Common Optimization Targets

| Bottleneck | Typical Cause | Fix |
|-----------|---------------|-----|
| Too many search calls | Worker searches for each subtask separately, even when one search covers multiple | Cache search results, batch queries |
| Long context windows | Accumulating all search results in working memory without summarization | Summarize intermediate results, keep only essential context |
| Worker loops | Conditional routing sends worker back to same state without progress | Tighten routing logic, add iteration cap, improve re-plan quality |
| Slow synthesis | Supervisor tries to synthesize 50+ search results at once | Cluster results thematically, synthesize in groups, then combine |
| High cost | Using GPT-4 for every worker, even for simple tasks | Route simple tasks to cheaper models, keep GPT-4 for synthesis only |

### ✅ Success Criteria

- Evaluation dataset covers easy, medium, and hard queries
- At least 3 metrics tracked per trajectory (decision quality, efficiency, safety)
- Baseline report captures average cost, latency, and success rate
- At least 2 optimization improvements implemented with measurable before/after
- Regression test: running the same evaluation twice produces consistent results

### ❌ Common Pitfalls

- **Optimizing the wrong thing**: Reducing cost by 50% but cutting success rate from 92% to 75%. Always optimize against a fixed quality floor.
- **Single-trial evaluation**: Agents are non-deterministic. A single trial can be lucky or unlucky. Run at least 3 trials per test case.
- **Survivorship bias**: Only evaluating completed trajectories. Failed/crashed agent runs are the most important to evaluate.

> **🤔 Question for this milestone:** Your evaluation shows the system achieves 94% task success with an average cost of $0.12 per query. You optimize and get 92% success at $0.08 per query — a 33% cost reduction for a 2% quality drop. Is this a good tradeoff? Where's your line? At what success rate does the cost savings become unacceptable? What if the 2% drop is concentrated on the hardest queries (which also represent your most valuable users)?

---

## Milestone 6: Production Polish & Streaming

**Goal**: Make your system production-ready with streaming, graceful degradation, cost tracking, and a simple API.

### 🏗️ What to Build

1. **FastAPI server** wrapping your multi-agent system
2. **Streaming endpoint**: SSE-based streaming that shows agent progress in real-time
3. **Graceful degradation**: Timeout handling, partial results on failure, fallback responses
4. **Cost dashboard**: Per-query cost tracking with alert thresholds
5. **Configuration**: YAML/config-based model selection, tool enable/disable

### 🔧 API Design

```python
# api.py
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import asyncio
import json


app = FastAPI(title="Multi-Agent Research System")


class ResearchRequest(BaseModel):
    query: str
    max_cost: float = 0.50
    timeout_seconds: int = 120
    model_preference: str = "auto"  # "auto" | "cheap" | "quality"


class ResearchResponse(BaseModel):
    task_id: str
    answer: str
    sources: list[dict]
    metrics: dict


@app.post("/research")
async def research(request: ResearchRequest) -> ResearchResponse:
    """Execute a research task with the multi-agent system."""
    try:
        result = await research_system.run(
            task=request.query,
            max_cost=request.max_cost,
            timeout=request.timeout_seconds,
        )
        return ResearchResponse(
            task_id=result.task_id,
            answer=result.answer,
            sources=result.sources,
            metrics=result.metrics,
        )
    except TimeoutError:
        raise HTTPException(408, "Research task timed out")
    except Exception as e:
        raise HTTPException(500, f"Research failed: {str(e)}")


@app.post("/research/stream")
async def research_stream(request: ResearchRequest):
    """Stream research progress via SSE."""
    
    async def event_generator():
        yield f"data: {json.dumps({'type': 'status', 'message': 'Starting research...'})}\n\n"
        
        async for event in research_system.run_streaming(
            task=request.query,
            max_cost=request.max_cost,
        ):
            yield f"data: {json.dumps(event)}\n\n"
            
            if event["type"] == "complete":
                break
    
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        },
    )


@app.get("/health")
async def health():
    """Health check that verifies all MCP servers are connected."""
    statuses = await research_system.check_connections()
    all_ok = all(s["connected"] for s in statuses.values())
    return {
        "status": "healthy" if all_ok else "degraded",
        "servers": statuses,
    }


@app.get("/metrics/cost")
async def cost_metrics():
    """Get cost metrics for recent runs."""
    return await cost_tracker.get_summary(hours=24)
```

### Streaming Events

The streaming endpoint emits events at each stage:

```
data: {"type": "status", "message": "Decomposing research question..."}
data: {"type": "subtask", "id": 1, "description": "Search for X", "worker": "search"}
data: {"type": "worker_start", "worker": "search_worker", "subtask_id": 1}
data: {"type": "tool_call", "tool": "search_web", "params": {"query": "..."}, "worker": "search_worker"}
data: {"type": "tool_result", "tool": "search_web", "result_count": 5}
data: {"type": "worker_complete", "worker": "search_worker", "subtask_id": 1}
data: {"type": "status", "message": "Analyzing results..."}
data: {"type": "status", "message": "Generating answer..."}
data: {"type": "complete", "answer": "...", "sources": [...], "metrics": {...}}
```

### Graceful Degradation Rules

```python
DEGRADATION_RULES = {
    "search_timeout": {
        "action": "retry_with_fallback",
        "fallback": "use_cached_or_prior_knowledge",
        "max_retries": 2,
    },
    "worker_crash": {
        "action": "reroute_to_alternative",
        "fallback": "supervisor handles subtask directly",
        "max_retries": 1,
    },
    "total_timeout": {
        "action": "return_partial_results",
        "message": "Research incomplete due to timeout. Returning available findings.",
    },
    "mcp_server_down": {
        "action": "disable_affected_tools",
        "notify": True,
        "fallback": "continue with remaining tools",
    },
    "cost_exceeded": {
        "action": "complete_current_phase_then_stop",
        "notify": True,
        "message": "Cost limit reached. Returning results from completed phases.",
    },
}
```

### ✅ Success Criteria

- FastAPI server starts and responds to health checks
- Streaming endpoint shows real-time agent progress
- Graceful degradation: killing one MCP server doesn't crash the system
- Cost tracking shows per-query breakdown by worker and model
- System runs for 1 hour without crashing under test load (simulate concurrent queries)

### ❌ Common Pitfalls

- **Streaming with state**: Streaming intermediate results means the user sees partial, potentially misleading information. What if the early results say "Python 3.14 released" but later analysis shows the source was wrong? Do you emit corrections?
- **Concurrent state corruption**: Two concurrent research requests share the same memory store. Request A writes a finding that Request B reads before Request A's quality check passes. Implement session-scoped isolation.
- **Cost tracking as an afterthought**: Don't log costs — compute them. Log token counts per model per call, and compute cost from a configurable rate table. This lets you update prices without re-deploying.

---

## 📦 Deliverables Checklist

Your project is complete when you can check all of these:

### Repository Structure
```
multi-agent-research-system/
├── README.md                      # Architecture decisions, setup, usage
├── requirements.txt
├── .env.example                   # API keys (without real values)
├── pyproject.toml                 # Or setup.py
│
├── agents/
│   ├── __init__.py
│   ├── supervisor.py              # Supervisor agent with planning
│   ├── search_worker.py           # Web search worker
│   ├── analysis_worker.py         # Analysis & synthesis worker
│   ├── citation_worker.py         # Source verification worker
│   └── quality_checker.py         # Final answer validation
│
├── tools/
│   ├── __init__.py
│   ├── search_web.py              # Web search tool (standalone)
│   ├── fetch_page.py              # Page fetching & parsing
│   ├── extract_text.py            # Information extraction
│   ├── verify_source.py           # Source credibility check
│   └── calculate.py               # Arithmetic tool
│
├── memory/
│   ├── __init__.py
│   ├── working_memory.py          # Session-scoped context
│   ├── semantic_memory.py         # Vector-based cross-session store
│   └── consolidation.py           # Background consolidation pipeline
│
├── mcp/
│   ├── __init__.py
│   ├── servers/
│   │   ├── search_server.py       # MCP server: search tools
│   │   ├── memory_server.py       # MCP server: memory tools
│   │   └── citation_server.py     # MCP server: citation tools
│   └── client.py                  # MCP client agent connection
│
├── orchestration/
│   ├── __init__.py
│   ├── state_graph.py             # LangGraph StateGraph definition
│   ├── supervisor_graph.py        # Supervisor state machine
│   └── checkpointer.py            # Human-in-the-loop checkpoint
│
├── evaluation/
│   ├── __init__.py
│   ├── trajectory_recorder.py     # Async trajectory recording
│   ├── decision_eval.py           # Per-step decision metrics
│   ├── task_eval.py               # Task completion metrics
│   ├── safety_eval.py             # Safety & policy checks
│   ├── pipeline.py                # Batch evaluation pipeline
│   └── eval_dataset.json          # 20+ test cases with gold data
│
├── api/
│   ├── __init__.py
│   ├── server.py                  # FastAPI server
│   ├── streaming.py               # SSE event stream
│   └── models.py                  # Request/response models
│
├── config/
│   ├── settings.py                # Pydantic Settings
│   ├── models.yaml                # Model selection & pricing
│   └── policies.yaml              # Safety policies
│
└── tests/
    ├── test_tools.py              # Unit tests for each tool
    ├── test_workers.py            # Worker integration tests
    ├── test_evaluation.py         # Evaluation framework tests
    └── test_end_to_end.py         # Full pipeline smoke tests
```

### Functional Requirements
- [ ] At least 3 workers operational (search, analysis, citation)
- [ ] Supervisor decomposes questions into subtasks correctly for easy/medium queries
- [ ] LangGraph handles conditional routing between workers
- [ ] Semantic memory persists across sessions
- [ ] Supervisor checks memory before external search
- [ ] Memory consolidation runs after completed tasks (async)
- [ ] At least 3 MCP servers running and discoverable
- [ ] Tools work through MCP without direct imports
- [ ] Trajectory recorder captures every agent step
- [ ] Evaluation dataset has 20+ test cases with gold trajectories
- [ ] Evaluation pipeline produces aggregate report
- [ ] FastAPI server with streaming endpoint
- [ ] Graceful degradation: system handles MCP server failure
- [ ] Cost tracking with per-query breakdown

### Quality Requirements
- [ ] Success rate >85% on evaluation dataset (medium difficulty)
- [ ] Average cost per query <$0.15 (or within your budget)
- [ ] Average latency <30s for easy queries, <120s for hard
- [ ] Zero critical safety violations in 100 test runs
- [ ] At least 1 optimization with documented before/after
- [ ] Streaming shows real-time progress (not just final answer)
- [ ] README documents all architecture decisions with tradeoffs

---

## 🏆 Portfolio Presentation

Your README.md should tell this story:

```
# Multi-Agent Research System

## One-Liner
"A production-grade multi-agent research system that decomposes complex questions,
coordinates specialized workers, synthesizes findings, and evaluates its own quality."

## Architecture (with your ASCII diagram)
Show the supervisor-worker pattern, MCP tool layer, memory systems, and evaluation pipeline.

## Key Design Decisions (at least 5)
1. Why supervisor-worker vs. debate pattern?
2. Why LangGraph vs. custom state machine?
3. Why MCP vs. direct tool imports?
4. Why vector-based semantic memory?
5. Why this evaluation approach?

## Performance Metrics
- Success rate: XX% on 20-test evaluation suite
- Average cost: $X.XX per query
- Average latency: XXs (easy), XXs (hard)
- Safety violations: X per 100 runs (target: 0)

## Optimization Journey
- Baseline: X% success, $Y.YY cost
- After optimization 1: X% success, $Y.YY cost
- After optimization 2: X% success, $Y.YY cost
- Key insight from each optimization

## Example Outputs
Show 3 example queries with trajectories, highlighting
the agent's reasoning, tool calls, and final answer.

## What I'd Do Differently
Honest reflection on at least 2 things you'd change
with more time or different constraints.
```

---

## 🚦 Gate Check

Before declaring Phase 6 complete, verify you can:

1. **☐** Explain the supervisor-worker pattern and when it beats single-agent approaches
2. **☐** Implement a ReAct loop from scratch without a framework
3. **☐** Design tools with proper input validation, error handling, and safety boundaries
4. **☐** Distinguish between working memory, episodic memory, and semantic memory — and implement each
5. **☐** Build a LangGraph state machine with conditional routing and human-in-the-loop
6. **☐** Compare multi-agent patterns (supervisor, debate, pipeline, swarm) and pick the right one for a given problem
7. **☐** Implement an MCP server and connect an agent to it via the MCP protocol
8. **☐** Build a trajectory recorder and evaluate agents across decision quality, efficiency, and safety
9. **☐** Ship a complete multi-agent system with API, streaming, and documentation

### 🛑 Final Reflection

Answer these to yourself before moving to Phase 7:

1. **What surprised you most about building multi-agent systems?** Was it harder or easier than you expected? Were the failure modes what you anticipated, or did new ones emerge?

2. **If you had to deploy this system to production serving 10,000 queries/day, what's the one thing you'd prioritize improving right now?** Not the coolest feature — the most critical. How did evaluation guide you to this answer?

3. **Which concept from a previous phase was most essential to understanding Phase 6?** (Embeddings? Prompt engineering? RAG evaluation?) This cross-phase connection is exactly the kind of integration that senior engineers recognize.

4. **What was the most valuable thing you learned from building the ReAct loop yourself before using LangGraph?** If you were to teach this module, what would you tell a junior engineer who asks "Why not just start with LangGraph?"

---

### 📚 Continue Your Learning

**Phase 6 complete.** You've gone from understanding the agent loop to building a production multi-agent system with memory, MCP, and evaluation.

Next: **Phase 7 — Production Evals & Observability**, where you'll apply everything from Phase 6's evaluation module at production scale, build an evaluation platform, and integrate observability into every AI system you build.

**Phase 7 builds directly on Phase 6.** The evaluation harness you built for your research system becomes the foundation for a general-purpose evaluation platform. The trajectory recording you implemented becomes part of production observability. The safety policies become guardrail specifications.

Don't start Phase 7 until you've shipped your research system. A portfolio with a shipped project beats a portfolio with 10 half-finished ones.
