# 06 — Content Moderation: Keeping AI Outputs Safe at Scale

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about moderation APIs and category classifiers, you need to understand why content moderation is the HARDEST guardrail to get right.

Prompt injection (File 02) is about detecting attacks. Input guardrails (File 03) filter before the LLM. Output guardrails (File 04) filter what the LLM generates. PII redaction (File 05) protects sensitive data.

Content moderation is DIFFERENT. It's not about detecting a specific attack pattern or data type. It's about making JUDGMENT CALLS about what content is acceptable for YOUR platform, YOUR users, YOUR use case, YOUR jurisdiction.

And here's the brutal truth: **Content moderation is ALWAYS wrong for someone.**

- Block too much → users complain about censorship, useful content is lost
- Block too little → users encounter harmful content, regulators fine you
- Block perfectly (impossible) → everyone who WANTED to post harmful content is unhappy they can't

The other brutal truth: **Content moderation at scale is a HUMAN problem masquerading as a technical problem.**

- You can build the best ML classifier in the world
- You can have 99.9% accuracy
- On a platform with 1M daily interactions, that's 1,000 errors per day
- Every error is either: a user who saw something they shouldn't have, or a user who was unfairly blocked
- Both are REAL people with REAL experiences

Moderation is not a "set it and forget it" system. It's a continuous OPERATION with SLAs, escalation paths, appeals, policy updates, and human judgment. The code is the easy part. The PROCESS is the hard part.

---

### 🤔 Discovery Question 1: The Subjectivity Problem

You're building a customer support AI for a social media platform. Your content policy says:

> *"No hate speech. No harassment. No violent content."*

User A writes: *"I hate this company. Their customer service is a complete joke and their product is garbage. I've been waiting 3 weeks for a refund."*

User B writes: *"I hate immigrants. They're ruining this country and should go back where they came from."`

**🤔 Before reading on, answer these:**

1. Both sentences start with "I hate". Both use strong negative language. One is about a company (product feedback), one is about a group of people (hate speech). Your content policy only says "No hate speech" — it doesn't define where the line between "negative feedback" and "hate speech" is. How does your moderation system DISTINGUISH between these?

2. User A's message contains: anger, frustration, negative product feedback, possibly defamation ("their product is garbage" could be considered libel). Should your system block it? Flag it for review? Let it through? What's the principle you use to decide?

3. **The hardest question:** User B's message is clearly hate speech — it attacks a group of people. But what if the exact same sentiment is expressed in a different way?
   - "I hold concerns about the pace of demographic change" (polite version of same idea)
   - "Some people say immigrants are ruining... I disagree, but that's what they claim" (quoting hate speech to argue against it)
   - "I hate immigrants #sarcasm #notreally" (sarcasm tagged version)
   
   Your classifier needs to handle ALL of these differently despite similar surface content. How do you design a system that understands INTENT and CONTEXT, not just keywords?

---

### 🤔 Discovery Question 2: The Scale & Speed Problem

Your platform processes 10,000 messages per minute during peak hours. You have 5 human moderators working 8-hour shifts. Your content policy requires human review for certain categories (threats, self-harm, illegal content).

A user posts: *"I'm going to kill myself tonight. No one cares anyway. Goodbye."*

Your automated system flags it as "self-harm" — requires human review. The message enters a queue. Average wait time: 12 minutes.

**🤔 Before reading on, answer these:**

1. The user posted a self-harm message. It will take 12 minutes for a human to review it. What if the user needed immediate help? What if they're serious? Is a 12-minute delay acceptable for self-harm content?

2. If you make self-hand content IMMEDIATELY actionable (no human review), your system can respond faster: show crisis resources, notify emergency contacts, etc. But what about FALSE POSITIVES? What if someone posts "I'm going to kill this project deadline tonight" (figurative) and your system responds with crisis intervention? You've now escalated something that was never a crisis — that's embarrassing and could be harmful in its own way.

3. **The hardest question:** You can't hire enough moderators to review everything in real-time. It's physically impossible — there are more messages than human reading capacity. Your automated classifier has 95% precision on self-harm detection (meaning 5% of flagged content is NOT actually self-harm). If you auto-respond to flagged self-hand content, 1 in 20 responses will be to a false positive. If you require human review, 12-minute delays on genuine crises. **What's the right balance between speed and accuracy for different categories? Should ALL categories get the same treatment?**

---

### 🤔 Discovery Question 3: The Adversarial Evasion Problem

You've built a content moderation system. It classifies text into 8 categories: safe, hate, harassment, violence, self-harm, sexual, spam, illegal. Your classifier has 98% accuracy on your test set.

An organized group wants to spread hate speech on your platform. They know automated moderation exists. They adapt:

```
Original: "I hate [group]. They're disgusting vermin who should be eliminated."
Adapted:  "I have some concerns about [group]'s impact on our community's values 🫡"
Adapted:  "Not saying I agree with it, but SOME people think [group] is problematic"
Adapted:  "I d!spise [gr0up]. they're d1sgusting ver|n who sh0uld be e|iminated"
```

**🤔 Before reading on, answer these:**

1. The original message is clearly hate speech. But the adapted versions evade keyword filters, evade sentiment classifiers (they use neutral language), and even evade semantic classifiers by using indirection ("SOME people think"). How does your moderation system catch content that DELIBERATELY avoids detection patterns?

2. Every evasion technique has a countermeasure:
   - Leetspeak (gr0up) → normalization layer
   - Neutral language ("concerns about impact") → context analysis, conversation history
   - Indirection ("some people think") → intent inference, source attribution
   - Emoji signaling 🫡 → emoji-to-text mapping
   
   But every countermeasure adds complexity, latency, and false positives. A normalization layer might flag "gr0up" but also flag legitimate uses like "W0rld of Warcraft guild recruitment." At what point does detection complexity outweigh the benefit?

3. **The hardest question:** The group is ORGANIZED. They're iterating on evasion techniques faster than you can add countermeasures. Your moderation team spends all their time playing whack-a-mole — adding rules, updating classifiers, handling new evasion patterns. The attackers only need to succeed ONCE to cause damage. You need to succeed EVERY TIME to prevent it. **Is there a moderation strategy that doesn't depend on detecting specific patterns? What's the "un-gameable" approach?**

---

By the end of this module, you will:

1. **Design a content moderation taxonomy** — categories, severity levels, action mappings
2. **Build a multi-tier moderation pipeline** — automated filtering → ML classification → human review
3. **Implement human-in-the-loop workflows** — queue management, review interfaces, SLAs
4. **Design escalation logic** — category-specific routing, severity-based priority
5. **Handle adversarial evasion** — normalization, pattern-agnostic detection, behavioral signals
6. **Build appeal systems** — user appeals, false positive recovery, policy feedback loops
7. **Measure moderation effectiveness** — precision/recall per category, latency, false positive rate, human review accuracy

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Questions | 25 min |
| Moderation Categories & Taxonomy | 20 min |
| Tiered Moderation Architecture | 25 min |
| Code: Moderation Pipeline | 35 min |
| Code: Human Review Queue | 30 min |
| Code: Escalation & Routing Engine | 30 min |
| Code: Appeal System | 25 min |
| Adversarial Evasion & Countermeasures | 20 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 45 min |
| Gate Check | 15 min |
| **Total** | **~5 hours** |

---

## 📖 Concept Deep-Dive

### 1. Content Moderation Categories

Before you build moderation, you need a CATEGORY TAXONOMY. Not all harmful content is the same, and different categories need different handling:

```
MODERATION CATEGORY TAXONOMY:

CATEGORY           │ SEVERITY │ ACTION         │ REVIEW NEEDED │ SLA
───────────────────┼──────────┼────────────────┼───────────────┼─────
Hate Speech        │ CRITICAL │ Block + Report │ Post-hoc      │ 5 min
Harassment         │ HIGH     │ Block          │ Pre-action    │ 15 min
Violent Content    │ CRITICAL │ Block + Escalate │ Immediate    │ 2 min
Self-Harm          │ CRITICAL │ Respond + Escalate │ Immediate │ 1 min
Sexual Content     │ MEDIUM   │ Flag/Blur      │ Pre-action    │ 30 min
Spam               │ LOW      │ Auto-remove    │ No review     │ N/A
Misinformation     │ MEDIUM   │ Flag + Context │ Pre-action    │ 1 hour
Impersonation      │ HIGH     │ Block + Verify │ Pre-action    │ 15 min
Illegal Activity   │ CRITICAL │ Block + Report │ Legal review  │ 5 min

CRITICAL: Immediate action required, potential real-world harm
HIGH: Significant harm, requires moderation intervention
MEDIUM: Policy violation, non-urgent
LOW: Nuisance, automated handling sufficient

