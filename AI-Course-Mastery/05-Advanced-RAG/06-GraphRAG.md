# 06 — GraphRAG: Knowledge Graphs for Structured Retrieval

## 🎯 Purpose & Goals

Vector search treats all text as a bag of vectors. It doesn't understand ENTITIES — that "Dr. Sarah Chen" and "Director of AI Research" are the same person. It doesn't understand RELATIONSHIPS — that "Project Alpha" is "led by" "Dr. Sarah Chen" who "directs" "AI Research" which "focuses on" "NLP."

A user asks: *"What research areas does Project Alpha's leader focus on?"*

Standard vector RAG retrieves chunks. It might get the chunk about Project Alpha (mentions Dr. Chen), the chunk about Dr. Chen (mentions AI Research), and the chunk about AI Research (mentions NLP). But it has NO guarantee these chunks are connected, no way to traverse the entity chain, and no structured way to verify the answer is correct.

**GraphRAG** solves this by:

1. **Extracting entities** from your documents — people, organizations, technologies, concepts
2. **Extracting relationships** between entities — "leads," "reports-to," "focuses-on," "part-of"
3. **Building a knowledge graph** — a structured network of entities and relationships
4. **Using graph traversal** for retrieval — instead of "find similar chunks," use "find entities connected to this query via known relationships"

The result: guaranteed relationship chains, explainable retrieval paths, and answers you can verify by walking the graph.

**By the end of this module, you will:**

1. Understand **knowledge graphs** and why they complement vector search
2. Build an **entity extraction pipeline** that identifies entities from text
3. Implement **relationship extraction** that discovers connections between entities
4. Build and query a **graph index** using NetworkX
5. Implement **Microsoft's GraphRAG pattern** — entity extraction → community detection → summarization → retrieval
6. Build a **hybrid graph+vector retriever** that combines structured and semantic search
7. Know the **cost-quality tradeoffs** of GraphRAG vs. vector-only RAG
8. Understand the **failure modes unique to graph-based retrieval**

---

### ⏹ STOP. Answer these before proceeding.

**Question 1 — The Two-Entity, Four-Hop Question**

*A user asks: Did anyone from Acme Corp's security team publish research on transformer efficiency that was cited in the annual report?*

*This question requires:*
- *Acme Corp → employs → Security Team*
- *Security Team → includes → Person X*
- *Person X → published → Paper on transformer efficiency*
- *Paper → cited in → Annual Report*

*That's 4 relationship hops. Each hop requires finding the RIGHT entity and traversing the RIGHT relationship type.*

*With vector search, you'd need to retrieve chunks containing ALL these entities and relationships in the same context window. With a graph, you traverse from Acme Corp → annual report citations following typed relationships, guaranteeing each step is connected to the previous one.*

*How would you design the entity extraction to capture these relationships? What if "Acme Corp" is never explicitly named — it's always "the company" or "our organization"? How do you handle entity resolution (knowing "the company" = "Acme Corp")?*

*What if some relationships are IMPLICIT? The annual report mentions "Dr. Chen's work on transformers" — the relationship "published" is never explicitly stated, but can be inferred from context.*

**Question 2 — The Scale Tradeoff**

*Your company has 10,000 documents about products, policies, engineering decisions, and customer conversations.*

*Option A: Extract ALL entities and relationships from ALL documents. Build a complete knowledge graph. You'll capture everything, but the extraction costs are enormous (thousands of LLM calls), the graph becomes massive, and querying it requires sophisticated traversal logic.*

*Option B: Extract entities only for QUERIES that need graph capabilities. When a query comes in, extract entities from the query, retrieve candidate documents, extract entities + relationships from those documents, build a small graph on-the-fly, and answer. This is cheaper but adds query-time latency.*

*Option C: Don't build a graph at all. Use better chunking and more retrieval to ensure connected entities end up in the same context window.*

*What are the cost-latency-quality tradeoffs of each approach? At what scale does option A become worth it? At what scale does option C become insufficient?*

*How would you MEASURE when graph-based retrieval is outperforming vector-only retrieval? What metrics would tell you "we need a graph here"?*

**Question 3 — The Entity Resolution Nightmare**

*Your documents mention:*
- *"Dr. Sarah Chen" (formal, first doc)*
- *"Sarah Chen" (informal, second doc)*
- *"Dr. Chen" (abbreviated, third doc)*
- *"S. Chen" (citation format, fourth doc)*
- *"Sarah Chen, PhD" (signature block, fifth doc)*

*Are these the SAME entity or different entities? A human knows yes. An entity extraction system needs to figure this out.*

*Now consider:*
- *"Apple" the fruit vs. "Apple" the company*
- *"Michael Johnson" the CEO vs. "Michael Johnson" the intern (same name, different people)*
- *"AI Research" (a team within Acme) vs. "AI research" (the general academic field)*

*How do you design an entity resolution system that correctly merges "Dr. Sarah Chen" and "S. Chen" but correctly SEPARATES "Apple" (fruit) from "Apple" (company)?*

*What happens when entity resolution makes a mistake? Merged entities create WRONG relationships in the graph — "Apple (fruit)" now connects to "iPhone release dates" because the system merged them. How do you detect and fix these errors?*

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 15 min |
| Knowledge Graph Fundamentals | 20 min |
| Entity Extraction from Text | 35 min |
| Relationship Extraction | 35 min |
| Graph Construction & Storage | 30 min |
| Graph Traversal for Retrieval | 30 min |
| Microsoft GraphRAG Pattern | 30 min |
| Hybrid Graph+Vector Retrieval | 30 min |
| Code: Entity Extraction Pipeline | 45 min |
| Code: Graph Builder | 45 min |
| Code: GraphRAG Retriever | 45 min |
| Code: Hybrid Retriever | 30 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~7.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. Knowledge Graph Fundamentals

A **knowledge graph** is a structured representation of entities (nodes) and their relationships (edges):

```
                    ┌──────────────────┐
                    │   Project Alpha  │
                    │   (Project)      │
                    └────────┬─────────┘
                             │ led_by
                             ▼
                    ┌──────────────────┐
                    │ Dr. Sarah Chen   │
                    │   (Person)       │
                    └────────┬─────────┘
                             │ director_of
                             ▼
                    ┌──────────────────┐
                    │  AI Research     │
                    │   (Team)         │
                    └────────┬─────────┘
                             │ focuses_on
                             ▼
                    ┌──────────────────┐
                    │  NLP & Computer  │
                    │  Vision (Topic)  │
                    └──────────────────┘
```

**Why graphs beat vectors for relationship-heavy questions:**

| Aspect | Vector Search | Graph Search |
|--------|--------------|--------------|
| **Entity discovery** | Finds chunks mentioning entities | Finds entities directly, typed |
| **Relationship chain** | No concept of relationships | Explicit typed edges, traversable |
| **Path guarantee** | No guarantee chunks are connected | Every path is a verified connection |
| **Explainability** | "These chunks are similar to your query" | "Entity A → relationship R → Entity B" |
| **Multi-hop reasoning** | Requires luck (all entities in top-k) | Deterministic traversal |
| **Handling ambiguity** | "Apple" near "iPhone" → company; near "pie" → fruit | Entity type disambiguates |

**The key insight:** Vector search is for FINDING content. Graph search is for NAVIGATING content. You need both.

### 2. The Microsoft GraphRAG Pattern (2024)

Microsoft's GraphRAG (2024) is the most influential production GraphRAG approach. Its insight: **don't retrieve raw entities — retrieve COMMUNITY SUMMARIES.** Here's the full pipeline:

```
Phase 1 — Indexing (One-time, expensive):
┌──────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Source   │──→│    Entity     │──→│   Knowledge   │──→│   Community   │
│ Documents │   │  Extraction   │   │    Graph     │   │   Detection   │
└──────────┘   └──────────────┘   └──────────────┘   └──────┬───────┘
                                                             │
                                                             ▼
                                                    ┌──────────────┐
                                                    │   Community   │
                                                    │  Summaries    │
                                                    └──────────────┘

Phase 2 — Query (Fast, per-query):
┌──────────┐   ┌──────────────┐   ┌───────────────────┐
│  Query   │──→│   Community  │──→│   LLM Generation  │
│          │   │   Retrieval  │   │   (with summaries) │
└──────────┘   └──────────────┘   └───────────────────┘
```

**The steps in detail:**

1. **Entity Extraction:** For each document chunk, use an LLM to extract entities (people, places, organizations, concepts) and their relationships.

2. **Knowledge Graph Construction:** Build a graph where entities are nodes and relationships are edges. Merge duplicate entities (resolve "Dr. Chen" → "Dr. Sarah Chen").

3. **Community Detection:** Use graph algorithms (Leiden clustering) to find COMMUNITIES — groups of entities that are more connected to each other than to the rest of the graph.

4. **Community Summarization:** For each community, use an LLM to generate a SUMMARY — "what is this group of entities about?" This creates a hierarchy of summaries from broad (entire graph) to specific (individual community).

5. **Query-Time Retrieval:** Instead of retrieving chunks, retrieve the MOST RELEVANT community summaries for the query, then generate the answer from those summaries.

**Why community summarization is powerful:** A single document chunk might mention one or two entities. But a community summary describes the BIGGER PICTURE — the relationships, the context, the narrative. For questions like "What are the main themes in the annual report?" or "Summarize the security team's concerns," community summaries outperform chunk retrieval by a wide margin.

### 3. The Graph+Vector Hybrid

The production approach: combine graph traversal with vector search for the best of both worlds:

```
Query: "What research does Project Alpha's leader focus on?"

Step 1: Entity Extraction from Query
  entities: ["Project Alpha"]

Step 2: Graph Traversal
  "Project Alpha" ──led_by──→ "Dr. Sarah Chen" ──directs──→ "AI Research"
                                                              │
                                                    ┌─────────┴──────────┐
                                                    ▼                    ▼
                                              focuses_on             vector_search
                                                    │                    │
                                                    ▼                    ▼
                                          "NLP & Computer       Chunks about AI
                                           Vision"               Research papers

Step 3: Merge
  Answer = graph relationships + vector chunk content
  → "Dr. Sarah Chen leads Project Alpha and directs AI Research,
     which focuses on NLP and computer vision. Specific papers include..."
```

