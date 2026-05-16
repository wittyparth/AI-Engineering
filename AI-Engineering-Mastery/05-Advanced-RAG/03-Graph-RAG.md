# Graph RAG

## The Problem

Vector search finds similar documents. It can't follow relationships:
- "Find all employees who work on projects using Python"
- "Which research papers cite this paper AND are in the same field?"

For multi-hop reasoning, you need a graph.

## Architecture

```
Vector DB (similarity search) + Graph DB (relationship traversal)
```

## Knowledge Graph Construction

```python
from neo4j import GraphDatabase

class GraphBuilder:
    def __init__(self, uri, user, password):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))

    def extract_entities_and_relations(self, text: str, llm) -> list[tuple]:
        prompt = f"""Extract entities and their relationships from this text.
Format: (Entity1, RELATIONSHIP, Entity2)

Text: {text}"""
        response = llm.generate(prompt)
        return parse_triples(response)

    def add_to_graph(self, triples: list[tuple]):
        with self.driver.session() as session:
            for subj, rel, obj in triples:
                session.run("""
                    MERGE (a:Entity {name: $subj})
                    MERGE (b:Entity {name: $obj})
                    MERGE (a)-[r:RELATION {type: $rel}]->(b)
                """, subj=subj, rel=rel, obj=obj)
```

## Querying Graph RAG

```python
class GraphRAG:
    def __init__(self, vector_db, graph_driver, embedder, llm):
        self.vector_db = vector_db
        self.graph = graph_driver
        self.embedder = embedder
        self.llm = llm

    async def answer(self, query: str) -> str:
        # 1. Initial vector search
        chunks = await self.vector_db.search(query)

        # 2. Extract entities from query
        entities = await self._extract_entities(query)

        # 3. Traverse graph for related entities
        graph_context = await self._traverse_graph(entities)

        # 4. Combine + generate
        combined_context = chunks + graph_context
        return await self._generate(query, combined_context)

    async def _traverse_graph(self, entities: list[str]) -> list[str]:
        with self.graph.session() as session:
            results = await session.run("""
                MATCH (e:Entity)-[r*1..2]-(related)
                WHERE e.name IN $entities
                RETURN related.name, r.type
            """, entities=entities)
            return [format_result(r) for r in results]
```

## Microsoft's GraphRAG Pattern

Microsoft's approach: build a graph index over documents, then use community detection to answer questions at different levels of abstraction.

```
Documents → Entity Extraction → Knowledge Graph → Community Detection → Summarization
                                                                           ↓
                                                                    Query → Answer
```

## 🔴 Senior: When Graph RAG vs Vector RAG

| Use Case | Best Approach |
|----------|--------------|
| "Find similar docs" | Vector RAG |
| "How does X relate to Y?" | Graph RAG |
| "Summarize this topic" | Vector RAG |
| "Trace the influence chain" | Graph RAG |
| Hybrid questions | Both (graph + vector) |
