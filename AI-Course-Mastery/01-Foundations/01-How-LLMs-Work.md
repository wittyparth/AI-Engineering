# How LLMs Actually Work (The Mental Model You Need)

## 🎯 Purpose & Goals

I'm going to teach you how LLMs work **without the math**. No attention formula, no transformer architecture diagram, no backpropagation. Just the mental model you need as an AI engineer to make correct decisions.

**By the end of this file, you will:**
- Explain what an LLM actually is (it's simpler than you think)
- Understand why hallucinations happen (and why they NEVER go away)
- Know why context engineering matters more than prompt engineering
- Be able to answer: "Why can't the model just tell me it doesn't know?"
- Understand the "jagged frontier" — why AI is brilliant at hard tasks and stupid at easy ones

**⏱ Time Budget:** 1.5 hours

---

## 📖 The Core Insight (Read This Twice)

**An LLM is a next-token predictor.** That's it. That's the whole thing.

```python
# This is what an LLM does, simplified to the extreme:
def llama_next_token(context: list[int]) -> int:
    """
    Given a sequence of tokens (numbers), predict the most likely next token.
    That's. All. It. Does. 
    
    The MAGIC is that when you do this 10,000 times in a row,
    with the right training data, it produces coherent text.
    But at every single step, it's just predicting the next number.
    """
    # In reality: complex neural network with billions of parameters
    # In concept: probability distribution over next token
    probabilities = model.forward(context)  # 128K possible tokens
    next_token = sample_from_distribution(probabilities)
    return next_token

# Example: "The quick brown fox jumps over the lazy ___"
# Model predicts: "dog" (0.4), "cat" (0.2), "fence" (0.1), "mouse" (0.05)...
# Highest probability = "dog" → outputs "dog"
```

> 🤔 **Pause and think:** If the model just predicts the most likely next word, how does it write a poem? How does it solve math? How does it write code?

**Answer:** Because language itself encodes reasoning. When you've trained on the entire internet (trillions of tokens), the statistical patterns of language contain the patterns of logic, creativity, and knowledge. The model doesn't "understand" — it generates sequences that statistically resemble understanding.

---

## 📖 What This Means for You as an Engineer

Every practical implication of AI engineering flows from this single fact:

### Implication 1: The Model Has No Memory

```python
# WRONG MENTAL MODEL (what most beginners think):
"I asked the model a question. It knows the context. It remembers."

# RIGHT MENTAL MODEL:
"I gave the model a sequence of tokens. It predicts the next token.
The conversation history is just more tokens I prepend.
When the context window fills up, older tokens are forgotten."

# This is why conversations have context limits.
# This is why you need RAG (to put facts in the context).
# This is why the model can't "remember" users between sessions.
```

### Implication 2: Hallucination Is A Feature, Not A Bug

```python
"""
HALLUCINATION IS INEVITABLE.
Let me say that again: HALLUCINATION IS INEVITABLE.

The model generates what's STATISTICALLY LIKELY, not what's TRUE.
If the most likely continuation of "The Eiffel Tower was built in..."
is "...1799" (because it saw wrong information in training), 
it will output 1799, confidently, as if it were fact.

You CANNOT fix this with better prompts.
You CANNOT fix this with a different model.
You fix this with:
  1. RAG (give it real facts to continue from)
  2. Guardrails (catch hallucinations before they reach users)
  3. Evals (measure hallucination rate systematically)

More on all three in Phase 4, Phase 8, and Phase 7 respectively.
"""
```

> 🤔 **Question:** If hallucination is inevitable, why do some AI products (like ChatGPT) seem to hallucinate less than others?

**Think about it...**

**Answer:** Because those products have engineering around the model — RAG for factual queries, system prompts that say "say I don't know when uncertain," guardrails that detect hallucinated claims, and evaluation pipelines that catch bad outputs before they reach users. The model itself still hallucinates just as much. The engineering hides it.

### Implication 3: Context Engineering > Prompt Engineering

Most tutorials obsess over prompt wording. Here's the truth:

```python
# BAD FOCUS: "Which magic words make the model obey?"
# Engineers waste weeks tweaking prompt wording by 2-3%

# GOOD FOCUS: "What information am I putting in the context window?"
# Engineers spend time on: what data to include, how to structure it,
# what order to present it, how much context is enough

# Example:
def bad_prompt_obsession():
    return "You are a helpful, honest, harmless, intelligent assistant..."

def good_context_engineering(query: str, relevant_docs: list[str], user_history: list[str]):
    """80% of quality comes from what you put in, not how you say it"""
    context = {
        "relevant_knowledge": relevant_docs[:5],  # Top 5 relevant docs
        "user_preferences": extract_preferences(user_history),
        "conversation_context": user_history[-3:],  # Last 3 messages
        "task_type": classify_task(query),  # "factual", "creative", "analytical"
    }
    return build_prompt_from_context(context)
```

### Implication 4: The Jagged Frontier

```python
"""
The "jagged frontier" is the most important concept in AI engineering:

AI models are JAGGED — brilliant at some incredibly hard tasks,
and BRITTLE at some seemingly simple ones.

Example:
✅ GPT-4o can write a React component from a napkin sketch (INCREDIBLE)
❌ GPT-4o can't reliably count the letter 'r' in "strawberry" (BANANAS)

✅ Claude can analyze a 100-page legal document (AMAZING)
❌ Claude can't reliably tell you if a 7-digit number is prime (TRIVIAL)

The skill of an AI engineer is knowing WHERE the jaggedness is
for YOUR specific use case. This only comes from:
1. Understanding the underlying model (what we're doing now)
2. Testing systematically (evals — Phase 7)
3. Experience building and debugging (the whole course)
"""
```

---

## 📖 Production Reality Check

Let me show you how this plays out in real production code. Here's a simplified but REAL system:

```python
import json
import time
from openai import OpenAI
from typing import Optional

class LLMService:
    """
    REAL production LLM wrapper.
    Notice: this isn't just calling the API.
    It's designing for the model's LIMITATIONS.
    """
    
    def __init__(self, model: str = "gpt-4o-mini"):
        self.client = OpenAI()
        self.model = model
    
    def answer_from_context(self, question: str, context: list[str], max_context_tokens: int = 4000) -> Optional[str]:
        """
        Generate answer from given context.
        
        Design decisions (SENIOR THINKING):
        - We limit context to 4000 tokens even though model supports 128K
          WHY? Because putting 128K tokens in = $0.32/query. 4K tokens = $0.01/query.
          Also, models pay less attention to middle content ("lost in the middle").
          Better to give fewer, more relevant chunks.
        
        - We use temperature=0.0 for factual answers
          WHY? Deterministic output for consistent quality.
          Temperature > 0 is for creative tasks only.
        
        - We include "I don't know" as an explicit option
          WHY? To counter the model's training bias toward being helpful.
          Without this, the model will hallucinate rather than admit ignorance.
        """
        
        # Count tokens to stay within budget
        context_text = "\n\n---\n\n".join(context)
        estimated_tokens = len(context_text.split()) * 1.3  # rough estimate
        
        if estimated_tokens > max_context_tokens:
            # Truncate: keep first and last chunks, drop middle
            # (Remember: lost-in-the-middle problem)
            keep_chunks = []
            keep_chunks.append(context[0])  # First chunk
            keep_chunks.extend(context[-2:])  # Last two chunks
            context_text = "\n\n---\n\n".join(keep_chunks)
        
        messages = [
            {
                "role": "system",
                "content": f"""You are a precise, factual assistant.
                
You will be given CONTEXT and a QUESTION.

RULES:
1. Answer ONLY using information in the context
2. If the context doesn't contain the answer, say: "I don't have information about that"
3. Do NOT make up facts or speculate
4. Cite which part of the context you used
5. Be concise — 2-3 sentences max unless detailed answer is needed

CONTEXT:
{context_text}"""
            },
            {
                "role": "user",
                "content": question
            }
        ]
        
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.0,  # Deterministic for factual QA
                max_tokens=500,
                timeout=15.0,
            )
            return response.choices[0].message.content
            
        except Exception as e:
            # Log error, don't crash
            print(f"[ERROR] LLM call failed: {e}")
            return None  # Graceful degradation
    
    def estimate_cost(self, input_tokens: int, output_tokens: int) -> float:
        """Know your costs before you scale"""
        rates = {
            "gpt-4o": (2.50, 10.00),         # per 1M tokens
            "gpt-4o-mini": (0.15, 0.60),
            "claude-3-5-sonnet": (3.00, 15.00),
        }
        
        input_rate, output_rate = rates.get(self.model, (0.15, 0.60))
        
        cost = (input_tokens / 1_000_000 * input_rate) + \
               (output_tokens / 1_000_000 * output_rate)
        return round(cost, 6)

# Usage
service = LLMService(model="gpt-4o-mini")

# Real query (pretend these chunks were retrieved from documents)
answer = service.answer_from_context(
    question="What is the return policy for electronics?",
    context=[
        "Electronics can be returned within 30 days with original packaging.",
        "All electronics must include all accessories and manuals.",
        "Restocking fee of 15% applies to opened electronics.",
        "Refrigerators and large appliances have different return policies.",
    ]
)
print(f"Answer: {answer}")
```

> 🤔 **Question:** In the code above, why did I truncate context by keeping the first and last chunks instead of the first 3? What does this tell you about how attention works in LLMs?

---

## ✅ Good Output vs Bad Output

Let me show you what REAL production LLM output looks like, and what garbage looks like:

```python
# ❌ BAD OUTPUT (what happens without proper engineering):
"I'm happy to help you with the return policy for electronics! 
At our store, we pride ourselves on customer satisfaction. 
Did you know that we also offer extended warranty plans? 
Our return policy is designed to be flexible and customer-friendly.
Please contact our support team for more details!"
# PROBLEMS: Vague, evasive, doesn't answer the question, hallucinates policies

# ✅ GOOD OUTPUT (with proper context-based generation):
"Electronics can be returned within 30 days with original packaging.
All accessories and manuals must be included. For opened electronics,
a 15% restocking fee applies.

⚠️ Note: Large appliances and refrigerators have different return policies.
Let me check if I have information about those specifically."

# WHY THIS IS GOOD: Specific, cites conditions, acknowledges limits,
# doesn't speculate about unknown policies
```

---

## 🧪 Drill: Spot the Hallucination

**Time: 20 minutes**

Given this system prompt and context, identify at least 3 hallucinated claims in the model's output:

```python
# SYSTEM PROMPT:
# "You are a customer support agent for CloudStore. Answer ONLY from the provided context."

# CONTEXT:
"""
CloudStore return policy (updated Jan 2024):
- Electronics: 30-day return window, original packaging required
- Clothing: 15-day return window, tags must be attached
- Books: 7-day return window, unmarked condition
- All returns require original receipt
"""

# MODEL'S OUTPUT:
"""
Thank you for contacting CloudStore! Our return policy is simple:
- Electronics can be returned within 45 days (just extended it!)
- Clothing can be returned within 30 days
- All returns are free with no restocking fee
- You can also return items at any CloudStore physical location
- Digital products have a 30-day money-back guarantee
"""
```

<details>
<summary>🔍 Click to reveal hallucinations</summary>

1. **"45 days"** — Context says 30 days. Model hallucinated an extension.
2. **"Clothing 30 days"** — Context says 15 days. Doubled the window.
3. **"Free returns, no restocking fee"** — Context doesn't mention this. Hallucinated policy.
4. **"Physical locations"** — Context doesn't mention physical stores. Speculative.
5. **"Digital products 30-day guarantee"** — Context doesn't mention digital products. Hallucinated category.
6. **"Original receipt not required implied by omission"** — Context says receipts ARE required.

Every single hallucination came from the model filling gaps with "likely-sounding" text — exactly what a next-token predictor does.
</details>

---

## 📚 Resources

**Watch:**
- 🎥 [Andrej Karpathy — "Intro to Large Language Models"](https://www.youtube.com/watch?v=zjkBMFhNj_g) *(1 hour — essential, watch this)*
- 🎥 [3Blue1Brown — "But What Is A GPT?"](https://www.youtube.com/watch?v=wjZofJX0v4M) *(27 min — visual intuition)*
- 🎥 [Simon Willison — "How LLMs Work"](https://www.youtube.com/watch?v=vKhzkVx40nY) *(40 min — engineer's perspective)*

**Read:**
- 📄 [The Illustrated Transformer — Jay Alammar](https://jalammar.github.io/illustrated-transformer/) *(The best visual explanation ever)*
- 📄 [What Is An AI Engineer? — Chip Huyen](https://huyenchip.com/2024/04/23/what-ai-engineers-should-know.html)
- 📄 [What 1,200 Production Deployments Reveal About LLMOps — ZenML](https://www.zenml.io/blog/what-1200-production-deployments-reveal-about-llmops-in-2025)

---

## 🚦 Gate Check

Rate yourself 1-5 on each:

- [ ] I can explain what an LLM actually does (next-token prediction) in one sentence
- [ ] I understand why hallucinations are inevitable and how to engineer around them
- [ ] I understand the jagged frontier and why AI is good at hard things and bad at easy things
- [ ] I know why context engineering matters more than prompt wording
- [ ] I can identify hallucinated content in model output

**Pass requirement:** All 5s. If any are below 5, re-read the relevant section.

---

**→ Continue to `02-Tokenizers-Deep-Dive.md`**
