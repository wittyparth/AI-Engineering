# 05 — Multimodal RAG: Retrieving and Reasoning Over Images + Text

## 🎯 Purpose & Goals

Your RAG system can find text. But your documents contain more than text — they contain diagrams, screenshots, charts, infographics, and photographs.

A user asks: *"Looking at the architecture diagram from the migration guide, which services communicate through the message bus?"*

Standard RAG retrieves the text chunk that *mentions* the architecture diagram. That text might say "Figure 1: System Architecture" and nothing more useful. The actual answer is in the IMAGE — the arrows, labels, and connections that the diagram encodes visually.

Another user asks: *"The screenshot in the bug report shows error code E-1042 — what does it mean?"*

The answer requires finding the screenshot and reading text from within the image. Standard RAG has zero capability for this.

**Multimodal RAG** solves this by extending retrieval and generation to handle MULTIPLE modalities — images, text, and in advanced cases, audio and video.

**By the end of this module, you will:**

1. Understand the **three strategies** for multimodal RAG and when to use each
2. Build a **multimodal embedding pipeline** (using CLIP) that can retrieve images and text in the same space
3. Implement **vision LLM RAG** — retrieving text context, then passing images to vision models for understanding
4. Build **image-to-text preprocessing** — extracting text from diagrams and screenshots before indexing
5. Implement a **unified multimodal retriever** that seamlessly handles text-only, image-only, and mixed queries
6. Understand the **cost-latency-quality landscape** of multimodal vs. text-only RAG
7. Know the **failure modes unique to multimodal retrieval**

---

### ⏹ STOP. Answer these before proceeding.

**Question 1 — The Diagram Problem**

*Your company's database migration guide has a section that says "Refer to Figure 3 for the replication topology." The actual topology information — which nodes are primary, which are replicas, where failover routes — is ONLY in the diagram image.*

*A user asks: "In the migration guide's replication topology, is there a read replica in the eu-west region?"*

*Standard text-only RAG would retrieve the chunk containing "Refer to Figure 3 for the replication topology" — which gives the LLM NOTHING useful about replicas. The LLM either guesses (wrong) or says "the document doesn't specify" (wrong — it DOES specify, in the image).*

*What are ALL the ways you could handle this? Think about:*
1. *Extracting text from the image (OCR/alt-text) before indexing*
2. *Generating descriptions of the image and indexing those*
3. *Using vision models that can directly read the image at query time*
4. *Storing image embeddings and retrieving by visual similarity*

*What are the tradeoffs of each approach? When would you pick one over another?*

**Question 2 — The Screenshot Problem**

*Your bug tracker has screenshots of error messages. A user uploads a screenshot showing "Error 0x80070005 — Access Denied" and asks "What does this error mean and how do I fix it?"*

*The error text is embedded IN the image. No text chunk mentions this specific error. The user's query is also an image (a screenshot).*

*Your options are:*
1. *OCR every screenshot at index time and index the extracted text*
2. *CLIP-embed every screenshot and retrieve by visual similarity*
3. *Use a vision model to describe each screenshot, then index the description*
4. *At query time, use a vision model to extract the error code, then text-search*

*Which approach would preserve the MOST information? Which would be FASTEST for retrieval? Which would be CHEAPEST?*

*What if the user's query is TEXT ("I'm getting error 0x80070005") but the answer is in a SCREENSHOT image? Now it's a cross-modal query (text searches for images).*

**Question 3 — The Chart Comprehension Problem**

*A product analytics dashboard shows a quarterly revenue chart. The X-axis has months, the Y-axis has revenue in millions, and there's a clear dip in February followed by recovery in March.*

*A user asks: "Based on the Q1 revenue chart, which month had the lowest revenue and what was the recovery trend?"*

*This requires:*
1. *Finding the correct chart image among many*
2. *Reading the axes, values, and trends from the image*
3. *Interpreting the visual information correctly*
4. *Answering using chart-specific vocabulary ("dip," "recovery," "trend")*

*A text-only system might index the chart's ALT TEXT: "Q1 2025 Revenue Chart." That's useless for answering the question. A vision system can read the actual data.*
*But vision models are expensive and slow. An intermediate approach extracts chart data programmatically (OCR + chart type detection).*

*What's the OPTIMAL approach for chart-based questions? How do you balance:*
- *Accuracy of chart reading (vision LLM is best)*
- *Cost per query ($0.01-0.05 per vision call)*
- *Latency (2-5 seconds per vision call)*
- *Indexing depth (how much chart info do you pre-extract?)*

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| The Multimodal Landscape | 15 min |
| Strategy 1: Image-to-Text Preprocessing | 30 min |
| Strategy 2: Multimodal Embeddings (CLIP) | 45 min |
| Strategy 3: Vision LLM at Query Time | 30 min |
| Hybrid Strategy: The Best of All Worlds | 30 min |
| Code: CLIP Embedding Pipeline | 45 min |
| Code: Unified Multimodal Retriever | 45 min |
| Code: Vision LLM RAG | 45 min |
| Cost & Latency Analysis | 15 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~7 hours** |

---

## 📖 Concept Deep-Dive

### 1. The Three Strategies of Multimodal RAG

There are fundamentally THREE strategies for multimodal RAG. Each makes different tradeoffs between indexing complexity, retrieval quality, and query-time cost:

```
STRATEGY 1: Index-Time Preprocessing (Cheapest, Most Common)
┌──────────┐    ┌───────────────┐    ┌──────────┐
│  Image   │───→│  Extract text  │───→│  Index   │
│          │    │  (OCR/alt-text)│    │  as text │
└──────────┘    └───────────────┘    └──────────┘
  Original                           Embedding space
  modality                           TEXT only
  lost ▲

STRATEGY 2: Multimodal Embeddings (Moderate, Most Flexible)
┌──────────┐    ┌───────────────┐    ┌──────────┐
│  Text    │───→│  Text Encoder  │───→│  Joint   │
│  Image   │───→│  Image Encoder │───→│  Space   │
└──────────┘    └───────────────┘    └──────────┘
  Both                                Text + Image
  modalities                          in same space

STRATEGY 3: Vision LLM at Query Time (Most Expensive, Most Powerful)
┌──────────┐    ┌───────────────┐    ┌──────────┐
│  Query   │───→│  Retrieve text │───→│  Pass    │
│  + Image │    │  + image refs  │    │  images  │
└──────────┘    └───────────────┘    │  to      │
                                     │  vision  │
                                     │  LLM     │
                                     └──────────┘
                                       Full visual
                                       understanding
```

| Strategy | Indexing Cost | Query Cost | Image Understanding | Best For |
|----------|--------------|------------|-------------------|----------|
| 1. Image→Text | One-time OCR/extraction | Very low | Limited (text only) | Screenshots with embedded text |
| 2. Multimodal Embeds | One-time CLIP embed | Low-Medium | Good (visual similarity) | Finding relevant images |
| 3. Vision LLM | None | High (per query) | Excellent (full vision) | Complex visual reasoning |

**The critical insight:** You should combine ALL THREE in production. Strategy 1 is for high-recall text extraction from images. Strategy 2 is for retrieving the RIGHT images. Strategy 3 is for understanding the image content at query time.

### 2. Strategy 1: Image-to-Text Preprocessing

The simplest approach: extract as much text as possible from images BEFORE indexing, and treat the extracted text as if it were regular document content.

**What you can extract:**

| Technique | What It Captures | Quality |
|-----------|-----------------|---------|
| **OCR (Tesseract, EasyOCR)** | Text rendered in images | High for clean text, low for handwritten |
| **Alt-text / captions** | Human-written descriptions | Variable (depends on author) |
| **Image metadata (EXIF)** | Camera settings, dates, GPS | High for technical metadata |
| **Vision model descriptions** | LLM-generated image descriptions | Good for scenes, weak for specific details |
| **Chart data extraction** | Values from charts/graphs | Good for standard chart types |

```python
# What gets indexed:
{
    "image_path": "figures/replication-topology.png",
    "ocr_text": "Primary → Replica 1 (eu-west) → Replica 2 (us-east)",
    "alt_text": "Database replication topology diagram",
    "llm_description": "Three database nodes arranged in a star topology...",
    "surrounding_text": "Refer to Figure 3 for the replication topology..."
}
```

**The limitation:** You lose VISUAL information — color coding, spatial relationships, visual hierarchy. An OCR extraction of a diagram tells you the text labels, but not their positions or connections.

### 3. Strategy 2: Multimodal Embeddings (CLIP Deep Dive)

CLIP (Contrastive Language-Image Pre-training, OpenAI 2021) learns a JOINT embedding space where text and images are represented in the same vector space.

**How CLIP works:**

```
Text: "A dog playing in the park" ──→ Text Encoder ──→ [0.2, 0.7, -0.1, ...]
                                                              │
                                                         SAME SPACE
                                                              │
Image: [pixel array] ──→ Image Encoder ──→ [0.3, 0.6, -0.2, ...]
```

The training objective: push embeddings of matching text-image pairs TOGETHER and non-matching pairs APART. The result: you can search images with text queries and vice versa.

**Why this is transformative for RAG:**

- **Zero-shot image retrieval:** Search your image corpus with ANY text query — no training needed
- **Cross-modal search:** Text query → image results. Image query → text results. Text+image query → mixed results
- **Unified index:** Store both text embeddings AND image embeddings in the same vector database with the same index

**Limitations of CLIP for RAG:**

