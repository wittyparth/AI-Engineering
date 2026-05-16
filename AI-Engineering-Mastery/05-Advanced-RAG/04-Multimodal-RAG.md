# Multimodal RAG

## The Problem

Documents contain more than text: images, tables, diagrams, charts. Standard RAG ignores all of them.

## Approaches (Ranked by Quality)

### 1. Multimodal Embedding Models (Best)
Models like `jina-clip-v1` embed text AND images in the same space:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("jinaai/jina-clip-v1")

# Embed text and images in the same space
text_emb = model.encode("A diagram showing neural network architecture")
image_emb = model.encode(Image.open("architecture.png"))

# Now you can search images with text queries
similarity = cosine_similarity(text_emb, image_emb)
```

### 2. Image Captioning + Text RAG (Good fallback)
```python
class CaptioningRAG:
    def __init__(self):
        self.captioner = pipeline("image-to-text", model="blip-large")
        self.text_rag = StandardRAG()

    async def ingest(self, document: Document):
        for image in document.images:
            caption = self.captioner(image)
            # Store caption as if it were text
            self.text_rag.add_chunk(
                content=f"[Image: {image.path}] {caption}",
                metadata={"type": "image", "source": document.source}
            )
```

### 3. Multimodal LLM + Raw Images (Most powerful, most expensive)
```python
class MultimodalRAG:
    def __init__(self, llm: str = "gpt-4o"):
        self.client = OpenAI()

    async def answer(self, query: str, retrieved_chunks: list) -> str:
        messages = [{"role": "system", "content": "Answer based on provided context."}]

        # Add text context
        text_content = [c.content for c in retrieved_chunks if c.type == "text"]

        # Add images directly
        image_content = [c.image for c in retrieved_chunks if c.type == "image"]

        messages.append({"role": "user", "content": [
            {"type": "text", "text": f"Context:\n{''.join(text_content)}\n\nQuestion: {query}"},
            *[{"type": "image_url", "image_url": {"url": img}} for img in image_content],
        ]})

        return await self.client.chat.completions.create(
            model="gpt-4o", messages=messages
        )
```

## 🔴 Senior: Multimodal Cost

| Strategy | Storage | Retrieval | Generation | Quality |
|----------|---------|-----------|------------|---------|
| Caption-only | Low | Standard | Standard | Poor (loses visual info) |
| Multimodal embedding | Medium | Hybrid | Standard | Good |
| Full multimodal | Low (URLs) | Standard | High cost | Best |

For most applications: use multimodal embeddings + captions as a fallback. Reserve full multimodal generation for when visual detail is critical.
