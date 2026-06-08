# Streaming, SSE & WebSockets — Making AI Feel Fast

## 🎯 Purpose & Goals

> **🛑 STOP. Think about this:**
>
> You type a question into ChatGPT. You see text appearing word by word. The first word appears in ~500ms. The full response takes 10 seconds.
>
> Now imagine ChatGPT showed NOTHING for 10 seconds, then dumped ALL the text at once.
>
> **Which feels faster?** (The first one, even though both take 10 seconds).
>
> **Why does the first one feel better?** And more importantly — **how would you implement it?**

---

**By the end of this file, you will:**
- Understand SSE (Server-Sent Events) — the backbone of ChatGPT
- Build a streaming FastAPI endpoint that streams AI responses
- Handle streaming errors gracefully (partial responses, disconnects)
- Know when to use SSE vs WebSockets for AI applications

**⏱ Time Budget:** 1.5 hours

---

## 📖 The Streaming Mental Model

```python
"""
NON-STREAMING (what beginners build):
User sends request → wait 10 seconds → get full response
┌─────────┐     ─────10 seconds─────     ┌──────────┐
│  User   │                              │  Server  │
│         │ ───── question ──────────→   │          │
│         │ ←──── full answer ────────   │          │
└─────────┘                              └──────────┘
User sees NOTHING for 10 seconds. Feels broken.

STREAMING (what production apps do):
User sends request → get tokens as they're generated
┌─────────┐                              ┌──────────┐
│  User   │ ───── question ──────────→   │  Server  │
│         │ ←─── token 1 (500ms) ────   │          │
│         │ ←─── token 2 (600ms) ────   │          │
│         │ ←─── token 3 (700ms) ────   │          │
│         │         ...                  │          │
│         │ ←─── token N (10s) ──────   │          │
└─────────┘                              └──────────┘
User sees FIRST TOKEN in 500ms. Reads along. Feels fast.
"""
```

> **🤔 QUESTION:** Streaming doesn't make the total time faster. It makes the **time-to-first-token** faster. Why does time-to-first-token matter more than total time for user perception?

---

## 📖 Building a Streaming API (FastAPI + SSE)

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI
import asyncio

app = FastAPI()
client = AsyncOpenAI()

# 🛑 YOUR TURN: Before reading the solution, write a FastAPI endpoint
# that streams tokens from OpenAI. Think about:
# 1. What should the endpoint look like? (GET? POST?)
# 2. How do you yield tokens one by one?
# 3. How do you handle errors mid-stream?
# 4. How does the frontend receive the stream?

# Write your approach:
"""
Endpoint: ___ 
Method: ___
Parameters: ___
Response type: ___
Error handling: ___
"""

<details>
<summary>👀 Production solution</summary>

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI
from pydantic import BaseModel

app = FastAPI()
client = AsyncOpenAI()

class ChatRequest(BaseModel):
    messages: list[dict]
    model: str = "gpt-4o-mini"
    temperature: float = 0.0

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    """
    Streams AI response token by token using SSE.
    
    Response format:
    data: {"token": "Hello", "done": false}
    data: {"token": " world", "done": false}
    data: {"token": "", "done": true}
    """
    
    async def generate():
        try:
            stream = await client.chat.completions.create(
                model=request.model,
                messages=request.messages,
                temperature=request.temperature,
                stream=True,
            )
            
            async for chunk in stream:
                if chunk.choices[0].delta.content:
                    token = chunk.choices[0].delta.content
                    yield f"data: {json.dumps({'token': token, 'done': false})}\n\n"
            
            # Signal completion
            yield f"data: {json.dumps({'token': '', 'done': true})}\n\n"
            
        except Exception as e:
            # Send error as SSE event
            yield f"data: {json.dumps({'error': str(e), 'done': true})}\n\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        }
    )

# Test with curl:
# curl -N -X POST http://localhost:8000/chat/stream \
#   -H "Content-Type: application/json" \
#   -d '{"messages": [{"role": "user", "content": "Tell me a story"}]}'
```

**Critical production details:**
1. **`data:` prefix** — SSE protocol requires this (frontend EventSource parses it)
2. **`\n\n` suffix** — SSE event delimiter
3. **`Cache-Control: no-cache`** — prevents proxy caching
4. **`X-Accel-Buffering: no`** — prevents nginx from buffering the entire response
5. **Error handling** — send error as an SSE event, don't crash the connection

**Without these headers, your streaming will buffer and feel NON-streaming.**
</details>

---

## 📖 Frontend Integration (Next.js)

```typescript
// 🛑 YOUR TURN: How would you consume the SSE endpoint from the frontend?
// Write the TypeScript/React code to read the stream and display tokens.

// Think about:
// 1. Fetch API or WebSocket or EventSource?
// 2. How to read streaming data?
// 3. How to update the UI as tokens arrive?