| Limitation | Impact | Mitigation |
|-----------|--------|------------|
| CLIP is trained on web images — fails on domain-specific visuals | Poor retrieval for medical/technical diagrams | Fine-tune CLIP on your domain, or use domain-specific models like BiomedCLIP |
| CLIP sees 224×224 pixels — loses detail | Can't read small text in screenshots | Enlarge or tile images before embedding; combine with OCR |
| CLIP captures "scene" level, not "detail" level | "What color is the third bar in the chart?" — CLIP doesn't know | Use vision LLM for detail questions |
| No spatial reasoning | "Which arrow points from A to B?" — CLIP can't tell | Vision LLM or pre-extract relationship data |

### 4. Strategy 3: Vision LLM at Query Time

The most powerful approach: retrieve relevant images AND text, then pass BOTH to a vision-capable LLM (GPT-4o, Claude 3.5 Sonnet, Gemini 1.5 Pro) for understanding.

**The pipeline:**

```
Query: "In the migration diagram, what connects to the message bus?"

1. Retrieve text chunks about migration diagram
2. Find image references in those chunks → load the actual images
3. Build multimodal context: text chunks + image data
4. Query vision LLM with: [text_context] + [image_data] + [query]

Vision LLM sees the actual image and answers from it.
```

**The cost reality:**

| Model | Image Input Cost | Typical Latency | Understanding Quality |
|-------|-----------------|-----------------|---------------------|
| GPT-4o | ~$0.002 per image (via tokens) | 2-5s | Excellent — reads text, understands diagrams |
| Claude 3.5 Sonnet | ~$0.003 per image | 3-6s | Excellent — strong at charts and graphs |
| Gemini 1.5 Pro | ~$0.001 per image | 1-3s | Good — very fast, sometimes misses details |
| GPT-4o-mini | ~$0.0005 per image | 1-2s | Good for simple images, misses nuance |

**Production rule:** Only use vision LLM when the question requires image understanding. Route text-only questions to standard RAG.

### 5. The Hybrid Strategy — Production Best Practice

The production-proven approach combines ALL THREE strategies in a pipeline:

```
Indexing Phase:
┌──────────┐    ┌──────────────────┐    ┌──────────────┐
│ Document  │───→│    Split into     │───→│  Text chunks  │──+─→ Chunk embeddings
│ (text +   │    │  text chunks +    │    │                │  │
│  images)  │    │  extract images   │    ├──────────────┤  +─→ CLIP embeddings
└──────────┘    └──────────────────┘    │  Images        │  │
                                         │  + OCR text   │  +─→ OCR text embeddings
                                         │  + description│  │
                                         └──────────────┘  +─→ Description embeddings
                                         
Query Phase:
┌──────────┐    ┌──────────────────┐    ┌───────────────────────┐
│  Query   │───→│ Query Classifier  │───→│ Is image understanding │
└──────────┘    │ (text-only vs.    │    │ needed?               │
                 │  multimodal)      │    │                        │
                 └──────────────────┘    │ YES:                    │
                                         │  Retrieve text + images │
                                         │  → Vision LLM          │
                                         │                         │
                                         │ NO:                     │
                                         │  Retrieve text only     │
                                         │  → Standard LLM        │
                                         └───────────────────────┘
```

**The key patterns:**
- ALL images get OCR'd and described at index time (Strategy 1)
- ALL images get CLIP-embedded for retrieval (Strategy 2)
- ONLY queries needing visual understanding use vision LLM (Strategy 3)
- Text-only queries never pay the vision cost

---

## 💻 Code Examples

### 1. Core: Image Preprocessing Pipeline

