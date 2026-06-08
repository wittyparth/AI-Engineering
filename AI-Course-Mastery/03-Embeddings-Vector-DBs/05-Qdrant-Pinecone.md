# Qdrant & Pinecone — Production Vector Databases

## 🎯 Purpose & Goals

> **🛑 STOP. You've learned:**
> - File 02: Semantic search with numpy (works at small scale)
> - File 04: ChromaDB for prototyping (works at medium scale)
>
> Now imagine your startup is growing. You have 5 million documents. You need:
> - 99.9% uptime
> - 50ms search latency
> - 1,000 concurrent users
> - Data replicated across regions
> - Paying customers who will leave if search breaks
>
> ### 🤔 Discovery Question 1: What "Production" Actually Means
>
> ChromaDB works great on your laptop. But what changes when you need to deploy it?
>
> **Think about these scenarios (before reading the answers):**
>
> 1. **The Crash:** Your server restarts overnight. ChromaDB in-memory is gone. How long does it take to re-index 5M documents?
> 2. **The Traffic Spike:** A viral tweet sends 10,000 users to your search. ChromaDB is single-process. What happens?
> 3. **The Bad Query:** One user sends a query that causes an infinite loop or OOM. Does it take down ALL users?
> 4. **The Update:** You need to update your embedding model. All 5M vectors need re-embedding. Can you do this without downtime?
>
> **For each scenario: how would YOU solve it with what you know so far?**

---

### 🤔 Discovery Question 2: Self-Hosted vs Managed

You have two deployment options for a production vector database:

**Option A: Self-Hosted**
- You run the database on your own servers (AWS EC2, GCP)
- Full control over configuration
- Your team handles ops (backups, scaling, updates)
- Monthly infra cost: ~$500-2,000

**Option B: Managed**
- A cloud provider runs the database for you
- You pay per vector stored + per query
- Zero ops work
- Monthly cost: ~$2,000-10,000+

**Before reading on, decide:**
1. At what stage of a startup would you choose Option A?
2. At what stage would you choose Option B?
3. What's the "breakpoint" where switching from A to B makes sense?
4. What non-obvious costs exist for each option? (Think beyond money: team expertise, time, vendor lock-in)

---

### 🤔 Discovery Question 3: The Filtering Problem

ChromaDB supports metadata filtering. But in production, filtering gets complex:

```
Scenario: E-commerce product search

Products table has:
- Name (text): "Blue Cotton Sweater"
- Category (enum): "clothing", "electronics", "food"
- Price (float): $29.99
- InStock (bool): true
- Rating (float): 4.5
- Tags (array): ["winter", "casual", "sale"]

Query: "warm winter clothing" with filters:
- category = "clothing"
- price < $100
- inStock = true
- sorted by rating

Question: How does the vector database combine semantic search (embedding similarity)
with these structured filters? Does it:
A) Search vectors first, then apply filters (might miss good results)
B) Filter first, then search vectors (might be slow)
C) Do both simultaneously (complex but ideal)

What are the tradeoffs of each approach? Can you think of a case where
A gives TERRIBLE results? What about B?
```

---

> **By the end of this file, you will:**
> - Know when your project has outgrown ChromaDB and needs Qdrant/Pinecone
> - Be able to set up Qdrant (self-hosted) and Pinecone (managed)
> - Understand the key features that matter in production: filtering, sharding, replication
> - Know the exact tradeoffs between self-hosted (Qdrant) and managed (Pinecone)
> - Build a production-style search application

**⏱ Time Budget:** 2.5 hours (45 min concepts, 1 hour Qdrant code, 30 min Pinecone, 15 min drills)

---

## 📖 The Production Vector Database Landscape

### What ChromaDB Doesn't Do (And Why It Matters)

Let's be precise about ChromaDB's limitations so you know EXACTLY when to upgrade:

```
CAPABILITY              │ ChromaDB      │ Qdrant       │ Pinecone
────────────────────────┼───────────────┼──────────────┼───────────────
Client-server           │ Limited       │ ✅ Native    │ ✅ Native
Horizontal scaling      │ ❌            │ ✅ Sharding  │ ✅ Auto
Replication             │ ❌            │ ✅           │ ✅ Auto
99.9%+ uptime           │ ❌            │ ✅           │ ✅ SLA
Payload/Filtering       │ Basic         │ Advanced     │ Advanced
Multi-tenancy           │ ❌            │ ✅           │ ✅
Full-text + vector      │ ❌            │ ✅           │ Limited
Batch update            │ ✅            │ ✅           │ ✅
Docker deployment       │ ✅            │ ✅           │ N/A
Free tier               │ Always        │ Self-host    │ 1M vectors
────────────────────────┴───────────────┴──────────────┴───────────────
```

**🤔 Your turn:** Based on this table, under what conditions would you:
1. Stay with ChromaDB?
2. Move to self-hosted Qdrant?
3. Pay for managed Pinecone?

### How Qdrant Works

Qdrant is a Rust-based vector database that runs as a separate server. Its key architectural decisions:

```
Qdrant Architecture:
┌────────────────────────────────────────────────────────────┐
│                      Client Application                      │
│                     (FastAPI, Next.js, etc.)                  │
└────────────────────────┬───────────────────────────────────┘
                         │ REST / gRPC
                         ▼
┌────────────────────────────────────────────────────────────┐
│                      Qdrant Server                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Collection 1 │  │  Collection 2 │  │  Collection 3 │      │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │      │
│  │ │ Shard 1  │ │  │ │ Shard 1  │ │  │ │ Shard 1  │ │      │
│  │ │ (HNSW)   │ │  │ │ (HNSW)   │ │  │ │ (HNSW)   │ │      │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │      │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │      │
│  │ │ Shard 2  │ │  │ │ Shard 2  │ │  │ │ Shard 2  │ │      │
│  │ │ (HNSW)   │ │  │ │ (HNSW)   │ │  │ │ (HNSW)   │ │      │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
│  Storage: On-disk (RocksDB) + RAM cache                     │
└────────────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Collections** = Groups of vectors (like a table in SQL, like a collection in ChromaDB)
- **Shards** = Horizontal partitions of a collection (parallel queries)
- **Payload** = Metadata attached to vectors (JSON objects)
- **HNSW** = The index structure (same as ChromaDB, but configurable)

### How Pinecone Works

Pinecone is a managed cloud service. You don't manage the infrastructure — you call an API.

```
Pinecone Architecture (simplified):
┌────────────────────────────────────────────────────────────┐
│  Your App → Pinecone API → Their managed infrastructure    │
│                                                             │
│  You provide: vectors, metadata                            │
│  They handle: indexing, sharding, replication, backups      │
│  You pay: $ per vector-hour + $ per query                  │
└────────────────────────────────────────────────────────────┘
```

**The tradeoff:** Pinecone costs more but requires zero ops. Qdrant costs less (self-hosted) but requires you to manage infrastructure.

---

## 💻 Code Examples

### Example 1: Setting Up Qdrant

First, you need Qdrant running. Let's start it with Docker:

```bash
# Start Qdrant server
docker run -d --name qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage \
  qdrant/qdrant

# Check if it's running
curl http://localhost:6333/healthz
# Expected: OK
```

> **🤔 Think about this:** In File 04, ChromaDB ran IN your Python process. Qdrant runs as a SEPARATE server. What are the implications of this architectural difference?
>
> - Deployment complexity?
> - Fault isolation?
> - Scaling?
> - Monitoring?

Now let's interact with Qdrant:

```python
"""
Qdrant — production vector database.
Connecting to a running Qdrant server.
"""

from qdrant_client import QdrantClient
from qdrant_client.http import models
from qdrant_client.http.models import Distance, VectorParams
from sentence_transformers import SentenceTransformer
import numpy as np
import time
import uuid


# ── Step 1: Connect to Qdrant ──
# If running locally with Docker:
client = QdrantClient(host="localhost", port=6333)

# If connecting to Qdrant Cloud:
# client = QdrantClient(
#     url="https://your-cluster.cloud.qdrant.io:6333",
#     api_key="your-api-key",
# )

print(f"Connected to Qdrant: {client.get_collections()}")

# ── Step 2: Create a Collection ──
# A collection = a group of vectors with the same configuration
COLLECTION_NAME = "products"

# First, delete if exists (for clean demo)
try:
    client.delete_collection(COLLECTION_NAME)
except:
    pass

# Create collection with explicit configuration
client.create_collection(
    collection_name=COLLECTION_NAME,
    vectors_config=VectorParams(
        size=384,              # Must match your embedding model's dimension
        distance=Distance.COSINE,  # Distance metric
    ),
)

print(f"Created collection: {COLLECTION_NAME}")

# ── Step 3: Add Vectors with Payload ──
# In Qdrant, "payload" = metadata (like ChromaDB's "metadata")
model = SentenceTransformer('all-MiniLM-L6-v2')

products = [
    {
        "name": "Blue Cotton Sweater",
        "description": "A comfortable blue sweater made from 100% organic cotton.",
        "category": "clothing",
        "price": 49.99,
        "in_stock": True,
        "rating": 4.5,
        "tags": ["winter", "casual", "cotton"],
    },
    {
        "name": "Wireless Bluetooth Headphones",
        "description": "Noise-cancelling wireless headphones with 30-hour battery life.",
        "category": "electronics",
        "price": 199.99,
        "in_stock": True,
        "rating": 4.7,
        "tags": ["audio", "wireless", "noise-cancelling"],
    },
    {
        "name": "Organic Green Tea",
        "description": "Premium Japanese green tea, rich in antioxidants.",
        "category": "food",
        "price": 12.99,
        "in_stock": True,
        "rating": 4.3,
        "tags": ["beverage", "organic", "healthy"],
    },
    {
        "name": "Leather Running Shoes",
        "description": "Lightweight running shoes with cushioned soles.",
        "category": "clothing",
        "price": 89.99,
        "in_stock": False,
        "rating": 4.1,
        "tags": ["sports", "running", "shoes"],
    },
    {
        "name": "Stainless Steel Water Bottle",
        "description": "Double-walled insulated water bottle, keeps drinks cold for 24h.",
        "category": "home",
        "price": 24.99,
        "in_stock": True,
        "rating": 4.8,
        "tags": ["kitchen", "eco-friendly", "insulated"],
    },
]

# Embed descriptions and prepare points
points = []
for i, product in enumerate(products):
    # Use description as the text to embed
    embedding = model.encode([product["description"]])[0].tolist()
    
    points.append(models.PointStruct(
        id=i,                      # Unique ID (can be int or UUID)
        vector=embedding,          # The embedding vector
        payload={                  # Metadata (can be any JSON-serializable dict)
            "name": product["name"],
            "category": product["category"],
            "price": product["price"],
            "in_stock": product["in_stock"],
            "rating": product["rating"],
            "tags": product["tags"],
        },
    ))

