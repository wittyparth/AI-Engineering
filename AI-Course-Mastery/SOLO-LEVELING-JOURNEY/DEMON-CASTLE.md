# 🏰 DEMON CASTLE — Boss Battle Challenges

```
[SYSTEM]─────────────────────────────────────
  The Demon Castle is not for the weak.
  
  Unlike dungeons where you learn new things,
  the Demon Castle tests your ability to
  COMBINE everything you already know.
  
  Each floor is a challenge that requires
  concepts from MULTIPLE gates.
  
  Enter only when you're ready.
  Leave only when you've conquered.
─────────────────────────────────────────────
```

---

## How It Works

1. **Demon Castle floors appear after certain gates** — you can't attempt a floor until you've completed the required gates
2. **Each floor is 90 min (1 sprint)** — time-boxed
3. **No hand-holding** — each challenge describes WHAT to build, not HOW
4. **You must type every line** — no copying from past projects
5. **Each floor completed = +200 XP** and a **Demon Castle badge** in your tracker
6. **RED GATE days** are the perfect time to attempt a Demon Castle floor

---

## FLOOR 1: The Crossroads

*Available after: Gate 1 (The First Gate)*
*Requires: Gateway project + basic prompt knowledge*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
You built a Multi-Provider LLM Gateway in Gate 1.
You learned about system prompts in Gate 2.

CHALLENGE:
Your Gateway currently routes based on model name.
Add PROMPT-BASED ROUTING.

When a request includes a system prompt with certain
keywords, automatically route to the best provider.

Example:
- System prompt mentions "code" → route to Claude
- System prompt mentions "creative" → route to GPT-4
- Default: use your standard routing logic

Requirements:
- Don't look at your Gateway code from Gate 1
- Rebuild the routing logic from memory
- Add the keyword-based prompt analysis
- Test it with at least 3 scenarios
```

**Gate Pass Checklist:**
- [ ] Code works without errors
- [ ] I didn't copy from old Gateway code
- [ ] At least 3 routing scenarios work
- [ ] I can explain the routing logic in 2 sentences

---

## FLOOR 2: The Memory Trial

*Available after: Gate 3 (The Compass Tower)*
*Requires: Gateway + embeddings knowledge*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
Your Gateway routes requests to providers.

CHALLENGE:
Add SEMANTIC ROUTING using embeddings.

Instead of keyword matching on prompts:
1. Convert the incoming query to an embedding
2. Compare against pre-computed embeddings of "query types"
3. Route to the provider best suited for that type

Here's the twist: You don't have a vector database yet.
Implement cosine similarity FROM SCRATCH (no libraries).

Then build the semantic router.

Test scenarios:
- "What's the weather in Tokyo?" → fast/cheap provider
- "Write a poem about AI" → creative provider  
- "Explain quantum computing" → deep/accurate provider
```

**This tests:** Embedding intuition + Gateway architecture + coding from scratch

---

## FLOOR 3: The Hall of Mirrors

*Available after: Gate 4 (The Archive)*
*Requires: RAG knowledge + Gateway*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
You now know RAG. You have a Gateway.

CHALLENGE:
Build a RAG-AWARE GATEWAY.

