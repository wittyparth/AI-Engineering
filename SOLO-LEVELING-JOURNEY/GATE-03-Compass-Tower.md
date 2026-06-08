# ⛩️ GATE 03 — Compass Tower (Embeddings & Vector Databases)

**Rank Requirement:** D+
**Theme:** Understanding embeddings, vector search, and the foundations of semantic retrieval
**Total Days:** 7 Dungeons + 1 Boss Battle
**Core Skills:** Text embeddings, cosine similarity, vector databases, semantic search

---

## Dungeon 1 — What Are Embeddings?

**Day 1 | Concept:** Embeddings are dense vector representations of meaning. Words/phrases with similar meaning are close in vector space.

**The Key Insight:**
- "King" - "Man" + "Woman" ≈ "Queen"
- Embeddings capture semantics, not just syntax
- Typical embedding dimension: 384 (all-MiniLM-L6-v2) to 3072 (text-embedding-3-large)

**🗡️ Dungeon Mastery:** Use sentence-transformers to embed 3 sentences. Compute which 2 are most similar semantically.

**💻 Code — First Embeddings:**
```python
from sentence_transformers import SentenceTransformer
import numpy as np

# Load model (tiny, runs locally)
model = SentenceTransformer('all-MiniLM-L6-v2')

sentences = [
    "The cat sat on the mat",
    "A dog is playing in the yard",
    "Machine learning transforms industries",
]

embeddings = model.encode(sentences)

print(f"Shape: {embeddings.shape}")  # (3, 384)
print(f"First 5 values: {embeddings[0][:5]}")

# Compute similarity matrix
sim_matrix = np.dot(embeddings, embeddings.T)
print(f"Similarity matrix:\n{sim_matrix}")
```

**👹 Boss Lesson:** Embeddings turn language into math. Once text is a vector, you can compute, search, and cluster it.

---

## Dungeon 2 — Cosine Similarity & Distance Metrics

**Day 2 | Concept:** Cosine similarity measures the angle between two vectors. Range: -1 (opposite) to 1 (identical). 0 = orthogonal/unrelated.

**Why Cosine?**
- Magnitude-independent — cares about direction, not length
- Works well for sparse and dense vectors
- Fast to compute on GPU/CPU

**Other metrics:**
| Metric | Best For | Range |
|--------|----------|-------|
| Cosine | Semantic similarity | [-1, 1] |
| Dot Product | Normalized vectors | [-1, 1] |
| Euclidean | Spatial clustering | [0, ∞) |
| Manhattan | High-dimensional sparse | [0, ∞) |

**🗡️ Dungeon Mastery:** Implement cosine similarity from scratch. Then use it to find the best-matching sentence for a query.

**💻 Code — Cosine Similarity:**
```python
import numpy as np

def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    """Compute cosine similarity between two vectors."""
    dot_product = np.dot(a, b)
    norm_a = np.linalg.norm(a)
    norm_b = np.linalg.norm(b)
    return dot_product / (norm_a * norm_b)

def semantic_search(query: str, documents: list[str], model, top_k: int = 3):
    """Find most semantically similar documents to query."""
    query_emb = model.encode([query])[0]
    doc_embs = model.encode(documents)

    scores = [cosine_similarity(query_emb, doc_emb) for doc_emb in doc_embs]
    top_indices = np.argsort(scores)[-top_k:][::-1]

    results = []
    for idx in top_indices:
        results.append({
            "document": documents[idx],
            "score": float(scores[idx]),
        })
    return results

# Test it
docs = [
    "Python is a programming language",
    "JavaScript runs in the browser",
    "Embeddings capture semantic meaning",
    "Vector databases enable similarity search",
]
query = "How do I search by meaning?"
results = semantic_search(query, docs, model)
for r in results:
    print(f"{r['score']:.3f}: {r['document']}")
```

**👹 Boss Lesson:** Cosine similarity is the workhorse of semantic search. Know it, love it, implement it from memory.

---

## Dungeon 3 — Chunking Strategies

**Day 3 | Concept:** Before embedding, you must chunk documents. Chunk size and strategy directly impact retrieval quality.

**Chunking Approaches:**
| Strategy | Chunk Size | Overlap | Use Case |
|----------|------------|---------|----------|
| Fixed | 256-512 tokens | 10-20% | General purpose |
| Sentence | 1-5 sentences | 0-1 sentences | Q&A, facts |
| Paragraph | 1-3 paragraphs | 0 | Long-form content |
| Semantic | Variable | N/A | Best quality, hardest |