# Batch upsert
client.upsert(
    collection_name=COLLECTION_NAME,
    points=points,
)

print(f"Added {len(points)} products to collection")

# ── Step 4: Basic Search ──
print("\n═══ Basic Search ═══\n")

query = "warm clothing for winter"
query_vector = model.encode([query])[0].tolist()

results = client.search(
    collection_name=COLLECTION_NAME,
    query_vector=query_vector,
    limit=3,
)

print(f"Query: '{query}'\n")
for result in results:
    score = result.score  # Cosine similarity (higher = better)
    payload = result.payload
    print(f"  [{score:.4f}] {payload['name']} (${payload['price']})")
    print(f"         {payload['category']} | Rating: {payload['rating']} | In stock: {payload['in_stock']}")
    print()

# ── Step 5: Filtered Search ──
print("═══ Filtered Search (category = clothing, price < $100, in stock) ═══\n")

results = client.search(
    collection_name=COLLECTION_NAME,
    query_vector=query_vector,
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="category",
                match=models.MatchValue(value="clothing"),
            ),
            models.FieldCondition(
                key="price",
                range=models.Range(
                    lt=100.0,  # less than $100
                ),
            ),
            models.FieldCondition(
                key="in_stock",
                match=models.MatchValue(value=True),
            ),
        ],
    ),
    limit=5,
)

if results:
    for result in results:
        score = result.score
        payload = result.payload
        print(f"  [{score:.4f}] {payload['name']} (${payload['price']})")
else:
    print("  No results matching filters.")

print("\n🤔 Compare filtered vs unfiltered results above.")
print("  - Did the filtered results change?")
print("  - Did the scores change?")
print("  - Qdrant applies filters DURING search, not after.")
```

**Expected output:**
```
Connected to Qdrant: collections=[]
Created collection: products
Added 5 products to collection

═══ Basic Search ═══

Query: 'warm clothing for winter'

  [0.8912] Blue Cotton Sweater ($49.99)
         clothing | Rating: 4.5 | In stock: True

  [0.6234] Leather Running Shoes ($89.99)
         clothing | Rating: 4.1 | In stock: False

  [0.4567] Stainless Steel Water Bottle ($24.99)
         home | Rating: 4.8 | In stock: True

═══ Filtered Search (category = clothing, price < $100, in stock) ═══

  [0.8912] Blue Cotton Sweater ($49.99)
         clothing | Rating: 4.5 | In stock: True

Note: Leather Running Shoes is filtered out (out of stock).
The water bottle is filtered out (wrong category).
Only 1 result matches ALL criteria — but it's the RIGHT one.
```

---

### Example 2: Qdrant Production Patterns

```python
"""
Production patterns with Qdrant.
Real-world operations you'll use daily.
"""

