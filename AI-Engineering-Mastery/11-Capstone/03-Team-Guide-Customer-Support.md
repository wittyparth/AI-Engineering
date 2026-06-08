# 🏗️ Team-Guided Build: AI Customer Support System

> **Your Role:** Junior Engineer — you write all the code.
> **Our Role:** Product Manager + Tech Lead + Senior Engineer + QA — we define everything, review everything, set every standard.
> **Goal:** Ship a production-grade AI Customer Support Agent that competes with Intercom Fin and Zendesk AI.

---

## 📋 Project Brief (Product Manager)

**Product Name:** `SupportAI`
**Market:** AI Customer Support ($15.12B in 2026, growing at ~26% CAGR toward $47.8B by 2030)
**Real competitors:** Intercom Fin (86% resolution rate, $0.99/resolution), Zendesk AI ($200M ARR, 20K customers), Ada (83% resolution, $60K+/year enterprise), Freshdesk Freddy AI
**Problem:** 72% of customers expect immediate resolution. Support teams spend 60%+ of their time on repetitive tier-1 questions (password resets, order status, refund policies). Companies waste $80B/year on contact center labor that could be automated. Existing solutions are expensive, hard to customize, or vendor-locked.

### User Stories

```
As a customer, I want instant answers to common questions (order status, refund policy, hours)
  — and when things get complex, I want a seamless handoff to a human who knows what I already tried
So that I don't wait 30 minutes for a password reset.

As a support agent, I want an AI that triages tickets, drafts replies, and surfaces knowledge articles
So that I spend my time on complex issues instead of copy-pasting the same refund policy.

As a support manager, I want to see resolution rates, CSAT scores, and escalation patterns
So that I can prove ROI and identify training gaps.

As a company, I want the AI to learn from our knowledge base, past tickets, and policies
So that answers stay accurate and consistent with our brand voice.
```

### Acceptance Criteria

```
✅ AI resolves common tier-1 queries autonomously (password reset, order status, policy questions)
✅ Intent detection classifies tickets into 10+ categories with >90% accuracy
✅ Sentiment analysis detects frustrated customers and prioritizes/escalates
✅ Human handoff includes full conversation transcript so agents don't ask customers to repeat
✅ Agent copilot suggests replies based on knowledge base + past resolved tickets
✅ Multi-turn conversations maintain context across messages
✅ Answers include citations to source articles
✅ System cost stays under $0.05 per resolution
✅ All conversations traced for quality monitoring
```

### Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Autonomous resolution rate | >50% of tier-1 tickets | Track AI-resolved vs human-handled |
| Intent accuracy | >90% | Manual audit of 200 classified tickets |
| Customer satisfaction (CSAT) | >4.0/5.0 on AI-handled tickets | Post-conversation survey |
| Average handle time (AHT) | <30 seconds for AI, <3 min for human | From ticket creation to resolution |
| Human handoff satisfaction | >4.0/5.0 | Survey after handoff |
| Cost per resolution | <$0.05 | Total LLM cost / resolutions |
| False escalation rate | <10% | AI sends to human when it could have resolved |

---

## 🏗️ System Architecture (Tech Lead)

### High-Level Design

```
┌──────────────────────────────────────────────────────────────────┐
│                        SupportAI System                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Channels (API/Widget/Email)                                      │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────────┐    ┌──────────────────┐                        │
│  │  Classifier   │───→│  Intent + Senti- │                        │
│  │  (FastAPI)    │    │  ment Detector   │                        │
│  └──────┬───────┘    └────────┬─────────┘                        │
│         │                     │                                    │
│         ▼                     ▼                                    │
│  ┌─────────────────────────────────────────────┐                  │
│  │           Routing Decision                    │                  │
│  │  High confidence → AI Agent                  │                  │
│  │  Medium confidence → AI + KB lookup          │                  │
│  │  Low confidence / escalated → Human          │                  │
│  └──────┬──────────────────────────┬───────────┘                  │
│         │                          │                               │
│         ▼                          ▼                               │
│  ┌──────────────┐          ┌──────────────┐                       │
│  │  AI Agent     │          │  Human Agent │                       │
│  │  (LangGraph)  │          │  Dashboard   │                       │
│  │  - Retrieves  │          │  - Context   │                       │
│  │  - Generates  │          │  - Copilot   │                       │
│  │  - Resolves   │          │  - Drafts    │                       │
│  └──────┬───────┘          └──────┬───────┘                       │
│         │                         │                                │
│         └──────────┬──────────────┘                                │
│                    ▼                                               │
│  ┌─────────────────────────────────────┐                          │
│  │         Knowledge Base (RAG)         │                          │
│  │  - ChromaDB of articles + policies   │                          │
│  │  - Past resolved tickets             │                          │
│  │  - Product documentation             │                          │
│  └─────────────────────────────────────┘                          │
│                                                                   │
│  ┌─────────────────────────────────────┐                          │
│  │         Observability (Langfuse)     │                          │
│  │  - Trace every conversation         │                          │
│  │  - Score resolution quality         │                          │
│  └─────────────────────────────────────┘                          │
└──────────────────────────────────────────────────────────────────┘
```

### Tech Stack

| Component | Choice | Why |
|-----------|--------|-----|
| API Framework | FastAPI | Async. Python. Familiar. |
| Agent Framework | LangGraph | Stateful conversation management, checkpoints |
| LLM | GPT-4o-mini | Cheap ($0.15/1M input), fast, good at intent/sentiment |
| Embeddings | text-embedding-3-small | $0.02/1M tokens, 1536-dim |
| Vector DB | ChromaDB → Qdrant | Start local, scale later |
| Storage | SQLite → PostgreSQL | Ticket history, analytics |
| Deployment | Docker + Railway/Fly | Zero-infra |
| Observability | Langfuse | Open-source tracing + eval |
| CI/CD | GitHub Actions | Eval suite on push |

### Data Model

```python
# supportai/models.py
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class CustomerMessage(BaseModel):
    conversation_id: str
    message_id: str
    customer_id: Optional[str]
    content: str
    channel: str  # web_chat | email | api | slack
    timestamp: datetime
    attachments: list[str] = []

class IntentResult(BaseModel):
    intent: str  # refund_request | password_reset | order_status | account_issue | etc
    confidence: float
    sub_intent: Optional[str]
    entities: dict[str, str] = {}  # {"order_id": "12345", "product": "shoes"}

class SentimentResult(BaseModel):
    sentiment: str  # positive | neutral | negative | frustrated
    urgency: int  # 1-5
    language: str  # ISO code
    needs_escalation: bool

class KnowledgeArticle(BaseModel):
    article_id: str
    title: str
    content: str
    category: str
    tags: list[str]
    last_updated: datetime

class ConversationState(BaseModel):
    conversation_id: str
    customer_id: Optional[str]
    messages: list[CustomerMessage]
    intent: Optional[IntentResult]
    sentiment: Optional[SentimentResult]
    resolution_status: str  # open | ai_resolved | human_resolved | escalated | closed
    resolved_by: Optional[str]
    agent_notes: Optional[str]
    csat_score: Optional[float]
    cost: float = 0.0
    created_at: datetime = datetime.now()
    resolved_at: Optional[datetime]

class AgentCopilotSuggestion(BaseModel):
    suggestion_id: str
    ticket_id: str
    suggested_reply: str
    confidence: float
    source_articles: list[str]
    generated_at: datetime
```

---

## 📅 Sprint Plan (Tech Lead)

### Week 1: Foundation (Days 1-3)

| Day | What to Build | Senior Engineer Note |
|-----|---------------|---------------------|
| 1 | Project setup + ticket ingestion API | FastAPI server that receives support messages |
| 2 | Intent + sentiment detector | Classify every incoming message |
| 3 | Knowledge base RAG pipeline | Index articles, retrieve relevant context |

### Week 2: AI Agent + Human Handoff (Days 4-5)

