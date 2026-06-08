# 06 — Evaluating RAG: Measuring What Matters

## 🎯 Purpose & Goals

You've built a RAG system. You've tuned the chunking, the retrieval, the context integration. You've worked through the failure modes. Now comes the hard question: **how do you know it actually works?**

Evaluation is the most overlooked part of RAG development. Most teams ship their RAG system after testing 10-20 queries manually, seeing "good enough" answers, and calling it done. Then they discover in production that the system fails on edge cases, user satisfaction is lower than expected, and they have no data to guide improvements.

This module is about building a **rigorous evaluation framework** for RAG systems — the kind that lets you:
- Know whether a change (new chunk size, different model, prompt tweak) actually improves things
- Detect regressions before they reach users
- Compare different architectures objectively
- Have data-driven conversations about where to invest next

**By the end of this module, you will:**

1. Understand the 4 core RAGAS metrics — what they measure, how they work, and their limitations
2. Build evaluation datasets from scratch (when you have no existing data) and from production logs (when you do)
3. Implement both component-level evaluation (is retrieval good?) and end-to-end evaluation (is the final answer good?)
4. Build an evaluation pipeline that runs automatically, caches results, and tracks changes over time
5. Know how to avoid the most common evaluation traps — metrics that look good but lie

---

### ⏹ STOP. Answer these questions before proceeding.

**Question 1 — The 90% Problem**

*Your RAG system achieves a faithfulness score of 0.92 and an answer relevance score of 0.89 on your evaluation dataset. The team celebrates. You deploy to production. User satisfaction is at 65%.*

*Users complain the answers are "technically correct but not helpful." The metrics said the answers were faithful and relevant. What are the metrics not capturing? What's the gap between "the answer is correct" and "the answer is what the user needs"? How would you design a metric that captures this gap?*

*Dig deeper: When you check the evaluation dataset, you find that the questions in it were written by your own team — and they match the documents closely. Production questions are messier, more indirect, and require combining information across documents. What does this tell you about your evaluation dataset?*

**Question 2 — The Metric That Lies**

*You're comparing two RAG configurations:*
- *Config A: Standard retrieval (top-5, cosine similarity). Faithfulness: 0.94, Context Recall: 0.72*
- *Config B: Hybrid retrieval with MMR (top-8, then MMR-select top-5). Faithfulness: 0.88, Context Recall: 0.91*

*Config A has better faithfulness (the answers stick to the context). Config B has better context recall (the answers use more of the available information). Which configuration is better? What additional information do you need to decide? Is there a scenario where Config A is the right choice and a different scenario where Config B is?*

*Now consider: What if Config B's lower faithfulness is because the broader context contains contradictions, and the model is faithfully reproducing contradictory information? Is that a faithfulness failure or a retrieval failure? How does your evaluation framework distinguish between "the model is wrong" and "the context was bad"?*

**Question 3 — The Evaluation Cost Trap**

*You have 10,000 production queries per day. You want to evaluate every answer for quality. Your evaluation pipeline costs $0.01 per query in LLM-as-judge calls. That's $100/day, $3,000/month just for evaluation — more than your generation costs.*

*What's your sampling strategy? How do you ensure your sample is representative? How do you detect issues that only appear in the 99% of queries you're NOT evaluating? What cheap signals (response length, latency, user feedback) can you use to flag queries for deeper evaluation?*

*And the hard one: If your evaluation uses an LLM judge (like GPT-4), and your RAG system also uses an LLM (GPT-4o-mini, say), you're evaluating a weaker model with a stronger one. But what if you want to use GPT-4o-mini as the judge (for cost)? How do you ensure the judge isn't biased toward answers that "sound like" good RAG answers without actually being correct?*

**After answering, proceed to the deep-dive. Evaluation is a design problem, not a library choice.**

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The 4 RAGAS Metrics — Deep Dive | 40 min |
| Building Evaluation Datasets | 30 min |
| Component-Level Evaluation | 25 min |
| End-to-End Evaluation | 30 min |
| LLM-as-Judge: Design & Biases | 25 min |
| Evaluation Pipeline Architecture | 30 min |
| Regression & CI/CD for RAG | 20 min |
| Code: Full Evaluation Framework | 60 min |
| Code: Synthetic Data Generation | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~6 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Four RAGAS Metrics

RAGAS (RAG Assessment) defines four core metrics that measure different aspects of RAG quality. Let's understand each one deeply — not just what they measure, but how they work, what they assume, and where they fail.

#### Metric 1: Faithfulness

**What it measures:** Are the claims in the generated answer supported by the retrieved context?

**How it works (conceptually):**
1. Extract all atomic claims from the answer
2. For each claim, check if it can be inferred from the provided context
3. Faithfulness = (number of supported claims) / (total claims)

**What a claim looks like:**
```text
Answer: "Our API supports rate limiting of 1000 requests per minute for
free tier users, and 5000 for pro tier. Rate limits reset daily at midnight UTC."

Claims extracted:
1. "API supports rate limiting of 1000 requests per minute for free tier users"
2. "API supports rate limiting of 5000 requests per minute for pro tier users"
3. "Rate limits reset daily at midnight UTC"
```

**Implementation approach:**