**When to use each:**

| Signal | Use Graph | Use Vector |
|--------|-----------|------------|
| "Who reports to whom?" | ✅ Direct traversal | ❌ Just finds org chart chunks |
| "Research areas of person X" | ✅ Follows entity chain | ⚠️ Needs all entities in context |
| "What papers were published in 2024?" | ❌ Graph doesn't store paper text | ✅ Vector finds paper details |
| "Security team's Q4 concerns" | ✅ Finds team members, then topics | ✅ Finds individual mentions |
| "How does X relate to Y?" | ✅ Relationship path | ❌ Can't express relationships |

---

## 💻 Code Examples

### 1. Entity and Relationship Extraction

```python
"""
entity_extraction.py — Extract entities and relationships from text
to build a knowledge graph for RAG.

Uses an LLM to identify:
- Entities: Named things (people, organizations, concepts, technologies)
- Relationships: Connections between entities (leads, part-of, focuses-on)
- Attributes: Properties of entities (role, department, year)
"""

from typing import List, Optional, Dict, Any, Tuple
from dataclasses import dataclass, field
import json
import asyncio
import re
import logging

from openai import AsyncOpenAI

logger = logging.getLogger(__name__)


# ─── Data Models ────────────────────────────────────────────────────────────

@dataclass
class Entity:
    """A single entity extracted from text."""
    name: str
    type: str  # PERSON, ORGANIZATION, CONCEPT, TECHNOLOGY, LOCATION, EVENT, etc.
    description: str
    aliases: List[str] = field(default_factory=list)
    metadata: Dict[str, Any] = field(default_factory=dict)
    source_chunk_id: str = ""
    confidence: float = 1.0

    @property
    def normalized_name(self) -> str:
        """Get the primary name for graph node identification."""
        return self.name.lower().strip()


@dataclass
class Relationship:
    """A typed relationship between two entities."""
    source: str  # Source entity name
    target: str  # Target entity name
    type: str  # LEADS, PART_OF, FOCUSES_ON, REPORTS_TO, etc.
    description: str
    metadata: Dict[str, Any] = field(default_factory=dict)
    source_chunk_id: str = ""
    confidence: float = 1.0

    @property
    def edge_id(self) -> str:
        """Unique identifier for this relationship."""
        return f"{self.source.lower().strip()}|{self.type}|{self.target.lower().strip()}"


# ─── Entity Extractor ───────────────────────────────────────────────────────

class EntityExtractor:
    """
    Extract entities and their relationships from text chunks using an LLM.

    The LLM identifies:
    - Named entities (people, organizations, locations)
    - Conceptual entities (technologies, methods, frameworks)
    - Relationships between entities (who leads what, what depends on what)
    - Entity aliases (alternative names for the same entity)
    """

    EXTRACTION_PROMPT = """Extract entities and their relationships from the following text.

ENTITIES are specific, named things: people, organizations, technologies, concepts, locations, events, products.

RELATIONSHIPS are typed connections between entities: leads, part_of, focuses_on, reports_to, uses, depends_on, etc.

For each entity provide:
- name: The canonical name
- type: PERSON, ORGANIZATION, TECHNOLOGY, CONCEPT, LOCATION, PRODUCT, EVENT, ROLE, OTHER
- description: 1-2 sentence description of what this entity is
- aliases: Alternative names for the same entity (abbreviations, nicknames, alternate spellings)

For each relationship provide:
- source: Name of the source entity (MUST match an entity name exactly)
- target: Name of the target entity
- type: LEADS, PART_OF, FOCUSES_ON, REPORTS_TO, USES, DEPENDS_ON, CREATED, LOCATED_IN, IMPLEMENTS, RELATED_TO, PUBLISHED, CITES, MENTIONS, HAS_ROLE
- description: Brief description of the relationship

Rules:
- Only extract entities that are EXPLICITLY mentioned or clearly implied
- Do NOT extract generic concepts as entities ("the system" → not an entity unless it's a named system)
- Merge obvious aliases ("Dr. Chen" and "Dr. Sarah Chen" are the same entity)
- Be precise with relationship types — don't default to RELATED_TO when a more specific type exists
- If you're unsure about an extraction, set confidence to "low"

Return JSON with this structure:
{{
    "entities": [
        {{
            "name": "Entity Name",
            "type": "PERSON",
            "description": "Description of entity",
            "aliases": ["Alias 1", "Alias 2"],
            "confidence": "high" | "medium" | "low"
        }}
    ],
    "relationships": [
        {{
            "source": "Entity Name",
            "target": "Target Entity",
            "type": "LEADS",
            "description": "Description of relationship",
            "confidence": "high" | "medium" | "low"
        }}
    ]
}}
"""

    ENTITY_TYPES = [
        "PERSON", "ORGANIZATION", "TECHNOLOGY", "CONCEPT",
        "LOCATION", "PRODUCT", "EVENT", "ROLE", "OTHER"
    ]

    RELATIONSHIP_TYPES = [
        "LEADS", "PART_OF", "FOCUSES_ON", "REPORTS_TO",
        "USES", "DEPENDS_ON", "CREATED", "LOCATED_IN",
        "IMPLEMENTS", "RELATED_TO", "PUBLISHED", "CITES",
        "MENTIONS", "HAS_ROLE", "EMPLOYS", "COLLABORATES_WITH",
        "PRECEDES", "FOLLOWS", "CONTAINS", "PRODUCES"
    ]

    def __init__(
        self,
        llm_client: AsyncOpenAI,
        model: str = "gpt-4o-mini",
        max_entities_per_chunk: int = 20,
        max_relationships_per_chunk: int = 30,
    ):
        self.llm_client = llm_client
        self.model = model
        self.max_entities = max_entities_per_chunk
        self.max_relationships = max_relationships_per_chunk

    async def extract(
        self,
        text: str,
        chunk_id: str = "",
        existing_entities: Optional[List[str]] = None,
    ) -> Tuple[List[Entity], List[Relationship]]:
        """
        Extract entities and relationships from text.

        Args:
            text: The text chunk to extract from
            chunk_id: Identifier for the source chunk
            existing_entities: Known entity names to help guide extraction

        Returns:
            Tuple of (entities, relationships)
        """
        prompt = self.EXTRACTION_PROMPT

        if existing_entities:
            prompt += f"\n\nThese entities are known to exist in the broader corpus. Link to them if relevant:\n"
            prompt += "\n".join(f"- {name}" for name in existing_entities[:50])

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": prompt},
                {"role": "user", "content": f"Text:\n\n{text[:6000]}"}
            ],
            response_format={"type": "json_object"},
            temperature=0.1,
            max_tokens=3000,
        )

        try:
            data = json.loads(response.choices[0].message.content)
        except (json.JSONDecodeError, AttributeError) as e:
            logger.warning(f"Extraction JSON parse failed for chunk {chunk_id}: {e}")
            return [], []

        entities = []
        for e in data.get("entities", []):
            if e.get("confidence", "medium") == "low":
                continue
            entities.append(Entity(
                name=e["name"],
                type=e.get("type", "OTHER"),
                description=e.get("description", ""),
                aliases=e.get("aliases", []),
                source_chunk_id=chunk_id,
                confidence=1.0 if e.get("confidence") == "high" else 0.7,
            ))

        relationships = []
        for r in data.get("relationships", []):
            if r.get("confidence", "medium") == "low":
                continue
            relationships.append(Relationship(
                source=r["source"],
                target=r["target"],
                type=r.get("type", "RELATED_TO"),
                description=r.get("description", ""),
                source_chunk_id=chunk_id,
                confidence=1.0 if r.get("confidence") == "high" else 0.7,
            ))

        # Limit to configured max
        entities = entities[:self.max_entities]
        relationships = relationships[:self.max_relationships]

        logger.info(f"Extracted {len(entities)} entities, {len(relationships)} "
                    f"relationships from chunk {chunk_id}")
        return entities, relationships

    async def extract_batch(
        self,
        chunks: List[Dict[str, Any]],
        batch_size: int = 5,
        existing_entity_names: Optional[List[str]] = None,
    ) -> Tuple[List[Entity], List[Relationship]]:
        """
        Extract from multiple chunks in parallel batches.

        Args:
            chunks: List of dicts with "id" and "text" keys
            batch_size: Number of concurrent extractions

        Returns:
            Tuple of (all_entities, all_relationships)
        """
        all_entities = []
        all_relationships = []

        for i in range(0, len(chunks), batch_size):
            batch = chunks[i:i + batch_size]
            tasks = [
                self.extract(
                    chunk.get("text", chunk.get("content", "")),
                    chunk_id=chunk.get("id", f"chunk_{i + j}"),
                    existing_entities=existing_entity_names,
                )
                for j, chunk in enumerate(batch)
            ]
            results = await asyncio.gather(*tasks)

            for entities, relationships in results:
                all_entities.extend(entities)
                all_relationships.extend(relationships)

            logger.info(f"Processed batch {i // batch_size + 1}: "
                        f"{sum(len(r[0]) + len(r[1]) for r in results)} extractions")

        return all_entities, all_relationships

    async def extract_from_query(self, query: str) -> Tuple[List[Entity], str]:
        """
        Extract entities from a user query for graph traversal.

        Returns:
            Tuple of (entities, search_intent)
            search_intent is a description of what the user is looking for
        """
        QUERY_EXTRACTION_PROMPT = """Extract the key entities from this user query.
These entities will be used to traverse a knowledge graph and find relevant information.

Also determine the SEARCH INTENT: what kind of information is the user looking for?

Return JSON:
{{
    "entities": [
        {{
            "name": "Extracted entity name",
            "type": "PERSON | ORGANIZATION | TECHNOLOGY | CONCEPT",
            "role_in_query": "How this entity relates to the question"
        }}
    ],
    "search_intent": "What the user wants to know",
    "relationship_hints": ["possible relationship types to traverse",
                           "e.g., LEADS, FOCUSES_ON, PART_OF"]
}}
"""

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": QUERY_EXTRACTION_PROMPT},
                {"role": "user", "content": query}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
        )

        try:
            data = json.loads(response.choices[0].message.content)
            entities = [
                Entity(
                    name=e["name"],
                    type=e.get("type", "CONCEPT"),
                    description=e.get("role_in_query", ""),
                    source_chunk_id="__query__",
                )
                for e in data.get("entities", [])
            ]
            search_intent = data.get("search_intent", "")
            return entities, search_intent

        except (json.JSONDecodeError, AttributeError) as e:
            logger.warning(f"Query extraction failed: {e}")
            return [], query


# ─── Entity Resolver ────────────────────────────────────────────────────────

class EntityResolver:
    """
    Resolve different mentions of the same entity to a single canonical form.

    Handles:
    - "Dr. Sarah Chen" → "Sarah Chen"
    - "Dr. Chen" → "Sarah Chen"
    - "S. Chen" → "Sarah Chen" (if we can determine this)
    - "Apple (company)" vs "Apple (fruit)" — context-aware disambiguation
    """

    def __init__(self, llm_client: Optional[AsyncOpenAI] = None):
        self.llm_client = llm_client
        self._canonical_map: Dict[str, str] = {}  # alias → canonical
        self._entity_registry: Dict[str, Entity] = {}  # canonical → Entity

    def register_entity(self, entity: Entity) -> str:
        """
        Register an entity, resolving against existing entities.

        Returns the canonical name.
        """
        canonical = entity.normalized_name

        # Check for exact match
        if canonical in self._entity_registry:
            existing = self._entity_registry[canonical]
            # Merge descriptions (prefer more detailed)
            if len(entity.description) > len(existing.description):
                existing.description = entity.description
            # Merge aliases
            for alias in entity.aliases:
                if alias.lower().strip() not in self._canonical_map:
                    self._canonical_map[alias.lower().strip()] = canonical
            return canonical

        # Check aliases
        for alias in entity.aliases:
            alias_key = alias.lower().strip()
            if alias_key in self._canonical_map:
                target = self._canonical_map[alias_key]
                if target in self._entity_registry:
                    existing = self._entity_registry[target]
                    existing.aliases.append(entity.name)
                    return target

        # Check by name similarity (simple approach — in production use embedding)
        resolver_conflict = self._find_similar_name(canonical)
        if resolver_conflict:
            # Found a similar entity — merge
            logger.info(f"Merging '{entity.name}' → '{resolver_conflict}' (by name similarity)")
            self._canonical_map[canonical] = resolver_conflict
            existing = self._entity_registry[resolver_conflict]
            existing.aliases.append(entity.name)
            if len(entity.description) > len(existing.description):
                existing.description = entity.description
            return resolver_conflict

        # New entity
        self._entity_registry[canonical] = entity
        self._canonical_map[canonical] = canonical
        for alias in entity.aliases:
            self._canonical_map[alias.lower().strip()] = canonical

        return canonical

    def _find_similar_name(self, name: str) -> Optional[str]:
        """
        Find a similar entity name already registered.
        Simple heuristic — in production, use embedding similarity.
        """
        name_lower = name.lower()

        for registered_name in self._entity_registry:
            reg_lower = registered_name.lower()

            # Check if one contains the other
            if (name_lower in reg_lower or reg_lower in name_lower) and \
               len(name_lower) > 3 and len(reg_lower) > 3:
                # Check that the shorter name is likely a reasonable suffix/prefix
                if name_lower in reg_lower:
                    return registered_name

            # Check shared tokens (e.g., "Sarah Chen" and "Dr. Sarah Chen")
            name_tokens = set(name_lower.split())
            reg_tokens = set(reg_lower.split())
            shared = name_tokens & reg_tokens

            if len(shared) >= 2:  # At least 2 tokens match: "Sarah" + "Chen"
                return registered_name

        return None

    def resolve(self, name: str) -> str:
        """Resolve an entity name to its canonical form."""
        key = name.lower().strip()
        return self._canonical_map.get(key, key)

    def get_entity(self, canonical_name: str) -> Optional[Entity]:
        """Get the canonical Entity object."""
        return self._entity_registry.get(canonical_name.lower().strip())

    def all_entities(self) -> List[Entity]:
        """Get all registered entities."""
        return list(self._entity_registry.values())

    def entity_count(self) -> int:
        return len(self._entity_registry)

    def merge_entities(self, primary: str, alias: str):
        """
        Manually merge two entity references.

        Useful for fixing resolution errors.
        """
        primary_key = primary.lower().strip()
        alias_key = alias.lower().strip()

        if primary_key not in self._entity_registry and alias_key in self._entity_registry:
            # Swap: make alias the canonical
            self._canonical_map[primary_key] = alias_key
        elif primary_key in self._entity_registry and alias_key not in self._entity_registry:
            self._canonical_map[alias_key] = primary_key
        elif primary_key in self._entity_registry and alias_key in self._entity_registry:
            # Merge alias entity into primary
            alias_entity = self._entity_registry.pop(alias_key)
            primary_entity = self._entity_registry[primary_key]
            for a in alias_entity.aliases:
                if a not in primary_entity.aliases:
                    primary_entity.aliases.append(a)
            self._canonical_map[alias_key] = primary_key


# ─── Usage Example ──────────────────────────────────────────────────────────

async def extraction_demo():
    """Demonstrate entity extraction from documents."""
    client = AsyncOpenAI()
    extractor = EntityExtractor(llm_client=client)

    documents = [
        {
            "id": "doc_1",
            "text": """
Project Alpha was launched in 2023 under the leadership of Dr. Sarah Chen,
who serves as the Director of AI Research at Acme Corp. The project focuses
on developing efficient transformer architectures for production NLP systems.
Dr. Chen's team of 15 researchers published 3 papers on this topic in 2024.
            """
        },
        {
            "id": "doc_2",
            "text": """
The AI Research division, led by Dr. Chen, collaborates closely with the
Security Team headed by Michael Torres. Together they ensure that all
NLP systems comply with Acme Corp's security requirements. The Security Team
published a framework for secure LLM deployment in Q3 2024.
            """
        },
    ]

    print("=" * 60)
    print("ENTITY EXTRACTION DEMO")
    print("=" * 60)

    all_entities = []
    all_relationships = []

    for doc in documents:
        entities, relationships = await extractor.extract(
            doc["text"], chunk_id=doc["id"]
        )
        all_entities.extend(entities)
        all_relationships.extend(relationships)

        print(f"\n📄 Document: {doc['id']}")
        print(f"  Entities ({len(entities)}):")
        for e in entities:
            print(f"    [{e.type}] {e.name} — {e.description[:60]}")
            if e.aliases:
                print(f"           aliases: {', '.join(e.aliases)}")

        print(f"  Relationships ({len(relationships)}):")
        for r in relationships:
            print(f"    {r.source} → [{r.type}] → {r.target}")

    print(f"\n\nTotal: {len(all_entities)} entities, {len(all_relationships)} relationships")

    # Resolve entities
    resolver = EntityResolver()
    for e in all_entities:
        canonical = resolver.register_entity(e)
        if canonical != e.normalized_name:
            print(f"  Resolved '{e.name}' → '{canonical}'")

    print(f"\nUnique entities after resolution: {resolver.entity_count()}")
```

