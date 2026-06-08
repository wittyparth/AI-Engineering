# pgvector — Vector Search in PostgreSQL

## 🎯 Purpose & Goals

> **🛑 STOP. Picture this:**
>
> You work at a company that already uses PostgreSQL for everything — users, products, orders, inventory. Your CTO says: "We need semantic search on our product catalog."
>
> Your options:
> - **Option A:** Run PostgreSQL + Qdrant (two databases to manage)
> - **Option B:** Use pgvector (extension on your existing PostgreSQL)
>
> ### 🤔 Discovery Question 1: One DB vs Two DBs
>
> **Think about the operational implications of EACH choice:**
>
> Option A (PostgreSQL + Qdrant):
> - Two databases to back up
> - Two databases to monitor
> - Data needs to sync between them
> - If a product is added to Postgres, how does Qdrant know?
> - Two different query languages (SQL + REST/gRPC)
> - Two different failure modes
>
> Option B (PostgreSQL with pgvector):
> - One database for everything
> - Product data AND vectors are in the same place
> - Can join vector search with SQL queries
> - No sync needed — it's the same database
> - One query language (SQL)
>
> **Before reading on, answer:**
> 1. When would you ABSOLUTELY choose Option A despite the extra complexity?
> 2. When would Option B be clearly better?
> 3. What's the ONE thing that could make Option B fail that Option A handles fine?

---

### 🤔 Discovery Question 2: The JOIN Problem

Your products table has columns: `id, name, description, category_id, price, in_stock, created_at`.

You want to: "Find products similar to 'blue cotton sweater' that are in stock, under $100, in the 'clothing' category, and sort by rating."

**With ChromaDB/Qdrant:** You'd store the vector in the vector DB, and the metadata in PostgreSQL. The query would be:
1. Search Qdrant with filter
2. If you need additional data (e.g., "show me reviews count"), you'd need to JOIN with PostgreSQL
3. Two round-trips: one to Qdrant, one to Postgres

**With pgvector:**
```sql
SELECT p.*, pr.avg_rating
FROM products p
LEFT JOIN product_reviews pr ON p.id = pr.product_id
WHERE p.category_id = 'clothing'
  AND p.in_stock = true
  AND p.price < 100
ORDER BY p.embedding <=> '[0.1, 0.2, ...]'  -- cosine distance
LIMIT 10;
```

**Question:** What's the operational difference between these two approaches? When does the JOIN capability of pgvector save you real engineering time?

---

### 🤔 Discovery Question 3: PostgreSQL's Vector Indexes

PostgreSQL supports TWO indexing methods for vector search:

**IVFFlat (Inverted File with Flat Compression):**
- Build time: Fast
- Search quality: Good (but depends on training)
- Build parameters: number of "lists" (clusters)
- Accuracy vs speed tradeoff: controlled by `probes` parameter

**HNSW (Hierarchical Navigable Small World):**
- Build time: Slower
- Search quality: Excellent
- Build parameters: `m` (connections per node)
- Memory: Higher (keeps more in memory)

**Before reading the details:**
1. Which sounds better for a dataset that UPDATES frequently (new products added every hour)?
2. Which sounds better for a STATIC dataset where search speed is critical?
3. If you had to guess: which one uses MORE memory? Which is FASTER to build?

---

> **By the end of this file, you will:**
> - Install and configure pgvector in PostgreSQL
> - Know when pgvector beats dedicated vector DBs AND when it falls short
> - Understand the two indexing methods (IVFFlat vs HNSW) and choose between them
> - Build hybrid queries that combine SQL filters with vector search
> - Know the exact limits: where pgvector breaks and you need Qdrant

**⏱ Time Budget:** 2 hours (30 min concept, 1 hour code + SQL, 30 min drills)

---

## 📖 pgvector — The PostgreSQL Way

### What pgvector Actually Is

pgvector is an **extension** to PostgreSQL. It's not a separate database. It adds:
- A new data type: `vector(n)` — stores an n-dimensional vector
- Distance operators: `<->` (L2), `<#>` (negative inner product), `<=>` (cosine distance)
- Index types: IVFFlat and HNSW for approximate nearest neighbor search

```sql
-- Without pgvector, you'd store vectors as JSON:
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    embedding JSONB  -- [0.1, 0.2, 0.3, ...]
);
-- This works, but you can't INDEX it or SEARCH it efficiently

-- With pgvector:
CREATE EXTENSION vector;
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    embedding vector(384)  -- Native type, indexable, searchable
);
```

### The pgvector vs Dedicated Vector DB Tradeoff

```
CRITERION               │ pgvector  │ Qdrant    │ When You Care
────────────────────────┼───────────┼───────────┼─────────────────────
Operational complexity  │ LOW       │ HIGH      │ Small team, no ops
Query expressiveness    │ HIGH      │ MEDIUM    │ Complex SQL queries
Vector search speed     │ MEDIUM    │ HIGH      │ > 1M vectors
Joins with other data   │ NATIVE    │ MANUAL    │ Product + reviews
Full-text search        │ NATIVE    │ SEPARATE  │ Hybrid search
Index build speed       │ FAST      │ MEDIUM    │ Frequent updates
Vector dimension limit  │ 2000      │ 10000+    │ Large embeddings
Multi-tenancy           │ RLS       │ Built-in  │ SaaS applications
────────────────────────┴───────────┴───────────┴─────────────────────
```