```python
"""
faithfulness_eval.py — Measure faithfulness of RAG answers.

Faithfulness = fraction of claims in the answer that are
supported by the retrieved context.
"""

from pydantic import BaseModel, Field
from typing import List
from openai import AsyncOpenAI


class Claim(BaseModel):
    claim: str
    supported: bool = Field(description="Is this claim supported by the context?")
    supporting_evidence: str = Field(
        description="The exact text from context that supports or refutes this claim"
    )


class FaithfulnessResult(BaseModel):
    claims: List[Claim]
    supported_count: int
    total_count: int
    faithfulness_score: float
    unsupported_claims: List[Claim]


class FaithfulnessEvaluator:
    """
    Evaluates how faithful an answer is to the provided context.

    Faithfulness = fraction of claims in the answer that are
    supported by the retrieved context.

    Range: 0.0 (no claims supported) to 1.0 (all claims supported)
    """

    def __init__(self, llm_client: AsyncOpenAI, model: str = "gpt-4o-mini"):
        self.llm_client = llm_client
        self.model = model

    async def evaluate(
        self,
        question: str,
        answer: str,
        contexts: List[str]
    ) -> FaithfulnessResult:
        """
        Evaluate faithfulness of an answer.

        Args:
            question: The user's original question
            answer: The RAG system's answer
            contexts: The retrieved context chunks provided to the LLM

        Returns:
            FaithfulnessResult with score and detailed breakdown
        """

        # Step 1: Extract all claims from the answer
        claims = await self._extract_claims(answer)

        if not claims:
            return FaithfulnessResult(
                claims=[],
                supported_count=0,
                total_count=0,
                faithfulness_score=1.0,  # No claims = vacuously faithful
                unsupported_claims=[]
            )

        # Step 2: Check each claim against the context
        supported = []
        unsupported = []

        for claim in claims:
            result = await self._check_support(claim, contexts)
            if result.supported:
                supported.append(result)
            else:
                unsupported.append(result)

        score = len(supported) / len(claims) if claims else 1.0

        return FaithfulnessResult(
            claims=supported + unsupported,
            supported_count=len(supported),
            total_count=len(claims),
            faithfulness_score=score,
            unsupported_claims=unsupported
        )

    async def _extract_claims(self, answer: str) -> List[str]:
        """Decompose an answer into individual factual claims."""
        response = await self.llm_client.beta.chat.completions.parse(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "Decompose the given answer into individual factual claims. "
                        "Each claim should be a single verifiable statement. "
                        "Break compound sentences into separate claims. "
                        "Ignore opinion, speculation, and meta-commentary."
                    )
                },
                {"role": "user", "content": f"Answer:\n{answer}"}
            ],
            response_model=ClaimsList
        )

        return response.claims

    async def _check_support(
        self,
        claim: str,
        contexts: List[str]
    ) -> Claim:
        """Check if a single claim is supported by the context."""
        context_text = "\n\n---\n\n".join(
            f"[CONTEXT {i+1}]\n{c}" for i, c in enumerate(contexts)
        )

        response = await self.llm_client.beta.chat.completions.parse(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "Determine if the given claim is SUPPORTED, "
                        "CONTRADICTED, or NOT MENTIONED in the provided context. "
                        "A claim is supported if the context directly implies it. "
                        "A claim is contradicted if the context states the opposite. "
                        "A claim is not mentioned if the context neither supports "
                        "nor contradicts it."
                    )
                },
                {
                    "role": "user",
                    "content": (
                        f"Context:\n{context_text}\n\n"
                        f"Claim: {claim}\n\n"
                        f"Is this claim supported? "
                        f"Reply with the exact text that supports "
                        f"or refutes it."
                    )
                }
            ],
            response_model=Claim
        )

        return response


class ClaimsList(BaseModel):
    claims: List[str]


# ============================================================
# Important: Faithfulness Has Subtleties
# ============================================================

"""
1. Claim decomposition varies by LLM judge:
   One model might split "The API supports 1000 req/min for free
   and 5000 for pro" into 2 claims. Another might keep it as 1.
   This changes the denominator and thus the score.

2. Synonym support:
   If the context says "throttle at 1000 requests per 60 seconds"
   and the answer says "rate limit of 1000 per minute" — is that
   supported? Different judges disagree. You need consistent rules.

3. Implicit vs. explicit support:
   Context: "Free tier: 1000 req/min, Pro: unlimited"
   Answer: "Free tier is limited to 1000 req/min"
   This is supported by inference.

   Context: "API keys can be created in Settings"
   Answer: "You need an API key to use the API"
   This is NOT supported — the context doesn't say API keys are
   required, only where to create them.

4. Faithfulness in, noise out:
   If your context is contradictory, a faithful answer will
   reproduce the contradiction — which looks like a low-quality
   answer. But that's not the model's fault. Always interpret
   faithfulness in context of retrieval quality.
"""
```

**When faithfulness is misleading:**
- If the answer makes no claims at all ("I don't know"), faithfulness = 1.0 but the answer is useless
- If the context is contradictory and the model reproduces the contradiction faithfully, that's "high faithfulness" but poor user experience
- If the model correctly synthesizes information from multiple chunks into a general statement, faithfulness checks might flag it as unsupported because no single chunk contains the synthesized statement

#### Metric 2: Answer Relevance

**What it measures:** Does the answer actually address the question?

**How it works:**
1. Generate "reverse questions" — given the answer, what questions does it answer?
2. Compare the generated questions to the original question
3. Answer Relevance = cosine similarity between generated questions and the original question

**Why this approach?** Instead of directly asking "Is this answer relevant?", the metric avoids the LLM's tendency to say "yes" (acquiescence bias). By generating questions and measuring similarity, it gets a more objective signal.

```python
class AnswerRelevanceEvaluator:
    """
    Evaluates how relevant the answer is to the question.

    Works by generating N questions that the answer could be answering,
    then measuring how similar those questions are to the original.
    """

    async def evaluate(
        self,
        question: str,
        answer: str,
        n_questions: int = 3
    ) -> Dict:
        """
        Evaluate answer relevance.

        Returns score (0-1) and generated questions.
        """
        # Step 1: Generate questions from the answer
        gen_prompt = (
            f"Generate {n_questions} different questions that this "
            f"answer could be responding to:\n\nAnswer: {answer}\n\n"
            f"Generate questions that cover the FULL range of information "
            f"in the answer. Questions should be complete and specific."
        )

        gen_response = await self.llm_client.complete(gen_prompt)

        # Parse generated questions
        questions = self._parse_questions(gen_response)

        if not questions:
            return {"score": 0.0, "questions": [], "note": "Could not parse"}

        # Step 2: Compute similarity between original and generated questions
        similarities = []
        for gen_q in questions:
            sim = await self._compute_similarity(question, gen_q)
            similarities.append(sim)

        # Score = average similarity
        avg_similarity = sum(similarities) / len(similarities)

        return {
            "score": avg_similarity,
            "original_question": question,
            "generated_questions": questions,
            "individual_similarities": similarities
        }

    async def _compute_similarity(self, q1: str, q2: str) -> float:
        """Compute semantic similarity between two questions."""
        # Use an embedding model
        emb1 = await self.embedding_model.embed(q1)
        emb2 = await self.embedding_model.embed(q2)
        return cosine_similarity(emb1, emb2)
```

**When answer relevance is misleading:**
- Short, vague answers can score highly because they generate vague reverse questions that match anything
- An answer that says "The answer is 42" generates "What is 42?" which is semantically similar to "What's the meaning of life?" — but the answer is useless
- This metric captures topical relevance, not completeness or correctness

#### Metric 3: Context Precision

**What it measures:** Are the retrieved chunks actually necessary and sufficient to answer the question?

**How it works:**
1. Check if each chunk in the ranked list is relevant to the question
2. Context Precision = weighted sum where earlier chunks matter more
3. A system that puts relevant chunks at the top scores higher

```python
class ContextPrecisionEvaluator:
    """
    Evaluates how precise the retrieved context is.

    High precision = most retrieved chunks are relevant,
    AND relevant chunks appear early in the ranking.
    """

    async def evaluate(
        self,
        question: str,
        contexts: List[str]
    ) -> Dict:
        """
        Evaluate context precision.

        For each chunk (in order), determine if it's relevant.
        Weight: earlier chunks contribute more to the score.
        """
        chunk_relevance = []

        for i, context in enumerate(contexts):
            is_relevant = await self._check_relevance(question, context)
            chunk_relevance.append({
                "position": i + 1,
                "is_relevant": is_relevant,
                "text_preview": context[:100]
            })

        # Compute precision@k for each k
        precisions = []
        relevant_so_far = 0

        for i, cr in enumerate(chunk_relevance):
            if cr["is_relevant"]:
                relevant_so_far += 1
            prec_at_k = relevant_so_far / (i + 1)
            precisions.append(prec_at_k)

        # Weighted score: earlier ranks weighted higher
        weights = [1.0 / (i + 1) for i in range(len(precisions))]
        weighted_precision = sum(
            p * w for p, w in zip(precisions, weights)
        ) / sum(weights) if weights else 0.0

        return {
            "score": weighted_precision,
            "precision_at_k": {
                f"p@{i+1}": p for i, p in enumerate(precisions)
            },
            "chunk_relevance": chunk_relevance
        }

    async def _check_relevance(
        self, question: str, context: str
    ) -> bool:
        """Check if a chunk is relevant to the question."""
        response = await self.llm_client.complete(
            f"Question: {question}\n\n"
            f"Document: {context[:500]}\n\n"
            f"Is this document relevant to answering the question? "
            f"Reply ONLY: YES or NO"
        )
        return response.strip().upper() == "YES"
```