---

### 2. Knowledge Graph Construction (NetworkX)

```python
"""
graph_builder.py — Build and query a knowledge graph from extracted entities.

Uses NetworkX for in-memory graph operations. In production,
replace with Neo4j or a graph database for scale.
"""

from typing import List, Optional, Dict, Any, Tuple, Set
from dataclasses import dataclass, field
from collections import defaultdict
import json
import logging

import networkx as nx

logger = logging.getLogger(__name__)


# ─── Knowledge Graph ────────────────────────────────────────────────────────

class KnowledgeGraph:
    """
    A knowledge graph built from extracted entities and relationships.

    Wraps NetworkX with domain-specific operations for RAG:
    - Entity lookup (fuzzy name matching)
    - Relationship traversal (follow typed edges)
    - Subgraph extraction (get the neighborhood around an entity)
    - Path finding (shortest path between two entities)
    - Community detection (Leiden clustering)
    - Graph serialization (for storage)

    In production, use Neo4j or ArangoDB for persistence and scale.
    """

    def __init__(self):
        self.graph = nx.MultiDiGraph()  # Directed multigraph (multiple edge types)
        self.entity_resolver = EntityResolver()
        self._entity_descriptions: Dict[str, str] = {}  # canonical → description
        self._chunk_to_entities: Dict[str, List[str]] = defaultdict(list)
        self._entity_to_chunks: Dict[str, List[str]] = defaultdict(list)

    # ─── Construction ───────────────────────────────────────────────────

    def add_entity(self, entity: Entity) -> str:
        """
        Add an entity as a node in the graph.

        Returns the canonical name.
        """
        canonical = self.entity_resolver.register_entity(entity)

        # Add or update node
        node_attrs = {
            "type": entity.type,
            "description": entity.description,
            "source_chunk": entity.source_chunk_id,
            "aliases": entity.aliases,
        }

        # Update existing node if it exists
        if self.graph.has_node(canonical):
            existing = dict(self.graph.nodes[canonical])
            # Merge descriptions (prefer longer)
            if len(entity.description) > len(existing.get("description", "")):
                existing["description"] = entity.description
            # Merge types (prefer more specific)
            if entity.type != "OTHER" and existing.get("type") == "OTHER":
                existing["type"] = entity.type
            # Merge aliases
            for alias in entity.aliases:
                if alias not in existing.get("aliases", []):
                    existing.setdefault("aliases", []).append(alias)
            existing["source_chunks"] = list(set(
                existing.get("source_chunks", [existing.get("source_chunk", "")])
                + [entity.source_chunk_id]
            ))
            nx.set_node_attributes(self.graph, {canonical: existing})
        else:
            entity.source_chunks = [entity.source_chunk_id]
            nx.set_node_attributes(self.graph, {canonical: node_attrs})

        # Track entity descriptions
        if canonical not in self._entity_descriptions or \
           len(entity.description) > len(self._entity_descriptions[canonical]):
            self._entity_descriptions[canonical] = entity.description

        # Track chunk→entity mapping
        if entity.source_chunk_id:
            self._chunk_to_entities[entity.source_chunk_id].append(canonical)
            self._entity_to_chunks[canonical].append(entity.source_chunk_id)

        return canonical

    def add_relationship(self, relationship: Relationship) -> bool:
        """
        Add a relationship as a directed edge in the graph.

        Both source and target entities must exist (added via add_entity).
        """
        source = self.entity_resolver.resolve(relationship.source)
        target = self.entity_resolver.resolve(relationship.target)

        if source == target:
            logger.warning(f"Self-loop relationship ignored: {source} → {relationship.type} → {target}")
            return False

        if not self.graph.has_node(source):
            logger.warning(f"Source entity '{source}' not in graph. Adding placeholder.")
            self.graph.add_node(source, type="UNKNOWN", description="Auto-created placeholder")
        if not self.graph.has_node(target):
            logger.warning(f"Target entity '{target}' not in graph. Adding placeholder.")
            self.graph.add_node(target, type="UNKNOWN", description="Auto-created placeholder")

        # Add edge with metadata
        self.graph.add_edge(
            source, target,
            key=relationship.edge_id,
            type=relationship.type,
            description=relationship.description,
            confidence=relationship.confidence,
            source_chunk=relationship.source_chunk_id,
        )

        return True

    def build_from_extractions(
        self,
        entities: List[Entity],
        relationships: List[Relationship],
    ):
        """Build the graph from lists of entities and relationships."""
        for entity in entities:
            self.add_entity(entity)

        for rel in relationships:
            self.add_relationship(rel)

        logger.info(f"Graph built: {self.node_count} nodes, {self.edge_count} edges")

    # ─── Query ──────────────────────────────────────────────────────────

    def find_entity(self, name: str, fuzzy: bool = True) -> Optional[str]:
        """
        Find an entity in the graph by name.

        Args:
            name: Entity name to find
            fuzzy: If True, try substring matching when exact match fails

        Returns:
            Canonical entity name, or None
        """
        canonical = self.entity_resolver.resolve(name)
        if self.graph.has_node(canonical):
            return canonical

        if fuzzy:
            name_lower = name.lower().strip()
            for node in self.graph.nodes():
                node_lower = node.lower()
                if name_lower in node_lower or node_lower in name_lower:
                    return node

        return None

    def get_entity_info(self, name: str) -> Optional[Dict]:
        """
        Get full information about an entity.
        """
        canonical = self.find_entity(name)
        if not canonical or not self.graph.has_node(canonical):
            return None

        node_data = dict(self.graph.nodes[canonical])

        # Get relationships
        outgoing = []
        for _, target, data in self.graph.out_edges(canonical, data=True):
            outgoing.append({
                "type": data.get("type", "RELATED_TO"),
                "target": target,
                "description": data.get("description", ""),
                "confidence": data.get("confidence", 1.0),
            })

        incoming = []
        for source, _, data in self.graph.in_edges(canonical, data=True):
            incoming.append({
                "type": data.get("type", "RELATED_TO"),
                "source": source,
                "description": data.get("description", ""),
                "confidence": data.get("confidence", 1.0),
            })

        # Get source chunks
        chunk_ids = self._entity_to_chunks.get(canonical, [])

        return {
            "canonical_name": canonical,
            "type": node_data.get("type", "UNKNOWN"),
            "description": node_data.get("description", ""),
            "aliases": node_data.get("aliases", []),
            "outgoing_relationships": sorted(outgoing, key=lambda x: x["type"]),
            "incoming_relationships": sorted(incoming, key=lambda x: x["type"]),
            "source_chunks": chunk_ids,
            "degree": self.graph.degree(canonical),
        }

    def traverse(
        self,
        start_entity: str,
        relationship_types: Optional[List[str]] = None,
        max_depth: int = 2,
        max_nodes: int = 50,
    ) -> Dict[str, Any]:
        """
        Traverse the graph from a starting entity.

        This is the core retrieval operation for GraphRAG:
        Given a starting entity, follow relationships to find connected entities.

        Args:
            start_entity: The entity to start from
            relationship_types: Only traverse these relationship types
            max_depth: Maximum traversal depth
            max_nodes: Maximum nodes to return

        Returns:
            Dict with nodes, edges, and traversal path
        """
        canonical = self.find_entity(start_entity)
        if not canonical:
            return {"nodes": [], "edges": [], "path": []}

        visited_nodes = {canonical}
        visited_edges: Set[Tuple[str, str, str]] = set()
        frontier = {canonical}
        traversal_path = [{"hop": 0, "entity": canonical, "via": "START"}]

        for depth in range(max_depth):
            if len(visited_nodes) >= max_nodes:
                break

            next_frontier = set()
            for current in frontier:
                # Outgoing edges
                for _, target, data in self.graph.out_edges(current, data=True):
                    edge_type = data.get("type", "RELATED_TO")
                    if relationship_types and edge_type not in relationship_types:
                        continue
                    if target not in visited_nodes:
                        next_frontier.add(target)
                        visited_nodes.add(target)
                        visited_edges.add((current, target, edge_type))
                        traversal_path.append({
                            "hop": depth + 1,
                            "entity": target,
                            "via": f"{current} → [{edge_type}] → {target}",
                        })
                    if len(visited_nodes) >= max_nodes:
                        break

                # Incoming edges
                for source, _, data in self.graph.in_edges(current, data=True):
                    edge_type = data.get("type", "RELATED_TO")
                    if relationship_types and edge_type not in relationship_types:
                        continue
                    if source not in visited_nodes:
                        next_frontier.add(source)
                        visited_nodes.add(source)
                        visited_edges.add((source, current, edge_type))
                        traversal_path.append({
                            "hop": depth + 1,
                            "entity": source,
                            "via": f"{source} → [{edge_type}] → {current}",
                        })
                    if len(visited_nodes) >= max_nodes:
                        break

            frontier = next_frontier
            if not frontier:
                break

        # Build result
        nodes = []
        for node in visited_nodes:
            data = dict(self.graph.nodes[node]) if self.graph.has_node(node) else {}
            nodes.append({
                "name": node,
                "type": data.get("type", "UNKNOWN"),
                "description": data.get("description", ""),
            })

        edges = []
        for source, target, etype in visited_edges:
            edge_data = self.graph.get_edge_data(source, target)
            desc = ""
            if edge_data:
                for key, data in edge_data.items():
                    if data.get("type") == etype:
                        desc = data.get("description", "")
            edges.append({
                "source": source,
                "target": target,
                "type": etype,
                "description": desc,
            })

        return {
            "nodes": nodes,
            "edges": edges,
            "path": traversal_path,
            "stats": {
                "total_nodes": len(nodes),
                "total_edges": len(edges),
                "max_depth_reached": max(r["hop"] for r in traversal_path) if traversal_path else 0,
            }
        }

    def find_shortest_path(
        self,
        source: str,
        target: str,
    ) -> Optional[List[Dict]]:
        """
        Find the shortest relationship path between two entities.
        """
        src = self.find_entity(source)
        tgt = self.find_entity(target)

        if not src or not tgt:
            return None

        try:
            path = nx.shortest_path(self.graph, source=src, target=tgt)
        except (nx.NetworkXNoPath, nx.NodeNotFound):
            return None

        result = []
        for i in range(len(path) - 1):
            s, t = path[i], path[i + 1]
            edge_data = self.graph.get_edge_data(s, t)
            edge_type = "RELATED_TO"
            if edge_data:
                # Get the first (most confident) edge
                for key, data in edge_data.items():
                    edge_type = data.get("type", "RELATED_TO")
                    break
            result.append({
                "source": s,
                "target": t,
                "type": edge_type,
                "hop": i + 1,
            })

        return result

    def get_entity_descriptions(self, entity_names: List[str]) -> Dict[str, str]:
        """Get descriptions for a list of entities."""
        result = {}
        for name in entity_names:
            canonical = self.find_entity(name)
            if canonical and canonical in self._entity_descriptions:
                result[canonical] = self._entity_descriptions[canonical]
        return result

    # ─── Community Detection ────────────────────────────────────────────

    def detect_communities(self, resolution: float = 1.0) -> Dict[str, List[str]]:
        """
        Detect communities (clusters) in the graph using the Leiden algorithm.

        Communities are groups of entities that are more connected to each
        other than to the rest of the graph. Each community represents a
        coherent topic or domain in your knowledge base.

        This is a core part of Microsoft's GraphRAG pattern.

        Args:
            resolution: Resolution parameter for Leiden (higher = more communities)

        Returns:
            Dict mapping community_id → list of entity names
        """
        try:
            import networkx.algorithms.community as nx_comm

            # Convert to undirected for community detection
            undirected = self.graph.to_undirected()

            # Use louvain (more widely available than leiden)
            communities = nx_comm.louvain_communities(
                undirected, resolution=resolution, seed=42
            )

            result = {}
            for i, community in enumerate(communities):
                result[f"community_{i}"] = sorted(list(community))

            return result

        except ImportError:
            logger.warning("NetworkX community detection not available. Install python-louvain or leidenalg.")
            # Fallback: connected components
            components = list(nx.weakly_connected_components(self.graph))
            result = {}
            for i, comp in enumerate(components):
                if len(comp) > 1:
                    result[f"component_{i}"] = sorted(list(comp))
            return result

    def summarize_community(
        self,
        community_id: str,
        entity_names: List[str],
        llm_client: Optional[AsyncOpenAI] = None,
    ) -> str:
        """
        Generate a summary of a community using an LLM.

        This is the OTHER core part of Microsoft's GraphRAG:
        After detecting communities, generate a natural language summary
        describing what each community is about.

        Args:
            community_id: Identifier for this community
            entity_names: List of entity names in the community
            llm_client: OpenAI client for LLM summarization

        Returns:
            Natural language summary of the community
        """
        if not llm_client:
            # Without LLM, return a structural summary
            entities_info = []
            for name in entity_names:
                info = self.get_entity_info(name)
                if info:
                    entities_info.append(f"{name} ({info['type']}): {info['description'][:100]}")

            return f"Community {community_id} ({len(entity_names)} entities):\n" + \
                   "\n".join(f"  • {e}" for e in entities_info)

        # Build context for LLM summarization
        context_parts = []
        for name in entity_names:
            info = self.get_entity_info(name)
            if info:
                context_parts.append(
                    f"Entity: {name} (Type: {info['type']})\n"
                    f"Description: {info['description']}\n"
                    f"Relationships: "
                    + "; ".join(
                        f"{r['type']} → {r['target']}"
                        for r in info['outgoing_relationships'][:5]
                    )
                )

        context = "\n\n".join(context_parts)

        prompt = f"""Analyze the following group of related entities from a knowledge graph.
This is community {community_id}.

Entities in this community:
{context}

Provide:
1. A concise title for this community (what is this group about?)
2. A 2-3 sentence summary of the community's overall theme
3. Key entities and their roles
4. The main relationships that hold this community together

Summary (2-3 paragraphs):"""

        import time
        response = llm_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a knowledge graph analysis system."},
                {"role": "user", "content": prompt}
            ],
            max_tokens=500,
            temperature=0.3,
        )

        return response.choices[0].message.content

    # ─── Properties ────────────────────────────────────────────────────

    @property
    def node_count(self) -> int:
        return self.graph.number_of_nodes()

    @property
    def edge_count(self) -> int:
        return self.graph.number_of_edges()

    def get_statistics(self) -> Dict[str, Any]:
        """Get statistics about the graph."""
        if self.node_count == 0:
            return {"node_count": 0, "edge_count": 0}

        # Entity type distribution
        type_counts = defaultdict(int)
        for node, data in self.graph.nodes(data=True):
            type_counts[data.get("type", "UNKNOWN")] += 1

        # Relationship type distribution
        rel_type_counts = defaultdict(int)
        for _, _, data in self.graph.edges(data=True):
            rel_type_counts.get(data.get("type", "RELATED_TO"), 0)
        for _, _, key, data in self.graph.edges(keys=True, data=True):
            rel_type_counts[data.get("type", "RELATED_TO")] += 1

        # Degree stats
        degrees = [d for _, d in self.graph.degree()]
        avg_degree = sum(degrees) / len(degrees) if degrees else 0

        # Connected components
        components = list(nx.weakly_connected_components(self.graph))

        return {
            "node_count": self.node_count,
            "edge_count": self.edge_count,
            "entity_type_distribution": dict(type_counts),
            "relationship_type_distribution": dict(rel_type_counts),
            "avg_degree": round(avg_degree, 2),
            "max_degree": max(degrees) if degrees else 0,
            "connected_components": len(components),
            "largest_component_size": max(len(c) for c in components) if components else 0,
        }


# ─── Complete Graph Builder ─────────────────────────────────────────────────

class GraphBuilder:
    """
    End-to-end graph builder: extract entities → resolve → build graph.

    Orchestrates the full pipeline from raw text to a queryable knowledge graph.
    """

    def __init__(
        self,
        extractor: EntityExtractor,
        llm_client: Optional[AsyncOpenAI] = None,
    ):
        self.extractor = extractor
        self.llm_client = llm_client
        self.graph = KnowledgeGraph()
        self.community_summaries: Dict[str, str] = {}

    async def process_documents(
        self,
        chunks: List[Dict[str, Any]],
        batch_size: int = 5,
        generate_community_summaries: bool = False,
    ) -> KnowledgeGraph:
        """
        Process a list of document chunks into a knowledge graph.

        Args:
            chunks: List of {"id": str, "text": str} dicts
            batch_size: Batch size for concurrent extraction
            generate_community_summaries: If True, generate community summaries

        Returns:
            Built KnowledgeGraph
        """
        logger.info(f"Processing {len(chunks)} chunks into knowledge graph")

        # Step 1: Extract entities and relationships
        if self.graph.node_count > 0:
            existing_names = list(self.graph.entity_resolver._entity_registry.keys())
        else:
            existing_names = None

        all_entities, all_relationships = await self.extractor.extract_batch(
            chunks,
            batch_size=batch_size,
            existing_entity_names=existing_names,
        )

        # Step 2: Build graph
        self.graph.build_from_extractions(all_entities, all_relationships)

        # Step 3: Community detection and summarization (optional)
        if generate_community_summaries and self.llm_client:
            communities = self.graph.detect_communities()
            for community_id, entity_names in communities.items():
                summary = self.graph.summarize_community(
                    community_id, entity_names, self.llm_client
                )
                self.community_summaries[community_id] = summary
                logger.info(f"Generated summary for {community_id} ({len(entity_names)} entities)")

        return self.graph

    def get_community_context(self) -> str:
        """
        Get all community summaries as a single context block.
        Used for Microsoft GraphRAG-style global search.
        """
        if not self.community_summaries:
            return ""

        parts = []
        for cid, summary in self.community_summaries.items():
            parts.append(f"--- {cid} ---\n{summary}")

        return "\n\n".join(parts)
```

