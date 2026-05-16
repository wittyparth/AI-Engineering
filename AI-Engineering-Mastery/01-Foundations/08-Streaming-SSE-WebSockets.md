# Streaming: SSE & WebSockets

## Why Streaming Matters

Users expect to see tokens appearing as the model generates. Without streaming:
- **Perceived latency** = full generation time (often 5-30s)
- **With streaming**: first token in <1s, rest trickles in

## Server-Sent Events (SSE)

Best for: Unidirectional streaming (server → client), chat applications.

```python
from fastapi.responses import StreamingResponse

@app.get("/chat")
async def chat(message: str):
    async def event_stream():
        # Send the prompt to the model
        stream = await llm_client.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": message}],
            stream=True,
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield f"data: {json.dumps({'token': chunk.choices[0].delta.content})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        }
    )
```

## WebSockets

Best for: Bidirectional (client ↔ server), tool calls, agent loops.

```python
@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_json()
        if data["type"] == "message":
            async for chunk in llm_client.generate_stream(data["content"]):
                await websocket.send_json({"type": "token", "content": chunk})
            await websocket.send_json({"type": "done"})
        elif data["type"] == "cancel":
            break  # Client wants to stop generation
```

## 🔴 Senior: Streaming Gotchas

### 1. Token-Level Is Not Character-Level
- One chunk may contain multiple tokens
- Never display partial tokens if you care about UX

### 2. Error Handling During Stream
```python
try:
    async for chunk in stream:
        yield chunk
except ProviderError:
    yield {"type": "error", "message": "Generation interrupted"}
```

### 3. Connection Management
- SSE connections can drop. Client MUST implement reconnection.
- WebSocket must handle ping/pong for long generations.

### 4. Backpressure
- Client consumes slower than server produces
- Implement flow control or buffer management

### 5. Cost Tracking
```python
# Track token usage even during streaming
total_tokens = 0
async for chunk in stream:
    total_tokens += 1
    yield chunk
finally:
    telemetry.log_tokens(total_tokens)  # Log when stream ends
```

## Drill: Stream Debug Challenge

Given broken streaming code that has these bugs:
1. Buffer underrun — chunks arrive out of order
2. Connection closes prematurely
3. Error during stream is silently swallowed
4. No timeout — stream hangs forever on provider failure
5. Token counting is wrong

Find and fix all 5 bugs.