**When context precision is misleading:**
- A system that retrieves only 1 highly relevant chunk scores perfectly (p@1 = 1.0), but the answer might need 3 chunks
- Precision doesn't measure recall — you can have perfect precision while missing half the information
- The relevance check itself is LLM-based and inherits all LLM biases

#### Metric 4: Context Recall

**What it measures:** Are ALL the necessary chunks retrieved?

**How it works:**
1. Determine which parts of the "ground truth" answer (or the expected information) are covered by the retrieved context
2. Context Recall = (% of truth-spanning information present in context)

```python
class ContextRecallEvaluator:
    """
    Evaluates whether the retrieved context contains ALL the
    information needed to answer the question.
    """

    async def evaluate(
        self,
        question: str,
        answer: str,
        contexts: List[str]
    ) -> Dict:
        """
        Context recall: fraction of answer's claims that can be
        supported by the retrieved context.

        NOT the same as faithfulness! Faithfulness checks if the
        answer matches the context. Recall checks if the context
        COULD support the answer (even if the answer didn't use it).
        """
        # Step 1: Extract claims from the answer
        claims = await self._extract_claims(answer)

        if not claims:
            return {"score": 1.0, "covered_claims": [], "missed_claims": []}

        # Step 2: Check if each claim CAN be supported by context
        covered = []
        missed = []

        for claim in claims:
            is_covered = await self._check_coverage(claim, contexts)
            if is_covered:
                covered.append(claim)
            else:
                missed.append(claim)

        recall = len(covered) / len(claims) if claims else 1.0

        return {
            "score": recall,
            "covered_claims": covered,
            "missed_claims": missed,
            "total_claims": len(claims)
        }

    async def _check_coverage(
        self, claim: str, contexts: List[str]
    ) -> bool:
        """Check if ANY context chunk could support this claim."""
        prompt = f"""Claim: {claim}

Context:
{chr(10) + chr(10).join(f'[S{i+1}] {c[:400]}' for i, c in enumerate(contexts))}

Does ANY context chunk contain information that supports this claim?
The support can be explicit or through reasonable inference.
Reply: YES or NO
"""
        response = await self.llm_client.complete(prompt)
        return response.strip().upper() == "YES"
```

---

### 2. Building Evaluation Datasets

Your evaluation is only as good as your dataset. A biased, small, or unrepresentative dataset will give you confident but wrong conclusions.

#### Strategy 1: Manual Gold Standard

**When to use:** When you need high-quality evaluation with ground truth.

**Process:**
1. Select 100-200 representative queries from production (or expected production queries)
2. For each query, have a domain expert write:
   - The ideal answer
   - The required context chunks (explicitly, which documents and which sections)
   - Acceptable answer variants (edge cases, different levels of detail)
3. Use these as your "gold standard" for all automated metrics

**Advantages:** Highest quality, catches subtle errors
**Disadvantages:** Expensive, doesn't scale, needs domain experts

```python
@dataclass
class GoldStandardExample:
    """A single gold standard evaluation example."""
    query: str
    ideal_answer: str
    required_context_ids: List[str]  # Which chunks contain the answer
    acceptable_answer_variants: List[str] = None
    category: str = "factual"  # factual, comparative, procedural, etc.
    difficulty: str = "medium"  # easy, medium, hard
    notes: str = ""
```

#### Strategy 2: Synthetic Data Generation

**When to use:** When you have documents but no queries yet (starting from scratch).

**Process:**
1. Take your documents and chunk them
2. For each chunk (or group of chunks), use an LLM to generate:
   - A question that the chunk answers
   - An answer using only the chunk
   - Questions at different difficulty levels (direct, inferred, cross-document)
3. Filter for quality (check: does the chunk actually answer the question?)

```python
class SyntheticEvalDataGenerator:
    """
    Generate evaluation datasets from your existing documents.
    Useful when you have no production data yet.
    """

    def __init__(self, llm_client, chunk_repository):
        self.llm_client = llm_client
        self.chunk_repository = chunk_repository

    async def generate_questions_for_chunk(
        self,
        chunk_text: str,
        difficulty: str = "medium",
        n_questions: int = 3
    ) -> List[Dict]:
        """Generate questions that this chunk can answer."""

        difficulty_prompts = {
            "easy": (
                "Generate questions where the answer is explicitly "
                "stated in the text. Direct lookup questions."
            ),
            "medium": (
                "Generate questions that require understanding the "
                "text but don't directly quote it. Questions that "
                "test comprehension, not just lookup."
            ),
            "hard": (
                "Generate questions that require inference, combining "
                "multiple pieces of information, or applying the "
                "information to a new scenario."
            )
        }

        prompt = (
            f"Document:\n{chunk_text}\n\n"
            f"{difficulty_prompts[difficulty]}\n\n"
            f"Generate {n_questions} questions. For each question:\n"
            f"- The question should be answerable ONLY from this document\n"
            f"- The question should be how a real user would ask\n"
            f"- Provide the answer (as a gold standard)\n"
            f"- Note which specific part of the document supports the answer\n\n"
            f"Format each as:\n"
            f"Q: [question]\n"
            f"A: [answer]\n"
            f"Source: [exact text that supports the answer]"
        )

        response = await self.llm_client.complete(prompt)
        return self._parse_qa_pairs(response, chunk_text)

    async def generate_cross_document_questions(
        self,
        chunk_ids: List[str],
        n_questions: int = 2
    ) -> List[Dict]:
        """
        Generate questions that require information from multiple chunks.

        These are the hardest queries for RAG systems — they test
        whether the system can synthesize across documents.
        """
        chunks = [
            self.chunk_repository.get(cid) for cid in chunk_ids
        ]
        combined = "\n\n=== DOCUMENT BREAK ===\n\n".join(
            f"[Document {i+1}: {c.document_title}]\n{c.text}"
            for i, c in enumerate(chunks)
        )

        prompt = (
            f"Multiple documents:\n\n{combined}\n\n"
            f"Generate {n_questions} questions that REQUIRE information "
            f"from MULTIPLE documents to answer. The questions should "
            f"test whether a RAG system can synthesize across sources.\n\n"
            f"For each question, specify which documents are needed, "
            f"what part of each document provides the needed info, "
            f"and the ideal synthesized answer."
        )

        response = await self.llm_client.complete(prompt)
        return self._parse_multi_doc_response(response)

    def _parse_qa_pairs(self, response: str, chunk_text: str) -> List[Dict]:
        """Parse structured Q&A from LLM response."""
        pairs = []
        current = {}
        for line in response.strip().split("\n"):
            if line.startswith("Q:"):
                if current:
                    pairs.append(current)
                current = {"question": line[2:].strip(), "source_chunk": chunk_text[:200]}
            elif line.startswith("A:"):
                current["answer"] = line[2:].strip()
            elif line.startswith("Source:"):
                current["supporting_text"] = line[7:].strip()
        if current:
            pairs.append(current)
        return pairs
```

