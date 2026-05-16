# Multimodal Systems

## What Multimodal Means

Processing multiple types of data in a single system:
- Text + Images (most common)
- Text + Audio (voice assistants)
- Text + Video (video understanding)
- Text + Tables (document analysis)

## Multimodal Architectures

### 1. Unified Model (GPT-4o, Gemini)
```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": {"url": "https://example.com/photo.jpg"}},
        ],
    }],
)
```

### 2. Separate Encoder + LLM
```python
# CLIP for images + LLM for text
image_features = clip_model.encode(image)
text_description = caption_model.generate(image_features)
# Then feed description into your regular RAG pipeline
```

### 3. Multi-Vector RAG
```python
# Store text and image embeddings in the same vector space
# Use multimodal embedding model (Jina CLIP, etc.)
text_emb = embedder.encode("A cat sitting on a chair")
image_emb = embedder.encode(cat_image)
# Same space — search image with text and vice versa
```

## Building a Multimodal RAG Pipeline

```python
class MultimodalRAG:
    def __init__(self, text_embedder, image_embedder, vector_db, llm):
        self.text_embedder = text_embedder
        self.image_embedder = image_embedder
        self.db = vector_db
        self.llm = llm

    async def ingest_document(self, doc: Document):
        for page in doc.pages:
            # Embed text
            text_emb = self.text_embedder.embed(page.text)
            # Embed images on the page
            for img in page.images:
                img_emb = self.image_embedder.embed(img)
                self.db.store(img_emb, {"type": "image", "caption": img.caption, "data": img})
            # Store text
            self.db.store(text_emb, {"type": "text", "content": page.text})

    async def search(self, query: str, k: int = 5) -> list:
        query_emb = self.text_embedder.embed(query)
        results = self.db.search(query_emb, k=k)
        text_results = [r for r in results if r["type"] == "text"]
        image_results = [r for r in results if r["type"] == "image"]
        return text_results, image_results

    async def answer(self, query: str) -> str:
        texts, images = await self.search(query)
        prompt = f"Context: {' '.join(texts)}\n"
        if images:
            # Include image references
            prompt += f"\nRelevant images: {[img['caption'] for img in images[:3]]}"
        prompt += f"\nQuestion: {query}\nAnswer:"
        return await self.llm.generate(prompt)
```

## 🔴 Senior: Multimodal Cost vs Benefit

| Type | Storage Cost | Retrieval Cost | Quality Gain |
|------|-------------|---------------|-------------|
| Text-only captions | Negligible | Standard | Misses visual info |
| Image embeddings | 2x more vectors | 2x more search | Catches visual similarity |
| Full multimodal LLM | Standard | Standard | Best quality, 3x cost |

**Rule**: Try text-only + captions first. Add multimodal only when the visual information is critical.
