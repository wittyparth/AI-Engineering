# How This Course Works

## Per-Phase Structure

Each phase follows this loop:

```
┌─────────────────────────────────────────┐
│  Concept Files (read + take notes)      │
│         ↓                               │
│  Drill(s) (30 min, focused)             │
│         ↓                               │
│  Cornerstone Project (build + ship)     │
│         ↓                               │
│  Debug Challenge (fix broken code)      │
│         ↓                               │
│  Design Review (self + rubric)          │
│         ↓                               │
│  Reflection (what would you do diff)    │
│         ↓                               │
│  Gate Check (pass to proceed)           │
└─────────────────────────────────────────┘
```

## Gates

You must pass each gate before moving on. Gates are practical:
- "Your RAG system achieves >0.7 faithfulness on the test set"
- "Your agent handles 3 consecutive failures gracefully"
- "Your deployment serves 100 concurrent requests with <2s latency"

No soft gates. No "vibes-based progress."

## Difficulty Dial

Every concept is rated:
- **Easy** — builds on what you know
- **Medium** — requires new mental models
- **Hard** — genuinely difficult, may need multiple attempts
- **Trap** — looks easy but has hidden depth (called out explicitly)

## Senior Engineer Callouts

Look for `🔴 Senior` markers. These distinguish junior vs. senior thinking:

```python
# 🔴 Junior: catches the exception
# 🔴 Senior: knows which exceptions are worth catching,
#            designs the system so certain failures are impossible,
#            and explicitly decides what to surface to the user
```

## Portfolio Log

Every project goes into your portfolio log with:
1. What you built
2. Key design decisions
3. What broke and how you fixed it
4. What you'd do differently
5. Cost analysis (tokens, latency, $)