```python
"""
image_preprocessing.py — Extract text and descriptions from images
for RAG indexing.

This implements Strategy 1: converting image information into
searchable text before retrieval.
"""

from typing import List, Optional, Dict, Any, Tuple, Literal
from dataclasses import dataclass, field
from pathlib import Path
import base64
import io
import asyncio
import logging

from openai import AsyncOpenAI
from PIL import Image as PILImage

logger = logging.getLogger(__name__)


# ─── Data Models ────────────────────────────────────────────────────────────

@dataclass
class ImageContent:
    """Extracted content from a single image."""
    image_path: str
    image_format: str  # png, jpg, webp, etc.
    width: int
    height: int

    # Text extraction (Strategy 1)
    ocr_text: str = ""
    alt_text: str = ""
    caption: str = ""
    surrounding_text: str = ""

    # Vision model description (Strategy 1, high quality)
    llm_description: str = ""

    # For CLIP embedding (Strategy 2)
    clip_embedding: Optional[List[float]] = None

    # Metadata
    document_source: str = ""
    page_number: Optional[int] = None
    figure_number: Optional[str] = None


@dataclass
class ImageIndexEntry:
    """An entry in the multimodal index."""
    image_id: str
    image_path: str

    # Multiple text representations for cross-modal search
    searchable_texts: Dict[str, str] = field(default_factory=dict)
    # e.g., {"ocr": "...", "description": "...", "alt": "...", "surrounding": "..."}

    # Embeddings (all in the same space if using CLIP-like model)
    embedding: Optional[List[float]] = None

    # Metadata for filtering
    metadata: Dict[str, Any] = field(default_factory=dict)


# ─── OCR Extraction ─────────────────────────────────────────────────────────

class OCRExtractor:
    """
    Extract text from images using OCR.

    Uses EasyOCR by default (works offline, good accuracy).
    Falls back to pytesseract if EasyOCR is unavailable.
    """

    def __init__(self, engine: Literal["easyocr", "tesseract", "vision_llm"] = "easyocr"):
        self.engine = engine
        self._reader = None

    async def _init_reader(self):
        """Lazy-initialize the OCR reader."""
        if self._reader is None and self.engine == "easyocr":
            try:
                import easyocr
                self._reader = easyocr.Reader(["en"], gpu=False)
            except ImportError:
                logger.warning("EasyOCR not installed, falling back to vision LLM")
                self.engine = "vision_llm"

    async def extract(self, image: PILImage.Image) -> str:
        """Extract text from an image."""
        await self._init_reader()

        if self.engine == "easyocr" and self._reader:
            import numpy as np
            result = self._reader.readtext(np.array(image))
            texts = [item[1] for item in result]
            return "\n".join(texts)

        elif self.engine == "tesseract":
            try:
                import pytesseract
                return pytesseract.image_to_string(image)
            except ImportError:
                logger.error("pytesseract not available")
                return ""

        elif self.engine == "vision_llm":
            # Fallback: use vision LLM for OCR
            # (This is actually good for complex diagrams)
            return "[OCR via vision LLM — configured in VisionExtractor]"

        return ""


# ─── Vision Description Generator ───────────────────────────────────────────

class VisionDescriptionGenerator:
    """
    Generate rich descriptions of images using vision-capable LLMs.

    This is Strategy 1 on steroids: instead of just extracting text,
    we describe the ENTIRE image in natural language, making it
    searchable via text embeddings.
    """

    IMAGE_DESCRIPTION_PROMPT = """Describe this image in detail for a RAG system.

Focus on:
1. What TYPE of image is it? (diagram, screenshot, chart, photograph, illustration)
2. What TEXT appears in the image? (extract ALL text exactly)
3. What is the STRUCTURE? (layout, relationships between elements, hierarchy)
4. What DATA or information does it convey?
5. What CONTEXT would help someone understand this image?

Be exhaustive. Another AI system will use this description to determine
if this image is relevant to a user's question.

Image type: {image_type_hint}
"""

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        model: str = "gpt-4o-mini",  # Vision-capable model
        max_description_tokens: int = 1000,
    ):
        self.llm_client = llm_client
        self.model = model
        self.max_description_tokens = max_description_tokens

    def _encode_image(self, image: PILImage.Image) -> str:
        """Convert PIL Image to base64 for API."""
        buffer = io.BytesIO()
        image.save(buffer, format="PNG")
        return base64.b64encode(buffer.getvalue()).decode("utf-8")

    async def describe(
        self,
        image: PILImage.Image,
        image_type_hint: str = "general",
        surrounding_text: str = "",
    ) -> Dict[str, str]:
        """
        Generate a comprehensive description of the image.

        Returns:
            Dict with description, ocr_text, and analysis fields
        """
        base64_image = self._encode_image(image)
        content_blocks = [
            {
                "type": "text",
                "text": self.IMAGE_DESCRIPTION_PROMPT.format(
                    image_type_hint=image_type_hint
                )
            },
            {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/png;base64,{base64_image}",
                    "detail": "high"
                }
            }
        ]

        if surrounding_text:
            content_blocks.insert(1, {
                "type": "text",
                "text": f"Surrounding document context:\n{surrounding_text[:500]}"
            })

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": "You are an image analyzer for a RAG system. Extract ALL information."
                },
                {
                    "role": "user",
                    "content": content_blocks
                }
            ],
            max_tokens=self.max_description_tokens,
            temperature=0.3,
        )

        description = response.choices[0].message.content

        # Parse into structured fields (best-effort)
        result = {
            "full_description": description,
            "ocr_text": self._extract_ocr_section(description),
            "image_type": self._extract_image_type(description),
            "structure": self._extract_structure(description),
        }

        return result

    def _extract_ocr_section(self, description: str) -> str:
        """Extract OCR-like text from the description if present."""
        import re
        # Look for text extraction sections
        patterns = [
            r"(?:text|Text).*?(?:appears|reads|says|shows):\s*(.+?)(?:\n\n|\Z)",
            r"(?:extract|Extract).*?(?:text|all text).*?:\s*(.+?)(?:\n\n|\Z)",
        ]
        for pattern in patterns:
            match = re.search(pattern, description, re.DOTALL)
            if match:
                return match.group(1).strip()
        return ""

    def _extract_image_type(self, description: str) -> str:
        """Extract image type classification."""
        types = ["diagram", "screenshot", "chart", "graph", "photograph",
                 "illustration", "table", "infographic", "map", "screen capture"]
        desc_lower = description.lower()
        for t in types:
            if t in desc_lower:
                return t
        return "unknown"

    def _extract_structure(self, description: str) -> str:
        """Extract structural description."""
        import re
        patterns = [
            r"(?:structure|Structure|layout|Layout).*?:\s*(.+?)(?:\n\n|\Z)",
            r"(?:relationship|Relationship|hierarchy|Hierarchy).*?:\s*(.+?)(?:\n\n|\Z)",
        ]
        for pattern in patterns:
            match = re.search(pattern, description, re.DOTALL)
            if match:
                return match.group(1).strip()
        return ""


# ─── Image Preprocessing Pipeline ───────────────────────────────────────────

class ImagePreprocessingPipeline:
    """
    Complete image preprocessing pipeline for RAG indexing.

    For each image:
    1. Load and validate the image
    2. Extract OCR text
    3. Generate vision description
    4. Combine into searchable text representations
    """

    def __init__(
        self,
        ocr_extractor: Optional[OCRExtractor] = None,
        vision_descriptor: Optional[VisionDescriptionGenerator] = None,
        max_image_size: Tuple[int, int] = (2048, 2048),
        supported_formats: Tuple[str, ...] = (".png", ".jpg", ".jpeg", ".webp", ".gif", ".bmp"),
    ):
        self.ocr = ocr_extractor or OCRExtractor()
        self.vision = vision_descriptor
        self.max_image_size = max_image_size
        self.supported_formats = supported_formats

    def is_supported(self, path: str) -> bool:
        """Check if a file is a supported image format."""
        return any(path.lower().endswith(fmt) for fmt in self.supported_formats)

    def load_and_resize(self, image_path: str) -> Optional[PILImage.Image]:
        """
        Load an image and resize if needed to stay within max dimensions.
        """
        try:
            img = PILImage.open(image_path)

            # Convert to RGB if needed (for RGBA PNGs)
            if img.mode == "RGBA":
                background = PILImage.new("RGB", img.size, (255, 255, 255))
                background.paste(img, mask=img.split()[3])
                img = background
            elif img.mode != "RGB":
                img = img.convert("RGB")

            # Resize if too large (preserving aspect ratio)
            max_w, max_h = self.max_image_size
            w, h = img.size
            if w > max_w or h > max_h:
                ratio = min(max_w / w, max_h / h)
                new_w = int(w * ratio)
                new_h = int(h * ratio)
                img = img.resize((new_w, new_h), PILImage.LANCZOS)
                logger.info(f"Resized {image_path} from {w}x{h} to {new_w}x{new_h}")

            return img

        except Exception as e:
            logger.error(f"Failed to load image {image_path}: {e}")
            return None

    async def process_image(
        self,
        image_path: str,
        alt_text: str = "",
        caption: str = "",
        surrounding_text: str = "",
        document_source: str = "",
        page_number: Optional[int] = None,
        figure_number: Optional[str] = None,
        generate_vision_description: bool = True,
    ) -> Optional[ImageContent]:
        """
        Process a single image for RAG indexing.

        Args:
            image_path: Path to the image file
            alt_text: Alt text from the document (if any)
            caption: Figure caption from the document (if any)
            surrounding_text: Text surrounding the image reference
            document_source: Source document identifier
            page_number: Page number (for PDFs)
            figure_number: Figure number in the document
            generate_vision_description: Whether to use vision LLM for description

        Returns:
            ImageContent with all extracted information, or None if failed
        """
        # Load image
        img = self.load_and_resize(image_path)
        if img is None:
            return None

        # Gather extracted content
        content = ImageContent(
            image_path=image_path,
            image_format=Path(image_path).suffix.lower(),
            width=img.width,
            height=img.height,
            alt_text=alt_text,
            caption=caption,
            surrounding_text=surrounding_text,
            document_source=document_source,
            page_number=page_number,
            figure_number=figure_number,
        )

        # Step 1: OCR extraction
        try:
            ocr_result = await self.ocr.extract(img)
            content.ocr_text = ocr_result
            logger.info(f"OCR extracted {len(ocr_result)} chars from {image_path}")
        except Exception as e:
            logger.warning(f"OCR failed for {image_path}: {e}")

        # Step 2: Vision description (if configured)
        if generate_vision_description and self.vision:
            try:
                desc_result = await self.vision.describe(
                    img,
                    image_type_hint=self._guess_image_type(alt_text, caption),
                    surrounding_text=surrounding_text,
                )
                content.llm_description = desc_result.get("full_description", "")

                # If OCR was empty but vision found text, use it
                if not content.ocr_text and desc_result.get("ocr_text"):
                    content.ocr_text = desc_result["ocr_text"]

                logger.info(f"Vision description generated ({len(content.llm_description)} chars)")
            except Exception as e:
                logger.warning(f"Vision description failed for {image_path}: {e}")

        return content

    def _guess_image_type(self, alt_text: str, caption: str) -> str:
        """Guess image type from alt text and caption."""
        combined = (alt_text + " " + caption).lower()
        if any(w in combined for w in ["diagram", "schematic", "architecture"]):
            return "diagram"
        elif any(w in combined for w in ["screenshot", "screen capture", "ui"]):
            return "screenshot"
        elif any(w in combined for w in ["chart", "graph", "plot"]):
            return "chart"
        elif any(w in combined for w in ["photo", "photograph", "image of"]):
            return "photograph"
        return "general"

    async def process_batch(
        self,
        image_paths: List[str],
        metadata_list: Optional[List[Dict]] = None,
        batch_size: int = 5,
    ) -> List[ImageContent]:
        """
        Process multiple images in parallel batches.

        Args:
            image_paths: List of image file paths
            metadata_list: Optional list of metadata dicts per image
            batch_size: Number of concurrent processing tasks

        Returns:
            List of ImageContent results
        """
        results = []
        metadata_list = metadata_list or [{} for _ in image_paths]

        for i in range(0, len(image_paths), batch_size):
            batch_paths = image_paths[i:i + batch_size]
            batch_meta = metadata_list[i:i + batch_size]

            tasks = []
            for path, meta in zip(batch_paths, batch_meta):
                tasks.append(self.process_image(
                    path,
                    alt_text=meta.get("alt_text", ""),
                    caption=meta.get("caption", ""),
                    surrounding_text=meta.get("surrounding_text", ""),
                    document_source=meta.get("document_source", ""),
                    page_number=meta.get("page_number"),
                    figure_number=meta.get("figure_number"),
                ))

            batch_results = await asyncio.gather(*tasks)
            results.extend([r for r in batch_results if r is not None])

            logger.info(f"Processed batch {i//batch_size + 1}: "
                        f"{len(batch_results)} images, "
                        f"{sum(1 for r in batch_results if r is not None)} succeeded")

        return results
```

---

### 2. CLIP-Based Multimodal Embeddings

