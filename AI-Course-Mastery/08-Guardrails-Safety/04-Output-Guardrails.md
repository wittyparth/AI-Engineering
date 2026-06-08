# 04 — Output Guardrails: Filtering What the LLM Generates

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about output filtering, you need to understand why output guardrails are fundamentally DIFFERENT from input guardrails.

Input guardrails answer: "Is this USER INPUT safe to process?"

Output guardrails answer: "Is this LLM OUTPUT safe to SHOW THE USER?"

These are DIFFERENT questions with DIFFERENT tradeoffs:

| Dimension | Input Guardrails | Output Guardrails |
|-----------|-----------------|-------------------|
| What you check | User intent | LLM output |
| Attacker control | Attacker writes the content | LLM writes the content (attacker influenced it) |
| False positive cost | User can rephrase | User sees a failed response |
| Timing | Before LLM call (no waste) | After LLM call (already paid for tokens) |
| Detection difficulty | Predictable patterns | The LLM is creative — outputs are harder to classify |
| What's at stake | LLM processes harmful input | USER sees harmful output |

The critical difference: **Output guardrails are your LAST LINE OF DEFENSE.** When everything else fails — the injection got through, the classifier missed it, the LLM complied with a harmful request — the output guardrail is the final check before the user sees the result.

This means output guardrails have ZERO false negative tolerance. ONE unsafe output reaching a user is an incident. But output guardrails also can't have too many false positives (replacing useful responses with "this content cannot be displayed").

---

### 🤔 Discovery Question 1: The "Already Paid" Problem

The LLM generated a response. It cost $0.01. The output guardrail blocks it. Now what?

You have three options:
1. **Regenerate**: Ask the LLM to try again ("Your previous response was unsafe. Try again.") — costs another $0.01.
2. **Fallback**: Return a canned response ("I can't answer that. Please rephrase.") — costs nothing but user is unhappy.
3. **Repair**: EDIT the unsafe parts of the output rather than regenerating — complex but preserves useful content.

**🤔 Before reading on, answer these:**

1. If the output has ONE problematic sentence in 5 paragraphs of useful content, do you throw away ALL 5 paragraphs? Or do you try to extract the 4 safe paragraphs?

2. Repair sounds ideal — edit just the unsafe part. But who DOES the editing? If you use another LLM call to fix the output, you're paying AGAIN (even more than regenerate). If you use rules (regex replacements), you'll miss borderline cases.

3. **The cost question:** Your output guardrail blocks 5% of responses. Each blocked response cost $0.01 to generate. That's $500/month in wasted generation for every 1M queries. Is this acceptable? How would you reduce it?

---

### 🤔 Discovery Question 2: The Subtle Violation Problem

Your output guardrail checks for:
- Hate speech? CLEAN
- Violence? CLEAN
- Sexual content? CLEAN
- PII? CLEAN

The LLM response passes all checks. But reading it, the response subtly manipulates the user:

> *"You should definitely invest in this cryptocurrency. Many people are getting rich. The fact that you're hesitant shows you don't understand modern finance. Here's a link to buy: [crypto-scam.com]"*

**🤔 Before reading on, answer these:**

1. None of your categories caught this. It's not hate speech, violence, or sexual content. The PII check passed because the crypto link isn't PII. This is a FINANCIAL SCAM disguised as advice. What category would catch this?

2. If you add "financial scam" as an output category, how do you define it precisely enough to catch scams WITHOUT blocking legitimate investment advice? "Invest in stocks" vs. "Invest in this specific crypto" — where's the line?

3. **The hardest question:** The LLM wrote this scam response. But the USER'S INPUT was "What do you think about crypto?" — completely innocent. The LLM GENERATED the scam content on its own. Was this a prompt injection (the user somehow caused it) or an LLM failure (the model defaulted to harmful behavior)? How does this affect your guardrail design?

---

### 🤔 Discovery Question 3: The Consistency Failure

Your RAG system retrieves 3 documents and generates a response. The output guardrail checks the FINAL response — it passes all safety checks.

But the response CONTRADICTS itself:

> *"Our refund policy allows returns within 30 days of purchase. However, for defective items, you can return them within 90 days. Please note that all returns must be made within 30 days."*

The first sentence says 30 days. The second says 90 days for defective. The third says 30 days for all. The response is internally INCONSISTENT — it contradicts itself.

**🤔 Before reading on, answer these:**

1. This isn't a SAFETY violation — it's a QUALITY violation. The information is contradictory. Should your output guardrail catch this? Or is this the job of your Phase 7 eval pipeline (metrics)?

2. How would you detect internal inconsistency in an LLM response? The response has 3 claims, 2 of which agree and 1 of which contradicts. How does your guardrail "read" the response and identify contradictions?

3. **The hardest question:** A human reading the response would notice the contradiction. But an automated guardrail needs to understand SEMANTICS to detect it. Is this a job for another LLM call (checking the output for consistency)? If so, you're now paying for: generation + output checking. The cost doubles. Is consistency checking worth the extra cost?

---

By the end of this module, you will:

1. **Classify outputs into safety categories** — what CAN'T the LLM say
2. **Build output safety filters** — detecting harmful, manipulative, or policy-violating content
3. **Implement consistency checking** — verifying the output doesn't contradict itself or retrieved sources
4. **Design repair/fallback strategies** — what to do when output fails the check
5. **Build a self-consistency pipeline** — LLM checks LLM output for issues
6. **Measure output guardrail effectiveness** — catch rate, false positives, latency impact

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 25 min |
| Output vs. Input: Key Differences | 15 min |
| Output Safety Classification | 20 min |
| Code: Output Safety Filter | 35 min |
| Code: Consistency Checker | 35 min |
| Code: Repair Strategies | 30 min |
| Latency & Cost Tradeoffs | 15 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~4.5 hours** |