ACTIONS:
Block       → Content never shown to users
Flag        → Content shown with warning/reduced visibility
Escalate    → Sent to specialized team (trust & safety, legal)
Report      → Filed with platform authorities or law enforcement
Respond     → Automated response (crisis resources, disclaimers)
Blur        → Media content hidden behind interstitial
Remove      → Deleted after confirmation
```

**Why category matters for system design:**

Different categories have DIFFERENT requirements:
- **Self-harm needs SPEED** — if you wait 10 minutes for human review, someone could die
- **Misinformation needs ACCURACY** — better to flag cautiously than block legitimate speech
- **Spam needs SCALE** — automated handling because volume is too high for humans
- **Illegal content needs AUDIT** — must preserve evidence for law enforcement, never auto-delete

Your moderation system must handle ALL of these with different policies, SLAs, and workflows. A one-size-fits-all approach fails because the requirements CONTRADICT: self-harm needs fast auto-response, misinformation needs careful human review, spam needs automated deletion.

### 2. Tiered Moderation Architecture

A production moderation system has THREE tiers, each optimized for different tradeoffs:

```
TIER 1: PATTERN-BASED FILTER (Cheap, Fast, Low Accuracy)
────────────────────────────────────────────────────────────
- Regex patterns, keyword lists, blocklist matching
- Latency: < 5ms
- Cost: Negligible (compute only)
- Use: Catch obvious violations, reduce load on higher tiers
- Examples: Block known hate slurs, scam URLs, spam patterns
- Limitation: Can't handle context, nuance, or evasion

  Input → [Regex patterns] → MATCH → Block/Flag (based on severity)
           No match ↓
           [Keyword list] → MATCH → Block/Flag
           No match ↓
                      ↓ To Tier 2

TIER 2: ML CLASSIFIER (Moderate Cost, Good Accuracy)
────────────────────────────────────────────────────────
- Fine-tuned text classifier, zero-shot classifier, or small LLM
- Latency: 50-200ms
- Cost: $0.0001-0.001 per classification
- Use: Semantic understanding within specific categories
- Examples: Hate speech classifier, spam classifier, sentiment + intent
- Limitation: Struggles with subtle/novel patterns, requires training data

  Input → [Category classifiers] → Category + Score
           Score > HIGH_THRESHOLD → Auto-block
           Score > MEDIUM_THRESHOLD → Flag for human review
           Score < MEDIUM_THRESHOLD → To Tier 3

TIER 3: LLM JUDGE (Expensive, Best Accuracy)
────────────────────────────────────────────────────────
- Large LLM with moderation-specific prompting
- Latency: 500-3000ms (depends on model)
- Cost: $0.01-0.10 per classification
- Use: Borderline cases, nuanced decisions, multi-turn context
- Examples: LLM-as-judge for context-dependent violations
- Limitation: Expensive, slow, possible to jailbreak

  Input → [LLM prompt: classify content with context] → Category + Reasoning
           → Action decision based on policy rules
```

**Key insight:** Most content (80-90%) should be handled by Tier 1 and Tier 2. Only 1-5% should reach Tier 3 (the expensive LLM call). And only 0.1-0.5% should need human review.

This is a real production pattern used by platforms like Discord, Reddit, and Twitch — not by calling a single LLM for every piece of content, but by routing through progressively more expensive and accurate filters.

### 3. Human-in-the-Loop Design

Human review is the SAFETY NET for automated moderation. But poorly designed human review creates new problems:

**Queue Design:**
```
Content → Auto-classify → [HUMAN REVIEW QUEUE]
                          ├── Priority 1: CRITICAL (self-harm, violence, illegal)
                          │   SLA: 2 minutes
                          │   Action: Auto-escalate if unassigned after 1 min
                          │
                          ├── Priority 2: HIGH (hate speech, harassment, impersonation)
                          │   SLA: 15 minutes
                          │   Action: Auto-flag content (reduced visibility) if pending > 10 min
                          │
                          ├── Priority 3: MEDIUM (sexual content, misinformation)
                          │   SLA: 1 hour
                          │   Action: Auto-flag with warning banner if pending > 30 min
                          │
                          └── Priority 4: APPEALS (user disputes automated decision)
                              SLA: 24 hours
                              Action: Acknowledge receipt, provide ETA
```

**Moderator Experience:**
- Moderators see: content + context (user history, conversation thread) + current classification + similar past decisions
- Moderators decide: Allow / Block / Escalate / Policy Exception
- Every decision feeds back into the automated system (active learning loop)
- Moderator decisions are SAMPLED for quality assurance (10% audited by senior moderators)

**The Cost Reality:**
```
Human moderation cost calculation:

Time to review one item           : 30 seconds average (simple categories)
                                    to 5 minutes (complex/nuanced categories)
Moderator hourly cost (fully loaded): $25-40/hr (varies by region)
Items per moderator per day        : ~500-800 (with breaks, QA, fatigue)
Team size for 10K items/day        : 12-20 moderators (with shift coverage)
Monthly cost                        : $15,000-30,000+ 

vs.

Automated classification cost:
Tier 1 cost per item                : $0.000001
Tier 2 cost per item                : $0.0005
Tier 3 cost per item                : $0.03
Items per day                       : 10,000
Daily cost (80/15/5 split)         : $0.08 + $0.75 + $15.00 ≈ $15.83/month
```

This is why every platform uses automated moderation as the FIRST line of defense and human review as the LAST resort. Humans are 1,000x more expensive than automated classification per decision.

### 4. Escalation & Routing Logic

Not all flags are equal. Your system needs INTELLIGENT ROUTING:

```
ESCALATION ENGINE:

Input: Content + Category + Confidence Score + User Context

ROUTING DECISIONS:

IF category == SELF_HARM AND score > 0.7:
    → Action: IMMEDIATE_RESPOND (show crisis resources)
    → Route: TIER_1_MODERATOR (priority queue)
    → Notify: Duty manager (SMS/pager)

IF category == HATE_SPEECH AND score > 0.9:
    → Action: AUTO_BLOCK
    → Route: NO_REVIEW (auto-enforced, logged for audit)
    → Audit: Random sample for quality check

IF category == HATE_SPEECH AND 0.5 < score < 0.9:
    → Action: FLAG_FOR_REVIEW (content hidden pending review)
    → Route: STANDARD_QUEUE
    → Policy: If 3+ moderators disagree → escalate to policy team

IF category == MISINFORMATION:
    → Action: FLAG_WITH_CONTEXT (content visible with warning banner)
    → Route: FACT_CHECK_QUEUE (specialized reviewers)
    → Priority: Based on virality/ reach (higher reach = higher priority)

IF category == SPAM AND score > 0.7:
    → Action: AUTO_REMOVE
    → Route: NO_REVIEW (high confidence, low impact if wrong)
    → User action: Auto-suspend after N spam violations

IF content was previously appealed AND upheld:
    → Action: AUTO_ALLOW (double jeopardy prevention)
    → Log: Previous appeal decision
```

The escalation engine must also handle CONFLICTING signals:
- Tier 1 says SAFE, Tier 2 says VIOLATION, Tier 3 says SAFE → which wins?
- Answer: **Tier with highest confidence wins, but always log the disagreement for model improvement**

### 5. Adversarial Evasion & Countermeasures

Content moderation is an ARMS RACE. Attackers evolve as defenses improve:

| Evasion Technique | Description | Countermeasure |
|---|---|---|
| **Lexical obfuscation** | Leetspeak, misspellings, word breaks | Normalization layer (before classification) |
| **Semantic drift** | Use code words, dog whistles, in-group language | Community-specific models, context analysis |
| **Structuring** | Break content into safe parts across messages | Conversation-level analysis, stitching detection |
| **Reframing** | Quote hate speech to "argue against" it | Speaker attribution, intent classification |
| **Sarcasm/irony** | Surface-level safe, actually harmful | Tone detection, contradictory signals |
| **Image-based text** | Post hate speech as screenshots | OCR pipeline for images, multimodal moderation |
| **Gradual escalation** | Start safe, slowly push boundaries | User-level risk scoring, trajectory analysis |
| **Coordinated inauthentic** | Many accounts, same message, looks organic | Network analysis, pattern detection at account level |

**The normalization layer is your first defense:**

```python
# Pseudo-code for text normalization
def normalize_text(text: str) -> str:
    # 1. Lowercase
    text = text.lower()
    
    # 2. Normalize leetspeak substitutions
    leet_map = {
        '0': 'o', '1': 'i', '3': 'e', '4': 'a',
        '5': 's', '7': 't', '8': 'b',
        '@': 'a', '$': 's', '!': 'i', '+': 't'
    }
    for char, replacement in leet_map.items():
        text = text.replace(char, replacement)
    
    # 3. Normalize unicode (confusables, homoglyphs)
    # e.g., Cyrillic 'а' looks like Latin 'a'
    text = unicode_normalize(text)
    
    # 4. Remove zero-width characters, invisible Unicode
    text = strip_invisible_chars(text)
    
    # 5. Normalize whitespace (multiple spaces → single)
    text = re.sub(r'\s+', ' ', text).strip()
    
    return text
```

**But normalization is not enough.** Sophisticated attackers use SEMANTIC evasion — saying something harmful without using any flagged words:

> "I'm not saying anything bad, but I think you know what I mean wink wink"

This requires behavioral signals: user history, account age, network patterns, posting velocity. Moderation at the platform level is NOT just about individual messages — it's about user-level and community-level patterns.

---

## 💻 CODE EXAMPLES

### Example 1: Multi-Tier Moderation Pipeline (The Naive Approach — Flawed)

Let's start with the approach most beginners take, and discover why it fails:

```python
import re
import time
from typing import Optional
from openai import OpenAI

