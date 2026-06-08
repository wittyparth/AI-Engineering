# Project 9: Advanced Financial RAG System

- **Tier:** 3 — Real Constraints
- **Project #:** 9 of 12
- **Tech Stack:** Qdrant (hybrid search), LangGraph (agentic retrieval), vision model (for tables), RAGAS/DeepEval (evaluation), FastAPI
- **Concepts:** Agentic RAG, query rewriting, hybrid search, vision for table extraction, citation enforcement, answer abstention (I don't know), faithfulness evaluation
- **Quality Gate:** ✅ APPROVED when faithfulness score > 0.95 on test set, zero hallucinated figures in 100-test evaluation, and system correctly abstains from answering on 20% of "unanswerable" test queries

---

## Phase 1: Brief (Priya)

> *Priya: "This client scares me. If this system hallucinates a number, people could lose money. Do not screw this up."*

**Client:** **Atlas Capital** — a financial services firm with regulatory documents, earnings reports, market analysis, and internal research. Compliance-sensitive. High-stakes.

**Their problem:** Analysts need to query earnings reports and regulatory filings. Questions like: *'What was Q3 revenue for the industrial division?'* or *'What are the risk factors listed in the 10-K?'* The answers must be EXACT — the exact figure from the exact page of the exact document.

**Why RAG alone isn't enough:** Standard RAG retrieves chunks, but:
- Financial documents have TABLES. Chunking tables destroys their structure
- Cross-document questions need MULTIPLE retrieval passes: *'How did revenue growth compare between the industrial and consumer divisions in 2024?'*
- Hallucinating a figure is catastrophic. The system must ABSTAIN when uncertain
- Regulatory documents cross-reference each other — *'as described in Note 14 of the 2023 Annual Report'*

**What we're building:** An advanced RAG system with:
- **Agentic retrieval** — the system decides HOW to search (multiple passes, different indexes)
- **Vision-based table extraction** — render PDF pages as images, use a vision model
- **Citation enforcement** — every claim must have a verifiable source document + page number
- **Answer abstention** — if confidence < threshold, say "I cannot answer this from the available documents"
- **Zero-hallucination guarantee** — no made-up figures, no fake citations

**Non-negotiable:**
- Faithfulness score > 0.95 on held-out test set
- Zero hallucinated figures in 100-test evaluation
- Must correctly abstain on at least 20% of "unanswerable" test queries (queries about topics not in the documents)
- Every answer must be traceable to a specific document + page number

**Deadline:** 12 days. They have a quarterly review where this will be demonstrated to the compliance team.

**Definition of done:** An analyst types a question → system returns a cited answer where every claim maps to a verifiable source. If it can't answer, it says so. Zero hallucinations.

---

## Phase 2: Learning Path (Maya)

> *Maya is serious. "This is the hardest RAG project you'll ever build. It's also the most important one."*

### Why This Is Hard

"Standard RAG (Project 4) fails on financial documents for 4 reasons:

1. **Tables.** Chunking a table destroys the row-column relationships. You need vision-based extraction.
2. **Multi-hop queries.** *'Compare industrial vs consumer revenue in 2024'* needs TWO retrieval passes — one for each division. Standard RAG retrieves chunks for the whole question at once.
3. **Cross-references.** *'As described in Note 14'* means the LLM needs to resolve references across documents.
4. **Zero hallucination.** If a RAG system hallucinates a figure in a legal Q&A, the lawyer might find it funny. If it hallucinates in financial reporting, people lose money.

### Learning Order (Scratch-First)

**Step 1: Hybrid search with Qdrant (from Project 4 — improve it).**
You already know this. Improve chunking for financial documents (smaller chunks, more metadata).

**Step 2: Vision-based table extraction.**
When a PDF page contains a table, render the page as an image and use a vision model (GPT-4o vision) to extract the table as structured data. Store as Markdown alongside the text.

> *"This changes the game. Instead of losing table data to chunking, you extract it perfectly as structured data. A $0.01 vision call per page saves you from hallucinated financial figures."*

**Step 3: Query rewriting + decomposition.**
Before retrieval, rewrite multi-part queries into multiple search queries. *'Compare industrial vs consumer revenue'* → search for 'industrial division Q3 revenue' AND 'consumer division Q3 revenue' separately.

**Step 4: Agentic retrieval.**
Instead of one-shot retrieval, build an agent (LangGraph) that:
1. Understands the question → decides HOW to search
2. Executes one or more searches (different queries, different filters)
3. Reviews the results → decides if it needs more information
4. When satisfied → passes context to the answer generator
5. Answers with citations + abstains if uncertain

**Step 5: Citation verification.**
After generation, verify every citation: does the cited passage actually say what the answer claims? If a citation is unverifiable, reject the answer and regenerate.

**Step 6: Evaluation with financial-specific metrics.**
Faithfulness > 0.95. Figure hallucination rate = 0. Abstention accuracy > 80%.

### Memory Triage

**Memorize cold:**
- Agentic RAG loop: `analyze → search → review → search more → generate → verify → output`
- Citation format: `[Doc: filename, Page: X, Section: "text preamble..."]`
- Answer abstention prompt: *"You must NOT answer if the context does not contain sufficient information. Say 'I cannot answer based on the available documents.'"*
- The vision extraction pattern: `page_image → vision_model → structured_markdown_table`

**Understand deeply:**
- Why abstention is harder than answering — *"LLMs are trained to be helpful. Saying 'I don't know' goes against their training. You need to make abstention EASIER than guessing."*
- Why citation verification is a closed loop — *"you verify citations using the same context the LLM used. It catches hallucinated page numbers and fabricated passages. But it WON'T catch the case where the passage exists but is misinterpreted."*
- Why financial RAG needs temperature 0 AND structured output — *"temperature 0 for deterministic answers. Structured output for parseable citations. Both are non-negotiable."*

### First Concrete Step

> "Take your Project 4 RAG pipeline. Add metadata to each chunk: `document_name`, `page_number`, `section_title`, `has_table`. Run a financial query through it. See where it fails — what does a standard RAG system do wrong with a query like 'what was Q3 revenue for the industrial division?'"

---

## Phase 3: The Build

> *This is RAG at its most demanding. Every component needs to be tighter than before.*

### Milestone 1: Enhanced Financial Document Processing

Improve chunking and metadata for financial documents:

```python
import fitz
from openai import OpenAI

vision_client = OpenAI()

def extract_financial_page(pdf_path: str, page_num: int) -> dict:
    """Extract text AND detect/parse tables from a financial PDF page."""
    doc = fitz.open(pdf_path)
    page = doc[page_num]
    
    # Text extraction
    text = page.get_text()
    
    # Check if page likely contains a table
    # Simple heuristic: high tab/whitespace density or structured layout
    if detect_table(text):
        # Render page as image for vision extraction
        pix = page.get_pixmap(dpi=200)
        img_bytes = pix.tobytes("png")
        
        # Use vision model to extract table as structured data
        response = vision_client.chat.completions.create(
            model="gpt-4o-mini",  # vision-capable
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": "Extract this financial table as a Markdown table. Preserve all numbers, labels, and column headers exactly."},
                        {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_to_b64(img_bytes)}"}}
                    ]
                }
            ]
        )
        table_md = response.choices[0].message.content
        text += f"\n\n[TABLE FROM PAGE {page_num}]:\n{table_md}"
    
    return {
        "text": text,
        "page": page_num,
        "has_table": detect_table(text)
    }
```

**Expected stuck point:** Vision extraction costs $0.01 per page. A 100-page PDF costs $1 just for table extraction. For a 500-document corpus, this gets expensive.

**Maya's Socratic question:**
> *"Your vision extraction costs $50 for a 500-page filing. Is this worth it? What if you only extract tables on pages that are LIKELY to contain financial figures?"*

> They should: use a cheap classifier (keyword-based: 'revenue', 'earnings', 'net income', '$' in the raw text) to detect table-having pages BEFORE calling the vision model. Only 20-30% of financial pages have tables. This saves 70-80% of vision costs.

### Milestone 2: Agentic Retrieval

Build a LangGraph agent that decides how to search:

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, List

class RetrievalState(TypedDict):
    question: str
    search_queries: List[str]
    retrieved_chunks: List[dict]
    needs_more_info: bool
    final_context: str
    can_answer: bool
    reasoning: str

def analyze_question(state: RetrievalState) -> dict:
    """Break down the question into search strategies."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": """Analyze this financial question. Break it into sub-questions if needed. 
             Determine: (1) What entities/documents to search? (2) What time period? (3) Single or multi-hop?"""},
            {"role": "user", "content": state["question"]}
        ],
        response_format=SearchPlan  # Pydantic model
    )
    plan = response.choices[0].message.parsed
    return {"search_queries": plan.queries}

def search_documents(state: RetrievalState) -> dict:
    """Execute hybrid search for each query."""
    all_chunks = []
    for query in state["search_queries"]:
        chunks = qdrant_hybrid_search(query, top_k=5)
        all_chunks.extend(chunks)
    return {"retrieved_chunks": all_chunks}

def review_results(state: RetrievalState) -> dict:
    """Review retrieved chunks. Decide if more info needed."""
    context = format_chunks(state["retrieved_chunks"])
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Review the retrieved information. Can you answer the question with this? If not, what's missing?"},
            {"role": "user", "content": f"Question: {state['question']}\n\nRetrieved context:\n{context}"}
        ]
    )
    review = response.choices[0].message.content
    needs_more = "cannot answer" in review.lower() or "missing" in review.lower()
    return {"needs_more_info": needs_more, "reasoning": review}

def decide_next(state: RetrievalState) -> str:
    """Route to answer generation or more search."""
    if state["needs_more_info"]:
        return "search"  # try again with refined queries
    return "generate"

# Build the graph
builder = StateGraph(RetrievalState)
builder.add_node("analyze", analyze_question)
builder.add_node("search", search_documents)
builder.add_node("review", review_results)

builder.add_edge(START, "analyze")
builder.add_edge("analyze", "search")
builder.add_edge("search", "review")
builder.add_conditional_edges("review", decide_next, {
    "search": "search",  # loop back (with max iterations guard)
    "generate": END
})

agent = builder.compile()
```

**Expected stuck point:** The agent loops forever, making 10+ retrievals and never being satisfied. Without a MAX_RETRIEVALS guard, it uses infinite tokens.

**Maya's Socratic question:**
> *"Your agent made 8 retrieval passes for the question 'What were Q3 earnings?' and spent $0.40 on API calls. At what point does the agent decide 'I have enough information' vs 'I need more'?"**

> They should: enforce a hard limit of 3 retrieval passes, implement a confidence threshold for "I can answer this confidently," and force a final answer after 3 passes even if confidence is low (with an honesty caveat).

### Milestone 3: Answer Abstention

The mechanism that makes the system say "I don't know":

```python
class RAGAnswer(BaseModel):
    can_answer: bool
    answer: Optional[str] = None
    citations: List[Citation] = []
    confidence: float = Field(..., ge=0.0, le=1.0)
    abstention_reason: Optional[str] = None

class Citation(BaseModel):
    document: str
    page: int
    passage: str  # exact text that supports the claim

ABSTENTION_PROMPT = """You are a financial document assistant. Your job is to answer questions based ONLY on the provided context.

CRITICAL RULES:
1. If the context does NOT contain sufficient information to answer, say you cannot answer.
2. If you are not confident about a specific figure or fact, do NOT guess.
3. Every claim MUST have a supporting citation from the context.
4. If you cannot provide exact figures from the context, say 'I cannot find the exact figure in the available documents.'

Context:
{context}

Question:
{question}

Answer in the structured format provided. Set can_answer=false if you cannot answer confidently."""

def answer_with_abstention(question: str, context_chunks: List[dict]) -> RAGAnswer:
    context = format_chunks(context_chunks)
    
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": ABSTENTION_PROMPT.format(context=context, question=question)}
        ],
        response_format=RAGAnswer,
        temperature=0.0
    )
    
    result = response.choices[0].message.parsed
    
    # Post-processing: verify citations
    if result.can_answer:
        for citation in result.citations:
            if not verify_citation(citation, context_chunks):
                result.can_answer = False
                result.abstention_reason = f"Unverifiable citation: {citation.document} page {citation.page}"
                break
    
    return result
```

**Expected stuck point:** The LLM almost never sets `can_answer=false` — it's too helpful. Even with explicit instructions, it prefers to guess.

**Maya's Socratic question:**
> *"Your system claims it can answer 95% of questions. But 30% of those answers rely on weak evidence. Your abstention mechanism isn't working. Why?"*

> They should realize: (1) the RLHF-trained LLM has a strong "be helpful" bias, (2) zero temperature doesn't fix this — the bias is in the model, not the sampling, (3) they need to make abstention the DEFAULT and only answer when confidence is HIGH. Change the mindset from "should I answer?" to "should I NOT answer?".

### Milestone 4: Citation Verification

```python
def verify_citation(citation: Citation, chunks: List[dict]) -> bool:
    """Verify that a citation actually says what the answer claims."""
    # Find the chunk matching this citation
    matching_chunks = [
        c for c in chunks 
        if c.payload["source"] == citation.document 
        and c.payload["page"] == citation.page
    ]
    
    if not matching_chunks:
        return False  # cited document/page doesn't exist in context
    
    chunk_text = matching_chunks[0].payload["text"]
    
    # Check if the cited passage appears in the chunk
    if citation.passage not in chunk_text:
        return False  # passage doesn't appear at cited location
    
    return True
```

### Rohan's Mid-Build Interruption

> *Rohan drops in during abstention testing.*

"I asked your system 10 questions that are DEFINITELY not in the documents. Like 'What was Atlas Capital's Bitcoin investment return in 2025?' when there's nothing about crypto in the corpus.

Your system correctly answered 'I cannot answer' for 6 out of 10. For the other 4, it hallucinated answers from vaguely related documents.

**4 out of 10.** That's a 40% hallucination rate on unanswerable questions. Your abstention mechanism is failing. Fix it before you show this to the compliance team."

### Priya's Requirement Change

> *Priya: "Atlas Capital's compliance team wants an audit trail. Every answer must be accompanied by a complete trace: which documents were retrieved, why, how the agent decided what to search for, and the final citation verification results. They need this for regulatory audits (SEC). Two extra days."*

This forces: add a `trace` field to each answer containing the full retrieval history, agent decisions, and verification results.

---

## Phase 4: Review (Rohan)

> *You submit: agentic RAG system, vision-based table extraction, citation verification, abstention mechanism, compliance audit trail, 100-question evaluation report.*

### Decision Documentation Required

1. Your abstention strategy — how you make the model default to "I don't know" and your abstention accuracy numbers
2. Your vision extraction strategy — which pages you extract tables from, cost analysis
3. Your agentic retrieval design — when and why the agent does multiple search passes
4. Citation verification results — false positive rate, false negative rate

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | End-to-end: question → cited answer. Audit trail for every response. |
| 2 | **Edge cases?** | ✅ PASS | Handles tables, multi-hop queries, cross-references. Scanned documents via vision. |
| 3 | **Cost-aware?** | 🔄 REVISE | Your agentic retrieval averages 2.3 search passes per question. That's ~3x the cost of single-pass RAG. For 75% of queries, single-pass is sufficient. Add a classifier that decides whether to go agentic or single-pass. |
| 4 | **Observable?** | ✅ PASS | Audit trail is comprehensive. I can trace every decision. Compliance team will approve. |
| 5 | **Right approach?** | ✅ PASS | Agentic retrieval for complex queries, vision for tables, citation verification, abstention. Correct architecture. |
| 6 | **Decisions justified?** | ✅ PASS | Clear reasoning on every design choice. |
| 7 | **Measurable quality?** | ✅ PASS | Faithfulness 0.96. Zero hallucinated figures. Abstention accuracy: 85% on unanswerable queries. Passes the gate. |

### Verdict: 🔄 REVISE

*Rohan reads your audit trail carefully.*

"One thing:

**Cost optimization for single-hop queries.** Your agent is smart — it makes 2.3 search passes on average. But 40% of your test queries were simple single-hop questions ('what was Q3 revenue') that needed only ONE search pass. You're over-searching on simple queries.

Fix: before entering the agentic loop, classify the question as 'simple' or 'complex.' Simple → single-pass RAG (faster, cheaper). Complex → enter agentic loop. Target: 1.5 average search passes."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "This was the hardest RAG project in the course. Here's what you now know that most AI engineers never learn:

- **Tables break standard RAG.** Chunking destroys structured data. Vision-based extraction is the fix. Any serious document Q&A system needs this.
- **One-shot retrieval is not enough.** Multi-hop queries need multiple retrieval passes. Agentic retrieval — where the system decides HOW to search — is the production pattern for complex domains.
- **Abstention is harder than answering.** Making an LLM say 'I don't know' requires fighting its training. You need structural enforcement (Pydantic with `can_answer` flag) + prompt design + post-processing verification.
- **Citation verification is a safety net, not a magic shield.** You can verify that a citation EXISTS in the context. You CANNOT verify that the LLM INTERPRETED it correctly. This is a fundamental limitation of current RAG.

**What you'll see again:**
- **Agentic retrieval** — You built this yourself. LangGraph's agentic RAG patterns formalize it. You'll see this in every complex RAG system from now on.
- **Vision extraction** — Project 14 (Multi-Agent Content Pipeline) uses vision for content analysis.
- **Safety constraints** — Project 15 (Autonomous Customer Support) has similar abstention requirements.

> *Maya: "Project 10 is a complete change of pace. No RAG. No documents. You're building an agent that reviews code — a GitHub PR reviewer that catches bugs before humans do."*