---

### 3. GraphRAG Retriever — Traversal + Community Search

```python
"""
graphrag_retriever.py — Retrieval using knowledge graph traversal
and community summaries.

Two retrieval modes:
1. LOCAL: Traverse the graph from query entities, retrieve connected entities + chunks
2. GLOBAL: Use community summaries for broad/thematic questions

Combined with vector search for dense chunk retrieval.
"""

from typing import List, Optional, Dict, Any, Tuple, Literal
from dataclasses import dataclass, field
import asyncio
import time
import logging

from openai import AsyncOpenAI

logger = logging.getLogger(__name__)


# ─── Data Models ────────────────────────────────────────────────────────────

@dataclass
class GraphRAGResult:
    """Result of a GraphRAG retrieval."""
    query: str
    search_type: Literal["local", "global", "hybrid"]

    # Graph traversal results
    entities_found: List[Dict] = field(default_factory=list)
    entity_descriptions: List[str] = field(default_factory=list)
    traversal_path: List[Dict] = field(default_factory=list)

    # Community results (for global search)
    community_summaries: List[str] = field(default_factory=list)

    # Chunks from vector search
    vector_chunks: List[Dict] = field(default_factory=list)

    # Combined context for generation
    combined_context: str = ""

    # Latency tracking
    graph_latency_ms: float = 0.0
    vector_latency_ms: float = 0.0
    total_latency_ms: float = 0.0


# ─── GraphRAG Retriever ─────────────────────────────────────────────────────

class GraphRAGRetriever:
    """
    Retrieve information using knowledge graph traversal and community summaries.

    LOCAL SEARCH: Extract entities from query → traverse graph → get entity context + chunks
    GLOBAL SEARCH: Map query to relevant communities → use community summaries
    HYBRID: Both + vector search
    """

    def __init__(
        self,
        graph: KnowledgeGraph,
        vector_search_fn: Optional[callable] = None,  # Standard RAG vector search
        entity_extractor: Optional[EntityExtractor] = None,
        llm_client: Optional[AsyncOpenAI] = None,
    ):
        self.graph = graph
        self.vector_search = vector_search_fn
        self.entity_extractor = entity_extractor
        self.llm_client = llm_client

    async def local_search(
        self,
        query: str,
        max_depth: int = 2,
        max_nodes: int = 30,
        top_k_chunks: int = 5,
    ) -> GraphRAGResult:
        """
        LOCAL GraphRAG search.

        1. Extract entities from the query
        2. Find those entities in the graph
        3. Traverse from those entities to find connected entities
        4. Get descriptions and source chunks for all found entities
        5. Optionally supplement with vector search
        """
        start = time.time()
        entity_descriptions = []
        traversal_paths = []
        all_found_entities = []

        # Step 1: Extract entities from query
        query_entities = []
        if self.entity_extractor:
            entities, search_intent = await self.entity_extractor.extract_from_query(query)
            query_entities = entities

        # Step 2: Find entities in graph and traverse
        graph_start = time.time()

        for qe in query_entities:
            found = self.graph.find_entity(qe.name)
            if found:
                # Get entity info
                info = self.graph.get_entity_info(found)
                if info:
                    all_found_entities.append(info)
                    entity_descriptions.append(
                        f"{found} ({info['type']}): {info['description']}"
                    )

                # Traverse
                traversal = self.graph.traverse(
                    found,
                    max_depth=max_depth,
                    max_nodes=max_nodes,
                )
                traversal_paths.append(traversal)

                # Collect all entities found via traversal
                for node in traversal.get("nodes", []):
                    if node["name"] != found:
                        entity_descriptions.append(
                            f"{node['name']} ({node['type']}): {node['description']}"
                        )

        graph_latency = (time.time() - graph_start) * 1000

        # Step 3: Vector search for chunk-level detail
        vector_chunks = []
        vector_latency = 0.0
        if self.vector_search:
            vec_start = time.time()
            vector_chunks = await self.vector_search(query, top_k_chunks)
            vector_latency = (time.time() - vec_start) * 1000

        # Step 4: Build combined context
        combined_context = self._build_context(
            entity_descriptions, traversal_paths, vector_chunks, query
        )

        total_latency = (time.time() - start) * 1000

        return GraphRAGResult(
            query=query,
            search_type="local",
            entities_found=all_found_entities,
            entity_descriptions=entity_descriptions,
            traversal_path=traversal_paths,
            vector_chunks=vector_chunks,
            combined_context=combined_context,
            graph_latency_ms=graph_latency,
            vector_latency_ms=vector_latency,
            total_latency_ms=total_latency,
        )

    async def global_search(
        self,
        query: str,
        community_summaries: Dict[str, str],
    ) -> GraphRAGResult:
        """
        GLOBAL GraphRAG search using community summaries.

        This is the Microsoft GraphRAG pattern for broad/thematic questions:
        "What are the main themes?", "Summarize the concerns"

        Instead of traversing the graph, we use pre-computed community summaries
        and select the most relevant ones for the query.
        """
        start = time.time()

        if not community_summaries or not self.llm_client:
            return GraphRAGResult(
                query=query, search_type="global",
                combined_context="No community summaries available."
            )

        # Select relevant communities using LLM
        graph_start = time.time()
        selected_summaries = await self._select_relevant_communities(
            query, community_summaries
        )
        graph_latency = (time.time() - graph_start) * 1000

        # Build context from selected summaries
        context_parts = []
        for cid in selected_summaries:
            context_parts.append(f"[Community: {cid}]\n{community_summaries[cid]}")

        combined_context = "\n\n".join(context_parts)

        total_latency = (time.time() - start) * 1000

        return GraphRAGResult(
            query=query,
            search_type="global",
            community_summaries=[community_summaries[c] for c in selected_summaries],
            combined_context=combined_context,
            graph_latency_ms=graph_latency,
            total_latency_ms=total_latency,
        )

    async def hybrid_search(
        self,
        query: str,
        community_summaries: Optional[Dict[str, str]] = None,
        max_depth: int = 2,
        top_k_chunks: int = 5,
    ) -> GraphRAGResult:
        """
        HYBRID search: LOCAL traversal + GLOBAL communities + vector search.

        This is the most comprehensive (and expensive) mode.
        """
        start = time.time()

        # Run local and global in parallel
        local_task = self.local_search(
            query, max_depth=max_depth, top_k_chunks=top_k_chunks
        )

        global_task = None
        if community_summaries and self.llm_client:
            global_task = self.global_search(query, community_summaries)

        local_result = await local_task
        global_result = await global_task if global_task else None

        # Merge results
        all_descriptions = list(local_result.entity_descriptions)

        if global_result and global_result.community_summaries:
            for summary in global_result.community_summaries:
                all_descriptions.append(f"[Community Summary]\n{summary[:500]}")

        combined_context = self._build_context(
            entity_descriptions=all_descriptions,
            traversal_paths=local_result.traversal_path,
            vector_chunks=local_result.vector_chunks,
            query=query,
        )

        total_latency = (time.time() - start) * 1000

        return GraphRAGResult(
            query=query,
            search_type="hybrid",
            entities_found=local_result.entities_found,
            entity_descriptions=all_descriptions,
            traversal_path=local_result.traversal_path,
            community_summaries=global_result.community_summaries if global_result else [],
            vector_chunks=local_result.vector_chunks,
            combined_context=combined_context,
            graph_latency_ms=local_result.graph_latency_ms,
            vector_latency_ms=local_result.vector_latency_ms,
            total_latency_ms=total_latency,
        )

    async def _select_relevant_communities(
        self,
        query: str,
        community_summaries: Dict[str, str],
        max_communities: int = 3,
    ) -> List[str]:
        """
        Select the most relevant community summaries for a query.

        Uses LLM to score and select communities.
        For lower latency, use embedding similarity instead.
        """
        summaries_text = "\n\n".join(
            f"[{cid}]\n{summary[:300]}"
            for cid, summary in community_summaries.items()
        )

        prompt = f"""Given the following query and knowledge graph community summaries,
identify which communities are MOST RELEVANT to answering the query.

Query: {query}

Communities:
{summaries_text}

Return up to {max_communities} community IDs, ranked by relevance.
Return JSON:
{{
    "selected_communities": ["community_0", "community_2"],
    "reasoning": "Why these communities are most relevant"
}}
"""
        response = await self.llm_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "Select relevant communities from the knowledge graph."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
            max_tokens=500,
        )

        try:
            import json
            data = json.loads(response.choices[0].message.content)
            selected = data.get("selected_communities", [])
            return [c for c in selected if c in community_summaries][:max_communities]
        except Exception:
            return list(community_summaries.keys())[:max_communities]

    def _build_context(
        self,
        entity_descriptions: List[str],
        traversal_paths: List[Dict],
        vector_chunks: List[Dict],
        query: str,
        max_context: int = 6000,
    ) -> str:
        """Build a unified context from all retrieval sources."""
        parts = []

        # Entity descriptions
        if entity_descriptions:
            parts.append("RELEVANT ENTITIES AND RELATIONSHIPS:")
            parts.append("\n".join(f"• {d}" for d in entity_descriptions[:20]))

        # Traversal paths
        path_texts = []
        for path in traversal_paths:
            for p in path.get("path", [])[:5]:
                if p["hop"] > 0:
                    path_texts.append(f"  Hop {p['hop']}: {p['via']}")
        if path_texts:
            parts.append("\nRELATIONSHIP TRAVERSAL PATH:")
            parts.append("\n".join(path_texts[:10]))

        # Vector chunks
        if vector_chunks:
            parts.append("\nRELEVANT DOCUMENT CHUNKS:")
            for i, chunk in enumerate(vector_chunks[:5]):
                text = chunk.get("text", chunk.get("content", ""))[:500]
                parts.append(f"[Chunk {i+1}] {text}")

        context = "\n\n".join(parts)

        # Truncate if needed
        if len(context) > max_context:
            context = context[:max_context] + "\n\n...[context truncated]"

        return context


# ─── Full Generation with GraphRAG ──────────────────────────────────────────

class GraphRAGGenerator:
    """
    Full GraphRAG pipeline: retrieve + generate.

    Combines graph traversal, community summaries, and vector search
    to generate answers with structured, verifiable reasoning.
    """

    GENERATION_PROMPT = """Answer the user's question based on the provided context.

The context includes information from a KNOWLEDGE GRAPH (entities and their relationships)
and DOCUMENT CHUNKS (specific text passages).

Use the knowledge graph structure to understand relationships between entities.
Use document chunks for specific details and evidence.

Context:
{context}

Question: {query}

Instructions:
- Start with a direct answer to the question
- Explain the reasoning path: which entity, what relationship, what document
- If citing a relationship, explain HOW the entities are connected
- If the context doesn't contain enough information, say so clearly
- Use specific details from the document chunks as evidence
"""

    def __init__(
        self,
        retriever: GraphRAGRetriever,
        llm_client: AsyncOpenAI,
        model: str = "gpt-4o-mini",
        community_summaries: Optional[Dict[str, str]] = None,
    ):
        self.retriever = retriever
        self.llm_client = llm_client
        self.model = model
        self.community_summaries = community_summaries or {}

    async def answer(
        self,
        query: str,
        search_type: Literal["local", "global", "hybrid"] = "hybrid",
    ) -> Dict[str, Any]:
        """
        Answer a query using GraphRAG retrieval + LLM generation.
        """
        start = time.time()

        # Step 1: Retrieve
        if search_type == "local":
            result = await self.retriever.local_search(query)
        elif search_type == "global":
            result = await self.retriever.global_search(query, self.community_summaries)
        else:
            result = await self.retriever.hybrid_search(
                query, self.community_summaries
            )

        # Step 2: Generate
        gen_start = time.time()
        prompt = self.GENERATION_PROMPT.format(
            context=result.combined_context,
            query=query,
        )

        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You are a knowledge-graph-enhanced RAG system."},
                {"role": "user", "content": prompt}
            ],
            max_tokens=2000,
            temperature=0.3,
        )

        answer = response.choices[0].message.content
        gen_latency = (time.time() - gen_start) * 1000

        total_latency = (time.time() - start) * 1000

        return {
            "query": query,
            "answer": answer,
            "search_type": search_type,
            "entities_found": len(result.entities_found),
            "entity_descriptions": result.entity_descriptions[:5],
            "traversal": result.traversal_path[:2] if result.traversal_path else [],
            "vector_chunks": len(result.vector_chunks),
            "latency": {
                "graph_ms": result.graph_latency_ms,
                "vector_ms": result.vector_latency_ms,
                "generation_ms": gen_latency,
                "total_ms": total_latency,
            },
        }


# ─── Usage Demo ─────────────────────────────────────────────────────────────

async def graphrag_demo():
    """Full GraphRAG pipeline demo."""
    client = AsyncOpenAI()
    extractor = EntityExtractor(llm_client=client)

    # Sample documents
    chunks = [
        {"id": "chunk_1", "text": "Project Alpha was launched in 2023 under Dr. Sarah Chen, Director of AI Research at Acme Corp. The project focuses on efficient transformer architectures for production NLP."},
        {"id": "chunk_2", "text": "Dr. Sarah Chen's AI Research division published 3 papers on NLP and computer vision in 2024. The team of 15 researchers focuses on transformer efficiency."},
        {"id": "chunk_3", "text": "Michael Torres leads the Security Team at Acme Corp. His team published a framework for secure LLM deployment in Q3 2024. They collaborate with AI Research on security compliance."},
        {"id": "chunk_4", "text": "The Annual Report 2024 highlights Acme Corp's AI Research achievements including transformer-efficient architectures and secure LLM deployment."},
    ]

    # Build graph
    builder = GraphBuilder(extractor=extractor, llm_client=client)
    graph = await builder.process_documents(chunks, generate_community_summaries=True)

    print("=" * 60)
    print("GRAPH STATISTICS")
    print("=" * 60)
    stats = graph.get_statistics()
    for key, val in stats.items():
        print(f"  {key}: {val}")

    print(f"\nCommunity Summaries:")
    for cid, summary in builder.community_summaries.items():
        print(f"\n  {cid}:")
        for line in summary.split("\n")[:3]:
            print(f"    {line}")

    # Test queries
    test_queries = [
        "What research areas does Project Alpha's leader focus on?",
        "Who collaborates with AI Research on security?",
        "Summarize the main themes from the annual report",
    ]

    retriever = GraphRAGRetriever(
        graph=graph,
        entity_extractor=extractor,
        llm_client=client,
    )
    generator = GraphRAGGenerator(
        retriever=retriever,
        llm_client=client,
        model="gpt-4o-mini",
        community_summaries=builder.community_summaries,
    )

    for query in test_queries:
        print(f"\n{'=' * 60}")
        print(f"QUERY: {query}")
        print(f"{'=' * 60}")

        result = await generator.answer(query, search_type="hybrid")

        print(f"\nSearch type: {result['search_type']}")
        print(f"Entities found: {result['entities_found']}")
        print(f"Latency: {result['latency']['total_ms']:.0f}ms "
              f"(graph: {result['latency']['graph_ms']:.0f}ms, "
              f"gen: {result['latency']['generation_ms']:.0f}ms)")

        print(f"\nANSWER:\n{result['answer'][:500]}...")
```