# BAD APPROACH: Single-pass LLM moderation
# Problem: Every single message hits the LLM → expensive, slow, no prioritization

client = OpenAI()

class NaiveModerator:
    """
    A naive content moderation system that:
    - Calls OpenAI Moderation API for every message
    - Falls back to the LLM for ambiguous cases
    - Has no tiering, no prioritization, no human review
    
    Problems:
    1. Every message costs $0.01-0.03 for LLM classification
    2. No real-time capability (1-3 second latency per message)
    3. No human review path — decisions are final and automated
    4. No context awareness — each message judged independently
    """
    
    def __init__(self):
        self.blocked_categories = [
            "hate", "harassment", "violence", "self-harm",
            "sexual", "illegal"
        ]
        self.category_action = {
            "hate": "block",
            "harassment": "block",
            "violence": "block",
            "self-harm": "flag",
            "sexual": "flag",
            "illegal": "block"
        }
    
    def moderate_message(self, message: str) -> dict:
        """
        Moderate a single message. Directly calls LLM for classification.
        
        PROBLEM 1: No pre-filtering
        - Even "What's the weather today?" hits the LLM
        - Cost adds up fast at scale
        
        PROBLEM 2: No categorization routing
        - Self-harm and spam get the same treatment
        - No distinction between "immediate action needed" vs "can wait"
        
        PROBLEM 3: No human fallback
        - If the LLM is wrong, the decision is still enforced
        - No appeal path, no second review
        """
        # Call moderation API (costs money every time)
        response = client.moderations.create(input=message)
        result = response.results[0]
        
        # Find which categories were flagged
        flagged = []
        for category in self.blocked_categories:
            category_key = category.replace("-", "_")
            if getattr(result.categories, category_key, False):
                flagged.append(category)
        
        if not flagged:
            return {"action": "allow", "reason": "No violations detected"}
        
        # PROBLEM: Flat decision, no nuance
        # - All violations get the same action regardless of severity
        # - "I hate this product" (negative feedback) == "I hate [group]" (hate speech)
        # - Both get the same treatment
        decision = self.category_action.get(flagged[0], "flag")
        
        return {
            "action": decision,
            "category": flagged,
            "scores": self._extract_scores(result),
            "auto_action": True  # PROBLEM: No human review
        }
    
    def _extract_scores(self, result) -> dict:
        return {
            "hate": result.category_scores.hate,
            "hate/threatening": result.category_scores.hate_threatening,
            "harassment": result.category_scores.harassment,
            "self_harm": result.category_scores.self_harm,
            "sexual": result.category_scores.sexual,
            "violence": result.category_scores.violence,
        }


# DEMONSTRATION OF FAILURES:

moderator = NaiveModerator()

# Test 1: Clearly safe (but still hits the LLM)
result1 = moderator.moderate_message(
    "What's the weather like today?"
)
# => action: "allow" (but cost $0.01 for nothing)

# Test 2: Product feedback (not actually hate speech)
result2 = moderator.moderate_message(
    "I hate this company's customer service"
)
# => May flag as "harassment" or "hate" => BLOCKED
# Problem: This is LEGITIMATE negative feedback!
# The user is complaining about a service, not attacking a group

# Test 3: Self-harm (needs IMMEDIATE human response)
result3 = moderator.moderate_message(
    "I want to kill myself"
)
# => action: "flag"
# Problem: "flag" just means "mark it" — no crisis resources shown
# No automatic response, no immediate human review
# Just silently flagged while the user waits

# Test 4: Adversarial evasion
result4 = moderator.moderate_message(
    "i h4te th1s gr0up. they're d1sgusting ver|n"
)
# => May or may not be flagged (depends on LLM's normalization)
# Problem: No normalization layer → attackers easily evade
```

### 🔍 Find The Bugs

Before reading the fixed version, answer these:

1. **The cost bug**: This system costs $0.01-0.03 per message. At 10K messages/day, that's $100-300/day just for moderation. Most startups can't afford this. What's the first thing you'd add to REDUCE cost?

2. **The latency bug**: Every message takes 1-3 seconds for moderation. Real-time chat needs <100ms for a good user experience. Users typing messages experience 3-second delays before their message appears. How would you architect for SPEED?

3. **The routing bug**: Self-harm content gets "flagged" — same as borderline harassment. But self-harm needs IMMEDIATE response (crisis resources shown, human reviewer notified within seconds). Spam can wait 30 minutes. All categories are NOT equal. What should change?

4. **The human-in-the-loop bug**: This system is fully automated. No human can override, no appeals, no second review. What happens when the LLM classifier is WRONG about a borderline case? The user has no recourse.

---

### Example 2: Production Moderation Pipeline (Fixed)

Now let's build it properly with tiers, routing, and human review:

```python
"""
Production Content Moderation Pipeline

Architecture:
- Tier 1: Regex + keyword filter (ultra-fast, zero cost)
- Tier 2: ML classifier (fast, moderate cost)
- Tier 3: LLM judge (slow, expensive, only borderline cases)
- Tier 4: Human review (slowest, most expensive, last resort)

Routing Rules:
- CRITICAL categories (self-harm, violence) → Immediate action + notify
- HIGH categories (hate, harassment) → Auto-block if confident, else review
- MEDIUM categories (sexual, misinformation) → Flag + contextual action
- LOW categories (spam) → Auto-remove
- Appeals → Always human-reviewed
"""

import enum
import re
import time
import json
from dataclasses import dataclass, field
from typing import Optional, Callable
from enum import Enum
from datetime import datetime, timedelta


class ContentCategory(Enum):
    SAFE = "safe"
    HATE_SPEECH = "hate_speech"
    HARASSMENT = "harassment"
    VIOLENCE = "violence"
    SELF_HARM = "self_harm"
    SEXUAL = "sexual"
    SPAM = "spam"
    MISINFORMATION = "misinformation"
    ILLEGAL = "illegal"
    IMPERSONATION = "impersonation"


class SeverityLevel(Enum):
    CRITICAL = "critical"  # Immediate action: self-harm, violence, illegal
    HIGH = "high"         # Quick action: hate, harassment, impersonation
    MEDIUM = "medium"     # Flag + review: sexual, misinformation
    LOW = "low"           # Auto-handle: spam


class ActionType(Enum):
    ALLOW = "allow"
    BLOCK = "block"
    FLAG = "flag"
    ESCALATE = "escalate"
    RESPOND = "respond"  # Automated response (e.g., crisis resources)
    REPORT = "report"    # Report to authorities
    REMOVE = "remove"
    FLAG_WITH_CONTEXT = "flag_with_context"  # Show with warning


@dataclass
class ModerationResult:
    content_id: str
    message: str
    category: ContentCategory
    severity: SeverityLevel
    action: ActionType
    confidence: float
    tier_used: int  # 1=pattern, 2=ML, 3=LLM, 4=human
    requires_human_review: bool
    human_reviewed: bool = False
    human_decision: Optional[str] = None
    needs_immediate_response: bool = False
    escalation_reason: Optional[str] = None
    normalized_text: str = ""
    timestamp: datetime = field(default_factory=datetime.now)


class NormalizationEngine:
    """
    Text normalization to handle evasion attempts.
    Runs BEFORE any classifiers to ensure consistent input.
    """
    
    def __init__(self):
        # Leetspeak substitution map
        self.leet_map = {
            '0': 'o', '1': 'i', '2': 'z', '3': 'e', '4': 'a',
            '5': 's', '6': 'g', '7': 't', '8': 'b', '9': 'g',
            '@': 'a', '$': 's', '!': 'i', '+': 't',
            '<': 'l', '>': '', '(': '', ')': '',
            '|': 'i', '\\': '', '/': '', '.': '',
            ',': '', '"': '', "'": '',
        }
        
        # Zero-width and invisible Unicode characters
        self.invisible_chars = re.compile(
            '[\u200b\u200c\u200d\u2060\u2061\u2062\u2063\u2064'
            '\u2066\u2067\u2068\u2069\ufeff\u00ad\u034f'
            '\u061c\u115f\u1160\u17b4\u17b5\u180e\u2000'
            '\u2001\u2002\u2003\u2004\u2005\u2006\u2007'
            '\u2008\u2009\u200a\u2028\u2029\u202a\u202b'
            '\u202c\u202d\u202e\u202f\u205f]'
        )
    
    def normalize(self, text: str) -> str:
        """Full normalization pipeline."""
        text = text.lower()
        text = self._normalize_leet(text)
        text = self._strip_invisible(text)
        text = self._normalize_whitespace(text)
        return text
    
    def _normalize_leet(self, text: str) -> str:
        result = []
        for char in text:
            result.append(self.leet_map.get(char, char))
        return ''.join(result)
    
    def _strip_invisible(self, text: str) -> str:
        return self.invisible_chars.sub('', text)
    
    def _normalize_whitespace(self, text: str) -> str:
        return re.sub(r'\s+', ' ', text).strip()


