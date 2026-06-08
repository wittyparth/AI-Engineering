# 04 — Memory Systems: How Agents Remember

## 🎯 Purpose & Goals

### 🛑 STOP. Before reading about memory, think about these problems.

Your ReAct agent from File 02 works well for single queries. But now give it a multi-turn task:

> *User Session, Turn 1: "Research the top 3 AI chip companies"*
> *Agent searches, analyzes, delivers a report on NVIDIA, AMD, and Intel.*
>
> *User Session, Turn 2: "Which one has the best AI training performance?"*
> *Agent searches again...*

**🤔 Problem:** The agent already FOUND this information in Turn 1. But it has no memory of Turn 1. It starts from scratch. It searches the same things, pays the same API costs, and wastes time.

---

### 🤔 Scenario 1: The Conversation That Doesn't Learn

A user has a 30-minute conversation with your agent:

```
Turn 1: "What's our Q3 revenue?"
  → Agent searches financial DB, finds $12.4M

Turn 15: "How does Q3 compare to Q2?"
  → Agent searches the SAME financial DB again
  → It already found Q3 revenue! But it doesn't remember.

Turn 30: "Show me a trend of our quarterly revenue"
  → Agent searches AGAIN
  → It has all this data in past results, but can't access it
```

**Your challenge before reading on:**

The agent's conversation history grows with every turn. Each turn adds:
- The user's message
- The agent's thoughts and reasoning
- Multiple tool calls and their results
- The final answer

After 30 turns, this history might be 15,000+ tokens. The LLM's attention degrades. Cost increases. And the agent still can't efficiently ACCESS what it learned 20 turns ago.

**Design a solution.** Think about:
- What if the agent periodically summarizes old conversation turns?
- What if the agent stores key facts in a searchable format?
- What information should be kept vs. discarded?
- How does the agent DECIDE what to remember?

---

### 🤔 Scenario 2: The Forgetting Agent

Your agent helps a user with a long-running project:

```
Week 1: User: "I'm starting a project to build a chatbot. What's the best LLM?"
        Agent: "GPT-4o is best for chatbots due to function calling quality."
        User: "Great, I'll use GPT-4o."

Week 2: User: "I need to deploy the chatbot. What's the best hosting?"
        Agent: "Here are options..." ← Has no memory of Week 1!

Week 3: User: "Remember my chatbot project? I need to add memory to it."
        Agent: "What chatbot project?" ← COMPLETELY FORGOTTEN!
```

**The problem:** The agent treats every session as a blank slate. It has no persistent memory between conversations.

**Your challenge:**
1. What information from Week 1 should the agent carry to Week 2?
2. How does the agent know what's important to remember vs. what's ephemeral?
3. How does the agent RETRIEVE the right memory when it's needed?
4. What if the user's project CHANGES direction? Should old memories be updated or discarded?

> **Real production incident:** A startup built an agent for customer support that had NO long-term memory. Every interaction was a blank slate. Customers complained: "I already told you my order number. Why are you asking again?" The fix: a memory system that stored key customer facts (order numbers, issues, resolutions) between sessions. Complaint rate dropped 60%.

---

### 🤔 Scenario 3: The Memory That Grew Too Large

You implement long-term memory. The agent stores everything:

```
After 1 week of use: 500 facts stored
After 1 month: 5,000 facts stored
After 3 months: 50,000 facts stored

Now every query requires:
1. Search 50,000 facts for relevant memories
2. Insert top-10 relevant facts into context
3. LLM processes query + 10 facts + conversation history

Problem:
- Memory search takes 500ms (too slow)
- LLM gets distracted by irrelevant memories (noise)
- Storage costs growing
- Contradictory memories accumulating
```

**Your challenge:** You need a memory system that:
- Grows to 100K+ facts without slowing down
- Returns ONLY relevant memories (99%+ precision)
- Handles contradictions (old vs. new information)
- Manages storage cost

**🤔 Before reading the solutions:** Connect back to Phase 3. What did you build that solved a similar problem? How did you handle search at scale? How did you handle relevance?

> **Hint:** Phase 3 was about embeddings and vector search. The same tool is the foundation of agent memory.

---

> **Keep these 3 scenarios in mind. They define the entire memory landscape for agents.**

**By the end of this file, you will:**
1. Understand the **4 types of agent memory** — and why most tutorials only cover 1 of them
2. Build a **working memory manager** that keeps the conversation focused
3. Implement **episodic memory** using vector search (connecting to Phase 3)
4. Implement **semantic memory** — key facts that persist across sessions
5. Build a **memory consolidation system** that summarizes and prunes
6. Handle **memory contradictions** — what happens when new info conflicts with old

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Scenarios | 15 min |
| The 4 Types of Agent Memory | 20 min |
| Working Memory: Context Management | 20 min |
| Code: Working Memory with Sliding Window | 25 min |
| Episodic Memory: Vector-Based Recall | 25 min |
| Code: Building an Episodic Memory Store | 45 min |
| Semantic Memory: Facts That Persist | 20 min |
| Code: Semantic Memory Implementation | 30 min |
| Memory Consolidation & Summarization | 25 min |
| Code: Memory Consolidation Pipeline | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 20 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~5.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. The 4 Types of Agent Memory