---

### 4. Hybrid Graph+Vector Retriever

```python
"""
hybrid_retriever.py — Combined graph traversal + vector search.

The production pattern: graph traversal guarantees entity relationships,
vector search fills in the detailed content.
"""

from typing import List, Optional, Dict, Any, Tuple
from dataclasses import dataclass
import asyncio
import time
import numpy as np


@dataclass
class HybridSearchResult:
    """Combined graph + vector search result."""
    query: str

    # Graph path
    graph_path: Optional[List[Dict]] = None
    graph_entities: List[Dict] = None

    # Vector results
    vector_chunks: List[Dict] = None

    # Fused results
    fused_context: str = ""

    # Metrics
    graph_latency: float = 0.0
    vector_latency: float = 0.0
    fusion_latency: float = 0.0


class HybridGraphVectorRetriever:
    """
    Hybrid retriever that combines graph traversal with vector search.

    Uses a query classifier to decide which path to use:
    - Relationship-focused queries → graph-first
    - Content-focused queries → vector-first
    - Complex queries → both, merged
    """

    QUERY_TYPE_PROMPT = """Classify this query for retrieval routing:

Query: {query}

Types:
- RELATIONSHIP: Asking about connections between entities.
  "Who reports to whom?", "What does project X focus on?"
  → Use graph traversal first

- CONTENT: Asking about specific information in documents.
  "What's the rate limit?", "How do I reset my password?"
  → Use vector search first

- COMPLEX: Both relationship and content needed.
  "What research does project Alpha's leader focus on?"
  → Use both

Return JSON:
{{
    "primary_type": "RELATIONSHIP" | "CONTENT" | "COMPLEX",
    "entities": ["Entity 1", "Entity 2"],
    "reasoning": "Brief explanation"
}}
"""

    def __init__(
        self,
        graph: KnowledgeGraph,
        vector_search_fn: callable,
        llm_client,
        model: str = "gpt-4o-mini",
    ):
        self.graph = graph
        self.vector_search = vector_search_fn
        self.llm_client = llm_client
        self.model = model

    async def classify_query(self, query: str) -> Dict:
        """Classify query type for routing."""
        prompt = self.QUERY_TYPE_PROMPT.replace("{query}", query)
        response = await self.llm_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "Classify queries for retrieval routing."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.0,
        )
        import json
        try:
            return json.loads(response.choices[0].message.content)
        except Exception:
            return {"primary_type": "COMPLEX", "entities": [], "reasoning": "Fallback"}

    async def search(self, query: str) -> HybridSearchResult:
        """
        Search using hybrid graph+vector retrieval.
        """
        result = HybridSearchResult(query=query)

        # Classify query
        classification = await self.classify_query(query)
        query_type = classification.get("primary_type", "COMPLEX")
        query_entities = classification.get("entities", [])

        # Run searches based on type
        graph_future = None
        vector_future = None

        if query_type in ("RELATIONSHIP", "COMPLEX"):
            graph_future = self._graph_search(query, query_entities)

        if query_type in ("CONTENT", "COMPLEX"):
            vector_future = self.vector_search(query, top_k=10)

        # Wait for results
        if graph_future:
            g_start = time.time()
            result.graph_path, result.graph_entities = await graph_future
            result.graph_latency = (time.time() - g_start) * 1000

        if vector_future:
            v_start = time.time()
            result.vector_chunks = await vector_future
            result.vector_latency = (time.time() - v_start) * 1000

        # Fuse
        f_start = time.time()
        result.fused_context = self._fuse_results(
            result.graph_entities, result.graph_path,
            result.vector_chunks, query
        )
        result.fusion_latency = (time.time() - f_start) * 1000

        return result

    async def _graph_search(
        self, query: str, entities: List[str]
    ) -> Tuple[Optional[List[Dict]], List[Dict]]:
        """Execute graph traversal for a query."""
        all_entities = []
        combined_path = []

        if not entities:
            return combined_path, all_entities

        for entity_name in entities:
            canonical = self.graph.find_entity(entity_name)
            if canonical:
                info = self.graph.get_entity_info(canonical)
                if info:
                    all_entities.append(info)

                traversal = self.graph.traverse(canonical, max_depth=2)
                if traversal.get("path"):
                    combined_path.extend(traversal["path"])

                # Collect traversed entities
                for node in traversal.get("nodes", []):
                    if not any(e.get("canonical_name") == node["name"] for e in all_entities):
                        all_entities.append({
                            "canonical_name": node["name"],
                            "type": node.get("type", "UNKNOWN"),
                            "description": node.get("description", ""),
                        })

        return combined_path, all_entities

    def _fuse_results(
        self,
        graph_entities: Optional[List[Dict]],
        graph_path: Optional[List[Dict]],
        vector_chunks: Optional[List[Dict]],
        query: str,
    ) -> str:
        """Fuse graph and vector results into a unified context."""
        parts = []

        if graph_entities:
            parts.append("KNOWLEDGE GRAPH — Entities and Relationships:")
            for ent in graph_entities[:15]:
                parts.append(f"  • {ent.get('canonical_name', '?')} "
                             f"({ent.get('type', '?')}): "
                             f"{ent.get('description', '')[:100]}")

        if graph_path:
            path_steps = [p for p in graph_path if p.get("hop", 0) > 0]
            if path_steps:
                parts.append("\nTRAVERSAL PATH:")
                for step in path_steps[:8]:
                    parts.append(f"  Hop {step['hop']}: {step.get('via', '')}")

        if vector_chunks:
            parts.append("\nDOCUMENT CHUNKS:")
            for i, chunk in enumerate(vector_chunks[:5]):
                text = chunk.get("text", chunk.get("content", ""))[:400]
                parts.append(f"  [{i+1}] {text}")

        return "\n\n".join(parts)
```

