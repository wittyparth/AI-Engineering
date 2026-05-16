# Senior vs Junior AI Engineer Mindset

## The Core Difference

| Junior | Senior |
|--------|--------|
| Asks "what code do I write?" | Asks "what problem am I solving?" |
| Learns framework syntax | Understands the problem the framework solves |
| Ships a demo that works on their laptop | Ships a system that works for real users |
| Fixes bugs when they appear | Anticipates failure modes before shipping |
| Optimizes for accuracy | Optimizes for accuracy + cost + latency + reliability |
| Treats AI output as magic | Treats AI as a probabilistic service that needs guardrails |
| Says "the model hallucinated" | Says "I didn't build enough guardrails" |
| Builds features | Owns outcomes |

## Real Examples

### Junior: "I built a RAG system with LangChain"
```python
from langchain.chains import RetrievalQA
qa = RetrievalQA.from_chain_type(llm=llm, retriever=retriever)
result = qa.run("What is our refund policy?")
# Works on their laptop with 3 test documents
# Crashes in production when user asks in Spanish
```

### Senior: "I own a production RAG system"
```python
# Senior asks BEFORE writing code:
# - What languages do users speak? (multilingual retrieval?)
# - What's the latency budget? (< 2s or < 500ms?)
# - What's the cost per query budget?
# - How do we know if retrieval quality degrades?
# - What happens when the vector DB is down?
# - How do we handle queries with no relevant documents?
```

## 🔴 Key Insight: Framework Proficiency is NOT Seniority

You can know every LangChain API and still be junior. You can build from scratch and still be junior. Seniority is about:
1. **Scope** — How much of the system do you own?
2. **Judgment** — Can you make the right tradeoff without being told?
3. **Anticipation** — Do you see failure before it happens?
4. **Impact** — Does your work make the team/company more effective?

## Reflection

- What's the most complex AI system you've built? What did you NOT think about?
- If you had to redesign it for 10,000 users, what would break first?
- What's the difference between "it works" and "it works in production"?