Humans have multiple memory systems. So should agents.

```
MEMORY TYPE        │ WHAT IT IS                    │ AGENT ANALOGY
───────────────────┼───────────────────────────────┼─────────────────────────
WORKING MEMORY     │ Current conversation context  │ The message history
                   │ (what you're thinking now)    │ (system + user + assistant + tool results)

EPISODIC MEMORY    │ Past experiences              │ Past conversations and
                   │ (what happened before)        │ their outcomes, searchable

SEMANTIC MEMORY    │ Facts and knowledge            │ Key-value store of learned
                   │ (what you know)                │ facts about the user/world

PROCEDURAL MEMORY  │ How to do things              │ The system prompt + tool
                   │ (skills and procedures)        │ definitions + learned patterns
```

### 2. Working Memory — You Already Have This

Your ReAct agent's `self.messages` list IS working memory. It holds the current conversation context.

**The problem with working memory:**

```
Turn 1:  user + thought + tool_call + result = 500 tokens
Turn 2:  user + thought + tool_call + result = 500 tokens
...
Turn 20: 10,000 tokens (getting expensive)
Turn 40: 20,000 tokens (context is degrading)
Turn 80: 40,000 tokens (hitting limits, very expensive)
```

**🤔 The key question: Does every token from Turn 1 matter for Turn 40?**

Most of the time, no. The agent doesn't need the exact text of a search result from 3 hours ago. It needs:
- Key facts that were discovered
- Decisions that were made
- The user's preferences

Everything else can be consolidated.

### 3. Episodic Memory — Your Phase 3 Knowledge Search

**Cross-reference (Phase 3):** Remember your Personal Knowledge Search Engine? You indexed documents, embedded them, and searched by similarity.

Episodic memory is the SAME pattern, applied to agent conversations instead of documents:

```
Phase 3 Knowledge Search:                Agent Episodic Memory:
────────────────────────────             ────────────────────────────
Documents                                Past conversation episodes
↓                                        ↓
Embed (text-embedding-3-small)           Embed (same model)
↓                                        ↓
Vector DB (ChromaDB/Qdrant)              Vector DB (same DB)
↓                                        ↓
Search by query similarity               Search by current context similarity
↓                                        ↓
Return relevant documents                Return relevant past conversations
```

**The difference:**
- In Phase 3, you embedded static documents
- Here, you embed DYNAMIC conversation episodes
- The search query is the CURRENT conversation context
- Results are PAST experiences that might be relevant

### 4. The Memory Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    AGENT MEMORY SYSTEM                     │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │ WORKING MEMORY (always in context)                  │   │
│  │  - Current user message                             │   │
│  │  - Recent conversation history (last 5-10 turns)    │   │
│  │  - Current tool results                             │   │
│  │  - Retrieved relevant memories (injected)           │   │
│  │  MAX: ~4000 tokens (sliding window)                 │   │
│  └────────────────────────────────────────────────────┘   │
│                            │                               │
│                            ▼                               │
│  ┌────────────────────────────────────────────────────┐   │
│  │ MEMORY CONSOLIDATION (background)                   │   │
│  │  - Summarize completed conversation segments        │   │
│  │  - Extract key facts → semantic memory              │   │
│  │  - Index episodes → episodic memory                 │   │
│  │  - Prune redundant/outdated information             │   │
│  └────────────────────────────────────────────────────┘   │
│                            │                               │
│         ┌──────────────────┴──────────────────┐            │
│         ▼                                      ▼            │
│  ┌────────────────────┐              ┌────────────────────┐ │
│  │ EPISODIC MEMORY    │              │ SEMANTIC MEMORY    │ │
│  │ (vector searchable)│              │ (key-value store)  │ │
│  │  - Past episodes   │              │  - User preferences│ │
│  │  - Conversation    │              │  - Facts learned   │ │
│  │    summaries       │              │  - Decisions made  │ │
│  │  - Outcomes/lessons│              │  - Entity info     │ │
│  └────────────────────┘              └────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## 💻 Code Examples

### Example 1: Working Memory with Sliding Window

The most immediate memory fix — stop shoving everything into the LLM context:

