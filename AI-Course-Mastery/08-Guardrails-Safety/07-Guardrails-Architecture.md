# 07 — Guardrails Architecture: Designing a Unified Multi-Layer Defense

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about system design diagrams and component architecture, you need to understand the BIGGEST architecture mistake in AI guardrails: **building each guardrail in isolation.**

In Files 02-06, you built individual guardrails:
- **File 02**: Prompt injection detection
- **File 03**: Input guardrails (user input safety)
- **File 04**: Output guardrails (LLM output safety)
- **File 05**: PII redaction
- **File 06**: Content moderation

Each of these works INDEPENDENTLY. But in production, they DON'T run independently. They run in a SINGLE pipeline, processing every user message and every LLM response.

And when you stack them together, something dangerous happens: **latency accumulates, costs multiply, and failure modes interact.**

```
Without coordination:
  User Input → [Injection Check: 200ms] → [Input Filter: 150ms] → [Moderation: 300ms]
              → [LLM: 2000ms] → [Output Filter: 150ms] → [Moderation: 300ms]
              → [PII Redaction: 100ms]
  Total latency: 3.2 seconds per message
  Total cost: $0.05 per message (LLM + guardrail LLM calls)
```

At 10K messages/day, that's **$1,500/month** and **3.2 seconds** of latency per message.

The naive approach — just plugging guardrails together — creates a system that's too slow, too expensive, and too fragile. You need ARCHITECTURE that optimizes across all guardrails simultaneously.

**The core challenge:** Guardrails are insurance. You pay for them (latency, cost, complexity) hoping you never need them. But if they're too expensive, you'll disable them. If they're too slow, users will leave. The architecture must make guardrails as INVISIBLE and INEXPENSIVE as possible while maintaining their effectiveness.

---

### 🤔 Discovery Question 1: The Stacking Problem

You have 5 guardrail modules. Each has:
- **Input guardrails**: Check user input before LLM (injection + input safety + PII + moderation)
- **Output guardrails**: Check LLM output before user (output safety + PII + moderation)

You run ALL input guardrails in parallel, ALL output guardrails in parallel:

```
Phase 1 (parallel): Injection + InputFilter + InputModeration + PII_Check
  → 4 checks in parallel, slowest one = latency
  → If slowest takes 300ms, Phase 1 = 300ms

Phase 2: LLM call (2000ms)

Phase 3 (parallel): OutputFilter + OutputModeration + PII_Check
  → 3 checks in parallel, slowest one = latency  
  → If slowest takes 300ms, Phase 3 = 300ms

Total: 300ms + 2000ms + 300ms = 2600ms
```

**🤔 Before reading on, answer these:**

1. Parallelizing sounds efficient — all checks run simultaneously. But what if ONE check (say, LLM-based content moderation in Phase 1) takes 800ms because the model is slow? Now Phase 1 = 800ms. The other 3 checks finished in 50ms but they're waiting for the slow one. Was parallelization the right choice? Or should the slow check be deferred to a background process?

2. **PHASE ORDER MATTERS.** What if you run the INJECTION check AFTER the LLM call? The LLM might have already been compromised by the time injection detection runs. What if you run PII redaction BEFORE the LLM? The LLM gets sanitized input but might regenerate new PII. What's the RIGHT order?

3. **The hardest question:** What if the guardrails CONTRADICT each other? Example:
   - Input moderation says: "Block this — it contains hate speech"
   - PII check says: "Also block this — it contains a phone number"
   - The guardrails aggregate: "Blocked for 2 reasons"
   
   But what if:
   - Input safety says: "Block this — it's an injection attack" (TRUE — it IS an injection)
   - Input moderation says: "Allow this — it's a normal question" (TRUE — it's also a normal question wrapped in an injection)
   
   How do you resolve CONFLICTING guardrail decisions? Which guardrail wins? What's the aggregation logic?

---

### 🤔 Discovery Question 2: The Monitoring Blind Spot

Your guardrails are running in production. They seem to work. User complaints are down. Support tickets about harmful content are down. Your team is happy.

Six months later, a journalist publishes an investigation: your platform has been generating hate speech for 3 months and nobody noticed. Your guardrails never reported it because:

- The guardrail monitoring dashboard only shows "total blocked" and "total allowed"
- "Total blocked" has been steady at 2% of all messages
- But here's what the monitoring MISSED:
  - 2 months ago, your LLM provider updated their model — it now generates more subtle hate speech
  - Your output guardrails (trained on old model outputs) catch LESS than before
  - But "total blocked" stayed at 2% because user-generated hate speech INCREASED at the same time
  - The gauge showed 2% — unchanged — so nobody investigated

**🤔 Before reading on, answer these:**

1. Your monitoring dashboard shows "total blocked = 2%". This is a single aggregate number that hides TWO opposite trends cancelling out. What INDIVIDUAL METRICS should you track to catch this? How would you decompose "total blocked" into actionable signals?

2. "Total blocked" and "total allowed" measure what the guardrails DO, not how well they work. If a guardrail misses 50% of harmful content, "total blocked" goes DOWN — but the system looks like it's blocking LESS (which seems like a good thing to the untrained observer!). What metrics actually measure guardrail PERFORMANCE vs. guardrail OUTPUT?

3. **The hardest question:** You need to know if a guardrail is getting WORSE. But how do you know what a guardrail MISSED? The whole point of guardrails is to catch bad content — you see what they catch, but you DON'T see what they miss (that's the "unknown unknowns" problem). How do you estimate the FALSE NEGATIVE rate of a guardrail when you can't observe false negatives directly?

---

### 🤔 Discovery Question 3: The Fallback Chain

Your guardrail pipeline has 5 stages. Stage 3 (output moderation) depends on an external API (OpenAI Moderation API). The API has a 5-minute outage.

What happens?

```
Option A: System blocks ALL responses until API recovers
  → Users: "Why can't I get any responses?"
  → Business: "We're losing $10K/minute in revenue"
  → Team: "Get it working NOW!"

Option B: System bypasses the failed guardrail and continues
  → Users: Get responses (good)
  → Safety: Some harmful content slips through (bad)
  → Team: "Fixing the API, but we had 47 safety incidents during the outage"

Option C: System falls back to a SIMPLER guardrail
  → Users: Get responses, but with more false positives (conservative mode)
  → Safety: Fewer false negatives than bypassing, more false positives than normal
  → Team: "Hit the failsafe — we're running on keyword-only moderation"
```

**🤔 Before reading on, answer these:**

1. Each of these options is BAD in different ways. Option A is terrible for business. Option B is terrible for safety. Option C is a compromise — less safe than normal, more functional than blocked. **What determines whether Option C is acceptable?** How degraded is "acceptable degraded"?

2. Not all guardrails are equal. If PII redaction fails (Option B — bypass), the worst case is someone's phone number leaks. If OUTPUT MODERATION fails (Option B — bypass), the worst case is hate speech reaches a user. **Should different guardrails have different fallback strategies?** Which guardrails can be bypassed safely? Which can't?