| Day | What to Build | Senior Engineer Note |
|-----|---------------|---------------------|
| 4 | LangGraph conversation agent + response generation | AI resolves tickets autonomously with citations |
| 5 | Human handoff + agent copilot dashboard | Escalate seamlessly, copilot suggests replies |

### Week 3: Production Polish (Days 6-7)

| Day | What to Build | Senior Engineer Note |
|-----|---------------|---------------------|
| 6 | Feedback loop + continuous improvement | Learn from corrections, track resolution quality |
| 7 | Langfuse observability + eval suite | Trace conversations, benchmark accuracy |

---

## 👨‍💻 Day 1: Project Setup + Ticket Ingestion API

### Problem (Senior Engineer)

We need a FastAPI server that accepts customer support messages from multiple channels (API, webhook, email) and routes them into a unified conversation system. This is the entry point for every support interaction.

### What to Build

```
supportai/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app + message ingestion
│   ├── config.py            # Settings + env vars
│   ├── models.py            # Pydantic models
│   ├── database.py          # SQLite models + session
│   └── conversations/
│       ├── __init__.py
│       └── store.py         # Conversation CRUD
├── .env
├── requirements.txt
├── Dockerfile
└── README.md
```

### Step-by-Step Instructions

**Step 1.1: Project skeleton**

Create the directory structure. Initialize with `requirements.txt`:

```
fastapi==0.115.0
uvicorn[standard]
pydantic==2.0
pydantic-settings
python-dotenv
httpx
openai
sqlalchemy
aiosqlite
langgraph
chromadb
langfuse
```

**Step 1.2: Config**

```python
# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    openai_api_key: str
    database_url: str = "sqlite+aiosqlite:///supportai.db"
    langfuse_public_key: str = ""
    langfuse_secret_key: str = ""
    langfuse_host: str = "https://cloud.langfuse.com"
    model_name: str = "gpt-4o-mini"
    embedding_model: str = "text-embedding-3-small"
    max_resolution_cost: float = 0.05
    max_conversation_turns: int = 20
    auto_resolve_threshold: float = 0.85  # confidence to resolve without human

    class Config:
        env_file = ".env"

settings = Settings()
```

**Step 1.3: Database models**

```python
# app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Text, Float, DateTime, Boolean
from datetime import datetime
import uuid

engine = create_async_engine(settings.database_url)
async_session = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

class ConversationDB(Base):
    __tablename__ = "conversations"
    
    id: Mapped[str] = mapped_column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    customer_id: Mapped[str] = mapped_column(String, nullable=True)
    channel: Mapped[str] = mapped_column(String, default="api")
    intent: Mapped[str] = mapped_column(String, nullable=True)
    intent_confidence: Mapped[float] = mapped_column(Float, nullable=True)
    sentiment: Mapped[str] = mapped_column(String, nullable=True)
    urgency: Mapped[int] = mapped_column(default=1)
    resolution_status: Mapped[str] = mapped_column(String, default="open")
    resolved_by: Mapped[str] = mapped_column(String, nullable=True)
    csat_score: Mapped[float] = mapped_column(Float, nullable=True)
    cost: Mapped[float] = mapped_column(Float, default=0.0)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)
    resolved_at: Mapped[datetime] = mapped_column(DateTime, nullable=True)

class MessageDB(Base):
    __tablename__ = "messages"
    
    id: Mapped[str] = mapped_column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    conversation_id: Mapped[str] = mapped_column(String, index=True)
    role: Mapped[str] = mapped_column(String)  # customer | ai | human_agent
    content: Mapped[str] = mapped_column(Text)
    metadata_json: Mapped[str] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)

async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def get_session():
    async with async_session() as session:
        yield session
```

**Step 1.4: Conversation store**

```python
# app/conversations/store.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.database import ConversationDB, MessageDB
from datetime import datetime

async def create_conversation(
    session: AsyncSession, customer_id: str, channel: str = "api"
) -> ConversationDB:
    conv = ConversationDB(customer_id=customer_id, channel=channel)
    session.add(conv)
    await session.commit()
    await session.refresh(conv)
    return conv

async def add_message(
    session: AsyncSession,
    conversation_id: str,
    role: str,
    content: str,
    metadata_json: str = None,
) -> MessageDB:
    msg = MessageDB(
        conversation_id=conversation_id,
        role=role,
        content=content,
        metadata_json=metadata_json,
    )
    session.add(msg)
    await session.commit()
    await session.refresh(msg)
    return msg

async def get_conversation(
    session: AsyncSession, conversation_id: str
) -> ConversationDB | None:
    result = await session.execute(
        select(ConversationDB).where(ConversationDB.id == conversation_id)
    )
    return result.scalar_one_or_none()

async def get_conversation_messages(
    session: AsyncSession, conversation_id: str
) -> list[MessageDB]:
    result = await session.execute(
        select(MessageDB)
        .where(MessageDB.conversation_id == conversation_id)
        .order_by(MessageDB.created_at)
    )
    return list(result.scalars().all())

async def update_conversation_status(
    session: AsyncSession,
    conversation_id: str,
    status: str,
    resolved_by: str = None,
):
    conv = await get_conversation(session, conversation_id)
    if conv:
        conv.resolution_status = status
        conv.resolved_by = resolved_by
        if status in ("ai_resolved", "human_resolved"):
            conv.resolved_at = datetime.now()
        await session.commit()

async def set_csat(
    session: AsyncSession, conversation_id: str, score: float
):
    conv = await get_conversation(session, conversation_id)
    if conv:
        conv.csat_score = score
        await session.commit()
```

**Step 1.5: Main app — ticket ingestion API**

```python
# app/main.py
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime
import uuid

from app.config import settings
from app.database import init_db, get_session
from app.conversations.store import (
    create_conversation, add_message, get_conversation,
    get_conversation_messages, update_conversation_status,
)

app = FastAPI(title="SupportAI")

@app.on_event("startup")
async def startup():
    await init_db()

# ── API Models ──

class IncomingMessage(BaseModel):
    conversation_id: str | None = None  # None = new conversation
    customer_id: str | None = None
    content: str
    channel: str = "api"

class MessageResponse(BaseModel):
    conversation_id: str
    message_id: str
    timestamp: datetime

# ── Endpoints ──

@app.post("/api/messages", response_model=MessageResponse)
async def receive_message(
    msg: IncomingMessage,
    session: AsyncSession = Depends(get_session),
):
    """Receive a customer support message. Creates or continues a conversation."""
    
    # New conversation?
    if not msg.conversation_id:
        conv = await create_conversation(
            session, msg.customer_id or "anonymous", msg.channel
        )
        conversation_id = conv.id
    else:
        conv = await get_conversation(session, msg.conversation_id)
        if not conv:
            raise HTTPException(404, "Conversation not found")
        conversation_id = msg.conversation_id
    
    # Store the message
    db_msg = await add_message(
        session, conversation_id, "customer", msg.content
    )
    
    # For Day 1: just echo back. Tomorrow we classify + route.
    print(f"[{conversation_id}] {msg.content[:100]}...")
    
    return MessageResponse(
        conversation_id=conversation_id,
        message_id=db_msg.id,
        timestamp=db_msg.created_at,
    )

@app.get("/api/conversations/{conversation_id}")
async def get_conversation_history(
    conversation_id: str,
    session: AsyncSession = Depends(get_session),
):
    """Get full conversation history."""
    conv = await get_conversation(session, conversation_id)
    if not conv:
        raise HTTPException(404, "Conversation not found")
    messages = await get_conversation_messages(session, conversation_id)
    return {
        "conversation_id": conv.id,
        "status": conv.resolution_status,
        "intent": conv.intent,
        "sentiment": conv.sentiment,
        "messages": [
            {"role": m.role, "content": m.content, "timestamp": m.created_at}
            for m in messages
        ],
    }

@app.get("/health")
async def health():
    return {"status": "ok", "model": settings.model_name}
```

### ✅ Check: Day 1 Gate