```python
"""
Working Memory Manager — keeps the LLM's context window focused.
"""

from collections import deque
from typing import Optional
from dataclasses import dataclass, field
import tiktoken


@dataclass
class ConversationTurn:
    """A single turn in the conversation."""
    user_message: str
    agent_reasoning: str
    tool_calls: list[dict]  # [{tool, args, result}]
    final_answer: str
    token_count: int = 0


class WorkingMemory:
    """
    Manages the LLM context window efficiently.
    
    Instead of appending everything, it:
    1. Keeps the most recent N turns
    2. Summarizes old turns into a compressed form
    3. Injects relevant memories from long-term storage
    
    Connect back to Phase 2: This is the SAME principle as context
    engineering for prompts — you're not giving the LLM EVERYTHING,
    you're giving it the RIGHT things.
    """
    
    def __init__(
        self,
        max_tokens: int = 4000,
        system_prompt: str = "",
        model: str = "gpt-4o-mini",
    ):
        self.max_tokens = max_tokens
        self.system_prompt = system_prompt
        self.encoder = tiktoken.encoding_for_model(model)
        
        # Core working memory — always in context
        self.recent_turns: deque[ConversationTurn] = deque(maxlen=10)
        
        # Compressed summaries of older turns
        self.summaries: list[str] = []
        
        # Track token usage
        self._current_tokens: int = 0
    
    def _count_tokens(self, text: str) -> int:
        return len(self.encoder.encode(text))
    
    def _system_tokens(self) -> int:
        return self._count_tokens(self.system_prompt)
    
    def _summary_tokens(self) -> int:
        return sum(self._count_tokens(s) for s in self.summaries[-3:])  # Last 3 summaries
    
    def _recent_tokens(self) -> int:
        total = 0
        for turn in self.recent_turns:
            total += turn.token_count
        return total
    
    def add_turn(self, turn: ConversationTurn):
        """Add a conversation turn. May trigger summarization."""
        # Count tokens in this turn
        turn_text = (
            turn.user_message + turn.agent_reasoning + turn.final_answer +
            "".join(str(tc) for tc in turn.tool_calls[:3])  # Only count first 3 tool calls
        )
        turn.token_count = self._count_tokens(turn_text)
        
        self.recent_turns.append(turn)
        
        # Check if we need to consolidate
        if self._recent_tokens() + self._system_tokens() + self._summary_tokens() > self.max_tokens:
            self._consolidate()
    
    def _consolidate(self):
        """
        Move old turns to summaries.
        
        🤔 Why "summarize" instead of just keeping the raw text?
        
        Raw text of 10 old turns might be 3000 tokens.
        A summary of the same 10 turns might be 300 tokens.
        That's 2700 tokens saved — which can hold MORE recent context
        or be used for relevant memory injection.
        
        Tradeoff: Summaries lose detail. But for old turns,
        the KEY INSIGHTS matter more than the exact words.
        """
        if len(self.recent_turns) < 3:
            return  # Not enough to consolidate
        
        # Take the oldest 30% of recent turns and summarize them
        n_to_summarize = max(1, len(self.recent_turns) // 3)
        old_turns = list(self.recent_turns)[:n_to_summarize]
        
        # Create a summary
        summary_parts = []
        for turn in old_turns:
            summary_parts.append(
                f"User asked: {turn.user_message[:100]}\n"
                f"Key findings: {turn.final_answer[:200]}"
            )
        
        summary = "\n---\n".join(summary_parts)
        self.summaries.append(f"[Past conversation summary]:\n{summary}")
        
        # Remove old turns from recent
        for _ in range(n_to_summarize):
            self.recent_turns.popleft()
    
    def build_context(
        self,
        user_message: str,
        episodic_memories: Optional[list[str]] = None,
        semantic_facts: Optional[list[str]] = None,
    ) -> list[dict]:
        """
        Build the optimized message list for the LLM.
        
        This is the KEY method — it constructs a context that fits
        within the token budget while maximizing useful information.
        """
        messages = [{"role": "system", "content": self.system_prompt}]
        
        # Add relevant long-term memories (if any)
        memory_context = ""
        if semantic_facts:
            memory_context += "## Facts I know:\n" + "\n".join(f"- {f}" for f in semantic_facts) + "\n\n"
        if episodic_memories:
            memory_context += "## Relevant past experiences:\n" + "\n".join(f"- {m}" for m in episodic_memories[:3])
        
        if memory_context:
            # Inject memories as context
            messages.append({
                "role": "system",
                "content": f"Here is information from your long-term memory that's relevant to this conversation:\n\n{memory_context}"
            })
        
        # Add conversation summaries (compressed history)
        for summary in self.summaries[-3:]:
            messages.append({
                "role": "system",
                "content": f"[Context from earlier in conversation]: {summary}"
            })
        
        # Add recent turns (full detail)
        for turn in list(self.recent_turns):
            messages.append({"role": "user", "content": turn.user_message})
            messages.append({"role": "assistant", "content": turn.final_answer})
        
        # Add current user message
        messages.append({"role": "user", "content": user_message})
        
        return messages
```

---

### 🤔 Checkpoint: The Sliding Window Tradeoff

You're configuring the working memory. You have 4000 tokens budget. What do you prioritize?

```
Option A: Keep last 10 turns in full detail (400 tokens each)
           → User can reference anything from last 10 turns
           → But nothing from before that (except summaries)

Option B: Keep last 3 turns in full detail (1000 tokens each)
           + 1000 tokens reserved for episodic memories
           → Less recent history, but relevant past experiences

Option C: Keep last 20 turns but compressed (200 tokens each)
           → More total history, but each turn has less detail
```

**Which is best for:**
- **Customer support** (user might reference something from 15 turns ago)
- **Data analysis** (each turn has detailed results that might be needed again)
- **Creative writing** (long-running context matters for consistency)

> **Think about this before reading my recommendation below.**

---

**My recommendation:**
- **Customer support:** Option B. Customer support queries are usually independent. Recent context matters more than long history. Episodic memory (past customer interactions) is gold.
- **Data analysis:** Option C with a higher budget (6000 tokens). Data analysis often reuses results from 10+ turns ago. Compression preserves the gist while fitting more turns.
- **Creative writing:** Option A with periodic snapshots saved to episodic memory. Creative work needs full context of recent decisions, but can summarize further back.

