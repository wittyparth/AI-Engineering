# Phase 6 — AI Agents (Days 70-83)

**Goal:** Master agent fundamentals, ReAct loop, tool design, memory, LangGraph, multi-agent orchestration, MCP — build a multi-agent research system.
**Files:** 9 concept + 1 project (10 files — many large, up to 1740 lines)
**Total days:** 14 study days + 1 phase-end review

---

### Day 70 — Agent Foundations

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Module overview + Agent Foundations | `00-Module-Overview.md`, `01-Agent-Foundations.md` |
| 🌤️ Afternoon | Code: build a minimal agent from scratch | Implement: perceive → think → act → observe loop (no framework) |
| 🌙 Evening | Flashback | Write: "What is an AI agent and what distinguishes it from a simple LLM call?" |

**Key Concepts:** Agent architecture, perceive-think-act loop, autonomy levels, agent vs chain vs workflow
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Agent classification (simple, reflex, goal-based, utility-based), agent vs LLM call

---

### Day 71 — ReAct Loop (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | ReAct Loop deep dive | `02-ReAct-Loop.md` (concepts + theory) |
| 🌤️ Afternoon | Code: implement ReAct from scratch | Build: Thought → Action → Observation → Thought... loop with tool calling |
| 🌙 Evening | Flashback | Write: "How does the ReAct loop work? Walk through 3 iterations." |

**Key Concepts:** ReAct reasoning, thought-action-observation cycle, tool calling, stopping conditions
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** ReAct vs Plan-and-Execute, ReAct failure modes (loops, stuck, hallucination)

---

### Day 72 — Tool Design

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Tool Design | `03-Tool-Design.md` |
| 🌤️ Afternoon | Code: build 3 custom tools | Web search tool, calculator tool, document store tool — with proper schemas and error handling |
| 🌙 Evening | Flashback | Write: "What makes a good tool schema for an LLM? Give 3 principles." |

**Key Concepts:** Tool schema design, function calling integration, error handling in tools, tool selection strategies
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Tool design anti-patterns, tool description quality impact, tool composition

---

### Day 73 — Memory Systems (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Memory Systems | `04-Memory-Systems.md` (concepts: short-term, long-term, episodic, procedural) |
| 🌤️ Afternoon | Code: implement memory systems | Build: conversation buffer, sliding window, summarization memory, vector memory |
| 🌙 Evening | Flashback | Write: "Compare short-term, long-term, episodic, and procedural memory in agents — give an example of each." |

**Key Concepts:** Memory types, memory retrieval, consolidation strategies, memory vs context window
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Memory type selection, retrieval-augmented memory, memory persistence

---

### Day 74 — LangGraph Deep Dive (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | LangGraph Deep Dive | `05-LangGraph-Deep-Dive.md` (concepts + graph architecture) |
| 🌤️ Afternoon | Code: build a LangGraph agent | State graph, nodes, edges, conditional routing, checkpointing |
| 🌙 Evening | Flashback | Write: "How does LangGraph model agent state differently than a simple loop?" |

**Key Concepts:** State graphs, nodes/edges, conditional routing, persistent state, human-in-the-loop
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** LangGraph vs simple loop, state management patterns, checkpointing

---

### Day 75 — Multi-Agent Systems (Full Day — Large File)

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Multi-Agent Systems | `06-Multi-Agent-Systems.md` (architectures: supervisor, swarm, debate) |
| 🌤️ Afternoon | Code: implement a 2-agent system | Supervisor + worker agent with handoff protocol |
| 🌙 Evening | Flashback | Write: "Compare supervisor, swarm, and debate multi-agent architectures — when would you use each?" |

**Key Concepts:** Multi-agent patterns, agent communication, handoff protocols, shared state, synchronization
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Single-agent vs multi-agent decision, communication overhead, failure propagation

---

### Day 76 — MCP Protocol

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | MCP Protocol | `07-MCP-Protocol.md` |
| 🌤️ Afternoon | Code: build an MCP client + server | Implement a tool provider and consumer via MCP |
| 🌙 Evening | Flashback | Write: "What problem does MCP solve for agents? How is it different from direct tool implementation?" |

**Key Concepts:** Model Context Protocol, tool discovery, standardized tool interfaces, MCP vs custom tooling
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** MCP architecture, when MCP matters, MCP adoption considerations

---

