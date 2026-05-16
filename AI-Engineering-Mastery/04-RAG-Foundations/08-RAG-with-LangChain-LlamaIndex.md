# RAG with LangChain & LlamaIndex

## Why Use a Framework for RAG

You built RAG from scratch. Now you know what's happening under the hood. Frameworks handle the boilerplate:

| Aspect | Raw Python | Framework |
|--------|-----------|-----------|
| Document loading | Manual parsing per format | 100+ built-in loaders |
| Chunking | Write your own splitter | Configurable splitters |
| Embedding | Manual API calls | One-line embed + index |
| Retrieval | Manual vector search | Built-in retrievers |
| Prompt building | String formatting | Template systems |
| Evaluation | Manual metrics | Built-in eval frameworks |

## LangChain RAG

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_qdrant import QdrantVectorStore
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate

# 1. Load documents
loader = PyPDFLoader("document.pdf")
docs = loader.load()

# 2. Chunk
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# 3. Embed and store
vector_store = QdrantVectorStore.from_documents(
    chunks,
    embedding=OpenAIEmbeddings(),
    location=":memory:",  # or ":memory:" for testing
    collection_name="my_docs",
)

# 4. Build RAG chain
llm = ChatOpenAI(model="gpt-4o-mini")
retriever = vector_store.as_retriever(search_kwargs={"k": 5})

prompt = ChatPromptTemplate.from_template(
    "Answer based on context:\n{context}\nQuestion: {input}"
)
chain = create_stuff_documents_chain(llm, prompt)
rag_chain = create_retrieval_chain(retriever, chain)

# 5. Query
result = rag_chain.invoke({"input": "What is this document about?"})
print(result["answer"])
```

## LlamaIndex RAG

```python
from llama_index.core import (
    VectorStoreIndex, SimpleDirectoryReader, Settings
)
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
from llama_index.core.postprocessor import SimilarityPostprocessor
from llama_index.core.query_engine import RetrieverQueryEngine

# Configure
Settings.llm = OpenAI(model="gpt-4o-mini")
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# 1. Load + chunk + embed in one shot
documents = SimpleDirectoryReader("./data").load_data()
index = VectorStoreIndex.from_documents(documents)

# 2. Build query engine with post-processing
query_engine = index.as_query_engine(
    similarity_top_k=5,
    node_postprocessors=[
        SimilarityPostprocessor(similarity_cutoff=0.7),
    ],
)

# 3. Query
response = query_engine.query("What is this document about?")
print(response)
```

## Framework Comparison for RAG

| Feature | LangChain | LlamaIndex | Notes |
|---------|-----------|------------|-------|
| Document loaders | 100+ | 100+ | Comparable |
| Chunking strategies | 10+ built-in | 5+ built-in | LangChain more flexible |
| Vector DB integrations | 20+ | 15+ | Comparable |
| Advanced RAG patterns | Via LangGraph | Via workflows | Different approaches |
| Query transformation | Built-in chains | Built-in modules | LlamaIndex cleaner API |
| Evaluation | LangSmith | LlamaIndex evals | Different ecosystems |
| Learning curve | Steeper | Moderate | LangChain more abstractions |

## 🔴 Senior: Framework Trap

```
Junior: "I'll use LangChain because it has everything"
Senior: "I'll use libraries for things that are hard, and raw code for things that are easy"
```

**What to framework vs what to DIY:**

| Layer | Framework | DIY |
|-------|-----------|-----|
| Document loading | ✅ LangChain/LlamaIndex loaders | — |
| Chunking | ✅ LangChain splitters | — |
| Embedding | ✅ Any | ✅ Fine-tuned models |
| Vector DB | ✅ Qdrant client | ✅ In-memory for <100K |
| Retrieval logic | — | ✅ Custom strategies |
| Prompt building | — | ✅ Your templates |
| Reranking | ✅ Cohere/cross-encoder | ✅ Custom |
| Evaluation | ✅ RAGAS/LangSmith | ✅ Custom evals |