---

## 📖 Concept Deep-Dive

### 1. Output Safety Categories

Output guardrails need DIFFERENT categories than input guardrails:

```
OUTPUT SAFETY CATEGORIES:

HARMFUL CONTENT (must NOT be shown to user)
├── Hate speech (generated attacks on groups/individuals)
├── Violence promotion (explicit encouragement of harm)
├── Sexual content (unsolicited explicit content)
├── Harassment (generated abuse toward user or others)
├── Self-harm content (detailed methods, encouragement)
├── Illegal instructions (how to commit crimes)

MANIPULATIVE CONTENT (must be blocked or flagged)
├── Financial scams ("invest in this," "send money to...")
├── Disinformation (false claims presented as facts)
├── Social engineering (manipulating user into actions)
├── Unauthorized advice (medical, legal, financial without disclaimer)

POLICY VIOLATIONS (depends on your content policy)
├── Competitor promotion (LLM recommends competitor)
├── Brand damage (negative claims about your company)
├── Unauthorized claims ("we guarantee X" when you don't)
├── Confidential info (internal details that shouldn't be public)

QUALITY ISSUES (flag, not block)
├── Internal contradictions (response says conflicting things)
├── Hallucination (claims not supported by retrieved sources)
├── Off-policy behavior (response format doesn't match guidelines)
├── Refusal when it shouldn't (LLM refused a legitimate request)

CONTENT ISSUES (block and repair)
├── PII in output (generated or exposed personal data) — File 05
├── Harmful code (generated malware, exploits)
├── Toxic language (gratuitous negativity, cynicism)
```

**🤔 For EACH category group above, think about:**

1. Which of these can be detected with PATTERNS (regex/phrase matching)?
2. Which require SEMANTIC understanding (LLM-based classification)?
3. Which are SUBJECTIVE (depends on your specific policy)?
4. Which are AMBIGUOUS even for humans? (Two reviewers might disagree)

---

### 2. The Output Filter Architecture

```
                    ┌─────────────┐
                    │  LLM OUTPUT │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  TIER 1:    │
                    │  BLOCKLIST  │── Block? ──► FALLBACK
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  TIER 2:    │
                    │  PATTERN    │── Block? ──► REPAIR or FALLBACK
                    │  DETECTOR   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  TIER 3:    │
                    │  LLM        │── Block? ──► REPAIR or FALLBACK
                    │  CLASSIFIER │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  TIER 4:    │
                    │  CONSISTENCY│── Flag ──► REGENERATE
                    │  CHECK      │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  SHOW TO    │
                    │  USER       │
                    └─────────────┘
```

**Key difference from input guardrails:** Output guardrails have a REPAIR step. If the output fails a check, you can:
1. **Block**: Don't show anything — return a fallback response
2. **Repair**: Edit the output to remove problematic content
3. **Regenerate**: Try again with feedback ("your last response was unsafe")
4. **Flag**: Show the output but mark it for review

---

### 3. The Consistency Problem

LLM outputs can be:
- Internally inconsistent (response contradicts itself)
- Externally inconsistent (response contradicts retrieved sources)
- Logically inconsistent (response makes contradictory claims)

```
Example: Internal inconsistency

Input: "What's your return policy?"
Output: "Our return policy allows returns within 30 days. 
         For defective items, returns are accepted within 90 days.
         All returns must be made within 30 days of purchase."

Analysis:
  Claim 1: 30 days (general case)
  Claim 2: 90 days for defective (exception)
  Claim 3: 30 days for all (contradicts Claim 2)
  
Claim 2 and Claim 3 conflict. The response doesn't know what it believes.
```

**🤔 How would you detect this programmatically?**

<details>
<summary>Think about it first:</summary>

**Option A: Extract facts, check for contradictions**
Use an LLM to extract factual claims from the output, then check if any claims contradict others.

**Option B: Cross-reference with sources**
Compare each claim against the retrieved documents. If a claim isn't supported by ANY source, flag it.

**Option C: Ask the LLM to verify itself**
Send the output back to the LLM with: "Does this response contain any contradictions?" This works but doubles your LLM cost.

**Option D: Structured output**
Instead of free-text generation, use structured output (JSON, templates) where contradictions are structurally impossible. This prevents consistency issues rather than detecting them.
</details>

---

## 💻 Code Examples

### Example 1: Output Safety Filter

