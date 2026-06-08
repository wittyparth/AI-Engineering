# Context Windows & Attention — Your Most Important Resource

## 🎯 Purpose & Goals

The context window is the **memory** of an LLM — everything the model can "see" at once. It's also the most misunderstood and misused resource in AI engineering. After this file, you will stop making the most common (and expensive) mistakes engineers make with context.

**By the end of this file, you will:**
- Understand exactly what happens inside a context window (attention distribution)
- Know why "lost in the middle" matters and how to design around it
- Be able to strategically allocate context window space for maximum quality
- Understand how different models handle context differently
- Have a production-ready context management strategy

**⏱ Time Budget:** 1.5 hours

---

## 📖 What Goes In The Context Window?

```
┌─────────────────────────────────────────────────────────────────┐
│                     CONTEXT WINDOW (~128K tokens)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [SYSTEM PROMPT] ← Your instructions to the model              │
│                                                                  │
│  [CONVERSATION HISTORY] ← Previous messages in this chat       │
│                                                                  │
│  [RETRIEVED DOCUMENTS (RAG)] ← Facts you give the model        │
│                                                                  │
│  [USER'S CURRENT MESSAGE] ← What they're asking now            │
│                                                                  │
│  ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓  │
│                                                                  │
│  [MODEL'S GENERATED RESPONSE] ← What it outputs                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Total = ALL OF THE ABOVE must fit within the context window.
If total > context window: OLDEST content gets truncated (silently!).
```

> 🤔 **Think about this:** If you have a 128K context window and your system prompt + history + RAG chunks = 60K tokens, and the model generates 1000 tokens... is that fine? Yes, but here's the catch: the model doesn't pay equal attention to all 60K tokens. Some parts get more attention than others. Some parts get almost none.

---

## 📖 The "Lost in the Middle" Problem

**This is the single most important research finding for AI engineers.**

A Stanford paper (2023) tested: give the model 20 relevant documents. Ask a question. Only 1 document contains the answer. Where in the context is that document?

**Results:**
```
Document position → Accuracy
─────────────────────────────
Position 1  (start):   75% accuracy  ← Best for important info
Position 5  (early):   65% accuracy  
Position 10 (middle):  35% accuracy  ← WORST!
Position 15 (late):    45% accuracy
Position 20 (end):     70% accuracy  ← Second best
```

**CRITICAL INSIGHT:** Models pay most attention to the BEGINNING and END of context. Content in the MIDDLE gets significantly less attention.

### How Senior Engineers Apply This:

```python
# WRONG (puts important info in the middle):
messages_wrong = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": f"""
    Here's some background info: {less_important_context}
    
    And here are the policies you need for this question: {critical_policy_info}
    
    Also: {some_metadata}
    
    My question is: {question}
    """}
]

# RIGHT (puts critical info at beginning and end):
messages_right = [
    {"role": "system", "content": f"""
    CRITICAL CONTEXT (read first):
    {critical_policy_info}
    
    You are a helpful assistant. Answer questions using the provided context.
    """},
    {"role": "user", "content": f"""
    Background: {less_important_context}
    
    Question: {question}
    
    Remember: Base your answer on the policy information provided above.
    """}  # The last thing the model sees = high attention
]
```

> 🤔 **Question:** If the "lost in the middle" problem means middle content gets less attention, what does this mean for RAG systems that retrieve 10 chunks? Where should you place the MOST relevant chunk?

---

## 📖 Production Context Management Patterns