class Tier1PatternFilter:
    """
    Ultra-fast regex + keyword filter.
    Latency: < 5ms
    Cost: Negligible
    Purpose: Catch obvious violations, reduce load on higher tiers.
    """
    
    def __init__(self):
        # Category-specific patterns
        self.patterns = {
            ContentCategory.HATE_SPEECH: {
                "high_confidence": [
                    r'\b(racial slur patterns)\b',
                    # ... real patterns would go here
                ],
                "medium_confidence": [
                    r'\b(hate|despise|can\'?t stand)\s+(those|these)\s+\w+',
                ],
            },
            ContentCategory.SELF_HARM: {
                "high_confidence": [
                    r'\b(kill myself|end my life|suicide|want to die)\b',
                ],
            },
            ContentCategory.SEXUAL: {
                "high_confidence": [
                    r'(porn|sexual content|explicit patterns)',
                ],
            },
            ContentCategory.SPAM: {
                "high_confidence": [
                    r'(buy now|click here|limited offer|crypto giveaway)',
                    r'(http[s]?://(?:[a-zA-Z]|[0-9]|[$-_@.&+]){50,})',
                ],
            },
        }
    
    def classify(self, text: str) -> Optional[tuple[ContentCategory, float]]:
        """
        Returns (category, confidence) if matched, None otherwise.
        
        Only returns if confidence > 0.8 (HIGH threshold)
        to avoid false positives from partial matches.
        """
        normalized = text.lower()
        
        for category, levels in self.patterns.items():
            # Check high-confidence patterns first
            for pattern in levels["high_confidence"]:
                if re.search(pattern, normalized):
                    return (category, 0.95)
            
            # Check medium-confidence patterns
            for pattern in levels["medium_confidence"]:
                if re.search(pattern, normalized):
                    return (category, 0.85)
        
        return None


class Tier2MLClassifier:
    """
    ML-based classifier for semantic understanding.
    In production, this would load a fine-tuned model (e.g., BERT-based).
    Here we simulate with a rule-based fallback to demonstrate the pattern.
    
    Latency: 50-200ms
    Cost: $0.0001-0.001 per call (6-100x cheaper than LLM)
    """
    
    def __init__(self):
        # In production: self.model = load_model("moderation-bert")
        pass
    
    def classify(self, text: str, normalized: str) -> Optional[tuple[ContentCategory, float, dict]]:
        """
        Returns (category, confidence, metadata) if detected.
        
        In production, this uses a fine-tuned classifier model.
        For this demo, we simulate the interface.
        """
        # This is a STUB — in real code you'd call your model
        # scores = self.model.predict([normalized])[0]
        
        # Simulated return for demonstration
        return None  # No confident classification from ML layer
        
    def _extract_features(self, text: str, normalized: str) -> dict:
        """Feature extraction for classifier input (in production)."""
        return {
            "text_length": len(text),
            "has_urls": bool(re.search(r'https?://', text)),
            "has_caps_ratio": sum(1 for c in text if c.isupper()) / max(len(text), 1),
            "exclamation_count": text.count('!'),
            "question_count": text.count('?'),
            "has_normalized_leetspeak": text != normalized,
        }


class Tier3LLMJudge:
    """
    LLM-as-judge for borderline and nuanced cases.
    Only fires when Tier 1 and Tier 2 are uncertain.
    
    Latency: 500-3000ms
    Cost: $0.01-0.05 per call
    """
    
    def __init__(self, client=None):
        self.client = client  # OpenAI or other LLM client
        self.system_prompt = """You are a content moderation judge. Your task is to 
classify the following user message into ONE category.

IMPORTANT CONTEXT:
- The user's message is part of a conversation
- Not every mention of a sensitive topic is a violation
- "I hate this product" is NEGATIVE FEEDBACK, not hate speech
- "I hate [group of people]" targeting a protected group IS hate speech
- Self-harm threats require immediate crisis response
- Product complaints and criticism are ALLOWED

CATEGORIES:
- safe: Normal content, questions, opinions, feedback
- hate_speech: Attacks on protected groups (race, religion, gender, orientation, etc.)
- harassment: Targeted abuse toward an individual
- violence: Promotion or threat of physical harm
- self_harm: Suicide, self-injury, eating disorders
- sexual: Explicit sexual content
- spam: Commercial spam, scams, phishing
- misinformation: False claims presented as facts
- illegal: Promotion of illegal activities

Respond ONLY with a JSON object: {"category": "...", "confidence": 0.0-1.0, "reasoning": "..."}"""
    
    def classify(self, text: str, context: Optional[dict] = None) -> tuple[ContentCategory, float, str]:
        """
        Use LLM to classify borderline content.
        In production, this would make an actual API call.
        """
        # In production:
        # response = self.client.chat.completions.create(
        #     model="gpt-4o-mini",  # Small, fast model for cost efficiency
        #     messages=[
        #         {"role": "system", "content": self.system_prompt},
        #         {"role": "user", "content": text}
        #     ],
        #     response_format={"type": "json_object"},
        #     temperature=0,
        #     max_tokens=100
        # )
        # result = json.loads(response.choices[0].message.content)
        # return (ContentCategory(result["category"]), result["confidence"], result["reasoning"])
        
        # For demo purposes
        return (ContentCategory.SAFE, 0.7, "simulated response")


class EscalationEngine:
    """
    Routes content to the right action based on category + confidence.
    Handles all the business logic of WHAT to do with a classification.
    """
    
    # Category → severity mapping
    CATEGORY_SEVERITY = {
        ContentCategory.SELF_HARM: SeverityLevel.CRITICAL,
        ContentCategory.VIOLENCE: SeverityLevel.CRITICAL,
        ContentCategory.ILLEGAL: SeverityLevel.CRITICAL,
        ContentCategory.HATE_SPEECH: SeverityLevel.HIGH,
        ContentCategory.HARASSMENT: SeverityLevel.HIGH,
        ContentCategory.IMPERSONATION: SeverityLevel.HIGH,
        ContentCategory.SEXUAL: SeverityLevel.MEDIUM,
        ContentCategory.MISINFORMATION: SeverityLevel.MEDIUM,
        ContentCategory.SPAM: SeverityLevel.LOW,
        ContentCategory.SAFE: SeverityLevel.LOW,
    }
    
    # Decision thresholds per category
    # Different categories have different confidence requirements
    CATEGORY_THRESHOLDS = {
        ContentCategory.SELF_HARM: {
            "auto_action": 0.6,  # Lower threshold — better safe than sorry
            "action": ActionType.ESCALATE,
            "requires_review": False,  # Immediate action, review later
        },
        ContentCategory.VIOLENCE: {
            "auto_action": 0.7,
            "action": ActionType.BLOCK,
            "requires_review": True,
        },
        ContentCategory.HATE_SPEECH: {
            "auto_action": 0.9,  # High threshold — avoid false positives
            "action": ActionType.BLOCK,
            "requires_review": True,
        },
        ContentCategory.SPAM: {
            "auto_action": 0.7,
            "action": ActionType.REMOVE,
            "requires_review": False,  # Low stakes
        },
        ContentCategory.SEXUAL: {
            "auto_action": 0.95,  # Very high threshold for auto-action
            "action": ActionType.FLAG,
            "requires_review": True,
        },
        ContentCategory.MISINFORMATION: {
            "auto_action": 0.95,
            "action": ActionType.FLAG_WITH_CONTEXT,
            "requires_review": True,
        },
    }
    
    def route(self, category: ContentCategory, confidence: float,
              tier_used: int) -> ModerationResult:
        """
        Determine the action based on category and confidence.
        
        Key design decisions:
        - Self-harm: Low threshold (0.6) for escalation → immediate help
        - Hate speech: High threshold (0.9) for blocking → avoid censoring criticism
        - Spam: Medium threshold → acceptable false positive rate
        - Sexual: Very high threshold → avoid blocking legitimate content
        
        Each decision is a TRADEOFF between safety and user experience.
        """
        severity = self.CATEGORY_SEVERITY[category]
        
        if category == ContentCategory.SAFE:
            return ModerationResult(
                content_id="",
                message="",
                category=category,
                severity=severity,
                action=ActionType.ALLOW,
                confidence=confidence,
                tier_used=tier_used,
                requires_human_review=False,
            )
        
        thresholds = self.CATEGORY_THRESHOLDS.get(category, {
            "auto_action": 0.9,
            "action": ActionType.FLAG,
            "requires_review": True,
        })
        
        if confidence >= thresholds["auto_action"]:
            # High confidence → auto-enforce
            action = thresholds["action"]
            requires_review = thresholds["requires_review"]
            
            # CRITICAL categories need escalation regardless
            if severity == SeverityLevel.CRITICAL:
                return ModerationResult(
                    content_id="",
                    message="",
                    category=category,
                    severity=severity,
                    action=action,
                    confidence=confidence,
                    tier_used=tier_used,
                    requires_human_review=True,  # Always review critical
                    needs_immediate_response=(category == ContentCategory.SELF_HARM),
                    escalation_reason=f"Critical severity: auto-action={action.value}",
                )
            
            return ModerationResult(
                content_id="",
                message="",
                category=category,
                severity=severity,
                action=action,
                confidence=confidence,
                tier_used=tier_used,
                requires_human_review=requires_review,
            )
        
        else:
            # Below auto-action threshold → human review
            # For borderline cases, FLAG (show with moderation) rather than block
            return ModerationResult(
                content_id="",
                message="",
                category=category,
                severity=severity,
                action=ActionType.FLAG,
                confidence=confidence,
                tier_used=tier_used,
                requires_human_review=True,
                escalation_reason=f"Below auto-action threshold ({confidence:.2f} < {thresholds['auto_action']})",
            )