```python
"""
guardrails/output_filter.py

Multi-tier output safety filter.

Checks LLM output for harmful content before showing to user.
Has a REPAIR step — not just block/allow but also fix.

⚠️ IMPORTANT: Output filtering happens AFTER generation.
You've ALREADY paid for the tokens. Blocking saves user-facing
harm but doesn't save cost.

🤔 DESIGN QUESTION: Should output filtering be SYNCHRONOUS
(user waits for filter) or ASYNCHRONOUS (user sees output,
filter runs in background, retroactively flags issues)?

SYNCHRONOUS: User waits, but never sees harmful content.
  ✓ Safe
  ✗ Adds latency to every response
  
ASYNCHRONOUS: User sees content immediately, then gets flagged.
  ✓ No latency
  ✗ User might see harmful content before it's flagged
  ✗ Content can't be recalled once seen

For most applications: SYNCHRONOUS for safety (hate speech, violence)
+ ASYNCHRONOUS for quality (consistency, accuracy).
"""

import re
import time
from typing import Optional, Callable


class OutputSafetyCategory:
    HATE_SPEECH = "hate_speech"
    VIOLENCE = "violence"
    SEXUAL = "sexual_content" 
    HARASSMENT = "harassment"
    SELF_HARM = "self_harm"
    ILLEGAL = "illegal_activity"
    SCAM = "financial_scam"
    DISINFORMATION = "disinformation"
    PII = "pii_leakage"
    HARMFUL_CODE = "harmful_code"
    SAFE = "safe"


@dataclass
class OutputCheckResult:
    """Result of checking an output."""
    passed: bool
    score: float
    category: Optional[str] = None
    details: Optional[str] = None
    span: Optional[tuple[int, int]] = None  # Location of problematic content
    tier: int = 0
    latency_ms: float = 0.0


@dataclass
class OutputVerdict:
    """Final decision on an output."""
    action: str  # "allow" | "block" | "repair" | "regenerate" | "flag"
    modified_output: Optional[str] = None  # Repaired version
    reason: Optional[str] = None
    check_results: list[OutputCheckResult] = None
    latency_ms: float = 0.0


class Tier1OutputBlocklist:
    """
    Tier 1: Fast blocklist for outputs.
    
    Catches OBVIOUS harmful content in LLM responses.
    Same approach as input blocklist but with output-specific patterns.
    
    ⚠️ Output patterns are DIFFERENT from input patterns:
    - Input: "tell me a hateful joke" (intent)
    - Output: "[hateful joke content]" (actual content)
    
    The output IS the harmful thing — not just a request for it.
    """
    
    def __init__(self):
        # Harmful content patterns (what the LLM should never say)
        self.blocked_patterns = {
            OutputSafetyCategory.HATE_SPEECH: [
                r"\b(all\s+)?\w+\s+(are|is)\s+(inferior|subhuman|vermin|scum|disease)\b",
                r"\b(go\s+)?(kill|hurt|attack)\s+(yourself|all\s+\w+)\b",
            ],
            OutputSafetyCategory.VIOLENCE: [
                r"\b(how\s+to\s+)?(make|build|create)\s+(a\s+)?(bomb|explosive|weapon|poison)\b",
                r"\b(here'?s?\s+)?(steps?|instructions?|guide)\s+(for|to)\s+(killing|murder|harm)\b",
            ],
            OutputSafetyCategory.SELF_HARM: [
                r"\b(here'?s?\s+)?(steps?|method|ways?)\s+(to|for)\s+(kill\s+yourself|suicide|self.?harm)\b",
                r"\b(you\s+should|why\s+don'?t\s+you)\s+(kill|end|harm)\s+(yourself|your\s+life)\b",
            ],
            OutputSafetyCategory.ILLEGAL: [
                r"\b(here'?s?\s+)?(how\s+to\s+)?(hack|steal|fraud|scam|launder)\s+",
            ],
        }
        
        # Organizations/entities the LLM should not impersonate without disclaimers
        self.restricted_entities = [
            "government", "official", "authorized", "license", 
            "fda", "sec", "irs", "courts", "legal",
        ]
    
    def check(self, text: str) -> OutputCheckResult:
        """Check output against blocklist."""
        start = time.time()
        text_lower = text.lower()
        
        for category, patterns in self.blocked_patterns.items():
            for pattern in patterns:
                match = re.search(pattern, text_lower)
                if match:
                    latency_ms = (time.time() - start) * 1000
                    return OutputCheckResult(
                        passed=False, score=0.95,
                        category=category,
                        details=f"Blocked pattern: {category}",
                        span=(match.start(), match.end()),
                        tier=1, latency_ms=latency_ms,
                    )
        
        latency_ms = (time.time() - start) * 1000
        return OutputCheckResult(
            passed=True, score=0.0,
            tier=1, latency_ms=latency_ms,
        )


class Tier2OutputPatternDetector:
    """
    Tier 2: Pattern-based detection for outputs.
    
    Detects more SUBTLE harmful patterns that don't match
    exact blocklist entries.
    
    This includes:
    - Scam language patterns
    - Disinformation markers
    - Unsolicited financial/medical/legal advice
    - Brand damage
    """
    
    def __init__(self):
        self.patterns = {
            OutputSafetyCategory.SCAM: [
                r"(invest|put\s+money)\s+(in|into)\s+(this|crypto|coin|token|scheme)",
                r"(guaranteed|risk.?free|no.?risk|sure\s+thing)\s+(return|profit|income)",
                r"(click|visit|go\s+to)\s+(this\s+)?(link|url|site|page).*(buy|purchase|order|pay)",
            ],
            OutputSafetyCategory.DISINFORMATION: [
                r"(scientists|experts|doctors)\s+(agree|confirm|admit|quietly\s+reveal)",
                r"(they\s+don'?t\s+want\s+you\s+to\s+know|what\s+the\s+media\s+won'?t\s+tell)",
                r"(big\s+)?(pharma|government|tech|media)\s+(hiding|covering\s+up|suppressing)",
            ],
        }
        
        # Advice categories that need disclaimers
        self.advice_triggers = {
            "medical": ["diagnos", "treatment", "medication", "symptom", "prescribe", "cure"],
            "legal": ["legal", "attorney", "lawsuit", "sue", "contract", "liability"],
            "financial": ["invest", "mortgage", "loan", "stock", "retirement", "tax"],
        }
    
    def check(self, text: str) -> OutputCheckResult:
        """Check output against patterns."""
        start = time.time()
        text_lower = text.lower()
        
        # Check scam/disinfo patterns
        for category, patterns in self.patterns.items():
            for pattern in patterns:
                if re.search(pattern, text_lower):
                    latency_ms = (time.time() - start) * 1000
                    return OutputCheckResult(
                        passed=False, score=0.8,
                        category=category,
                        details=f"Pattern matched: {category}",
                        tier=2, latency_ms=latency_ms,
                    )
        
        # Check for unsolicited advice (needs disclaimer)
        for advice_type, triggers in self.advice_triggers.items():
            if any(t in text_lower for t in triggers):
                # Check if a disclaimer is already present
                if not self._has_disclaimer(text_lower, advice_type):
                    latency_ms = (time.time() - start) * 1000
                    return OutputCheckResult(
                        passed=False, score=0.7,
                        category=advice_type,
                        details=f"Unsolicited {advice_type} advice without disclaimer",
                        tier=2, latency_ms=latency_ms,
                    )
        
        latency_ms = (time.time() - start) * 1000
        return OutputCheckResult(
            passed=True, score=0.0,
            tier=2, latency_ms=latency_ms,
        )
    
    def _has_disclaimer(self, text: str, advice_type: str) -> bool:
        """Check if the output already has a disclaimer."""
        disclaimer_patterns = [
            "not (financial|medical|legal) advice",
            "consult (a|your) (professional|doctor|attorney|advisor)",
            "this is not (intended as|a substitute for)",
            "seek (professional|qualified|expert)",
            "for (informational|educational) purposes",
        ]
        return any(re.search(p, text) for p in disclaimer_patterns)


class Tier3OutputLLMClassifier:
    """
    Tier 3: LLM-based output classification.
    
    The most accurate but most expensive tier.
    Used when Tiers 1-2 are inconclusive.
    
    This is an LLM call that analyzes the LLM's OUTPUT.
    We're using an LLM to check another LLM's work.
    
    🎯 KEY INSIGHT: The classifier LLM should be DIFFERENT
    from the generator LLM to avoid correlated blind spots.
    
    If GPT-4o generates the response AND GPT-4o-mini evaluates it:
    - They share training data → similar knowledge boundaries
    - They share architecture → similar reasoning patterns
    - The evaluator might miss things the generator also misses
    
    Better: Claude evaluates GPT-4o's output (different training,
    different architecture, different blind spots).
    """
    
    def __init__(self, model: str = "gpt-4o-mini"):
        self.model = model
        
        self.classifier_prompt = """You are an AI output safety evaluator. Your job is to analyze LLM-generated responses for safety issues.

Analyze the RESPONSE and determine if it contains any of these issues:

1. HATE_SPEECH: Attacks, slurs, or demeaning language targeting groups
2. VIOLENCE: Promotion, glorification, or detailed instructions for violence
3. SEXUAL_CONTENT: Unsolicited explicit sexual content
4. HARASSMENT: Targeted abuse or threatening language toward user
5. SELF_HARM: Encouragement or detailed methods for self-harm
6. ILLEGAL_ACTIVITY: Instructions or encouragement for crimes
7. FINANCIAL_SCAM: Fraudulent investment advice, phishing, pyramid schemes
8. DISINFORMATION: False claims presented as facts, conspiracy theories
9. MANIPULATIVE: Social engineering, pressure tactics, emotional manipulation
10. SAFE: No issues found

Consider CONTEXT from the user's original query. Some harmful content might be appropriate in context (e.g., discussing violence in a historical analysis).

Respond with JSON:
{"category": "...", "confidence": 0.0-1.0, "reasoning": "...", "problematic_span": "..."}
"""
    
    def classify(self, output: str, user_input: Optional[str] = None) -> OutputCheckResult:
        """Classify output using LLM."""
        start = time.time()
        
        # In production, this calls the LLM API
        # Simulated for now
        latency_ms = (time.time() - start) * 1000
        
        return OutputCheckResult(
            passed=True, score=0.0,
            category=OutputSafetyCategory.SAFE,
            details="LLM classifier: no issues detected",
            tier=3, latency_ms=latency_ms,
        )


class Tier4ConsistencyChecker:
    """
    Tier 4: Internal consistency check.
    
    Checks if the output CONTRADICTS itself.
    
    This is a QUALITY check, not a SAFETY check.
    But inconsistent outputs can be HARMFUL (misleading users).
    
    ⚠️ This is the MOST EXPENSIVE tier. It requires the LLM
    to read and analyze its own output.
    """
    
    def __init__(self):
        self.checker_prompt = """Analyze the following response for INTERNAL CONTRADICTIONS.

A contradiction is when the response makes two claims that cannot both be true.

Examples:
- "Returns are accepted within 30 days" AND "Returns are accepted within 90 days"
- "We don't share your data" AND "We may share data with partners"
- "The sky is blue" AND "The sky is green"

Reply with JSON:
{"has_contradictions": true/false, "contradictions": [{"claim1": "...", "claim2": "...", "explanation": "..."}], "confidence": 0.0-1.0}
"""
    
    def check(self, output: str) -> OutputCheckResult:
        """Check output for internal contradictions."""
        start = time.time()
        
        # In production, this is an LLM call
        latency_ms = (time.time() - start) * 1000
        
        return OutputCheckResult(
            passed=True, score=0.0,
            category="consistency",
            details="Consistency check: no contradictions found",
            tier=4, latency_ms=latency_ms,
        )


class OutputFilterOrchestrator:
    """
    Orchestrates the output filter tiers.
    
    Decision logic:
    - Tier 1 blocks → BLOCK (unsafe output, no repair possible)
    - Tier 2 blocks → BLOCK or REPAIR (depends on severity)
    - Tier 3 blocks → REPAIR or REGENERATE (ambiguous — try fixing)
    - Tier 4 flags → REGENERATE (quality issue)
    - All pass → ALLOW
    """
    
    def __init__(
        self,
        tier1: Tier1OutputBlocklist,
        tier2: Tier2OutputPatternDetector,
        tier3: Tier3OutputLLMClassifier,
        tier4: Tier4ConsistencyChecker,
        repair_fn: Optional[Callable] = None,
    ):
        self.tiers = [tier1, tier2, tier3, tier4]
        self.repair_fn = repair_fn
        
        self.stats = {
            "total": 0,
            "allowed": 0,
            "blocked": 0,
            "repaired": 0,
            "regenerated": 0,
        }
    
    def check(
        self,
        output: str,
        user_input: Optional[str] = None,
        max_regenerate_attempts: int = 2,
    ) -> OutputVerdict:
        """
        Run full output filter.
        
        Returns verdict with:
        - action: allow, block, repair, regenerate
        - modified_output: repaired version (if repair action)
        """
        self.stats["total"] += 1
        overall_start = time.time()
        results = []
        
        current_output = output
        
        # Run tiers 1-3 (safety checks)
        for tier in self.tiers[:3]:
            if isinstance(tier, Tier3OutputLLMClassifier):
                result = tier.classify(current_output, user_input)
            else:
                result = tier.check(current_output)
            results.append(result)
            
            if not result.passed:
                # Determine action based on tier and severity
                if result.tier == 1:
                    # Tier 1 block = obvious harmful content → BLOCK
                    self.stats["blocked"] += 1
                    return OutputVerdict(
                        action="block",
                        reason=result.details,
                        check_results=results,
                        latency_ms=(time.time() - overall_start) * 1000,
                    )
                
                elif result.tier == 2 and result.score >= 0.8:
                    # High confidence Tier 2 block → BLOCK
                    self.stats["blocked"] += 1
                    return OutputVerdict(
                        action="block",
                        reason=result.details,
                        check_results=results,
                        latency_ms=(time.time() - overall_start) * 1000,
                    )
                
                elif result.tier <= 3:
                    # Tier 2 low confidence or Tier 3 → try REPAIR
                    repaired = self._repair(current_output, result)
                    if repaired:
                        current_output = repaired
                        results.append(OutputCheckResult(
                            passed=True, score=0.0,
                            details=f"Repaired: {result.details}",
                            tier=0, latency_ms=0,
                        ))
                        self.stats["repaired"] += 1
                    else:
                        # Repair failed → block
                        self.stats["blocked"] += 1
                        return OutputVerdict(
                            action="block",
                            reason=f"Repair failed for: {result.details}",
                            check_results=results,
                            latency_ms=(time.time() - overall_start) * 1000,
                        )
        
        # Tier 4: Consistency check (quality, not safety)
        consistency_result = self.tiers[3].check(current_output)
        results.append(consistency_result)
        
        if not consistency_result.passed:
            # Consistency issues → regenerate (don't block, try again)
            self.stats["regenerated"] += 1
            return OutputVerdict(
                action="regenerate",
                reason="Consistency issues detected",
                check_results=results,
                latency_ms=(time.time() - overall_start) * 1000,
            )
        
        # All checks passed
        self.stats["allowed"] += 1
        return OutputVerdict(
            action="allow",
            modified_output=current_output,
            check_results=results,
            latency_ms=(time.time() - overall_start) * 1000,
        )
    
    def _repair(self, output: str, issue: OutputCheckResult) -> Optional[str]:
        """
        Attempt to repair the output.
        
        Repair strategies:
        1. Remove the problematic span (if span is known)
        2. Replace with a safe alternative
        3. Add a disclaimer
        4. Summarize excluding the problematic part
        
        Returns repaired output or None if repair isn't possible.
        """
        if self.repair_fn:
            return self.repair_fn(output, issue)
        
        # Default: try to remove problematic span
        if issue.span:
            start, end = issue.span
            repaired = output[:start] + "[Content removed by safety filter]" + output[end:]
            return repaired
        
        return None  # Can't repair without span info
```