**💻 Code — Chunkers:**
```python
import tiktoken

def fixed_chunker(text: str, chunk_size: int = 500, overlap: int = 50):
    """Fixed-size token chunking with overlap."""
    encoder = tiktoken.get_encoding("cl100k_base")
    tokens = encoder.encode(text)

    chunks = []
    start = 0
    while start < len(tokens):
        end = start + chunk_size
        chunk_tokens = tokens[start:end]
        chunks.append(encoder.decode(chunk_tokens))
        start += chunk_size - overlap

    return chunks

def sentence_chunker(text: str, max_sentences: int = 3):
    """Sentence-aware chunking."""
    import nltk
    sentences = nltk.sent_tokenize(text)

    chunks = []
    current = []
    for sent in sentences:
        current.append(sent)
        if len(current) >= max_sentences:
            chunks.append(" ".join(current))
            current = []
    if current:
        chunks.append(" ".join(current))
    return chunks

def semantic_chunker(text: str, model, threshold: float = 0.7):
    """Chunk at semantic boundaries (topic shifts)."""
    sentences = nltk.sent_tokenize(text)
    if len(sentences) <= 1:
        return [text]

    # Embed each sentence
    embeddings = model.encode(sentences)

    chunks = []
    current = [sentences[0]]

    for i in range(1, len(sentences)):
        sim = cosine_similarity(embeddings[i-1], embeddings[i])
        if sim < threshold:
            # Topic shift detected
            chunks.append(" ".join(current))
            current = []
        current.append(sentences[i])

    if current:
        chunks.append(" ".join(current))
    return chunks
```

**🗡️ Dungeon Mastery:** Take a 1000-word article. Chunk it with all 3 strategies. Count chunks per strategy. Which gives the most coherent chunks?

**👹 Boss Lesson:** Bad chunking = bad retrieval. Spend time on chunking strategy — it's the most underrated hyperparameter in RAG.

---

## Dungeon 4 — Vector Database Fundamentals

**Day 4 | Concept:** Vector databases store embeddings and enable fast approximate nearest neighbor (ANN) search. Unlike traditional DBs, they index by vector distance.

**Key Concepts:**
- **Index:** Data structure for fast ANN search (HNSW, IVF, PQ)
- **HNSW** (Hierarchical Navigable Small World): Graph-based, very fast, high memory
- **IVF** (Inverted File Index): Cluster-based, slower but memory efficient
- **PQ** (Product Quantization): Compresses vectors for massive scale

**🗡️ Dungeon Mastery:** Set up ChromaDB (embedded, local). Insert 10 documents with embeddings. Query with a semantic search.

**💻 Code — ChromaDB Quickstart:**
```python
import chromadb
from chromadb.utils import embedding_functions

# Initialize
client = chromadb.Client()  # In-memory (use PersistentClient for persistence)
sentence_transformer_ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2"
)

# Create collection
collection = client.create_collection(
    name="my_docs",
    embedding_function=sentence_transformer_ef,
    metadata={"hnsw:space": "cosine"},
)

# Add documents
collection.add(
    documents=[
        "Embeddings capture semantic meaning of text",
        "Vector databases enable fast similarity search",
        "Chunking strategies impact retrieval quality",
        "Cosine similarity measures angle between vectors",
        "HNSW is a popular ANN algorithm",
    ],
    ids=["doc1", "doc2", "doc3", "doc4", "doc5"],
)

# Query
results = collection.query(
    query_texts=["How do I search through vectors?"],
    n_results=3,
)

for doc, dist in zip(results['documents'][0], results['distances'][0]):
    print(f"{dist:.3f}: {doc}")
```

**👹 Boss Lesson:** Vector DBs are the database equivalent of LLMs — new paradigm that requires new thinking. Relational joins become vector similarity.

---

## Dungeon 5 — Dense vs Sparse Embeddings

**Day 5 | Concept:** Dense embeddings (BERT-based) capture semantic meaning. Sparse embeddings (BM25, TF-IDF) capture exact keyword matching. Hybrid search combines both.

**Dense (all-MiniLM-L6-v2):**
- Pros: Understands synonyms, paraphrasing, context
- Cons: Expensive to compute, needs GPU for speed

**Sparse (BM25):**
- Pros: Fast, interpretable, exact keyword match
- Cons: Misses semantics ("car" ≠ "vehicle")

**Hybrid:**
```
score = 0.3 * dense_score + 0.7 * sparse_score
```

**🗡️ Dungeon Mastery:** Implement hybrid search. Query for "cheap automobile insurance" — dense should find "affordable car coverage", sparse should not. See why hybrid wins.

**💻 Code — Hybrid Search:**
```python
from rank_bm25 import BM25Okapi
from sentence_transformers import SentenceTransformer
import numpy as np

class HybridSearch:
    def __init__(self, documents: list[str], dense_weight: float = 0.5):
        self.documents = documents
        self.dense_weight = dense_weight
        self.sparse_weight = 1 - dense_weight

        # Dense
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.dense_embeddings = self.model.encode(documents)

        # Sparse
        tokenized = [doc.split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized)

    def search(self, query: str, top_k: int = 5):
        # Dense scores
        query_emb = self.model.encode([query])[0]
        dense_scores = np.dot(self.dense_embeddings, query_emb)

        # Sparse scores
        tokenized_query = query.split()
        sparse_scores = self.bm25.get_scores(tokenized_query)

        # Normalize scores to [0, 1]
        dense_scores = (dense_scores - dense_scores.min()) / (dense_scores.max() - dense_scores.min() + 1e-8)
        sparse_scores = (sparse_scores - sparse_scores.min()) / (sparse_scores.max() - sparse_scores.min() + 1e-8)

        # Combine
        hybrid_scores = self.dense_weight * dense_scores + self.sparse_weight * sparse_scores

        top_indices = np.argsort(hybrid_scores)[-top_k:][::-1]
        return [(self.documents[i], float(hybrid_scores[i])) for i in top_indices]

# Test
docs = [
    "Cheap car insurance for young drivers",
    "Affordable automobile coverage",
    "Health insurance plans for families",
    "Motorcycle insurance quotes online",
]
search = HybridSearch(docs)
results = search.search("cheap automobile insurance")
for doc, score in results:
    print(f"{score:.3f}: {doc}")
```