```
1. ❓ Can you start the server?   uvicorn app.main:app --reload
2. ❓ Can you send a message?     curl -X POST http://localhost:8000/api/messages \
     -H "Content-Type: application/json" \
     -d '{"content": "I forgot my password", "customer_id": "user_123"}'
3. ❓ Does it return a conversation_id?
4. ❓ Can you retrieve the conversation history?
5. ❓ Send 3 messages — are all stored in order?

If all 5 pass → proceed.
```

---

## 👨‍💻 Day 2: Intent + Sentiment Classifier

### Problem (Senior Engineer)

Raw messages mean nothing. We need to classify every incoming ticket: what does the customer want (intent), how do they feel (sentiment), and how urgent is it? This determines everything — routing, response, escalation.

### What to Build

Create `app/classifier/`:
1. `intent.py` — Classifies message into intent categories with entities
2. `sentiment.py` — Detects sentiment, urgency, escalation signals
3. Wire it into the message pipeline

### Implementation

```python
# app/classifier/intent.py
from openai import AsyncOpenAI
from app.config import settings
from app.models import IntentResult
import json

client = AsyncOpenAI(api_key=settings.openai_api_key)

INTENT_SYSTEM_PROMPT = """You are an intent classifier for customer support.
Classify the customer's message into exactly one of these intents:

- refund_request: Customer wants money back for a purchase
- password_reset: Customer can't log in or forgot credentials
- order_status: Customer wants to know where their order is
- cancel_subscription: Customer wants to stop a recurring payment
- account_issue: Login problems, account locked, profile changes
- product_inquiry: Questions about features, specs, availability
- billing_issue: Wrong charge, payment failed, invoice question
- technical_support: Bug report, error message, integration help
- complaint: Negative feedback, poor experience, escalation
- general_inquiry: Everything else — directions, hours, partnership

Return JSON:
{
  "intent": "refund_request",
  "confidence": 0.95,
  "sub_intent": "late_delivery_refund",
  "entities": {"order_id": "ORD-12345", "amount": "$49.99"},
  "explanation": "Customer explicitly asks for refund and provides order ID"
}

Extract entities like order_id, product_name, amount, email from the text.
If confidence < 0.5, set intent to "general_inquiry".
"""

async def classify_intent(message: str, history: list[str] = None) -> IntentResult:
    """Classify the intent of a customer message."""
    context = ""
    if history:
        context = "Previous messages:\n" + "\n".join(history[-3:]) + "\n\n"
    
    response = await client.chat.completions.create(
        model=settings.model_name,
        messages=[
            {"role": "system", "content": INTENT_SYSTEM_PROMPT},
            {"role": "user", "content": f"{context}Current message: {message}"}
        ],
        response_format={"type": "json_object"},
        temperature=0.0,
        max_tokens=500,
    )
    
    try:
        data = json.loads(response.choices[0].message.content)
        return IntentResult(
            intent=data.get("intent", "general_inquiry"),
            confidence=data.get("confidence", 0.0),
            sub_intent=data.get("sub_intent"),
            entities=data.get("entities", {}),
        )
    except json.JSONDecodeError:
        return IntentResult(intent="general_inquiry", confidence=0.0)
```

```python
# app/classifier/sentiment.py
from openai import AsyncOpenAI
from app.config import settings
from app.models import SentimentResult
import json

client = AsyncOpenAI(api_key=settings.openai_api_key)

SENTIMENT_SYSTEM_PROMPT = """Analyze the customer's emotional state and urgency.
Return JSON:
{
  "sentiment": "frustrated",  // positive | neutral | negative | frustrated | angry
  "urgency": 4,               // 1 (not urgent) to 5 (emergency)
  "language": "en",           // ISO 639-1 code
  "needs_escalation": true,   // true if very angry, threatening, or high urgency
  "signals": ["repeated requests", "all caps", "swearing", "mention of competitor"]
}

Signal escalation when:
- Customer is angry/frustrated AND urgency >= 4
- Customer mentions legal action, BBB, or social media complaint
- Customer has asked 3+ times without resolution
- Message contains threats or abusive language
"""

async def analyze_sentiment(message: str, history: list[str] = None) -> SentimentResult:
    """Analyze customer sentiment and urgency."""
    context = ""
    if history:
        context = "Previous messages:\n" + "\n".join(history[-5:]) + "\n\n"
    
    response = await client.chat.completions.create(
        model=settings.model_name,
        messages=[
            {"role": "system", "content": SENTIMENT_SYSTEM_PROMPT},
            {"role": "user", "content": f"{context}Message: {message}"}
        ],
        response_format={"type": "json_object"},
        temperature=0.0,
        max_tokens=300,
    )
    
    try:
        data = json.loads(response.choices[0].message.content)
        return SentimentResult(
            sentiment=data.get("sentiment", "neutral"),
            urgency=data.get("urgency", 1),
            language=data.get("language", "en"),
            needs_escalation=data.get("needs_escalation", False),
        )
    except json.JSONDecodeError:
        return SentimentResult(sentiment="neutral", urgency=1, language="en", needs_escalation=False)
```

Now wire classification into the message pipeline:

```python
# In app/main.py — add to receive_message:
from app.classifier.intent import classify_intent
from app.classifier.sentiment import analyze_sentiment

@app.post("/api/messages", response_model=MessageResponse)
async def receive_message(
    msg: IncomingMessage,
    session: AsyncSession = Depends(get_session),
):
    # ... existing conversation creation ...
    
    # Store the message
    db_msg = await add_message(session, conversation_id, "customer", msg.content)
    
    # Get conversation history for context
    messages = await get_conversation_messages(session, conversation_id)
    history = [m.content for m in messages[:-1]]  # all except current
    
    # Classify intent + sentiment in parallel
    import asyncio
    intent_result, sentiment_result = await asyncio.gather(
        classify_intent(msg.content, history),
        analyze_sentiment(msg.content, history),
    )
    
    # Update conversation metadata
    conv = await get_conversation(session, conversation_id)
    if conv:
        conv.intent = intent_result.intent
        conv.intent_confidence = intent_result.confidence
        conv.sentiment = sentiment_result.sentiment
        conv.urgency = sentiment_result.urgency
        await session.commit()
    
    print(f"[{conversation_id}] Intent: {intent_result.intent} ({intent_result.confidence:.0%}) | "
          f"Sentiment: {sentiment_result.sentiment} | "
          f"Urgency: {sentiment_result.urgency}/5")
    
    # If high urgency/escalation needed → notify (we'll build routing tomorrow)
    if sentiment_result.needs_escalation:
        print(f"  ⚠️ ESCALATION RECOMMENDED: {sentiment_result.signals}")
    
    return MessageResponse(
        conversation_id=conversation_id,
        message_id=db_msg.id,
        timestamp=db_msg.created_at,
    )
```

### ✅ Check: Day 2 Gate

```
Create test_classifier.py and verify:
1. ❓ "I need a refund for order 12345" → intent=refund_request, entity={order_id}
2. ❓ "I can't log in, this is ridiculous!" → intent=password_reset or account_issue,
     sentiment=frustrated or angry
3. ❓ "What are your store hours?" → intent=general_inquiry, sentiment=neutral
4. ❓ "I've asked THREE TIMES about my refund!!! I'm posting on Twitter"
     → needs_escalation=true, urgency >= 4
5. ❓ "Thanks, that worked perfectly" → sentiment=positive, urgency=1
6. ❓ Can you classify 100 messages and measure accuracy?

If all 6 work → proceed.
```

---

## 👨‍💻 Day 3: Knowledge Base + RAG Pipeline

### Problem (Senior Engineer)

The AI can't answer questions if it doesn't know your products, policies, or processes. We need a knowledge base that the agent can query to find relevant articles, policies, and past solutions.

### What to Build