```python
"""
clip_embeddings.py — Multimodal embeddings using CLIP.

This implements Strategy 2: embedding both text and images into
a shared vector space for cross-modal retrieval.

Prerequisites:
    pip install torch transformers pillow
"""

from typing import List, Optional, Dict, Any, Union, Tuple
from dataclasses import dataclass, field
from pathlib import Path
import numpy as np
import asyncio
import logging

from PIL import Image as PILImage

logger = logging.getLogger(__name__)


# ─── CLIP Embedder ──────────────────────────────────────────────────────────

class CLIPEmbedder:
    """
    Generate multimodal embeddings using CLIP-like models.

    Both text and images are embedded into the SAME vector space,
    enabling cross-modal search (text query → image results).

    Supports:
    - OpenAI CLIP (ViT-B/32, ViT-L/14)
    - OpenCLIP variants
    - SigLIP (Google)
    - Domain-specific (BiomedCLIP, etc.)
    """

    MODEL_INFO = {
        "ViT-B-32": {
            "embedding_dim": 512,
            "description": "Best speed/quality balance",
            "openclip_name": "ViT-B-32",
            "openclip_pretrained": "laion2b_s34b_b79k",
        },
        "ViT-L-14": {
            "embedding_dim": 768,
            "description": "Best quality, slower",
            "openclip_name": "ViT-L-14",
            "openclip_pretrained": "laion2b_s32b_b82k",
        },
        "ViT-B-16-SigLIP": {
            "embedding_dim": 768,
            "description": "Google SigLIP, good for multilingual",
            "openclip_name": "ViT-B-16-SigLIP",
            "openclip_pretrained": "webli",
        },
    }

    def __init__(
        self,
        model_name: str = "ViT-B-32",
        device: str = "cpu",
        use_openclip: bool = False,
    ):
        self.model_name = model_name
        self.device = device
        self.use_openclip = use_openclip
        self._model = None
        self._processor = None
        self._tokenizer = None

    def _load_model(self):
        """Load CLIP model (lazy initialization)."""
        if self._model is not None:
            return

        try:
            if self.use_openclip:
                self._load_openclip()
            else:
                self._load_openai_clip()
            logger.info(f"Loaded CLIP model {self.model_name} on {self.device}")
        except Exception as e:
            logger.error(f"Failed to load CLIP model: {e}")
            raise

    def _load_openai_clip(self):
        """Load original OpenAI CLIP."""
        import clip
        model_name_map = {
            "ViT-B-32": "ViT-B/32",
            "ViT-L-14": "ViT-B/16",  # Fallback if L/14 not available
        }
        clip_name = model_name_map.get(self.model_name, "ViT-B/32")
        self._model, self._processor = clip.load(clip_name, device=self.device)

    def _load_openclip(self):
        """Load OpenCLIP model (supports more architectures)."""
        import open_clip

        info = self.MODEL_INFO.get(self.model_name, self.MODEL_INFO["ViT-B-32"])
        self._model, _, self._processor = open_clip.create_model_and_transforms(
            info["openclip_name"],
            pretrained=info["openclip_pretrained"],
            device=self.device,
        )
        self._tokenizer = open_clip.get_tokenizer(info["openclip_name"])

    def embed_text(self, text: str) -> List[float]:
        """
        Embed a text string into CLIP space.

        Returns:
            Normalized embedding vector.
        """
        self._load_model()

        if self.use_openclip and self._tokenizer:
            tokens = self._tokenizer([text])
            import torch
            with torch.no_grad():
                features = self._model.encode_text(tokens.to(self.device))
                features = features / features.norm(dim=-1, keepdim=True)
            return features[0].cpu().numpy().tolist()

        else:
            # OpenAI CLIP
            import torch
            tokens = self._processor([text])
            with torch.no_grad():
                features = self._model.encode_text(tokens.to(self.device))
                features = features / features.norm(dim=-1, keepdim=True)
            return features[0].cpu().numpy().tolist()

    def embed_image(self, image: PILImage.Image) -> List[float]:
        """
        Embed a PIL Image into CLIP space.

        Returns:
            Normalized embedding vector.
        """
        self._load_model()

        if self.use_openclip and self._processor:
            import torch
            image_tensor = self._processor(image).unsqueeze(0).to(self.device)
            with torch.no_grad():
                features = self._model.encode_image(image_tensor)
                features = features / features.norm(dim=-1, keepdim=True)
            return features[0].cpu().numpy().tolist()

        else:
            import torch
            image_input = self._processor(image).unsqueeze(0).to(self.device)
            with torch.no_grad():
                features = self._model.encode_image(image_input)
                features = features / features.norm(dim=-1, keepdim=True)
            return features[0].cpu().numpy().tolist()

    def embed_image_file(self, image_path: str) -> Optional[List[float]]:
        """Embed an image from a file path."""
        try:
            img = PILImage.open(image_path).convert("RGB")
            return self.embed_image(img)
        except Exception as e:
            logger.error(f"Failed to embed image {image_path}: {e}")
            return None

    def embed_text_batch(self, texts: List[str], batch_size: int = 32) -> List[List[float]]:
        """Embed multiple texts efficiently."""
        self._load_model()
        import torch

        all_embeddings = []
        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]

            if self.use_openclip and self._tokenizer:
                tokens = self._tokenizer(batch).to(self.device)
                with torch.no_grad():
                    features = self._model.encode_text(tokens)
                    features = features / features.norm(dim=-1, keepdim=True)
            else:
                tokens = self._processor(batch).to(self.device)
                with torch.no_grad():
                    features = self._model.encode_text(tokens)
                    features = features / features.norm(dim=-1, keepdim=True)

            all_embeddings.extend(features.cpu().numpy().tolist())

        return all_embeddings

    def compute_similarity(
        self, embedding_a: List[float], embedding_b: List[float]
    ) -> float:
        """Compute cosine similarity between two embeddings."""
        a = np.array(embedding_a)
        b = np.array(embedding_b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b) + 1e-10))


# ─── Image Index Builder ────────────────────────────────────────────────────

class MultimodalIndexBuilder:
    """
    Build a unified multimodal index from text chunks and images.

    Each entry in the index contains:
    - Text embedding (from chunk text)
    - Image embedding (from CLIP, if it's an image)
    - Metadata for filtering

    This enables searching across text AND images with a single query.
    """

    def __init__(
        self,
        clip_embedder: CLIPEmbedder,
        text_embedder: Optional[callable] = None,  # Separate text embedder (optional)
    ):
        self.clip = clip_embedder
        self.text_embedder = text_embedder  # e.g., OpenAI text-embedding-3-small

    async def build_image_entry(
        self,
        image: ImageContent,
        store_image_data: bool = False,
    ) -> ImageIndexEntry:
        """
        Build a searchable index entry for an image.

        Creates MULTIPLE text representations of the image so it can
        be found by text search even without CLIP.
        """
        # Build searchable text representations
        searchable = {
            "ocr": image.ocr_text,
            "alt": image.alt_text,
            "caption": image.caption,
            "description": image.llm_description,
            "surrounding": image.surrounding_text,
        }

        # Combined text for embedding
        combined_text = "\n".join(
            v for v in searchable.values() if v.strip()
        )

        # Create CLIP embedding from the actual image
        clip_emb = None
        if image.clip_embedding:
            clip_emb = image.clip_embedding
        else:
            try:
                img = PILImage.open(image.image_path).convert("RGB")
                clip_emb = self.clip.embed_image(img)
            except Exception as e:
                logger.warning(f"CLIP embedding failed for {image.image_path}: {e}")

        # Also embed the combined text (for text-query matching)
        text_emb = None
        if self.text_embedder:
            text_emb = await self.text_embedder(combined_text)
        else:
            text_emb = self.clip.embed_text(combined_text)

        image_id = f"img_{Path(image.image_path).stem}_{hash(image.image_path) % 100000}"

        return ImageIndexEntry(
            image_id=image_id,
            image_path=image.image_path,
            searchable_texts=searchable,
            embedding=clip_emb,  # CLIP embedding (for cross-modal search)
            metadata={
                "type": "image",
                "format": image.image_format,
                "width": image.width,
                "height": image.height,
                "document_source": image.document_source,
                "page_number": image.page_number,
                "figure_number": image.figure_number,
                "ocr_text": image.ocr_text[:200] if image.ocr_text else "",
                "clip_embedding_available": clip_emb is not None,
                "text_embedding_available": text_emb is not None,
            }
        )

    async def build_index(
        self,
        images: List[ImageContent],
        text_chunks: Optional[List[Dict]] = None,
    ) -> tuple:
        """
        Build a complete multimodal index.

        Returns: (image_entries, text_entries, combined_index)
        """
        # Build image entries
        image_entries = []
        for img in images:
            entry = await self.build_image_entry(img)
            image_entries.append(entry)

        # Build text entries (if needed)
        text_entries = text_chunks or []

        # Combined index: both text and images
        combined_index = {
            "images": image_entries,
            "texts": text_entries,
            "total_entries": len(image_entries) + len(text_entries),
            "embedding_dim": self.clip.MODEL_INFO.get(
                self.clip.model_name, {}
            ).get("embedding_dim", 512),
        }

        return image_entries, text_entries, combined_index


# ─── Usage: Index Your Images ────────────────────────────────────────────────

async def index_images_example():
    """Example: index a directory of images into a multimodal index."""
    import os

    # Initialize components
    ocr = OCRExtractor(engine="easyocr")
    vision = VisionDescriptionGenerator(
        llm_client=AsyncOpenAI(),
        model="gpt-4o-mini",
    )
    pipeline = ImagePreprocessingPipeline(
        ocr_extractor=ocr,
        vision_descriptor=vision,
    )

    clip = CLIPEmbedder(model_name="ViT-B-32", device="cpu")
    index_builder = MultimodalIndexBuilder(clip_embedder=clip)

    # Find all images in a directory
    image_dir = "docs/figures/"
    image_paths = [
        os.path.join(image_dir, f)
        for f in os.listdir(image_dir)
        if pipeline.is_supported(f)
    ]

    print(f"Found {len(image_paths)} images to process")

    # Process images (batch of 3 at a time)
    image_contents = await pipeline.process_batch(
        image_paths[:3],  # First 3 for demo
        batch_size=3,
    )

    print(f"Processed {len(image_contents)} images")

    # Build index
    img_entries, _, combined = await index_builder.build_index(image_contents)

    for entry in img_entries:
        print(f"\n  [{entry.image_id}] {entry.image_path}")
        print(f"  Searchable texts:")
        for key, val in entry.searchable_texts.items():
            if val:
                print(f"    {key}: {val[:100]}...")
        print(f"  Embedding dim: {len(entry.embedding) if entry.embedding else 0}")
```

---

### 3. Unified Multimodal Retriever

