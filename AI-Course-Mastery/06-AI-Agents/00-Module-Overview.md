# Phase 6: AI Agents — From RAG to Autonomous Action

## 🎯 Phase Purpose

Phase 5 gave you a RAG engine that can FIND information. Phase 6 gives you an AGENT that can ACT on it.

The difference is fundamental:

| RAG Engine (Phase 5) | AI Agent (Phase 6) |
|----------------------|-------------------|
| Answers questions | Accomplishes goals |
| Single-turn retrieval | Multi-step reasoning |
| Passive (user drives) | Active (agent drives) |
| Fixed strategy | Dynamic planning |
| No tool use | Can call APIs, run code, query databases |
| Stateless | Has memory of past actions |
| Returns text | Returns completed tasks |

**What changes in Phase 6:**

Your Phase 5 RAG engine is a tool. An agent that uses that RAG engine is an AI AGENT — it decides WHEN to search, WHAT to search for, whether the answer is sufficient, and what to do NEXT.

An agent that can search the web, run Python code, query a SQL database, AND use your RAG engine is exponentially more capable than any single-purpose system.

**By the end of this phase, you will:**

1. Master the **agent loop** — the fundamental cycle of reasoning, acting, and observing
2. Implement the **ReAct pattern** — interleaving reasoning traces with actions
3. Design and build **tools** — the capabilities your agent can use
4. Build **memory systems** — short-term, long-term, and working memory
5. Build **stateful agents** with LangGraph — the production standard for agent orchestration
6. Design **multi-agent systems** — agents that collaborate, debate, and delegate
7. Integrate with **MCP (Model Context Protocol)** — the emerging standard for agent-tool communication
8. **Evaluate agents** — not just answers, but decisions, efficiency, and safety
9. Ship a **Multi-Agent Research System** — your portfolio project

---

### ⏹ STOP. Answer these before proceeding.

**Scenario 1 — The Multi-Step Research Problem**

*You ask an agent: "Write a report comparing the Q4 2024 revenue growth of OpenAI, Anthropic, and Google's AI divisions. Include recent funding rounds, valuation changes, and market share shifts."*

*A RAG system would search a vector store, find some chunks, and generate an answer. It would miss recent news (not in the index). It would miss funding round data (not in your docs). It would give you a stale, incomplete answer.*

*An AGENT would:*
1. *Search the web for Q4 2024 AI company revenue*
2. *Read 3-5 articles, extract revenue figures*
3. *Search for funding rounds and valuations*
4. *Cross-reference data points for consistency*
5. *Structure the comparison in a table*
6. *Write the report with citations*
7. *Check if any data is still missing → search more if needed*
8. *Deliver the completed report*

*Each step is a TOOL CALL — web search, web fetch, data analysis, report writing. The agent decides the order, when to stop, and when to ask for clarification.*

*How would you design the agent's TOOL LIST? What tools does it need? How does it decide which tool to use at each step? How does it know when the research is COMPLETE vs. when to dig deeper?*

**Scenario 2 — The Tool Execution Trap**

*An agent has a tool: "execute_python(code) → Runs Python code and returns output." A user asks: "Run this script to analyze our sales data: [code that imports os and deletes files]".*

*The agent receives the instruction. Should it:*
1. *Run the code as requested (the user asked for it)*
2. *Refuse because the code looks malicious*
3. *Run the code but with restricted permissions*
4. *Check with the user before running*

*This is the TOOL SAFETY problem. An agent with tools is powerful, but every tool is an attack surface. How do you design tools that are USEFUL but not DANGEROUS? What sandboxing, validation, and confirmation patterns do you need?*

*What if the tool is "send_email(to, subject, body)" and a prompt injection attack tells the agent to email everyone in your contact list? How do you prevent this?*

**Scenario 3 — The Multi-Agent Coordination Problem**

*You have THREE agents:*
- *Research Agent: Searches the web and summarizes findings*
- *Analysis Agent: Takes data and produces insights*
- *Writing Agent: Takes insights and produces a polished document*