1. `app/knowledge_base/loader.py` — Load articles from files/database into ChromaDB
2. `app/knowledge_base/retriever.py` — Query relevant articles for a given question
3. `seed_data/` — Sample knowledge articles to index
4. `scripts/seed_kb.py` — Initialize the knowledge base

### Implementation

```python
# app/knowledge_base/retriever.py
import chromadb
from chromadb.config import Settings
from openai import OpenAI
from app.config import settings
from app.models import KnowledgeArticle

client = OpenAI(api_key=settings.openai_api_key)
chroma_client = chromadb.PersistentClient(path="./knowledge_db")

KB_COLLECTION = "support-knowledge-base"

def get_collection():
    return chroma_client.get_or_create_collection(
        name=KB_COLLECTION,
        metadata={"hnsw:space": "cosine"}
    )

def embed_texts(texts: list[str]) -> list[list[float]]:
    response = client.embeddings.create(
        model=settings.embedding_model,
        input=texts
    )
    return [item.embedding for item in response.data]

def seed_knowledge_base(articles: list[KnowledgeArticle]):
    """Index knowledge articles into vector DB."""
    collection = get_collection()
    
    texts = [f"{a.title}\n{a.content}" for a in articles]
    ids = [a.article_id for a in articles]
    metadatas = [{"category": a.category, "tags": ",".join(a.tags)} for a in articles]
    
    # Clear existing
    try:
        existing = collection.get()["ids"]
        if existing:
            collection.delete(existing)
    except Exception:
        pass
    
    embeddings = embed_texts(texts)
    
    collection.add(
        documents=texts,
        embeddings=embeddings,
        ids=ids,
        metadatas=metadatas,
    )
    
    print(f"✓ Indexed {len(articles)} knowledge articles")

def query_knowledge_base(
    query: str, n_results: int = 3, category: str = None
) -> list[dict]:
    """Find most relevant knowledge articles for a query."""
    collection = get_collection()
    
    query_embedding = client.embeddings.create(
        model=settings.embedding_model,
        input=query
    ).data[0].embedding
    
    where = {"category": category} if category else None
    
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=n_results,
        where=where,
    )
    
    articles = []
    if results["documents"]:
        for i, doc in enumerate(results["documents"][0]):
            articles.append({
                "article_id": results["ids"][0][i] if results["ids"] else "",
                "content": doc,
                "score": results["distances"][0][i] if results["distances"] else 0,
                "metadata": results["metadatas"][0][i] if results["metadatas"] else {},
            })
    
    return articles
```

```python
# scripts/seed_kb.py
"""Seed the knowledge base with sample articles."""
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent))

from app.knowledge_base.retriever import seed_knowledge_base
from app.models import KnowledgeArticle

SAMPLE_ARTICLES = [
    KnowledgeArticle(
        article_id="refund-policy",
        title="Refund Policy",
        content="""We offer a 30-day refund policy for all products.
To request a refund:
1. Log into your account
2. Go to Orders → select the order → Request Refund
3. Select a reason and submit
Refunds are processed within 5-7 business days to the original payment method.
Expedited shipping costs are non-refundable.
Digital products (software licenses, e-books) are non-refundable after download.""",
        category="billing",
        tags=["refund", "money back", "return"],
    ),
    KnowledgeArticle(
        article_id="password-reset",
        title="How to Reset Your Password",
        content="""To reset your password:
1. Go to the login page and click 'Forgot Password'
2. Enter your email address
3. Check your inbox for a password reset link (valid for 1 hour)
4. Click the link and enter your new password (min 8 chars, 1 uppercase, 1 number)
If you don't receive the email:
- Check your spam folder
- Wait 5 minutes and try again
- Contact support if still having issues""",
        category="account",
        tags=["password", "login", "forgot password"],
    ),
    KnowledgeArticle(
        article_id="order-tracking",
        title="Track Your Order",
        content="""To track your order:
1. Log into your account
2. Go to Orders → Order History
3. Click 'Track' next to your order
You'll see real-time updates including:
- Order confirmed
- Processing
- Shipped (with tracking number)
- Out for delivery
- Delivered
Standard shipping: 5-8 business days
Express shipping: 2-3 business days
International: 10-15 business days (may be delayed by customs)""",
        category="orders",
        tags=["order status", "tracking", "delivery", "shipping"],
    ),
    KnowledgeArticle(
        article_id="cancel-subscription",
        title="Cancel Your Subscription",
        content="""To cancel your subscription:
1. Go to Settings → Subscription
2. Click 'Cancel Subscription'
3. Select a reason and confirm
Your subscription will remain active until the end of the current billing period.
No further charges will be made.
To request a refund for the current period, see our refund policy.
If you're on an annual plan, you may qualify for a prorated refund.""",
        category="billing",
        tags=["cancel", "subscription", "unsubscribe"],
    ),
    KnowledgeArticle(
        article_id="shipping-policy",
        title="Shipping Policy",
        content="""We ship to all 50 US states and 40+ international countries.
Shipping costs are calculated at checkout based on weight and destination.
Free shipping on orders over $50 (US only).
Processing time: 1-2 business days.
We use USPS, FedEx, and UPS depending on your location.
Tracking is included free on all orders.
PO Box delivery is available via USPS only.
We are not responsible for packages lost after confirmed delivery.""",
        category="orders",
        tags=["shipping", "delivery", "free shipping"],
    ),
    KnowledgeArticle(
        article_id="billing-issue",
        title="Billing Issues and Disputes",
        content="""If you see an incorrect charge on your account:
1. Check your order history for matching transactions
2. Check for pending charges that may resolve automatically
3. If the charge doesn't match any order, contact billing support

Common causes:
- Currency conversion differences
- Sales tax adjustments
- Split payments showing as multiple charges
- Temporary authorization holds (released in 3-5 days)

For disputed charges, please provide the transaction ID, date, and amount.
We'll investigate and respond within 48 hours.""",
        category="billing",
        tags=["billing", "charge", "payment", "dispute"],
    ),
]

if __name__ == "__main__":
    seed_knowledge_base(SAMPLE_ARTICLES)
```

### ✅ Check: Day 3 Gate

```
1. ❓ Run:  python scripts/seed_kb.py → "Indexed X knowledge articles"
2. ❓ Query: "how do I get my money back" → returns refund-policy as top result
3. ❓ Query: "where is my package" → returns order-tracking as top result
4. ❓ Query: "I can't log in" → returns password-reset as top result
5. ❓ Query with filtering by category → only returns articles in that category
6. ❓ Query something unrelated → returns results with lower scores

If all 6 pass → proceed.
```

---

## 👨‍💻 Day 4: LangGraph Conversation Agent + Response Generation

### Problem (Senior Engineer)

Now we connect everything: receive a message → classify intent/sentiment → query knowledge base → generate a response → decide if we can resolve or need human.

### What to Build

1. `app/agent/graph.py` — LangGraph state machine for the support conversation
2. `app/agent/nodes.py` — Individual agent nodes (classify, retrieve, generate, route)
3. Wire it into the API

### Implementation