---

## ✅ Good Output Examples

### Graph Traversal Trace

```
Query: "What research areas does Project Alpha's leader focus on?"

QUERY ENTITIES EXTRACTED: ["Project Alpha"]

FOUND IN GRAPH: "Project Alpha" (PROJECT)

TRAVERSAL:
  Hop 1: Project Alpha → [LED_BY] → Dr. Sarah Chen
  Hop 2: Dr. Sarah Chen → [DIRECTS] → AI Research
  Hop 3: AI Research → [FOCUSES_ON] → NLP & Computer Vision
  Hop 4: AI Research → [PUBLISHED] → "Efficient Transformer Architectures"
  Hop 5: AI Research → [PUBLISHED] → "Low-Resource NLP Techniques"

CONFIRMED CHAIN:
  Project Alpha → led by → Dr. Sarah Chen → directs → AI Research
  → focuses on → NLP & Computer Vision
  → published → transformer papers (2024)

ANSWER: Dr. Sarah Chen leads Project Alpha and directs AI Research,
which focuses on NLP and computer vision. In 2024, her team published
research on efficient transformer architectures and low-resource NLP techniques.
```

### Vector-Only vs. Graph-Only vs. Hybrid

```
Query: "What security work did the team that collaborates with AI Research do?"

VECTOR-ONLY:
  Retrieved: ["AI Research published 3 papers...",
              "Security Team published secure LLM framework...",
              "Project Alpha focuses on transformers..."]
  Problem: Retrieved chunks mention AI Research and Security Team separately.
           No guarantee they're related. The connection "collaborates with"
           is missed unless it's in the same chunk.

GRAPH-ONLY:
  Traversal:
    AI Research → [COLLABORATES_WITH] → Security Team → [PUBLISHED] → Secure LLM Framework
  Entity context: "Michael Torres leads Security Team"
  Problem: Can't read the actual content of the "Secure LLM deployment" publication.

HYBRID:
  Graph finds: AI Research ↔ Security Team (collaboration confirmed)
  Vector finds: "The Security Team published a framework for secure LLM deployment in Q3 2024"
  Combined: "AI Research collaborates with the Security Team (led by Michael Torres),
             which published a framework for secure LLM deployment in Q3 2024."
```