**Senior engineer note:** There's no universal right answer. You need to ADAPT your memory strategy to your use case. The framework for building memory systems matters more than any specific configuration.

---

### Example 2: Episodic Memory with Vector Search

Now let's build the long-term memory that makes the agent remember across sessions:

```python
"""
Episodic Memory — vector-searchable past conversation episodes.

Connects directly to Phase 3. This is the same pattern:
embeddings → vector DB → similarity search.

The difference: instead of indexing documents, we index
conversation EPISODES.
"""

import json
import os
from datetime import datetime
from typing import Optional, Any
from dataclasses import dataclass, field


@dataclass
class Episode:
    """A memory episode — a meaningful unit of past interaction."""
    id: str
    timestamp: str
    summary: str                    # Short description of what happened
    user_goal: str                  # What the user was trying to do
    key_facts: list[str]            # Facts discovered/decisions made
    embedding: list[float]          # Vector embedding for search
    outcome: str = ""               # Was the goal achieved?
    tokens_used: int = 0            # Cost tracking
    tags: list[str] = field(default_factory=list)


class EpisodicMemory:
    """
    Long-term episodic memory using vector search.
    
    This is YOUR Phase 3 knowledge search engine, repurposed
    for agent memory. Same embedding model. Same vector DB.
    Different data being indexed.
    
    🤔 Realization: Vector databases are not just for RAG.
    They're for ANY semantic search problem. Agent memory.
    Recommendation systems. Code search. Duplicate detection.
    The pattern is universal.
    """
    
    def __init__(self, collection_name: str = "agent_memory"):
        # Using ChromaDB (from Phase 3)
        import chromadb
        from chromadb.config import Settings
        
        self.client = chromadb.Client(Settings(
            persist_directory="./agent_memory_db",
            anonymized_telemetry=False,
        ))
        
        # Get or create collection
        self.collection = self.client.get_or_create_collection(
            name=collection_name,
            metadata={"hnsw:space": "cosine"},
        )
        
        # Embedding model (from Phase 3)
        from sentence_transformers import SentenceTransformer
        self.embedder = SentenceTransformer("all-MiniLM-L6-v2")
    
    def store_episode(self, episode: Episode):
        """
        Store a conversation episode to memory.
        
        Called AFTER a significant interaction completes.
        """
        # Generate embedding from summary + key facts
        text_to_embed = f"{episode.summary}\nGoal: {episode.user_goal}\nFacts: {' '.join(episode.key_facts)}"
        embedding = self.embedder.encode(text_to_embed, normalize_embeddings=True)
        
        self.collection.add(
            embeddings=[embedding.tolist()],
            documents=[json.dumps({
                "summary": episode.summary,
                "user_goal": episode.user_goal,
                "key_facts": episode.key_facts,
                "outcome": episode.outcome,
                "timestamp": episode.timestamp,
                "tags": episode.tags,
            })],
            metadatas=[{
                "timestamp": episode.timestamp,
                "outcome": episode.outcome,
                "tokens_used": episode.tokens_used,
            }],
            ids=[episode.id],
        )
    
    def recall(self, query: str, n_results: int = 5) -> list[dict[str, Any]]:
        """
        Recall relevant past episodes based on current context.
        
        Called BEFORE every LLM interaction to inject relevant memories.
        
        🤔 Think about this: The query is the CURRENT conversation
        context. We're searching for PAST conversations that are
        semantically similar to what's happening NOW.
        
        This is EXACTLY how human memory works — you see something
        that reminds you of a past experience, and you recall it.
        """
        embedding = self.embedder.encode([query], normalize_embeddings=True)
        
        results = self.collection.query(
            query_embeddings=[embedding.tolist()],
            n_results=n_results,
        )
        
        memories = []
        for i in range(len(results["documents"][0])):
            doc = json.loads(results["documents"][0][i])
            memories.append({
                "id": results["ids"][0][i],
                "summary": doc["summary"],
                "key_facts": doc.get("key_facts", []),
                "outcome": doc.get("outcome", ""),
                "timestamp": doc.get("timestamp", ""),
                "distance": results["distances"][0][i] if results["distances"] else None,
            })
        
        return memories
    
    def forget(self, episode_id: str):
        """Delete a specific memory."""
        self.collection.delete(ids=[episode_id])
    
    def get_recent(self, n: int = 10) -> list[dict]:
        """Get most recent episodes."""
        results = self.collection.get(limit=n)
        memories = []
        for i in range(len(results["documents"])):
            doc = json.loads(results["documents"][i])
            memories.append({
                "id": results["ids"][i],
                "summary": doc["summary"],
                "timestamp": doc.get("timestamp", ""),
            })
        # Sort by timestamp descending
        memories.sort(key=lambda x: x.get("timestamp", ""), reverse=True)
        return memories
    
    def search_by_tag(self, tag: str) -> list[dict]:
        """Search memories by tag."""
        results = self.collection.get(
            where={"tag": tag},
            limit=20,
        )
        return [
            json.loads(doc)
            for doc in results["documents"]
        ]


# ── Memory Consolidation Agent ──

class MemoryConsolidator:
    """
    Periodically consolidates raw conversation turns into stored memories.
    
    This runs in the BACKGROUND after a significant interaction.
    It:
    1. Takes raw conversation history
    2. Identifies key facts discovered
    3. Identifies decisions made
    4. Creates compact episode records
    5. Stores to episodic memory
    """
    
    def __init__(self, model: str = "gpt-4o-mini"):
        from openai import OpenAI
        self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.model = model
        self.episodic = EpisodicMemory()
    
    def consolidate(
        self,
        conversation: list[dict],
        user_id: str,
        session_id: str,
    ) -> Optional[Episode]:
        """
        Convert a conversation segment into a stored memory.
        
        Returns an Episode if there are meaningful facts to remember.
        Returns None if the conversation was trivial.
        """
        # Convert conversation to text
        conv_text = json.dumps(conversation[-10:], indent=2)  # Last 10 turns
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": """You extract memories from agent conversations.
                
Analyze the conversation and extract:
1. SUMMARY: What happened in 1-2 sentences
2. USER_GOAL: What the user was trying to accomplish
3. KEY_FACTS: Important facts discovered (as a list)
4. OUTCOME: Was the goal achieved? (achieved/partial/failed)
5. TAGS: 2-5 keywords for categorization
6. TOKENS_USED: Total tokens from the conversation

Respond ONLY with a JSON object.
If nothing meaningful happened (casual chat), respond with {"meaningful": false}"""},
                {"role": "user", "content": conv_text},
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
        )
        
        result = json.loads(response.choices[0].message.content)
        
        if not result.get("meaningful", True):
            return None
        
        episode = Episode(
            id=f"{user_id}_{session_id}_{datetime.utcnow().timestamp()}",
            timestamp=datetime.utcnow().isoformat(),
            summary=result["summary"],
            user_goal=result["user_goal"],
            key_facts=result.get("key_facts", []),
            embedding=[],  # Will be computed in store_episode
            outcome=result.get("outcome", ""),
            tokens_used=result.get("tokens_used", 0),
            tags=result.get("tags", []),
        )
        
        self.episodic.store_episode(episode)
        return episode
```