#### Strategy 3: Production Log Mining

**When to use:** When you have production data.

**Process:**
1. Collect real user queries (anonymized)
2. For a sample, have the correct answer written (or extracted from successful interactions)
3. Use this as your evaluation set — it's the most realistic

```python
class ProductionEvalSetBuilder:
    """Build evaluation datasets from production logs."""

    def build_from_logs(
        self,
        log_path: str,
        sample_size: int = 200,
        min_query_length: int = 5
    ):
        """Sample production queries for evaluation dataset."""
        import json, random

        queries = []
        with open(log_path) as f:
            for line in f:
                entry = json.loads(line)
                q = entry.get("query", "")
                if len(q) >= min_query_length:
                    queries.append(entry)

        # Stratified sample
        sampled = random.sample(queries, min(sample_size, len(queries)))

        return [
            {"query": s["query"], "source": "production"}
            for s in sampled
        ]
```

#### Dataset Quality Checklist

Regardless of how you build your dataset, verify:

1. **Coverage**: Does your dataset cover all document types/categories?
2. **Difficulty distribution**: Do you have easy, medium, and hard questions?
3. **Query diversity**: Are queries phrased in varied ways (not just "What is X?")?
4. **Edge cases**: Do you have questions that test boundaries (ambiguous, multi-intent, out-of-scope)?
5. **No leakage**: Are the evaluation questions different from any examples in the prompt/system prompt?
6. **Temporal split**: If using production data, is the evaluation set from a DIFFERENT time period than your training/adjustment data?

---

### 3. Component-Level Evaluation

Don't just evaluate the end-to-end system. Evaluate each component independently to know where to invest.

#### Retrieval Evaluation (Independent of Generation)

```python
class RetrievalEvaluator:
    """
    Evaluate retrieval in isolation — before the LLM touches anything.

    Metrics:
    - precision@k: fraction of retrieved chunks that are relevant
    - recall@k: fraction of all relevant chunks that were retrieved
    - MRR: Mean Reciprocal Rank (how early does the first relevant result appear)
    - NDCG: Normalized Discounted Cumulative Gain (ranking quality)
    """

    def __init__(self, relevance_judgments: Dict[str, List[str]]):
        """
        Args:
            relevance_judgments: Map of query -> list of relevant chunk IDs
        """
        self.judgments = relevance_judgments

    def precision_at_k(
        self, retrieved_ids: List[str], relevant_ids: List[str], k: int
    ) -> float:
        """Precision@k: fraction of top-k retrieved that are relevant."""
        top_k = retrieved_ids[:k]
        relevant_in_top_k = sum(1 for rid in top_k if rid in relevant_ids)
        return relevant_in_top_k / k if k > 0 else 0.0

    def recall_at_k(
        self, retrieved_ids: List[str], relevant_ids: List[str], k: int
    ) -> float:
        """Recall@k: fraction of all relevant documents found in top-k."""
        top_k = set(retrieved_ids[:k])
        relevant_set = set(relevant_ids)
        if not relevant_set:
            return 1.0  # No relevant docs = vacuously perfect
        return len(top_k & relevant_set) / len(relevant_set)

    def mean_reciprocal_rank(
        self, retrieved_ids: List[str], relevant_ids: List[str]
    ) -> float:
        """MRR: 1/(rank of first relevant result), 0 if none found."""
        for i, rid in enumerate(retrieved_ids):
            if rid in relevant_ids:
                return 1.0 / (i + 1)
        return 0.0

    def ndcg_at_k(
        self, retrieved_ids: List[str], relevance_scores: Dict[str, float], k: int
    ) -> float:
        """
        NDCG@k: Normalized Discounted Cumulative Gain.
        Accounts for graded relevance (not just binary).
        """
        import math

        dcg = 0.0
        for i, rid in enumerate(retrieved_ids[:k]):
            rel = relevance_scores.get(rid, 0.0)
            dcg += (2**rel - 1) / math.log2(i + 2)

        # Ideal ordering
        ideal = sorted(
            relevance_scores.values(), reverse=True
        )[:k]
        idcg = sum(
            (2**rel - 1) / math.log2(i + 2)
            for i, rel in enumerate(ideal)
        )

        return dcg / idcg if idcg > 0 else 0.0

    def evaluate_retrieval(
        self, query: str, retrieved_chunks: List[Any], k_values: List[int] = [1, 3, 5, 10]
    ) -> Dict:
        """Full retrieval evaluation for a single query."""
        relevant_ids = self.judgments.get(query, [])
        retrieved_ids = [c.id for c in retrieved_chunks]

        results = {"query": query, "num_retrieved": len(retrieved_ids)}

        for k in k_values:
            results[f"p@{k}"] = self.precision_at_k(retrieved_ids, relevant_ids, k)
            results[f"r@{k}"] = self.recall_at_k(retrieved_ids, relevant_ids, k)

        results["mrr"] = self.mean_reciprocal_rank(retrieved_ids, relevant_ids)

        return results
```

#### Generation Evaluation (Independent of Retrieval)

This tests the LLM's ability to use context well, controlling for retrieval quality:

```python
class GenerationEvaluator:
    """
    Evaluate generation in isolation.

    Feed the LLM the EXACT ideal context (from gold standard) and
    measure how well it uses it. This separates generation quality
    from retrieval quality.
    """

    async def evaluate_with_perfect_context(
        self,
        query: str,
        ideal_context: str,
        ideal_answer: str,
        rag_system: "YourRAGSystem"
    ) -> Dict:
        """
        Feed the LLM the ideal context directly, bypassing retrieval.
        This measures: given perfect context, can the LLM produce
        a correct answer?
        """
        # Bypass retrieval — inject ideal context
        result = await rag_system.generate_with_context(
            query=query,
            context=ideal_context
        )

        # Compare to gold standard
        comparison = await self._compare_answers(
            generated=result.answer,
            ideal=ideal_answer
        )

        return {
            "query": query,
            "generated_answer": result.answer,
            "ideal_answer": ideal_answer,
            "comparison": comparison,
            "verdict": "pass" if comparison["score"] > 0.8 else "fail"
        }

    async def _compare_answers(
        self, generated: str, ideal: str
    ) -> Dict:
        """Compare generated answer to ideal answer."""
        prompt = (
            f"Compare the GENERATED answer to the IDEAL answer.\n\n"
            f"Ideal: {ideal}\n\n"
            f"Generated: {generated}\n\n"
            f"Rate 0-10 on:\n"
            f"- factual_correctness: (are the facts right?)\n"
            f"- completeness: (does it cover everything in the ideal?)\n"
            f"- conciseness: (is it appropriately detailed?)\n"
            f"- citations: (are sources cited correctly?)\n\n"
            f"Return JSON with scores and brief explanation."
        )
        response = await self.llm_client.complete(prompt)
        return self._parse_scores(response)
```

