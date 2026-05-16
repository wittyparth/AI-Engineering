# Design Doc Template

For every cornerstone project (and the capstone), write a short design doc before building.

```markdown
# Design Doc: [Project Name]

## 1. Problem Statement
What problem does this solve? Who is it for?

## 2. Architecture

```
[Architecture diagram — ASCII or Mermaid]
```

### Key Components
- Component 1: What it does, why it exists
- Component 2: ...
- ...

## 3. Key Design Decisions

| Decision | Options Considered | Chosen | Rationale |
|----------|-------------------|--------|-----------|
| LLM provider | OpenAI, Anthropic, local | ... | ... |
| Vector DB | Qdrant, Pinecone, pgvector | ... | ... |
| Chunking strategy | Recursive, semantic, hybrid | ... | ... |
| ... | ... | ... | ... |

## 4. Data Flow

```
User → [Step 1] → [Step 2] → [Step 3] → Response
```

## 5. Evaluation Plan

| Metric | Target | How Measured |
|--------|--------|-------------|
| Faithfulness | >0.85 | RAGAS |
| Latency p95 | <3s | Prometheus |
| Cost per query | <$0.05 | Cost tracker |
| ... | ... | ... |

## 6. Failure Modes

| Failure Mode | Mitigation |
|-------------|-----------|
| LLM provider down | Fallback to secondary |
| Vector DB slow | Reduce top-k, scale |
| ... | ... |

## 7. Security Considerations

- Input validation
- Rate limiting
- PII handling
- Authentication

## 8. Open Questions

- [ ] What's the capacity limit?
- [ ] How do we handle conflicting context?
- [ ] ...
```