class HumanReviewQueue:
    """
    Manages the queue of content needing human review.
    Different priorities have different SLAs.
    
    This is a simplified version — in production this would be
    backed by a database with real-time updates, assignment logic,
    moderator workload balancing, QA sampling, etc.
    """
    
    def __init__(self):
        self.queues = {
            SeverityLevel.CRITICAL: [],  # SLA: 2 minutes
            SeverityLevel.HIGH: [],      # SLA: 15 minutes
            SeverityLevel.MEDIUM: [],    # SLA: 1 hour
            SeverityLevel.LOW: [],       # SLA: 24 hours
            "appeals": [],               # SLA: 24 hours
        }
    
    def enqueue(self, result: ModerationResult):
        """Add an item to the review queue."""
        queue = self.queues.get(result.severity, self.queues[SeverityLevel.LOW])
        queue.append(result)
        
        # Log for monitoring
        print(f"[QUEUE] {result.category.value} → {result.severity.value} queue "
              f"(SLA: {self._get_sla(result.severity)}, "
              f"confidence: {result.confidence:.2f})")
    
    def _get_sla(self, severity: SeverityLevel) -> str:
        return {
            SeverityLevel.CRITICAL: "2 minutes",
            SeverityLevel.HIGH: "15 minutes",
            SeverityLevel.MEDIUM: "1 hour",
            SeverityLevel.LOW: "24 hours",
        }.get(severity, "24 hours")
    
    def get_pending_count(self, severity: Optional[SeverityLevel] = None) -> dict:
        """Get queue sizes for monitoring dashboards."""
        if severity:
            return {severity: len(self.queues[severity])}
        return {
            level: len(items)
            for level, items in self.queues.items()
        }
    
    def dequeue_next(self, moderator_id: str) -> Optional[ModerationResult]:
        """
        Get the highest-priority pending item for a moderator.
        Always serves CRITICAL first, then HIGH, etc.
        """
        for priority in [SeverityLevel.CRITICAL, SeverityLevel.HIGH,
                         SeverityLevel.MEDIUM, SeverityLevel.LOW]:
            if self.queues[priority]:
                return self.queues[priority].pop(0)
        return None


class CrisisResponse:
    """
    Automated response for self-harm and crisis content.
    This runs IMMEDIATELY, before human review.
    
    Why: When someone talks about suicide, every second counts.
    Waiting 2+ minutes for human review could be fatal.
    """
    
    CRISIS_RESOURCES = {
        "us": {
            "name": "National Suicide Prevention Lifeline",
            "number": "988",
            "website": "https://988lifeline.org/",
            "message": "If you're having thoughts of suicide, please reach out for help. "
                       "You're not alone — there are people who care and want to support you."
        },
        "uk": {
            "name": "Samaritans",
            "number": "116 123",
            "website": "https://www.samaritans.org/",
            "message": "If things are getting too much, Samaritans are here to listen. "
                       "You can call them 24/7 for free."
        },
        "global": {
            "name": "International Association for Suicide Prevention",
            "website": "https://www.iasp.info/resources/Crisis_Centres/",
            "message": "Please reach out to a crisis helpline in your country. "
                       "There are people who want to help you through this."
        }
    }
    
    @classmethod
    def respond(cls, content_id: str) -> dict:
        """
        Generate an automated crisis response.
        The response is shown to the user immediately, alongside
        routing the content to the critical review queue.
        """
        # In production, you'd detect the user's region
        # to show appropriate resources
        resources = cls.CRISIS_RESOURCES["global"]
        
        return {
            "type": "crisis_response",
            "content_id": content_id,
            "response": resources["message"],
            "resources": resources,
            "timestamp": datetime.now().isoformat(),
        }


class ProductionModerationPipeline:
    """
    Complete content moderation pipeline with all tiers.
    
    Flow:
    Message → Normalize → Tier 1 (Pattern) → Tier 2 (ML) → Tier 3 (LLM) → Escalate
                ↑                                                           |
                └───────────────── Human Review ←───────────────────────────┘
    """
    
    def __init__(self, openai_client=None):
        self.normalizer = NormalizationEngine()
        self.tier1 = Tier1PatternFilter()
        self.tier2 = Tier2MLClassifier()
        self.tier3 = Tier3LLMJudge(openai_client)
        self.escalation = EscalationEngine()
        self.review_queue = HumanReviewQueue()
        self.stats = {
            "tier1_handled": 0,
            "tier2_handled": 0,
            "tier3_handled": 0,
            "human_review_required": 0,
            "total_processed": 0,
        }
    
    def moderate(self, content_id: str, message: str,
                 context: Optional[dict] = None) -> ModerationResult:
        """
        Moderate a single message through the tiered pipeline.
        
        Goal: Handle 80%+ in Tier 1/2, 15% in Tier 3, <5% human review.
        """
        start_time = time.time()
        self.stats["total_processed"] += 1
        
        # Step 1: Always normalize first
        normalized = self.normalizer.normalize(message)
        
        # Step 2: Tier 1 — Pattern Filter (~5ms)
        tier1_result = self.tier1.classify(normalized)
        
        if tier1_result:
            category, confidence = tier1_result
            result = self.escalation.route(category, confidence, tier_used=1)
            result.content_id = content_id
            result.message = message
            result.normalized_text = normalized
            
            self.stats["tier1_handled"] += 1
            
            # CRITICAL: Handle self-harm immediately
            if result.needs_immediate_response:
                crisis = CrisisResponse.respond(content_id)
                result.escalation_reason = (
                    f"{result.escalation_reason or ''} | "
                    f"Crisis response triggered: {crisis['response'][:50]}..."
                )
                # Content still goes to human review for follow-up
            
            if result.requires_human_review:
                self.stats["human_review_required"] += 1
                self.review_queue.enqueue(result)
            
            elapsed = (time.time() - start_time) * 1000
            result.timestamp = datetime.now()
            return result
        
        # Step 3: Tier 2 — ML Classifier (~100ms)
        tier2_result = self.tier2.classify(message, normalized)
        
        if tier2_result:
            category, confidence, metadata = tier2_result
            result = self.escalation.route(category, confidence, tier_used=2)
            result.content_id = content_id
            result.message = message
            result.normalized_text = normalized
            
            self.stats["tier2_handled"] += 1
            
            if result.needs_immediate_response:
                CrisisResponse.respond(content_id)
            
            if result.requires_human_review:
                self.stats["human_review_required"] += 1
                self.review_queue.enqueue(result)
            
            elapsed = (time.time() - start_time) * 1000
            result.timestamp = datetime.now()
            return result
        
        # Step 4: Tier 3 — LLM Judge (~1-3 seconds, expensive)
        # Only reached if Tier 1 AND Tier 2 returned None
        category, confidence, reasoning = self.tier3.classify(message, context)
        
        result = self.escalation.route(category, confidence, tier_used=3)
        result.content_id = content_id
        result.message = message
        result.normalized_text = normalized
        
        self.stats["tier3_handled"] += 1
        
        if result.needs_immediate_response:
            CrisisResponse.respond(content_id)
        
        if result.requires_human_review:
            self.stats["human_review_required"] += 1
            self.review_queue.enqueue(result)
        
        elapsed = (time.time() - start_time) * 1000
        result.timestamp = datetime.now()
        return result


# DEMONSTRATION

pipeline = ProductionModerationPipeline()

# Test 1: Safe message → Tier 1 handles it (no match) → Tier 2 (no match) → Tier 3 → SAFE
result1 = pipeline.moderate(
    "msg_001",
    "What's the weather like today?"
)
print(f"Test 1 - {result1.category.value}: {result1.action.value} "
      f"(Tier {result1.tier_used}, conf={result1.confidence:.2f})")

# Test 2: Product feedback (should NOT be hate speech)
result2 = pipeline.moderate(
    "msg_002",
    "I hate this company's customer service. They're incompetent."
)
print(f"Test 2 - {result2.category.value}: {result2.action.value} "
      f"(Tier {result2.tier_used}, conf={result2.confidence:.2f})")

# Test 3: Self-harm → Tier 1 catches it → IMMEDIATE crisis response
result3 = pipeline.moderate(
    "msg_003",
    "I want to kill myself. I can't take this anymore."
)
print(f"Test 3 - {result3.category.value}: {result3.action.value} "
      f"(Tier {result3.tier_used}, conf={result3.confidence:.2f}, "
      f"crisis={'YES' if result3.needs_immediate_response else 'NO'})")