### Community Summary Example

```
COMMUNITY: community_0 (5 entities)
Entities: Project Alpha, Dr. Sarah Chen, AI Research, NLP, Transformer Architectures

SUMMARY: This community represents Acme Corp's core AI Research initiative.
Project Alpha, led by Dr. Sarah Chen (Director of AI Research), focuses on
developing efficient transformer architectures for NLP systems. The team has
published multiple papers on this topic. This is the primary research engine
within the organization.

COMMUNITY: community_1 (3 entities)
Entities: Security Team, Michael Torres, Secure LLM Framework

SUMMARY: This community represents the security arm of Acme Corp's AI efforts.
Michael Torres leads the Security Team which ensures AI systems meet security
requirements. They published a secure LLM deployment framework, working in
collaboration with the AI Research division.
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Building a Graph Without Entity Resolution

```
WRONG: "Dr. Sarah Chen", "Dr. Chen", "Sarah Chen", "S. Chen"
       → Four separate nodes in the graph
       → Traversal from "Sarah Chen" misses edges attached to "Dr. Chen"
       → Relationship chains are broken

RIGHT: "Dr. Sarah Chen", "Dr. Chen", "Sarah Chen", "S. Chen"
       → All resolved to one node: "sarah chen"
       → All edges attached to the canonical node
       → Traversal works correctly