#### The Eval Sandbox Pattern

```python
class EvalSandbox:
    """
    Test individual components in isolation by controlling variables.

    Use this to answer questions like:
    - "Is retrieval bad, or is the LLM bad at using context?"
    - "Does chunk size X work better for this query type?"
    - "Is the embedding model the bottleneck?"
    """

    async def isolate_retrieval_issue(
        self, query: str, chunks: List[Any], gold_answer: str
    ):
        """
        Determine if the issue is retrieval or generation.

        If the LLM produces a good answer when given the CORRECT
        chunks manually, but fails with retrieved chunks = retrieval issue.
        If it fails even with correct chunks = generation issue.
        """
        # Test with retrieved chunks
        retrieved_result = await self.rag.answer(query)

        # Test with gold standard chunks
        gold_context = self._build_context_from(
            chunks=self.gold_chunks_for(query)
        )
        gold_result = await self.rag.generate_with_context(query, gold_context)

        verdict = "retrieval_issue"
        if gold_result.score < 0.7:
            verdict = "generation_issue"
        elif retrieved_result.score >= 0.7:
            verdict = "no_issue"

        return {
            "verdict": verdict,
            "retrieval_score": retrieved_result.score,
            "gold_score": gold_result.score,
            "retrieved_answer": retrieved_result.answer,
            "gold_answer": gold_result.answer
        }
```

---

### 4. End-to-End Evaluation Pipeline

Real evaluation requires running many queries, computing multiple metrics, and aggregating results.

```python
"""
full_eval_pipeline.py — Complete RAG evaluation pipeline.

Features:
- Parallel evaluation of multiple queries
- Multiple metrics per query (faithfulness, relevance, precision, recall)
- Aggregation across queries
- Comparison across configurations
- Caching to avoid re-computing expensive LLM calls
"""

import asyncio
import json
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Any, Callable
from datetime import datetime
import hashlib
from pathlib import Path


@dataclass
class EvalConfig:
    """Configuration for an evaluation run."""
    name: str = "rag_eval"
    model_for_judge: str = "gpt-4o-mini"
    metrics: List[str] = field(default_factory=lambda: [
        "faithfulness", "answer_relevance", "context_precision", "context_recall"
    ])
    num_workers: int = 5  # Parallel queries
    cache_dir: Optional[str] = ".eval_cache"
    max_context_length: int = 3000  # Truncate contexts for eval to save cost


@dataclass
class QueryResult:
    """Results for a single evaluation query."""
    query: str
    answer: str
    contexts: List[str]
    faithfulness: Optional[float] = None
    answer_relevance: Optional[float] = None
    context_precision: Optional[float] = None
    context_recall: Optional[float] = None
    latency_ms: float = 0.0
    context_token_count: int = 0
    error: Optional[str] = None


class RAGEvaluator:
    """
    Full RAG evaluation pipeline.

    Usage:
        evaluator = RAGEvaluator(llm_client=client, config=EvalConfig())

        # Run evaluation
        results = await evaluator.evaluate(
            rag_system=my_rag,
            queries=["What is X?", "How to Y?"],
            gold_answers=["X is...", "To Y, do..."],
            gold_contexts=[["doc1"], ["doc2", "doc3"]]
        )

        # Get report
        report = evaluator.summarize(results)
        print(report)
    """

    def __init__(self, llm_client, config: EvalConfig = None):
        self.llm_client = llm_client
        self.config = config or EvalConfig()
        self.cache = {}
        self._load_cache()

    async def evaluate(
        self,
        rag_system,
        queries: List[str],
        gold_answers: Optional[List[str]] = None,
        gold_contexts: Optional[List[List[str]]] = None
    ) -> List[QueryResult]:
        """Run full evaluation on a set of queries."""

        semaphore = asyncio.Semaphore(self.config.num_workers)

        async def evaluate_one(i: int) -> QueryResult:
            async with semaphore:
                query = queries[i]
                gold_answer = gold_answers[i] if gold_answers else None
                gold_context = gold_contexts[i] if gold_contexts else None

                t0 = datetime.now()

                # Run RAG
                try:
                    result = await rag_system.answer(query)
                except Exception as e:
                    return QueryResult(
                        query=query, answer="", contexts=[],
                        error=str(e)
                    )

                latency = (datetime.now() - t0).total_seconds() * 1000

                # Truncate contexts for eval
                contexts = [c.text[:self.config.max_context_length]
                           for c in result.chunks]

                qr = QueryResult(
                    query=query,
                    answer=result.answer,
                    contexts=contexts,
                    latency_ms=latency,
                    context_token_count=sum(
                        len(c.split()) for c in contexts
                    )
                )

                # Compute metrics
                if "faithfulness" in self.config.metrics:
                    qr.faithfulness = await self._measure_faithfulness(
                        query, result.answer, contexts
                    )

                if "answer_relevance" in self.config.metrics:
                    qr.answer_relevance = await self._measure_answer_relevance(
                        query, result.answer
                    )

                if "context_precision" in self.config.metrics:
                    qr.context_precision = await self._measure_context_precision(
                        query, contexts
                    )

                if "context_recall" in self.config.metrics:
                    qr.context_recall = await self._measure_context_recall(
                        query, result.answer, contexts
                    )

                return qr

        # Run all evaluations in parallel
        tasks = [evaluate_one(i) for i in range(len(queries))]
        results = await asyncio.gather(*tasks)

        self._save_cache()
        return results

    def summarize(self, results: List[QueryResult]) -> Dict:
        """Aggregate results into a summary report."""
        n = len(results)

        report = {
            "eval_name": self.config.name,
            "timestamp": datetime.now().isoformat(),
            "num_queries": n,
            "num_errors": sum(1 for r in results if r.error),
            "latency": {
                "mean_ms": sum(r.latency_ms for r in results) / n,
                "p50_ms": sorted(r.latency_ms for r in results)[n // 2],
                "p95_ms": sorted(r.latency_ms for r in results)[int(n * 0.95)],
                "p99_ms": sorted(r.latency_ms for r in results)[int(n * 0.99)],
            },
            "context_tokens": {
                "mean": sum(r.context_token_count for r in results) / n,
                "max": max(r.context_token_count for r in results),
            },
            "metrics": {}
        }

        for metric in self.config.metrics:
            values = [
                getattr(r, metric) for r in results
                if getattr(r, metric) is not None
            ]
            if values:
                report["metrics"][metric] = {
                    "mean": sum(values) / len(values),
                    "min": min(values),
                    "max": max(values),
                    "values": values,  # Full distribution for analysis
                }

        # Failure analysis
        low_scores = {}
        for metric in self.config.metrics:
            low = [
                (r.query, getattr(r, metric))
                for r in results
                if getattr(r, metric) is not None
                and getattr(r, metric) < 0.5
            ]
            if low:
                low_scores[metric] = sorted(low, key=lambda x: x[1])[:10]

        report["low_scoring_queries"] = low_scores

        return report

    async def _measure_faithfulness(
        self, query: str, answer: str, contexts: List[str]
    ) -> float:
        """Measure faithfulness (cached)."""
        cache_key = self._cache_key("faithfulness", query, answer)
        cached = self.cache.get(cache_key)
        if cached is not None:
            return cached

        score = await self._compute_faithfulness(query, answer, contexts)
        self.cache[cache_key] = score
        return score

    def _cache_key(self, metric: str, *args) -> str:
        content = metric + "".join(str(a) for a in args)
        return hashlib.md5(content.encode()).hexdigest()

    def _load_cache(self):
        if self.config.cache_dir:
            cache_path = Path(self.config.cache_dir) / "eval_cache.json"
            if cache_path.exists():
                with open(cache_path) as f:
                    self.cache = json.load(f)

    def _save_cache(self):
        if self.config.cache_dir:
            cache_dir = Path(self.config.cache_dir)
            cache_dir.mkdir(exist_ok=True)
            with open(cache_dir / "eval_cache.json", "w") as f:
                json.dump(self.cache, f)


# ============================================================
# Config Comparison: A/B Test Your RAG Changes
# ============================================================

async def compare_configs(
    rag_config_a: Callable,
    rag_config_b: Callable,
    eval_queries: List[str],
    evaluator: RAGEvaluator
) -> Dict:
    """
    Compare two RAG configurations.

    Usage:
        config_a = lambda: RAGSystem(chunk_size=500, top_k=5)
        config_b = lambda: RAGSystem(chunk_size=1000, top_k=3)

        comparison = await compare_configs(config_a, config_b, test_queries, evaluator)
    """
    results_a = await evaluator.evaluate(rag_config_a(), eval_queries)
    results_b = await evaluator.evaluate(rag_config_b(), eval_queries)

    report_a = evaluator.summarize(results_a)
    report_b = evaluator.summarize(results_b)

    comparison = {
        "config_a": "config_a",
        "config_b": "config_b",
        "metric_deltas": {}
    }

    for metric in evaluator.config.metrics:
        a_score = report_a["metrics"].get(metric, {}).get("mean", 0)
        b_score = report_b["metrics"].get(metric, {}).get("mean", 0)
        delta = b_score - a_score

        comparison["metric_deltas"][metric] = {
            "a": a_score,
            "b": b_score,
            "delta": delta,
            "winner": "a" if a_score > b_score else "b" if b_score > a_score else "tie"
        }

    # Statistical significance check (simple version)
    for metric in evaluator.config.metrics:
        a_vals = [getattr(r, metric) for r in results_a if getattr(r, metric)]
        b_vals = [getattr(r, metric) for r in results_b if getattr(r, metric)]
        if a_vals and b_vals:
            from scipy import stats
            t_stat, p_value = stats.ttest_ind(a_vals, b_vals)
            comparison["metric_deltas"][metric]["p_value"] = float(p_value)
            comparison["metric_deltas"][metric]["significant"] = p_value < 0.05

    return comparison
```