**🤔 Key insight: pgvector's superpower is not speed — it's INTEGRATION.** If you're already using PostgreSQL, pgvector eliminates an entire database from your stack. That's fewer backups, fewer failure modes, less sync code.

---

### IVFFlat vs HNSW — Let's Discover the Difference

Instead of telling you which is better, let's think about what each does:

**IVFFlat works like this:**
1. During indexing: cluster all vectors into K groups (lists)
2. During search: find which clusters the query is close to, search only those clusters

```
IVFFlat visualization (2D):
┌────────────────────────────────────────────┐
│                                            │
│    ● ● ●   Cluster A    ● ● ●             │
│   ● ● ● ●              ● ● ● ●   Cluster B│
│    ● ● ●                ● ● ●             │
│                                            │
│         ○  ← query                     │
│                                            │
│  Cluster C  ● ● ●      ● ● ●  Cluster D   │
│             ● ● ●      ● ● ●              │
│              ● ●        ● ●               │
└────────────────────────────────────────────┘

Query checks distance to each cluster CENTROID
→ Finds it's close to clusters A and C
→ Only searches points in clusters A and C (not B and D)
→ 50% fewer comparisons!
```

**HNSW works like this (you learned this in File 04):**
- Builds a layered graph
- Navigation: coarse → fine
- More accurate, more memory, slower to build

**🤔 Which one should YOU choose? Let's derive the answer:**
- What if you add new data every minute? (IVFFlat needs periodic rebuild, HNSW is incremental)
- What if you want EXACT results? (Both are approximate, but HNSW is closer)
- What if your server has only 2GB RAM? (IVFFlat is lighter)
- What if search speed is critical? (HNSW is faster at query time)

---

## 💻 Code Examples — pgvector in Action

### Example 1: Setting Up pgvector

```bash
# ── Step 1: Install PostgreSQL (if not already) ──
# macOS:
brew install postgresql@16
brew services start postgresql@16

# Linux (Ubuntu/Debian):
# sudo apt install postgresql postgresql-contrib

# Windows: Download from https://www.postgresql.org/download/windows/

# ── Step 2: Install pgvector extension ──
# macOS:
brew install pgvector

# Linux:
# cd /tmp
# git clone --branch v0.7.4 https://github.com/pgvector/pgvector.git
# cd pgvector
# make
# sudo make install

# ── Step 3: Enable the extension in your database ──
# psql -U postgres
# CREATE DATABASE ai_course;
# \c ai_course
# CREATE EXTENSION vector;
# \dx  -- should show "vector" extension
```

Now let's use it:

```python
"""
pgvector — vector search in PostgreSQL.
Combining SQL queries with semantic search.
"""

import psycopg2
import psycopg2.extras
from sentence_transformers import SentenceTransformer
import numpy as np
from typing import List, Dict, Optional, Tuple
import time
import os

# ── Database connection ──
DB_CONFIG = {
    "dbname": "ai_course",
    "user": os.getenv("PGUSER", "postgres"),
    "password": os.getenv("PGPASSWORD", "postgres"),
    "host": os.getenv("PGHOST", "localhost"),
    "port": os.getenv("PGPORT", "5432"),
}


def get_connection():
    """Get a database connection."""
    return psycopg2.connect(**DB_CONFIG)


# ── Step 1: Set up tables ──
def setup_database():
    """Create tables and indexes for vector search."""
    conn = get_connection()
    cur = conn.cursor()
    
    # Enable pgvector extension
    cur.execute("CREATE EXTENSION IF NOT EXISTS vector;")
    
    # Create products table with embedding column
    cur.execute("""
        CREATE TABLE IF NOT EXISTS products (
            id SERIAL PRIMARY KEY,
            name TEXT NOT NULL,
            description TEXT,
            category TEXT,
            price DECIMAL(10, 2),
            in_stock BOOLEAN DEFAULT true,
            rating DECIMAL(3, 2) DEFAULT 0.0,
            tags TEXT[],
            created_at TIMESTAMP DEFAULT NOW(),
            embedding vector(384)  -- pgvector native type
        );
    """)
    
    conn.commit()
    cur.close()
    conn.close()
    print("Database setup complete.")


# ── Step 2: Insert data with embeddings ──
def insert_product(
    name: str,
    description: str,
    category: str,
    price: float,
    in_stock: bool,
    rating: float,
    tags: List[str],
    embedding: List[float],
) -> int:
    """Insert a product with its embedding."""
    conn = get_connection()
    cur = conn.cursor()
    
    cur.execute("""
        INSERT INTO products (name, description, category, price, in_stock, rating, tags, embedding)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s::vector)
        RETURNING id;
    """, (
        name, description, category, price, in_stock, rating, tags, embedding
    ))
    
    product_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return product_id


# ── Step 3: Build an IVFFlat index ──
def build_index():
    """
    Build an IVFFlat index for faster vector search.
    
    IVFFlat works by clustering vectors into 'lists' (groups).
    During search, only the closest lists are searched.
    
    🤔 Why IVFFlat and not HNSW?
    - IVFFlat is simpler and uses less memory
    - HNSW is more accurate but more complex
    - The "lists" parameter controls the number of clusters
    - Rule of thumb: lists = sqrt(n_docs) for n_docs < 1M
      For 1000 docs: lists = 32
      For 100K docs: lists = 316
    """
    conn = get_connection()
    cur = conn.cursor()
    
    # Count documents
    cur.execute("SELECT COUNT(*) FROM products;")
    count = cur.fetchone()[0]
    
    if count < 100:
        print(f"Only {count} documents — index not needed yet.")
        print("(IVFFlat needs enough data to form meaningful clusters)")
        cur.close()
        conn.close()
        return
    
    # Drop existing index if any
    cur.execute("DROP INDEX IF EXISTS products_embedding_idx;")
    
    # Create IVFFlat index
    # lists = sqrt(count) as rule of thumb
    lists = max(10, int(np.sqrt(count)))
    
    print(f"Building IVFFlat index with {lists} lists for {count} documents...")
    start = time.time()
    
    cur.execute(f"""
        CREATE INDEX products_embedding_idx
        ON products
        USING ivfflat (embedding vector_cosine_ops)
        WITH (lists = {lists});
    """)
    
    conn.commit()
    elapsed = time.time() - start
    print(f"Index built in {elapsed:.2f}s")
    
    cur.close()
    conn.close()


# ── Step 4: Search with hybrid queries ──
def search_products(
    query_text: str,
    model,
    top_k: int = 10,
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    in_stock_only: bool = False,
    min_rating: Optional[float] = None,
    probes: int = 10,
) -> List[Dict]:
    """
    Hybrid search: SQL filters + vector similarity.
    
    THIS IS pgvector's SUPERPOWER.
    One query. One database. Joins, filters, and vector search simultaneously.
    
    The `<=>` operator computes cosine distance.
    Lower distance = more similar.
    """
    # Embed query
    query_embedding = model.encode([query_text])[0].tolist()
    
    # Build SQL
    sql = """
        SELECT
            id, name, description, category, price,
            in_stock, rating, tags,
            1 - (embedding <=> %s::vector) AS similarity
        FROM products
        WHERE 1=1
    """
    params = [query_embedding]
    
    # Add filters
    if category:
        sql += " AND category = %s"
        params.append(category)
    if min_price is not None:
        sql += " AND price >= %s"
        params.append(min_price)
    if max_price is not None:
        sql += " AND price <= %s"
        params.append(max_price)
    if in_stock_only:
        sql += " AND in_stock = true"
    if min_rating is not None:
        sql += " AND rating >= %s"
        params.append(min_rating)
    
    # Add vector search ordering
    sql += """
        ORDER BY embedding <=> %s::vector
        LIMIT %s;
    """
    params.extend([query_embedding, top_k])
    
    # Set probes for better recall
    # probes = how many clusters to search during IVFFlat
    conn = get_connection()
    cur = conn.cursor()
    
    cur.execute(f"SET ivfflat.probes = {probes};")
    
    start = time.time()
    cur.execute(sql, params)
    results = cur.fetchall()
    elapsed = time.time() - start
    
    # Format results
    columns = ["id", "name", "description", "category", "price",
               "in_stock", "rating", "tags", "similarity"]
    formatted = [dict(zip(columns, row)) for row in results]
    
    cur.close()
    conn.close()
    
    return formatted, elapsed


# ── Demo ──
if __name__ == "__main__":
    print("═══ pgvector Demo ═══\n")
    
    # Setup
    setup_database()
    model = SentenceTransformer('all-MiniLM-L6-v2')
    
    # Add products
    products = [
        ("Blue Cotton Sweater", "A comfortable blue sweater made from organic cotton, perfect for casual winter wear.", "clothing", 49.99, True, 4.5, ["winter", "casual", "cotton"]),
        ("Wireless Bluetooth Headphones", "Noise-cancelling over-ear headphones with 30-hour battery life and premium sound quality.", "electronics", 199.99, True, 4.7, ["audio", "wireless", "premium"]),
        ("Organic Green Tea", "Premium Japanese green tea, rich in antioxidants with a smooth flavor profile.", "food", 12.99, True, 4.3, ["beverage", "organic", "healthy"]),
        ("Leather Running Shoes", "Lightweight performance running shoes with responsive cushioning and breathable mesh upper.", "clothing", 89.99, False, 4.1, ["sports", "running", "footwear"]),
        ("Stainless Steel Water Bottle", "Double-walled vacuum insulated bottle, keeps drinks cold for 24 hours or hot for 12.", "home", 24.99, True, 4.8, ["kitchen", "eco-friendly", "insulated"]),
        ("Merino Wool Winter Scarf", "Luxuriously soft merino wool scarf, perfect for cold weather and formal occasions.", "clothing", 39.99, True, 4.2, ["winter", "accessory", "wool"]),
        ("USB-C Hub 7-in-1", "Multi-port adapter with HDMI 4K, SD card reader, USB 3.0 ports, and PD charging.", "electronics", 34.99, True, 4.0, ["accessory", "usb", "hub"]),
        ("Dark Chocolate Collection", "Assorted 72% cacao dark chocolates, single origin from 6 different countries.", "food", 24.99, True, 4.6, ["gift", "chocolate", "premium"]),
        ("Yoga Mat Premium", "Extra thick non-slip yoga mat with alignment lines, includes carrying strap.", "sports", 44.99, True, 4.4, ["fitness", "yoga", "exercise"]),
        ("Desk LED Lamp", "Adjustable LED desk lamp with 5 brightness levels, USB charging port, and eye-care technology.", "home", 39.99, True, 4.3, ["office", "lighting", "led"]),
    ]
    
    print("Inserting products...")
    for name, desc, cat, price, stock, rating, tags in products:
        embedding = model.encode([desc])[0].tolist()
        pid = insert_product(name, desc, cat, price, stock, rating, tags, embedding)
        print(f"  Inserted: {name} (id={pid})")
    
    # Build index
    build_index()
    
    # ── Demo queries ──
    print("\n═══ Query Demos ═══\n")
    
    # Query 1: Pure vector search
    print("1. Pure Vector Search: 'warm winter clothing'")
    results, elapsed = search_products("warm winter clothing", model, top_k=5)
    print(f"   ({elapsed*1000:.1f}ms)")
    for r in results[:5]:
        print(f"   [{r['similarity']:.4f}] {r['name']} — ${r['price']} ({r['category']})")
    print()
    
    # Query 2: Filtered by category
    print("2. Filtered: 'audio equipment' (category = electronics)")
    results, elapsed = search_products("audio equipment", model, category="electronics", top_k=5)
    print(f"   ({elapsed*1000:.1f}ms)")
    for r in results[:5]:
        print(f"   [{r['similarity']:.4f}] {r['name']} — ${r['price']}")
    print()
    
    # Query 3: Multiple filters
    print("3. Multi-Filter: 'something to wear' (clothing, in stock, < $100, rating >= 4)")
    results, elapsed = search_products(
        "something to wear", model,
        category="clothing", in_stock_only=True, max_price=100, min_rating=4.0,
        top_k=5
    )
    print(f"   ({elapsed*1000:.1f}ms)")
    for r in results[:5]:
        print(f"   [{r['similarity']:.4f}] {r['name']} — ${r['price']} (rating: {r['rating']})")
    print()
    
    # Query 4: What happens with bad data?
    print("4. Edge case: 'asdfghjkl' (nonsense query)")
    results, elapsed = search_products("asdfghjkl", model, top_k=5)
    print(f"   ({elapsed*1000:.1f}ms)")
    if results:
        print(f"   Top score: {results[0]['similarity']:.4f} (should be LOW for nonsense)")
        for r in results[:3]:
            print(f"   [{r['similarity']:.4f}] {r['name']}")
    print()
```