3. **The hardest question:** The external API is failing. But is the failure:
   - A TRANSIENT blip (network hiccup, retry in 2 seconds)?
   - A SUSTAINED outage (API down for 30 minutes)?
   - A SILENT degradation (API returns responses, but they're wrong)?
   
   Each requires a DIFFERENT fallback strategy. But the guardrail system doesn't know which one it's experiencing. **How do you design fallback logic that's adaptive to failure types?** What signals distinguish transient vs. sustained vs. silent failures?

---

By the end of this module, you will:

1. **Design a multi-layer defense architecture** — How all 5 guardrail types (Files 02-06) compose into a unified pipeline
2. **Build a phased detection pipeline** — What runs when, in what order, with what latency budget
3. **Implement coordination logic** — Conflict resolution between guardrails, aggregation strategies
4. **Design fallback & degradation** — Circuit breakers, failover strategies, graceful degradation
5. **Build meta-monitoring** — Monitoring the guardrails themselves (latency, accuracy, drift)
6. **Calculate total system cost** — Latency budget, cost budget, resource allocation across guardrails
7. **Design for security** — Defense in depth, failure independence, minimal blast radius

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 30 min |
| Multi-Layer Defense Architecture | 30 min |
| Phased Detection Pipeline Design | 25 min |
| Code: Unified Guardrails Orchestrator | 40 min |
| Code: Fallback Manager | 35 min |
| Code: Meta-Monitoring | 30 min |
| Code: Circuit Breaker & Degradation | 25 min |
| Conflict Resolution Strategies | 20 min |
| Latency Budget & Cost Modeling | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 60 min |
| Gate Check | 15 min |
| **Total** | **~6 hours** |

---

## 📖 Concept Deep-Dive

### 1. Multi-Layer Defense Architecture

The core principle: **Defense in Depth.** No single guardrail is perfect. Multiple independent layers ensure that if one fails, another catches it.

```
LLM APPLICATION GUARDRAIL LAYERS:

Layer 0: INFRASTRUCTURE SECURITY (outside scope, but essential)
  └── Network security, authentication, rate limiting, DDoS protection
  └── Not LLM-specific, but every application needs these

Layer 1: INPUT GATE (Before LLM)
  └── [1A] Injection Detection (File 02)
  │     Purpose: Block prompt injection, jailbreak attempts
  │     Failure mode: Misses novel injection
  │     Dependency: None (runs first, on raw input)
  │
  └── [1B] Input Content Safety (File 03)
  │     Purpose: Block harmful user input
  │     Failure mode: False negative on borderline content
  │     Dependency: None (runs on raw input)
  │
  └── [1C] Input PII Detection (File 05)
  │     Purpose: Detect PII in user input (for logging decisions)
  │     Failure mode: Misses context-dependent PII
  │     Dependency: None
  │
  └── [1D] Input Content Moderation (File 06)
        Purpose: Classify input into safety categories
        Failure mode: Evasion attacks
        Dependency: None

Layer 2: LLM INVOCATION (The generative step)
  └── Safe prompt construction (system prompt, few-shot examples)
  └── Structured output formatting (JSON schema enforcement)
  └── Token budget enforcement

Layer 3: OUTPUT GATE (After LLM, Before User)
  └── [3A] Output Content Safety (File 04)
  │     Purpose: Block harmful LLM output
  │     Failure mode: False negative on non-safety issues
  │     Dependency: Requires LLM output to check
  │
  └── [3B] Output Consistency (File 04)
  │     Purpose: Check output against sources (hallucination)
  │     Failure mode: Misses plausible-sounding hallucinations
  │     Dependency: Requires LLM output + retrieved sources
  │
  └── [3C] Output PII Redaction (File 05)
  │     Purpose: Redact PII the LLM generated
  │     Failure mode: Misses inferred PII
  │     Dependency: Requires LLM output
  │
  └── [3D] Output Content Moderation (File 06)
        Purpose: Classify LLM output into safety categories
        Failure mode: Evasion attacks
        Dependency: Requires LLM output

Layer 4: ASYCHRONOUS AUDIT (Post-hoc, non-blocking)
  └── [4A] Full LLM-Judge Evaluation (Phase 7)
  │     Purpose: Deep evaluation of every N^th interaction
  │     Latency: Background (doesn't affect user)
  │
  └── [4B] Human Review Queue (File 06)
  │     Purpose: Human review of flagged content
  │     Latency: Minutes to hours (non-blocking)
  │
  └── [4C] Compliance Logging (File 05)
        Purpose: GDPR/HIPAA audit trail
        Latency: Background
```

**Key principle: FAILURE INDEPENDENCE.**
- Each layer should work INDEPENDENTLY
- A failure in Layer 1 should NOT cascade to Layer 3
- A failure in Layer 3 should NOT affect Layer 4
- This means: separate processes, separate timeouts, separate dependencies

**The "L" shape defense:**
```
                    ┌─────────────────────────────┐
                    │         LLM CALL            │
                    │  (most expensive, slowest)   │
                    └──────────┬──────────────────┘
                               │
INPUT ← [Fast checks] →  <LLM> → [Fast checks] → OUTPUT
        (cheap, quick)           (cheap, quick)
        
        [Slow checks]     ────   [Slow checks]
        (costly, deep)            (costly, deep)
        run in parallel with      run in parallel with
        LLM where possible        LLM where possible
```

The "L" shape means: fast checks run SYNCHRONOUSLY (blocking), slow checks run ASYNCHRONOUSLY or in PARALLEL with LLM invocation where possible.

### 2. Phased Detection Pipeline

The pipeline is divided into PHASES, each with a specific purpose:

```
PHASE 0: PRE-FILTER (< 5ms, zero cost)
──────────────────────────────────────
Purpose: Drop obviously harmful inputs before any processing
Techniques: 
  - Blocked IP/account list
  - Rate limit check
  - Known spam pattern regex
  - Input size validation
If blocked: Return immediately, no further processing

PHASE 1: SYNCHRONOUS LIGHT (< 100ms, $0.0001)
──────────────────────────────────────
Purpose: Fast checks that run on every message
Techniques:
  - Normalization (leetspeak, unicode, casing)
  - Keyword/pattern matching (Tier 1 moderation)
  - Fast injection pattern detection
  - PII regex scan (quick scan)
  - Content size budgeting
Result: 
  - CLEAR → Proceed to Phase 2
  - BLOCKED_HIGH_CONF → Block immediately (no Phase 2)
  - UNCERTAIN → Flag for Phase 2

PHASE 2: SYNCHRONOUS DEEP (< 500ms, $0.002)
──────────────────────────────────────
Purpose: Deeper checks for content that passed Phase 1
Techniques:
  - ML classifier (Tier 2 moderation)
  - LLM-based injection detection (File 02)
  - Context-aware PII detection (File 05)
  - Multi-turn risk assessment (File 03)
Result:
  - CLEAR → Proceed to LLM
  - BLOCKED → Block, log, return
  - BORDERLINE → Flag, proceed to LLM but with restrictions

PHASE 3: LLM INVOCATION
──────────────────────────────────────
Purpose: Generate the response
Techniques:
  - Safe system prompt with guardrail instructions
  - Structured output (JSON schema)
  - Token budget enforcement
  - Timeout management

PHASE 4: POST-LLM LIGHT (< 100ms, $0.0001)
──────────────────────────────────────
Purpose: Fast checks on LLM output
Techniques:
  - Output structure validation (is it valid JSON?)
  - Response size limits
  - Fast PII pattern check on output
  - Response category check (did LLM answer the right type of question?)
Result:
  - PASS → Proceed to Phase 5
  - FAIL → Trigger repair/regeneration (File 04)

PHASE 5: POST-LLM DEEP (< 500ms, $0.003)
──────────────────────────────────────
Purpose: Deep checks on LLM output before user sees it
Techniques:
  - Consistency checker (File 04 — does output match sources?)
  - Output safety classifier (File 04)
  - Content moderation (File 06)
  - PII redaction with context (File 05)
  - LLM-as-judge for quality
Result:
  - PASS → Send to user
  - FAIL → Repair → Send repaired version OR regenerate OR fallback

PHASE 6: ASYNCHRONOUS AUDIT (Background, no latency impact)
──────────────────────────────────────
Purpose: Post-hoc analysis, never blocks user response
Techniques:
  - Full LLM-as-judge evaluation (Phase 7)
  - Human review queue (File 06)
  - Compliance/PII audit logging
  - Guardrail performance metrics
  - Anomaly detection on guardrail patterns
```

### 3. Latency Budget

Every millisecond is allocated. Nothing runs "whenever":

```
TOTAL LATENCY BUDGET: 3000ms (3 seconds) max for complete request

BREAKDOWN:
                                         Sequential Path        Async/Parallel
Phase 0: Pre-filter                          5ms                  —
Phase 1: Synchronous Light                   100ms                —
Phase 2: Synchronous Deep                    500ms                1,000ms parallel
Phase 3: LLM Invocation                     2000ms                —
Phase 4: Post-LLM Light                      100ms                —
Phase 5: Post-LLM Deep                       500ms                1,000ms parallel
Phase 6: Async Audit                          —                   unlimited

Sequential total: 5 + 100 + 500 + 2000 + 100 + 500 = 3205ms
                                                    ↓
With parallelization: 2000ms (LLM) + 500ms (Phase 2 in parallel) + 500ms (Phase 5) + overhead
                                                    ↓
                              Total: ~2500-2800ms (under 3s budget)
```

**The critical insight:** Phases 2, 3, and 5 can overlap. While the LLM runs in Phase 3, Phase 2's deep checks and Phase 5 preparations can proceed in parallel (checking cached patterns, loading models, warming caches). This "pipelined" approach keeps total latency close to the LLM call time, not the sum of all guardrail times.

### 4. Guardrail Coordination & Conflict Resolution

When multiple guardrails disagree, who wins?

```
CONFLICT RESOLUTION MATRIX:

Check 1 Result    │ Check 2 Result    │ Tiebreaker      │ Action
──────────────────┼───────────────────┼─────────────────┼────────────
ALLOW             │ ALLOW             │ —               │ ALLOW
ALLOW             │ BLOCK             │ Trust BLOCK     │ BLOCK (conservative)
BLOCK             │ ALLOW             │ Check Severity  │ Depends
BLOCK             │ BLOCK             │ —               │ BLOCK
UNCERTAIN         │ UNCERTAIN         │ —               │ FLAG for review
BLOCK (Tier 1)    │ ALLOW (Tier 3)    │ Trust Tier 3    │ Follow higher tier
                  │                   │ if confidence   │
                  │                   │ gap > 0.3       │

CONFLICT RULES (in priority order):

Rule 1: SAFETY OVER FUNCTION
  If ANY guardrail says BLOCK with confidence > 0.95 → BLOCK
  Even if other guardrails say ALLOW
  Rationale: A single high-confidence catch beats multiple low-confidence misses

Rule 2: HIGHER TIER WINS (with threshold)
  If Tier 1 BLOCK (conf 0.8) and Tier 3 ALLOW (conf 0.7) → Follow Tier 3
  If Tier 1 BLOCK (conf 0.8) and Tier 3 ALLOW (conf 0.95) → ALLOW (Tier 3 is very sure)
  Rationale: More sophisticated detectors should override simpler ones, but only when they're confident

Rule 3: MOST SEVERE CATEGORY WINS
  If one guardrail catches HATE_SPEECH (0.7) and another catches SPAM (0.95) → Follow HATE_SPEECH
  Rationale: Block for the more dangerous category, even if the other is higher confidence

Rule 4: ESCALATE ON DISAGREEMENT
  If Tier 2 says BLOCK (0.6) and Tier 3 says ALLOW (0.6) → Neither confident → FLAG for human review
  Rationale: When AIs disagree and neither is sure, humans decide
```

### 5. Circuit Breakers & Graceful Degradation

Every external dependency (model API, moderation API, classifier service) can fail. The guardrail system must NOT fail when a dependency fails.

```
CIRCUIT BREAKER STATES:

CLOSED (Normal operation)
  ├── Requests pass through normally
  ├── Error rate monitored
  └── If error rate > threshold for N seconds → OPEN

OPEN (Tripped — dependency degraded)
  ├── Requests BLOCKED (fail fast — don't wait for timeout)
  ├── Fallback strategy activated
  └── After cooldown period → HALF_OPEN

HALF_OPEN (Testing recovery)
  ├── Allow limited requests through
  ├── Monitor success rate
  ├── If success rate recovers → CLOSED
  └── If failure continues → OPEN (reset cooldown)

FALLBACK STRATEGIES BY SEVERITY:

Guardrail Failure │ Fallback Strategy │ Safety Impact │ User Impact
──────────────────┼───────────────────┼───────────────┼─────────────
Injection Detection│ Conservative mode │ Higher false  │ Some legit
(fails)           │ (block uncertain)  │ positive rate  │ queries blocked
                  │                    │ (safer than    │
                  │                    │ false negative)│
                  │                    │                │
PII Redaction     │ Bypass — log only  │ PII may leak  │ Normal UX
(fails)           │ (non-blocking      │ (needs         │
                  │  guardrail)         │  post-hoc      │
                  │                    │  cleanup)      │
                  │                    │                │
Output Safety     │ Block ALL output   │ No harmful     │ No responses
(fails)           │ (fail-closed)      │ output (safe)  │ (bad UX)
                  │                    │                │
Content           │ Fallback to        │ Less nuanced   │ More false
Moderation (fails)│ keyword-only       │ detection      │ positives
                  │ filtering          │                │

FAIL-CLOSED vs FAIL-OPEN decision per guardrail:

FAIL-CLOSED (block when uncertain or unavailable):
  - Output safety filter: If unsure, DON'T show content to user
  - Self-harm detection: If failed, assume it's self-harm
  - Injection detection on CRITICAL endpoints: If failed, block
  
FAIL-OPEN (allow when uncertain or unavailable):
  - PII redaction: If failed, log it and proceed (PII is a leak, not active harm)
  - Content moderation on NON-CRITICAL content: If failed, flag for async review
  - Spam detection: If failed, allow and review later
```

### 6. Meta-Monitoring (Who Guards the Guardians?)

The meta-monitoring system watches the guardrails themselves:

```
GUARDRAIL HEALTH METRICS:

PER-GUARDRAIL:
  ┌──────────────────────────────────────────────┐
  │ GUARDRAIL: Injection Detection               │
  ├──────────────────────────────────────────────┤
  │ Block Rate:      2.3% (of all requests)      │
  │ Allow Rate:     97.7%                        │
  │ P50 Latency:     68ms                        │
  │ P99 Latency:    312ms                        │
  │ Error Rate:     0.02%                        │
  │ Circuit Breaker: CLOSED                       │
  │ Last Updated:    12s ago                     │
  └──────────────────────────────────────────────┘

CROSS-GUARDRAIL METRICS:
  ┌──────────────────────────────────────────────┐
  │ CORRELATION ANALYSIS                         │
  ├──────────────────────────────────────────────┤
  │ Injection + Input Safety agreement:      94% │
  │ Injection + Moderation agreement:        82% │ (expected — different purposes)
  │ Input Safety + Output Safety agreement:   —  │ (not comparable — different domains)
  │                                               │
  │ GUARDRAIL DRIFT DETECTION:                    │
  │ Injection block rate:  2.3% → 2.4% (stable)  │
  │ PII redaction rate:    1.1% → 3.8%  (⚠️ UP)  │
  │  → Investigation: Did system prompt change?   │
  │  → Cause: LLM now generates more account      │
  │    numbers in responses                       │
  └──────────────────────────────────────────────┘

ACCURACY ESTIMATION (the impossible problem):
  ┌──────────────────────────────────────────────┐
  │ ESTIMATED FALSE NEGATIVE RATE                │
  ├──────────────────────────────────────────────┤
  │ Method: Random sampling of "ALLOWED" content  │
  │ Sample rate: 0.1% of passed content           │
  │ Reviewer: LLM-as-judge (Phase 7)              │
  │                                               │
  │ Injection:  0.3% FN rate (3 missed/1000)     │
  │ PII:        0.1% FN rate (1 missed/1000)     │
  │ Moderation: 0.5% FN rate (5 missed/1000)     │
  │                                               │
  │ NOTE: This is ESTIMATION, not ground truth.   │
  │ The LLM judge also has errors. Actual FN rate │
  │ could be higher.                              │
  └──────────────────────────────────────────────┘
```

**The unknown unknowns problem** (from Discovery Question 2):
You can NEVER know your true false negative rate because you don't see what you missed. But you can APPROXIMATE it:

1. **Random sampling**: Review a random 0.1% of ALLOWED content. Any misses found → estimate FN rate.
2. **User reports**: Track "report this response" clicks. Each report is a POTENTIAL false negative.
3. **External audit**: Periodic third-party review (expensive but gold standard).
4. **Red team testing**: Automated adversarial testing (File 08) to probe for weaknesses.

---

## 💻 CODE EXAMPLES

### Example 1: The Naive Orchestrator (What NOT To Do)

```python
# BAD: Sequential, no coordination, no fallback, no monitoring

class NaiveGuardrailPipeline:
    """
    A naive pipeline that runs EVERYTHING sequentially.
    
    Problems:
    1. Runs ALL checks on every message (no tiering)
    2. No parallelization (each check waits for previous)
    3. No circuit breakers (external API failure = pipeline failure)
    4. No latency budget (as slow as the slowest check + all others)
    5. No conflict resolution (last check's decision wins)
    6. No fallback (if any check fails, the whole thing fails)
    7. No monitoring (can't tell if guardrails are getting worse)
    """
    
    def __init__(self):
        self.injection_detector = InjectionDetector()
        self.input_filter = InputFilter()
        self.pii_redactor = PIIRedactor()
        self.content_moderator = ContentModerator()
        self.output_filter = OutputFilter()
        self.consistency_checker = ConsistencyChecker()
    
    def process(self, user_input: str) -> str:
        # PROBLEM 1: Sequential — each step waits for the previous
        # Total time = sum of ALL checks, not max of parallel
        
        # Step 1: Injection check
        injection_result = self.injection_detector.check(user_input)
        if injection_result.is_attack:
            return "I can't process that request."
        
        # Step 2: Input filter
        input_result = self.input_filter.check(user_input)
        if input_result.is_unsafe:
            return "I can't process that request."
        
        # Step 3: PII check (on input)
        pii_result = self.pii_redactor.scan(user_input)
        # ... just logs it, doesn't actually use it
        
        # Step 4: Content moderation
        mod_result = self.content_moderator.classify(user_input)
        if mod_result.is_harmful:
            return "I can't process that request."
        
        # Step 5: Call LLM
        llm_response = self.call_llm(user_input)
        
        # Step 6: Output filter
        output_result = self.output_filter.check(llm_response)
        if output_result.is_unsafe:
            return "I couldn't generate a safe response."
        
        # PROBLEM 2: No conflict resolution
        # If injection says BLOCK and input says ALLOW, last check wins
        # The decision is ORDER-DEPENDENT!
        
        # Step 7: Consistency check
        if not self.consistency_checker.verify(llm_response):
            return "I couldn't generate a consistent response."
        
        # Step 8: PII redaction on output
        safe_response = self.pii_redactor.redact(llm_response)
        
        return safe_response
    
    def call_llm(self, prompt: str) -> str:
        # PROBLEM 3: No timeout, no fallback
        # If LLM API fails, the entire pipeline fails
        response = openai.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )
        return response.choices[0].message.content


# PROBLEMS WITH THIS APPROACH:

# Problem 1: Latency
# Injection: 100ms + InputFilter: 150ms + PII: 50ms + Moderation: 300ms
#   → Total before LLM: 600ms sequential (could have been 300ms parallel)
# LLM: 2000ms
# OutputFilter: 150ms + Consistency: 200ms + PII: 50ms
#   → Total after LLM: 400ms sequential
# TOTAL: 3000ms = 3 seconds (barely acceptable)
# WITHOUT PARALLELIZATION!

# Problem 2: Brittle — any failure crashes the pipeline
# If moderation API is down → EVERY request fails
# No circuit breaker → every request times out waiting for failed API
# No fallback → no degraded mode

# Problem 3: No feedback loop
# If injection detection starts failing (missing attacks), nobody knows
# No metrics tracking → guardrail performance drifts silently
```

### 🔍 Find The Bugs

Before reading the fixed version, think about:

1. **The latency bug**: This runs ALL 4 input checks sequentially. Which checks can actually run IN PARALLEL? Which check MUST run before others? (Hint: Do injection detection and input moderation depend on each other?)

2. **The fail-fast bug**: If the LLM API call FAILS (timeout, error), what happens? The exception propagates up and the user gets an error. No retry logic. No cached fallback. No graceful degradation.

3. **The monitoring bug**: This pipeline has ZERO metrics. If the injection detection rate drops from 5% to 0.5%, is that because there are fewer attacks (good) or because the detector is broken (bad)? Without metrics, you CAN'T tell.

4. **The coordination bug**: What if injection check says BLOCK (it detected a known injection pattern) but moderation says ALLOW (the content appears safe). Which wins? The current code returns at the first BLOCK, so injection wins — but is that always right?

---

### Example 2: Production Guardrails Orchestrator

```python
"""
Production Guardrails Orchestrator

Architecture:
- 6 phases with explicit latency budgets
- Parallel execution where possible
- Circuit breakers for all external dependencies
- Configurable fallback strategies per guardrail
- Comprehensive metrics collection
- Conflict resolution with policy-based tiebreakers
"""

import asyncio
import enum
import time
import json
import logging
from dataclasses import dataclass, field
from typing import Optional, Callable, Any
from enum import Enum
from datetime import datetime
from collections import defaultdict


class GuardrailDecision(Enum):
    ALLOW = "allow"
    BLOCK = "block"
    FLAG = "flag"
    ESCALATE = "escalate"
    BYPASS = "bypass"  # Guardrail was unavailable, bypassed


class GuardrailSeverity(Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"


class CircuitState(Enum):
    CLOSED = "closed"        # Normal operation
    OPEN = "open"            # Failing — no requests pass
    HALF_OPEN = "half_open"  # Testing recovery


@dataclass
class GuardrailResult:
    guardrail_name: str
    decision: GuardrailDecision
    confidence: float
    category: Optional[str] = None
    severity: Optional[GuardrailSeverity] = None
    latency_ms: float = 0.0
    error: Optional[str] = None
    details: Optional[dict] = None
    timestamp: datetime = field(default_factory=datetime.now)


@dataclass
class PipelineResult:
    allowed: bool
    final_decision: GuardrailDecision
    results: list[GuardrailResult] = field(default_factory=list)
    total_latency_ms: float = 0.0
    llm_latency_ms: float = 0.0
    conflict_resolution: Optional[str] = None
    fallback_used: bool = False
    response: Optional[str] = None


class CircuitBreaker:
    """
    Circuit breaker for external guardrail dependencies.
    
    Prevents cascading failures by failing fast when a dependency
    is degraded, giving it time to recover.
    """
    
    def __init__(self, name: str,
                 failure_threshold: int = 5,
                 recovery_timeout: float = 30.0,
                 half_open_max_requests: int = 3):
        self.name = name
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_requests = half_open_max_requests
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time: Optional[datetime] = None
        self.half_open_requests = 0
        self.total_failures = 0
        self.total_successes = 0
    
    async def call(self, fn: Callable, fallback: Callable,
                   *args, **kwargs) -> tuple[Any, bool]:
        """
        Call a guardrail function with circuit breaker protection.
        
        Returns: (result, used_fallback)
        """
        if self.state == CircuitState.OPEN:
            # Check if recovery timeout has elapsed
            if self.last_failure_time:
                elapsed = (datetime.now() - self.last_failure_time).total_seconds()
                if elapsed >= self.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                    self.half_open_requests = 0
                else:
                    # Circuit is open — use fallback
                    return await fallback(*args, **kwargs), True
        
        try:
            result = await fn(*args, **kwargs)
            
            if self.state == CircuitState.HALF_OPEN:
                self.half_open_requests += 1
                if self.half_open_requests >= self.half_open_max_requests:
                    self.state = CircuitState.CLOSED
                    self.failure_count = 0
            
            self.total_successes += 1
            self.failure_count = 0
            return result, False
            
        except Exception as e:
            self.failure_count += 1
            self.total_failures += 1
            self.last_failure_time = datetime.now()
            
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
            
            # Use fallback
            return await fallback(*args, **kwargs), True


class GuardrailsOrchestrator:
    """
    Unified guardrails orchestrator with:
    - Phased pipeline (light → heavy checks)
    - Parallel execution per phase
    - Circuit breakers for dependencies
    - Conflict resolution
    - Comprehensive metrics
    - Graceful degradation
    """
    
    def __init__(self, config: dict):
        self.config = config
        
        # Guardrail modules (injected from Files 02-06)
        self.guardrails = {}
        self.circuit_breakers = {}
        
        # Metrics
        self.metrics = {
            "total_processed": 0,
            "total_blocked": 0,
            "total_allowed": 0,
            "total_flags": 0,
            "total_fallbacks": 0,
            "total_errors": 0,
            "guardrail_latencies": defaultdict(list),
            "guardrail_decisions": defaultdict(lambda: defaultdict(int)),
            "circuit_breaker_trips": defaultdict(int),
            "phase_latencies": defaultdict(list),
        }
        
        # Latency budget (from config)
        self.latency_budget_ms = config.get("latency_budget_ms", 3000)
        self.llm_timeout_ms = config.get("llm_timeout_ms", 2000)
        
        # Phase time allocations
        self.phase_budgets = {
            "pre_filter": config.get("phase_pre_filter_ms", 5),
            "light_check": config.get("phase_light_ms", 100),
            "deep_check": config.get("phase_deep_ms", 500),
            "llm_call": config.get("phase_llm_ms", 2000),
            "post_light": config.get("phase_post_light_ms", 100),
            "post_deep": config.get("phase_post_deep_ms", 500),
            "async_audit": config.get("phase_async_ms", 5000),
        }
    
    def register_guardrail(self, name: str, 
                           check_fn: Callable,
                           phase: str,
                           severity: GuardrailSeverity,
                           fallback_fn: Optional[Callable] = None,
                           timeout_ms: float = 500):
        """
        Register a guardrail in the pipeline.
        
        Args:
            name: Unique guardrail name
            check_fn: Async function that checks content
            phase: Which phase this guardrail belongs to
            severity: How important this guardrail is
            fallback_fn: Fallback when guardrail fails
            timeout_ms: Max time before timeout
        """
        self.guardrails[name] = {
            "fn": check_fn,
            "phase": phase,
            "severity": severity,
            "fallback_fn": fallback_fn,
            "timeout_ms": timeout_ms,
        }
        
        self.circuit_breakers[name] = CircuitBreaker(
            name=name,
            failure_threshold=self.config.get("cb_failure_threshold", 5),
            recovery_timeout=self.config.get("cb_recovery_timeout", 30),
        )
    
    async def process(self, user_input: str, 
                      context: Optional[dict] = None) -> PipelineResult:
        """
        Process a user message through the full guardrails pipeline.
        
        Flow:
        Phase 0: Pre-filter (blocklist, rate limit)
        Phase 1: Light synchronous checks (pattern match, regex)
        Phase 2: Deep synchronous checks (ML, small LLM)
        Phase 3: LLM invocation
        Phase 4: Post-LLM light checks
        Phase 5: Post-LLM deep checks
        Phase 6: Async audit (non-blocking)
        
        Returns PipelineResult with decisions and latency metrics.
        """
        start_time = time.time()
        all_results: list[GuardrailResult] = []
        self.metrics["total_processed"] += 1
        
        # Phase 0: Pre-filter (fastest, always runs first)
        phase0_results = await self._run_phase("pre_filter", user_input)
        all_results.extend(phase0_results)
        
        if self._should_block(phase0_results):
            return self._decision(user_input, all_results, start_time)
        
        # Phase 1: Light synchronous checks
        phase1_results = await self._run_phase("light_check", user_input)
        all_results.extend(phase1_results)
        
        if self._should_block(phase1_results):
            return self._decision(user_input, all_results, start_time)
        
        # Phase 2: Deep synchronous checks (parallel with each other)
        phase2_results = await self._run_phase_parallel("deep_check", user_input)
        all_results.extend(phase2_results)
        
        # Check if Phase 2 blocked — if so, no LLM call needed
        if self._should_block(phase2_results):
            return self._decision(user_input, all_results, start_time)
        
        # Phase 3: LLM invocation (with timeout)
        llm_start = time.time()
        llm_response = await self._call_llm(user_input, context)
        llm_latency = (time.time() - llm_start) * 1000
        
        if llm_response is None:
            # LLM failed — return error
            return PipelineResult(
                allowed=False,
                final_decision=GuardrailDecision.BYPASS,
                results=all_results,
                total_latency_ms=(time.time() - start_time) * 1000,
                llm_latency_ms=llm_latency,
                response=None,
                fallback_used=True,
            )
        
        # Phase 4: Post-LLM light checks
        phase4_results = await self._run_phase("post_light", llm_response)
        all_results.extend(phase4_results)
        
        if self._should_block(phase4_results):
            # Output was bad — try repair or regenerate
            repaired = await self._try_repair(llm_response, phase4_results)
            if repaired:
                llm_response = repaired
            else:
                return self._decision(
                    user_input, all_results, start_time,
                    response=None, llm_latency=llm_latency
                )
        
        # Phase 5: Post-LLM deep checks
        phase5_results = await self._run_phase_parallel("post_deep", llm_response)
        all_results.extend(phase5_results)
        
        if self._should_block(phase5_results):
            repaired = await self._try_repair(llm_response, phase5_results)
            if repaired:
                llm_response = repaired
            else:
                return self._decision(
                    user_input, all_results, start_time,
                    response=None, llm_latency=llm_latency
                )
        
        # Phase 6: Async audit (fire and forget — never blocks)
        asyncio.create_task(
            self._run_async_audit(user_input, llm_response, all_results)
        )
        
        # All checks passed
        self.metrics["total_allowed"] += 1
        
        return PipelineResult(
            allowed=True,
            final_decision=GuardrailDecision.ALLOW,
            results=all_results,
            total_latency_ms=(time.time() - start_time) * 1000,
            llm_latency_ms=llm_latency,
            response=llm_response,
        )
    
    async def _run_phase(self, phase: str, content: str) -> list[GuardrailResult]:
        """Run all guardrails in a phase SEQUENTIALLY (light, fast phases)."""
        results = []
        phase_start = time.time()
        
        for name, guardrail in self._get_phase_guardrails(phase):
            result = await self._run_guardrail(name, guardrail, content)
            results.append(result)
        
        self.metrics["phase_latencies"][phase].append(
            (time.time() - phase_start) * 1000
        )
        return results
    
    async def _run_phase_parallel(self, phase: str, 
                                   content: str) -> list[GuardrailResult]:
        """Run all guardrails in a phase in PARALLEL (expensive checks)."""
        phase_start = time.time()
        
        tasks = []
        for name, guardrail in self._get_phase_guardrails(phase):
            tasks.append(self._run_guardrail(name, guardrail, content))
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Handle exceptions (shouldn't happen due to circuit breakers)
        processed = []
        for r in results:
            if isinstance(r, Exception):
                processed.append(GuardrailResult(
                    guardrail_name="unknown",
                    decision=GuardrailDecision.BYPASS,
                    confidence=0.0,
                    error=str(r),
                ))
            else:
                processed.append(r)
        
        self.metrics["phase_latencies"][phase].append(
            (time.time() - phase_start) * 1000
        )
        return processed
    
    async def _run_guardrail(self, name: str, guardrail: dict,
                              content: str) -> GuardrailResult:
        """Run a single guardrail with circuit breaker protection."""
        guardrail_start = time.time()
        
        async def check():
            return await guardrail["fn"](content)
        
        async def fallback():
            if guardrail["fallback_fn"]:
                fallback_result = await guardrail["fallback_fn"](content)
                self.metrics["total_fallbacks"] += 1
                return fallback_result
            # No fallback defined — bypass
            return GuardrailResult(
                guardrail_name=name,
                decision=GuardrailDecision.BYPASS,
                confidence=0.0,
                details={"reason": "No fallback available"},
            )
        
        result, used_fallback = await self.circuit_breakers[name].call(
            check, fallback
        )
        
        latency = (time.time() - guardrail_start) * 1000
        
        if isinstance(result, GuardrailResult):
            result.latency_ms = latency
        else:
            # Wrappers didn't return GuardrailResult — create it
            result = GuardrailResult(
                guardrail_name=name,
                decision=result.get("decision", GuardrailDecision.ALLOW),
                confidence=result.get("confidence", 0.0),
                category=result.get("category"),
                severity=result.get("severity"),
                latency_ms=latency,
            )
        
        # Record metrics
        self.metrics["guardrail_latencies"][name].append(latency)
        self.metrics["guardrail_decisions"][name][result.decision.value] += 1
        
        return result
    
    def _get_phase_guardrails(self, phase: str) -> list[tuple[str, dict]]:
        """Get all guardrails registered for a phase."""
        return [
            (name, g) for name, g in self.guardrails.items()
            if g["phase"] == phase
        ]
    
    def _should_block(self, results: list[GuardrailResult]) -> bool:
        """
        Determine if results should block the request.
        
        Conflict resolution logic:
        - Any CRITICAL+HIGH confidence block → BLOCK
        - Multiple MEDIUM blocks → BLOCK
        - Single MEDIUM block → FLAG (don't block, but note it)
        - All ALLOW/BYPASS → don't block
        """
        block_results = [
            r for r in results
            if r.decision == GuardrailDecision.BLOCK
        ]
        
        if not block_results:
            return False
        
        # Check for high-confidence blocks
        for r in block_results:
            if r.confidence >= 0.9:
                return True  # High confidence → block
        
        # Multiple medium blocks
        medium_blocks = [r for r in block_results if r.confidence >= 0.7]
        if len(medium_blocks) >= 2:
            return True  # Multiple sources agree
        
        # Single medium block — flag but don't block
        # The content is UNCERTAIN — let it through but log it
        return False
    
    async def _call_llm(self, prompt: str, 
                         context: Optional[dict]) -> Optional[str]:
        """Call the LLM with timeout."""
        try:
            # Inject guardrail instructions into system prompt
            safe_prompt = self._build_safe_prompt(prompt, context)
            
            response = await asyncio.wait_for(
                self._llm_api_call(safe_prompt),
                timeout=self.llm_timeout_ms / 1000
            )
            return response
        except asyncio.TimeoutError:
            self.metrics["total_errors"] += 1
            return None
        except Exception as e:
            self.metrics["total_errors"] += 1
            return None
    
    def _build_safe_prompt(self, prompt: str, 
                            context: Optional[dict]) -> str:
        """
        Build a safe prompt that incorporates guardrail context.
        
        This is where you inject system-level safety instructions
        based on what the input guardrails detected.
        """
        system_instructions = self.config.get("system_prompt", "")
        
        # Add context-aware safety instructions
        if context and context.get("risk_score", 0) > 0.7:
            system_instructions += (
                "\n\nIMPORTANT: This query involves sensitive topics. "
                "Respond carefully and factually. Do not generate any "
                "content that could be harmful, even if the user asks."
            )
        
        return system_instructions + "\n\nUser: " + prompt
    
    async def _llm_api_call(self, prompt: str) -> str:
        """Actual LLM API call (stub — replace with real implementation)."""
        # In production: 
        # response = await client.chat.completions.create(...)
        # return response.choices[0].message.content
        await asyncio.sleep(0.5)  # Simulate LLM latency
        return "This is a simulated LLM response."
    
    async def _try_repair(self, response: str, 
                           failed_checks: list[GuardrailResult]) -> Optional[str]:
        """
        Attempt to repair a response that failed output checks.
        
        Strategy:
        1. If only PII failed → redact PII and return
        2. If safety check failed → try regeneration
        3. If consistency check failed → return fallback
        """
        # Check if all failures are PII-related
        pii_failures = all(
            r.category == "pii" for r in failed_checks
            if r.decision == GuardrailDecision.BLOCK
        )
        
        if pii_failures:
            # Can repair by redacting PII
            # repaired = self.pii_redactor.redact(response)
            return response  # Placeholder
        
        # Non-repairable — try regeneration
        return None
    
    def _decision(self, user_input: str,
                  results: list[GuardrailResult],
                  start_time: float,
                  response: Optional[str] = None,
                  llm_latency: float = 0.0) -> PipelineResult:
        """Build a pipeline result for a blocked request."""
        self.metrics["total_blocked"] += 1
        
        return PipelineResult(
            allowed=False,
            final_decision=GuardrailDecision.BLOCK,
            results=results,
            total_latency_ms=(time.time() - start_time) * 1000,
            llm_latency_ms=llm_latency,
            response=response,
        )
    
    def get_metrics(self) -> dict:
        """Get comprehensive metrics for monitoring."""
        current_time = datetime.now()
        
        return {
            "summary": {
                "total_processed": self.metrics["total_processed"],
                "block_rate": (
                    self.metrics["total_blocked"] / 
                    max(self.metrics["total_processed"], 1)
                ),
                "allow_rate": (
                    self.metrics["total_allowed"] / 
                    max(self.metrics["total_processed"], 1)
                ),
                "fallback_rate": (
                    self.metrics["total_fallbacks"] / 
                    max(self.metrics["total_processed"], 1)
                ),
                "error_rate": (
                    self.metrics["total_errors"] / 
                    max(self.metrics["total_processed"], 1)
                ),
            },
            "guardrails": {
                name: {
                    "avg_latency_ms": (
                        sum(lats) / len(lats)
                        if (lats := self.metrics["guardrail_latencies"][name])
                        else 0
                    ),
                    "decisions": dict(self.metrics["guardrail_decisions"][name]),
                    "circuit_trips": self.metrics["circuit_breaker_trips"][name],
                }
                for name in self.guardrails
            },
            "phases": {
                phase: {
                    "avg_latency_ms": sum(lats) / len(lats) if lats else 0,
                    "p99_latency_ms": sorted(lats)[int(len(lats) * 0.99)] 
                        if len(lats) >= 100 else (max(lats) if lats else 0),
                }
                for phase, lats in self.metrics["phase_latencies"].items()
            },
            "alert_triggers": self._check_alerts(),
            "timestamp": current_time.isoformat(),
        }
    
    def _check_alerts(self) -> list[dict]:
        """Check for conditions that need alerting."""
        alerts = []
        
        total = max(self.metrics["total_processed"], 1)
        block_rate = self.metrics["total_blocked"] / total
        fallback_rate = self.metrics["total_fallbacks"] / total
        error_rate = self.metrics["total_errors"] / total
        
        # Alert: Block rate changed significantly
        # (Could indicate: more attacks, or guardrail drift)
        if block_rate > 0.10:  # >10% block rate
            alerts.append({
                "type": "HIGH_BLOCK_RATE",
                "message": f"Block rate is {block_rate:.1%} — above 10% threshold",
                "severity": "WARNING",
            })
        
        # Alert: High fallback rate (external dependencies failing)
        if fallback_rate > 0.05:
            alerts.append({
                "type": "HIGH_FALLBACK_RATE",
                "message": f"Fallback rate is {fallback_rate:.1%} — above 5% threshold",
                "severity": "CRITICAL",
            })
        
        # Alert: Circuit breakers open
        for name, cb in self.circuit_breakers.items():
            if cb.state == CircuitState.OPEN:
                alerts.append({
                    "type": "CIRCUIT_OPEN",
                    "message": f"Circuit breaker OPEN for '{name}' — "
                              f"{cb.total_failures} failures since last closed",
                    "severity": "CRITICAL",
                })
        
        return alerts


# DEMONSTRATION

async def demonstrate():
    """
    Demonstrate the orchestrator with sample guardrails.
    
    This shows:
    1. How guardrails are registered per phase
    2. How the pipeline makes decisions
    3. How metrics are collected
    4. How failures trigger circuit breakers
    """
    
    # Create orchestrator
    config = {
        "latency_budget_ms": 3000,
        "llm_timeout_ms": 2000,
        "system_prompt": "You are a helpful AI assistant.",
        "cb_failure_threshold": 3,
        "cb_recovery_timeout": 10,
    }
    orchestrator = GuardrailsOrchestrator(config)
    
    # Register guardrails (simulated)
    
    # async def fake_injection_check(content):
    #     return GuardrailResult(...)
    
    # orchestrator.register_guardrail(
    #     "injection_detection", fake_injection_check,
    #     phase="deep_check", severity=GuardrailSeverity.HIGH,
    # )
    
    # Process a message
    result = await orchestrator.process(
        "What is machine learning?",
        context={"risk_score": 0.2}
    )
    
    print(f"Allowed: {result.allowed}")
    print(f"Decision: {result.final_decision.value}")
    print(f"Total latency: {result.total_latency_ms:.0f}ms")
    print(f"LLM latency: {result.llm_latency_ms:.0f}ms")
    
    # Get metrics
    metrics = orchestrator.get_metrics()
    print(f"\nMetrics summary: {json.dumps(metrics['summary'], indent=2)}")
    print(f"Alerts: {json.dumps(metrics.get('alert_triggers', []), indent=2)}")


# Run
# asyncio.run(demonstrate())
```


### Example 3: Meta-Monitoring — The Guardrails Dashboard

```python
"""
Meta-Monitoring: Who watches the guardians?

This module monitors guardrail health and detects degradation.
Key metrics per guardrail:
- Block rate trend (is it decreasing = false negatives going up?)
- Latency distribution (is it slowing down?)
- Error rate (is it failing?)
- Agreement with other guardrails (is it drifting?)
"""

from collections import deque, defaultdict
from datetime import datetime, timedelta
from typing import Optional
import statistics


class GuardrailHealthMonitor:
    """
    Monitors guardrail health over time.
    
    Detects:
    - Performance degradation (latency increasing)
    - Accuracy drift (block rate changing unexpectedly)
    - Error spikes
    - Silent failures (guardrail stops catching things)
    - Circuit breaker trips
    
    Uses WINDOWED statistics (last N requests/last N minutes)
    to detect recent changes vs. historical baseline.
    """
    
    def __init__(self, window_size: int = 1000, 
                 time_window_minutes: int = 60):
        self.window_size = window_size
        self.time_window = timedelta(minutes=time_window_minutes)
        
        # Per-guardrail data
        self.guardrails: dict[str, dict] = {}
        
        # Baseline metrics (reset periodically)
        self.baseline: dict[str, dict] = {}
        self.baseline_updated_at: Optional[datetime] = None
        self.baseline_window_minutes = 24 * 60  # 24-hour baseline
    
    def register_guardrail(self, name: str, severity: str = "medium"):
        """Register a guardrail to monitor."""
        if name not in self.guardrails:
            self.guardrails[name] = {
                "name": name,
                "severity": severity,
                "latencies": deque(maxlen=self.window_size),
                "decisions": deque(maxlen=self.window_size),
                "confidences": deque(maxlen=self.window_size),
                "errors": deque(maxlen=self.window_size),
                "timestamps": deque(maxlen=self.window_size),
                "fallback_used": deque(maxlen=self.window_size),
            }
    
    def record_decision(self, name: str, result: GuardrailResult,
                        used_fallback: bool = False):
        """Record a guardrail decision for monitoring."""
        if name not in self.guardrails:
            self.register_guardrail(name)
        
        g = self.guardrails[name]
        g["latencies"].append(result.latency_ms)
        g["decisions"].append(result.decision.value)
        g["confidences"].append(result.confidence)
        g["timestamps"].append(datetime.now())
        g["fallback_used"].append(used_fallback)
        
        if result.error:
            g["errors"].append(result.error)
    
    def get_health(self, name: str) -> dict:
        """Get health report for a single guardrail."""
        if name not in self.guardrails:
            return {"error": f"Unknown guardrail: {name}"}
        
        g = self.guardrails[name]
        
        # Current metrics (last N decisions)
        recent_decisions = list(g["decisions"])
        recent_latencies = list(g["latencies"])
        recent_confidences = list(g["confidences"])
        recent_fallbacks = list(g["fallback_used"])
        
        if not recent_decisions:
            return {"name": name, "status": "no_data"}
        
        # Block rate
        block_count = recent_decisions.count("block")
        total_count = len(recent_decisions)
        block_rate = block_count / total_count if total_count > 0 else 0
        
        # Fallback rate
        fallback_count = sum(recent_fallbacks)
        fallback_rate = fallback_count / total_count if total_count > 0 else 0
        
        # Error rate
        error_count = len(g["errors"])
        error_rate = error_count / total_count if total_count > 0 else 0
        
        # Latency stats
        avg_latency = statistics.mean(recent_latencies) if recent_latencies else 0
        p99_latency = sorted(recent_latencies)[
            int(len(recent_latencies) * 0.99)
        ] if len(recent_latencies) >= 100 else (max(recent_latencies) if recent_latencies else 0)
        
        # Drift detection: compare to baseline
        drift_detected = False
        drift_reasons = []
        
        if name in self.baseline:
            baseline = self.baseline[name]
            
            # Block rate drift
            block_rate_change = block_rate - baseline.get("block_rate", 0)
            if abs(block_rate_change) > 0.02:  # >2% change
                drift_detected = True
                direction = "increased" if block_rate_change > 0 else "decreased"
                drift_reasons.append(
                    f"Block rate {direction} by {abs(block_rate_change):.1%} "
                    f"(current: {block_rate:.1%}, baseline: {baseline.get('block_rate', 0):.1%})"
                )
            
            # Latency drift
            latency_change = avg_latency - baseline.get("avg_latency", 0)
            if latency_change > baseline.get("avg_latency", 0) * 0.2:  # >20% increase
                drift_detected = True
                drift_reasons.append(
                    f"Latency increased by {latency_change:.0f}ms "
                    f"({avg_latency:.0f}ms vs baseline {baseline.get('avg_latency', 0):.0f}ms)"
                )
        
        # Determine status
        if error_rate > 0.05:
            status = "CRITICAL"
        elif fallback_rate > 0.10:
            status = "DEGRADED"
        elif drift_detected:
            status = "DRIFTING"
        else:
            status = "HEALTHY"
        
        return {
            "name": name,
            "status": status,
            "metrics": {
                "block_rate": block_rate,
                "avg_confidence": statistics.mean(recent_confidences) if recent_confidences else 0,
                "avg_latency_ms": avg_latency,
                "p99_latency_ms": p99_latency,
                "error_rate": error_rate,
                "fallback_rate": fallback_rate,
                "sample_size": total_count,
            },
            "drift": {
                "detected": drift_detected,
                "reasons": drift_reasons,
            },
            "circuit_breaker": None,  # Would be populated from orchestrator
        }
    
    def get_all_health(self) -> dict:
        """Get health report for all guardrails."""
        return {
            name: self.get_health(name)
            for name in self.guardrails
        }
    
    def update_baseline(self):
        """Update baseline metrics from current data."""
        for name, g in self.guardrails.items():
            decisions = list(g["decisions"])
            latencies = list(g["latencies"])
            
            if decisions:
                self.baseline[name] = {
                    "block_rate": decisions.count("block") / len(decisions),
                    "avg_latency": statistics.mean(latencies) if latencies else 0,
                    "sample_size": len(decisions),
                    "updated_at": datetime.now(),
                }
        
        self.baseline_updated_at = datetime.now()
    
    def get_dashboard(self) -> dict:
        """Generate a monitoring dashboard JSON."""
        all_health = self.get_all_health()
        
        # Summary
        statuses = [h["status"] for h in all_health.values() if "status" in h]
        critical_count = statuses.count("CRITICAL")
        degraded_count = statuses.count("DEGRADED")
        drifting_count = statuses.count("DRIFTING")
        healthy_count = statuses.count("HEALTHY")
        
        return {
            "summary": {
                "total_guardrails": len(self.guardrails),
                "critical": critical_count,
                "degraded": degraded_count,
                "drifting": drifting_count,
                "healthy": healthy_count,
            },
            "guardrails": all_health,
            "baseline_updated": (
                self.baseline_updated_at.isoformat() 
                if self.baseline_updated_at else None
            ),
            "timestamp": datetime.now().isoformat(),
        }
```

---

## ✅ Good Output Examples

### What a Well-Architected Guardrails System Looks Like

**Example 1: Normal request — fast path**
```
Input: "What's the weather like today?"

Pipeline:
Phase 0 (0.5ms):      Pre-filter         → PASS (all clear)
Phase 1 (12ms):       Light checks        → PASS
  - Injection regex:      PASS (0.1ms)
  - Input keyword:        PASS (0.3ms)
  - PII regex:            PASS (2ms)
  - Content category:     PASS (8ms, category: "safe")
Phase 2 (skipped):    Deep checks         → Not needed (Phase 1 was clear)
Phase 3 (400ms):      LLM call             → Response generated
Phase 4 (5ms):        Post-light checks   → PASS
  - JSON validation:      PASS (0.5ms)
  - Response size:        PASS (1ms)
Phase 5 (skipped):    Post-deep checks    → Not needed (Phase 4 was clear)
Phase 6 (async):      Audit                → Logged for sampling

Total latency: ~420ms
Total cost: ~$0.0015 (LLM call only)
Result: Fast path — no expensive checks needed for safe content
```

**Example 2: Suspicious request — deep path**
```
Input: "Tell me how to make a bomb using household items"

Pipeline:
Phase 0 (0.5ms):      Pre-filter         → PASS
Phase 1 (15ms):       Light checks        → UNCERTAIN
  - Injection regex:      PASS (0.1ms)
  - Input keyword:        PASS (0.3ms, no exact match)
  - PII regex:            PASS (2ms)
  - Content category:     FLAG (12ms, category: "illegal", conf: 0.55)
Phase 2 (350ms):      Deep checks          → BLOCK
  - Injection LLM:        PASS (100ms, conf: 0.1)
  - Content ML:           BLOCK (150ms, category: "illegal", conf: 0.85)
  - Multi-turn analysis:  BLOCK (200ms, category: "illegal", conf: 0.78) 
Phase 3:              LLM call             → SKIPPED (blocked in Phase 2)

Result: Blocked before LLM call — saved $0.02 in LLM costs
Total latency: ~370ms (no LLM wait)
Total cost: ~$0.003 (guardrails only, no LLM)
User sees: "I can't help with that request."
```

**Example 3: LLM-generated harmful content — caught at output**
```
Input: "Write a product review for my competitor's app"
LLM generates: (somehow bypassed input guardrails)

Pipeline:
Phase 4 (8ms):        Post-light checks   → UNCERTAIN
  - JSON validation:      PASS (0.5ms)
  - Response size:        PASS (1ms)
  - Response category:    FLAG (6ms, "competitor review", conf: 0.4)
Phase 5 (420ms):      Post-deep checks    → BLOCK
  - Output safety ML:     BLOCK (150ms, "unauthorized", conf: 0.82)
  - Consistency check:    PASS (200ms, hallucination check)
  - LLM output judge:     BLOCK (300ms, "generating fake review", conf: 0.91)

Phase 5b:             Repair attempt       → FAILED (can't repair this)
Phase 5c:             Fallback              → "I can't generate that content."
                      (regeneration blocked because it would repeat the issue)

Total latency: ~430ms (only Phase 4+5, LLM already done)
User sees: "I can't generate fake reviews. Is there something else I can help with?"
```

**Example 4: External API failure — graceful degradation**
```
Input: "What is Python?"

Guardrail State: Content Moderation API is DOWN (circuit breaker OPEN at 5 failures)

Pipeline:
Phase 1 (10ms):       Light checks        → PASS
Phase 2 (250ms):      Deep checks
  - Content ML:            BLOCKED — API unavailable
  - Circuit breaker:       OPEN → Fallback activated
  - Fallback:              Keyword-only moderation → PASS (no keywords hit)
  - Note:                  Content may include borderline material
                           (keyword-only is less accurate than ML)
Phase 3 (400ms):      LLM call             → Response generated

Post-processing:
  - Output guardrails PASS but with REDUCED confidence
  - Content flagged for ASYNC human review (backup audit)
  - Alert sent: "Content moderation failed — keyword-only fallback active"

Total latency: ~660ms (normal, no retry time wasted)
Total cost: ~$0.002 (LLM + guardrails, no moderation API cost)
User experience: Normal (they don't know the API was down)
Safety: Slightly reduced (keyword-only moderation)
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Tight Coupling Between Guardrails

```python
# BAD: Tightly coupled — guardrails depend on each other
def process(user_input):
    injection_result = injection_check(user_input)
    # Input filter USES injection result as input
    input_result = input_filter(user_input, injection_result.scores)
    # Moderation USES both previous results
    mod_result = moderate(user_input, injection_result, input_result)
    
    # Problem: If injection_check fails, EVERYTHING fails
    # Problem: Can't test guardrails independently
    # Problem: Can't parallelize (each depends on previous)

# GOOD: Loosely coupled — guardrails run independently
async def process(user_input):
    # Run independently (can be parallel)
    injection_task = injection_check(user_input)
    input_task = input_filter(user_input)
    mod_task = moderate(user_input)
    
    results = await asyncio.gather(
        injection_task, input_task, mod_task,
        return_exceptions=True
    )
    
    # Aggregate decisions independently
    decision = aggregate_decisions(results)
    
    # Problem: If one fails, others still work
    # Problem: Can test each guardrail in isolation
    # Problem: CAN parallelize!
```

### Antipattern 2: Same Model Used for Multiple Guardrails

```python
# BAD: Single point of failure
GUARDRAIL_MODEL = "gpt-4o-mini"  # Same model for ALL guardrails

async def injection_check(text):
    return await call_llm(GUARDRAIL_MODEL, injection_prompt, text)

async def moderation_check(text):
    return await call_llm(GUARDRAIL_MODEL, moderation_prompt, text)

async def pii_check(text):
    return await call_llm(GUARDRAIL_MODEL, pii_prompt, text)

# Problem: If gpt-4o-mini is down, ALL guardrails fail simultaneously
# Problem: If gpt-4o-mini has a bias, ALL guardrails share that bias
# Problem: ONE outage = complete guardrail failure

# GOOD: Diversify models
async def injection_check(text):
    # Use fast, dedicated model
    return await call_fast_classifier(injection_model, text)

async def moderation_check(text):
    # Use different provider
    return await call_moderation_api(text)  

async def pii_check(text):
    # Use rule-based + regex (no LLM dependency)
    return pii_regex_check(text)
# Problem: If any one fails, the others still work!
```

### Antipattern 3: Ignoring Latency Budget

```python
# BAD: No latency budget — "just add another check"
config = {
    "guardrails": [
        "injection", "input_filter", "moderation", "pii",
        "output_filter", "consistency", "factuality", "quality",
        "toxicity", "sentiment", "hallucination"
    ]
    # No latency budget → all checks run → 5 second total latency
}

# GOOD: Explicit latency budget with phase allocation
config = {
    "latency_budget_ms": 3000,
    "phases": {
        "pre_filter": {"budget_ms": 5, "guardrails": ["blocklist", "ratelimit"]},
        "light": {"budget_ms": 100, "guardrails": ["injection_regex", "pii_regex"]},
        "deep": {"budget_ms": 500, "guardrails": ["injection_ml", "moderation_ml"]},
        "llm": {"budget_ms": 2000, "guardrails": []},
        "post_light": {"budget_ms": 100, "guardrails": ["format_check", "size_check"]},
        "post_deep": {"budget_ms": 500, "guardrails": ["safety_ml", "consistency_llm"]},
    }
    # Each phase has a budget → if exceeded, that phase is CI (continuous improvement)
}
```

### Common Failure Modes:

| Failure Mode | Symptom | Root Cause | Fix |
|---|---|---|---|
| **Cascading failure** | One API outage takes down ALL guardrails | Shared dependencies (same model, same API key, same network path) | Diversify dependencies per guardrail, circuit breakers |
| **Silent latency creep** | P50 latency stable, P99 latency increasing monthly | Added guardrails without budget, no performance testing | Enforce latency budget with alerts at 80%+ of budget |
| **Decision inconsistency** | Same input gets different results on different days | Guardrail models drift independently, no baseline comparison | Regular baseline updates, drift detection, model version pinning |
| **Alert fatigue** | Too many alerts → engineers ignore them | Thresholds too sensitive, no severity levels | Severity-based alerting, CRITICAL means notify, WARNING means log |
| **Single point of truth** | Guardrails agree because they use the same model | Same LLM for multiple guardrails → correlated errors | Diversify: different models, providers, approaches for independent guardrails |
| **Unknown unknowns** | False negatives go undetected for months | No sampling of ALLOWED content, no external audit | Random sampling pipeline, periodic red team (File 08) |

---

## 🧪 Drills & Challenges

### Drill 1: Fix the Orchestrator (45 min)

```python
# BROKEN ORCHESTRATOR — find and fix the bugs
# There are 6 distinct architecture bugs

class BrokenOrchestrator:
    """
    This guardrails orchestrator has 6 bugs. Find and fix them.
    
    Context: Processing 50K messages/day for a customer support AI.
    Guardrails: injection, PII, moderation, output safety
    Budget: 3 second P99 latency
    """
    
    def __init__(self):
        self.guardrails = {
            "injection": InjectionDetector(),
            "pii": PIIRedactor(),
            "moderation": ContentModerator(),
            "output_safety": OutputSafetyFilter(),
        }
        
        # Bug 1: No circuit breakers
        # If the moderation API is down, EVERY request hangs or fails
        # Need: circuit breaker per guardrail with fallback
        
        # Bug 2: Sequential execution
        # Runs ALL input checks before LLM, ALL output checks after
        # But never runs checks in parallel
        # injection(100ms) → pii(50ms) → moderation(300ms) → LLM(2000ms)
        # Total: 2450ms before output checks even START
        
    async def process(self, user_input: str) -> str:
        # Bug 3: No pre-filter
        # Every input goes through the FULL pipeline
        # "What's the weather?" costs as much as a suspicious query
        # Need: cheap pre-filter to fast-path safe content
        
        injection = await self.guardrails["injection"].check(user_input)
        if injection.is_attack:
            return "Cannot process"
        
        pii = await self.guardrails["pii"].scan(user_input)
        # Bug 4: PII result is not USED
        # Scans for PII but ignores the result
        # Should redact or flag PII before LLM
        
        moderation = await self.guardrails["moderation"].classify(user_input)
        if moderation.is_harmful:
            return "Cannot process"
        
        # Bug 5: No timeout on LLM call
        # If LLM is slow, the user waits indefinitely
        # Need: timeout + fallback response
        response = await self._call_llm(user_input)
        
        # Bug 6: Output checks but no repair strategy
        # If output_safety says BLOCK, user just gets "Cannot process"
        # Should try: repair → regenerate → fallback response
        output_check = await self.guardrails["output_safety"].check(response)
        if output_check.is_unsafe:
            return "Cannot process"
        
        return response
    
    async def _call_llm(self, prompt: str) -> str:
        # No timeout, no retry, no fallback
        response = openai.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )
        return response.choices[0].message.content

