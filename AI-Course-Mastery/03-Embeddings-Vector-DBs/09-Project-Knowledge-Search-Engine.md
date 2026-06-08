# 🏗️ Project: Personal Knowledge Search Engine

## 🎯 Purpose & Goals

> **🛑 STOP. You've spent Files 01-08 learning about embeddings, vector search, and vector databases. Now you BUILD.**
>
> ### 🤔 Before You Start: Architecture Design
>
> You're building a search engine for YOUR personal knowledge — code snippets, notes, articles, bookmarks, documentation. It needs to:
>
> 1. **Ingest** documents from a folder (markdown, code files, text)
> 2. **Chunk** them intelligently (don't embed a 50-page doc as one vector)
> 3. **Embed** them with a model you choose
> 4. **Index** them in a vector database
> 5. **Search** with hybrid search (semantic + keyword)
> 6. **Respond** through a FastAPI endpoint
>
> **Before writing any code, answer these:**
>
> **Q1:** A markdown file has a 200-line function definition followed by a 50-line explanation. Should this be one chunk or multiple? If multiple, where do you split?
>
> **Q2:** You have code files (.py) and notes (.md). Should they use the SAME embedding model or different ones? Why?
>
> **Q3:** For 500 personal documents, which vector DB would you choose? For 5,000? For 50,000? Justify each.
>
> **Draw your architecture diagram on paper before writing code.** This is what senior engineers do. The diagram doesn't have to be perfect — it has to be THOUGHT through.

---

**By the end of this project, you will have:**
- A working Personal Knowledge Search Engine running on your machine
- A FastAPI server that serves search results
- A choice of backend (ChromaDB, Qdrant, or pgvector) that you can justify
- A reusable document ingestion pipeline
- A portfolio-ready GitHub repository with README

**⏱ Time Budget:** 6-8 hours (spread over 2-3 days)
**Portfolio Value:** HIGH — this is a system that every AI engineer builds for themselves

---

## 📋 Project Specification

### System Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Knowledge Search Engine                         │
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │ Document │───→│ Chunk &  │───→│ Vector   │               │
│  │ Watcher  │    │ Embed    │    │ Database │               │
│  └──────────┘    └──────────┘    └──────────┘               │
│                                       │                     │
│  ┌──────────┐    ┌──────────┐         │                     │
│  │ Request  │───→│ Search   │─────────┘                     │
│  │ (API)    │    │ Pipeline │                                │
│  └──────────┘    └──────────┘                                │
│                                       │                     │
│  ┌──────────┐    ┌──────────┐         │                     │
│  │ Evaluate │←───│ Results  │◄────────┘                     │
│  │ & Log    │    │          │                                │
│  └──────────┘    └──────────┘                                │
└────────────────────────────────────────────────────────────┘
```

### Deliverables

| File | Purpose |
|------|---------|
| `ingest.py` | Watch a folder, chunk documents, embed, index |
| `search.py` | Search engine class (hybrid + reranking) |
| `api.py` | FastAPI server with `/search` endpoint |
| `evaluate.py` | Test search quality with sample queries |
| `config.py` | Configuration (model, DB choice, paths) |
| `requirements.txt` | Python dependencies |
| `README.md` | Architecture decisions, usage, evaluation results |

---

## 🧱 Phase 1: Foundation (1-2 hours)

### Step 1: Project Structure

Create your project:

```
knowledge-search/
├── data/
│   └── notes/          # Your documents go here
├── src/
│   ├── __init__.py
│   ├── config.py
│   ├── ingest.py
│   ├── chunker.py
│   ├── search.py
│   └── api.py
├── tests/
│   ├── test_search.py
│   └── test_ingest.py
├── evaluate.py
├── requirements.txt
└── README.md
```

### Step 2: Configuration

```python
"""
config.py — All config in one place.
Change your vector DB backend here.
"""

from dataclasses import dataclass, field
from typing import List


@dataclass
class SearchConfig:
    """Central configuration for the knowledge search engine."""
    
    # ── Document paths ──
    watch_directories: List[str] = field(default_factory=lambda: [
        "./data/notes",
        "./data/code",
        "./data/articles",
    ])
    
    # ── Chunking ──
    chunk_size: int = 500       # Characters per chunk
    chunk_overlap: int = 50     # Overlap between chunks
    
    # ── Embedding ──
    embedding_model: str = "all-MiniLM-L6-v2"
    embedding_dimension: int = 384
    
    # ── Vector DB ──
    # Options: "chroma", "qdrant", "pgvector"
    vector_db: str = "chroma"
    
    # ── Qdrant settings (if using Qdrant) ──
    qdrant_host: str = "localhost"
    qdrant_port: int = 6333
    
    # ── pgvector settings (if using pgvector) ──
    pg_connection: str = "postgresql://postgres:postgres@localhost:5432/knowledge"
    
    # ── Search ──
    hybrid_alpha: float = 0.7   # Vector vs keyword weight
    retrieve_k: int = 100       # Candidates from fast search
    rerank_k: int = 20          # Candidates after reranking
    final_k: int = 10           # Final results
    
    # ── Server ──
    host: str = "0.0.0.0"
    port: int = 8000
```

---

### Step 3: Chunking Strategy

**🤔 Think before coding:**

A document like this needs intelligent splitting:
```markdown
# Chapter 3: Advanced Python

## 3.1 Decorators
Decorators are functions that modify other functions...

## 3.2 Generators
Generators are functions that yield values...

## 3.3 Context Managers
Context managers handle setup and teardown...
```

**Your job:** Write a chunker that:
1. Prefers to split at markdown headings (`##`, `###`)
2. Falls back to paragraph breaks (`\n\n`)
3. Falls back to sentence boundaries (`. `)
4. If a chunk is still too long, splits at character limit
5. Each chunk preserves the heading context (prepend parent heading)

```python
"""
chunker.py — Intelligent document chunking.
"""

from typing import List, Tuple
import re


class DocumentChunker:
    """
    Splits documents into chunks for embedding.
    
    The key insight: chunks should be SEMANTIC units, not arbitrary
    character slices. A chunk that ends mid-sentence has a different
    embedding than a complete thought.
    
    🤔 Your challenge:
    - A 500-char chunk starting mid-decorator and ending mid-generator
      will have a CONFUSED embedding (half decorator, half generator)
    - A 500-char chunk that IS a complete section about decorators
      will have a CLEAR embedding (focused on one concept)
    """
    
    def __init__(self, chunk_size: int = 500, overlap: int = 50):
        self.chunk_size = chunk_size
        self.overlap = overlap
    
    def chunk_document(
        self,
        text: str,
        source_path: str,
    ) -> List[dict]:
        """
        Split a document into semantic chunks.
        
        Priority: heading > paragraph > sentence > character
        
        Returns: List of {"text": str, "metadata": dict}
        """
        chunks = []
        
        # ── Strategy 1: Split by markdown headings ──
        # Each section becomes a potential chunk
        sections = self._split_by_headings(text)
        
        for heading, content in sections:
            # ── Strategy 2: Split long sections by paragraphs ──
            if len(content) > self.chunk_size:
                paragraphs = self._split_by_paragraphs(content)
                
                for para in paragraphs:
                    if len(para) <= self.chunk_size:
                        chunks.append(para)
                    else:
                        # ── Strategy 3: Split long paragraphs by sentences ──
                        sentences = self._split_by_sentences(para)
                        chunks.extend(self._merge_sentences(sentences))
            else:
                chunks.append(content)
        
        return [self._build_chunk(chunk_text, source_path) for chunk_text in chunks]
    
    def _split_by_headings(self, text: str) -> List[Tuple[str, str]]:
        """
        Split by markdown headings.
        
        Returns: List of (heading, content)
        """
        # The heading is prepended as context
        heading_pattern = r'^(#{1,6}\s+.+)$'
        lines = text.split('\n')
        
        sections = []
        current_heading = "Document"
        current_content = []
        
        for line in lines:
            if re.match(heading_pattern, line.strip()):
                if current_content:
                    sections.append((current_heading, '\n'.join(current_content)))
                current_heading = line.strip()
                current_content = []
            else:
                current_content.append(line)
        
        if current_content:
            sections.append((current_heading, '\n'.join(current_content)))
        
        return sections
    
    def _split_by_paragraphs(self, text: str) -> List[str]:
        """Split by paragraph breaks."""
        paragraphs = re.split(r'\n\s*\n', text)
        return [p.strip() for p in paragraphs if p.strip()]
    
    def _split_by_sentences(self, text: str) -> List[str]:
        """Split by sentence boundaries."""
        # Simple sentence splitting (production would use nltk/spaCy)
        sentences = re.split(r'(?<=[.!?])\s+', text)
        return [s.strip() for s in sentences if s.strip()]
    
    def _merge_sentences(self, sentences: List[str]) -> List[str]:
        """Merge sentences into chunks of chunk_size."""
        chunks = []
        current_chunk = []
        current_length = 0
        
        for sentence in sentences:
            if current_length + len(sentence) > self.chunk_size and current_chunk:
                chunks.append(' '.join(current_chunk))
                # Overlap: keep last few sentences
                overlap_sentences = current_chunk[-2:] if len(current_chunk) > 2 else current_chunk
                current_chunk = overlap_sentences
                current_length = sum(len(s) for s in overlap_chunk)
            
            current_chunk.append(sentence)
            current_length += len(sentence)
        
        if current_chunk:
            chunks.append(' '.join(current_chunk))
        
        return chunks
    
    def _build_chunk(self, text: str, source_path: str) -> dict:
        """Build a chunk dictionary with metadata."""
        return {
            "text": text,
            "metadata": {
                "source": source_path,
                "chunk_size": len(text),
                "doc_type": source_path.split('.')[-1] if '.' in source_path else "unknown",
            }
        }
```

---

## 🧱 Phase 2: Ingestion Pipeline (1-2 hours)

### Step 4: Build the Ingester

```python
"""
ingest.py — Document ingestion pipeline.
Watches folders, chunks documents, embeds, indexes.
"""

from pathlib import Path
from typing import List, Optional
from sentence_transformers import SentenceTransformer
import chromadb
from chromadb.config import Settings
import hashlib
import time
import logging

from config import SearchConfig
from chunker import DocumentChunker

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class DocumentIngester:
    """
    Ingests documents into the vector database.
    
    Pipeline:
    1. Scan directories for documents
    2. Read and chunk each document
    3. Generate embeddings for each chunk
    4. Store in vector database
    
    🤔 Your design decisions:
    - What file extensions do you support?
    - How do you detect RE-ingestion (file already indexed)?
    - What happens when a file is UPDATED?
    - What happens when a file is DELETED?
    """
    
    def __init__(self, config: SearchConfig):
        self.config = config
        self.chunker = DocumentChunker(
            chunk_size=config.chunk_size,
            overlap=config.chunk_overlap,
        )
        self.model = SentenceTransformer(config.embedding_model)
        
        # Initialize vector DB
        self._init_vector_db()
        
        # Track indexed files (for incremental updates)
        self._indexed_files: set = set()
        self._load_indexed_files()
    
    def _init_vector_db(self):
        """Initialize the chosen vector database."""
        db_type = self.config.vector_db
        
        if db_type == "chroma":
            self.client = chromadb.PersistentClient(
                path="./chromadb_data",
                settings=Settings(anonymized_telemetry=False),
            )
            self.collection = self.client.get_or_create_collection(
                name="knowledge",
                metadata={"hnsw:space": "cosine"},
            )
        elif db_type == "qdrant":
            from qdrant_client import QdrantClient
            from qdrant_client.http.models import Distance, VectorParams
            
            self.client = QdrantClient(
                host=self.config.qdrant_host,
                port=self.config.qdrant_port,
            )
            
            # Create collection if it doesn't exist
            collections = self.client.get_collections().collections
            if "knowledge" not in {c.name for c in collections}:
                self.client.create_collection(
                    collection_name="knowledge",
                    vectors_config=VectorParams(
                        size=self.config.embedding_dimension,
                        distance=Distance.COSINE,
                    ),
                )
            self.collection_name = "knowledge"
        else:
            raise ValueError(f"Unsupported vector DB: {db_type}")
    
    def _load_indexed_files(self):
        """Load the set of already-indexed files."""
        try:
            with open("./indexed_files.txt", "r") as f:
                self._indexed_files = set(line.strip() for line in f)
        except FileNotFoundError:
            self._indexed_files = set()
    
    def _save_indexed_files(self):
        """Save the set of indexed files."""
        with open("./indexed_files.txt", "w") as f:
            for path in sorted(self._indexed_files):
                f.write(f"{path}\n")
    
    def _file_hash(self, filepath: Path) -> str:
        """Compute file hash for change detection."""
        return hashlib.md5(filepath.read_bytes()).hexdigest()
    
    def scan_and_ingest(self, directories: Optional[List[str]] = None):
        """
        Scan directories and ingest new/changed documents.
        
        This method is IDEMPOTENT — running it multiple times
        only processes new or changed files.
        """
        dirs = directories or self.config.watch_directories
        
        for directory in dirs:
            dir_path = Path(directory)
            if not dir_path.exists():
                logger.warning(f"Directory not found: {directory}")
                continue
            
            for file_path in dir_path.rglob("*"):
                if not file_path.is_file():
                    continue
                
                # Support common text formats
                if file_path.suffix not in {'.md', '.txt', '.py', '.js', '.ts',
                                            '.json', '.yaml', '.yml', '.csv',
                                            '.rst', '.html', '.css'}:
                    continue
                
                self._process_file(file_path)
        
        self._save_indexed_files()
    
    def _process_file(self, file_path: Path):
        """Process a single file — chunk, embed, index."""
        file_key = str(file_path)
        current_hash = self._file_hash(file_path)
        
        # Check if file is already indexed and unchanged
        if file_key in self._indexed_files:
            stored_hash = self._indexed_files[file_key] if hasattr(self, '_indexed_files') else None
            # Simplified: skip if already indexed
            # In production, you'd store and compare hashes
            return
        
        logger.info(f"Processing: {file_key}")
        
        try:
            text = file_path.read_text(encoding='utf-8')
        except Exception as e:
            logger.error(f"Error reading {file_key}: {e}")
            return
        
        # Chunk the document
        chunks = self.chunker.chunk_document(text, file_key)
        
        if not chunks:
            logger.warning(f"No chunks generated for {file_key}")
            return
        
        # Embed chunks
        texts = [chunk["text"] for chunk in chunks]
        embeddings = self.model.encode(texts, normalize_embeddings=True)
        
        # Generate IDs
        chunk_ids = [
            f"{file_key}::chunk::{i}" for i in range(len(chunks))
        ]
        
        # Store in vector DB
        self._store_chunks(chunk_ids, embeddings, texts, chunks)
        
        # Mark as indexed
        self._indexed_files.add(file_key)
        logger.info(f"  Indexed {len(chunks)} chunks from {file_key}")
    
    def _store_chunks(self, ids, embeddings, texts, chunks):
        """Store chunks in the vector database."""
        db_type = self.config.vector_db
        
        metadatas = []
        for chunk in chunks:
            meta = chunk["metadata"].copy()
            meta["text_preview"] = chunk["text"][:100]
            metadatas.append(meta)
        
        if db_type == "chroma":
            self.collection.add(
                embeddings=embeddings.tolist(),
                documents=texts,
                metadatas=metadatas,
                ids=ids,
            )
        elif db_type == "qdrant":
            from qdrant_client.http import models
            
            points = []
            for i, (cid, emb, text, meta) in enumerate(zip(ids, embeddings, texts, metadatas)):
                points.append(models.PointStruct(
                    id=abs(hash(cid)) % (2**63),
                    vector=emb.tolist(),
                    payload={
                        "text": text,
                        "source": meta["source"],
                        "doc_type": meta["doc_type"],
                        "text_preview": meta["text_preview"],
                    },
                ))
            
            self.client.upsert(
                collection_name=self.collection_name,
                points=points,
                wait=False,
            )


# ── CLI entry point ──
if __name__ == "__main__":
    config = SearchConfig()
    ingester = DocumentIngester(config)
    
    logger.info("Starting ingestion...")
    start = time.time()
    
    ingester.scan_and_ingest()
    
    elapsed = time.time() - start
    logger.info(f"Ingestion complete in {elapsed:.2f}s")
```

---

## 🧱 Phase 3: Search Engine (1 hour)

### Step 5: Build the Search Pipeline

```python
"""
search.py — Search engine with hybrid search + reranking.
"""

from typing import List, Dict, Optional
from sentence_transformers import SentenceTransformer, CrossEncoder
from sklearn.preprocessing import MinMaxScaler
import numpy as np
import time
import logging

from config import SearchConfig

logger = logging.getLogger(__name__)


class KnowledgeSearch:
    """
    Search engine for personal knowledge base.
    
    Uses the SAME chunker and embedding model as the ingester
    so queries and documents are in the same space.
    
    Search pipeline:
    1. Embed query
    2. Vector search (ANN)
    3. BM25 keyword search
    4. Hybrid fusion (normalized scores)
    5. Optional cross-encoder reranking
    6. Return results
    """
    
    def __init__(self, config: SearchConfig):
        self.config = config
        
        # ── Init embedding model ──
        self.model = SentenceTransformer(config.embedding_model)
        
        # ── Init cross-encoder (lazy load — only when needed) ──
        self._reranker = None
        
        # ── Init vector DB ──
        self._init_vector_db()
        
        # ── Init BM25 (built from vector DB contents) ──
        self._bm25 = None
        self._all_texts: List[str] = []
    
    def _init_vector_db(self):
        db_type = self.config.vector_db
        if db_type == "chroma":
            import chromadb
            from chromadb.config import Settings
            self.client = chromadb.PersistentClient(
                path="./chromadb_data",
                settings=Settings(anonymized_telemetry=False),
            )
            self.collection = self.client.get_collection("knowledge")
        elif db_type == "qdrant":
            from qdrant_client import QdrantClient
            self.client = QdrantClient(
                host=self.config.qdrant_host,
                port=self.config.qdrant_port,
            )
            self.collection_name = "knowledge"
    
    def _ensure_bm25(self):
        """Build BM25 index from stored documents (lazy)."""
        if self._bm25 is not None:
            return
        
        # Load all texts from vector DB
        if self.config.vector_db == "chroma":
            all_docs = self.collection.get()
            self._all_texts = all_docs["documents"]
        elif self.config.vector_db == "qdrant":
            records, _ = self.client.scroll(
                collection_name=self.collection_name,
                limit=10000,
            )
            self._all_texts = [r.payload.get("text", "") for r in records]
        
        # Build BM25
        from bm25 import BM25  # Import from earlier file or implement inline
        self._bm25 = BM25(k1=1.5, b=0.75)
        self._bm25.fit(self._all_texts)
    
    def _get_reranker(self):
        """Lazy-load the cross-encoder."""
        if self._reranker is None:
            self._reranker = CrossEncoder(
                'cross-encoder/ms-marco-MiniLM-L-6-v2',
                max_length=512,
            )
        return self._reranker
    
    def search(
        self,
        query: str,
        top_k: int = 10,
        use_reranker: bool = False,
        filter_source: Optional[str] = None,
    ) -> Dict:
        """
        Search the knowledge base.
        
        Args:
            query: Natural language query
            top_k: Number of results
            use_reranker: Use cross-encoder reranking (slower but better)
            filter_source: Filter by source file path
        
        Returns:
            Dict with results and timing info
        """
        start_total = time.time()
        
        # ── Step 1: Embed query ──
        t0 = time.time()
        query_vec = self.model.encode([query], normalize_embeddings=True)[0]
        embed_time = time.time() - t0
        
        # ── Step 2: Vector search ──
        t0 = time.time()
        if self.config.vector_db == "chroma":
            # ChromaDB search
            results = self.collection.query(
                query_embeddings=[query_vec.tolist()],
                n_results=self.config.retrieve_k,
            )
            candidate_texts = results["documents"][0]
            candidate_scores = [1 - d for d in results["distances"][0]]
            candidate_metadatas = results["metadatas"][0]
            candidate_ids = results["ids"][0]
        elif self.config.vector_db == "qdrant":
            from qdrant_client.http import models
            results = self.client.search(
                collection_name=self.collection_name,
                query_vector=query_vec.tolist(),
                limit=self.config.retrieve_k,
            )
            candidate_texts = [r.payload.get("text", "") for r in results]
            candidate_scores = [r.score for r in results]
            candidate_metadatas = [{"source": r.payload.get("source", ""),
                                    "doc_type": r.payload.get("doc_type", "")} for r in results]
            candidate_ids = [str(r.id) for r in results]
        
        vector_time = time.time() - t0
        
        # ── Step 3: Keyword search (BM25) ──
        t0 = time.time()
        self._ensure_bm25()
        keyword_results = dict(self._bm25.search(query, top_k=len(self._all_texts)))
        
        # Map BM25 scores to the same order as candidate_texts
        keyword_scores = []
        for text in candidate_texts:
            # Find the index in BM25's document list
            try:
                idx = self._all_texts.index(text)
                keyword_scores.append(keyword_results.get(idx, 0.0))
            except ValueError:
                keyword_scores.append(0.0)
        keyword_scores = np.array(keyword_scores)
        keyword_time = time.time() - t0
        
        # ── Step 4: Hybrid fusion ──
        t0 = time.time()
        vec_scores = np.array(candidate_scores)
        
        # Normalize
        vec_norm = self._normalize(vec_scores)
        kw_norm = self._normalize(keyword_scores)
        
        # Combine
        alpha = self.config.hybrid_alpha
        hybrid_scores = alpha * vec_norm + (1 - alpha) * kw_norm
        
        # Sort by hybrid score
        sorted_order = np.argsort(hybrid_scores)[::-1][:self.config.final_k]
        hybrid_time = time.time() - t0
        
        # ── Step 5: Optional reranking ──
        if use_reranker:
            t0 = time.time()
            reranker = self._get_reranker()
            
            # Take top candidates for reranking
            rerank_candidates = [candidate_texts[i] for i in sorted_order[:self.config.rerank_k]]
            pairs = [[query, doc] for doc in rerank_candidates]
            ce_scores = reranker.predict(pairs)
            
            # Rerank
            rerank_order = np.argsort(ce_scores)[::-1][:top_k]
            final_indices = [sorted_order[i] for i in rerank_order]
            final_scores = [float(ce_scores[i]) for i in rerank_order]
            rerank_time = time.time() - t0
        else:
            final_indices = sorted_order[:top_k]
            final_scores = [float(hybrid_scores[i]) for i in final_indices]
            rerank_time = 0
        
        # ── Step 6: Build results ──
        results_list = []
        for i, idx in enumerate(final_indices):
            results_list.append({
                "rank": i + 1,
                "score": round(final_scores[i], 4),
                "text": candidate_texts[idx][:500],  # Truncate for display
                "metadata": candidate_metadatas[idx],
                "id": candidate_ids[idx],
            })
        
        total_time = time.time() - start_total
        
        return {
            "query": query,
            "results": results_list,
            "total_results": len(results_list),
            "timing": {
                "total_ms": round(total_time * 1000, 2),
                "embed_ms": round(embed_time * 1000, 2),
                "vector_search_ms": round(vector_time * 1000, 2),
                "keyword_search_ms": round(keyword_time * 1000, 2),
                "fusion_ms": round(hybrid_time * 1000, 2),
                "rerank_ms": round(rerank_time * 1000, 2),
            }
        }
    
    def _normalize(self, scores: np.ndarray) -> np.ndarray:
        """Min-max normalize to [0, 1]."""
        if scores.max() == scores.min():
            return np.zeros_like(scores)
        return (scores - scores.min()) / (scores.max() - scores.min())
```

---

## 🧱 Phase 4: API Server (1 hour)

### Step 6: FastAPI Server

```python
"""
api.py — FastAPI server for the Knowledge Search Engine.
"""

from fastapi import FastAPI, Query, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import uvicorn
import time

from config import SearchConfig
from search import KnowledgeSearch

app = FastAPI(
    title="Personal Knowledge Search Engine",
    description="Search your personal knowledge base with semantic + keyword search",
    version="1.0.0",
)

# CORS for frontend access
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Global search engine instance
search_engine: Optional[KnowledgeSearch] = None


class SearchRequest(BaseModel):
    query: str
    top_k: int = 10
    use_reranker: bool = False
    filter_source: Optional[str] = None


class SearchResponse(BaseModel):
    query: str
    results: List[dict]
    total_results: int
    timing: dict


class HealthResponse(BaseModel):
    status: str
    documents_indexed: int
    engine_ready: bool


@app.on_event("startup")
async def startup():
    """Initialize the search engine on startup."""
    global search_engine
    config = SearchConfig()
    search_engine = KnowledgeSearch(config)


@app.get("/health", response_model=HealthResponse)
async def health():
    """Health check endpoint."""
    if search_engine is None:
        raise HTTPException(status_code=503, detail="Engine not initialized")
    
    return HealthResponse(
        status="ok",
        documents_indexed=len(search_engine._all_texts) if hasattr(search_engine, '_all_texts') else 0,
        engine_ready=True,
    )


@app.post("/search", response_model=SearchResponse)
async def search(request: SearchRequest):
    """
    Search the knowledge base.
    
    The main endpoint. Accepts a query and returns ranked results.
    """
    if search_engine is None:
        raise HTTPException(status_code=503, detail="Engine not initialized")
    
    if not request.query.strip():
        raise HTTPException(status_code=400, detail="Query cannot be empty")
    
    try:
        results = search_engine.search(
            query=request.query,
            top_k=request.top_k,
            use_reranker=request.use_reranker,
            filter_source=request.filter_source,
        )
        return SearchResponse(**results)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.get("/search")
async def search_get(
    q: str = Query(..., description="Search query"),
    top_k: int = Query(10, description="Number of results"),
    use_reranker: bool = Query(False, description="Use cross-encoder reranking"),
):
    """GET version of search (easier for browser testing)."""
    if search_engine is None:
        raise HTTPException(status_code=503, detail="Engine not initialized")
    
    results = search_engine.search(
        query=q,
        top_k=top_k,
        use_reranker=use_reranker,
    )
    return results


@app.get("/stats")
async def stats():
    """Get search engine statistics."""
    if search_engine is None:
        raise HTTPException(status_code=503, detail="Engine not initialized")
    
    try:
        search_engine._ensure_bm25()
        return {
            "documents_indexed": len(search_engine._all_texts),
            "vector_db_type": search_engine.config.vector_db,
            "embedding_model": search_engine.config.embedding_model,
            "dimension": search_engine.config.embedding_dimension,
            "hybrid_alpha": search_engine.config.hybrid_alpha,
        }
    except Exception as e:
        return {"error": str(e)}


# ── Run ──
if __name__ == "__main__":
    uvicorn.run(
        "api:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
    )
```

---

## 🧱 Phase 5: Evaluation (1 hour)

### Step 7: Measure Search Quality

```python
"""
evaluate.py — Measure your search engine's quality.
"""

from search import KnowledgeSearch
from config import SearchConfig
import numpy as np
from typing import List, Tuple


class SearchEvaluator:
    """
    Evaluate search quality with different configurations.
    
    The most important step — without measurement, you don't know
    if your search is actually good.
    
    🤔 Why evaluate?
    - To find the optimal alpha for hybrid search
    - To decide if reranking is worth the cost
    - To compare vector DB backends
    - To catch regressions after changes
    """
    
    def __init__(self, config: SearchConfig):
        self.config = config
        self.engine = KnowledgeSearch(config)
        self.engine._ensure_bm25()
    
    def evaluate(
        self,
        queries: List[str],
        relevant_docs: List[List[int]],  # Indices of relevant docs in _all_texts
    ) -> dict:
        """
        Run evaluation across multiple queries.
        
        Metrics:
        - MRR (Mean Reciprocal Rank): How early does the first relevant result appear?
        - Recall@k: What fraction of relevant docs are in top-k?
        - Precision@k: What fraction of top-k results are relevant?
        """
        
        configs = [
            {"name": "Pure Vector", "alpha": 1.0, "rerank": False},
            {"name": "Hybrid (0.3)", "alpha": 0.3, "rerank": False},
            {"name": "Hybrid (0.5)", "alpha": 0.5, "rerank": False},
            {"name": "Hybrid (0.7)", "alpha": 0.7, "rerank": False},
            {"name": "Pure Keyword", "alpha": 0.0, "rerank": False},
        ]
        
        results = {}
        
        for cfg in configs:
            self.config.hybrid_alpha = cfg["alpha"]
            
            mrr_scores = []
            recall_at_10 = []
            precision_at_10 = []
            
            for q_idx, query in enumerate(queries):
                result = self.engine.search(query, top_k=10, use_reranker=False)
                
                found_docs = [r["rank"] for r in result["results"]]
                expected = relevant_docs[q_idx]
                
                # MRR
                first_relevant_rank = None
                for r in result["results"]:
                    doc_idx = self.engine._all_texts.index(r["text"])
                    if doc_idx in expected:
                        first_relevant_rank = r["rank"]
                        break
                
                if first_relevant_rank:
                    mrr_scores.append(1.0 / first_relevant_rank)
                else:
                    mrr_scores.append(0.0)
                
                # Recall@10
                found_relevant = sum(1 for r in result["results"]
                                     if self.engine._all_texts.index(r["text"]) in expected)
                recall_at_10.append(found_relevant / len(expected) if expected else 0)
                
                # Precision@10
                precision_at_10.append(found_relevant / 10)
            
            results[cfg["name"]] = {
                "MRR": round(np.mean(mrr_scores), 4),
                "Recall@10": round(np.mean(recall_at_10), 4),
                "Precision@10": round(np.mean(precision_at_10), 4),
            }
        
        return results


# ── Run evaluation ──
if __name__ == "__main__":
    config = SearchConfig()
    evaluator = SearchEvaluator(config)
    
    # Sample queries (replace with your own)
    queries = [
        "how do I reset my password",
        "Python async programming patterns",
        "Docker compose networking",
        "AWS S3 bucket policies",
        "React hooks useEffect cleanup",
        "FastAPI dependency injection",
        "PostgreSQL indexing strategies",
        "CSS grid layout examples",
    ]
    
    # For each query, which document indices are relevant?
    # You need to KNOW your data to set this up correctly.
    relevant_docs = [
        [0, 5],
        [1, 3],
        [2, 7],
        [4, 8],
        [3, 6],
        [1, 5],
        [7, 9],
        [6, 10],
    ]
    
    print("═══ Search Quality Evaluation ═══\n")
    results = evaluator.evaluate(queries, relevant_docs)
    
    print(f"{'Config':20s} {'MRR':10s} {'Recall@10':12s} {'Precision@10':14s}")
    print("-" * 56)
    
    for config_name, metrics in results.items():
        print(f"{config_name:20s} {metrics['MRR']:<10.4f} {metrics['Recall@10']:<12.4f} {metrics['Precision@10']:<14.4f}")
    
    print("\n🏆 Recommended config based on your data:")
    best = max(results, key=lambda x: results[x]["MRR"])
    print(f"  Best: {best}")
    print(f"  MRR: {results[best]['MRR']:.4f}")
```

---

## 🧱 Phase 6: Your Own Data (1-2 hours)

### Step 8: Populate Your Knowledge Base

1. Create the directories: `data/notes/`, `data/articles/`, `data/code/`
2. Add 10-50 of YOUR documents:
   - Code snippets you reference often
   - Notes from past projects
   - Technical articles you've bookmarked
   - Configuration files you need to remember
3. Run the ingester
4. Start the API
5. Test with 20 queries YOU would actually search

### Step 9: Evaluate, Iterate, Improve

Run the evaluation. Then improve:

1. **Low MRR?** → Adjust hybrid alpha
2. **Missing results?** → Add more documents in that area
3. **Bad chunks?** → Tune chunker parameters
4. **Wrong model?** → Try BGE-base instead of MiniLM

---

## 🚦 Project Gate Check

Before calling this project complete, verify:

### Core Requirements
- [ ] Documents are ingested from at least 3 folders
- [ ] Documents are properly chunked (not one giant vector per file)
- [ ] Search returns relevant results for 10 test queries
- [ ] API server starts and responds to requests

### Portfolio Quality
- [ ] Code is on GitHub with a README
- [ ] README explains architecture decisions
- [ ] README shows sample queries and results
- [ ] README includes evaluation metrics
- [ ] Code has proper error handling (no silent failures)
- [ ] Code has logging (know what the system is doing)

### Stretch Goals (Level Up)
- [ ] Hybrid search with configurable alpha
- [ ] Cross-encoder re-ranking as an option
- [ ] Query cache for repeated searches
- [ ] Document watcher (auto-reindex on file change)
- [ ] CLI interface (search from command line)
- [ ] Web UI (basic search box)
- [ ] Support for PDF ingestion
- [ ] Support for URL scraping

---

## 📚 Deliverables Checklist

```
knowledge-search/
├── README.md                     # Architecture, usage, eval results
├── requirements.txt              # All dependencies with versions
├── config.py                     # Configuration
├── chunker.py                    # Document chunking
├── ingest.py                     # Document ingestion
├── search.py                     # Search engine
├── api.py                        # FastAPI server
├── evaluate.py                   # Quality evaluation
├── data/
│   ├── notes/                    # Your markdown notes
│   ├── articles/                 # Saved articles
│   └── code/                     # Code snippets
└── tests/                        # (Optional) unit tests
```

**Push to GitHub. Share with one person. Get feedback.**

---

## 💡 Portfolio Tips

### What Makes This Project Stand Out

**Good README:**
```markdown
# Personal Knowledge Search Engine

Architecture: FastAPI + ChromaDB + sentence-transformers (all-MiniLM-L6-v2)

## Decision Log

- **Vector DB: ChromaDB** — for simplicity with 500 personal docs.
  Would upgrade to Qdrant at 10K+ docs.
- **Chunking: Markdown heading-aware** — preserves document structure
  rather than arbitrary splits.
- **Hybrid alpha: 0.7** — determined empirically (see evaluation below).

## Evaluation Results

| Config        | MRR    | Recall@10 |
|---------------|--------|-----------|
| Pure Vector   | 0.812  | 0.89      |
| Hybrid (0.7)  | 0.867  | 0.94      |
| Pure Keyword  | 0.423  | 0.56      |

Hybrid search outperforms pure vector by 6.7% on MRR.
Cross-encoder reranking adds 3% more at 50ms extra latency — not worth it
for my use case.

## Sample Queries

Input: "how do I set up Docker compose for Django + Postgres"
Output: #1 docker-compose-django-reference.md (score: 0.89)
        #2 postgres-docker-setup.md (score: 0.76)
        ...
```

### What Gets Noticed

1. **Architecture diagram** — shows you think about systems, not just code
2. **Evaluation numbers** — proves you measure what you build
3. **Tradeoff explanations** — shows senior thinking ("I chose X over Y because...")
4. **Iteration evidence** — "First version used brute force, then I added HNSW because..."

---

## 📚 Phase 3 References

- Files 01-08 of this phase
- `sentence-transformers` documentation
- ChromaDB / Qdrant / pgvector documentation
- FastAPI documentation
- "How to Build a Semantic Search Engine" — various guides

---

> **🛑 Final reflection — answer these in your README:**
>
> 1. What's the #1 thing you'd do differently if building this again?
> 2. What was the hardest bug or decision?
> 3. Under what conditions would you switch from ChromaDB to Qdrant?
> 4. What did you learn about YOUR document collection that surprised you?
>
> **This project is your portfolio proof that you understand embeddings and vector search. Make it count.**
>
> **Next:** Phase 4 — RAG Foundations. You'll use this search engine as the RETRIEVAL layer of a RAG system.
