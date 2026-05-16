# Logging & Tracing

## Structured Logging

```python
import structlog

logger = structlog.get_logger()

# In your AI service
@app.post("/chat")
async def chat(request: ChatRequest):
    request_id = str(uuid.uuid4())
    logger.info("chat_request", request_id=request_id, query=request.query)

    try:
        start = time.time()
        chunks = await retrieve(request.query)
        logger.info("retrieval_complete", request_id=request_id, chunks=len(chunks))

        response = await generate(request.query, chunks)
        latency = time.time() - start
        logger.info("generation_complete", request_id=request_id,
                    latency=latency, tokens=response.usage.total_tokens)

        return response
    except Exception as e:
        logger.error("chat_failed", request_id=request_id, error=str(e))
        raise
```

## OpenTelemetry Tracing

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

tracer = trace.get_tracer(__name__)

@app.post("/rag")
async def rag_query(query: str):
    with tracer.start_as_current_span("rag_query") as span:
        span.set_attribute("query.length", len(query))

        # Sub-span for retrieval
        with tracer.start_as_current_span("retrieve") as retrieve_span:
            chunks = await retrieve(query)
            retrieve_span.set_attribute("chunks.count", len(chunks))

        # Sub-span for generation
        with tracer.start_as_current_span("generate") as gen_span:
            response = await generate(query, chunks)
            gen_span.set_attribute("tokens", response.usage.total_tokens)

        return response

# Auto-instrument FastAPI
FastAPIInstrumentor.instrument_app(app)
```

## Trace Context Propagation

```python
# Include trace_id in every log line
class TraceIDFilter(logging.Filter):
    def filter(self, record):
        span = trace.get_current_span()
        if span:
            ctx = span.get_span_context()
            record.trace_id = format(ctx.trace_id, "032x")
            record.span_id = format(ctx.span_id, "016x")
        else:
            record.trace_id = "none"
            record.span_id = "none"
        return True

# Now every log line includes trace_id — cross-reference with traces
```

## 🔴 Senior: What to Instrument

| Component | Measure | Why |
|-----------|---------|-----|
| Tokenization | Time, token count | Know context size before LLM call |
| Embedding | Time, dimensions | First potential bottleneck |
| Vector search | Time, results count, score distribution | Know if search is working |
| Reranking | Time, score delta before/after | Know if reranking helped |
| LLM call | Time, input/output tokens, cost | Biggest cost driver |
| Guardrails | Time, pass/fail rate | Know if security is working |
| User feedback | Thumbs up/down rate | Ground truth quality signal |