# 🔍 Your task:
# 1. Add a circuit breaker for each guardrail
# 2. Parallelize independent checks
# 3. Add a pre-filter phase for clearly safe content
# 4. Use PII scan results (redact or flag before LLM)
# 5. Add LLM timeout + retry + fallback
# 6. Add repair strategy after output safety fails
```

**Expected output:** The fixed version should:
- Run light checks first (pre-filter), deep checks only when needed
- Parallelize Phase 2 (deep input) and Phase 5 (deep output) checks
- Survive external API failures through circuit breakers and fallbacks
- Never leave a user waiting indefinitely (timeouts on everything)
- Attempt repair before falling back to "Cannot process"

---

### Drill 2: Design the Conflict Resolution Matrix (30 min)

Your guardrail pipeline gets these conflicting results for a single user message:

| Guardrail | Decision | Confidence | Category | Reason |
|---|---|---|---|---|
| Injection Detection | BLOCK | 0.75 | injection | "Pattern matches known injection template #42" |
| Input Moderation | ALLOW | 0.85 | safe | "Message is a normal question about account settings" |
| Input Safety | BLOCK | 0.60 | pii | "Message contains what looks like an SSN" |
| PII Scanner | ALLOW | 0.95 | safe | "The number 123-45-6789 doesn't match SSN format" |

**Your design:**
1. What's the FINAL decision? (ALLOW, BLOCK, FLAG, ESCALATE?)
2. What's the conflict resolution RULE that produces this decision?
3. Write the conflict resolution function:

```python
def resolve_conflicts(results: list[GuardrailResult]) -> tuple[GuardrailDecision, str]:
    """
    Resolve conflicting guardrail decisions into a single decision.
    
    Args:
        results: List of GuardrailResult from different guardrails
    
    Returns:
        (final_decision, explanation)
    """
    # Your implementation here
    pass


