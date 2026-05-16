# Drill: RAG with LangChain

**Time**: 30 min  **Difficulty**: Easy

## Task

Rebuild your from-scratch RAG project using LangChain. Compare the two.

```python
# Step 1: Load a document collection (use 10 PDFs or markdown files)
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader("./data/", glob="**/*.md")
docs = loader.load()
print(f"Loaded {len(docs)} documents")

# Step 2: Chunk with different strategies
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# Step 3: Embed and index
from langchain_openai import OpenAIEmbeddings
from langchain_qdrant import QdrantVectorStore

vector_store = QdrantVectorStore.from_documents(
    chunks,
    embedding=OpenAIEmbeddings(),
    location=":memory:",
)

# Step 4: Different retrieval strategies
retriever_basic = vector_store.as_retriever(search_kwargs={"k": 3})
retriever_mmr = vector_store.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 5, "fetch_k": 20, "lambda_mult": 0.5},
)

# Step 5: Build chain
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain

prompt = ChatPromptTemplate.from_template("Answer: {context}\nQ: {input}")
chain = create_retrieval_chain(
    retriever_basic,
    create_stuff_documents_chain(llm, prompt),
)

# Step 6: Query
result = chain.invoke({"input": "Your question"})
```

## Report

Compare LangChain RAG vs your from-scratch implementation:
1. Lines of code (frameworks should win)
2. Flexibility (raw should win)
3. Debugging difficulty (raw is easier to trace)
4. Which would you use in production, and why?

## Challenge

Add these LangChain features to your RAG:
1. **Conversation retriever memory** (multi-turn chat)
2. **Parent document retriever** (retrieve chunks, return full documents)
3. **Self-querying retriever** (extract metadata filters from queries)