```python
class ContextStrategist:
    """
    Professional-grade context management.
    A senior engineer's approach to allocating context window space.
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.model_limits = {
            "gpt-4o": 128000,
            "gpt-4o-mini": 128000,
            "claude-3-5-sonnet": 200000,
            "gemini-1.5-pro": 1000000,
            "llama-3.2": 128000,
        }
        self.max_tokens = self.model_limits.get(model, 128000)
    
    def build_strategic_context(self, 
                               system_prompt: str,
                               conversation_history: list[dict],
                               retrieved_chunks: list[dict],  # dicts with content + relevance_score
                               user_query: str,
                               reserve_output_tokens: int = 2000) -> list[dict]:
        """
        Build context that strategically allocates attention.
        
        Strategy:
        1. System prompt = always at top (high attention)
        2. Most relevant chunks = right after system prompt
        3. Less relevant chunks = in the middle
        4. Conversation history (recent first)
        5. User query = at the end (highest attention)
        """
        
        messages = []
        available = self.max_tokens - reserve_output_tokens
        
        # --- HIGH ATTENTION ZONE: System Prompt ---
        system_tokens = self.count_tokens_fast(system_prompt)
        if system_tokens > available * 0.1:  # Cap system prompt at 10%
            system_prompt = self.truncate_to_budget(system_prompt, int(available * 0.1))
        
        messages.append({"role": "system", "content": system_prompt})
        available -= system_tokens
        
        # --- HIGH ATTENTION ZONE: Top chunks ---
        sorted_chunks = sorted(retrieved_chunks, key=lambda x: x.get("relevance_score", 0), reverse=True)
        
        # Top 2 chunks go right after system prompt
        high_priority = sorted_chunks[:2]
        mid_priority = sorted_chunks[2:]
        
        context_parts = []
        for chunk in high_priority:
            chunk_tokens = self.count_tokens_fast(chunk["content"])
            if available - chunk_tokens > 500:
                context_parts.append(chunk["content"])
                available -= chunk_tokens
        
        # --- LOW ATTENTION ZONE: Middle chunks ---
        for chunk in mid_priority:
            chunk_tokens = self.count_tokens_fast(chunk["content"])
            if available - chunk_tokens > 1000:  # Leave room for history + query
                context_parts.append(chunk["content"])
                available -= chunk_tokens
            else:
                break
        
        if context_parts:
            messages.append({
                "role": "system",
                "content": f"Use this reference information:\n\n{'---'.join(context_parts)}"
            })
        
        # --- HIGH ATTENTION ZONE: Recent history + User query ---
        # Keep only enough history to understand context
        history_budget = int(available * 0.3)
        history_tokens = 0
        for msg in reversed(conversation_history[-5:]):  # Max 5 messages
            msg_tokens = self.count_tokens_fast(msg["content"])
            if history_tokens + msg_tokens < history_budget:
                messages.insert(-1 if messages else 0, msg)
                history_tokens += msg_tokens
            else:
                break
        
        # User query always goes last (highest attention after system prompt)
        messages.append({"role": "user", "content": user_query})
        
        return messages
    
    def count_tokens_fast(self, text: str) -> int:
        """Fast approximate token count (no API call needed)"""
        return len(text.split()) * 1.3  # Rough estimate
    
    def truncate_to_budget(self, text: str, budget: int) -> str:
        """Truncate text to fit within budget"""
        approx_tokens = self.count_tokens_fast(text)
        if approx_tokens <= budget:
            return text
        
        # Truncate by character ratio (rough)
        ratio = budget / approx_tokens
        truncated_len = int(len(text) * ratio * 0.9)  # 10% safety margin
        return text[:truncated_len] + "\n... [truncated]"
```

---

## 📖 Context Window Strategies by Use Case

```python
"""
DIFFERENT USE CASES NEED DIFFERENT CONTEXT STRATEGIES.
There is no one-size-fits-all approach.
"""

class ContextStrategy:
    """Factory for context strategies based on use case"""
    
    @staticmethod
    def for_chatbot():
        """
        Strategy: Recent history focused
        - Keep last N messages only
        - Summarize older messages into a single "conversation summary"
        - Customer service: keep policy docs at start, rotate conversations
        """
        return {
            "max_history_messages": 10,
            "history_budget_pct": 0.4,
            "rag_budget_pct": 0.3,
            "system_prompt_budget_pct": 0.1,
            "summarize_old": True,
        }
    
    @staticmethod
    def for_document_analysis():
        """
        Strategy: Document focused
        - The document IS the context
        - Minimal system prompt
        - No history (each query is independent)
        - Use model with largest context window (Claude 200K or Gemini 1M)
        """
        return {
            "max_history_messages": 0,
            "history_budget_pct": 0.0,
            "doc_budget_pct": 0.85,
            "system_prompt_budget_pct": 0.05,
        }
    
    @staticmethod
    def for_code_generation():
        """
        Strategy: File context focused
        - Include related files as context
        - Keep imports and type definitions at top
        - Current file at bottom
        """
        return {
            "max_history_messages": 3,
            "file_context_pct": 0.7,
            "system_prompt_pct": 0.1,
            "conversation_pct": 0.2,
        }
```

---

## 🧪 Debug Challenge: The Disappearing Context

**Scenario:** Your team built a document analysis tool using Claude 3.5 Sonnet (200K context). Users upload long documents (50-100 pages). The tool worked in testing with 20-page docs, but with 80+ pages, the analysis quality drops sharply.

```python
# BUGGY CODE — find what's wrong:
from anthropic import Anthropic

client = Anthropic()

def analyze_document(document_text: str, question: str) -> str:
    """
    Analyze a document and answer a question.
    Works great for 20-page docs. Quality drops for 80+ page docs.
    """
    prompt = f"""You are a document analyst. Read this document and answer the question.
    
DOCUMENT:
{document_text}
    
QUESTION:
{question}
    
Provide a detailed answer based on the document."""
    
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text
```