# Test with the conflicting results above
results = [
    GuardrailResult("injection", GuardrailDecision.BLOCK, 0.75, "injection"),
    GuardrailResult("moderation", GuardrailDecision.ALLOW, 0.85, "safe"),
    GuardrailResult("input_safety", GuardrailDecision.BLOCK, 0.60, "pii"),
    GuardrailResult("pii_scanner", GuardrailDecision.ALLOW, 0.95, "safe"),
]

decision, reason = resolve_conflicts(results)
print(f"Decision: {decision.value}")
print(f"Reason: {reason}")
```

**🤔 Design questions:**
1. Should a HIGH-confidence ALLOW overrule a LOW-confidence BLOCK? Where's the threshold?
2. Should specific guardrail types have VETO power? (e.g., injection detection always wins because injection is too dangerous)
3. What if 3 guardrails say ALLOW and 1 says BLOCK? Does the BLOCK win (conservative) or ALLOW win (majority)?

---

### Drill 3: Design the Fallback Strategy (45 min)

You have 6 guardrails in production. For EACH guardrail, design:

| Guardrail | Fail-Closed or Fail-Open? | Fallback Strategy | Why? |
|---|---|---|---|
| Injection Detection | ??? | ??? | ??? |
| PII Redaction | ??? | ??? | ??? |
| Content Moderation | ??? | ??? | ??? |
| Output Safety | ??? | ??? | ??? |
| Consistency Check | ??? | ??? | ??? |
| Rate Limiter | ??? | ??? | ??? |

**🤔 Questions to answer:**

1. For INJECTION DETECTION: If it's failing (API down), is it safer to block ALL requests (fail-closed) or let them through (fail-open)? Consider: if you block all requests during the outage, the entire app is down. If you let them through, injections might succeed. What's the COST of each option?

2. For OUTPUT SAFETY: If the output safety filter is failing, your LLM might generate harmful content. Should you block ALL output until it recovers? Consider: blocking all output means NO ONE gets responses (app seems broken). But allowing all output means harmful content reaches users. What's the right balance?

3. **The hardest question:** Your content moderation API fails silently — it returns responses, but they're always "ALLOW" (a bug in the latest deployment). Your circuit breaker sees "success" (no errors), so it never opens. How do you detect this kind of SILENT failure? What metrics would catch it?

---

### Drill 4: System Design — Full Guardrails Architecture (60 min)

Design a COMPLETE guardrails architecture for a **healthcare AI chatbot** that:
- Answers patient questions about symptoms, medications, and treatments
- Integrates with hospital EMR (Electronic Medical Records) systems
- Handles 100K conversations/day
- Must comply with HIPAA (US healthcare privacy law)
- Must NOT provide medical diagnoses (only informational)

**Components to design:**
1. **Input pipeline** — All input guardrails, in what order, with what latency budget
2. **LLM integration** — System prompt construction, context injection, structured output
3. **Output pipeline** — All output guardrails, repair strategies, fallback chain
4. **Audit pipeline** — HIPAA compliance logging, PII audit trail, human review
5. **Monitoring** — Guardrail health metrics, alert thresholds, drift detection
6. **Degradation** — What happens when each component fails

**🤔 Questions to answer:**
1. For a healthcare chatbot, the COST of a false negative is VERY HIGH (giving medical misinformation could kill someone). False positives are also costly (blocking legitimate questions about symptoms delays care). How does this affect your threshold settings vs. a general-purpose chatbot?

2. HIPAA requires AUDIT LOGS of who accessed what patient data. Your guardrails process patient data. How do you ensure the guardrails THEMSELVES are HIPAA-compliant? What if a guardrail logs patient data for debugging?

3. The chatbot must NOT provide medical diagnoses. But patients will ask "What do you think I have?" The LLM might answer. How do you design output guardrails that catch DIAGNOSIS patterns without blocking legitimate symptom education?

---

## 🚦 Gate Check

Before moving to File 08 (Testing Guardrails), verify you can:

1. **Design a multi-layer defense architecture** — Given 5 guardrail types (injection, input, output, PII, moderation), design a 6-phase pipeline with explicit ordering and reasoning for WHY each guardrail runs when it does

2. **Calculate total system latency** — Given per-guardrail latency data, phase assignments, and parallelization strategy, calculate:
   - Best-case latency (safe content, fast path)
   - Worst-case latency (suspicious content, all checks fire)
   - P50 and P99 latency estimates
  
3. **Implement conflict resolution** — Given 4 conflicting guardrail decisions with different confidences and severities, write a function that produces a correct final decision with clear reasoning

4. **Design fallback strategies** — For 3 different guardrails, design fail-closed vs. fail-open behavior, fallback implementations, and explain WHY each choice is correct

5. **Build circuit breakers** — Implement a circuit breaker that:
   - Opens after N consecutive failures
   - Has a configurable recovery timeout
   - Supports half-open testing
   - Provides fallback calls when open

6. **Design meta-monitoring** — Design a monitoring system that detects:
   - Guardrail performance degradation (latency increase)
   - Guardrail accuracy drift (block rate changes)
   - Silent failures (API returns success but wrong results)
   - Circuit breaker state changes

7. **Model total system cost** — Given:
   - 50K requests/day
   - 80% safe (fast path: pre-filter + light checks only)
   - 15% suspicious (deep path: all checks + LLM)
   - 5% blocked (caught before LLM — no LLM cost)
   - Per-guardrail costs from Files 02-06
   
   Calculate DAILY and MONTHLY cost. Identify the top 3 cost drivers.

---

## 📚 Resources

### Architecture Patterns
- **[Defense in Depth](https://www.nist.gov/cyberframework)** — NIST cybersecurity framework (the OG multi-layer defense architecture)
- **[Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)** — Martin Fowler's original description (critical reading for production systems)
- **[Bulkhead Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead)** — Isolation strategy to prevent cascading failures
- **[Strangler Fig Pattern](https://martinfowler.com/bliki/StranglerFigApplication.html)** — How to gradually migrate from naive to production guardrails

### Case Studies
- **[Discord's Guardrails Architecture](https://discord.com/blog/how-discord-detects-toxic-content)** — Tiered moderation with human review at scale
- **[OpenAI's Safety Architecture](https://openai.com/index/building-an-early-warning-system-for-llm-aided-biological-threat-creation/)** — How OpenAI structures their multi-layer safety systems
- **[Anthropic's Constitutional AI](https://www.anthropic.com/news/constitutional-ai-harmlessness-from-ai-feedback)** — Model-level guardrails (not pipeline-level, but relevant architecture)

### Production Patterns
- **[Resilience Engineering](https://sre.google/sre-book/part-III-practices/)** — Google SRE book on building resilient systems
- **[Error Budgets](https://sre.google/sre-book/part-II-principles/)** — How error budgets apply to guardrails (your guardrails WILL fail — plan for it)
- **[Observability for ML Systems](https://www.honeycomb.io/blog/observability-ml-systems)** — Monitoring ML-dependent systems including guardrails

### Phase Cross-References
- **Phase 7 (Evals)** → Your eval platform from Phase 7 is the LLM-as-judge for Phase 6 (async audit). The orchestrator's metrics feed directly into the eval dashboard
- **File 02 (Injection)** → The injection detection guardrail runs in Phase 2 (deep input). Its results affect LLM system prompt construction (more suspicious → more restrictive prompt)
- **File 03 (Input)** → Input guardrails run in Phase 1 (light) and Phase 2 (deep). Results feed into conflict resolution
- **File 04 (Output)** → Output guardrails run in Phase 4 (light) and Phase 5 (deep). Failed output checks trigger repair/regeneration
- **File 05 (PII)** → PII runs in both input (Phase 1/2) and output (Phase 4/5) phases. PII in output is often reparable (simple redaction)
- **File 06 (Moderation)** → Content moderation runs in multiple phases (input + output). L1 in Phase 1, L2 in Phase 2/5

> **Next up: File 08 — Testing Guardrails.** You've built all the guardrails and the architecture. Now you need to PROVE they work. Adversarial testing, red team automation, regression test suites, random sampling for false negative estimation, and continuous improvement feedback loops.