---

### Example 3: Semantic Memory — Facts That Persist

Not everything needs vector search. Some things should just be KEY-VALUE pairs:

```python
"""
Semantic Memory — persistent facts stored as key-value pairs.

Different from episodic memory:
- Episodic: "What happened during the conversation about project X?"
- Semantic: "The user's name is Partha and their project uses FastAPI"

Semantic memory is FASTER and MORE PRECISE than vector search.
But it can only store simple facts, not complex narratives.
"""

import json
import os
from datetime import datetime
from typing import Optional


class SemanticMemory:
    """
    Key-value store for persistent facts about users and the world.
    
    This uses a simple JSON file for development.
    In production, use Redis or a database.
    
    🤔 Think about what goes where:
    - Episodic memory: "In our last session, we discussed..."
      → Vector search (semantic matching)
    - Semantic memory: "User prefers GPT-4o, has Pro subscription"
      → Key-value lookup (exact match)
    
    Episodic = fuzzy recall ("something like this happened before")
    Semantic = precise recall ("this is a known fact")
    """
    
    def __init__(self, storage_path: str = "semantic_memory.json"):
        self.storage_path = storage_path
        self.memory: dict[str, dict] = {}
        self._load()
    
    def _load(self):
        if os.path.exists(self.storage_path):
            with open(self.storage_path) as f:
                self.memory = json.load(f)
    
    def _save(self):
        with open(self.storage_path, "w") as f:
            json.dump(self.memory, f, indent=2)
    
    def remember(self, key: str, value: str, confidence: float = 1.0):
        """
        Store a fact in semantic memory.
        
        Args:
            key: Unique identifier (e.g., "user:partha:preferred_model")
            value: The fact to remember
            confidence: How sure we are (0.0 to 1.0)
        """
        self.memory[key] = {
            "value": value,
            "confidence": confidence,
            "updated_at": datetime.utcnow().isoformat(),
        }
        self._save()
    
    def recall(self, key: str) -> Optional[str]:
        """Retrieve a fact from semantic memory."""
        entry = self.memory.get(key)
        if entry:
            return entry["value"]
        return None
    
    def search(self, prefix: str) -> dict[str, str]:
        """Find all facts with a given key prefix (e.g., "user:partha:")."""
        return {
            k: v["value"]
            for k, v in self.memory.items()
            if k.startswith(prefix)
        }
    
    def update_or_conflict(self, key: str, new_value: str, new_confidence: float) -> str:
        """
        Store a fact, handling potential conflicts with old values.
        
        Returns: "stored" | "updated" | "conflict"
        
        This solves Scenario 3's contradiction problem:
        what happens when new info conflicts with old info?
        """
        existing = self.memory.get(key)
        
        if not existing:
            self.remember(key, new_value, new_confidence)
            return "stored"
        
        if existing["value"] == new_value:
            return "stored"  # Same value, no change
        
        if new_confidence > existing["confidence"]:
            self.remember(key, new_value, new_confidence)
            return "updated"  # New info is more reliable
        else:
            return "conflict"  # We have conflicting info
    
    def get_all_facts_for_context(self, user_id: str) -> str:
        """
        Get all known facts about a user, formatted for LLM context.
        
        This is called at the START of each session to inject
        what the agent knows about the user.
        """
        user_facts = self.search(f"user:{user_id}")
        if not user_facts:
            return ""
        
        lines = ["## What I know about you:"]
        for key, value in user_facts.items():
            # Convert "user:partha:preferred_model" → "Preferred model"
            readable_key = key.split(":", 2)[-1].replace("_", " ").title()
            lines.append(f"- {readable_key}: {value}")
        
        return "\n".join(lines)


# ── Integrating Memory into the Agent ──

class AgentWithMemory:
    """
    A ReAct agent with FULL memory support.
    
    This combines:
    1. Working memory (sliding window context)
    2. Episodic memory (vector searchable past)
    3. Semantic memory (persistent key-value facts)
    4. Memory consolidation (background processing)
    """
    
    def __init__(
        self,
        tools: list,
        system_prompt: str,
        user_id: str,
        model: str = "gpt-4o-mini",
    ):
        self.tools = tools
        self.model = model
        self.user_id = user_id
        
        # Memory systems
        self.working_memory = WorkingMemory(
            system_prompt=system_prompt,
            model=model,
        )
        self.episodic_memory = EpisodicMemory()
        self.semantic_memory = SemanticMemory()
        self.consolidator = MemoryConsolidator(model=model)
        
        # Session tracking
        self.session_id = f"session_{datetime.utcnow().timestamp()}"
        self.conversation_log: list[dict] = []
        
        # Load known facts about user
        user_context = self.semantic_memory.get_all_facts_for_context(user_id)
        self.working_memory.system_prompt = system_prompt
        if user_context:
            self.working_memory.system_prompt += f"\n\n{user_context}"
    
    def run(self, user_message: str) -> str:
        """
        Run the agent with full memory support.
        
        The flow:
        1. Search episodic memory for relevant past experiences
        2. Check semantic memory for known facts
        3. Build optimized context (working memory + memories)
        4. Run ReAct loop
        5. After completion, consolidate into long-term memory
        """
        # Step 1 & 2: Recall relevant memories
        episodic_memories = self.episodic_memory.recall(user_message)
        print(f"🧠 Recalled {len(episodic_memories)} relevant past episodes")
        
        # Step 3: Build context with memories
        context = self.working_memory.build_context(
            user_message=user_message,
            episodic_memories=[
                f"[{m['timestamp'][:10]}] {m['summary']}" 
                for m in episodic_memories[:3]
            ],
        )
        
        # Step 4: Run ReAct loop (simplified — see File 02 for full implementation)
        # ... ReAct loop using context instead of self.messages ...
        final_answer = self._react_loop(user_message, context)
        
        # Step 5: Consolidate (run after meaningful interactions)
        self.conversation_log.append({
            "user": user_message,
            "assistant": final_answer,
            "timestamp": datetime.utcnow().isoformat(),
        })
        
        # Consolidate every 5 turns or when conversation is long
        if len(self.conversation_log) % 5 == 0:
            episode = self.consolidator.consolidate(
                self.conversation_log[:-5],
                self.user_id,
                self.session_id,
            )
            if episode:
                print(f"💾 Saved memory: {episode.summary}")
        
        return final_answer
    
    def _react_loop(self, user_message: str, initial_context: list[dict]) -> str:
        """ReAct loop using memory-enhanced context."""
        # Implementation from File 02, but uses initial_context
        # instead of raw self.messages
        # ... (simplified for brevity - see File 02 for full code)
        return "Agent response with memory"  # Placeholder
```