**Find at least 3 problems.**

<details>
<summary>🔍 Click to reveal</summary>

**Problem 1: All document content is in the middle**
The entire 80-page document goes in ONE user message. Since the system prompt is tiny ("You are a document analyst"), the document occupies most of the context — which is the LOW-ATTENTION middle zone. The question at the bottom gets attention, but the model struggles to connect it to content in the middle.

**Problem 2: No chunking strategy**
The model's attention degrades quadratically with context length. An 80-page doc = ~60K tokens (at ~750 words/page). At this length, the "lost in the middle" effect is severe. The model literally does not "see" middle pages as clearly as first/last.

**Problem 3: Question placement amplifies the problem**
The question is at the bottom. If the answer is on page 40 (the middle), the model has to connect high-attention question with low-attention document content. It often misses.

**Fixed approach:**
```python
def analyze_document_strategic(document_text: str, question: str) -> str:
    # Strategy: Split doc into sections, put the MOST RELEVANT sections
    # at the beginning and end of context.
    
    sections = split_into_sections(document_text)  # By headers or paragraph breaks
    section_embeddings = embed_all(sections)
    question_embedding = embed(question)
    
    # Find most relevant sections
    scored_sections = [
        (cosine_similarity(question_embedding, sec_emb), section)
        for sec_emb, section in zip(section_embeddings, sections)
    ]
    scored_sections.sort(key=lambda x: x[0], reverse=True)
    
    # Build context: most relevant at START, second most relevant at END,
    # rest in middle (trimmed)
    context_parts = [scored_sections[0][1]]  # Top at start
    context_parts.extend(s[1] for s in scored_sections[2:-2])  # Middle (trimmed)
    context_parts.append(scored_sections[1][1])  # Second best at end
    
    prompt = f"""... context built with attention-aware ordering ..."""
    # ...
```
</details>

---

## 📖 Model-Specific Context Handling

```python
"""
DIFFERENT MODELS HANDLE CONTEXT DIFFERENTLY.
This isn't theoretical — it affects your architecture.
"""
MODEL_CONTEXT_NOTES = {
    "gpt-4o": {
        "max_window": 128000,
        "effective_window": 64000,  # Quality drops after ~64K
        "notes": "Strong at start/end, weak middle",
        "best_for": "Short-medium RAG, conversations",
    },
    "claude-3-5-sonnet": {
        "max_window": 200000,
        "effective_window": 150000,  # Better retention than GPT-4o
        "notes": "Best in class for long-context retention",
        "best_for": "Long documents, codebase analysis",
    },
    "gemini-1.5-pro": {
        "max_window": 1000000,
        "effective_window": 700000,  # MASSIVE, but quality suffers
        "notes": "Can handle entire codebases, but expensive",
        "best_for": "Massive document analysis, not real-time",
    },
    "llama-3.2-70b": {
        "max_window": 128000,
        "effective_window": 32000,  # Significantly worse than GPT-4o
        "notes": "Open-source, free, but quality drops faster",
        "best_for": "Development, privacy-critical, short contexts",
    },
}

# PRODUCTION INSIGHT:
# Never assume you can fill 100% of the context window.
# Effective window is typically 50-70% of max window.
# Beyond that, quality degrades faster than cost increases.
```

---

## ✅ Context Management Checklist

For EVERY AI feature, ask:

- [ ] What's the TOTAL tokens going into the context window per request?
- [ ] What's the effective attention zone of my model? (50-70% of max)
- [ ] Is the MOST important information at the START or END of context?
- [ ] Am I shipping content to the MIDDLE that the model will ignore?
- [ ] Do I NEED all this context, or can I trim?
- [ ] Am I accounting for the reserve output tokens?
- [ ] Does my context strategy change by use case?

**Cross-reference:** This connects to:
- **Phase 4 (RAG):** Retrieval quality is useless if context placement ignores attention
- **Phase 6 (Agents):** Agent loops generate massive context — need careful management
- **Phase 10 (Costs):** More context = more tokens = more money

---

## 🚦 Gate Check

- [ ] I can explain the "lost in the middle" problem in one sentence
- [ ] I know where to put critical information in the context window
- [ ] I can estimate effective vs maximum context window for any model
- [ ] I know why 80-page document analysis needs different strategies than chatbots
- [ ] I can identify context placement bugs in production code

---

**→ Continue to `04-Model-Families-Tradeoffs.md`**