```python
# app/agent/graph.py
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from openai import AsyncOpenAI
import json
import operator

from app.config import settings
from app.classifier.intent import classify_intent
from app.classifier.sentiment import analyze_sentiment
from app.knowledge_base.retriever import query_knowledge_base

client = AsyncOpenAI(api_key=settings.openai_api_key)

# ── State ──

class SupportState(TypedDict):
    conversation_id: str
    customer_message: str
    message_history: list[str]
    intent: str
    intent_confidence: float
    sentiment: str
    urgency: int
    needs_escalation: bool
    knowledge_articles: list[dict]
    ai_response: str
    response_type: str  # answer | clarify | escalate
    resolved: bool
    citations: list[str]
    cost: float

# ── Nodes ──

async def classify_node(state: SupportState) -> dict:
    """Classify incoming message intent and sentiment."""
    intent_result, sentiment_result = await asyncio.gather(
        classify_intent(state["customer_message"], state.get("message_history")),
        analyze_sentiment(state["customer_message"], state.get("message_history")),
    )
    
    return {
        "intent": intent_result.intent,
        "intent_confidence": intent_result.confidence,
        "sentiment": sentiment_result.sentiment,
        "urgency": sentiment_result.urgency,
        "needs_escalation": sentiment_result.needs_escalation,
    }


async def retrieve_node(state: SupportState) -> dict:
    """Retrieve relevant knowledge articles."""
    query = f"{state['intent']}: {state['customer_message']}"
    articles = query_knowledge_base(query, n_results=3)
    return {"knowledge_articles": articles}


async def generate_node(state: SupportState) -> dict:
    """Generate AI response using knowledge base context."""
    
    # Build context from knowledge articles
    kb_context = ""
    if state.get("knowledge_articles"):
        kb_context = "Relevant knowledge base articles:\n"
        for i, art in enumerate(state["knowledge_articles"], 1):
            kb_context += f"\n[{i}] {art['content']}\n"
    
    system_prompt = f"""You are SupportAI, a helpful customer support agent.
You resolve customer issues accurately and empathetically.

Rules:
1. Answer ONLY using the knowledge base articles provided below
2. If the knowledge base doesn't contain the answer, say "I don't have that information yet" and offer to connect to a human
3. Always include citations to source articles using [1], [2], etc.
4. Be concise but helpful (2-4 sentences typically)
5. Never make up policies or promises
6. If the customer is frustrated, acknowledge their frustration first
7. For refund/cancellation requests, confirm the action before proceeding

Customer intent: {state.get('intent', 'unknown')}
Customer sentiment: {state.get('sentiment', 'neutral')}

{kb_context}
"""
    
    history_text = ""
    if state.get("message_history"):
        history_text = "Conversation history:\n" + "\n".join(
            state["message_history"][-5:]
        ) + "\n\n"
    
    response = await client.chat.completions.create(
        model=settings.model_name,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"{history_text}Customer: {state['customer_message']}"}
        ],
        temperature=0.3,
        max_tokens=500,
    )
    
    ai_reply = response.choices[0].message.content
    
    # Extract citations
    import re
    citations = re.findall(r'\[(\d+)\]', ai_reply)
    
    # Map citations to article IDs
    article_ids = []
    if state.get("knowledge_articles"):
        for c in citations:
            idx = int(c) - 1
            if 0 <= idx < len(state["knowledge_articles"]):
                article_ids.append(state["knowledge_articles"][idx].get("article_id", ""))
    
    return {
        "ai_response": ai_reply,
        "citations": article_ids,
    }


async def router_node(state: SupportState) -> dict:
    """Decide whether to resolve, clarify, or escalate."""
    needs_esc = state.get("needs_escalation", False)
    confidence = state.get("intent_confidence", 0.0)
    
    if needs_esc or confidence < settings.auto_resolve_threshold:
        return {"response_type": "escalate", "resolved": False}
    
    return {"response_type": "answer", "resolved": True}


# ── Build Graph ──

import asyncio

def build_support_graph():
    workflow = StateGraph(SupportState)
    
    workflow.add_node("classify", classify_node)
    workflow.add_node("retrieve", retrieve_node)
    workflow.add_node("generate", generate_node)
    workflow.add_node("route", router_node)
    
    workflow.set_entry_point("classify")
    workflow.add_edge("classify", "retrieve")
    workflow.add_edge("retrieve", "generate")
    workflow.add_edge("generate", "route")
    workflow.add_conditional_edges(
        "route",
        lambda s: s["response_type"],
        {
            "answer": END,
            "escalate": END,
            "clarify": END,
        }
    )
    
    return workflow.compile()

support_graph = build_support_graph()


async def process_message(
    conversation_id: str,
    message: str,
    history: list[str],
) -> SupportState:
    """Process a customer message through the full support pipeline."""
    initial = SupportState(
        conversation_id=conversation_id,
        customer_message=message,
        message_history=history,
        intent="",
        intent_confidence=0.0,
        sentiment="neutral",
        urgency=1,
        needs_escalation=False,
        knowledge_articles=[],
        ai_response="",
        response_type="answer",
        resolved=False,
        citations=[],
        cost=0.0,
    )
    
    result = await support_graph.ainvoke(initial)
    return result
```

Now wire the agent into the API:

```python
# In app/main.py — replace the print with agent processing:
from app.agent.graph import process_message

@app.post("/api/messages", response_model=MessageResponse)
async def receive_message(
    msg: IncomingMessage,
    session: AsyncSession = Depends(get_session),
):
    # ... existing conversation creation + message storage ...
    
    # Get history
    messages = await get_conversation_messages(session, conversation_id)
    history = [f"{m.role}: {m.content}" for m in messages[:-1]]
    
    # Process through AI agent
    agent_result = await process_message(
        conversation_id, msg.content, history
    )
    
    # Update conversation
    conv = await get_conversation(session, conversation_id)
    if conv:
        conv.intent = agent_result["intent"]
        conv.intent_confidence = agent_result["intent_confidence"]
        conv.sentiment = agent_result["sentiment"]
        conv.urgency = agent_result["urgency"]
        
        if agent_result["resolved"]:
            conv.resolution_status = "ai_resolved"
            conv.resolved_by = "SupportAI"
            conv.resolved_at = datetime.now()
        elif agent_result["response_type"] == "escalate":
            conv.resolution_status = "escalated"
        
        await session.commit()
    
    # Store AI response as a message in the conversation
    if agent_result.get("ai_response"):
        await add_message(
            session, conversation_id, "ai", agent_result["ai_response"],
            metadata_json=json.dumps({
                "citations": agent_result.get("citations", []),
                "response_type": agent_result.get("response_type", "answer"),
                "resolved": agent_result.get("resolved", False),
            })
        )
    
    return MessageResponse(
        conversation_id=conversation_id,
        message_id=db_msg.id,
        timestamp=db_msg.created_at,
    )
```

Add a response endpoint so customers can get the AI's reply:

```python
@app.get("/api/conversations/{conversation_id}/latest")
async def get_latest_response(
    conversation_id: str,
    session: AsyncSession = Depends(get_session),
):
    """Get the latest AI response for a conversation."""
    messages = await get_conversation_messages(session, conversation_id)
    ai_messages = [m for m in messages if m.role == "ai"]
    if not ai_messages:
        return {"response": None}
    latest = ai_messages[-1]
    return {
        "response": latest.content,
        "timestamp": latest.created_at,
        "metadata": latest.metadata_json,
    }
```

### ✅ Check: Day 4 Gate

```
1. ❓ Send "I forgot my password" → AI should respond with password reset steps
2. ❓ Send "Where's my order #12345?" → AI should respond with tracking instructions
3. ❓ Send "I WANT A REFUND NOW!!!" → AI should detect frustration, respond empathetically,
     check for escalation
4. ❓ Does the response include citations [1], [2] to knowledge base articles?
5. ❓ Multi-turn: "I need help" → AI asks clarifying → "I can't log in" → AI remembers context
6. ❓ Send "What's the meaning of life?" → AI should say it doesn't know and offer human handoff

If all 6 work → proceed.
```

---

## 👨‍💻 Day 5: Human Handoff + Agent Copilot

### Problem (Senior Engineer)

AI can't handle everything. When escalation happens (low confidence, frustrated customer, complex issue), we need seamless handoff to a human agent — with full context so they don't ask customers to repeat themselves. We also need an agent copilot that suggests replies.

### What to Build

1. `app/handoff/escalate.py` — Escalation logic + context packaging
2. `app/copilot/suggester.py` — Reply suggestion based on ticket context + KB
3. `app/agent_dashboard.py` — Simple dashboard for agents to see escalated tickets
4. Wire escalation into the routing decision

### Implementation