---

## ✅ Good Output Examples

### What a Well-Memorizing Agent Looks Like

```
Session 1:
  User: "I'm building a chatbot with FastAPI and GPT-4o"
  Agent: "Great choices! FastAPI is excellent for async AI backends..."
  → Memory consolidated: "User building chatbot with FastAPI + GPT-4o"
  → Key fact: preferred_model = "GPT-4o"

Session 2 (next day):
  User: "I need to add streaming to my chatbot"
  Agent: "🧠 I remember you're building a chatbot with FastAPI + GPT-4o.
         Streaming with GPT-4o is straightforward..."
  User: "Yes! That's exactly what I need."

✅ MEMORY:
  • No repeated questions ("What project? What stack?")
  • Continuity between sessions
  • User feels understood

COST SAVED:
  • The agent didn't re-ask for context (1 extra turn)
  • The agent didn't re-search for the user's tech stack
  • Savings: ~$0.003 per session × 1000 sessions = $3/month
  • BUT the real value: user satisfaction
```

### What a Forgetting Agent Looks Like

```
Session 1:
  User: "I'm building a chatbot with FastAPI and GPT-4o"
  Agent: "Great choices! FastAPI is excellent..."

Session 2:
  User: "I need to add streaming to my chatbot"
  Agent: "Tell me about your project first. What stack are you using?"
  User: "I already told you yesterday! FastAPI + GPT-4o."
  Agent: "Ah, sorry about that. Yes, FastAPI supports streaming..."

❌ NO MEMORY:
  • Asks repeated questions
  • User is frustrated
  • Wastes tokens re-establishing context
  
COST WASTED:
  • Re-establishing context: ~$0.002 per repeat
  • User frustration: priceless
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: The Everything-In-Memory Trap

```python
# ❌ WRONG — Store everything in memory
def remember_everything(conversation_log):
    for turn in conversation_log:
        memory.store(turn)  # Every single message
