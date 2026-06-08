# Project 1: Internal Prompt Testing Harness

- **Tier:** 1 — Internal Studio Tools
- **Project #:** 1 of 3
- **Tech Stack:** Python, OpenAI/Anthropic API, basic file storage (JSON/CSV)
- **Concepts:** Prompt engineering, structured outputs (Strict Mode / `json_schema`), manual eval, systematic testing
- **Quality Gate:** ✅ APPROVED when you can demonstrate that changing one word in a prompt produces measurably different quality — and you can prove it with data, not vibes.

---

## Phase 1: Brief (Priya)

> *Priya walks over to your desk. She looks tired.*

"Hey. Quick one.

The team keeps tweaking prompts for different clients. Maya changes a system prompt, Rohan changes it back, someone adds 'please' to see if it helps. We have **zero idea** whether any of these changes actually make things better or worse. We're just guessing.

Here's what I need: **an internal tool that tests prompt versions against expected outputs.**

You give it:
- A prompt (or a few variations)
- A set of test inputs with expected outputs
- A way to score how close the actual output matches the expected

It tells us:
- Which prompt version scores highest
- Where each version fails
- How consistent each version is

**Non-negotiable:** The results have to be structured. I need to show them to Rohan in a format he can't argue with. No screenshots, no 'it felt better.'

**Deadline:** End of week. The team's about to push changes to a client prompt and we need to know if we're making things worse.

**Definition of done from my perspective:** I can paste in 3 prompt variations, run the test, and get a ranked score with specific failure cases for each."

> *She taps your desk twice. "Don't overthink it. Start simple. Ship it."*

---

## Phase 2: Learning Path (Maya)

> *Maya appears over your shoulder about 30 seconds after Priya walks away.*

"She's right — don't overthink this. You don't need a database. You don't need a web app. You need a script that takes prompts and test cases and spits out scores. That's it.

Here's the order I want you to learn this:

### Learning Order (Scratch-First)

**Step 1: Call the LLM API with raw HTTP.**
Before you use any SDK, I want you to make a raw `POST` request to the API. Feel the boilerplate. See exactly what goes over the wire. See the JSON structure of the request and response.

> *"When you later use the OpenAI SDK, you'll know exactly what `client.chat.completions.create()` is doing under the hood. And when it breaks, you can debug it."*

**Step 2: Get structured JSON back.**
Now the fun part. You need the LLM to return data you can program against — not prose you have to eyeball. Learn the Structured Outputs pattern: `response_format` with `json_schema` and `strict: true`.

> *"Research reality (2026): JSON Mode (`json_object`) is now considered legacy. The production standard is `json_schema` with Strict Mode enabled. OpenAI's SDK even has a `.parse()` helper that takes Pydantic models directly. We'll use that."*

**Step 3: Build a test runner.**
A function that takes:
- A prompt template
- A list of `(input, expected_output)` pairs
- A scoring function

And returns a score for each test case. Start with exact match. Then add semantic similarity (LLM-as-judge). Feel the difference between them.

> *"This is the same pattern Promptfoo uses — now part of OpenAI, still MIT open source. You're basically building a miniature version of it. When you use Promptfoo later, you'll understand why it works the way it does."*

**Step 4: Compare prompt versions.**
Now wrap the whole thing: run multiple prompt versions against the same test set, produce a comparison report. Rank them.

### Memory Triage

**Memorize cold:**
- Messages array structure (system/user/assistant) — you'll write this every single day
- LLM API call pattern — `client.chat.completions.create(model=..., messages=..., response_format=...)`
- Structured Outputs parameter — `response_format={"type": "json_schema", "json_schema": {...}, "strict": true}`
- Pydantic model syntax — `class MyModel(BaseModel): field: str`

**Look up when needed:**
- Specific model pricing (it changes monthly)
- OpenAI SDK `.parse()` helper signature (great docs, no need to memorize)
- API rate limit tiers

**Understand deeply:**
- Why structured outputs matter — *"everything you build as an AI engineer will need the LLM to return typed, parseable data. If you can't enforce schema, you can't build reliable systems."*
- The difference between exact-match eval and LLM-as-judge — *"exact match is strict but misses semantically correct answers. LLM-as-judge is flexible but can be biased. You need to know when to use each."*