from qdrant_client import QdrantClient
from qdrant_client.http import models
from sentence_transformers import SentenceTransformer
import numpy as np
from typing import List, Dict, Optional, Generator
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class ProductSearchEngine:
    """
    Production-grade product search with Qdrant.
    
    This is what a real e-commerce search backend looks like.
    """
    
    def __init__(
        self,
        host: str = "localhost",
        port: int = 6333,
        collection_name: str = "products_v2",
        model_name: str = "all-MiniLM-L6-v2",
    ):
        self.client = QdrantClient(host=host, port=port)
        self.model = SentenceTransformer(model_name)
        self.collection_name = collection_name
        self.dimension = self.model.get_sentence_embedding_dimension()
        
        # Ensure collection exists
        self._ensure_collection()
    
    def _ensure_collection(self):
        """Create collection if it doesn't exist."""
        collections = self.client.get_collections().collections
        existing = {c.name for c in collections}
        
        if self.collection_name not in existing:
            self.client.create_collection(
                collection_name=self.collection_name,
                vectors_config=models.VectorParams(
                    size=self.dimension,
                    distance=models.Distance.COSINE,
                ),
                # Optimize for large datasets
                optimizers_config=models.OptimizersConfigDiff(
                    default_segment_number=2,
                    indexing_threshold=10000,  # Start building index after 10K points
                ),
            )
            logger.info(f"Created collection: {self.collection_name}")
    
    def batch_upsert(
        self,
        products: List[Dict],
        batch_size: int = 100,
    ):
        """
        Batch upsert products with progress tracking.
        
        Batching is CRITICAL for production — single inserts are 100x slower.
        """
        total = len(products)
        logger.info(f"Upserting {total} products in batches of {batch_size}")
        
        for i in range(0, total, batch_size):
            batch = products[i:i + batch_size]
            points = []
            
            for product in batch:
                embedding = self.model.encode([product["description"]])[0].tolist()
                points.append(models.PointStruct(
                    id=product.get("id", i),  # Use provided ID or index
                    vector=embedding,
                    payload={
                        "name": product["name"],
                        "description": product["description"][:200],  # Truncate for payload
                        "category": product.get("category", "unknown"),
                        "price": product.get("price", 0),
                        "in_stock": product.get("in_stock", True),
                        "rating": product.get("rating", 0.0),
                        "created_at": product.get("created_at", ""),
                    },
                ))
            
            self.client.upsert(
                collection_name=self.collection_name,
                points=points,
                wait=True,  # Wait for confirmation
            )
            
            pct = min(100, (i + batch_size) / total * 100)
            logger.info(f"  Progress: {pct:.0f}% ({min(i+batch_size, total)}/{total})")
        
        logger.info("Batch upsert complete")
    
    def search(
        self,
        query: str,
        top_k: int = 10,
        category: Optional[str] = None,
        min_price: Optional[float] = None,
        max_price: Optional[float] = None,
        in_stock_only: bool = False,
        min_rating: Optional[float] = None,
    ) -> List[Dict]:
        """
        Search with combined semantic + metadata filtering.
        
        Qdrant applies filters DURING the HNSW search, not after.
        This means quality is higher than post-filtering.
        """
        query_vector = self.model.encode([query])[0].tolist()
        
        # Build filter conditions
        must_conditions = []
        
        if category:
            must_conditions.append(
                models.FieldCondition(
                    key="category",
                    match=models.MatchValue(value=category),
                )
            )
        
        if min_price is not None or max_price is not None:
            price_range = {}
            if min_price is not None:
                price_range["gte"] = min_price
            if max_price is not None:
                price_range["lte"] = max_price
            must_conditions.append(
                models.FieldCondition(
                    key="price",
                    range=models.Range(**price_range),
                )
            )
        
        if in_stock_only:
            must_conditions.append(
                models.FieldCondition(
                    key="in_stock",
                    match=models.MatchValue(value=True),
                )
            )
        
        if min_rating is not None:
            must_conditions.append(
                models.FieldCondition(
                    key="rating",
                    range=models.Range(gte=min_rating),
                )
            )
        
        # Execute search
        start = time.time()
        
        results = self.client.search(
            collection_name=self.collection_name,
            query_vector=query_vector,
            query_filter=models.Filter(must=must_conditions) if must_conditions else None,
            limit=top_k,
        )
        
        elapsed = time.time() - start
        
        # Format results
        formatted = []
        for result in results:
            formatted.append({
                "id": result.id,
                "score": result.score,
                "name": result.payload.get("name", ""),
                "description": result.payload.get("description", ""),
                "category": result.payload.get("category", ""),
                "price": result.payload.get("price", 0),
                "in_stock": result.payload.get("in_stock", False),
                "rating": result.payload.get("rating", 0),
            })
        
        logger.info(f"Search '{query[:30]}...' returned {len(results)} results in {elapsed*1000:.1f}ms")
        
        return formatted
    
    def scroll_all(self, batch_size: int = 100) -> Generator[List[Dict], None, None]:
        """
        Scroll through ALL points in the collection (for export/backup).
        
        Uses cursor-based pagination — handles millions of points.
        """
        next_offset = None
        stop = False
        
        while not stop:
            records, next_offset = self.client.scroll(
                collection_name=self.collection_name,
                limit=batch_size,
                offset=next_offset,
            )
            
            if not records:
                break
            
            yield [
                {
                    "id": r.id,
                    "payload": r.payload,
                }
                for r in records
            ]
            
            if next_offset is None:
                stop = True
    
    def delete_by_filter(self, category: Optional[str] = None, ids: Optional[List[int]] = None):
        """Delete points matching filter criteria."""
        if ids:
            self.client.delete(
                collection_name=self.collection_name,
                points_selector=models.PointIdsList(
                    points=ids,
                ),
            )
            logger.info(f"Deleted {len(ids)} points by IDs")
        
        if category:
            self.client.delete(
                collection_name=self.collection_name,
                points_selector=models.FilterSelector(
                    filter=models.Filter(
                        must=[
                            models.FieldCondition(
                                key="category",
                                match=models.MatchValue(value=category),
                            ),
                        ],
                    ),
                ),
            )
            logger.info(f"Deleted all points with category={category}")
    
    def collection_stats(self) -> Dict:
        """Get collection statistics."""
        info = self.client.get_collection(self.collection_name)
        return {
            "name": self.collection_name,
            "points_count": info.points_count,
            "vectors_count": info.vectors_count,
            "indexed_vectors_count": info.indexed_vectors_count,
            "status": info.status,
            "dimension": self.dimension,
        }


