# Chunking Strategies

## Why Chunking Is Hard

Most RAG tutorials treat chunking as one line of code. In production, chunking strategy is the single biggest determinant of retrieval quality.

## The Tradeoff

```
Small chunks (100-200 tokens)
  Pros: High precision, easy to find exact info
  Cons: Loses context, may miss connections

Large chunks (500-1000 tokens)
  Pros: Better context, captures relationships
  Cons: Lower precision, more noise in prompt

Variable chunks (depends on content structure)
  Pros: Best of both
  Cons: Harder to implement
```

## Chunking Strategies (Ranked)

### 1. Recursive Character Split (Good baseline)
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", ".", " ", ""],
)
```
Splits on natural boundaries. Best default. 80% solution.

### 2. Semantic Chunking (Better)
```python
def semantic_chunk(text: str, embedder, threshold: float = 0.3):
    sentences = split_sentences(text)
    chunks = []
    current_chunk = [sentences[0]]

    for i in range(1, len(sentences)):
        emb_i = embedder.embed(sentences[i])
        emb_prev = embedder.embed(sentences[i-1])
        similarity = cosine_similarity(emb_i, emb_prev)

        if similarity < threshold:
            chunks.append(" ".join(current_chunk))
            current_chunk = []
        current_chunk.append(sentences[i])

    chunks.append(" ".join(current_chunk))
    return chunks
```
Splits where topic shifts. Better quality but computationally expensive.

### 3. Document Structure-Aware (Best)
```python
def structure_aware_chunk(markdown: str):
    """Split by headers, preserve hierarchy."""
    sections = []
    current_h1 = None
    current_h2 = None
    current_text = []

    for line in markdown.split("\n"):
        if line.startswith("# "):
            if current_text:
                sections.append(flush(current_h1, current_h2, current_text))
            current_h1 = line
            current_h2 = None
            current_text = []
        elif line.startswith("## "):
            if current_text:
                sections.append(flush(current_h1, current_h2, current_text))
            current_h2 = line
            current_text = []
        else:
            current_text.append(line)

    return sections
```
Uses document structure. Best for HTML, Markdown, PDFs with headings.

## 🔴 Senior: Chunking for Different Content

| Content Type | Strategy | Chunk Size | Overlap |
|-------------|----------|-----------|---------|
| Code | Function/class boundary | Per function | 0 |
| Documentation | Section headers | Per section | 1-2 sentences |
| News articles | Paragraph | 3-5 paragraphs | 1 paragraph |
| Research papers | Section (abstract, method, results) | Per section | None |
| Conversation | Turn boundary | Per 3-5 turns | 1 turn |

## Drill: Chunking Optimization Lab

Test all 5 strategies on your corpus. Measure:
- Recall@k (does the right chunk appear in top-k retrieved?)
- Context preservation (does the chunk make sense alone?)
- Computational cost (time per chunk)
- Storage efficiency (chunks per document)

Report which strategy wins for YOUR content type.