**👹 Boss Lesson:** Hybrid search is the industry standard. Pure dense misses keywords, pure sparse misses semantics. Together they're unbeatable.

---

## Dungeon 6 — Embedding Models Compared

**Day 6 | Concept:** Different embedding models trade off speed, quality, and dimension. Pick the right one for your use case.

| Model | Dimensions | Speed | Quality | Size |
|-------|-----------|-------|---------|------|
| all-MiniLM-L6-v2 | 384 | Fast | Good | 80MB |
| all-mpnet-base-v2 | 768 | Medium | Better | 420MB |
| text-embedding-3-small | 1536 | Fast (API) | Great | API |
| text-embedding-3-large | 3072 | Slow (API) | Best | API |
| BGE-large-en-v1.5 | 1024 | Medium | Great | 1.3GB |

**🗡️ Dungeon Mastery:** Take the MTEB benchmark leaderboard. Pick 3 models. Embed the same 10 sentences. Compute pairwise similarities. Compare consistency across models.

**💻 Code — Model Comparison:**
```python
def compare_embedding_models(texts: list[str], model_names: list[str]):
    results = {}
    for name in model_names:
        print(f"Loading {name}...")
        model = SentenceTransformer(name)
        embeddings = model.encode(texts, show_progress_bar=False)

        # Compute self-similarity (diagonal should be ~1.0)
        sims = np.dot(embeddings, embeddings.T)
        avg_sim = np.mean(sims[np.triu_indices_from(sims, k=1)])

        results[name] = {
            "dimension": embeddings.shape[1],
            "avg_similarity": float(avg_sim),
            "embeddings": embeddings,
        }
        print(f"  Dim={embeddings.shape[1]}, AvgSim={avg_sim:.3f}")

    return results

# models = ["all-MiniLM-L6-v2", "all-mpnet-base-v2", "BAAI/bge-small-en-v1.5"]
# results = compare_embedding_models(my_texts, models)
```

**👹 Boss Lesson:** Bigger embedding ≠ better retrieval. Match your model to your latency budget. all-MiniLM-L6-v2 is surprisingly competitive.

---

## Dungeon 7 — Multimodal Embeddings

**Day 7 | Concept:** Embeddings aren't just for text. Images, audio, and video can all be embedded into the same vector space for cross-modal search.

**CLIP (Contrastive Language-Image Pre-training):**
- Embeds text and images into the same space
- Enables text-to-image and image-to-image search
- Zero-shot classification without training

**🗡️ Dungeon Mastery:** Use CLIP to find the most matching image for a text query (or vice versa with local files).

**💻 Code — CLIP Similarity:**
```python
from PIL import Image
import requests
from transformers import CLIPProcessor, CLIPModel

# Load model
model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

# Prepare inputs
images = [Image.open("photo1.jpg"), Image.open("photo2.jpg")]
texts = ["A dog playing in the park", "A cat sleeping on a couch"]

inputs = processor(
    text=texts,
    images=images,
    return_tensors="pt",
    padding=True,
)

# Get similarity scores
outputs = model(**inputs)
logits_per_image = outputs.logits_per_image  # image-text similarity
probs = logits_per_image.softmax(dim=1)      # probabilities

print(f"Image-Text probabilities:\n{probs}")
```

**🗡️ Dungeon Mastery:** Find 3 images online. Write 3 captions. Use CLIP to match each image to its correct caption. Accuracy should be high.

**👹 Boss Lesson:** Embeddings are universal. Once everything is a vector, every modality can search every other modality.

---

## ⚔️ BOSS BATTLE: Knowledge Search Engine

**Objective:** Build a mini semantic search engine that:
1. Ingests 10+ documents on a topic of your choice
2. Chunks them with intelligent strategy
3. Embeds all chunks with sentence-transformers
4. Stores in ChromaDB (or in-memory FAISS)
5. Supports hybrid search (dense + sparse)
6. Returns top-3 results with similarity scores and source chunks

**🗡️ Victory Conditions:**
- Ingestion pipeline works end-to-end
- Hybrid search returns relevant results
- At least 10 documents indexed
- Search latency < 500ms
- Can handle queries with synonyms (not just keyword match)

**🏆 Rewards:**
- +500 XP
- D+ Rank → C-Rank
- Skill Unlock: **Vector Sense** — 30% faster at debugging embedding/retrieval issues
- Title: **The Compass Bearer**

---

*"In the space of meaning, distance is measured in degrees of understanding."*