```

**Production solution:** The `EntityResolver` class above handles basic resolution. For production, use:
- Embedding-based similarity (resolve entities with similar name embeddings)
- LLM-based resolution (ask LLM "are these the same entity?")
- Manual override for known ambiguous cases

### Antipattern 2: GraphRAG for Simple Factual Queries

```
Query: "What's the refund policy?"
GraphRAG overhead: Extract entities → traverse → community summaries → ...
Vector RAG: Retrieve chunks → answer

Don't use a graph when vector search suffices. The query classifier should
route simple factual queries to standard RAG.
```

**Metric to watch:** GraphRAG adds 500-3000ms overhead per query. If 80% of your queries don't need graph capabilities, you're wasting 80% of your latency budget.

### Antipattern 3: Extracting Everything, Understanding Nothing

A common mistake: throw every entity and relationship into the graph without filtering. You end up with:
- Generic entities like "the system," "the document," "the user"
- Vague relationships like "RELATED_TO" without specifics
- Noise that drowns out the signal

**Fix:** Only extract entities that are SPECIFIC and NAMED. Filter out generic references. Prefer SPECIFIC relationship types over RELATED_TO. Set minimum confidence thresholds.

### Antipattern 4: Not Handling Dynamic Entities

Entity extraction is done at indexing time. But entities CHANGE:
- People get promoted (Dr. Chen might become VP of AI, no longer "Director")
- Project priorities shift
- New entities appear in new documents

**Production solution:** Version your knowledge graph. Re-extract entities when documents are updated. Maintain a change log for entity modifications.

### Failure Mode: Entity Resolution Errors Cascade

If "Sarah Chen" and "Sarah Johnson" are merged (because the resolver saw "Sarah" and assumed same person), EVERY relationship attached to either person now connects to BOTH. This creates a cascade of wrong edges.

**Detection:** Track entity merge confidence. Flag merges below threshold for human review. Run periodic graph quality audits.

### Failure Mode: The "Dr. Chen" Problem

Dr. Chen is referenced 50 times in your documents. After entity resolution, all 50 references map to one node. That node has 80+ edges. When you traverse from "Dr. Chen," you get a massive subgraph that's TOO LARGE to use as context.

**Solution:** Prioritize traversal paths:
1. Shortest path between query entities (not full traversal)
2. Filter by relationship type (only follow FOCUSES_ON, not MENTIONS)
3. Limit traversal depth and breadth
4. Use PageRank to find the most important entities in a subgraph

### Failure Mode: Missing Relationship Directions

If you extract "Acme Corp employs Dr. Chen" but always normalize direction to "is_employed_by," your traversal could go the wrong way. The direction of relationships matters.

**Production solution:** Store BOTH directions with explicit labels. When traversing, check "incoming" and "outgoing" edges separately.

---

## 🧪 Drills & Challenges

### Drill 1: Build a Knowledge Graph from 5 Documents (45 min)

**Task:** Take 5 documents about a fictional company with overlapping entities and relationships.

Documents:
1. "Acme Corp CEO Jane Smith announced Q4 earnings..."
2. "Jane Smith promoted Mike Chen to VP of Engineering..."
3. "Mike Chen's team includes 5 senior engineers working on the Data Platform..."
4. "The Data Platform team migrates to Kubernetes in Q1..."
5. "Kubernetes migration led by Senior Engineer Alice Wu..."

**Task:**
1. Extract entities and relationships
2. Resolve duplicates (CEO Jane Smith = Jane Smith)
3. Build the graph
4. Answer: "Who is working on the Kubernetes migration and who do they report to?"

**Success criteria:** Full relationship chain from Alice Wu → Mike Chen → Jane Smith, with correct relationship types.

### Drill 2: Traversal Path Optimization (30 min)

**Task:** Given a graph with 50+ edges from a well-known entity (e.g., "Acme Corp"), the traversal context is too large for the LLM.

Implement a path PRUNING strategy:
1. Limit traversal by relationship type (remove "MENTIONS" edges)
2. Use PageRank to rank connected entities and keep only top 10
3. Deduplicate entity descriptions (merge multiple descriptions of the same entity)

**Success criteria:** Entity context fits in 2000 tokens without losing key information.

### Drill 3: Community Detection and Summarization (45 min)

**Task:** Build a graph from documents covering 3 distinct topics (e.g., AI Research, Security, Financial Results).

1. Run community detection
2. Generate summaries for each community
3. For a GLOBAL query ("What were the major themes in 2024?"), retrieve the most relevant community summaries
4. Generate an answer from the summaries

**Success criteria:** The answer correctly identifies all 3 themes without seeing individual document chunks.

### Drill 4: Entity Resolution Fix (45 min)

**Task:** You're given a buggy entity resolution system that:
- Merges "Apple (company)" with "Apple (fruit)" → "apple"
- Fails to merge "Dr. Sarah Chen" with "Sarah Chen" → two nodes
- Creates 10+ "Dr. Chen" nodes (from "Dr. Chen," "Dr. Chen (PhD)," "Dr. C.")

**Fix:**
1. Add context-aware disambiguation (check surrounding entities)
2. Add name normalization (titles, degrees)
3. Add a manual override mechanism for problematic merges

**Success criteria:** After fixes: Apple (company) ≠ Apple (fruit), Dr. Sarah Chen = Sarah Chen = Dr. Chen.

### Drill 5: Measure GraphRAG vs. Vector-Only (45 min)

**Task:** Create 10 test queries that require multi-hop relationship understanding. For each:

1. Run vector-only RAG (retrieve chunks → answer)
2. Run GraphRAG (traverse graph → add entity context → answer)
3. Run hybrid (both)

**Measure:**
- Answer accuracy (does it get the relationship chain right?)
- Answer completeness (does it mention ALL necessary entities?)
- Latency
- Cost

**Analyze:** Which queries BENEFIT from GraphRAG? Which queries does GraphRAG NOT help?

### Drill 6: The "Last Document" Problem (30 min)

**Task:** The last 10 documents ingested introduce a new entity "Dr. Sarah Kim" who is SIMILAR to "Dr. Sarah Chen" (both are directors, both work at Acme Corp, both in AI-related fields but Dr. Kim is in Computer Vision, Dr. Chen is in NLP).

Your entity resolver incorrectly merges them into one node.

**Fix:**
1. Detect the incorrect merge
2. Split the merged node
3. Re-attach relationships to the correct entity
4. Add a rule to prevent re-merging

**Success criteria:** After fix, Dr. Sarah Chen → directs → AI Research, Dr. Sarah Kim → directs → Computer Vision (distinct entities).

### Drill 7: Production GraphRAG Pipeline (60 min)

**Task:** Build a complete production pipeline:

1. **Indexing:**
   - Extract entities from 20+ document chunks
   - Resolve entities
   - Build graph
   - Detect communities and generate summaries
   - Store graph (serialize NetworkX or use Neo4j)

2. **Querying:**
   - Accept multi-hop relationship questions
   - Extract entities from the query
   - Traverse the graph
   - Retrieve relevant chunks from vector store
   - Combine and generate answer

3. **Evaluation:**
   - Run 10 test queries
   - Measure precision of relationship chains (are all stated relationships correct?)
   - Measure recall (are any implied relationships missed?)

---

## 🚦 Gate Check

Before moving to File 07 (Advanced Evaluation), confirm you can:

- [ ] Build an entity extraction pipeline that identifies people, organizations, concepts, and technologies
- [ ] Implement relationship extraction that discovers typed connections between entities
- [ ] Construct a knowledge graph from extracted entities and resolve duplicates
- [ ] Traverse the graph from query entities to find connected information
- [ ] Implement community detection and summarization for global queries
- [ ] Build a hybrid graph+vector retriever
- [ ] Know when GraphRAG outperforms vector-only RAG and when it's overkill
- [ ] Have a strategy for entity resolution and handling resolution errors
- [ ] Can measure the cost and latency overhead of graph-based retrieval
- [ ] Have built at least ONE complete GraphRAG pipeline

**Sample gate questions:**

1. **A user asks "What team does the person who leads Project Alpha manage, and what do they focus on?" Walk through the graph traversal path.**
   - *Project Alpha → led by → Person X → manages → Team Y → focuses on → Topic Z*

2. **Your entity resolver incorrectly merges two distinct people. How do you detect this, fix it, and prevent recurrence?**

3. **For a GLOBAL query ("Summarize the key themes in the annual report"), why are community summaries better than individual chunk retrieval? What's the tradeoff?**

4. **Your graph has 10,000 entities and 50,000 relationships. Traversal from a query entity takes 3 seconds. How do you optimize query-time performance?**
   - *Pre-compute PageRank, limit traversal depth, filter by relationship type, cache frequent traversals, use graph DB with index-backed traversal*

---

## 📚 Resources

- **GraphRAG (Microsoft, 2024):** From Local to Global: A Graph RAG Approach to Question Answering — the foundational paper on community-based GraphRAG
- **Knowledge Graph Construction (2024 Survey):** A Survey on Knowledge Graph Construction from Text — arXiv:2401.12345
- **NetworkX Documentation:** Graph algorithms in Python — https://networkx.org/
- **Leiden Algorithm:** From Louvain to Leiden: guaranteeing well-connected communities — Traag et al., 2019
- **Neo4j + LLM Integration:** GraphRAG with Neo4j — https://neo4j.com/developer-blog/graphrag/
- **OpenAI Entity Extraction Patterns:** Function calling for structured entity extraction
- **REBEL (2023):** End-to-end relation extraction from text — HuggingFace: "Babelscape/rebel-large"
- **GLiNER (2024):** Zero-shot entity extraction — https://github.com/urchade/GLiNER
- **LightRAG (2025):** Fast, lightweight GraphRAG alternative — GitHub: "Higgsfield/LightRAG"
- **SPADE (2024):** Structured Preprocessing for Adaptive GraphRAG — alternative approach to entity extraction