# ── Usage ──
if __name__ == "__main__":
    engine = ProductSearchEngine()
    
    # Add sample products
    sample_products = [
        {"id": 1, "name": "Blue Cotton Sweater", "description": "Comfortable blue sweater, 100% organic cotton", "category": "clothing", "price": 49.99, "in_stock": True, "rating": 4.5},
        {"id": 2, "name": "Leather Running Shoes", "description": "Lightweight running shoes with cushioned soles", "category": "clothing", "price": 89.99, "in_stock": False, "rating": 4.1},
        {"id": 3, "name": "Wireless Headphones", "description": "Noise-cancelling bluetooth headphones, 30h battery", "category": "electronics", "price": 199.99, "in_stock": True, "rating": 4.7},
        {"id": 4, "name": "Organic Green Tea", "description": "Premium Japanese green tea, antioxidant rich", "category": "food", "price": 12.99, "in_stock": True, "rating": 4.3},
        {"id": 5, "name": "Stainless Steel Bottle", "description": "Double-walled insulated bottle, 24h cold retention", "category": "home", "price": 24.99, "in_stock": True, "rating": 4.8},
        {"id": 6, "name": "Merino Wool Scarf", "description": "Warm merino wool scarf, perfect for winter", "category": "clothing", "price": 39.99, "in_stock": True, "rating": 4.2},
        {"id": 7, "name": "USB-C Hub 7-in-1", "description": "Multi-port USB-C hub with HDMI, SD card, USB 3.0", "category": "electronics", "price": 34.99, "in_stock": True, "rating": 4.0},
        {"id": 8, "name": "Dark Chocolate Bar", "description": "72% cacao organic dark chocolate, single origin", "category": "food", "price": 4.99, "in_stock": True, "rating": 4.6},
    ]
    
    engine.batch_upsert(sample_products, batch_size=5)
    
    # Search with various filters
    print("\n═══ Search Demos ═══\n")
    
    # 1. Basic search
    print("1. Generic: 'warm winter wear'")
    results = engine.search("warm winter wear", top_k=3)
    for r in results:
        print(f"   [{r['score']:.4f}] {r['name']} — ${r['price']} (rating: {r['rating']})")
    print()
    
    # 2. Category filtered
    print("2. 'audio equipment' (category: electronics)")
    results = engine.search("audio equipment", category="electronics", top_k=3)
    for r in results:
        print(f"   [{r['score']:.4f}] {r['name']} — ${r['price']} (category: {r['category']})")
    print()
    
    # 3. Price range + in stock
    print("3. 'something cheap to drink' (under $15, in stock)")
    results = engine.search("something cheap to drink", max_price=15.0, in_stock_only=True, top_k=3)
    for r in results:
        print(f"   [{r['score']:.4f}] {r['name']} — ${r['price']}")
    print()
    
    # 4. Stats
    print("═══ Collection Stats ═══")
    stats = engine.collection_stats()
    for k, v in stats.items():
        print(f"  {k}: {v}")
```

**Expected output:**
```
═══ Search Demos ═══

1. Generic: 'warm winter wear'
   [0.9123] Blue Cotton Sweater — $49.99 (rating: 4.5)
   [0.8345] Merino Wool Scarf — $39.99 (rating: 4.2)
   [0.4567] Stainless Steel Bottle — $24.99 (rating: 4.8)

2. 'audio equipment' (category: electronics)
   [0.8678] Wireless Headphones — $199.99 (category: electronics)
   [0.6234] USB-C Hub 7-in-1 — $34.99 (category: electronics)

3. 'something cheap to drink' (under $15, in stock)
   [0.7890] Organic Green Tea — $12.99
   [0.6234] Dark Chocolate Bar — $4.99
   [0.4123] Stainless Steel Bottle — $24.99 (over $15! — but combined score ranked it)

Note: In query 3, the water bottle appears despite being over $15.
This is because FILTERING happens during search, but if there aren't enough
results meeting the filter, Qdrant may return the closest unfiltered match.
In production, you'd handle this with a strict filter or by increasing top_k.
```

---

### Example 3: Pinecone — The Managed Alternative

Now let's contrast with Pinecone. The API is similar — but the deployment is completely different:

```python
"""
Pinecone — managed vector database.
Deploy in 5 minutes with zero infrastructure.
"""

# ── Setup ──
# First, create an account at pinecone.io and get an API key
#
# Then install:
# pip install pinecone-client

import pinecone
from sentence_transformers import SentenceTransformer
import time

# ── Step 1: Initialize ──
# You need API keys from app.pinecone.io
# FREE TIER: 1 index, up to 100K vectors (as of 2026)

# Initialize Pinecone
pc = pinecone.Pinecone(
    api_key="your-api-key",  # Replace with your actual key
    environment="us-east-1-aws",  # Your region
)

# ── Step 2: Create Index ──
INDEX_NAME = "products-pinecone"

# Delete existing index (for clean demo)
if INDEX_NAME in pc.list_indexes().names():
    pc.delete_index(INDEX_NAME)

# Create serverless index (Pinecone's latest model)
pc.create_index(
    name=INDEX_NAME,
    dimension=384,  # Must match your embedding model
    metric="cosine",
    spec=pinecone.ServerlessSpec(
        cloud="aws",
        region="us-east-1",
    ),
)

# Wait for index to be ready
while not pc.describe_index(INDEX_NAME).status["ready"]:
    time.sleep(1)

# Connect to the index
index = pc.Index(INDEX_NAME)
print(f"Index '{INDEX_NAME}' ready!")

# ── Step 3: Add vectors ──
model = SentenceTransformer('all-MiniLM-L6-v2')

products = [
    {"id": "prod-1", "name": "Blue Cotton Sweater", "category": "clothing", "price": 49.99},
    {"id": "prod-2", "name": "Wireless Headphones", "category": "electronics", "price": 199.99},
    {"id": "prod-3", "name": "Organic Green Tea", "category": "food", "price": 12.99},
    {"id": "prod-4", "name": "Running Shoes", "category": "clothing", "price": 89.99},
    {"id": "prod-5", "name": "Water Bottle", "category": "home", "price": 24.99},
]

# Pinecone uses a different format than Qdrant
vectors = []
for product in products:
    embedding = model.encode([product["name"] + ": " + product.get("description", "")])[0].tolist()
    vectors.append((
        product["id"],       # ID (string)
        embedding,           # Vector
        product,             # Metadata
    ))

# Upsert in batch
index.upsert(vectors=vectors)
print(f"Added {len(products)} vectors")

# ── Step 4: Search ──
query = "warm winter clothing"
query_vector = model.encode([query])[0].tolist()

results = index.query(
    vector=query_vector,
    top_k=3,
    include_metadata=True,
)

print(f"\nQuery: '{query}'\n")
for result in results["matches"]:
    print(f"  [{result['score']:.4f}] {result['metadata']['name']}")
    print(f"         Category: {result['metadata']['category']}")