# Problem: 1000 conversations × 20 turns = 20,000 memories
# Most are useless noise. Search becomes slow and noisy.

# ✅ RIGHT — Store meaningful summaries
def consolidate_memory(conversation_log):
    if is_significant_interaction(conversation_log):
        summary = extract_key_insights(conversation_log)
        memory.store(summary)  # One high-quality memory
# Quality over quantity. Only store what matters.
```

### Antipattern 2: No Memory Freshness

```python
# ❌ WRONG — Never update or forget
memory.remember("user_preferred_model", "GPT-3.5")  # Stale!
# 6 months later, user uses Claude 3.5, but memory says GPT-3.5

# ✅ RIGHT — Update with confidence tracking
memory.remember("user_preferred_model", "GPT-3.5", confidence=0.8)
# Later...
conflict = memory.update_or_conflict(
    "user_preferred_model", "Claude 3.5", confidence=0.9
)
# Returns "updated" because new confidence > old confidence
```

### Antipattern 3: Memory as a Crutch for Bad Design

```python
# ❌ WRONG — Using memory when the root cause is something else
# Agent keeps forgetting the user's name
# "Fix": Store name in memory, look it up every time
# Root cause: The agent's system prompt doesn't capture user identity
# Better fix: Include user info in the system prompt

# ✅ RIGHT — Fix the root cause first
system_prompt = f"You are helping {user.name}, a {user.role} at {user.company}."
# Memory is for things that CHANGE (preferences, project status)
# NOT for things that are STATIC (name, role, company)
```

### Antipattern 4: Too Much Memory Injection

```python
# ❌ WRONG — Inject ALL memories
memories = memory.recall(query, n_results=20)
context += format_memories(memories)  # 20 memories = 3000 tokens
# LLM is overwhelmed with irrelevant noise

# ✅ RIGHT — Inject only the most relevant
memories = memory.recall(query, n_results=5)
context += format_memories(memories)  # 5 memories = 500 tokens
# Fewer, more relevant memories
```

### Antipattern 5: The "It Was In There" Assumption

```
❌ PROBLEM:
Agent: "I don't have that information"
User: "But I told you in our last session!"
Agent: "I'm sorry, I don't have memory of previous sessions."

The agent didn't check memory. It just said "I don't know."