*The user asks: "Analyze our competitor's pricing changes this quarter and write a recommendation report."*

*The Research Agent needs to find pricing data → hand off to Analysis Agent → hand off to Writing Agent. But what if:*
- *The Research Agent finds INSUFFICIENT data? Should it search more or hand off anyway?*
- *The Analysis Agent identifies a data CONTRADICTION? Should it ask Research to re-check or proceed with uncertainty?*
- *The Writing Agent thinks the analysis is INCOMPLETE? Should it send back to Analysis or work with what it has?*

*How do you design the COORDINATION PROTOCOL between agents? When does an agent escalate vs. proceed? How do you prevent agents from ping-ponging work back and forth indefinitely?*

---

**By the end of this phase, you will have answers to all three scenarios.**

---

## 🧠 The Agent Spectrum

Not all agents are the same. They exist on a spectrum from "simple tool user" to "autonomous multi-agent system":

```
SIMPLE ───────────────────────────────────────────────────────────► COMPLEX

[Function Call] ─→ [ReAct Agent] ─→ [Stateful Agent] ─→ [Multi-Agent System]
                                                                            
Single tool        Reason + Act       Maintain state       Multiple agents
call, no           loop, multiple     across steps,        collaborate,
reasoning          tool choices       conditional paths    delegate, debate

Cost: $0.001       Cost: $0.01-0.10   Cost: $0.05-0.50    Cost: $0.10-2.00+
Latency: <1s       Latency: 2-10s     Latency: 5-30s      Latency: 30s-5min
```

**The critical insight:** Most production systems should use ReAct agents or stateful agents. Full multi-agent systems are for COMPLEX research and automation tasks where the task genuinely benefits from specialization.

---

## 🔗 Connecting to Phase 5

Your Phase 5 Advanced RAG Engine is a **tool** that an agent can use:

```
Agent:
  Goal: "Research Project Alpha's impact on Q4 revenue"
  
  Step 1: [Think] I need to find internal documents about Project Alpha
  Step 2: [Act] Call RAG_Engine("Project Alpha Q4 2024 revenue impact")
  Step 3: [Observe] Retrieved chunks about Project Alpha's technical achievements but NO revenue data
  Step 4: [Think] Revenue data is likely in the finance documents, not engineering docs
  Step 5: [Act] Call RAG_Engine("Project Alpha financial performance Q4 2024")
  Step 6: [Observe] Found: "Project Alpha contributed $2.3M in cost savings in Q4"
  Step 7: [Think] I have the data. I should also check recent news for public information.
  Step 8: [Act] Call web_search("Project Alpha 2024 results")
  Step 9: [Observe] News article: "Acme Corp's Project Alpha named most innovative AI project"
  Step 10: [Think] I have internal data and external validation. Ready to write report.
  Step 11: [Act] Call write_report(...)
```

Your RAG engine handles retrieval. Your agent handles REASONING — deciding WHAT to retrieve and WHEN.

---

## 📖 Phase Map