---

### 5. LLM-as-Judge: Design & Biases

Using an LLM to evaluate another LLM is powerful but dangerous. Here are the known biases and how to mitigate them.

#### The 5 Biases of LLM Judges

| Bias | Description | Mitigation |
|------|-------------|------------|
| **Position bias** | Prefers answers in position A vs B | Run both orders (A/B and B/A), take average |
| **Verbosity bias** | Prefers longer, more detailed answers | Normalize by length or ask explicitly about conciseness |
| **Self-enhancement bias** | Prefers answers from the same model family | Use a different model as judge (e.g., GPT-4 eval of GPT-4o-mini) |
| **Format bias** | Prefers well-formatted answers (lists, bold) | Strip formatting before evaluation |
| **Syophancy bias** | Agrees with the user's framing | Blind evaluation (judge doesn't see which answer is which) |

#### Mitigation: Pairwise Comparison with Calibration

```python
class PairwiseJudge:
    """
    Compare two answers and determine which is better.

    Handles position bias by running both orderings.
    """

    async def compare(
        self,
        query: str,
        answer_a: str,
        answer_b: str,
        criteria: str = "correctness"
    ) -> Dict:
        """
        Compare two answers for the same query.

        Runs the comparison twice (A vs B, B vs A) and averages.
        """
        # Forward order
        result_ab = await self._single_comparison(
            query, answer_a, answer_b, criteria
        )

        # Reverse order (to detect position bias)
        result_ba = await self._single_comparison(
            query, answer_b, answer_a, criteria
        )

        # Check for position bias
        position_bias = result_ab["winner"] != self._invert(result_ba["winner"])

        # If position bias detected, result is unreliable
        if position_bias:
            return {
                "winner": "ambiguous",
                "confidence": "low",
                "position_bias_detected": True,
                "note": "Results flip with ordering — position bias suspected"
            }

        return {
            "winner": result_ab["winner"],
            "confidence": "high",
            "position_bias_detected": False,
            "reasoning": result_ab["reasoning"]
        }

    async def _single_comparison(
        self, query: str, first: str, second: str, criteria: str
    ) -> Dict:
        """Single pairwise comparison."""
        prompt = f"""
Compare these two answers to the same question.

Question: {query}

Answer A:
{first}

Answer B:
{second}

Which answer is better in terms of {criteria}?
Consider: factual accuracy, completeness, clarity, and helpfulness.

Reply with:
- Winner: "A" | "B" | "TIE"
- Reasoning: [brief explanation]
"""
        response = await self.llm_client.complete(prompt)
        return self._parse_verdict(response)

    def _invert(self, winner: str) -> str:
        if winner == "A": return "B"
        if winner == "B": return "A"
        return "TIE"
```

#### Mitigation: Rubric-Based Scoring

Instead of "Answer A is better," define explicit evaluation criteria:

```python
class RubricEvaluator:
    """
    Score answers against a detailed rubric.

    More reliable than holistic scoring because the LLM has
    explicit criteria to check against.
    """

    CORRECTNESS_RUBRIC = """
Score 1-5 for each criterion:

1. Factual Accuracy (1-5):
   5 = All claims supported by context, no errors
   3 = Minor errors or unsupported claims
   1 = Major hallucination or multiple unsupported claims

2. Completeness (1-5):
   5 = Fully answers the question, includes all relevant details
   3 = Missing minor details
   1 = Misses the main point or key information

3. Grounding (1-5):
   5 = All claims are properly cited with sources
   3 = Most claims cited, some missing
   1 = No citations provided

4. Conciseness (1-5):
   5 = Appropriate length, no unnecessary information
   3 = Some verbosity or missing necessary detail
   1 = Excessively long or too brief to be useful

Total: /20
"""

    async def score(
        self, query: str, answer: str, context: List[str]
    ) -> Dict:
        """Score an answer against the rubric."""
        context_text = "\n".join(context)

        response = await self.llm_client.complete(
            f"Question: {query}\n\n"
            f"Context:\n{context_text}\n\n"
            f"Answer:\n{answer}\n\n"
            f"Score this answer:\n{self.CORRECTNESS_RUBRIC}\n\n"
            f"Return scores with brief justification."
        )

        return self._parse_scores(response)
```

---

### 6. CI/CD Integration for RAG

Your RAG system should have a CI pipeline that runs evaluations on every change.