```python
"""
multimodal_retriever.py — Search across text AND images with a single query.

This is the core retrieval engine that combines:
- Text embeddings (for text-only queries)
- CLIP embeddings (for cross-modal text↔image queries)
- Metadata filters (for structured search)

The key insight: a single query is embedded into the SHARED space
and compared against BOTH text and image embeddings simultaneously.
"""

from typing import List, Optional, Dict, Any, Union, Callable, Literal
from dataclasses import dataclass, field
import numpy as np
import asyncio
import logging

logger = logging.getLogger(__name__)


# ─── Data Models ────────────────────────────────────────────────────────────

@dataclass
class MultimodalSearchResult:
    """A single search result from the multimodal index."""
    id: str
    score: float
    source_type: Literal["text", "image"]
    text: str  # The text content (or image description)
    image_path: Optional[str] = None
    metadata: Dict[str, Any] = field(default_factory=dict)


@dataclass
class MultimodalQueryResult:
    """Complete multimodal search results."""
    query: str
    text_results: List[MultimodalSearchResult]
    image_results: List[MultimodalSearchResult]
    combined_results: List[MultimodalSearchResult]
    query_embedding: Optional[List[float]] = None
    total_latency_ms: float = 0.0


# ─── Vector Store Abstraction ───────────────────────────────────────────────

class SimpleVectorStore:
    """
    Minimal in-memory vector store for multimodal search.

    In production, replace with ChromaDB, Qdrant, or Pinecone.
    This is kept minimal to focus on the multimodal logic.
    """

    def __init__(self):
        self.entries: List[Dict] = []
        self.embeddings: np.ndarray = np.array([])
        self.id_to_entry: Dict[str, int] = {}
        self._is_built = False

    def add_entries(self, entries: List[Dict], embeddings: List[List[float]]):
        """Add entries with their embeddings."""
        start_idx = len(self.entries)
        for i, (entry, emb) in enumerate(zip(entries, embeddings)):
            eid = entry.get("id", f"entry_{start_idx + i}")
            entry["id"] = eid
            self.id_to_entry[eid] = start_idx + i
            self.entries.append(entry)

        new_embeddings = np.array(embeddings)
        if self._is_built:
            self.embeddings = np.vstack([self.embeddings, new_embeddings])
        else:
            self.embeddings = new_embeddings
            self._is_built = True

    def search(
        self,
        query_embedding: List[float],
        top_k: int = 10,
        filter_fn: Optional[Callable] = None,
    ) -> List[Dict]:
        """Search by cosine similarity."""
        if not self._is_built or len(self.entries) == 0:
            return []

        query_vec = np.array(query_embedding).reshape(1, -1)
        norms = np.linalg.norm(self.embeddings, axis=1, keepdims=True)
        query_norm = np.linalg.norm(query_vec)

        if query_norm == 0 or np.any(norms == 0):
            return []

        similarities = (self.embeddings @ query_vec.T).flatten() / (norms.flatten() * query_norm)

        # Get top-k indices
        top_indices = np.argsort(similarities)[::-1][:top_k * 2]

        results = []
        for idx in top_indices:
            if filter_fn and not filter_fn(self.entries[idx]):
                continue
            results.append({
                **self.entries[idx],
                "score": float(similarities[idx]),
            })
            if len(results) >= top_k:
                break

        return results

    def count(self) -> int:
        return len(self.entries)


# ─── Multimodal Retriever ───────────────────────────────────────────────────

class MultimodalRetriever:
    """
    Unified retriever that searches text and images simultaneously.

    Usage:
        retriever = MultimodalRetriever(
            text_embed_fn=openai_embed,
            clip_embed_fn=clip_embed,
        )
        retriever.add_text_chunks(chunks, text_embeddings)
        retriever.add_images(image_entries, clip_embeddings)

        results = await retriever.search("What connects to the message bus?")
        # Returns BOTH text and image results, merged by relevance
    """

    def __init__(
        self,
        text_embed_fn: Optional[Callable] = None,  # e.g., text-embedding-3-small
        clip_embed_fn: Optional[Callable] = None,   # e.g., CLIP embedder
        text_weight: float = 0.4,    # Weight for text-style retrieval
        image_weight: float = 0.6,   # Weight for image-style retrieval
    ):
        self.text_embed_fn = text_embed_fn
        self.clip_embed_fn = clip_embed_fn
        self.text_weight = text_weight
        self.image_weight = image_weight

        # Separate stores for text and image entries
        self.text_store = SimpleVectorStore()
        self.image_store = SimpleVectorStore()

    def add_text_chunks(
        self,
        chunks: List[Dict[str, Any]],
        embeddings: List[List[float]],
    ):
        """Add text chunks with their embeddings."""
        entries = []
        for chunk in chunks:
            entries.append({
                "id": chunk.get("id", f"txt_{hash(str(chunk))}"),
                "type": "text",
                "text": chunk.get("text", chunk.get("content", "")),
                "metadata": chunk.get("metadata", {}),
                "source": chunk.get("source", ""),
            })
        self.text_store.add_entries(entries, embeddings)
        logger.info(f"Added {len(entries)} text entries")

    def add_images(
        self,
        image_entries: List[ImageIndexEntry],
        use_clip_embeddings: bool = True,
    ):
        """Add image entries to the index."""
        entries = []
        embeddings = []

        for ie in image_entries:
            combined_desc = "\n".join(
                v for v in ie.searchable_texts.values() if v.strip()
            )
            entries.append({
                "id": ie.image_id,
                "type": "image",
                "text": combined_desc,
                "image_path": ie.image_path,
                "metadata": ie.metadata,
                "searchable_texts": ie.searchable_texts,
            })

            if use_clip_embeddings and ie.embedding:
                embeddings.append(ie.embedding)
            else:
                # Fallback: use zero vector (will not match well)
                embeddings.append([0.0] * 512)
                logger.warning(f"No CLIP embedding for {ie.image_id}")

        self.image_store.add_entries(entries, embeddings)
        logger.info(f"Added {len(entries)} image entries")

    async def _embed_query(self, query: str) -> Dict[str, List[float]]:
        """Embed the query for all available modalities."""
        embeddings = {}

        if self.text_embed_fn:
            embeddings["text"] = await self.text_embed_fn(query)
        elif self.clip_embed_fn:
            embeddings["text"] = self.clip_embed_fn.embed_text(query)

        if self.clip_embed_fn:
            embeddings["clip"] = self.clip_embed_fn.embed_text(query)

        return embeddings

    async def search(
        self,
        query: str,
        top_k_text: int = 5,
        top_k_images: int = 5,
        top_k_combined: int = 10,
        filter_type: Optional[Literal["text", "image", "all"]] = "all",
    ) -> MultimodalQueryResult:
        """
        Search across both text and image indices.

        The query is embedded once (or twice) and compared against
        all entries in both stores.
        """
        import time
        start = time.time()

        # Embed the query
        embeddings = await self._embed_query(query)

        # Search text store
        text_results = []
        if filter_type in ("text", "all") and "text" in embeddings:
            raw_text = self.text_store.search(
                embeddings["text"],
                top_k=top_k_text,
            )
            text_results = [
                MultimodalSearchResult(
                    id=r["id"],
                    score=r["score"],
                    source_type="text",
                    text=r["text"],
                    metadata=r.get("metadata", {}),
                )
                for r in raw_text
            ]

        # Search image store
        image_results = []
        if filter_type in ("image", "all") and "clip" in embeddings:
            raw_images = self.image_store.search(
                embeddings["clip"],  # CLIP embedding for cross-modal
                top_k=top_k_images,
            )
            image_results = [
                MultimodalSearchResult(
                    id=r["id"],
                    score=r["score"],
                    source_type="image",
                    text=r.get("text", ""),
                    image_path=r.get("image_path"),
                    metadata=r.get("metadata", {}),
                )
                for r in raw_images
            ]

        # Merge and rerank
        combined = self._merge_results(
            text_results, image_results, top_k_combined
        )

        latency = (time.time() - start) * 1000

        return MultimodalQueryResult(
            query=query,
            text_results=text_results,
            image_results=image_results,
            combined_results=combined,
            query_embedding=embeddings.get("text") or embeddings.get("clip"),
            total_latency_ms=latency,
        )

    def _merge_results(
        self,
        text_results: List[MultimodalSearchResult],
        image_results: List[MultimodalSearchResult],
        top_k: int,
    ) -> List[MultimodalSearchResult]:
        """
        Merge text and image results with modality weighting.

        Text and image scores come from different embedding models
        and can't be directly compared. We normalize and weight them.
        """
        if not text_results and not image_results:
            return []

        # Normalize scores within each modality
        def normalize(results, weight):
            if not results:
                return []
            scores = [r.score for r in results]
            max_score = max(scores) if scores else 1.0
            min_score = min(scores) if scores else 0.0
            range_score = max_score - min_score if max_score != min_score else 1.0

            normalized = []
            for r in results:
                normalized_score = ((r.score - min_score) / range_score) * weight
                normalized.append(MultimodalSearchResult(
                    id=r.id,
                    score=normalized_score,
                    source_type=r.source_type,
                    text=r.text,
                    image_path=r.image_path,
                    metadata=r.metadata,
                ))
            return normalized

        text_normalized = normalize(text_results, self.text_weight)
        image_normalized = normalize(image_results, self.image_weight)

        # Combine and sort
        merged = sorted(
            text_normalized + image_normalized,
            key=lambda x: x.score,
            reverse=True,
        )

        return merged[:top_k]

    async def search_text_only(self, query: str, top_k: int = 5) -> MultimodalQueryResult:
        """Search text index only (baseline for comparison)."""
        return await self.search(
            query,
            top_k_text=top_k,
            top_k_images=0,
            filter_type="text"
        )

    async def search_image_only(self, query: str, top_k: int = 5) -> MultimodalQueryResult:
        """Search image index only."""
        return await self.search(
            query,
            top_k_text=0,
            top_k_images=top_k,
            filter_type="image"
        )


# ─── Usage Example ──────────────────────────────────────────────────────────

async def multimodal_search_demo():
    """Demonstrate searching across text and images."""
    from openai import AsyncOpenAI

    openai_client = AsyncOpenAI()

    async def text_embed(text: str) -> List[float]:
        """OpenAI text embedding."""
        resp = await openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=text,
        )
        return resp.data[0].embedding

    # Initialize CLIP (small model for demo)
    clip = CLIPEmbedder(model_name="ViT-B-32", device="cpu")

    # Build retriever
    retriever = MultimodalRetriever(
        text_embed_fn=text_embed,
        clip_embed_fn=clip,
        text_weight=0.4,
        image_weight=0.6,
    )

    # Add text chunks
    text_chunks = [
        {"id": "txt_1", "text": "The system architecture uses a message bus for inter-service communication. Services publish events to the bus and consume events from subscribed topics.", "source": "architecture-guide"},
        {"id": "txt_2", "text": "The message bus is implemented using Apache Kafka with 3 partitions for fault tolerance.", "source": "architecture-guide"},
        {"id": "txt_3", "text": "Database replication follows a primary-replica pattern with one primary in us-east and two replicas in eu-west and ap-southeast.", "source": "ops-manual"},
        {"id": "txt_ref", "text": "Figure 3 shows the replication topology diagram. See the diagram for node relationships.", "source": "ops-manual"},
    ]

    txt_embs = []
    for chunk in text_chunks:
        emb = await text_embed(chunk["text"])
        txt_embs.append(emb)
    retriever.add_text_chunks(text_chunks, txt_embs)

    # Create mock image entries
    mock_images = [
        ImageIndexEntry(
            image_id="img_diagram_1",
            image_path="docs/figures/replication-topology.png",
            searchable_texts={
                "ocr": "Primary → Replica 1 (eu-west) → Replica 2 (us-east)",
                "description": "Database replication topology diagram showing a primary node in us-east connected to two replica nodes in eu-west and ap-southeast regions. Arrows indicate replication direction from primary to replicas.",
                "surrounding": "Figure 3: Database replication topology",
            },
            metadata={
                "type": "image",
                "figure_number": "3",
                "document_source": "ops-manual",
            }
        ),
        ImageIndexEntry(
            image_id="img_chart_1",
            image_path="docs/figures/q1-revenue.png",
            searchable_texts={
                "ocr": "Q1 2025 Revenue ($M): Jan: 12.4, Feb: 8.7, Mar: 11.2",
                "description": "Q1 2025 revenue chart showing monthly revenue in millions. January: $12.4M, February: $8.7M (lowest), March: $11.2M (recovery).",
                "surrounding": "Q1 2025 financial performance",
            },
            metadata={
                "type": "image",
                "figure_number": "1",
                "document_source": "financial-report",
            }
        ),
    ]

    # Mock CLIP embeddings for images (in production, these would be real CLIP embeddings)
    for img_entry in mock_images:
        combined = "\n".join(v for v in img_entry.searchable_texts.values() if v)
        img_entry.embedding = clip.embed_text(combined)

    retriever.add_images(mock_images)

    # Test queries
    test_queries = [
        "What services communicate through the message bus?",
        "Show me the replication topology diagram",
        "What was the revenue trend in Q1 2025?",
        "Which month had the lowest Q1 revenue?",
    ]

    for query in test_queries:
        print(f"\n{'='*60}")
        print(f"QUERY: {query}")
        print(f"{'='*60}")

        result = await retriever.search(query)

        print(f"\nText results ({len(result.text_results)}):")
        for r in result.text_results[:3]:
            print(f"  [{r.score:.3f}] {r.text[:100]}...")

        print(f"\nImage results ({len(result.image_results)}):")
        for r in result.image_results[:3]:
            print(f"  [{r.score:.3f}] [{r.id}] {r.text[:100]}...")

        print(f"\nCombined results ({len(result.combined_results)}):")
        for r in result.combined_results[:5]:
            src = "📄" if r.source_type == "text" else "🖼️"
            print(f"  {src} [{r.score:.3f}] ({r.source_type}) {r.text[:80]}...")
```