Your Gateway should now:
1. Accept a query + a set of documents
2. Chunk the documents (implement chunking, don't use a library)
3. Generate embeddings for each chunk
4. Retrieve the top-3 most relevant chunks
5. Append them to the query as context
6. Route to the appropriate provider
7. Return the answer + which chunks were used

THE TWIST:
Add a "no relevant context" handler.
If retrieval confidence is below 0.5,
return: "I couldn't find relevant information
in the provided documents" instead of guessing.
```

---

## FLOOR 4: The Agent's Crucible

*Available after: Gate 6 (The Conductor's Hall)*
*Requires: Agent + RAG + Gateway knowledge*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
Build an AGENT that uses your Gateway.

The agent should:
1. Accept a research question
2. Decide whether it needs more information
3. If yes → use your Gateway to search (simulate search)
4. If the answer is insufficient → refine the query
5. Keep going until confident or max 3 iterations
6. Return the final answer + iteration count

THE TWIST:
Implement the ReAct loop FROM MEMORY.
No looking at your Gate 6 notes.
Thought → Action → Observation → Thought...
```

---

## FLOOR 5: The Evaluation Gauntlet

*Available after: Gate 7 (The Judge's Court)*
*Requires: Eval knowledge + any previous project*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
Take ANY project you've built (Gateway, RAG bot, Agent).
Build an evaluation pipeline for it in 90 minutes.

Your eval pipeline must:
1. Have at least 5 test cases
2. Use LLM-as-judge (a different model than your project uses)
3. Measure at least 2 metrics (accuracy, relevance, etc.)
4. Generate a report: pass/fail per test case + overall score
5. Identify 1 regression (what's worse than expected)

THE TWIST:
Make one of your test cases adversarial.
Try to make your project fail.
If it doesn't fail — make the test harder until it does.
```

---

## FLOOR 6: The Shield Wall

*Available after: Gate 8 (The Shield Fortress)*
*Requires: Guardrails + Gateway knowledge*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
Add GUARDRAILS to your Gateway.

Your Gateway currently accepts any input and sends it to providers.

CHALLENGE:
- Add INPUT guardrails: block prompt injection attempts
- Add OUTPUT guardrails: block harmful content
- Add PII redaction: detect and mask emails, phones, SSNs
- All 3 must work independently (you can turn each on/off)

Implement the guardrails from scratch:
- For injection detection: keyword patterns + LLM-based check
- For content safety: keyword patterns
- For PII: regex patterns

THE TWIST:
Create a test suite with 5 safe + 5 malicious inputs.
All 5 safe must pass through. All 5 malicious must be blocked.
```

---

## FLOOR 7: The Sculptor's Challenge

*Available after: Gate 9 (The Sculptor's Forge)*
*Requires: Fine-tuning knowledge*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
You know how fine-tuning works.
Now make the DECISION.

Given these 3 scenarios:
1. A customer support bot that needs to answer in a specific tone
2. A code generator that needs to follow your company's style guide
3. A translation system that handles medical terminology

For EACH scenario:
- Would you fine-tune, use RAG, or improve prompting?
- Justify your answer in 2-3 sentences
- What data would you need?
- What evaluation would you run to prove it worked?

THE TWIST:
Write the training dataset format for scenario 1.
(You don't need to train a model — just show you know
what the dataset should look like.)
```

---

## FLOOR 8: The Architect's Summit

*Available after: Gate 10 (The Architect's Spire)*
*Requires: Full course knowledge (Gates 0-10)*
*Time-box: 90 min*
*XP: +200*

### The Challenge

```
Design a production system that uses EVERYTHING you've learned.

REQUIREMENTS:
Build a "Production AI Assistant" that:
- Answers questions about your company's documentation
- Uses RAG to retrieve relevant context
- Has guardrails to prevent misuse
- Has evaluation to track quality
- Can fall back to a different provider
- Caches frequent queries
- Monitors cost, latency, and error rate

Write the architecture document:
- System diagram (text-based)
- Component descriptions
- Data flow
- Provider routing strategy
- Caching strategy
- Monitoring dashboard layout
- Cost estimate (per 10K queries)

THE TWIST:
Identify the TOP 3 things that would break first
in production and your mitigation for each.
```

---

## FLOOR 9: The Final Gate (Requires All Gates)

*Available after: Gate 10*
*Requires: ALL knowledge from Gates 0-10*
*Time-box: 2 sprints (1 full day)*
*XP: +500*

### The Challenge

```
BUILD YOUR CAPSTONE.

This is not a drill. This is the final test.
Everything you've learned culminates here.

You've already planned your Capstone in Gate 11.

For THIS challenge, build the CORE FEATURE
of your Capstone in 1 day.

Not the full project. Just the core AI pipeline.

Requirements:
- All code must be typed (no copy-paste)
- No looking at previous projects
- Must use at least 3 concepts from different gates
- Must handle errors gracefully
- Must output something demonstrable

After 1 day, answer:
┌──────────────────────────────────────┐
│ What was the hardest part?           │
│                                      │
│ What concept from an earlier gate     │
│ was most useful?                     │
│                                      │
│ What am I still confused about?      │
└──────────────────────────────────────┘
```

---

## Demon Castle Records

```
┌────────────────────────────────────────────┐
│          DEMON CASTLE PROGRESS              │
├────────────────────────────────────────────┤
│                                            │
│ Floor 1 — The Crossroads      ☐ / +200 XP │
│ Floor 2 — The Memory Trial     ☐ / +200 XP │
│ Floor 3 — Hall of Mirrors      ☐ / +200 XP │
│ Floor 4 — Agent's Crucible     ☐ / +200 XP │
│ Floor 5 — Eval Gauntlet        ☐ / +200 XP │
│ Floor 6 — The Shield Wall      ☐ / +200 XP │
│ Floor 7 — Sculptor's Challenge ☐ / +200 XP │
│ Floor 8 — Architect's Summit   ☐ / +200 XP │
│ Floor 9 — The Final Gate       ☐ / +500 XP │
│                                            │
│ Total XP Available: +2,100                 │
│ Total XP Earned: ________                  │
│                                            │
│ Title Earned: ________                     │
│ (0-1 floors: Apprentice)                   │
│ (2-4 floors: Knight)                       │
│ (5-7 floors: Elite)                        │
│ (8-9 floors: Demon Lord)                   │
└────────────────────────────────────────────┘
```

---

```
═══════════════════════════════════════════
  The Demon Castle does not forgive.
  The Demon Castle does not forget.
  
  But those who conquer it
  become something more than hunters.
  
  They become legends.
═══════════════════════════════════════════
```