# Test 4: Adversarial evasion → Normalized, then caught
result4 = pipeline.moderate(
    "msg_004",
    "i h4te th1s gr0up of pe0ple. they're d1sgusting"
)
print(f"Test 4 - {result4.category.value}: {result4.action.value} "
      f"(Tier {result4.tier_used}, conf={result4.confidence:.2f})")

# Test 5: Legitimate criticism of a policy
result5 = pipeline.moderate(
    "msg_005",
    "I think this policy is discriminatory and harms our community. "
    "Here's why: it disproportionately affects minority groups..."
)
print(f"Test 5 - {result5.category.value}: {result5.action.value} "
      f"(Tier {result5.tier_used}, conf={result5.confidence:.2f})")

# Print stats
print(f"\nPipeline Stats:")
print(f"  Total processed: {pipeline.stats['total_processed']}")
print(f"  Tier 1 handled:  {pipeline.stats['tier1_handled']}")
print(f"  Tier 2 handled:  {pipeline.stats['tier2_handled']}")
print(f"  Tier 3 handled:  {pipeline.stats['tier3_handled']}")
print(f"  Human review:    {pipeline.stats['human_review_required']}")
```


### Example 3: Appeal System

Every automated moderation decision can be WRONG. Users need a way to dispute decisions:

```python
"""
Appeal System for Content Moderation

The appeal system is NOT just a "re-do the classification."
It must:
1. Acknowledge the appeal immediately (user needs to know they're heard)
2. Re-evaluate with DIFFERENT context (original decision may have been wrong)
3. Route to a DIFFERENT moderator (not the one who made the original decision)
4. Feed back into the model (if automated decision was wrong, improve it)
5. Track appeal outcomes (some users abuse the appeal system)
"""

from datetime import datetime, timedelta
from enum import Enum
from typing import Optional, Callable


class AppealStatus(Enum):
    PENDING = "pending"
    UNDER_REVIEW = "under_review"
    UPHELD = "upheld"        # Original decision corrected
    REJECTED = "rejected"     # Original decision confirmed
    ESCALATED = "escalated"   # Needs policy team review


class UserAppealState(Enum):
    GOOD_STANDING = "good_standing"        # Normal user
    FREQUENT_APPELLANT = "frequent"         # Appeals often (may abuse system)
    BANNED = "banned"                        # Previously upheld appeals for actual violations


@dataclass
class Appeal:
    appeal_id: str
    content_id: str
    user_id: str
    original_decision: ModerationResult
    user_reason: str
    status: AppealStatus
    created_at: datetime = field(default_factory=datetime.now)
    resolved_at: Optional[datetime] = None
    moderator_id: Optional[str] = None
    resolution_reason: Optional[str] = None
    model_feedback: Optional[dict] = None  # For model improvement


class AppealSystem:
    """
    Handles user appeals against moderation decisions.
    
    Design principles:
    1. Speed matters — acknowledge immediately, even if review takes time
    2. Fresh eyes — never have the same moderator review their own decision
    3. Double jeopardy protection — don't re-flag content already appealed
    4. Abusive appellants — detect patterns of false appeals
    5. Model improvement — every upheld appeal is training data
    """
    
    def __init__(self):
        self.pending_appeals: list[Appeal] = []
        self.resolved_appeals: list[Appeal] = []
        self.user_appeal_counts: dict[str, int] = {}
        self.double_jeopardy_protection: set[str] = set()
        
        # SLA tracking
        self.sla_hours = 24  # Default SLA for appeals
        self.critical_sla_hours = 2  # For self-harm, violence appeals
    
    def submit_appeal(self, content_id: str, user_id: str,
                      original_decision: ModerationResult,
                      user_reason: str) -> Appeal:
        """
        Submit an appeal against a moderation decision.
        
        Key check: Double jeopardy protection
        If the user has previously appealed THIS content and it was
        upheld (decision changed), the system auto-allows the new appeal.
        """
        # Double jeopardy protection
        protection_key = f"{user_id}:{content_id}"
        if protection_key in self.double_jeopardy_protection:
            return Appeal(
                appeal_id="auto_upheld",
                content_id=content_id,
                user_id=user_id,
                original_decision=original_decision,
                user_reason=user_reason,
                status=AppealStatus.UPHELD,
                resolved_at=datetime.now(),
                moderator_id="system",
                resolution_reason="Double jeopardy protection: previously upheld appeal",
            )
        
        appeal = Appeal(
            appeal_id=f"appeal_{len(self.pending_appeals) + 1}",
            content_id=content_id,
            user_id=user_id,
            original_decision=original_decision,
            user_reason=user_reason,
            status=AppealStatus.PENDING,
        )
        
        self.pending_appeals.append(appeal)
        
        # Track user appeal frequency
        self.user_appeal_counts[user_id] = self.user_appeal_counts.get(user_id, 0) + 1
        
        return appeal
    
    def review_appeal(self, appeal_id: str, moderator_id: str,
                      uphold: bool, reason: str) -> Appeal:
        """
        Review an appeal and either uphold (reverse) or reject (confirm).
        
        The moderator reviewing the appeal MUST be different from the
        moderator who made the original decision (if human-reviewed).
        
        Upheld appeals are critical learning signals:
        - The automated system made a false positive
        - This content should never have been flagged
        - The classifier needs retraining or threshold adjustment
        """
        appeal = self._find_appeal(appeal_id)
        if not appeal:
            raise ValueError(f"Appeal {appeal_id} not found")
        
        if uphold:
            appeal.status = AppealStatus.UPHELD
            # Store in double jeopardy protection
            key = f"{appeal.user_id}:{appeal.content_id}"
            self.double_jeopardy_protection.add(key)
        else:
            appeal.status = AppealStatus.REJECTED
        
        appeal.resolved_at = datetime.now()
        appeal.moderator_id = moderator_id
        appeal.resolution_reason = reason
        
        # Move to resolved
        self.pending_appeals.remove(appeal)
        self.resolved_appeals.append(appeal)
        
        return appeal
    
    def _find_appeal(self, appeal_id: str) -> Optional[Appeal]:
        for appeal in self.pending_appeals:
            if appeal.appeal_id == appeal_id:
                return appeal
        for appeal in self.resolved_appeals:
            if appeal.appeal_id == appeal_id:
                return appeal
        return None
    
    def get_stats(self) -> dict:
        """Get appeal system metrics."""
        total = len(self.resolved_appeals)
        upheld = sum(1 for a in self.resolved_appeals if a.status == AppealStatus.UPHELD)
        
        return {
            "total_appeals": total + len(self.pending_appeals),
            "pending": len(self.pending_appeals),
            "resolved": total,
            "upheld_rate": upheld / total if total > 0 else 0,
            "rejected_rate": (total - upheld) / total if total > 0 else 0,
            "avg_resolution_time_hours": self._avg_resolution_time(),
        }
    
    def _avg_resolution_time(self) -> float:
        if not self.resolved_appeals:
            return 0
        total_time = sum(
            (a.resolved_at - a.created_at).total_seconds()
            for a in self.resolved_appeals
            if a.resolved_at
        )
        return (total_time / len(self.resolved_appeals)) / 3600
```

---

## ✅ Good Output Examples

### What a Well-Moderated System Looks Like

**Example 1: Self-Harm Content with Immediate Response**
```
User: "I don't want to be here anymore. Nothing matters."
Pipeline: Tier 1 pattern "want to die" → Category: SELF_HARM (0.92)
Crisis:  → Automated response shown immediately
         "If you're having thoughts of suicide, please reach out for help.
          You're not alone. Crisis resources: [988 Suicide & Crisis Lifeline]"
Queue:   → Routed to CRITICAL priority (SLA: 2 minutes)
         → Human moderator reviews within 2 minutes
         → Determines if follow-up or escalation is needed
Stats:   → Latency: 8ms (Tier 1 catch + crisis response)
         → Cost: ~$0.000001 (regex match only)
         → Outcome: User sees help resources within milliseconds
```

**Example 2: Nuanced Hate Speech with Correct Classification**
```
User: "This new policy is discriminatory and I hate it. My business will suffer."
Pipeline: Tier 1 → No match (keyword only)
          Tier 2 → No confident classification
          Tier 3 → LLM classifies: SAFE (0.85)
                    Reasoning: User is expressing opposition to a policy
                    This is political/feedback speech, not hate speech
Action:   → ALLOW message → Visible to others
Stats:    → Latency: ~1.2 seconds (went through all 3 tiers)
          → Cost: ~$0.02 (Tier 3 LLM call)
          → Outcome: Legitimate criticism is protected (not censored)
```

**Example 3: Adversarial Evasion with Normalization**
```
User: "th1s gr0up 0f p30pl3 4r3 scr3w1ng up 0ur c0mmun1ty"
Pipeline: Normalization → "this group of people are screwing up our community"
          Tier 1 → Keyword match for hate speech pattern (0.85)