**Expected output:**
```
═══ pgvector Demo ═══

Inserting products...
  Inserted: Blue Cotton Sweater (id=1)
  Inserted: Wireless Bluetooth Headphones (id=2)
  ...

═══ Query Demos ═══

1. Pure Vector Search: 'warm winter clothing'
   (3.45ms)
   [0.8912] Blue Cotton Sweater — $49.99 (clothing)
   [0.8345] Merino Wool Winter Scarf — $39.99 (clothing)
   [0.4567] Stainless Steel Water Bottle — $24.99 (home)

2. Filtered: 'audio equipment' (category = electronics)
   (1.23ms)
   [0.8678] Wireless Bluetooth Headphones — $199.99
   [0.6234] USB-C Hub 7-in-1 — $34.99

3. Multi-Filter: 'something to wear' (clothing, in stock, < $100, rating >= 4)
   (0.89ms)
   [0.8912] Blue Cotton Sweater — $49.99 (rating: 4.5)
   [0.8345] Merino Wool Winter Scarf — $39.99 (rating: 4.2)

4. Edge case: 'asdfghjkl' (nonsense query)
   (3.21ms)
   Top score: 0.1234 (should be LOW for nonsense)
   [0.1234] Dark Chocolate Collection — $24.99
```

---

### Example 2: HNSW Index in pgvector

pgvector 0.7+ also supports HNSW indexes. Let's compare:

```python
"""
pgvector HNSW indexing.
Comparing IVFFlat vs HNSW speed and accuracy.
"""

import psycopg2
import numpy as np
import time
from sentence_transformers import SentenceTransformer


def compare_indexes(model):
    """Compare IVFFlat vs HNSW index performance."""
    
    conn = psycopg2.connect(dbname="ai_course", user="postgres", password="postgres")
    cur = conn.cursor()
    
    # Generate more test data: 1000 products
    print("Generating test data...")
    categories = ["clothing", "electronics", "food", "home", "sports", "books", "toys", "beauty"]
    
    # Insert varied products
    for i in range(100):
        name = f"Product_{i}"
        cat = categories[i % len(categories)]
        desc = f"This is a {cat} product with various features and qualities, item number {i}."
        price = np.random.uniform(5, 500)
        embedding = model.encode([desc])[0].tolist()
        
        cur.execute("""
            INSERT INTO products (name, description, category, price, in_stock, rating, tags, embedding)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s::vector)
            ON CONFLICT DO NOTHING;
        """, (name, desc, cat, price, True, round(np.random.uniform(3, 5), 1), ["test"], embedding))
    
    conn.commit()
    
    # Count total
    cur.execute("SELECT COUNT(*) FROM products;")
    total = cur.fetchone()[0]
    print(f"Total products: {total}")
    
    # ── Test 1: Brute force (no index) ──
    print("\n═══ Benchmark: No Index (Sequential Scan) ═══")
    cur.execute("DROP INDEX IF EXISTS products_embedding_idx;")
    conn.commit()
    
    query_vec = model.encode(["test query"])[0].tolist()
    
    start = time.time()
    n_runs = 10
    for _ in range(n_runs):
        cur.execute("""
            SELECT id, 1 - (embedding <=> %s::vector) AS sim
            FROM products
            ORDER BY embedding <=> %s::vector
            LIMIT 10;
        """, (query_vec, query_vec))
        _ = cur.fetchall()
    seq_time = (time.time() - start) / n_runs * 1000
    print(f"  Average query time: {seq_time:.2f} ms")
    
    # ── Test 2: IVFFlat index ──
    print("\n═══ Benchmark: IVFFlat Index ═══")
    
    lists = max(10, int(np.sqrt(total)))
    cur.execute(f"""
        CREATE INDEX products_embedding_ivfflat_idx
        ON products
        USING ivfflat (embedding vector_cosine_ops)
        WITH (lists = {lists});
    """)
    conn.commit()
    
    # Test with different probe values
    for probes in [1, 10, 50, 100]:
        cur.execute(f"SET ivfflat.probes = {probes};")
        
        start = time.time()
        for _ in range(n_runs):
            cur.execute("""
                SELECT id, 1 - (embedding <=> %s::vector) AS sim
                FROM products
                ORDER BY embedding <=> %s::vector
                LIMIT 10;
            """, (query_vec, query_vec))
            _ = cur.fetchall()
        avg_time = (time.time() - start) / n_runs * 1000
        
        print(f"  probes={probes:4d}: {avg_time:.2f} ms")
    
    cur.execute("DROP INDEX products_embedding_ivfflat_idx;")
    conn.commit()
    
    # ── Test 3: HNSW index ──
    print("\n═══ Benchmark: HNSW Index ═══")
    
    cur.execute(f"""
        CREATE INDEX products_embedding_hnsw_idx
        ON products
        USING hnsw (embedding vector_cosine_ops)
        WITH (m = 32, ef_construction = 200);
    """)
    conn.commit()
    
    # Test with different ef_search values
    for ef_search in [10, 50, 100, 200]:
        cur.execute(f"SET hnsw.ef_search = {ef_search};")
        
        start = time.time()
        for _ in range(n_runs):
            cur.execute("""
                SELECT id, 1 - (embedding <=> %s::vector) AS sim
                FROM products
                ORDER BY embedding <=> %s::vector
                LIMIT 10;
            """, (query_vec, query_vec))
            _ = cur.fetchall()
        avg_time = (time.time() - start) / n_runs * 1000
        
        print(f"  ef_search={ef_search:3d}: {avg_time:.2f} ms")
    
    cur.close()
    conn.close()


# ── Run comparison ──
model = SentenceTransformer('all-MiniLM-L6-v2')
compare_indexes(model)
```

**Expected output:**
```
═══ Benchmark: No Index (Sequential Scan) ═══
  Average query time: 15.23 ms

═══ Benchmark: IVFFlat Index ═══
  probes=   1: 1.23 ms
  probes=  10: 2.45 ms
  probes=  50: 4.67 ms
  probes= 100: 8.12 ms

═══ Benchmark: HNSW Index ═══
  ef_search= 10: 0.89 ms
  ef_search= 50: 1.23 ms
  ef_search=100: 1.89 ms
  ef_search=200: 3.12 ms
```

**🤔 Analysis questions:**
1. Which index is faster at query time? (HNSW)
2. Which index is more affected by parameter tuning? (IVFFlat — probes matters a LOT)
3. What's the tradeoff between speed and accuracy? (More probes/ef_search = slower but more accurate)
4. At what scale would the sequential scan become IMPOSSIBLE? 

---

### Example 3: The pgvector-Qdrant Showdown

Let's compare pgvector and Qdrant head-to-head:

```python
"""
pgvector vs Qdrant: head-to-head comparison.
Understanding when to use which.
"""

import numpy as np
import time

# ── Scenario parameters ──
n_docs = 100_000
dim = 384
top_k = 10

print("═══ pgvector vs Qdrant: Theoretical Comparison ═══\n")
print(f"Dataset: {n_docs:,} documents, {dim} dimensions\n")

# ── Dimension 1: Search Speed ──
print("1. SEARCH SPEED")
print("─" * 50)
print(f"  pgvector (HNSW): ~{0.5 + n_docs/1000000 * 2:.1f} ms (depends on ef_search)")
print(f"  Qdrant (HNSW):   ~{0.3 + n_docs/1000000 * 1:.1f} ms (optimized Rust)")
print(f"  Difference: Qdrant is typically 2-3x faster for pure vector search")
print()

# ── Dimension 2: Hybrid Search Speed ──
print("2. HYBRID SEARCH (vector + SQL filters)")
print("─" * 50)
print("  pgvector:")
print("    ✓ Single query: SELECT ... WHERE category='x' ORDER BY embedding <=> ...")
print("    ✓ No network overhead, no JOIN round-trips")
print()
print("  Qdrant:")
print("    ✓ Pre-filtering during search")
print("    ✗ Need separate SQL query for rich joins")
print("    ✗ Network overhead for each query")
print()

# ── Dimension 3: Operational Cost ──
print("3. OPERATIONAL COST (monthly)")
print("─" * 50)
pgvector_storage_overhead = n_docs * dim * 4 / 1024**3  # GB for vectors
pgvector_storage = pgvector_storage_overhead * 1.5 + 1  # vectors + data + overhead

print(f"  pgvector:")
print(f"    Storage (vectors only): {pgvector_storage_overhead:.2f} GB")
print(f"    Total DB size (est): {pgvector_storage:.1f} GB")
print(f"    Managed PostgreSQL (RDS): ~$50-200/month (db.r6g.large)")
print()
print(f"  Qdrant (self-hosted):")
print(f"    Storage: ~{pgvector_storage_overhead/2:.1f} GB (Rust, more compact)")
print(f"    VM cost: ~$20-50/month (single instance)")
print(f"    Ops cost: ~5 hours/month (backups, monitoring, updates)")
print()

# ── Dimension 4: Consistency ──
print("4. DATA CONSISTENCY")
print("─" * 50)
print("  pgvector:")
print("    ✓ ACID transactions: vector update + metadata update = atomic")
print("    ✓ Point-in-time recovery: all data backed up together")
print("    ✓ Foreign keys work naturally")
print()
print("  Qdrant:")
print("    ✗ Eventual consistency between Qdrant and your SQL DB")
print("    ✗ Sync code required (or use change data capture)")
print("    ✗ If sync breaks, your search returns stale results")
print()

# ── Dimension 5: The Breaking Point ──
print("5. THE BREAKING POINT")
print("─" * 50)
print("  pgvector breaks when:")
print("    - > 5M vectors with high query load (PostgreSQL struggles)")
print("    - > 2000 dimensions (pgvector limit)")
print("    - Need < 10ms P99 latency at high concurrency")
print()
print("  Qdrant breaks when:")
print("    - Your team can't manage infrastructure")
print("    - Your data needs complex SQL joins with search")
print("    - Your budget can't support two databases")
print()

# ── Decision ──
print("═══ DECISION GUIDE ═══")
print()
scores = {
    "pgvector": {"vector_speed": 7, "hybrid_speed": 10, "ops_cost": 9, "consistency": 10, "scale_limit": 5},
    "Qdrant":   {"vector_speed": 9, "hybrid_speed": 6, "ops_cost": 6, "consistency": 4, "scale_limit": 9},
}

for db, metrics in scores.items():
    avg = sum(metrics.values()) / len(metrics)
    print(f"  {db}: avg score {avg:.1f}/10")
    for metric, score in metrics.items():
        bar = "█" * score + "░" * (10 - score)
        print(f"    {metric:15s} {bar} {score}/10")
    print()
```

---

## ✅ Good Output Examples

### What Strong pgvector Usage Looks Like

```
Scenario: You're building an e-commerce site with PostgreSQL already.
Products table has 500K rows. You need semantic search.

Strong approach:
  1. Add vector(384) column to products table
  2. Create a trigger: on INSERT/UPDATE, auto-embed the description
  3. Create an HNSW index on the embedding column
  4. Build search with parameterized SQL
  5. One database, zero sync code, ACID consistency

STRONG because:
  ✓ No new infrastructure
  ✓ Products and vectors are always in sync (same transaction)
  ✓ You can join with reviews, inventory, orders IN THE SEARCH QUERY
  ✓ SQL injection protection via parameterized queries
  ✓ Your existing backup/monitoring covers the vectors too
```

### What Weak pgvector Usage Looks Like

```
Scenario: Same as above, but vectors are stored in a SEPARATE JSONB column.

Weak approach:
  1. Store embedding as JSONB array in a separate column
  2. No vector index (JSONB can't use pgvector indexes)
  3. Load all 500K embeddings into Python for brute-force search
  4. JOIN after the search, not during

WEAK because:
  ✗ No index = sequential scan every time (30+ seconds for 500K)
  ✗ JSONB is ~3x larger than native vector type
  ✗ No native distance operators (need to implement in Python)
  ✗ Can't order by similarity in SQL
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Not Indexing

```python
# ❌ WRONG — No index, expecting good performance
cur.execute("""
    SELECT * FROM products
    ORDER BY embedding <=> %s::vector
    LIMIT 10;
""")
# For 100K docs: ~1-5 seconds per query
# For 1M docs: ~10-50 seconds — UNUSABLE