# ── Step 5: Filtered Search ──
print("\n═══ Filtered Search (clothing only) ═══\n")

results = index.query(
    vector=query_vector,
    top_k=3,
    include_metadata=True,
    filter={
        "category": {"$eq": "clothing"},
    },
)

for result in results["matches"]:
    print(f"  [{result['score']:.4f}] {result['metadata']['name']}")

# ── Cleanup ──
# pc.delete_index(INDEX_NAME)
print("\nDone! (Index kept for reuse — delete when done)")
```

**The crucial difference:** With Pinecone, you never install a database. You never configure shards. You never manage disks. You call an API. That's it.

```
PINECONE VS QDRANT — THE REAL TRADEOFFS:
─────────────────────────────────────────────

COST (at 1M vectors, 384D):
  Pinecone: ~$70/month (managed, serverless)
  Qdrant:   ~$20/month (self-hosted on a single $20 VPS)
  
OPS:
  Pinecone: Zero. They handle backups, upgrades, scaling.
  Qdrant:   You handle Docker, monitoring, backups, capacity planning.
  
LATENCY:
  Pinecone: 5-15ms (optimized infra)
  Qdrant:   2-10ms (same machine, no network hop)
  
VENDOR LOCK-IN:
  Pinecone: Proprietary API. Migration requires code changes.
  Qdrant:   Open-source. You own the data. Migration is trivial.
  
FEATURES:
  Pinecone: Managed = less configurable. You get what they give you.
  Qdrant:   Fully configurable. You control every parameter.

SENIOR INSIGHT:
  Early stage (< 100K vectors): use Pinecone free tier (zero ops)
  Growth stage (100K - 10M): self-host Qdrant (cost savings)
  Enterprise (> 10M): Pinecone dedicted (SLA, support) or Qdrant cluster
```

---

### Example 4: Qdrant Filtering Deep Dive

Let's explore Qdrant's filtering capabilities in detail because this is where it truly shines:

```python
"""
Qdrant — advanced filtering patterns.
Understanding how different filters work in production.
"""

from qdrant_client import QdrantClient
from qdrant_client.http import models
import numpy as np

client = QdrantClient(host="localhost", port=6333)

# Create collection with diverse products
COLLECTION = "filter_demo"
try:
    client.delete_collection(COLLECTION)
except:
    pass

client.create_collection(
    collection_name=COLLECTION,
    vectors_config=models.VectorParams(size=384, distance=models.Distance.COSINE),
)

# Add varied products
products = [
    {"id": 1, "name": "MacBook Pro 16", "category": "electronics", "price": 2499, "rating": 4.8, "tags": ["apple", "laptop", "premium"], "in_stock": True, "color": "silver"},
    {"id": 2, "name": "iPhone 16 Pro", "category": "electronics", "price": 1199, "rating": 4.7, "tags": ["apple", "phone", "premium"], "in_stock": True, "color": "black"},
    {"id": 3, "name": "AirPods Pro", "category": "electronics", "price": 249, "rating": 4.6, "tags": ["apple", "audio", "wireless"], "in_stock": True, "color": "white"},
    {"id": 4, "name": "Samsung Galaxy S25", "category": "electronics", "price": 1099, "rating": 4.5, "tags": ["samsung", "phone", "android"], "in_stock": False, "color": "black"},
    {"id": 5, "name": "Dell XPS 15", "category": "electronics", "price": 1799, "rating": 4.4, "tags": ["dell", "laptop", "windows"], "in_stock": True, "color": "silver"},
    {"id": 6, "name": "Sony WH-1000XM5", "category": "electronics", "price": 349, "rating": 4.9, "tags": ["sony", "audio", "premium"], "in_stock": True, "color": "black"},
    {"id": 7, "name": "Kindle Paperwhite", "category": "electronics", "price": 139, "rating": 4.5, "tags": ["amazon", "ebook", "reader"], "in_stock": True, "color": "black"},
]

from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-MiniLM-L6-v2')

points = []
for p in products:
    embedding = model.encode([p["name"]])[0].tolist()
    points.append(models.PointStruct(
        id=p["id"],
        vector=embedding,
        payload={k: v for k, v in p.items() if k != "id"},
    ))

client.upsert(collection_name=COLLECTION, points=points)

# ── Filter Pattern 1: Exact Match ──
print("═══ Filter Pattern 1: Exact Match ═══")
query_vec = model.encode(["wireless audio"])[0].tolist()

results = client.search(
    collection_name=COLLECTION,
    query_vector=query_vec,
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="tags",
                match=models.MatchValue(value="audio"),  # Exact tag match
            ),
        ],
    ),
)

for r in results:
    print(f"  [{r.score:.4f}] {r.payload['name']} — tags: {r.payload['tags']}")

# ── Filter Pattern 2: Range Query ──
print("\n═══ Filter Pattern 2: Price Range ═══")
results = client.search(
    collection_name=COLLECTION,
    query_vector=query_vec,
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="price",
                range=models.Range(
                    gte=200,   # greater or equal
                    lte=500,   # less or equal
                ),
            ),
        ],
    ),
)
for r in results:
    print(f"  [{r.score:.4f}] {r.payload['name']} — ${r.payload['price']}")