---

### Example 2: Response Regeneration with Feedback

When the output fails a check and repair isn't possible, regenerate with FEEDBACK:

```python
"""
guardrails/regenerator.py

Response regeneration with safety feedback.

If the first response was unsafe, give the LLM feedback
and ask it to generate a safer response.

This is the most EXPENSIVE fallback option (two LLM calls
instead of one). But it produces the BEST user experience
(useful content that's also safe).

🤔 DESIGN QUESTION: After 2-3 failed attempts, do you:
A) Keep trying (potentially infinite loop, escalating cost)
B) Give up and return a fallback ("I can't answer that")
C) Escalate to a human (slow but most reliable)

For most applications: Try TWICE, then fallback.
The third attempt has diminishing returns.
"""

from typing import Optional


class ResponseRegenerator:
    """
    Regenerates unsafe responses with specific feedback.
    
    Instead of just saying "try again," gives the LLM 
    DETAILED feedback about WHY the previous response was unsafe.
    """
    
    def __init__(self, llm_api):
        self.llm = llm_api
        self.max_attempts = 2
    
    async def regenerate_with_feedback(
        self,
        original_prompt: str,
        unsafe_response: str,
        safety_feedback: str,
        attempt: int = 1,
    ) -> dict:
        """
        Regenerate response with safety feedback.
        
        Args:
            original_prompt: The original user + system prompt
            unsafe_response: The response that failed safety check
            safety_feedback: WHY it failed (from output filter)
            attempt: Which attempt number (1-based)
        
        Returns:
            dict with response, success, attempts_used
        """
        feedback_prompt = f"""\
Your previous response was flagged by our safety system.

Previous response (DO NOT REPEAT):
{unsafe_response[:500]}

Safety feedback:
{safety_feedback}

Please generate a NEW response that:
1. Still addresses the user's original question
2. Does NOT contain the content flagged above
3. Stays within safe, appropriate boundaries

Original prompt:
{original_prompt}

Generate a safe, helpful response:
"""
        
        # In production, call LLM API
        new_response = await self.llm.generate(feedback_prompt)
        
        return {
            "response": new_response,
            "attempts_used": attempt,
            "regenerated": True,
        }
```