✅ FIX:
Always check memory before declaring failure.
The system prompt should include:
"Before saying you don't know something, check your 
long-term memory. The user may have told you in a previous session."
```

---

## 🧪 Drills & Challenges

### Drill 1: The Memory Audit (25 min)

You have an agent that runs customer support. Analyze its memory needs:

```
User types: "I need to return my order #12345"
Agent needs to remember:
1. The user's name and account status (from previous sessions)
2. The current conversation (what they've said so far)
3. The order details (from the database, not memory)
4. The return policy (from the knowledge base, not memory)
5. Previous return requests (from past sessions)

For each: Which memory type? (working/episodic/semantic/none)
How long should it persist? (session/1 day/1 month/forever)
```

**Then:** Implement the memory configuration for this use case.

### Drill 2: The Five-Turn Forgetting Test (20 min)

Run an agent through 5 turns of a conversation. After each turn, ask:

> "What information from Turn 1 is still accessible to the agent?"

Track when information is lost:
- Turn 1: "My name is Partha and I use Python 3.12"
- Turn 2: "I'm building a RAG system with ChromaDB"
- Turn 3: "I'm using sentence-transformers for embeddings"  
- Turn 4: "I want to deploy on AWS ECS"
- Turn 5: "What's the best embedding model for my project?"

**Does the agent still know your name? Your tech stack? Your deployment target?**

If not — your memory system is losing critical context.

### Drill 3: Memory Conflict Resolution (15 min)

Your semantic memory has:
```
user:partha:preferred_model = "GPT-4o" (confidence 0.9, from 3 months ago)
user:partha:preferred_model = "Claude 3.5" (new input, from today)
```

**Design the conflict resolution logic.** When should it:
- Overwrite the old value?
- Keep both with different timestamps?
- Ask the user for clarification?
- Flag for human review?

Also think about: What if the model preference changed because they're trying something new vs. permanently switching?

### Drill 4: The Cost of Memory (15 min)

```
Setup:
- 1000 conversations/day
- Average conversation: 10 turns, 2000 tokens
- Storing: every conversation → episodic memory (1000/day)
- Retrieval: every turn searches memory (10,000 searches/day)
- Embedding model: all-MiniLM-L6-v2 (free, local)
- Vector DB: ChromaDB (free)

Calculate:
1. Storage growth per day (KB/MB/GB)
2. After 1 year, how much storage?
3. Latency added per turn for memory retrieval (embed + search)
4. Cost: is there any? (embedding is local)
5. When would you need to prune old memories?

Compare to NOT having memory:
6. Extra tokens wasted re-establishing context per session
7. Additional LLM cost for those tokens
8. User satisfaction impact (hard to quantify, but real)
```

### Drill 5: Design the Memory Schema (20 min)

Design the memory schema for an **AI coding assistant** that:

1. Remembers which project the user is working on
2. Recalls recently edited files
3. Knows the user's coding style preferences (tabs vs spaces, etc.)
4. Remembers previous bugs and their fixes
5. Forgets temporary context (what I was doing 5 minutes ago)
6. Learns from user corrections

**For each type of memory, specify:**
- Storage format (vector? key-value? summary text?)
- Retention policy (session? day? month? forever?)
- Retrieval trigger (automatic? on-request?)
- Update strategy (overwrite? merge? conflict resolution?)

---

## 🚦 Gate Check

Before moving to File 05 (LangGraph Deep Dive), confirm you can:

- [ ] **Explain the 4 types of agent memory** with concrete examples
- [ ] **Implement a working memory** with sliding window and summarization
- [ ] **Build episodic memory** using vector search (connecting to Phase 3)
- [ ] **Build semantic memory** with conflict resolution
- [ ] **Implement memory consolidation** — converting raw conversations to stored memories
- [ ] **Handle memory conflicts** — what happens when old and new info disagree

### The Real Test

Build a memory system for this scenario:

```
User: A product manager who uses your agent daily for 3 months

Session 1: "I'm starting a project called 'Project Alpha' — a customer analytics dashboard"
Session 5: "Project Alpha's tech stack: Next.js frontend, FastAPI backend, Postgres DB"
Session 15: "We decided to use GPT-4o for the AI features"
Session 30: "Project Alpha is now in beta with 50 test users"
Session 45: (3 months later) "What's the status of Project Alpha again?"
```

**Your memory system must be able to answer:**
- What is Project Alpha? (project details)
- What's the tech stack? (from session 5)
- What AI model are they using? (from session 15)
- How many beta users? (from session 30)
- These facts were spread across 45 sessions over 3 months

### Final Questions

1. **Cross-reference Phase 3:** You built a vector search engine in Phase 3. How is episodic memory DIFFERENT from RAG retrieval? When should the agent use its RAG system vs. its episodic memory?

2. **The false memory problem:** An agent consolidates a conversation and stores a SUMMARY. The summary says "User prefers GPT-4o over Claude." But the user actually said "GPT-4o is cheaper, but Claude is better for long documents." The summary OVER-SIMPLIFIED. Next session, the agent says "You prefer GPT-4o" and the user corrects it. How do you detect and fix these memory distortions?

3. **The privacy problem:** You store 3 months of agent conversations in episodic memory. A user asks "Delete everything about me from the system." Your memory is a vector database — how do you find and delete ALL memories related to this user? (It's harder than it sounds.)

---

## 📚 Resources

**Foundational:**
- 📄 [Lilian Weng — LLM Powered Autonomous Agents (Memory Section)](https://lilianweng.github.io/posts/2023-06-23-agent/#memory) — Best survey on agent memory
- 📄 [MemGPT — Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — Paper on virtual memory management for LLMs
- 📄 [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442) — Stanford's paper on agent memory (the one with the AI town)

**Memory Patterns:**
- 📄 [Mem0: Memory Layer for AI Assistants](https://github.com/mem0ai/mem0) — Open-source memory system
- 📄 [LangGraph Memory Patterns](https://langchain-ai.github.io/langgraph/concepts/memory/) — Different memory strategies in LangGraph

**Phase 3 Connection:**
- 📄 [Sentence-Transformers Documentation](https://www.sbert.net/) — Your embedding model from Phase 3
- 📄 [ChromaDB Documentation](https://docs.trychroma.com/) — Your vector DB from Phase 3

**Your Memory Systems Progress Checklist:**
```
☐ 4 types of memory understood
☐ Working memory with sliding window implemented
☐ Episodic memory with vector search built
☐ Semantic memory with conflict resolution built
☐ Memory consolidation pipeline working
☐ Memory integrated into ReAct agent
☐ Memory performance and cost analyzed
☐ Privacy and deletion handled
```

---

> **Revisit the 3 scenarios from the beginning:**
>
> 1. **The Conversation That Doesn't Learn** — Solved by sliding window working memory + consolidation. Old turns are summarized, key facts are extracted, and the LLM gets compressed history instead of raw logs.
>
> 2. **The Forgetting Agent** — Solved by episodic + semantic memory. Past sessions are embedded and searchable. Key facts persist as key-value pairs. The agent starts each session by recalling what it knows about the user.
>
> 3. **The Memory That Grew Too Large** — Solved by memory consolidation + pruning. Not everything is stored — only meaningful episodes. Old memories can be further compressed, deleted, or archived based on retention policies.
>
> **Key realization:** Memory isn't about storing everything. It's about storing the RIGHT things and retrieving them at the RIGHT time. Vector search gives you semantic recall. Key-value stores give you precise recall. Working memory gives you immediate context. Each serves a different purpose, and production agents need ALL three.
>
> **Next:** File 05 — LangGraph Deep Dive. How to build stateful, observable, production-grade agent graphs.

---

> **Senior engineer closing thought:**
>
> *"Memory is the feature users notice most. They'll forgive a slow agent. They'll forgive a slightly wrong answer. But they will NOT forgive an agent that forgets who they are or what they discussed. The startup I worked for lost a $50K enterprise deal because the agent asked 'What company are you from?' three times in one call. The CTO said: 'If your AI can't remember basic context, I can't trust it with my data.' Memory is table stakes for production agents. Not a nice-to-have."*