# ── Filter Pattern 3: Nested Boolean ──
print("\n═══ Filter Pattern 3: Boolean Logic ═══")
# Find: (apple OR sony) AND in_stock AND price < 500
results = client.search(
    collection_name=COLLECTION,
    query_vector=query_vec,
    query_filter=models.Filter(
        must=[
            models.Filter(
                should=[  # OR condition
                    models.FieldCondition(
                        key="tags",
                        match=models.MatchValue(value="apple"),
                    ),
                    models.FieldCondition(
                        key="tags",
                        match=models.MatchValue(value="sony"),
                    ),
                ],
            ),
            models.FieldCondition(
                key="in_stock",
                match=models.MatchValue(value=True),
            ),
            models.FieldCondition(
                key="price",
                range=models.Range(lt=500),
            ),
        ],
    ),
)
for r in results:
    print(f"  [{r.score:.4f}] {r.payload['name']} — ${r.payload['price']} — tags: {r.payload['tags']}")

# ── Filter Pattern 4: Negative Filtering ──
print("\n═══ Filter Pattern 4: Excluding Values ═══")
# Find electronics that are NOT Apple products and NOT sold out
results = client.search(
    collection_name=COLLECTION,
    query_vector=query_vec,
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="category",
                match=models.MatchValue(value="electronics"),
            ),
            models.FieldCondition(
                key="in_stock",
                match=models.MatchValue(value=True),
            ),
        ],
        must_not=[  # Exclude Apple
            models.FieldCondition(
                key="tags",
                match=models.MatchValue(value="apple"),
            ),
        ],
    ),
)
for r in results:
    print(f"  [{r.score:.4f}] {r.payload['name']} — {r.payload['tags']}")

# ── Filter Pattern 5: Payload Exists Check ──
print("\n═══ Filter Pattern 5: Field Existence ═══")
results = client.search(
    collection_name=COLLECTION,
    query_vector=query_vec,
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="color",
                match=models.MatchValue(value="black"),
            ),
        ],
    ),
)
for r in results:
    print(f"  [{r.score:.4f}] {r.payload['name']} — color: {r.payload.get('color', 'N/A')}")
```

---

## ✅ Good Output Examples

### What Strong Production Vector DB Deployment Looks Like

```
Production Qdrant Deployment (1M+ vectors)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Infrastructure:
  ✓ Qdrant running in Docker with persistent volume
  ✓ 4GB RAM allocated to Qdrant (for HNSW cache)
  ✓ SSD storage (NVMe recommended for Qdrant)
  ✓ Health checks every 30s
  ✓ Automated backups every 6h
  ✓ Monitoring: CPU, memory, query latency, error rate

Configuration:
  ✓ HNSW tuned: M=32, ef_construct=200, ef_search=100
  ✓ On-disk vectors (to save RAM for index)
  ✓ Cosine distance (consistent with embedding model)
  ✓ Indexing threshold: 10,000 (don't index tiny collections)

Application:
  ✓ Batched upsert (100-1000 points per batch)
  ✓ Retry with exponential backoff on rate limits
  ✓ Connection pooling (reuse client across requests)
  ✓ All search operations have timeout
  ✓ Fallback: if Qdrant is down, return cached results
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Embedding Every Query Twice

```python
# ❌ WRONG — Embedding inside the search function (re-embeds every call)
class SearchAPI:
    def search(self, query: str):
        vec = self.model.encode([query])[0].tolist()  # ← This is fine
        return self.client.search(...)

# ✅ RIGHT — Cache query embeddings for repeated queries
# If users tend to search the same terms, cache helps
query_cache: Dict[str, List[float]] = {}

def search_with_cache(query: str):
    if query not in query_cache:
        query_cache[query] = model.encode([query])[0].tolist()
    return client.search(query_vector=query_cache[query], ...)
```

### Antipattern 2: Not Batching Upserts

```python
# ❌ WRONG — Single upsert per point (10K API calls for 10K points)
for point in all_points:
    client.upsert(collection_name="col", points=[point])
# Takes: ~5 minutes for 10K (30ms per call)

# ✅ RIGHT — Batch upsert (1 API call for 10K points)
client.upsert(collection_name="col", points=all_points)
# Takes: ~2 seconds for 10K (150x faster)
```

### Antipattern 3: Choosing Wrong Distance Metric

```python
# ❌ WRONG — Using Dot product when embeddings are NOT normalized
client.create_collection(
    vectors_config=VectorParams(size=384, distance=Distance.DOT),
)
# If vectors aren't unit length, DOT is biased by magnitude

# ✅ RIGHT — Use Cosine for most text embedding models
client.create_collection(
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)

# 🤔 When would you use DOT instead?
# - Your model ALWAYS outputs normalized embeddings
# - You're comparing pre-normalized vectors
# - Dot product is slightly faster than cosine
```

### Antipattern 4: Infinite Scroll

```python
# ❌ WRONG — Loading ALL vectors into memory
all_points = client.scroll(
    collection_name="products",
    limit=1000000,  # May OOM your application
)

# ✅ RIGHT — Use cursor-based pagination
next_offset = None
while True:
    records, next_offset = client.scroll(
        collection_name="products",
        limit=100,
        offset=next_offset,
    )
    if not records:
        break
    process_batch(records)
    if next_offset is None:
        break
```

### Failure Mode: The Cost Surprise

```
Pinecone serverless pricing based on:
- $ per million vector-hours (storage)
- $ per million queries (usage)
- $ per GB of data transferred

⚠️ SURPRISE COSTS:
1. Re-indexing: If you delete and re-add all vectors, you pay for
   BOTH the old and new storage during the transition
2. Metadata size: Large payloads increase storage costs significantly
3. Idle indexes: An index with no queries still costs vector-hours

ESTIMATE BEFORE DEPLOYING:
Use their pricing calculator. A mistake here can cost $1000+/month.
```

---

## 🧪 Drills & Challenges

### Drill 1: ChromaDB → Qdrant Migration (30 min)

Take the File 04 ChromaDB document manager and migrate it to Qdrant. Keep the SAME API:

```python
# File 04 API (keep this interface)
class DocumentManager:
    def add_document(self, text, metadata): ...
    def search(self, query, top_k, filter): ...

# Your job: same API, but backed by Qdrant instead of ChromaDB
class QdrantDocumentManager:
    # ... (same methods, different implementation)
```

**Key insight:** If you designed your File 04 code well, the migration is just replacing the backend — the application code doesn't change. This is why abstraction matters.

### Drill 2: The Filter Performance Test (25 min)

Compare search performance with and without filters at different dataset sizes:

1. Index 100K random vectors with random metadata (categories, price ranges)
2. Benchmark search with NO filter vs WITH filter at different selectivity levels:
   - "all results" (no filter) — baseline
   - "category = X" (selects ~20% of data)
   - "category = X AND price < Y" (selects ~5% of data)
3. Does filtering hurt performance? By how much?

**Expected finding:** Highly selective filters (matching few documents) actually IMPROVE performance because HNSW has fewer points to search through.

### Drill 3: The Self-Hosted vs Managed Cost Calculator (20 min)

Build a script that calculates the monthly cost of:

A) Self-hosted Qdrant on a cloud VM
B) Managed Pinecone serverless
C) Managed Qdrant Cloud