```
PHASE 6: AI AGENTS
═══════════════════════════════════════════════════════════════════════

FILE MAP:
┌─────────────────────────────────────────────────────────────────────┐
│ 00: Module Overview          ← YOU ARE HERE                          │
│                                                                      │
│ FOUNDATIONS:                                                         │
│ 01: Agent Foundations        → Agent loop, LLM as core, planning     │
│ 02: ReAct Loop               → Thought/Action/Observation cycles     │
│ 03: Tool Design              → Function calling, schemas, safety     │
│ 04: Memory Systems           → Short/long-term, vector, episodic     │
│                                                                      │
│ ADVANCED ORCHESTRATION:                                              │
│ 05: LangGraph Deep Dive      → State machines, graphs, routing       │
│ 06: Multi-Agent Systems      → Supervisor, debate, swarm patterns    │
│ 07: MCP (Model Context       → Agent-tool communication standard     │
│     Protocol)                                                         │
│                                                                      │
│ EVALUATION & CAPSTONE:                                               │
│ 08: Agent Evaluation         → Decision quality, efficiency, safety  │
│ 09: Project                  → Multi-Agent Research System           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📋 File-by-File Breakdown

| # | File | Core Question | Time |
|---|------|--------------|------|
| **00** | Module Overview | What changes from RAG to agents? | — |
| **01** | Agent Foundations | What IS an agent? How does the core loop work? | 3h |
| **02** | ReAct Loop | How does an agent reason and act in cycles? | 4h |
| **03** | Tool Design | How do I build safe, reliable tools for agents? | 3.5h |
| **04** | Memory Systems | How does an agent remember past steps and learn? | 3h |
| **05** | LangGraph Deep Dive | How do I build stateful agent graphs? | 5h |
| **06** | Multi-Agent Systems | How do multiple agents work together? | 4.5h |
| **07** | MCP Protocol | What is MCP and how do I use it? | 3h |
| **08** | Agent Evaluation | How do I evaluate agent behavior, not just answers? | 3h |
| **09** | Project | Multi-Agent Research System portfolio piece | 12-15h |

**⏱ Phase Budget:** 25-30 days (2-3 hours/day)
**Portfolio Project:** Multi-Agent Research System
**Tech Stack Additions:** LangGraph, Pydantic AI, MCP SDK, Docker (for sandboxed tools)

---

## 🛑 The Three Big Questions — Revisit After Each File

1. **WHEN should an agent act vs. ask the user for clarification?**  
   *Autonomy is powerful but dangerous. How do you decide when the agent should proceed independently vs. confirm with the user?*

2. **How do you PREVENT agents from making things worse?**  
   *A RAG system can give a wrong answer. An agent can SEND EMAILS, DELETE FILES, and MAKE API CALLS with wrong data. What safeguards do you need?*

3. **When do multi-agent systems outperform single agents?**  
   *Two agents are not always better than one. When does specialization help? When does it add unnecessary complexity?*

---

## 🔧 What You'll Install

```bash
# Agent frameworks
pip install langgraph             # State machine agent orchestration
pip install langchain-openai      # LangChain + OpenAI integration  
pip install pydantic-ai           # Pydantic-powered agent framework (preferred)

# Tool execution & sandboxing
pip install duckduckgo-search     # Web search tool
pip install requests              # HTTP tool
pip install beautifulsoup4        # Web scraping
pip install playwright            # Browser automation

# Memory
pip install chromadb              # Vector memory (you already have this)
pip install redis                 # Caching and state storage

# MCP
pip install mcp                   # Model Context Protocol SDK

# Evaluation
pip install deepeval              # Agent evaluation
pip install langfuse              # Agent tracing

# Code execution (sandboxed)
pip install docker                # Docker SDK for sandboxed Python execution
pip install pylint                # Code validation before execution
```

---

## 🚦 Phase Gate

Before moving to Phase 7 (Production Evals & Observability):

- [ ] Can you implement a ReAct agent from scratch?
- [ ] Have you built at least 3 different tool types (search, code, API)?
- [ ] Does your agent maintain memory across steps?
- [ ] Have you built a stateful agent with LangGraph?
- [ ] Have you designed a multi-agent system with at least 2 agents?
- [ ] Is your agent's tool use properly sandboxed and validated?
- [ ] Can you evaluate your agent on decision quality, not just answer quality?
- [ ] **Can you take a failing agent trace, diagnose WHY it failed, and fix it?**

---

> **The three scenarios from the top:**
> 1. **Multi-Step Research** → ReAct agent (File 02) with web search + RAG tools (File 03) and memory (File 04)
> 2. **Tool Execution Trap** → Tool safety patterns (File 03) and validation layers
> 3. **Multi-Agent Coordination** → LangGraph (File 05) + multi-agent patterns (File 06)
>
> **Next:** File 01 — Agent Foundations. What IS an agent?