```python
# app/handoff/escalate.py
"""Human handoff with full context preservation."""
from app.conversations.store import get_conversation, get_conversation_messages
from app.models import ConversationState

ESCALATION_REASONS = {
    "low_confidence": "AI confidence below threshold",
    "customer_frustration": "Customer is frustrated/angry (urgency >= 4)",
    "complex_query": "Query requires human judgment or discretion",
    "auth_required": "Customer needs identity verification",
    "customer_request": "Customer explicitly asked for a human",
    "third_turn_escalation": "Customer not satisfied after 3 AI attempts",
}

async def prepare_handoff_context(
    conversation_id: str,
    session,
    reason: str = "low_confidence",
) -> dict:
    """Package full conversation context for human agent handoff."""
    conv = await get_conversation(session, conversation_id)
    messages = await get_conversation_messages(session, conversation_id)
    
    return {
        "conversation_id": conversation_id,
        "customer_id": conv.customer_id if conv else None,
        "escalation_reason": ESCALATION_REASONS.get(reason, reason),
        "intent": conv.intent if conv else None,
        "sentiment": conv.sentiment if conv else None,
        "urgency": conv.urgency if conv else 1,
        "what_ai_tried": [
            m.content for m in messages if m.role == "ai"
        ],
        "full_transcript": [
            {"role": m.role, "content": m.content, "timestamp": m.created_at.isoformat()}
            for m in messages
        ],
        "suggested_action": _suggest_action(conv, messages) if conv else None,
    }

def _suggest_action(conv, messages) -> str:
    """Suggest what the human agent should do based on context."""
    intent = conv.intent if conv else ""
    
    suggestions = {
        "refund_request": "Verify order, check refund eligibility, process refund",
        "password_reset": "Verify identity via security questions, initiate admin reset",
        "account_issue": "Verify identity, check account status in admin panel",
        "billing_issue": "Check transaction logs, verify payment processor status",
        "complaint": "Acknowledge issue, escalate to supervisor if necessary",
    }
    
    return suggestions.get(intent, "Review transcript and respond appropriately")
```

```python
# app/copilot/suggester.py
"""AI copilot that suggests replies to human agents."""
from openai import AsyncOpenAI
from app.config import settings
from app.knowledge_base.retriever import query_knowledge_base
from app.models import AgentCopilotSuggestion
import json

client = AsyncOpenAI(api_key=settings.openai_api_key)

async def suggest_reply(
    ticket_context: dict,
) -> AgentCopilotSuggestion:
    """Generate a suggested reply for a human agent based on ticket context."""
    
    # Retrieve relevant knowledge
    query = ticket_context.get("intent", "") + ": " + (
        ticket_context.get("full_transcript", [{}])[-1].get("content", "")
        if ticket_context.get("full_transcript") else ""
    )
    articles = query_knowledge_base(query, n_results=2)
    kb_context = "\n".join(a["content"] for a in articles)
    
    transcript = "\n".join(
        f"{m['role']}: {m['content']}"
        for m in (ticket_context.get("full_transcript") or [])[-6:]
    )
    
    system_prompt = f"""You are an AI copilot for a customer support agent.
The AI agent was unable to resolve this ticket and it has been escalated to a human.

Ticket context:
- Intent: {ticket_context.get('intent', 'unknown')}
- Sentiment: {ticket_context.get('sentiment', 'neutral')}
- Urgency: {ticket_context.get('urgency', 1)}/5
- What AI already tried: {ticket_context.get('what_ai_tried', [])}

Relevant knowledge base articles:
{kb_context}

Generate a suggested reply for the human agent. Be professional and helpful.
The agent can modify this before sending.
Return JSON:
{{
  "suggested_reply": "the suggested reply text",
  "reasoning": "why this reply is appropriate",
  "agent_notes": "what the agent should check/verify before sending"
}}
"""
    
    response = await client.chat.completions.create(
        model=settings.model_name,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Transcript:\n{transcript}\n\nSuggest a reply."}
        ],
        response_format={"type": "json_object"},
        temperature=0.3,
        max_tokens=500,
    )
    
    try:
        data = json.loads(response.choices[0].message.content)
    except json.JSONDecodeError:
        data = {"suggested_reply": "I'm sorry for the trouble. Let me look into this for you.", "reasoning": "", "agent_notes": ""}
    
    return AgentCopilotSuggestion(
        suggestion_id=str(uuid.uuid4()),
        ticket_id=ticket_context.get("conversation_id", ""),
        suggested_reply=data.get("suggested_reply", ""),
        confidence=0.8,
        source_articles=[a.get("article_id", "") for a in articles],
        generated_at=datetime.now(),
    )
```

Now add the agent dashboard endpoints:

```python
# In app/main.py — agent dashboard
@app.get("/api/agent/escalations")
async def get_escalations(session: AsyncSession = Depends(get_session)):
    """Get all escalated tickets for human agents."""
    from sqlalchemy import select
    from app.database import ConversationDB
    
    result = await session.execute(
        select(ConversationDB)
        .where(ConversationDB.resolution_status == "escalated")
        .order_by(ConversationDB.urgency.desc())
    )
    escalations = list(result.scalars().all())
    
    return [
        {
            "conversation_id": c.id,
            "customer_id": c.customer_id,
            "intent": c.intent,
            "sentiment": c.sentiment,
            "urgency": c.urgency,
            "created_at": c.created_at.isoformat() if c.created_at else None,
        }
        for c in escalations
    ]

@app.get("/api/agent/tickets/{conversation_id}/copilot")
async def get_copilot_suggestion(
    conversation_id: str,
    session: AsyncSession = Depends(get_session),
):
    """Get an AI-suggested reply for an escalated ticket."""
    context = await prepare_handoff_context(conversation_id, session)
    suggestion = await suggest_reply(context)
    return {
        "suggested_reply": suggestion.suggested_reply,
        "confidence": suggestion.confidence,
        "source_articles": suggestion.source_articles,
    }
```

Update the router node to trigger escalation properly:

```python
# In app/agent/graph.py — enhance router_node
async def router_node(state: SupportState) -> dict:
    needs_esc = state.get("needs_escalation", False)
    confidence = state.get("intent_confidence", 0.0)
    history_len = len(state.get("message_history", []))
    
    # Explicit request for human
    human_keywords = ["talk to a human", "speak to someone", "real person", "agent please"]
    explicit_request = any(k in state.get("customer_message", "").lower() for k in human_keywords)
    
    if explicit_request:
        return {"response_type": "escalate", "resolved": False}
    
    if needs_esc or confidence < 0.7:
        return {"response_type": "escalate", "resolved": False}
    
    # Third attempt without resolution → escalate
    if history_len >= 6:  # 3 customer messages = 6 history entries with AI responses
        return {"response_type": "escalate", "resolved": False}
    
    return {"response_type": "answer", "resolved": True}
```

### ✅ Check: Day 5 Gate

```
1. ❓ Send an angry message ("I WANT TO SPEAK TO A MANAGER") → does it escalate?
2. ❓ After escalation, check GET /api/agent/escalations — is the ticket visible?
3. ❓ Check GET /api/agent/tickets/{id}/copilot — does it suggest a reply?
4. ❓ Does the escalation context include: full transcript, what AI tried, suggested action?
5. ❓ After 3 message exchanges without resolution → does it auto-escalate?
6. ❓ Send a low-confidence question — does it escalate with reason "low_confidence"?

If all 6 work → proceed.
```

---

## 👨‍💻 Day 6: Feedback Loop + Continuous Improvement

### Problem (Senior Engineer)

The AI won't be perfect on day one. Customers will correct it. Agents will override it. We need a feedback loop so the system learns from its mistakes and improves over time.

### What to Build

1. `app/feedback/tracker.py` — Track when agents correct AI responses
2. `app/feedback/learn.py` — Use corrections to improve future responses
3. `app/analytics/dashboard.py` — Resolution stats, CSAT, performance metrics

### Implementation