Action:   → BLOCK (Tier 1, confidence 0.85)
          → Flagged for human review (HIGH priority)
Stats:    → Latency: 3ms (Tier 1 after normalization)
          → Cost: Negligible (regex + normalization)
          → Outcome: Evasion attempt caught by normalization layer
```

**Example 4: False Positive Recovery via Appeal**
```
User posts: "I hate cancer. It took my mother from me."
Pipeline: Tier 1 → Keyword match "hate" → Category: HATE (0.50)
          → Auto-action threshold not met → FLAG for review
User sees: Message hidden pending review
User appeals: "This is about CANCER, not about hating people!"
Appeal: → Moderator reviews, sees context
        → UPHELD (message should be visible)
        → Double jeopardy protection stored
        → Model feedback: "Add 'cancer' exception to hate speech patterns"
Stats: → Appeal resolved in 15 minutes
       → Model improved based on feedback
       → User: positive experience with appeal process
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Treating All Categories the Same

```python
# BAD: Flat moderation
def moderate(message):
    result = moderation_api.classify(message)
    if result.is_flagged:
        return BLOCK
    return ALLOW

# This blocks 100% of flagged content regardless of:
# - What category it is
# - How confident the classifier is
# - Whether it's a genuine crisis
# - Whether the user needs immediate help
# - Whether it's the first offense or 100th

# GOOD: Category-aware routing
def moderate(message):
    result = moderation_api.classify(message)
    if result.category == "self_harm" and result.confidence > 0.6:
        return CRISIS_RESPONSE  # Immediate help, not block
    if result.category == "spam" and result.confidence > 0.7:
        return AUTO_REMOVE  # Low impact if wrong
    if result.category == "hate" and result.confidence > 0.9:
        return BLOCK  # High bar before censoring
    if 0.5 < result.confidence < 0.9:
        return FLAG_FOR_REVIEW  # Uncertain → human decides
    return ALLOW
```

### Antipattern 2: No Escape Valve (Appeals)

```python
# BAD: No appeal path
def moderate(message):
    result = classifier.classify(message)
    if result.flagged:
        message.hide()  # User can NEVER get this reversed
    # No appeal, no recourse, no second review
    
    # Outcome: Legitimate content is hidden forever
    # User: frustrated, leaves platform
    # Platform: loses user + gets bad reputation

# GOOD: Always provide an escape valve
def moderate(message):
    result = classifier.classify(message)
    if result.flagged:
        message.hide_with_appeal_option()  
        # User can appeal
        # Moderator reviews
        # If upheld → message becomes visible
        # Decision is final (double jeopardy)
```

### Antipattern 3: Relying Only on Content (No User Signals)

```python
# BAD: Content-only moderation
def moderate(message):
    return check_content_only(message)
    # Never considers:
    # - Is this user a new account? (likely troll)
    # - Has this user posted similar content before? (pattern)
    # - Is this user verified / trusted? (lower scrutiny)
    # - Is this from a coordinated group? (network effect)
    
    # Attackers can:
    # - Create 100 throwaway accounts to spam hate
    # - Each account posts once → looks like 100 independent users
    # - Content-only moderation misses the coordination

# GOOD: Content + user signals
def moderate(message, user):
    content_result = check_content(message)
    
    # User-level signals
    user_trust_score = calculate_trust_score(user)
    account_age_hours = (now - user.created_at).total_seconds() / 3600
    recent_violations = count_recent_violations(user, hours=24)
    
    if content_result.flagged and user_trust_score < 0.3:
        return LOW_THRESHOLD_BLOCK  # New/low-trust user → stricter
    if content_result.flagged and user_trust_score > 0.9:
        return FLAG_FOR_REVIEW  # Trusted user → more lenient
    if content_result.flagged and recent_violations > 5:
        return AUTO_BLOCK  # Repeat violator → immediate action
```

### Antipattern 4: No Recourse for False Positive (Sandbagging)

```python
# BAD: Auto-block without review for MEDIUM confidence
def moderate(message):
    result = classifier.classify(message)
    if result.confidence > 0.5:  # Too low for auto-block!
        return BLOCK
    
    # At 0.5 confidence, 50% of flagged content is WRONG
    # That's an unacceptable false positive rate
    # 1 in 2 blocked messages should NOT have been blocked

# GOOD: Conservative thresholds + human review
def moderate(message):
    result = classifier.classify(message)
    if result.confidence > 0.95:
        return BLOCK_WITH_AUDIT  # High confidence, still log for audit
    if result.confidence > 0.7:
        return FLAG_FOR_REVIEW  # Medium confidence → human decides
    # Below 0.7 → allow (with reduced visibility if close to threshold)
    return ALLOW
```

### Common Failure Modes:

| Failure Mode | Symptom | Root Cause | Fix |
|---|---|---|---|
| **Censorship backlash** | Users complain about blocked content | Overly aggressive Tier 1 patterns | Raise thresholds, add appeal path |
| **Moderation burnout** | Moderators quit, quality drops | High volume of traumatic content | Better Tier 1/2 pre-filtering, rotating assignments, wellness support |
| **Adversarial arms race** | Evasion techniques constantly evolving | Pattern-based detection is brittle | Shift to behavioral signals, user-level risk scoring, network analysis |
| **Appeal spam** | Users appeal everything | Too easy to appeal, no cost | Track appeal-to-content ratio, prioritize appeals from high-quality users |
| **Regulatory risk** | Govt fines for missed illegal content | Category definitions don't match legal requirements | Map categories to legal frameworks (GDPR, DSA, etc.) |
| **Bias in moderation** | Certain groups disproportionately flagged | Training data bias, cultural blind spots | Diverse training data, regular bias audits, community input on policies |

---

## 🧪 Drills & Challenges

### Drill 1: Fix the Broken Moderator (30 min)

```python
# BROKEN MODERATOR — find and fix the bugs
# There are 5 distinct bugs

class BrokenModerator:
    """
    This moderation system has 5 bugs. Find and fix them.
    
    Context: This is for a gaming platform with 50M MAU.
    Categories: hate, harassment, spam, self-harm, safe
    
    Budget: 50ms per message max, 10K messages/min peak
    """
    
    def __init__(self):
        self.blacklist = ["badword1", "badword2", "kill myself", "hate"]
        self.spam_patterns = [r'buy now', r'click here', r'http://']
    
    def moderate(self, message: str, user: dict) -> str:
        """
        Returns: "allow", "block", "flag"
        """
        # Bug 1: No normalization
        # "k1ll myself" bypasses the blacklist
        # "h4te" bypasses the blacklist
        
        # Bug 2: Case-sensitive blacklist
        # "Kill Myself" bypasses the blacklist
        # "HATE" bypasses the blacklist
        if message in self.blacklist:
            return "block"
        
        # Bug 3: URL detection doesn't account for legit URLs
        # "Check http://python.org for docs" → flagged as spam
        for pattern in self.spam_patterns:
            if re.search(pattern, message.lower()):
                return "block"
        
        # Bug 4: No distinction between categories
        # Self-harm ("I want to kill myself") gets same treatment as spam
        # Self-harm needs crisis response, spam needs removal
        
        # Bug 5: No user context
        # A trusted user with 5-year history gets same scrutiny as a new account
        # Repeat violators don't get escalated consequences
        
        return "allow"

# 🔍 Your task:
# 1. Add a normalization layer
# 2. Fix case sensitivity
# 3. Distinguish spam URLs from legitimate URLs
# 4. Add category-specific routing (self-harm → crisis, spam → remove)
# 5. Add user context (trust score, violation history)
```

**Expected output:** The fixed version should:
- Catch adversarially obfuscated hate speech
- Allow legitimate URLs
- Show crisis resources for self-harm content
- Apply stricter standards to new/low-trust accounts
- Not punish long-time users for one mistake

---

### Drill 2: Design an Escalation Matrix (45 min)

You're building moderation for a **healthcare AI platform** that provides medical information. Users can ask questions about symptoms, medications, treatments, and health conditions.

Design an escalation matrix for these categories:

| Category | Example | Auto-Action? | Threshold | Human Review? | SLA | Notes |
|---|---|---|---|---|---|---|
| **Medical misinformation** | "Vaccines cause autism" | ??? | ??? | ??? | ??? | Could cause real health harm |
| **Self-harm** | "I want to stop my medication" | ??? | ??? | ??? | ??? | May be genuine medical issue or crisis |
| **Personal health data** | "My HIV status is positive" | ??? | ??? | ??? | ??? | PII but also medical context |
| **Drug-seeking** | "I need Oxycontin for my pain" | ??? | ??? | ??? | ??? | Drug abuse vs. legitimate pain |
| **Product complaint** | "This drug gave me side effects" | ??? | ??? | ??? | ??? | Important safety signal |

**🤔 Questions to answer:**

