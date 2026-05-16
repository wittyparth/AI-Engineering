# Drill: Build RAG Without LangChain

**Time**: 45 min  **Difficulty**: Medium

## Task

Build a complete RAG pipeline using only:
- `openai` (or another LLM provider)
- `sentence-transformers` (for embeddings)
- `numpy` (for vector search)
- Your own code (no LangChain, no LlamaIndex)

```python
import numpy as np
from sentence_transformers import SentenceTransformer
import openai

class RAGFromScratch:
    def __init__(self, docs: list[str]):
        self.embedder = SentenceTransformer("all-MiniLM-L6-v2")
        self.docs = docs
        self.chunks = self._chunk(docs)
        self.embeddings = self.embedder.encode(self.chunks)

    def _chunk(self, docs: list[str]) -> list[str]:
        """Simple recursive chunking."""
        chunks = []
        for doc in docs:
            # Split by paragraph first, then by sentence if too long
            paragraphs = doc.split("\n\n")
            for para in paragraphs:
                if len(para) > 500:
                    # Further split long paragraphs
                    sentences = para.split(". ")
                    current = ""
                    for sent in sentences:
                        if len(current) + len(sent) < 500:
                            current += sent + ". "
                        else:
                            chunks.append(current.strip())
                            current = sent + ". "
                    chunks.append(current.strip())
                else:
                    chunks.append(para)
        return chunks

    def retrieve(self, query: str, k: int = 5) -> list[str]:
        query_emb = self.embedder.encode([query])[0]
        scores = np.dot(self.embeddings, query_emb) / (
            np.linalg.norm(self.embeddings, axis=1) * np.linalg.norm(query_emb)
        )
        top_k = np.argsort(scores)[-k:][::-1]
        return [self.chunks[i] for i in top_k]

    def generate(self, query: str, k: int = 5) -> str:
        chunks = self.retrieve(query, k)
        context = "\n\n".join(chunks)
        prompt = f"""Answer based on context. Say "I don't know" if unsure.

Context:
{context}

Question: {query}

Answer:"""
        return openai.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
        ).choices[0].message.content

# Test it
rag = RAGFromScratch(["Your documents here..."])
print(rag.generate("Your question here"))
```

## Challenge

After it works:
1. Add reranking (use a CrossEncoder)
2. Add query rewriting
3. Add streaming
4. Measure faithfulness with RAGAS