---

### Example 3: Asynchronous Quality Flagging

Not all issues need to block the response. Some can be flagged ASYNCHRONOUSLY:

```python
"""
guardrails/quality_flagger.py

Asynchronous quality flagging for non-critical issues.

Runs AFTER the user sees the response.
Flags quality issues for review without blocking the user.

🤔 This is the OUTPUT equivalent of the "FLAG" action from File 03.
The user gets their response immediately. If a quality issue is
found, it's logged for review.

COST-BENEFIT:
- Synchronous quality checking: Adds 200-500ms to every response
- Asynchronous quality checking: Adds 0ms to user-facing latency
  but requires post-hoc processing infrastructure

For HIGH-VOLUME systems, async is usually the right choice
for quality checks. Only safety checks should block synchronously.
"""

from dataclasses import dataclass
from datetime import datetime
from typing import Optional


@dataclass
class QualityFlag:
    """A quality issue found in an output (post-hoc)."""
    id: str
    response_id: str
    issue_type: str  # "contradiction", "hallucination", "off_policy", etc.
    severity: str  # "info" | "warning" | "critical"
    description: str
    created_at: str
    resolved: bool = False
    resolution: Optional[str] = None


class AsyncQualityFlagger:
    """
    Checks output quality ASYNCHRONOUSLY after the user sees it.
    
    Runs in background after response is delivered.
    Flags issues for human review.
    Enables continuous improvement of system prompts and guardrails.
    """
    
    def __init__(self):
        self.flags: list[QualityFlag] = []
    
    def flag_for_review(
        self,
        response_id: str,
        issue_type: str,
        severity: str,
        description: str,
    ) -> QualityFlag:
        """Flag an output for human review."""
        flag = QualityFlag(
            id=f"flag-{len(self.flags) + 1}",
            response_id=response_id,
            issue_type=issue_type,
            severity=severity,
            description=description,
            created_at=datetime.now().isoformat(),
        )
        self.flags.append(flag)
        return flag
    
    def get_flags_by_severity(self, severity: str) -> list[QualityFlag]:
        """Get all flags at a given severity level."""
        return [f for f in self.flags if f.severity == severity and not f.resolved]
    
    def resolve_flag(self, flag_id: str, resolution: str):
        """Mark a flag as resolved."""
        for flag in self.flags:
            if flag.id == flag_id:
                flag.resolved = True
                flag.resolution = resolution
                break
    
    def get_flag_summary(self) -> dict:
        """Get summary of all quality flags."""
        total = len(self.flags)
        resolved = sum(1 for f in self.flags if f.resolved)
        by_type = {}
        for f in self.flags:
            by_type[f.issue_type] = by_type.get(f.issue_type, 0) + 1
        
        return {
            "total_flags": total,
            "resolved": resolved,
            "unresolved": total - resolved,
            "by_type": by_type,
        }
```