1. For **medical misinformation**, the auto-action threshold should be HIGH (don't block legitimate debate) or LOW (block potential harm quickly)? What's the cost of a false positive vs. false negative here?

2. For **self-harm about medication**, how do you distinguish "I'm going to stop my meds" (crisis requiring intervention) from "I want to stop my meds because of side effects" (normal medical question that should be answered)?

3. For **personal health data**, the user is VOLUNTARILY sharing their health info. Should you block it (privacy) or allow it (they chose to share)? What if they didn't realize the AI platform logs everything?

4. The **drug-seeking** category needs special care because legitimate pain patients are already underserved and frequently disbelieved. How does your escalation matrix avoid further harming legitimate patients while catching actual drug seekers?

**Your deliverable:** Complete the escalation matrix table with your decisions and write a paragraph explaining the design philosophy for each category.

---

### Drill 3: Build a Normalization Layer (30 min)

Write a text normalization function that handles:

1. **Leetspeak**: "h4t3" → "hate", "th1s" → "this"
2. **Repeated characters**: "hateeeee" → "hate" (normalize 3+ repeats to 1)
3. **Zero-width characters**: Strip Unicode zero-width joiners/spaces
4. **Homoglyph normalization**: Cyrillic 'а' (U+0430) → Latin 'a' (U+0061)
5. **Space insertion**: "h a t e" → "hate" (single chars separated by spaces)
6. **Word boundary breaking**: "h-at-e" → "hate" (non-alphanumeric removal)

```python
def normalize_for_moderation(text: str) -> str:
    """
    Normalize text to catch evasion attempts while preserving meaning.
    
    Rules:
    1. Lowercase
    2. Leetspeak mapping
    3. Remove 3+ repeated chars
    4. Strip zero-width unicode
    5. Normalize homoglyphs
    6. Remove isolated single chars separated by spaces
    7. Remove non-alphanumeric between word chars
    """
    # Your implementation here
    pass


# Test cases
tests = [
    ("h4t3 sp33ch", "hate speech"),
    ("hateeeee thiiis", "hate this"),
    ("h\u200ba\u200bt\u200be", "hate"),  # Zero-width spaces
    ("оbject" + "ionable", "objectionable"),  # Cyrillic 'о'
    ("h a t e", "hate"),  # Space-separated
    ("h-a-t-e", "hate"),  # Dash-separated
    ("h!a@t#e$", "hate"),  # Symbol-separated
    ("k1ll myself", "kill myself"),
    ("th1s 1s t00 much", "this is too much"),
]
```

---

### Drill 4: Moderation System Design Problem (60 min)

You're the AI engineering lead at **QuickLearn**, an ed-tech platform with 5M monthly active users. Users can:
- Ask questions to an AI tutor
- Chat with other students in course discussion forums
- Submit code for AI review
- Get personalized learning recommendations

**The problem:** A coordinated group is using the AI tutor to generate hate speech, then posting it in discussion forums as screenshots. Your moderation system only checks TEXT content, not images. The screenshots evade detection entirely.

**Your task:** Design a solution that:

1. Catches screenshot-based hate speech
2. Prevents the AI tutor from generating hate speech in the first place (see Files 02-04)
3. Identifies coordinated behavior (same group, multiple accounts)
4. Has a clear escalation path for moderators
5. Stays within budget: $500/month moderation infrastructure cost

**🤔 Questions to answer:**

1. Do you add OCR to the moderation pipeline? This means scanning EVERY image for text — at 5M users, that's a LOT of images. An OCR API call costs ~$0.0015 per image. What's the monthly cost? Is it worth it?

2. Instead of OCR-ing every image, could you detect the PATTERN? If 10 accounts all post screenshot-style images within a 5-minute window with similar metadata (same resolution, same upload tool), is that enough to flag them? What if they vary the metadata?

3. The AI tutor generating hate speech is the ROOT CAUSE. If you fix the input guardrails (File 03) and output guardrails (File 04) on the tutor, the screenshots stop being generated at the source. Is this a better investment than OCR? Or do you need BOTH (prevent generation + catch evasions)?

4. **The hardest question:** The coordinated group has 50 accounts. You ban 50 accounts. They create 50 more. You're playing whack-a-mole. Is there a way to detect and stop coordinated behavior at the ACCOUNT level (not the content level)? What signals would you use?

---

## 🚦 Gate Check

Before moving to File 07 (Guardrails Architecture), verify you can:

1. **Design a moderation taxonomy** — Define at least 8 content categories with severity levels, action mappings, and review requirements for each

2. **Build a tiered moderation pipeline** — Implement a 3-tier system (pattern + ML + LLM) with cost-aware routing. 80%+ should be handled in Tier 1, <5% should need human review

3. **Implement human-in-the-loop workflows** — Design a queue system with category-based priority, SLAs, and moderator assignment. CRITICAL items should have 2-minute SLA, LOW items can wait 24 hours

4. **Handle adversarial evasion** — Implement a normalization layer that catches leetspeak, homoglyphs, zero-width characters, and space-insertion. Show it working against at least 5 evasion techniques

5. **Build an appeal system** — Users must be able to dispute moderation decisions. At least: 30% of legitimate appeals should be upheld (meaning the original automated decision was wrong), double jeopardy protection, appeal frequency tracking

6. **Calculate moderation costs** — Given 10K messages/day, 3-tier pipeline with category routing:
   - 80% handled by Tier 1 (regex, $0.000001/msg)
   - 15% handled by Tier 2 (ML, $0.0005/msg)
   - 4.9% handled by Tier 3 (LLM, $0.03/msg)
   - 0.1% need human review ($0.50/review)
   
   What's the DAILY cost? What's the MONTHLY cost?

7. **Explain the false positive tradeoff** — For 3 different categories (hate speech, self-harm, spam), explain what happens when the moderation system gets it WRONG (both false positive and false negative) and how that affects your threshold settings

---

## 📚 Resources

### Code & Tools
- **[OpenAI Moderation API](https://platform.openai.com/docs/guides/moderation)** — Pre-built content moderation, free to use (covers hate, harassment, violence, self-harm, sexual, illegal)
- **[Perspective API](https://perspectiveapi.com/)** — Google's toxicity classifier (used by many news platforms for comment moderation)
- **[Hugging Face Moderation Models](https://huggingface.co/models?pipeline_tag=text-classification&sort=downloads&search=toxic)** — Open-source toxicity classifiers (fine-tune on your data)
- **[Llama Guard](https://github.com/meta-llama/PurpleLlama/tree/main/LlamaGuard)** — Meta's safety classifier for LLM inputs/outputs, supports custom policies

### Design Patterns
- **Tiered Moderation Architecture** — Used by Discord, Reddit, Twitch, Facebook. Read: [Toxicity Moderation at Scale](https://discord.com/blog/how-discord-detects-toxic-content)
- **Human-in-the-Loop at Scale** — OpenAI's approach to content policy enforcement. Read: [OpenAI's Approach to Content Moderation](https://openai.com/index/our-approach-to-content-moderation/)
- **Appeal System Design** — How platforms handle moderation disputes. Read: [Reddit's Moderation Appeal Process](https://support.reddithelp.com/hc/en-us/articles/360058237711-What-is-the-Appeal-a-Mod-Decision-process)

### Policy & Legal
- **[EU Digital Services Act (DSA)](https://digital-strategy.ec.europa.eu/en/policies/digital-services-act-package)** — Legal framework for platform content moderation in EU
- **[Section 230 (US)](https://www.eff.org/issues/cda230)** — US legal protection for platform moderation decisions
- **[Santa Clara Principles](https://santaclaraprinciples.org/)** — Human rights principles for content moderation: transparency, due process, remedy

### Academic & Industry Research
- **[The Content Moderation Crisis](https://knightcolumbia.org/content/the-content-moderation-crisis)** — Understanding the challenges of moderation at scale
- **[Automated Content Moderation: A Survey](https://arxiv.org/abs/2102.05373)** — Academic survey of moderation techniques
- **[Dynamic Risk Scoring for Content Moderation](https://blog.twitch.tv/en/2021/03/04/an-update-on-our-work-to-improve-safety-and-moderation/)** — Twitch's approach to user-level risk signals

### Phase Cross-References
- **Phase 7 (Evals)** → Use your eval framework from 07-Production-Evals-Observability to measure moderation accuracy: precision, recall, F1 per category, false positive rate, human moderator agreement
- **File 02 (Prompt Injection)** → Injection attacks are a form of adversarial content; your moderation pipeline should integrate with injection detection
- **File 03 (Input Guardrails)** → Input filtering and content moderation share detection strategies; coordinate category taxonomies
- **File 04 (Output Guardrails)** → Output safety categories and content moderation categories should be ALIGNED (they're checking the same concepts, just at different points)
- **File 05 (PII Redaction)** → PII detection is a specialized form of content moderation; your moderation pipeline should call the PII detector for relevant categories
- **File 07 (Guardrails Architecture — Next)** → The tiered moderation pipeline you built here will be a LAYER in the overall guardrails architecture
- **Phase 6 (Agents)** → Agent-generated content needs the SAME moderation as user-generated content; agent loops add complexity (agent generates → agent checks own output)

> **Next up: File 07 — Guardrails Architecture.** You'll combine everything from Files 01-06 into a unified multi-layer defense architecture with phased detection, latency budgets, and system design diagrams.
