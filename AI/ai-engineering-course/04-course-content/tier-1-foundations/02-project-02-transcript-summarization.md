# Project 2: Meeting Transcript Summarization Pipeline

- **Tier:** 1 — Internal Studio Tools
- **Project #:** 2 of 3
- **Tech Stack:** Python, LLM API (OpenAI/Anthropic), Pydantic v2, Python file I/O (JSON/CSV), basic token counting (`tiktoken`)
- **Concepts:** Structured outputs at scale, Pydantic validation, batch processing, cost tracking, error recovery, rate limit handling
- **Quality Gate:** ✅ APPROVED when processing 10 transcripts costs less than $1 total AND extracts consistently structured output across varying transcript quality, lengths, and formats.

---

## Phase 1: Brief (Priya)

> *Priya walks in holding a laptop in one hand and a coffee in the other. She looks like she hasn't slept.*

"Okay. New internal tool. This one hurts.

I have **20+ hours of client meeting recordings every week.** Strategy calls, sprint reviews, client feedback sessions, emergency calls. Each one is 45-90 minutes. Someone transcribes them — that part's already solved.

What's NOT solved is that I have to **read every single transcript** to figure out what happened. Three bullet points. Action items. Key decisions. That's all I need. But I'm spending 4 hours a day reading transcripts instead of doing actual PM work.

**Here's what I need:** A tool that takes a raw transcript and produces:
1. A 3-bullet executive summary (what happened, why it matters)
2. Action items with owners (who needs to do what by when)
3. Key decisions made (what changed, who decided)

**Non-negotiable:**
- I need this for EVERY transcript, not just the clean ones
- Some transcripts are terrible quality — heavy accents, overlapping speech, 'um's and 'uh's everywhere
- Cost matters. Maya tells me every API call costs money. If this tool costs more than $1 per 10 transcripts, I can't justify it to Rohan
- The output format MUST be identical every time. I'm going to pipe this into a spreadsheet

**Deadline:** 4 days. I have a backlog of 30 transcripts I need processed by Monday.

**Definition of done:** I paste a transcript → I get a structured output. I paste 10 transcripts → I get 10 identical structured outputs. Total cost < $1 per batch."

> *She puts the coffee down. "I really need this. The reading is killing me."*

---

## Phase 2: Learning Path (Maya)

> *Maya catches you in the hallway. "I heard Priya's brief. Here's what this is really teaching you."*

"This project looks like a simple summarization task. It's not. It's teaching you **batch processing at scale** — how to process large volumes of data reliably, efficiently, and cost-effectively. This is the same pattern you'll use for support ticket triage, email classification, content moderation, and every batch AI pipeline in production.

The summarization is the easy part. The hard parts are: cost, consistency, error handling, and batch management.

### Learning Order (Scratch-First)

**Step 1: Build a single-transcript pipeline.**
One transcript in, structured output out. Use Pydantic + Structured Outputs (you already know this from Project 1).