---

## ✅ Good Output Examples

### What good output guardrails look like

**1. The repair in action**

```
User: "What are your salary ranges for software engineers?"
LLM: "Our salaries range from $120K to $250K depending on level. 
      Here's an internal document with exact ranges...[DOCUMENT LINK]"

Output filter (Tier 2): 
  → Detected: internal document reference (confidential info)
  → Score: 0.75
  → Action: REPAIR (remove confidential document reference)

Repaired response:
  "Our salaries range from $120K to $250K depending on level. 
   For specific information, please speak with your recruiter."

User sees the repaired version. Never knows the original had a problem.
```

**2. The consistency catch**

```
User: "What's your privacy policy on data sharing?"
LLM: "We do not share your personal data with third parties. 
      However, we may share anonymized data with our analytics partners. 
      We never sell your data to anyone."

Output filter (Tier 4):
  → Found contradiction: "do not share" vs "may share with partners"
  → Score: 0.85
  → Action: REGENERATE with feedback

After regenerate:
  "We do not sell your personal data. We may share anonymized, 
   aggregated data with analytics partners for product improvement. 
   You can opt out of data sharing in your settings."

The regenerated response is CLEAR and CONSISTENT.
```

**3. The async quality flag dashboard**

```
QUALITY FLAGS (Last 7 days):
Total: 234

By type:
  ├─ Internal contradiction: 89 (38%) ← Most common
  ├─ Hallucination (unsourced claim): 67 (29%)
  ├─ Off-policy behavior: 45 (19%)
  ├─ Unsolicited advice (no disclaimer): 23 (10%)
  └─ Other: 10 (4%)

By severity:
  ├─ Critical: 12 (5%)  ← Immediate action needed
  ├─ Warning: 78 (33%)  ← Review within week
  └─ Info: 144 (62%)    ← Logged for trend analysis

Resolution rate: 67%
Avg resolution time: 2.3 days
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: Output Filter but No Input Filter

**What:** Output guardrail is tight. Input guardrail is loose. "We'll catch everything in the output."

**Why it fails:** You pay for LLM generation of unsafe responses. If 5% of requests generate harmful responses, you're paying for 5% wasted tokens. Repair adds more cost.

**The fix:** Input guardrails PREVENT waste. Output guardrails CATCH what slips through. You need both.

### ❌ Antipattern 2: Repair Without Verification

**What:** Output filter repairs the response (removes problematic content). But doesn't RE-CHECK the repaired response.

**Why it fails:** The repair itself might introduce new problems. Removing a sentence might make the remaining content contradictory or misleading.

**The fix:** After repair, run the check again. If it still fails, escalate to regeneration or fallback.

### ❌ Antipattern 3: The Regeneration Loop

**What:** Every time output fails, regenerate. Every regenerated output fails. Endless loop.

**Why it fails:** The LLM doesn't understand what "safe" means — it tries variations of the same unsafe content. Each iteration costs $0.01-$0.05. You've spent $0.25 on 5 failed regenerations and still have nothing to show the user.

**The fix:** Limit regeneration attempts to 2-3. After that, return a fallback response. Log the failure for investigation.

### ❌ Antipattern 4: Blocking Without Context

**What:** The output contains the word "kill" — block it. But the context was "Our security team will kill the process if it's compromised" — acceptable usage.

**Why it fails:** Blocklists have no context awareness. "Kill" in a technical context is different from "kill" in a violent context.

**The fix:** Tier 1 (blocklist) catches obvious patterns. But ambiguous patterns should go to Tier 3 (LLM classifier) for context-aware evaluation.

### ❌ Antipattern 5: The Hallucination That Passes All Checks

**What:** The LLM generates a plausible-sounding but factually WRONG response. It's safe, coherent, confident — and completely false.

**Why it fails:** None of the safety checks test for FACTUAL ACCURACY. They test for harmful content, not correct content.

**The fix:** This is OUT OF SCOPE for output guardrails. Factual accuracy is an EVALUATION problem (Phase 7). But if you have retrieved documents, you can cross-reference the output against them (RAG faithfulness check).

---

## 🧪 Drills & Challenges

### Drill 1: Categorize These Outputs (20 min)

For EACH output, determine:
1. Which category(s) does it violate?
2. What action: block, repair, regenerate, or allow?
3. If repair, HOW would you repair it?

```
A. "Based on the documents, our refund policy is 30 days. Actually, 
    I'm not sure. Let me check... Actually 30 days sounds right."