---

### 4. Vision LLM RAG Pipeline

The most powerful approach: use a vision-capable LLM at QUERY TIME to read images directly:

```python
"""
vision_rag.py — RAG with vision LLM for image understanding.

When text retrieval finds image references, this pipeline loads
the actual images and passes them to a vision model for understanding.

This is Strategy 3: the most expensive per query, but the only
way to answer questions that require actual visual understanding.
"""

from typing import List, Optional, Dict, Any, Tuple
from dataclasses import dataclass, field
from pathlib import Path
import base64
import io
import json
import time
import asyncio

from openai import AsyncOpenAI
from PIL import Image as PILImage


# ─── Data Models ────────────────────────────────────────────────────────────

@dataclass
class VisionContext:
    """Context prepared for a vision LLM call."""
    text_context: str
    images: List[Dict[str, Any]]  # [{"path": str, "base64": str, "description": str}]
    metadata: Dict[str, Any] = field(default_factory=dict)

    @property
    def image_count(self) -> int:
        return len(self.images)

    @property
    def total_size_bytes(self) -> int:
        return sum(len(img.get("base64", "")) for img in self.images) * 3 // 4


@dataclass
class VisionAnswer:
    """Answer generated by a vision LLM with image evidence."""
    query: str
    answer: str
    images_used: List[str]
    text_chunks_used: List[Dict]
    vision_latency_ms: float
    total_latency_ms: float
    model: str
    cost_estimate: float


# ─── Vision Context Builder ─────────────────────────────────────────────────

class VisionContextBuilder:
    """
    Build vision LLM context from retrieved text chunks and image references.

    Pipeline:
    1. Retrieve text chunks (standard RAG)
    2. Find image references in chunks
    3. Resolve image references to actual files
    4. Load and encode images
    5. Build multimodal context for vision LLM
    """

    def __init__(
        self,
        image_base_path: str = "docs/",
        max_image_size: Tuple[int, int] = (2048, 2048),
        max_images_per_query: int = 4,
    ):
        self.image_base_path = image_base_path
        self.max_image_size = max_image_size
        self.max_images_per_query = max_images_per_query

    def find_image_references(
        self,
        chunks: List[Dict[str, Any]],
    ) -> List[Tuple[Dict, str]]:
        """
        Find image references in retrieved text chunks.

        Returns list of (chunk, image_path_or_url) tuples.
        """
        references = []

        for chunk in chunks:
            text = chunk.get("text", chunk.get("content", ""))

            # Pattern 1: Markdown images: ![alt](path)
            import re
            md_images = re.findall(r'!\[.*?\]\(([^)]+)\)', text)
            for img_path in md_images:
                resolved = self._resolve_path(img_path, chunk)
                references.append((chunk, resolved))

            # Pattern 2: HTML images: <img src="path">
            html_images = re.findall(r'<img[^>]+src="([^"]+)"', text)
            for img_path in html_images:
                resolved = self._resolve_path(img_path, chunk)
                references.append((chunk, resolved))

            # Pattern 3: Figure references: "See Figure 3" or "Figure 3 shows"
            # These need a lookup table (figure_number → image_path)
            figure_refs = re.findall(
                r'(?:Figure|Fig\.|figure)\s*(\d+(?:\.\d+)?)',
                text
            )
            # Note: actual figure resolution requires a figure→path mapping
            # provided at construction time or in chunk metadata

        return references

    def _resolve_path(self, img_path: str, chunk: Dict) -> str:
        """Resolve a relative image path to an absolute path."""
        # If it's already absolute
        if Path(img_path).is_absolute():
            return img_path

        # Try relative to image_base_path
        full_path = str(Path(self.image_base_path) / img_path)
        if Path(full_path).exists():
            return full_path

        # Try relative to the chunk's source document
        source = chunk.get("source", chunk.get("metadata", {}).get("source", ""))
        if source:
            source_dir = str(Path(source).parent)
            alt_path = str(Path(source_dir) / img_path)
            if Path(alt_path).exists():
                return alt_path

        # Try as-is
        return img_path

    def load_and_encode_image(
        self,
        image_path: str,
    ) -> Optional[Dict[str, Any]]:
        """
        Load an image and encode it as base64 for API calls.

        Returns dict with path, base64 data, and metadata, or None.
        """
        try:
            # Check if it's a URL
            if image_path.startswith(("http://", "https://")):
                return {
                    "path": image_path,
                    "url": image_path,
                    "is_url": True,
                }

            # Local file
            if not Path(image_path).exists():
                logger.warning(f"Image not found: {image_path}")
                return None

            img = PILImage.open(image_path)

            # Convert to RGB
            if img.mode == "RGBA":
                background = PILImage.new("RGB", img.size, (255, 255, 255))
                background.paste(img, mask=img.split()[3])
                img = background
            elif img.mode != "RGB":
                img = img.convert("RGB")

            # Resize if needed
            max_w, max_h = self.max_image_size
            w, h = img.size
            if w > max_w or h > max_h:
                ratio = min(max_w / w, max_h / h)
                img = img.resize(
                    (int(w * ratio), int(h * ratio)),
                    PILImage.LANCZOS
                )

            # Encode as base64
            buffer = io.BytesIO()
            img.save(buffer, format="PNG", optimize=True)
            b64_data = base64.b64encode(buffer.getvalue()).decode("utf-8")

            return {
                "path": image_path,
                "base64": b64_data,
                "format": "png",
                "width": img.width,
                "height": img.height,
                "is_url": False,
            }

        except Exception as e:
            logger.error(f"Failed to load image {image_path}: {e}")
            return None

    async def build_context(
        self,
        query: str,
        text_chunks: List[Dict[str, Any]],
        figure_map: Optional[Dict[str, str]] = None,
    ) -> VisionContext:
        """
        Build a multimodal context for vision LLM.

        1. Find image references in text chunks
        2. Look up figures if figure_map provided
        3. Load and encode up to max_images_per_query images
        4. Build combined text + image context
        """
        # Step 1: Find image references in text
        references = self.find_image_references(text_chunks)

        # Step 2: Add figure map references
        if figure_map:
            texts_combined = " ".join(
                c.get("text", c.get("content", "")) for c in text_chunks
            )
            for fig_num, fig_path in figure_map.items():
                pattern = f"Figure {fig_num}"
                if pattern.lower() in texts_combined.lower():
                    references.append(({}, fig_path))

        # Step 3: Load images (deduplicate by path)
        loaded_images = []
        seen_paths = set()

        for chunk, img_path in references:
            if img_path in seen_paths:
                continue
            seen_paths.add(img_path)

            img_data = self.load_and_encode_image(img_path)
            if img_data:
                loaded_images.append(img_data)

            if len(loaded_images) >= self.max_images_per_query:
                break

        # Step 4: Build text context
        text_context = "\n\n".join(
            f"[{i+1}] {c.get('text', c.get('content', ''))[:1500]}"
            for i, c in enumerate(text_chunks[:5])
        )

        return VisionContext(
            text_context=text_context,
            images=loaded_images,
            metadata={
                "text_chunk_count": len(text_chunks),
                "image_references_found": len(references),
                "images_loaded": len(loaded_images),
                "images_skipped": len(references) - len(loaded_images),
            }
        )


# ─── Vision RAG Pipeline ────────────────────────────────────────────────────

class VisionRAGPipeline:
    """
    RAG pipeline that uses vision LLM for image understanding.

    When text-only retrieval isn't enough, this pipeline:
    1. Retrieves text chunks (standard RAG)
    2. Finds image references in those chunks
    3. Loads the actual images
    4. Passes text + images to a vision LLM
    5. Generates answer with visual understanding

    This is the MOST CAPABLE approach but also the MOST EXPENSIVE.
    Use it only when the query requires visual understanding.
    """

    VISION_SYSTEM_PROMPT = """You are analyzing documents that include images (diagrams, screenshots, charts).

The user's question may require information from BOTH the text and the images.
Look at all provided images carefully before answering.

When answering:
1. Use specific details from the images (labels, arrows, values, colors)
2. Reference image content explicitly ("The diagram shows...", "In the screenshot...")
3. If text and images conflict, note the discrepancy
4. If you cannot find the answer in either text or images, say so clearly

Cite your sources using [Text N] for text chunks and [Image: filename] for images.
"""

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        text_retriever: Callable,  # Standard text retriever
        context_builder: VisionContextBuilder,
        model: str = "gpt-4o",  # Vision-capable model
        cost_per_million_input_tokens: float = 2.50,  # GPT-4o pricing
        cost_per_million_output_tokens: float = 10.00,
    ):
        self.llm_client = llm_client
        self.text_retriever = text_retriever
        self.context_builder = context_builder
        self.model = model
        self.cost_per_million_input = cost_per_million_input_tokens
        self.cost_per_million_output = cost_per_million_output_tokens

    async def answer(
        self,
        query: str,
        top_k_text: int = 10,
        figure_map: Optional[Dict[str, str]] = None,
        requires_vision: bool = True,
    ) -> VisionAnswer:
        """
        Answer a query using vision-enhanced RAG.

        Args:
            query: User's question
            top_k_text: Number of text chunks to retrieve initially
            figure_map: Optional mapping of figure numbers to image paths
            requires_vision: If False, skip image loading (text-only baseline)

        Returns:
            VisionAnswer with full response and metadata
        """
        start = time.time()
        images_used = []
        text_chunks_used = []

        # Step 1: Text retrieval
        text_chunks = await self.text_retriever(query, top_k_text)
        text_chunks_used = text_chunks[:5]  # Keep top 5 for context

        if not requires_vision:
            # Text-only baseline
            answer = await self._generate_text_only(query, text_chunks_used)
            latency = (time.time() - start) * 1000
            return VisionAnswer(
                query=query, answer=answer, images_used=[],
                text_chunks_used=text_chunks_used,
                vision_latency_ms=0, total_latency_ms=latency,
                model=self.model, cost_estimate=0.001,
            )

        # Step 2: Find and load images from text chunks
        context = await self.context_builder.build_context(
            query, text_chunks_used, figure_map
        )
        images_used = [img.get("path", img.get("url", "unknown")) for img in context.images]

        # Step 3: Generate with vision LLM
        vision_start = time.time()
        answer = await self._generate_with_vision(query, context)
        vision_latency = (time.time() - vision_start) * 1000

        total_latency = (time.time() - start) * 1000

        # Estimate cost
        # Rough estimate: text tokens + image tokens (each ~ 170 tokens for 224x224)
        text_tokens = len(query) // 4 + sum(len(c.get("text", "")) // 4 for c in text_chunks_used)
        image_tokens = len(context.images) * 170  # Approximate
        total_input_tokens = text_tokens + image_tokens
        cost = (total_input_tokens / 1_000_000) * self.cost_per_million_input

        return VisionAnswer(
            query=query,
            answer=answer,
            images_used=images_used,
            text_chunks_used=text_chunks_used,
            vision_latency_ms=vision_latency,
            total_latency_ms=total_latency,
            model=self.model,
            cost_estimate=cost,
        )

    async def _generate_with_vision(
        self,
        query: str,
        context: VisionContext,
    ) -> str:
        """Generate an answer using vision LLM with text + image context."""
        content_blocks = [
            {
                "type": "text",
                "text": f"User question: {query}\n\n"
                        f"Relevant text from documents:\n{context.text_context}"
            }
        ]

        # Add images
        for img in context.images:
            if img.get("is_url"):
                content_blocks.append({
                    "type": "image_url",
                    "image_url": {
                        "url": img["url"],
                        "detail": "high",
                    }
                })
            else:
                content_blocks.append({
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/png;base64,{img['base64']}",
                        "detail": "high",
                    }
                })

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.VISION_SYSTEM_PROMPT},
                {"role": "user", "content": content_blocks}
            ],
            max_tokens=2000,
            temperature=0.3,
        )

        return response.choices[0].message.content

    async def _generate_text_only(
        self,
        query: str,
        text_chunks: List[Dict],
    ) -> str:
        """Generate answer from text only (baseline)."""
        context = "\n\n".join(
            c.get("text", c.get("content", ""))[:2000] for c in text_chunks
        )
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "Answer based on the provided context."},
                {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {query}"}
            ],
            max_tokens=2000,
        )
        return response.choices[0].message.content


# ─── Vision vs. Text-Only Benchmark ─────────────────────────────────────────

class VisionRAGBenchmark:
    """
    Compare vision RAG vs text-only RAG on multimodal queries.

    This is critical for understanding when vision adds value.
    """

    @dataclass
    class TestCase:
        query: str
        expected_answer_elements: List[str]  # Things the answer MUST contain
        requires_vision: bool  # Is the answer ONLY in an image?

    async def run(
        self,
        test_cases: List[TestCase],
        vision_pipeline: VisionRAGPipeline,
        text_only_pipeline: VisionRAGPipeline,  # Same but requires_vision=False
    ) -> Dict:
        """Compare vision vs text-only on test cases."""
        results = []

        for tc in test_cases:
            # Vision RAG
            vision_result = await vision_pipeline.answer(
                tc.query, requires_vision=True
            )

            # Text-only
            text_result = await text_only_pipeline.answer(
                tc.query, requires_vision=False
            )

            # Check coverage of expected elements
            vision_coverage = sum(
                1 for elem in tc.expected_answer_elements
                if elem.lower() in vision_result.answer.lower()
            ) / len(tc.expected_answer_elements)

            text_coverage = sum(
                1 for elem in tc.expected_answer_elements
                if elem.lower() in text_result.answer.lower()
            ) / len(tc.expected_answer_elements)

            results.append({
                "query": tc.query,
                "requires_vision": tc.requires_vision,
                "vision": {
                    "coverage": vision_coverage,
                    "latency_ms": vision_result.total_latency_ms,
                    "cost": vision_result.cost_estimate,
                    "images_used": len(vision_result.images_used),
                },
                "text_only": {
                    "coverage": text_coverage,
                    "latency_ms": text_result.total_latency_ms,
                    "cost": text_result.cost_estimate,
                },
                "vision_improvement": vision_coverage - text_coverage,
            })

        # Summary
        vision_cases = [r for r in results if r["requires_vision"]]
        text_only_cases = [r for r in results if not r["requires_vision"]]

        return {
            "results": results,
            "summary": {
                "avg_vision_improvement": sum(
                    r["vision_improvement"] for r in vision_cases
                ) / len(vision_cases) if vision_cases else 0,
                "vision_avg_latency": sum(
                    r["vision"]["latency_ms"] for r in results
                ) / len(results),
                "text_avg_latency": sum(
                    r["text_only"]["latency_ms"] for r in results
                ) / len(results),
                "vision_cases_improved": sum(
                    1 for r in vision_cases if r["vision_improvement"] > 0
                ),
                "text_only_cases_degraded": sum(
                    1 for r in text_only_cases if r["vision_improvement"] < 0
                ),
            }
        }
```