### First Concrete Step

> Open a terminal. Create a Python virtual environment. Install `openai` and `pydantic`. Write a script that sends a simple "translate this to French" request to the API using a raw HTTP POST request — NOT the SDK.

> *"Yes, I said raw HTTP first. Use `requests` or `urllib`. I want you to construct the JSON body yourself. This will take 10 minutes and save you hours of confusion later."*

### Resources (Just-in-Time)

- **OpenAI API Reference** — Chat Completions (use when writing your first API call)
- **OpenAI Structured Outputs Guide** (use when you need to return JSON)
- **OpenAI Cookbook: Structured Outputs Intro** — worked examples with Pydantic
- **Pydantic v2 Docs** — model definitions, validators, serialization (reference when needed)
- *Don't touch Promptfoo yet. You'll get there after you've built this manually.*

---

## Phase 3: The Build

> *You build. Maya guides. Rohan interrupts. Priya changes the requirements. You know, a normal week.*

### Milestone 1: The Raw API Call

Build a Python script that:
1. Sends a POST request to `https://api.openai.com/v1/chat/completions` with the `Authorization: Bearer $OPENAI_API_KEY` header
2. Constructs the full JSON body: `model`, `messages` (system + user), `max_tokens`
3. Parses the response JSON and prints just the assistant's message content

**Expected stuck point:** Your JSON body has a typo and the API returns a 400 error. Maya will ask: *"What did the API tell you? Read the error message — it tells you exactly what's wrong."*

**Maya's Socratic question** (after you get it working):
> *"You passed the API key in the header. What happens if someone accidentally commits this to GitHub? How would you prevent that?"*

> Answer they should arrive at: environment variables + `.env` file + `.gitignore`. This is a lesson they'll remember because they'll feel the anxiety of "I almost committed my API key."

### Milestone 2: Structured Outputs

Switch from raw HTTP to the OpenAI SDK. Then implement structured output:

1. Define a Pydantic model for your test result:
```python
from pydantic import BaseModel

class TestResult(BaseModel):
    prompt_input: str
    expected_output: str
    actual_output: str
    score: float  # 0.0 to 1.0
    failure_reason: str | None = None
```

2. Use the SDK's `client.beta.chat.completions.parse()` method (2026 standard) with your Pydantic model as the `response_format`.

3. For the actual prompt test, define what the LLM should return when you give it a test input:

```python
class PromptOutput(BaseModel):
    response: str
    confidence: float  # 0.0 to 1.0
    reasoning: str | None = None  # why it answered this way
```

> *"Schema-First Development is the 2026 standard. Define your schemas in Pydantic first, then build prompts around them. Not the other way around."*

**Expected stuck point:** The model sometimes refuses to answer (safety filter) and returns `refusal: true` instead of content. Your code crashes because `content` is `None`.

**Maya's Socratic question:**
> *"Your code just crashed because the LLM refused to answer. How would you handle this in production so the system doesn't fall over?"*

> They should learn: always check for the `refusal` field before parsing. This is a real production bug that teams hit constantly.

### Milestone 3: Test Runner

Build the core comparison engine:

1. A `run_test(prompt_template, test_cases, scoring_fn)` function
2. It runs each test case through the LLM with the given prompt template
3. Scores each output against the expected output
4. Returns a summary: average score, per-case breakdown, failure patterns

**Scoring functions to build:**
- `exact_match_scorer(expected, actual)` — strict string comparison
- `semantic_scorer(expected, actual)` — use LLM-as-judge: ask a separate LLM call "does this response match the expected output? Rate 0-1"

> *"Run the same test set with both scorers. You'll see cases where exact match fails but the answer is semantically correct. That's the exact problem Promptfoo and RAGAS solve. You just built a miniature version of both."*

**Expected stuck point:** LLM-as-judge is slow and expensive. Each test case needs an extra API call just to score.