### Day 77 — Agent Evaluation

| Session | Focus | Files |
|---------|-------|-------|
| ☀️ Morning | Agent Evaluation | `08-Agent-Evaluation.md` |
| 🌤️ Afternoon | Code: build agent eval harness | Task completion rate, efficiency metrics, tool use correctness, robustness tests |
| 🌙 Evening | Flashback | Write: "How do you measure whether an agent is actually doing its job well?" |

**Key Concepts:** Agent-specific eval (task completion, efficiency, robustness), trajectory evaluation, tool use accuracy
**Checklist:** ☀️ ☐ 🌤️ ☐ 🌙 ☐ | Log updated ☐
**Revision points:** Agent eval vs RAG eval vs standard eval, trajectory-level evaluation

---

### Day 78-81 — Project: Multi-Agent Research System (4 days)

**All days = 🏗️ Project** — `09-Project-Multi-Agent-Research-System.md`

#### Day 78 — Project: Foundation + Single Agent

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Read project spec + design | Architecture: agent roles, tools, communication protocol |
| 🌤️ Afternoon | Build base research agent | Single agent with web search + summarization tools |
| 🌙 Evening | Flashback + commit | Agent architecture decisions → git commit |

**Checklist:** Spec read ☐ | Base agent working ☐ | GitHub updated ☐

#### Day 79 — Project: Multi-Agent Orchestration

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Add second agent | Research agent + writer agent with handoff |
| 🌤️ Afternoon | Memory + state persistence | Add conversation memory, research state tracking |
| 🌙 Evening | Flashback + commit | Multi-agent coordination decisions → git commit |

**Checklist:** Multi-agent working ☐ | Memory working ☐ | GitHub updated ☐

#### Day 80 — Project: MCP + Tools + Polish

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Add MCP tool integration | At least one tool via MCP protocol |
| 🌤️ Afternoon | Error handling + edge cases | Tool failures, agent loops, timeout handling |
| 🌙 Evening | Flashback + commit | Error handling approach → git commit |

**Checklist:** MCP working ☐ | Error handling ☐ | GitHub updated ☐

#### Day 81 — Project: Eval + Docs + Gate Check

| Session | Focus | Tasks |
|---------|-------|-------|
| ☀️ Morning | Agent evaluation | Run eval: task completion, tool accuracy, efficiency |
| 🌤️ Afternoon | Documentation + Gate Check | 🎯 README with architecture diagram, gate checks |
| 🌙 Evening | Final commit + reflection | What surprised me about building agents? |

**🎯 Gate Check — Can you:**
- [ ] Build an agent with ReAct loop from scratch?
- [ ] Implement custom tools with proper schemas?
- [ ] Build a multi-agent system with handoff?
- [ ] Implement at least 2 memory types?
- [ ] Use LangGraph for state management?
- [ ] Build an MCP tool server?
- [ ] Evaluate agent performance (task completion, efficiency)?
- [ ] Handle agent errors (loops, stuck states, tool failures)?

---

### Day 82-83 — Phase 6 Review

#### Day 82 — Phase-End Review

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Scan + recall | Re-read ALL 10 file headings. Re-type ReAct loop from memory. |
| 🌤️ Afternoon | Mind map + weak spots | ✍️ Mind map Phase 6. Re-read weak sections. |
| 🌙 Evening | 🧪 Self-test | Quiz yourself. |

#### Day 83 — Agent Skills Practice Day

| Session | Focus | Activities |
|---------|-------|------------|
| ☀️ Morning | Build from scratch | Build a REACT agent from memory (no looking at notes for the core loop) |
| 🌤️ Afternoon | Debug challenge | Take your working agent and intentionally break it, then fix |
| 🌙 Evening | Reflection | What do I still not feel confident about? |

**🧪 Self-Test Questions:**
1. What is an AI agent? Walk through the perceive-think-act loop
2. How does the ReAct loop work step by step?
3. What makes a good tool schema for LLM consumption?
4. Compare 4 memory types in agents — when would you use each?
5. How does LangGraph model agent state differently?
6. Compare supervisor vs swarm vs debate multi-agent architectures
7. What problem does MCP solve?
8. How do you evaluate an agent system?
9. What are the 3 most common agent failure modes and how do you detect them?
10. When would you NOT build an agent system?

**Self-Assessment:**
- Total score (out of 10): ________
- Weakest area: ________