**Step 2: Add token counting and cost estimation.**
Before you process a single transcript, you need to know how much it costs. Use `tiktoken` (OpenAI's tokenizer) to count tokens, estimate cost per transcript, and build a budget checker.

> *"Production teams don't guess costs — they track every token. If you can't tell me what a batch costs before you run it, you're not ready for production."*

**Step 3: Handle rate limits and errors.**
When you process 10 transcripts in sequence, you will hit API rate limits. You'll get 429 errors. You'll get timeouts. Build retry logic with exponential backoff.

> *"This is the skill that separates engineers who build demos from engineers who build systems. Demo engineers handle the happy path. System engineers handle the sad path."*

**Step 4: Add batch processing and progress tracking.**
Process a directory of transcripts. Show progress. Track per-file costs. Generate a batch summary. Handle partial failures (one file fails → continue with the rest).

**Step 5: (Optional, if time) Parallel processing.**
Use `asyncio` or `ThreadPoolExecutor` to process multiple transcripts concurrently. But add rate limit awareness — you can't fire 20 requests at once or the API will throttle you.

### Memory Triage

**Memorize cold:**
- Token counting with `tiktoken` — `encoding.encode(text)` and `len(tokens)`
- Exponential backoff retry pattern — `time.sleep(2 ** attempt)` in retry loop
- The `try/except` pattern for API calls — catch `openai.RateLimitError`, `openai.APITimeoutError`, `openai.APIConnectionError`
- The Pydantic dataclass-like syntax for structured output models

**Look up when needed:**
- Specific `tiktoken` encoding names (they change as models update)
- OpenAI/Anthropic rate limit tiers (depends on your account level)
- `asyncio` vs `ThreadPoolExecutor` tradeoffs (both valid, different use cases)

**Understand deeply:**
- Why cost tracking matters more than you think — *"Rohan checks cost on every submission. If you can't tell him per-query cost, he'll make you add it before APPROVING. And he should."*
- The tradeoff between quality and cost — *"Do you use GPT-4o (expensive, accurate) or GPT-4o-mini (cheap, good enough)? This is a real decision you'll make every week."*

### First Concrete Step

> "Take the Pydantic model you built for Project 1 (or build a new one). Define the output schema for a transcript summary. Three fields: `summary` (3 bullet points), `action_items` (list of `{owner, task, deadline}`), `key_decisions` (list of `{decision, rationale}`). Use Strict Mode with `additionalProperties: false`."

> *"Schema-First Development. Define what success looks like in data first. Then build the pipeline around it."*

### Resources (Just-in-Time)

- **tiktoken GitHub repo** — OpenAI's tokenizer, essential for cost estimation
- **OpenAI Rate Limits Guide** — understand 429 errors and how to handle them
- **OpenAI Error Codes** — which errors are retryable, which aren't
- **Pydantic v2 Docs: Models** — `BaseModel`, field types, validators

---

## Phase 3: The Build

> *Let's build this thing. There will be blood (metaphorically).*

### Milestone 1: Schema Design + Single Transcript

Define the output model:

```python
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime

class ActionItem(BaseModel):
    owner: str
    task: str
    deadline: Optional[str] = None  # "2026-05-20" or "Next sprint" or "TBD"

class KeyDecision(BaseModel):
    decision: str
    rationale: str
    decided_by: Optional[str] = None

class TranscriptSummary(BaseModel):
    executive_summary: List[str]  # exactly 3 bullet points
    action_items: List[ActionItem]
    key_decisions: List[KeyDecision]
    meeting_date: Optional[str] = None  # extract from transcript if present
```

Then build a single function: `summarize_transcript(transcript_text: str) -> TranscriptSummary`

**Expected stuck point:** The model sometimes returns 2 bullet points instead of 3, or 4 instead of 3. You asked for exactly 3 but LLMs are unreliable about array lengths.

**Maya's Socratic question:**
> *"Your model returned 2 bullet points instead of 3. Why did the LLM do that? And more importantly — how do you enforce 'exactly 3' in the schema?"*

> They should arrive at: you can't enforce array lengths in JSON Schema. The solution is either (a) use separate named fields (`bullet_1`, `bullet_2`, `bullet_3`) or (b) validate post-hoc and retry. Maya recommends (a) for production reliability.

### Milestone 2: Cost Estimation

Before processing, implement cost tracking:

```python
import tiktoken

def estimate_cost(text: str, model: str = "gpt-4o-mini") -> dict:
    encoding = tiktoken.encoding_for_model(model)
    tokens = encoding.encode(text)
    token_count = len(tokens)
    
    # Pricing (check current rates — these change)
    pricing = {
        "gpt-4o-mini": {"input": 0.15 / 1_000_000, "output": 0.60 / 1_000_000},
        "gpt-4o": {"input": 2.50 / 1_000_000, "output": 10.00 / 1_000_000},
    }
    
    estimated_input_cost = token_count * pricing[model]["input"]
    # Assume ~200 output tokens for structured summary
    estimated_output_cost = 200 * pricing[model]["output"]
    estimated_total = estimated_input_cost + estimated_output_cost
    
    return {
        "tokens": token_count,
        "estimated_cost": estimated_total,
        "model": model
    }
```

**Why this matters:** This function will save you from accidentally spending $5 on a single transcript (it happens).

**Bonus annoyance to discover:** The transcript itself might be 15,000 tokens. A 45-minute meeting transcript is easily 8,000-12,000 tokens.

> *"You just realized that a single transcript can cost $0.02-0.05 with GPT-4o-mini. With GPT-4o, that's $0.30-0.50. Now multiply by 30 transcripts. Priya's budget constraint makes sense now, right?"*

### Milestone 3: Error Handling + Rate Limiting

Write a robust API wrapper:

```python
import time
from openai import (
    OpenAI, RateLimitError, APITimeoutError, APIConnectionError, InternalServerError
)

def call_llm_with_retry(client, model, messages, response_format, max_retries=5):
    for attempt in range(max_retries):
        try:
            response = client.beta.chat.completions.parse(
                model=model,
                messages=messages,
                response_format=response_format,
            )
            return response.choices[0].message
            
        except RateLimitError as e:
            wait_time = 2 ** attempt + random.uniform(0, 1)
            print(f"Rate limited. Waiting {wait_time:.1f}s... (attempt {attempt + 1})")
            time.sleep(wait_time)
            
        except APITimeoutError:
            print(f"Timeout. Retrying... (attempt {attempt + 1})")
            
        except InternalServerError:
            print(f"OpenAI server error. Retrying... (attempt {attempt + 1})")
            time.sleep(5)  # server errors need longer cool-down
            
        except APIConnectionError:
            print(f"Connection error. Retrying... (attempt {attempt + 1})")
            time.sleep(2 ** attempt)
    
    raise Exception(f"Failed after {max_retries} retries")
```

> *"This is the same retry pattern used in LiteLLM and every production AI pipeline. It's not glamorous. It's necessary."*

**Expected stuck point:** After 5 retries you still fail. What do you do?

**Maya's Socratic question:**
> *"You've retried 5 times and the API is still failing. What now? The other 9 transcripts are waiting."*

> They should decide: log the failure, save the transcript filename to a `failed.txt` file, and continue with the next transcript. Fail one transcript — not the whole batch.

### Milestone 4: Batch Processing

Build the batch pipeline:

```python
def process_transcripts(
    input_dir: str, 
    output_dir: str, 
    model: str = "gpt-4o-mini",
    budget_limit: float = 1.00
):
    """
    Process all transcripts in input_dir.
    - Estimate total cost before starting
    - Process each transcript with retry logic
    - Save individual results as JSON
    - Generate batch summary CSV
    - Stop if budget exceeded
    """
```

**Expected stuck point:** A transcript is in a weird format — maybe plain text, maybe Markdown, maybe it has speaker labels (`[SPEAKER_00]:`), maybe it doesn't.

**Maya's Socratic question:**
> *"Some transcripts have speaker labels, some don't. Should you strip the labels before sending to the LLM? Does it help or hurt summarization quality?"*

> They should think about: speaker labels are useful context (who said what) but add tokens (cost). Real tradeoff.

### Rohan's Mid-Build Interruption

> *Rohan appears while you're testing the batch pipeline.*

"Show me your cost estimation. I want to know:
1. How much does 10 transcripts cost with GPT-4o-mini?
2. How much with GPT-4o?
3. What's the cost difference if a transcript is 20,000 tokens vs 5,000 tokens?

Don't guess. Run the numbers."

This forces the learner to actually instrument cost tracking, not just theorize about it.

### Priya's Requirement Change

> *Priya: "Hey. Two things. First — some transcripts come from Otter.ai, some from Fireflies, some from manual transcription. They have different formats. Your tool needs to handle them all.*

> *Second — Rohan wants a CSV export alongside the JSON. He needs to import action items into Jira. Same data, different format. One extra day."*

---

## Phase 4: Review (Rohan)

> *You submit: working batch pipeline, cost logs, 10 processed transcripts with all formats handled, CSV export.*

### Decision Documentation Required

1. Which model you chose (GPT-4o-mini vs GPT-4o) and why — with actual cost comparison data
2. How you handle malformed transcripts (empty text, gibberish, extremely long transcripts)
3. Why you chose sequential vs parallel processing (or your parallel approach with rate limiting)

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | 10 transcripts processed, structured output, consistent format |
| 2 | **Edge cases handled?** | ✅ PASS | Handles empty transcripts, different formats, API failures gracefully |
| 3 | **Cost-aware?** | ✅ PASS | $0.87 for 10 transcripts with GPT-4o-mini. Cost estimates before each batch. Budget limit enforced. Well done. |
| 4 | **Observable?** | 🔄 REVISE | You log successes but not failures in enough detail. When a transcript fails midway, I need to see which action item was being processed, not just "error at step 2" |
| 5 | **Right approach?** | ✅ PASS | Sequential processing was the right call for this scale. Parallel would have added complexity without benefit. |
| 6 | **Decisions justified?** | ✅ PASS | Clear cost comparison between models. Good reasoning on GPT-4o-mini. |
| 7 | **Measurable quality?** | 🔄 REVISE | You processed 10 transcripts, but how do you know the summaries are good? Manual spot-check 3 of them and document what you found. I need confidence, not just throughput. |

### Verdict: 🔄 REVISE

*Rohan nods slowly.*

"Almost there. Two fixes:

1. **Failure detail logging** — When a transcript fails, log the exact step (token counting? API call? parsing?), the raw error, and the first 500 characters of the transcript. I need to diagnose without reproducing.

2. **Quality spot-check** — Run 3 random transcripts through manually. Read the original transcript, read your summary, and rate it (accurate / mostly accurate / misses key points / hallucinates). Document every discrepancy. I need to know your system isn't hallucinating action items.

Also: your cost tracking is excellent. Most junior engineers skip this. You didn't. That's worth noting."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "You just built a production batch AI pipeline. Let me tell you what you actually learned:

- **Batch processing at scale** — processing multiple items reliably, handling failures in individual items without crashing the whole batch
- **Cost engineering** — you can now estimate, track, and optimize token usage. This is a legit senior engineer skill
- **Error recovery** — you've handled rate limits, timeouts, server errors, and malformed input. You've learned that production code spends 60% of its lines handling things that aren't the happy path
- **Format flexibility** — you handled multiple transcript formats. This is what 'works in production' means

**What you'll see again:**
- **Batch processing** — Project 6 (CV parsing pipeline) is this same pattern but harder. You'll process 1,000 CVs instead of 10 transcripts, with stricter accuracy requirements
- **Cost optimization** — Project 7 (model routing) is entirely about cost. You laid the foundation here
- **Quality evaluation** — The manual spot-check you did is a primitive version of the eval frameworks you'll use in Tier 2

**On GPT-4o-mini:** You chose the cheap model and it was good enough. This is the right instinct for 90% of production use cases. Expensive models are for when the cheap model demonstrably fails — not for everything.

> *Two projects down. Something's starting to click, isn't it? Project 3 is where it gets interesting — you're building search from scratch. No frameworks. Just vectors and math."*
