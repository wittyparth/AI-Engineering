# Document Ingestion Pipelines

## The Pipeline

```
Source → Parse → Clean → Chunk → Embed → Index → Store
```

## Source Handlers

```python
class DocumentSource(ABC):
    @abstractmethod
    async def fetch(self) -> list[RawDocument]: ...

class PDFSource(DocumentSource):
    async def fetch(self, path: str) -> list[RawDocument]:
        # PDF parsing with metadata extraction
        pass

class WebSource(DocumentSource):
    async def fetch(self, url: str) -> list[RawDocument]:
        # HTML extraction with main content parsing
        pass

class GitHubSource(DocumentSource):
    async def fetch(self, repo: str) -> list[RawDocument]:
        # README, docs, docstrings extraction
        pass
```

## Document Model

```python
class Document(BaseModel):
    id: str  # hash of content
    source: str  # file path, URL, etc.
    content: str  # cleaned text
    metadata: dict  # title, author, date, section, etc.
    chunks: list[Chunk] = []

class Chunk(BaseModel):
    id: str  # doc_id + index
    content: str
    metadata: dict  # parent doc info + position
    embedding: list[float] | None = None
```

## Orchestration with Airflow

```python
from airflow import DAG
from airflow.operators.python import PythonOperator

def ingest_documents():
    sources = discover_sources()
    for source in sources:
        docs = source.fetch()
        cleaned = [clean(d) for d in docs]
        chunked = [chunk(d) for d in cleaned]
        embedded = [embed(c) for c in chunked]
        index(embedded)

dag = DAG(
    "document_ingestion",
    schedule_interval="@daily",
    catchup=False,
)
ingest = PythonOperator(
    task_id="ingest",
    python_callable=ingest_documents,
    dag=dag,
)
```

## 🔴 Senior: Ingestion Failure Modes

1. **Duplicate documents**: Same content, different sources → deduplicate by content hash
2. **Encoding issues**: UTF-8 vs Latin-1 → detect and convert
3. **PDF extraction quality**: Some PDFs produce garbage text → have a quality threshold
4. **Rate limits**: API sources may throttle → implement backoff
5. **Partial failures**: Some docs fail, others succeed → atomic batches
6. **Incremental updates**: Don't re-index everything → track per-document hash