# ✅ RIGHT — Create an index first
cur.execute("""
    CREATE INDEX ON products
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
""")
# Then: sub-10ms queries even at 100K docs
```

### Antipattern 2: Wrong Index for Your Data

```
IVFFlat:
  ✓ Fast to build (minutes)
  ✓ Low memory
  ✓ Good for: static datasets, bulk-loaded data
  ✗ Requires periodic REINDEX if data changes a lot
  ✗ Accuracy depends on good initial clusters

HNSW:
  ✓ Fast search (2-3x faster than IVFFlat)
  ✓ Handles inserts/updates well (incremental)
  ✓ Higher recall
  ✗ Slower to build
  ✗ More memory (2-3x)
  ✗ Needs periodic VACUUM to remove deleted entries

CHOOSE:
  IVFFlat if: data is mostly static, memory is tight, you can reindex nightly
  HNSW if: data changes frequently, search speed is critical, you have RAM
```

### Antipattern 3: Setting probes to 1

```python
# ❌ WRONG — probes=1 means only search ONE cluster
SET ivfflat.probes = 1;
# You'll miss ~40% of good results!
# Basically useless for any real application

# ✅ RIGHT — tune probes based on your recall needs
SET ivfflat.probes = 10;   # Good balance for most apps
SET ivfflat.probes = 50;   # High recall, slower
SET ivfflat.probes = 100;  # Almost exact, much slower

# Rule of thumb: probes = sqrt(lists)
# If lists = 100, probes = 10
```

### Antipattern 4: Not Updating Embeddings When Data Changes

```sql
-- Products table has an ON UPDATE trigger?
-- If the product description changes, the embedding is now WRONG!

-- ❌ WRONG — manual sync
UPDATE products SET description = 'New description' WHERE id = 123;
-- Whoops, embedding still reflects the OLD description!

-- ✅ RIGHT — use a trigger or application-level consistency
-- Option A: Trigger function
CREATE OR REPLACE FUNCTION update_embedding()
RETURNS TRIGGER AS $$
BEGIN
    -- Call Python embedding function (via plpython3u)
    -- Or: set a flag, and have a background worker re-embed
    NEW.needs_reembed = true;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Option B: Application-level (cleaner)
class ProductService:
    def update_product(self, product_id, new_data):
        # Update in one transaction
        with self.conn:
            self.conn.execute("UPDATE products SET ... WHERE id = %s", ...)
            new_embedding = self.model.encode([new_data["description"]])
            self.conn.execute("UPDATE products SET embedding = %s WHERE id = %s",
                            (new_embedding, product_id))
```

### Failure Mode: The "IVFFlat Needs a Good Initial Clustering"

```
IVFFlat creates clusters during indexing. If your initial data distribution
is WEIRD (e.g., 90% of vectors are about "sports" and 10% about "cooking"),
the clusters will be imbalanced.

When you later add 10,000 new "cooking" documents, they all fall into
one cluster — overwhelming it. Search for "cooking" terms becomes slow
because that cluster is now huge.

SOLUTION:
1. Ensure your sample data for INDEX creation is representative of the full distribution
2. Reindex periodically (cron job: REINDEX INDEX products_embedding_idx)
3. Or use HNSW, which handles incremental inserts better
```

---

## 🧪 Drills & Challenges

### Drill 1: pgvector Setup & Seed (15 min)

1. Install PostgreSQL + pgvector on your machine
2. Create a database
3. Create a table with a `vector(384)` column
4. Insert 100 products with embeddings
5. Create an IVFFlat index

**If you hit errors:** Docker PostgreSQL + pgvector is the easiest path.

### Drill 2: The Filter-Pushdown Experiment (25 min)

Test whether pgvector applies filters BEFORE or AFTER vector search:

```sql
-- Experiment 1: Selective filter
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category = 'rare_category'  -- Only 5 products match
ORDER BY embedding <=> %s::vector
LIMIT 10;

-- Experiment 2: Non-selective filter
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category = 'common_category'  -- 50% of products match
ORDER BY embedding <=> %s::vector
LIMIT 10;
```

**Question:** Which is faster? What does `EXPLAIN ANALYZE` tell you about the query plan?

**Expected finding:** PostgreSQL pushes filters down when they're selective enough. But if the filter matches too many rows, it may do a full scan anyway.

### Drill 3: The Index Tuning Sweep (30 min)

1. Create a table with 50K random vectors
2. Build IVFFlat with different `lists` values: [10, 50, 100, 500, 1000]
3. For each, measure:
   - Index build time
   - Query time at probes = 1, 10, 50
   - Recall@10 compared to exact search
4. Plot: lists vs recall, lists vs query time

**Purpose:** You'll find the "sweet spot" for YOUR data size. No blog post can tell you this.

### Drill 4: The Hybrid Search Application (35 min)

Build a simple API that combines SQL JOINs with vector search:

```python
# Your task: Implement "search products with their average review rating"
# 
# Tables needed:
#   products (id, name, description, category, price, embedding)
#   reviews (id, product_id, rating, text)
#
# The query should:
# 1. Find products similar to the query text
# 2. Include each product's average review rating
# 3. Filter by category and price range
# 4. Return results in one SQL statement