**Maya's Socratic question:**
> *"Your LLM-as-judge just doubled your API costs. When is this worth it? When would you stick with exact match?"*

> They should: feel the cost tradeoff directly, understand that eval strategy is an economic decision.

### Rohan's Mid-Build Interruption

*Rohan appears suddenly, reading over your shoulder.*

> *"I see you're using 'score' as a float 0-1. How do you decide what's a PASS vs FAIL? 0.7? 0.8? And what happens when different scorers disagree? I need answers, not guesses."*

This forces the learner to think about **threshold calibration** — not just "it scores higher" but "how high is high enough."

### Priya's Requirement Change

*Two days in. You have a working test runner.*

> *Priya: "Quick update. Rohan wants to test across BOTH OpenAI and Anthropic. Some of our clients use Claude. Make the tool model-agnostic. You have one extra day."*

This forces the learner to abstract the model provider behind a common interface — a lesson in **API abstraction** that directly preps them for LiteLLM in Tier 2.

---

## Phase 4: Review (Rohan)

> *You submit your work with an `eval_results.json` showing 3 prompt versions tested against 20 test cases, with scores and failure analysis.*

Rohan opens your submission. Reads silently for 90 seconds.

### Decision Documentation Required

Submit with your code a brief explanation of:
1. Which scoring function you chose (exact match / semantic / both) and why
2. Where you set your PASS/FAIL threshold and how you decided
3. How you handled the model provider abstraction (OpenAI vs Anthropic)

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | Runs end-to-end, produces structured results |
| 2 | **Edge cases handled?** | 🔄 REVISE | What happens when the API is down? What if a test case is empty? What if the LLM returns a refusal? |
| 3 | **Cost-aware?** | 🔄 REVISE | You're re-running all tests every time. Cache results for unchanged prompt versions. Token logging is missing. |
| 4 | **Observable?** | 🔄 REVISE | When a test fails, I can't tell why. Log the actual vs expected output + the LLM's reasoning for each failure. |
| 5 | **Right approach?** | ✅ PASS | Solid. Starting with exact match before semantic scoring is the right progression. |
| 6 | **Decisions justified?** | ✅ PASS | Good explanation of scoring choice and rationale |
| 7 | **Measurable quality?** | ✅ PASS | You have scores. You can compare versions. This is measurable. |

### Verdict: 🔄 REVISE

*Rohan leans back.*

"This works. That's the baseline. But here's what needs fixing:

1. **Error handling** — Your script crashes on API errors. Wrap every API call in a try/except with exponential backoff retry. If the API is down for 30 seconds, I want it waiting, not dying.

2. **Result caching** — Save results to a JSON file keyed by `(prompt_hash, test_case_hash)`. If I run the same comparison twice, the second run should be instant. This is the same pattern production eval frameworks use.

3. **Failure logging** — For every failed test case, log: the prompt input, the expected output, the actual output, what the LLM scored, and any LLM reasoning. I need to be able to debug a single failure without re-running everything.

Fix these three and resubmit with an explanation of what you changed and why."

---

## Phase 5: Debrief (Maya)

> *After Rohan APPROVES your resubmission.*

**Maya:** "Good work. First project done. Here's what you should feel confident about now:

- You can call LLM APIs and handle the response
- You can enforce structured outputs with Pydantic + Strict Mode
- You understand the difference between exact-match eval and LLM-as-judge
- You've hit the real pain of prompt testing — and you know why tools like Promptfoo exist

**What you'll see again:**
- Structured outputs with Pydantic — **every single project from now on.** You'll be faster each time.
- LLM-as-judge eval — comes back in Project 4 in a much harder form (RAG evaluation)
- Model provider abstraction — you'll use LiteLLM for this in Project 7
- API error handling — you just learned a pattern that will save you in every project

**Quick note on what you didn't do:** You didn't build a UI. You didn't build a database. You made a CLI script with JSON file storage. That's the right call. In production, you start with the simplest thing that works, then add complexity only when the problem demands it.

> *Rohan approved your first project. That's not nothing. Most people don't get past their first REVISE without quitting.*

You're ready for Project 2. Priya's got the brief ready."