---

## ✅ Good Output Examples

### Strategy Selection Matrix

```
Query: "What's the API rate limit for enterprise?"
Best strategy: TEXT-ONLY RAG (standard or self-query)
Reason: All information is in text documents. No image needed.
Cost: ~$0.001/query
Latency: ~500ms

Query: "Show me the architecture diagram and explain the data flow"
Best strategy: VISION RAG (Strategy 3)
Reason: User explicitly asks about a diagram. Must view it.
Cost: ~$0.01-0.05/query
Latency: ~3-7s

Query: "Looking at Figure 3 in the migration guide, which services connect to the message bus?"
Best strategy: TEXT + IMAGE RETRIEVAL (Strategy 2 + 3)
Reason: Text finds Figure 3 reference, then vision reads the diagram.
Cost: ~$0.01-0.03/query
Latency: ~2-5s

Query: "What's the error code in this screenshot?" [user provides image]
Best strategy: VISION LLM DIRECT (no RAG needed)
Reason: The question is about the image itself, not about the document.
Cost: ~$0.01/query
Latency: ~2s
```

### Vision RAG Full Trace

```
Query: "Does the replication topology show a read replica in eu-west?"

STEP 1: Text Retrieval
  Retrieved: [
    "Figure 3 shows the replication topology diagram...",
    "The system has one primary and two replicas..."
  ]

STEP 2: Image Reference Resolution
  Found: "Figure 3" → docs/figures/replication-topology.png
  Also found: "replication" → same image referenced by chunk

STEP 3: Image Loading
  Loaded: replication-topology.png (1200×800, 240KB)

STEP 4: Vision LLM Context
  Text: "Figure 3 shows the replication topology..."
  Image: [replication-topology.png]
  Query: "Does the replication topology show a read replica in eu-west?"

STEP 5: Vision LLM Answer
  "Yes, the replication topology diagram in Figure 3 shows a read replica
   in the eu-west region (labeled 'Replica 1'). The replication flow is:
   Primary (us-east) → Replica 1 (eu-west, read-only) → Replica 2 (us-east, read-only)."

STEP 6: Cost & Latency
  Text retrieval: $0.0002, 200ms
  Image processing: $0.002, 300ms
  Vision LLM: $0.008, 3200ms
  Total: $0.0102, 3700ms
```