// Write your approach:
```

<details>
<summary>👀 React/Next.js solution</summary>

```typescript
// app/api/chat/route.ts — Next.js App Router
export async function POST(req: Request) {
  const { messages } = await req.json();
  
  const response = await fetch('http://localhost:8000/chat/stream', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages }),
  });
  
  // Forward the SSE stream to the client
  return new Response(response.body, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}

// app/page.tsx — React component
'use client';
import { useState } from 'react';

export default function Chat() {
  const [response, setResponse] = useState('');
  const [loading, setLoading] = useState(false);
  
  async function ask(question: string) {
    setLoading(true);
    setResponse('');
    
    const res = await fetch('/api/chat', {
      method: 'POST',
      body: JSON.stringify({
        messages: [{ role: 'user', content: question }]
      }),
    });
    
    const reader = res.body!.getReader();
    const decoder = new TextDecoder();
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      const text = decoder.decode(value);
      const lines = text.split('\n');
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = JSON.parse(line.slice(6));
          if (data.done) {
            setLoading(false);
          } else {
            setResponse(prev => prev + data.token);
          }
        }
      }
    }
  }
  
  return (
    <div>
      <button onClick={() => ask('Tell me a story')}>Ask</button>
      <p>{response}</p>
      {loading && <span className="cursor-blink">▊</span>}
    </div>
  );
}
```

**Key insight:** The browser's `ReadableStream` API reads streaming data. Each `data:` line is parsed as JSON. The cursor blink (`▊`) makes it feel alive.
</details>

---

## 📖 SSE vs WebSockets for AI Streaming

```python
"""
WHEN TO USE WHAT:

SSE (Server-Sent Events) — USE THIS FOR AI STREAMING
✅ One-directional (server → client) — perfect for token streaming
✅ Built into HTTP — works with all proxies, load balancers
✅ Automatic reconnection — browser EventSource reconnects on failure
✅ Simpler — just HTTP responses
❌ Client can't send data after initial request
❌ Limited to text data

WebSockets — USE FOR REAL-TIME CHAT
✅ Bi-directional — client and server can send messages anytime
✅ Lower overhead per message (after initial handshake)
✅ Good for: multi-turn conversations, live collaboration
❌ Complex reconnection logic
❌ Some proxies block WebSockets
❌ More complex to implement

GENERAL RULE:
- LLM response streaming → SSE (simpler, just works)
- Real-time chat with multiple events → WebSockets
- Most AI apps: SSE for streaming + REST for user input
"""
```

---

## 🧪 Debug Challenge: Buffered Streaming

```python
# BUGGY CODE — This SHOULD stream, but it doesn't. Why?

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI
import asyncio

app = FastAPI()
client = AsyncOpenAI()

@app.post("/chat")
async def chat(request: Request):
    body = await request.json()
    messages = body["messages"]
    
    async def generate():
        stream = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            stream=True,
        )
        
        full_response = ""
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                full_response += chunk.choices[0].delta.content
                yield full_response  # BUG! Sending accumulated text
                
        yield "DONE"
    
    return StreamingResponse(generate())
```

**🔍 Find at least 2 bugs.**

<details>
<summary>Click after thinking</summary>

**Bug 1: Sending accumulated text, not incremental tokens**
```python
# BAD: Sends "H", "He", "Hel", "Hell", "Hello" (duplicates!)
yield full_response  # Each yield sends ALL accumulated text

# GOOD: Sends only the NEW token
yield chunk.choices[0].delta.content  # Just "H", then "e", then "l", etc.
```

**Bug 2: No SSE format (data: prefix)**
Frontend using EventSource won't parse this. Need:
```python
yield f"data: {json.dumps({'token': token, 'done': false})}\n\n"
```

**Bug 3: No error handling mid-stream**
If the OpenAI API fails mid-stream, the generator raises an unhandled exception. The StreamResponse catches it but the frontend gets a truncated response with no error signal.

**Senior Insight:** These are THE three most common streaming bugs in production AI systems. Fix them once, use the pattern everywhere.
</details>

---

## 📖 Cross-Reference

- **Phase 2 (Prompt Engineering):** Streaming prompts need careful system prompt design (short enough to get fast first token)
- **Phase 6 (Agents):** Agent streaming is harder — you need to stream BOTH the thinking AND the final answer
- **Phase 10 (Deployment):** Streaming under load requires connection pooling, proper nginx config

---

## 🚦 Gate Check

- [ ] I can explain SSE in one sentence
- [ ] I can build a streaming FastAPI endpoint
- [ ] I know why time-to-first-token matters more than total time
- [ ] I understand SSE vs WebSockets tradeoffs
- [ ] I can fix the 3 most common streaming bugs

---

**→ Continue to `09-AI-Production-Framework-Landscape.md`**