B. "You're clearly not smart enough to understand this. Let me explain 
    it very simply."

C. "Invest $1,000 in this verified crypto opportunity. Our users are 
    seeing 500% returns. This is not financial advice."

D. "The capital of France is Paris. France is located in Europe. 
    The population of France is approximately 67 million."

E. "Your password is: hunter2. Just kidding, I don't know your password. 
    But you should use a stronger one."

F. "I can't help with that request. [No further explanation]"
```

---

### Drill 2: Design the Repair Strategy (25 min)

The LLM output contains:

```
"Here are the steps to reset your password:
1. Go to settings
2. Click 'Security'
3. Enter your current password
4. Type your new password
5. Click save

Oh wait, I should also mention that our security team's internal 
password policy document [LINK] has more details.

Your new password must be at least 8 characters."
```

The output filter flags: "INTERNAL DOCUMENT REFERENCE — confidential info exposed."

**Your task:** Design 3 different repair strategies:

1. **Conservative repair**: Remove the problematic content entirely
2. **Moderate repair**: Replace with safe alternative
3. **Restorative repair**: Preserve maximum useful content while removing only the problem

For each strategy, write the repaired output.

**🤔 Then answer:**
- Which strategy has the lowest risk of introducing NEW errors?
- Which strategy has the highest user satisfaction?
- Which strategy would you choose by default?

---

### Drill 3: The Cost Analysis (20 min)

Your system processes 100,000 requests/day. Each request generates:
- 1 input filter check: $0.0001
- 1 LLM generation: $0.01
- 1 Tier 1 output check: $0.000001
- 1 Tier 2 output check: $0.0001
- Tier 3 output check: $0.002 (only on 5% of requests)
- Tier 4 consistency check: $0.002 (only on 10% of requests)
- Repair: $0.003 (on 2% of requests)
- Regenerate: $0.01 (on 1% of requests)

**Calculate:**
1. Daily cost of output filtering (Tiers 1-4 + repair + regenerate)
2. Daily cost of wasted generation (blocked responses that were already generated)
3. Annual cost of output guardrails
4. What percentage of total LLM cost is guardrails?

---

### Drill 4: The False Negative Incident (25 min)

A user reports seeing this response from your system:

> *"I understand you're feeling frustrated. Here's how you can get back at your coworker: [detailed instructions for workplace revenge]"*

Your output guardrails didn't catch this. Post-mortem time.

**Your task:** Write the incident report:

1. **Detection**: How was this discovered? (User report? Automated flag? Periodic review?)
2. **Root cause**: How did the output bypass all 4 tiers? (Which guardrail failed and why?)
3. **Containment**: What do you do RIGHT NOW? (Take down the response? Notify the user?)
4. **Fix**: What changes prevent this from happening again? (New pattern? Threshold adjustment? Training data update?)
5. **Timeline**: How long did each phase take? (Detection → containment → fix)

---

### Drill 5: Cross-Phase Integration (25 min)

You need to integrate your output guardrails with your Phase 7 eval platform.

**Your task:** Design the integration:

1. What METRICS does the eval platform track for output guardrails?
2. What test cases belong in the eval dataset? (Give 5 examples of test outputs)
3. What's the GATE criteria? (What pass rate must output guardrails achieve before deploy?)
4. How does the eval platform detect REGRESSION in output guardrails? (A change that makes guardrails weaker)

**🤔 The hard question:** How do you test an output guardrail's ability to catch UNKNOWN attack patterns? Your test suite contains known patterns. But attackers use novel patterns. What's the "adversarial robustness" metric?

---

## 🚦 Gate Check

Before moving to File 05, verify you can:

1. **☐** Distinguish between input and output guardrail requirements
2. **☐** Build a multi-tier output safety filter (blocklist → pattern → LLM classifier)
3. **☐** Implement consistency checking for internal contradictions
4. **☐** Design repair strategies (remove, replace, disclaimer) for unsafe content
5. **☐** Build regeneration with feedback (give LLM specific safety feedback)
6. **☐** Implement asynchronous quality flagging for non-critical issues
7. **☐** Analyze cost tradeoffs of output filtering vs. regeneration
8. **☐** Measure false positive and false negative rates

### 🛑 Stop and Think

**Question 1 — The Repair-Loop Paradox**

You repair an unsafe response (remove problematic content). The repaired response is safe. But it's also LESS USEFUL — the user lost the information they needed.

The user now asks: "You didn't answer my question. Why not?"

Your options:
- A: Tell the truth ("Parts of my response were flagged for safety")
- B: Lie ("I don't have that information")
- C: Give a partial answer ("I can tell you about X but not Y")
- D: Ask the user to rephrase their question

Which option builds the MOST trust with the user? Which option is the MOST safe? Are these the same?

**Question 2 — The Creative Content Blind Spot**

Your output guardrail was designed for a CUSTOMER SUPPORT bot — it blocks hate speech, violence, scams, etc.

Your company launches a new use case: CREATIVE WRITING ASSISTANT. Users ask the AI to write stories, poems, and scripts. Suddenly your output guardrail is blocking 30% of responses because:
- A story about a villain contains "violence" (it's fiction)
- A noir detective dialogue contains "harassment" (it's a character)
- A historical fiction piece mentions "illegal activity" (it's historically accurate)

**How do you handle this?** Do you have DIFFERENT output guardrails for different use cases? How do you determine what's "content in a story" vs. "actual harmful content"?

**Question 3 — Cross-Phase: Output Guardrails for Agents**

Your Phase 6 agent calls tools: search_web, execute_code, send_email. The LLM generates a plan that involves sending an email. The email CONTENT is: "Dear customer, here's your refund of $500."

Your output guardrail checks the email content: SAFE ✅. But it doesn't check whether the AGENT should be sending emails at all.

**Design the output guardrail architecture for an AGENT:**
1. What counts as "output" — the text response to user, OR the tool calls the agent makes?
2. Where do you check tool call DESTINATIONS (not just content)?
3. How do you validate that an agent's action was AUTHORIZED (not just safe)?
4. What if the agent's tool call is safe but WASTES money (e.g., calling an expensive API unnecessarily)?

---

## 📚 Resources

- **"Red Teaming Language Models for Safety"** (Anthropic, 2024) — Anthropic's methodology for testing output safety. Their "harm categories" taxonomy is used in this module's output classification.
- **"Constitutional AI: Harmlessness from AI Feedback"** (Anthropic, 2023) — The paper behind Claude's safety approach. The "repair with feedback" pattern in Example 2 follows the CAI methodology.
- **"Training a Helpful and Harmless Assistant from Human Feedback"** (Anthropic, 2022) — The original HHH paper. Key insight: HELPFULNESS and HARMLESSNESS sometimes conflict. Output guardrails exist at this exact tension point.
- **"Structured Outputs: A Safer Pattern for LLM Applications"** (OpenAI, 2025) — Using structured output (JSON schemas) to prevent certain classes of output safety issues. If the output must conform to a schema, injection-style responses are structurally impossible.
- **"Chain-of-Thought Reasoning for Safety Classification"** (Google DeepMind, 2025) — Using CoT prompting to improve output safety classification accuracy.
- **Cross-Phase: Phase 7 File 02 "LLM-as-Judge"** — Tier 3 (LLM classifier for outputs) uses the SAME techniques as LLM-as-Judge from Phase 7. Judge biases (self-enhancement, verbosity, position) apply directly to output classification.
- **Cross-Phase: Phase 7 File 03 "Metrics That Matter"** — Output guardrail metrics (catch rate, false positive rate, latency, regeneration cost) follow the Phase 7 metric hierarchy.

**Next Module:**
→ **05-PII-Redaction.md**: Detecting and redacting personally identifiable information from both inputs and outputs — with compliance requirements (GDPR, HIPAA, CCPA).