def search_with_reviews(query, category, max_price):
    # ONE SQL query — no round-trips
    sql = """
        SELECT p.*, 
               1 - (p.embedding <=> %s::vector) AS similarity,
               COALESCE(AVG(r.rating), 0) AS avg_review_rating
        FROM products p
        LEFT JOIN reviews r ON p.id = r.product_id
        WHERE p.category = %s AND p.price <= %s
        GROUP BY p.id
        ORDER BY similarity DESC
        LIMIT 10;
    """
    ...
```

**Key insight:** This is a REAL query you'd write in production. It joins semantic search with review aggregation in one round-trip. Qdrant can't do this natively.

### Drill 5: The "When to Leave PostgreSQL" Threshold (20 min)

Design an experiment to find pgvector's breaking point:
1. Start with 10K vectors, measure P99 latency
2. Increase to 50K, 100K, 500K, 1M
3. At each step: measure latency, recall, and memory
4. Find the point where latency exceeds your acceptable threshold

**Document your findings:**
- At what count did you need an index? (Should be immediate)
- At what count did you need HNSW instead of IVFFlat?
- At what count would you MIGRATE to Qdrant?

---

## 🚦 Gate Check

Before moving to File 07 (Advanced Vector Search), confirm you can:

- [ ] **Set up pgvector** in PostgreSQL
- [ ] **Create a table with vector column** and insert embeddings
- [ ] **Build both IVFFlat and HNSW indexes**
- [ ] **Write hybrid queries** combining SQL filters with vector search
- [ ] **Explain when pgvector beats Qdrant** and when it doesn't
- [ ] **Answer these questions:**
  1. What's the maximum vector dimension pgvector supports?
  2. When would you choose IVFFlat over HNSW?
  3. What does `SET ivfflat.probes = N;` actually do?
  4. How does pgvector's consistency model differ from Qdrant's?
  5. Your PostgreSQL has 500K products with vectors. Search is getting slow. What do you check first? (Index, hardware, query, or data distribution?)
  6. Is it better to have one database (PostgreSQL) with good-enough vector search, or two databases (PostgreSQL + Qdrant) with excellent vector search?

- [ ] **Make a decision:** For YOUR current or next project, would you use pgvector? Why or why not?

---

## 📚 Resources

**Official:**
- [pgvector GitHub](https://github.com/pgvector/pgvector) — README has everything
- [pgvector Documentation](https://github.com/pgvector/pgvector#usage) — Simple and clear
- [PostgreSQL Indexes for Vector Data](https://www.postgresql.org/docs/current/indexes-types.html)

**Performance Guides:**
- "pgvector Performance at 10M Vectors" — Various benchmarks you should read
- "IVFFlat vs HNSW: A Practical Comparison" — Test with your own data
- "PostgreSQL Connection Pooling with pgvector" — For production deployments

**Production Patterns:**
- Using `ON CONFLICT` for upserts with embeddings
- Scheduled REINDEX for IVFFlat maintenance
- `EXPLAIN ANALYZE` for query tuning

**Your Decision Flowchart:**
```
START: Should you use pgvector?
│
├── Already use PostgreSQL? → YES → Strong candidate
│   └── Need < 5M vectors? → YES → pgvector is likely the answer
│       └── Need sub-10ms P99? → consider HNSW index
│
├── Not using PostgreSQL? → Consider if you should switch
│   ├── Need ACID consistency + vectors? → pgvector
│   └── Vectors only, no relational data? → Qdrant may be simpler
│
├── Need > 5M vectors AND high throughput? → Consider Qdrant
│   └── Or: pgvector + partitioning (advanced)
│
└── Need > 2000 dimensions? → pgvector can't do it → Qdrant
```

---

> **🛑 Revisit the 3 discovery questions from the beginning:**
>
> 1. **One DB vs Two DBs** — Does pgvector's operational simplicity outweigh its performance limits for YOUR use case?
> 2. **The JOIN Problem** — You've seen how one SQL query can combine vectors, filters, and aggregations. Can you see when this is game-changing vs when it doesn't matter?
> 3. **PostgreSQL's Vector Indexes** — Do you now know when to use IVFFlat vs HNSW? Can you explain the tradeoff?
>
> **Key realization:** pgvector isn't "worse" than Qdrant — it's DIFFERENT. It trades raw vector search speed for SQL integration, operational simplicity, and ACID consistency. The best engineer knows which tradeoff to make for their specific situation.
>
> **Next:** File 07 — Advanced Vector Search. Combining everything you've learned: hybrid search (keyword + vector), re-ranking, and production search architecture.