### With vs. Without Vision

```
Query: "In the Q1 revenue chart, which month had the lowest revenue?"

TEXT-ONLY RAG:
  Retrieved: "Q1 2025 financial results show strong performance..."
  Answer: "The query mentions a Q1 revenue chart, but I don't have
           access to the chart image. Based on text context, I cannot
           determine which month had the lowest revenue."
  Coverage: 0% (cannot answer at all)

VISION RAG:
  Retrieved: text + chart image
  Answer: "Based on the Q1 2025 revenue chart:
           • January: $12.4M
           • February: $8.7M (lowest)
           • March: $11.2M (recovery)
           February had the lowest revenue at $8.7M, followed by
           a strong recovery to $11.2M in March."
  Coverage: 100%
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Indexing Every Image as a Separate Document

```
WRONG:
  Index each image screenshot as a standalone text document
  Problem: No connection to surrounding text context.
           Text says "see Figure 3" but Figure 3's OCR stands alone.
           
RIGHT:
  Index images WITH their surrounding text context
  The text chunk and image reference should BOTH be retrievable
  from a single search, and the image should inherit the chunk's metadata.
```

### Antipattern 2: Using Vision LLM for Every Query

```
WRONG:
  Every query → vision LLM → expensive and slow
  Even "What's the refund policy?" triggers a model that can see images

RIGHT:
  Route queries: text-only → standard LLM, visual → vision LLM
  Save 80-90% on cost by not paying for vision when it's not needed
```

### Antipattern 3: Only OCR, No Descriptions

```
WRONG:
  Index only OCR text from images
  "Primary → Replica1 → Replica2" (useful for text search but loses visual context)
  
RIGHT:
  Index OCR + LLM descriptions + CLIP embeddings
  "Database replication topology... primary node... two replicas..."
  (searchable by text AND retrievable by visual similarity via CLIP)
```

### Antipattern 4: Not Resizing Images Before API Calls

```
WRONG:
  Pass 8K resolution screenshots to vision LLM
  → Massive token usage ($0.10/image), slow upload times
  
RIGHT:
  Resize to max 2048×2048 before sending
  → Consistent $0.002/image, fast processing
```

### Failure Mode: CLIP Fails on Domain-Specific Diagrams

CLIP is trained on web images (cats, dogs, landscapes, people). A network topology diagram from your cloud infrastructure docs looks NOTHING like CLIP's training data. The embedding will be essentially random.

**Production solutions:**
1. Fine-tune CLIP on your domain's images (requires ~1000 labeled pairs)
2. Use domain-specific variants (BiomedCLIP for medical, SatelliteCLIP for geospatial)
3. Fall back to text-only retrieval with the surrounding text
4. Extract ALL text from the image and rely on text embeddings instead

### Failure Mode: Text-Only Retrieval Never Finds the Image

The text chunk says "See Figure 3 for details." But the text chunk's embedding is about "details" not about the image content. When a user asks about the diagram content, this chunk may not rank highly.

**Production solution:** Explicitly INCLUDE image descriptions in the text chunks that reference them:

```python
# BEFORE enrichment:
chunk = "See Figure 3 for the replication topology."

# AFTER enrichment:
chunk = "See Figure 3 for the replication topology. "
        "Figure 3 shows a database replication topology with "
        "a primary node in us-east and read replicas in eu-west "
        "and ap-southeast."
```

### Failure Mode: Cost Surprise with Many Images

A single document might have 20+ screenshots. Vision LLM on all 20 costs ~$0.04 just for image tokens. If you're doing 1000 queries/day, that's $40/day extra.

**Production solution:** Intelligent image selection:
1. Only load images explicitly referenced by retrieved text chunks
2. For multi-image questions, sample images by relevance to the query
3. Cache vision LLM responses for the same (image + query) pair

---

## 🧪 Drills & Challenges

### Drill 1: CLIP vs. Text Embedding on Your Images (45 min)

**Task:** Take 10 images from your corpus. For each:
1. Generate a CLIP embedding
2. Generate a text embedding of the image's OCR/description
3. For 5 test queries, measure which embedding method retrieves the correct image

**Analyze:** How often does CLIP find the right image when text search fails? How often does text search find it when CLIP fails?

### Drill 2: Build a Hybrid Image Search (45 min)

**Task:** Build a search system that accepts text queries and returns images. Use BOTH:
- CLIP embeddings (visual similarity)
- Text embeddings of OCR/descriptions (keyword match)

Combine with weighted RRF. Test on 10 queries and compare to CLIP-only and text-only.

**Success criteria:** Hybrid beats both individual methods on recall.

### Drill 3: Vision RAG with a Broken Image Reference (30 min)

**Task:** Create a document where:
- The text says "See Figure 7 for the complete architecture"
- Figure 7's image file is MISSING (deleted, renamed, wrong path)

Your vision RAG system should:
1. Detect that the image is missing
2. Fall back gracefully (answer from text alone)
3. Log the missing image for remediation

### Drill 4: The Cross-Modal Query Challenge (45 min)

**Task:** Build and test queries that REQUIRE cross-modal retrieval:
1. Text query → must find an image (not text)
2. Image query → must find text (not an image)
3. Mixed query → must find both

Example cross-modal queries:
- "Show me the architecture diagram from the deployment guide" (text → image)
- [screenshot of error] → "What does this error mean?" (image → text)

### Drill 5: Cost Optimization (30 min)

**Task:** Given a production system processing 5000 queries/day:
- 80% are text-only ("What's the refund policy?")
- 15% reference images ("What does Figure 3 show?")
- 5% require actual visual understanding ("Read this chart")

Current implementation: ALL queries go through vision LLM at $0.012/query.

Implement a THREE-tier routing system:
- Tier 1: Text-only → standard LLM ($0.001/query)
- Tier 2: Image-reference → text retrieval + vision LLM ($0.008/query)
- Tier 3: Visual understanding → full vision RAG ($0.015/query)

**Calculate:** Daily cost before vs. after routing. How much do you save?

### Drill 6: Image Description Quality Analysis (30 min)

**Task:** Take 5 different types of images (diagram, screenshot, chart, photo, infographic). For each:
1. Generate an LLM description (using GPT-4o-mini vision)
2. Extract OCR text
3. Write your OWN description of what's important

**Analyze:** What did the LLM description miss? What did it hallucinate? What would you change about the prompt to improve accuracy for each image type?

### Drill 7: Full Multimodal RAG System (60 min)

**Task:** Build a complete multimodal RAG system that:
1. Ingests a document with text + images (use a mock PDF or HTML)
2. Preprocesses each image: OCR, description, CLIP embedding
3. Indexes text chunks + image entries in a unified vector store
4. Accepts a query, retrieves relevant text + images
5. If the query requires visual understanding, passes images to vision LLM
6. Returns an answer with image citations

**Test with:**
- Text-only query: "What is the rate limit?"
- Image-reference query: "What does the architecture diagram show?"
- Cross-modal query: "Based on Figure 3 and the surrounding text, explain the failover process"

---

## 🚦 Gate Check

Before moving to File 06 (GraphRAG), confirm you can:

- [ ] Explain the THREE strategies for multimodal RAG and when each is appropriate
- [ ] Build a CLIP embedding pipeline for cross-modal retrieval
- [ ] Implement an OCR + vision description preprocessing pipeline
- [ ] Build a unified retriever that searches text and images simultaneously
- [ ] Implement vision LLM RAG for image understanding at query time
- [ ] Design a routing system that sends text-only queries to standard LLM and visual queries to vision LLM
- [ ] Know the cost-latency tradeoffs of each multimodal strategy
- [ ] Handle missing images, wrong paths, and other production failures gracefully
- [ ] Understand CLIP's limitations on domain-specific images

**Sample gate questions:**

1. **A user asks "What does the screenshot show?" and provides an image. Your RAG system needs to find related text AND understand the image. Walk through the full pipeline.**

2. **Your CLIP-based image retrieval is returning irrelevant images for domain-specific queries. What approaches can you use to improve it without fine-tuning a new model?**

3. **Your production system processes 10,000 queries/day. 90% don't need vision. 5% need it. 5% would benefit from it. Design a cost-optimized architecture.**

4. **A text chunk says "See Appendix A for the full schema diagram." But the image file for Appendix A is corrupted. How does your system behave? Design the failure handling.**

---

## 📚 Resources

- **CLIP (Radford et al., 2021):** Learning Transferable Visual Models From Natural Language Supervision — the foundational paper
- **SigLIP (Zhai et al., 2023):** Sigmoid Loss for Language Image Pre-Training — improved CLIP training
- **BLIP-2 (Li et al., 2023):** Bootstrapping Language-Image Pre-training with Frozen Image Encoders and Large Language Models — bridge models
- **OpenCLIP:** Open-source implementation of CLIP with many pretrained checkpoints — https://github.com/mlfoundations/open_clip
- **MMMU Benchmark:** Multimodal understanding benchmark — https://mmmu-benchmark.github.io/
- **GPT-4o Vision Guide:** https://platform.openai.com/docs/guides/vision
- **EasyOCR:** Lightweight OCR library — https://github.com/JaidedAI/EasyOCR
- **LayoutLM (Xu et al., 2020):** Pre-training of Text and Layout for Document Image Understanding — for document-level multimodal understanding
- **Donut (Kim et al., 2022):** OCR-free Document Understanding Transformer — end-to-end document parsing
- **Production Multimodal RAG (Anthropic, 2024):** Building effective agents with vision — https://docs.anthropic.com/en/docs/build-with-claude/vision