```yaml
# .github/workflows/rag-eval.yml
name: RAG Evaluation
on:
  pull_request:
    paths:
      - 'rag/**'
      - 'prompts/**'
      - 'chunking/**'

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run RAG Evaluation
        run: |
          python -m eval.run_pipeline \
            --config baseline.json \
            --queries eval/sets/regression.json \
            --output eval/results/${{ github.sha }}.json

      - name: Check Against Thresholds
        run: |
          python -m eval.check_thresholds \
            --results eval/results/${{ github.sha }}.json \
            --thresholds eval/thresholds.json
        # Fails CI if metrics drop below thresholds

      - name: Comment on PR
        if: always()
        run: |
          python -m eval.pr_comment \
            --results eval/results/${{ github.sha }}.json \
            --baseline eval/results/main.json
```

```python
# eval/thresholds.json — Fails CI if metrics drop below these
{
    "faithfulness": {
        "mean": 0.85,
        "min_query": 0.5  # No single query should score below 0.5
    },
    "answer_relevance": {
        "mean": 0.80
    },
    "context_precision": {
        "mean": 0.75
    },
    "latency_ms": {
        "p95": 5000  # 95th percentile under 5 seconds
    }
}
```

---

## ✅ Good Output Examples

### What Good Evaluation Output Looks Like

**Clear pass/fail per query:**

```
Evaluation: RAG v2.3 vs baseline
Date: 2024-07-15
Queries: 200 (50 easy, 100 medium, 50 hard)

OVERALL RESULTS:
  Faithfulness:    0.91 (+0.03 vs baseline) ✓
  Answer Relevance: 0.86 (+0.01 vs baseline) ✓
  Context Precision: 0.79 (+0.05 vs baseline) ✓
  Context Recall:    0.73 (-0.02 vs baseline) ⚠

BREAKDOWN BY DIFFICULTY:
  Easy (50):    F=0.97, R=0.95, P=0.92, Re=0.94
  Medium (100): F=0.92, R=0.87, P=0.80, Re=0.74
  Hard (50):    F=0.84, R=0.76, P=0.65, Re=0.51

HARD FAILURES (score < 0.5):
  8 queries with low faithfulness:
    - "What about GDPR compliance?" (F=0.33)
      Reason: Retrieved chunk about "data retention" but not
      "GDPR compliance" — partial overlap
    ...

LATENCY: mean=1.2s, p95=3.1s, p99=5.8s
  ⚠ 3 queries exceeded 10s timeout

VERDICT: FAIL — context recall dropped below 0.75 threshold
```

**Good regression alert:**

```
⚠ REGRESSION DETECTED: Context Recall
   Before (commit a1b2c3): 0.75
   After  (commit d4e5f6): 0.69
   Delta: -0.06 (8% drop)

   Suspected cause: Changed chunk_size from 500 to 1000
   (commit d4e5f6 modified chunking_strategy.py)

   Recommended action: Revert chunk size change or investigate
   why larger chunks miss relevant information.
```

---

## ❌ Antipatterns & Failure Modes

### 1. Evaluating Only on Easy Queries

**What**: Building an eval set from obvious queries that map to single chunks.

**Why it fails**: You'll get scores of 0.95+ and think your system is production-ready. But real user queries are messy, multi-intent, and cross-document.

**Fix**: Ensure your eval set has a distribution of difficulty levels. If you don't have hard queries, your metrics don't mean anything.

### 2. The Single Metric Trap

**What**: Optimizing for one metric (e.g., faithfulness) at the expense of others.

**Example**: Increasing context filtering to be extremely aggressive → faithfulness goes up (less noise) but recall goes down (missing info the answer needs).

**Fix**: Define your "composite score" before you start tuning. For example: `score = 0.3*faithfulness + 0.3*relevance + 0.2*precision + 0.2*recall`. Understand which metric you're willing to sacrifice for which gain.

### 3. Using the Same LLM for Generation and Evaluation

**What**: Using GPT-4o-mini for both the RAG answer AND the evaluation.

**Why it fails**: The evaluator shares the same biases and blind spots as the generator. It won't catch errors that both models make.

**Fix**: Use a different model family for evaluation. If you generate with GPT-4o-mini, evaluate with Claude or GPT-4. The cost is worth the signal quality.

### 4. Over-Indexing on Aggregate Scores

**What**: Looking only at mean/median scores and ignoring the distribution.

**Why it fails**: A mean faithfulness of 0.90 could hide 10% of queries scoring 0.0. The system "averages out" catastrophic failures.

**Fix**: Always report the full distribution — min, p5, p25, p50, p75, p95, max. Set floor thresholds: no query should score below X.

### 5. Evaluation Dataset Leakage

**What**: Using the same documents to generate both the RAG index AND the evaluation questions.

**Why it fails**: Your eval data is "in distribution" — the questions perfectly match the documents. Real production queries won't. Your evaluation overestimates performance.

**Fix**: Hold out evaluation documents. Or better: use a separate corpus for evaluation questions, or use real production queries (unseen during development).

### 6. Caching Evaluation Results

**What**: Not using an evaluation cache.

**Why it fails**: LLM-as-judge evaluation is expensive and slow. Without caching, you re-evaluate the same (query, answer, context) triples on every run, wasting money and time.

**Fix**: Hash the (query, answer, context) triple and cache the evaluation result. Clear the cache only when the judge model or evaluation criteria change.

---

## 🧪 Drills & Challenges

### Drill 1: Build a Full RAG Evaluation Pipeline (60 min)

Create a `RAGEvaluator` class that:

1. Takes a list of (query, gold_answer, gold_contexts) triple
2. Runs your RAG system on each query
3. Computes faithfulness, answer relevance, context precision, and context recall using LLM-as-judge
4. Returns both per-query results and aggregated report
5. Includes a caching layer (don't re-evaluate the same thing twice)
6. Reports distribution (not just mean) for each metric

**Requirements:**
- Handle 50+ queries efficiently (parallel evaluation)
- Handle errors gracefully (log the error, continue, track failure rate)
- Provide a `summarize()` method that outputs actionable insights
- Export results to JSON for dashboard ingestion

### Drill 2: The Eval Dataset Dilemma (40 min)

You're building a RAG system for a company with 10,000 internal documents. You have no existing Q&A pairs, no user logs, and no domain experts available for labeling.

**Tasks:**
1. Design a synthetic data generation strategy that creates a diverse eval set
2. Implement the generation pipeline (use an LLM to generate questions from documents)
3. Generate at least 30 evaluation questions across 3 difficulty levels
4. For each generated question, verify:
   - Can the question actually be answered from the document?
   - Is the question realistic (how a real user would ask)?
   - Are there answer variants that would also be correct?
5. Document what biases your synthetic dataset might have

**Deliverable**: A JSON file with your evaluation dataset and a paragraph explaining what it might miss.

### Drill 3: The Faithfulness vs. Recall Tradeoff (35 min)

You have two RAG configurations:
- **Config A**: top-K=3, strict relevance threshold (0.8)
- **Config B**: top-K=8, loose threshold (0.5)

Run both on the same 20 queries and compute:
1. Faithfulness for each
2. Context recall for each
3. Answer relevance for each

**Analysis:**
- Which configuration has higher faithfulness? Why?
- Which configuration has higher recall? Why?
- For which types of queries does Config A perform better?
- For which types does Config B perform better?
- What's the optimal configuration if you weight faithfulness 2x over recall?

**Write your recommendation**: Given these results, would you choose A or B for production? What's the reasoning?

### Drill 4: The Judge Bias Experiment (45 min)

Design an experiment to measure position bias in your LLM judge.

**Setup:**
1. Take 10 evaluation queries
2. For each query, generate two answers: one clearly correct, one subtly wrong
3. Run pairwise comparison in both orders:
   - Order 1: (correct first, wrong second)
   - Order 2: (wrong first, correct second)
4. Measure how often the judge chooses the first answer vs. the second

**Measure:**
- Position bias rate: % of times where order determines the winner
- Is the bias consistent (always prefers position 1) or variable?
- Does the bias change with answer quality gap (big gap vs. small gap)?

**Mitigation:**
- Implement position bias correction in your judge pipeline
- Measure whether the correction helps

### Drill 5: Build a Regression Detector (30 min)

Create a script that:
1. Takes two evaluation result files (baseline.json and current.json)
2. For each metric, computes the delta between baseline and current
3. Flags any metric that dropped below its warning threshold (e.g., -0.05)
4. Flags any metric that dropped below its critical threshold (e.g., -0.10)
5. Outputs a structured report suitable for a CI pipeline

```python
def check_regression(
    baseline: Dict,
    current: Dict,
    thresholds: Dict[str, float]
) -> Dict:
    """
    Compare current eval results to baseline.

    Args:
        baseline: Results from the baseline run
        current: Results from the current run
        thresholds: {metric_name: max_allowed_drop}

    Returns:
        {passed: bool, regressions: [...], improvements: [...]}
    """
    # Your implementation
    pass
```

**Test with:**
```python
baseline = {
    "metrics": {
        "faithfulness": {"mean": 0.90},
        "answer_relevance": {"mean": 0.85},
        "context_precision": {"mean": 0.80},
    }
}
current = {
    "metrics": {
        "faithfulness": {"mean": 0.88},  # -0.02, ok
        "answer_relevance": {"mean": 0.78},  # -0.07, regression!
        "context_precision": {"mean": 0.82},  # +0.02, improvement
    }
}
thresholds = {"faithfulness": 0.05, "answer_relevance": 0.05, "context_precision": 0.05}
```

### Drill 6: The Evaluation Cost Analysis (30 min)

You have the following costs:
- GPT-4o-mini: $0.15/M input tokens, $0.60/M output tokens
- GPT-4o: $2.50/M input tokens, $10.00/M output tokens
- Embedding model (text-embedding-3-small): $0.02/M tokens
- Each evaluation query uses ~500 input tokens for the generator
- Each metric call uses ~1000 tokens for the judge

**Calculate:**
1. Cost to evaluate 1,000 queries with all 4 RAGAS metrics using GPT-4o-mini as judge
2. Cost to evaluate with GPT-4o as judge
3. Cost to evaluate 1,000 queries using the generator (just inference, no eval)
4. If you sample 10% of production queries for evaluation, how many queries/day can you evaluate for a $50/day budget?

**Design:**
5. Propose a tiered evaluation strategy:
   - Cheapest: which metrics on which queries?
   - Most comprehensive: which metrics on which queries?
   - What's the optimal sampling strategy for a $30/day budget?

---

## 🚦 Gate Check

Before proceeding to Phase 4.07 (Production RAG), you must demonstrate:

1. **Build a complete evaluation pipeline**: Create a `RAGEvaluator` that measures all 4 RAGAS metrics with caching and parallel execution.

2. **Create an evaluation dataset**: Generate at least 30 evaluation examples (either synthetic or from real data) spanning easy, medium, and hard difficulty.

3. **Run and analyze an evaluation**: Evaluate your RAG system from the Phase 4 project (or a mock RAG system) and produce:
   - Per-query scores for all metrics
   - Aggregated report with distribution
   - At least 3 specific findings (e.g., "Faithfulness drops on multi-document queries," "Context recall is excellent for product docs but poor for policy docs")

4. **A/B test a change**: Compare two configurations (e.g., different chunk sizes, different top-K, different context integration strategy) and determine which is better — with statistical evidence, not gut feel.

5. **Answer the reflection questions**:
   - What's the single most important metric for your specific RAG use case?
   - How do you know your evaluation dataset is representative of real usage?
   - What's your threshold for "good enough" to deploy?
   - How do you detect evaluation bias in your LLM judge?

**Passing criteria:** You can make a data-driven decision about a RAG change and explain WHY you made it. Your evaluation is reproducible (someone else can run the same evaluation and get the same results). You know which metrics to trust and which to treat skeptically.

---

## 📚 Resources

### Papers & Frameworks
- **RAGAS: Automated Evaluation of Retrieval Augmented Generation** (Es et al., 2023) — The original RAGAS paper. Understand the metrics, then build your own implementation.
- **Judging LLM-as-a-Judge** (Zheng et al., 2023) — MT-Bench paper. Essential reading on LLM judge biases.
- **ARES: Automatic RAG Evaluation System** — An alternative to RAGAS with different tradeoffs.
- **RGB: A Benchmark for RAG Systems** — Reference benchmark for RAG evaluation.

### Tools
- **RAGAS library** (`ragas`) — Production library for RAG evaluation. Use it after you've built your own from scratch (to understand what it's really doing).
- **DeepEval** (`deepeval`) — Alternative evaluation framework with RAG-specific metrics.
- **Langfuse** — Has built-in RAG evaluation traces (annotation queues, manual eval).
- **Arize Phoenix** — LLM observability with RAG evaluation support.

### Production Patterns
- **Stitch Fix's RAG evaluation blog** — How a production team evaluates their RAG system.
- **Glean's evaluation framework** — Enterprise RAG evaluation with multiple quality dimensions.
- **Anthropic's eval framework** — Not RAG-specific, but their approach to systematic evaluation is instructive.

### Your Own Code
- The **Customer Support Bot** (Phase 4 project) will need an evaluation framework. Everything in this module goes directly into the project.
- Revisit your **Knowledge Search Engine (Phase 3)**. It has retrieval but no generation. If you added generation, what would the evaluation look like? How would you measure whether the generated answers are better than raw search results?

### Expert-Level Deep Dives
- **For evaluation engineers**: The highest-leverage activity is improving your evaluation DATASET, not your evaluation METRICS. A perfect metric on a biased dataset is worse than a noisy metric on a representative dataset. Spend twice as much time on dataset quality as on metric implementation.
- **For production teams**: Build a "canary" evaluation — a small, fast eval that runs on EVERY code change (catching obvious regressions in < 2 minutes) and a "full" eval that runs nightly (catching subtle regressions across all query types). The canary catches commit-level breakage; the full eval catches week-over-week drift.
- **For critical thinkers**: The most important question in RAG evaluation is "Is my eval set representative of my production traffic?" If the answer is "I don't know," your eval numbers are meaningless regardless of how sophisticated your metrics are. Measure the distribution gap between eval and production, and don't trust your metrics until that gap is small.

---

*"Evaluation is not about proving your system works. It's about discovering where it doesn't."*