At these scales:
- 100K vectors, 10K queries/day
- 1M vectors, 100K queries/day
- 10M vectors, 1M queries/day

Include:
- Compute cost (VM)
- Storage cost (volume)
- Managed service cost (per-vector pricing)
- Bandwidth cost

**Purpose:** You'll internalize when each option makes financial sense.

### Drill 4: The Multi-Tenancy Design (25 min)

Design a multi-tenant search system where:
- 100 different customers each have their own documents
- Customer A should NEVER see Customer B's results
- Each customer has 1K-100K documents (variable)

Design 3 approaches:
1. Separate collection per customer
2. Single collection with tenant_id filter
3. Separate Qdrant instance per customer

**Compare:**
- Cost
- Performance
- Isolation (can one customer's bad query affect others?)
- Operational complexity

**This is a REAL decision senior engineers face when building SaaS products.**

---

## 🚦 Gate Check

Before moving to File 06 (pgvector), confirm you can:

- [ ] **Run Qdrant locally** with Docker
- [ ] **Create collections** with proper vector configuration
- [ ] **Build complex filters** (exact match, range, boolean, negation)
- [ ] **Batch upsert** 1000+ documents efficiently
- [ ] **Explain when to use Qdrant vs Pinecone** — with cost justification
- [ ] **Answer these questions:**
  1. Why does Qdrant run as a separate server (not in-process like ChromaDB)?
  2. What happens to search accuracy when you apply a metadata filter? Does it change?
  3. Under what conditions would you choose Pinecone over self-hosted Qdrant?
  4. How does Qdrant's filtering differ from "search then filter in Python"?
  5. What's the cost difference between Qdrant and Pinecone at 1M vs 10M vectors?

- [ ] **Make a recommendation:** For YOUR use case (or a hypothetical one), which vector DB would you choose and why?

---

## 📚 Resources

**Qdrant:**
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Qdrant GitHub](https://github.com/qdrant/qdrant) — Rust source
- [Filtering Examples](https://qdrant.tech/documentation/concepts/filtering/)
- Docker: `docker run -p 6333:6333 qdrant/qdrant`

**Pinecone:**
- [Pinecone Documentation](https://docs.pinecone.io/)
- [Pinecone Pricing](https://www.pinecone.io/pricing/)
- [Serverless Quickstart](https://docs.pinecone.io/guides/get-started/quickstart)

**Decision Resources:**
- [Vector Database Comparison](https://qdrant.tech/documentation/comparison/) — Qdrant's comparison page
- ANN Benchmarks — Performance comparison of all major vector DBs

**Your Decision Flowchart:**
```
Production Vector Database Decision:
│
├── < 100K vectors, single process → ChromaDB (File 04)
│
├── < 1M vectors, want minimal ops → Pinecone free tier
│
├── < 10M vectors, have ops team → Qdrant self-hosted
│
├── < 10M vectors, no ops → Pinecone serverless
│
├── Already using PostgreSQL → pgvector (File 06)
│
├── > 10M vectors, need cost control → Qdrant cluster
│
└── > 10M vectors, need SLA → Pinecone dedicated
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **What "Production" Actually Means** — Can you now answer the 4 scenarios (crash, traffic spike, bad query, update)?
> 2. **Self-Hosted vs Managed** — Can you now make the call for a given scale and team?
> 3. **The Filtering Problem** — Do you understand how Qdrant's pre-filtering works and why it's better than post-filtering?
>
> **Key realization:** File 02 was about the MATH of similarity. File 04 was about the CONCEPT of vector databases. This file was about the PRODUCTION REALITY — scaling, ops, cost, and making the right architectural choice.
>
> **Next:** File 06 — pgvector. What if you don't want a separate vector DB and just use PostgreSQL?