```python
# app/feedback/tracker.py
from app.database import MessageDB, ConversationDB
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json

class FeedbackTracker:
    """Track feedback on AI responses and agent corrections."""
    
    @staticmethod
    async def record_agent_correction(
        session: AsyncSession,
        conversation_id: str,
        ai_response: str,
        human_response: str,
        correction_reason: str = "",
    ):
        """Record when a human agent corrected an AI response."""
        conv = await session.get(ConversationDB, conversation_id)
        if not conv:
            return
        
        # Store correction metadata
        correction = {
            "ai_response": ai_response,
            "human_response": human_response,
            "correction_reason": correction_reason,
            "intent": conv.intent if conv else None,
        }
        
        # Add as a special message
        msg = MessageDB(
            conversation_id=conversation_id,
            role="human_agent",
            content=human_response,
            metadata_json=json.dumps(correction),
        )
        session.add(msg)
        await session.commit()
    
    @staticmethod
    async def record_csat(
        session: AsyncSession,
        conversation_id: str,
        score: int,  # 1-5
        feedback_text: str = "",
    ):
        """Record customer satisfaction score."""
        conv = await session.get(ConversationDB, conversation_id)
        if conv:
            conv.csat_score = float(score)
            await session.commit()
        
        # Store feedback
        msg = MessageDB(
            conversation_id=conversation_id,
            role="customer",
            content=f"CSAT: {score}/5" + (f" - {feedback_text}" if feedback_text else ""),
            metadata_json=json.dumps({"type": "csat", "score": score, "feedback": feedback_text}),
        )
        session.add(msg)
        await session.commit()
    
    @staticmethod
    async def get_correction_patterns(
        session: AsyncSession, limit: int = 50
    ) -> list[dict]:
        """Analyze common patterns where AI needs correction."""
        result = await session.execute(
            select(MessageDB)
            .where(MessageDB.metadata_json.isnot(None))
            .where(MessageDB.role == "human_agent")
            .limit(limit)
        )
        corrections = []
        for msg in list(result.scalars().all()):
            try:
                meta = json.loads(msg.metadata_json)
                if "correction_reason" in meta:
                    corrections.append(meta)
            except (json.JSONDecodeError, TypeError):
                pass
        return corrections
```

```python
# app/analytics/stats.py
"""Analytics and reporting for SupportAI."""
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func, and_
from app.database import ConversationDB
from datetime import datetime, timedelta

class AnalyticsEngine:
    
    @staticmethod
    async def get_resolution_stats(
        session: AsyncSession, days: int = 7
    ) -> dict:
        """Get resolution statistics for the last N days."""
        since = datetime.now() - timedelta(days=days)
        
        result = await session.execute(
            select(ConversationDB).where(ConversationDB.created_at >= since)
        )
        conversations = list(result.scalars().all())
        
        total = len(conversations)
        if total == 0:
            return {"total": 0, "ai_resolved": 0, "human_resolved": 0, "escalated": 0, "open": 0, "ai_rate": 0}
        
        ai_resolved = sum(1 for c in conversations if c.resolution_status == "ai_resolved")
        human_resolved = sum(1 for c in conversations if c.resolution_status == "human_resolved")
        escalated = sum(1 for c in conversations if c.resolution_status == "escalated")
        open_convos = sum(1 for c in conversations if c.resolution_status == "open")
        
        return {
            "total": total,
            "ai_resolved": ai_resolved,
            "human_resolved": human_resolved,
            "escalated": escalated,
            "open": open_convos,
            "ai_rate": round(ai_resolved / total * 100, 1) if total > 0 else 0,
        }
    
    @staticmethod
    async def get_intent_breakdown(
        session: AsyncSession, days: int = 7
    ) -> list[dict]:
        """Get ticket volume by intent."""
        since = datetime.now() - timedelta(days=days)
        
        result = await session.execute(
            select(
                ConversationDB.intent,
                func.count(ConversationDB.id).label("count")
            )
            .where(ConversationDB.created_at >= since)
            .where(ConversationDB.intent.isnot(None))
            .group_by(ConversationDB.intent)
            .order_by(func.count(ConversationDB.id).desc())
        )
        return [{"intent": r[0], "count": r[1]} for r in result]
    
    @staticmethod
    async def get_average_csat(
        session: AsyncSession, days: int = 7
    ) -> float:
        """Get average CSAT score for resolutions."""
        since = datetime.now() - timedelta(days=days)
        
        result = await session.execute(
            select(func.avg(ConversationDB.csat_score))
            .where(ConversationDB.created_at >= since)
            .where(ConversationDB.csat_score.isnot(None))
        )
        avg = result.scalar()
        return round(float(avg), 2) if avg else 0.0
```

Add analytics endpoints:

```python
# In app/main.py
from app.analytics.stats import AnalyticsEngine

@app.get("/api/analytics/overview")
async def analytics_overview(
    days: int = 7,
    session: AsyncSession = Depends(get_session),
):
    """Get support analytics overview."""
    stats = await AnalyticsEngine.get_resolution_stats(session, days)
    intents = await AnalyticsEngine.get_intent_breakdown(session, days)
    csat = await AnalyticsEngine.get_average_csat(session, days)
    
    return {
        "period_days": days,
        "resolution_stats": stats,
        "top_intents": intents[:5],
        "average_csat": csat,
    }
```

### ✅ Check: Day 6 Gate

```
1. ❓ Run a conversation → AI resolves → POST a CSAT score → check analytics shows it
2. ❓ Agent corrects an AI response → check correction_patterns() shows the correction
3. ❓ Run 10 conversations (mix of intents) → check GET /api/analytics/overview
     shows intent breakdown and resolution rate
4. ❓ What's the AI resolution rate? (Should be 50%+ for tier-1 queries)
5. ❓ What's the average CSAT? (Target >4.0)

If all 5 work → proceed.
```

---

## 👨‍💻 Day 7: Langfuse Observability + Eval Suite

### Problem (Senior Engineer)

We have zero visibility into how the system performs. Every conversation needs to be traced. We need automated benchmarks that measure accuracy, resolution rate, and cost.

### What to Build

1. Add `@observe()` decorators to trace every conversation through Langfuse
2. Create eval test suite with known test conversations
3. Create `scripts/run_benchmark.py` for automated benchmarking

### Implementation

```python
# app/observability.py
from langfuse.decorators import observe, langfuse_context
from app.config import settings
from langfuse import Langfuse

langfuse = Langfuse(
    public_key=settings.langfuse_public_key,
    secret_key=settings.langfuse_secret_key,
    host=settings.langfuse_host,
)

@observe(name="supportai-conversation")
async def traced_process_message(
    conversation_id: str, message: str, history: list[str]
):
    """Traced version of the message processing pipeline."""
    from app.agent.graph import process_message
    result = await process_message(conversation_id, message, history)
    
    langfuse_context.update_current_trace(
        name=f"Conv-{conversation_id}",
        metadata={
            "conversation_id": conversation_id,
            "intent": result.get("intent"),
            "sentiment": result.get("sentiment"),
            "urgency": result.get("urgency"),
            "resolved": result.get("resolved"),
            "response_type": result.get("response_type"),
            "num_citations": len(result.get("citations", [])),
        },
        tags=["customer-support", result.get("intent", "unknown")],
    )
    
    return result
```

Now create the eval test suite:

```python
# tests/test_evals.py
"""
Eval suite for SupportAI.
Each test case has a known scenario. We validate the AI's response quality.
"""
import pytest
from app.agent.graph import process_message
from app.classifier.intent import classify_intent
from app.classifier.sentiment import analyze_sentiment
from app.knowledge_base.retriever import query_knowledge_base

# ── Intent Classification Tests ──

@pytest.mark.asyncio
async def test_intent_refund():
    result = await classify_intent("I need a refund for order ORD-12345")
    assert result.intent == "refund_request", f"Expected refund_request, got {result.intent}"
    assert result.confidence > 0.7
    assert "ORD-12345" in str(result.entities.get("order_id", ""))

@pytest.mark.asyncio
async def test_intent_password_reset():
    result = await classify_intent("I forgot my password, can you help me log in?")
    assert result.intent in ("password_reset", "account_issue")
    assert result.confidence > 0.7

@pytest.mark.asyncio
async def test_intent_order_status():
    result = await classify_intent("Where is my order? I ordered 5 days ago.")
    assert result.intent == "order_status"
    assert result.confidence > 0.7

# ── Sentiment Analysis Tests ──

@pytest.mark.asyncio
async def test_sentiment_frustrated():
    result = await analyze_sentiment(
        "I've been waiting for OVER A WEEK and still no resolution! This is UNACCEPTABLE!"
    )
    assert result.sentiment in ("frustrated", "angry")
    assert result.urgency >= 4

@pytest.mark.asyncio
async def test_sentiment_neutral():
    result = await analyze_sentiment("Hi, I was wondering about your return policy.")
    assert result.sentiment in ("neutral", "positive")
    assert result.urgency <= 2
    assert result.needs_escalation == False

# ── Knowledge Base Retrieval Tests ──

@pytest.mark.asyncio
async def test_kb_refund_query():
    articles = query_knowledge_base("How do I get my money back for a purchase?")
    assert len(articles) > 0
    assert any("Refund Policy" in a.get("content", "") for a in articles)

@pytest.mark.asyncio
async def test_kb_password_query():
    articles = query_knowledge_base("I can't remember my login password")
    assert len(articles) > 0
    assert any("Reset Your Password" in a.get("content", "") for a in articles)

# ── End-to-End Conversation Tests ──

@pytest.mark.asyncio
async def test_full_refund_conversation():
    """Test a complete refund request conversation."""
    result = await process_message(
        "test-conv-1",
        "I need to return a product I bought last week. The size doesn't fit.",
        []
    )
    assert result.get("intent") == "refund_request"
    assert result.get("ai_response") is not None
    assert len(result.get("ai_response", "")) > 20
    # Should either answer the question or escalate
    assert result.get("response_type") in ("answer", "escalate")

@pytest.mark.asyncio
async def test_angry_escalation():
    """Test that very angry customers get escalated."""
    result = await process_message(
        "test-conv-2",
        "This is the WORST service I've ever experienced. I WANT A MANAGER RIGHT NOW!!!",
        []
    )
    assert result.get("response_type") == "escalate"

@pytest.mark.asyncio
async def test_knowledge_unknown():
    """Test that the AI doesn't hallucinate on unknown topics."""
    result = await process_message(
        "test-conv-3",
        "Can you tell me the chemical formula for caffeine?",
        []
    )
    response = result.get("ai_response", "").lower()
    # Should not make up an answer about something outside the KB
    has_human_offer = any(p in response for p in [
        "don't have that", "don't know", "connect to a human",
        "not in my knowledge", "cannot answer"
    ])
    assert has_human_offer, f"AI should admit it doesn't know: {response}"
```

Benchmark runner:

```python
# scripts/run_benchmark.py
"""Run the SupportAI eval suite and generate a benchmark report."""
import json
import sys
import time
from pathlib import Path
from datetime import datetime

sys.path.insert(0, str(Path(__file__).parent.parent))

import pytest

def run_benchmark():
    print("=" * 60)
    print("SupportAI Benchmark Suite")
    print(f"Date: {datetime.now().isoformat()}")
    print("=" * 60)
    
    start = time.time()
    
    exit_code = pytest.main([
        "-v", "--tb=short",
        "tests/test_evals.py"
    ])
    
    duration = time.time() - start
    
    report = {
        "timestamp": datetime.now().isoformat(),
        "duration_seconds": round(duration, 2),
        "passed": exit_code == 0,
        "exit_code": exit_code,
        "version": "1.0.0"
    }
    
    report_path = Path("benchmark_results.json")
    with open(report_path, "w") as f:
        json.dump(report, f, indent=2)
    
    print(f"\n{'=' * 60}")
    print(f"Status: {'✅ PASSED' if exit_code == 0 else '❌ FAILED'}")
    print(f"Duration: {duration:.1f}s")
    print(f"Report saved to: {report_path}")
    
    return exit_code

if __name__ == "__main__":
    sys.exit(run_benchmark())
```

### ✅ Check: Day 7 Gate — THIS IS YOUR SHIP GATE

```
1. ❓ Run the benchmark:  python scripts/run_benchmark.py
    → All 8 eval tests must pass.

2. ❓ Set up Langfuse (docker compose or cloud account)
    → Traces should appear in the Langfuse dashboard.

3. ❓ Do a full end-to-end test:
    - Send a support message via POST /api/messages
    - Message gets classified (intent + sentiment)
    - Knowledge base is queried
    - AI generates a response with citations
    - Response is stored in the conversation
    - Trace visible in Langfuse

4. ❓ Test escalation:
    - Send angry message → routes to human handoff
    - Check GET /api/agent/escalations → ticket visible
    - Check copilot suggestion → reply suggested
    - Cost tracked for the conversation

5. ❓ Edge cases:
    - Empty message
    - Non-English message
    - Very long message (2000+ words)
    - Multiple rapid messages in the same conversation
    - Conversation exceeding max_turns

6. ❓ Analytics check:
    - GET /api/analytics/overview → shows resolution stats
    - AI resolution rate > 50%
    - Average CSAT tracked

7. ❓ Cost analysis:
    - Average cost per resolution < $0.05
    - Monthly projection for 1000 conversations
```

---

## 🚀 Post-Build: What We Built vs Real Products

| Feature | SupportAI (Your Build) | Intercom Fin | Zendesk AI | Ada |
|---------|----------------------|--------------|------------|-----|
| Intent classification | ✅ | ✅ | ✅ | ✅ |
| Sentiment + urgency detection | ✅ | ✅ | ✅ | ✅ |
| RAG over knowledge base | ✅ | ✅ | ✅ | ✅ |
| Citation-grounded answers | ✅ | ✅ | ✅ | ✅ |
| Multi-turn conversations | ✅ | ✅ | ✅ | ✅ |
| Human handoff with full context | ✅ | ✅ | ✅ | ✅ |
| Agent copilot suggestion | ✅ | ✅ | ✅ | ✅ |
| Escalation routing | ✅ | ✅ | ✅ | ✅ |
| Feedback loop + learning | ✅ | ✅ | ✅ | ✅ |
| Analytics dashboard | ✅ | ✅ | ✅ | ✅ |
| Langfuse observability | ✅ | Proprietary | Proprietary | Proprietary |
| Voice AI support | Coming next | ✅ | ✅ | ✅ |
| Email channel | Coming next | ✅ | ✅ | ✅ |
| Multi-language | Configurable | 45+ languages | 50+ languages | 50+ languages |
| Cost per resolution | ~$0.01-0.05 | $0.99 | ~$0.50 | ~$0.30-0.50 |

**What you built is a functional competitor to products companies pay $60K+/year for.**

---

## 📚 Deliverables Checklist

```
[ ] Source code pushed to GitHub: supportai/
[ ] README.md with architecture, setup, usage
[ ] Dockerfile + docker-compose.yml
[ ] Knowledge base seeded with 6+ articles
[ ] All 8 eval tests passing
[ ] Langfuse dashboard showing conversation traces
[ ] Demo video (2-3 min): send messages, see classification, show handoff, dashboard
[ ] Cost analysis: $/resolution, monthly projection for 1000 conversations
[ ] Post-mortem: what broke, what surprised you, what's next

🏆 Portfolio entry template:

  "I built SupportAI — an AI customer support agent that classifies intent,
   detects sentiment, resolves tier-1 tickets autonomously via RAG over a
   knowledge base, and seamlessly escalates to humans with full context.
   It achieves >50% autonomous resolution with citation-grounded answers
   at ~$0.02 per resolution. Includes agent copilot, feedback loop, and
   Langfuse observability. Deployed as a FastAPI service."
```

---

**You built something that competes in a $15B market. Ship it.**
