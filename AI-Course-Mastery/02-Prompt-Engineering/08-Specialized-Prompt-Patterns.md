# Specialized Prompt Patterns — RAG, Agents, Code, Classification, Extraction

## 🎯 Purpose & Goals

> **🛑 STOP. Three real-world scenarios. Think through each before reading on.**

---

### 🤔 Scenario 1: The RAG That Can't Say "I Don't Know"

You build a RAG system. The prompt is clear: "Answer ONLY using the provided context. If the context doesn't contain the answer, say you don't know."

You test with 50 questions where the context DOES contain the answer. 100% accuracy. Great.

You test with 20 questions where the context does NOT contain the answer. Your prompt says "say you don't know." But the model answers anyway — it makes up plausible answers based on its training data.

**Your analysis shows:**
- When context partially overlaps (e.g., context talks about "returns policy" but user asks about "electronics returns"), the model fills in the gaps from training data 65% of the time
- When context is completely irrelevant, the model says "I don't know" 90% of the time
- When context is slightly relevant but doesn't fully answer, the model hallucinates 70% of the time

**Questions:**
1. Why does "partial relevance" trigger hallucination more than "no relevance"? What's the psychological/token-level mechanism?
2. Your prompt says "Answer ONLY using the provided context" — why doesn't this work for partial relevance cases? Is the prompt wrong, or is this a fundamental LLM limitation?
3. Design a RAG prompt that handles the "partial relevance" case. How would you make the model distinguish between "fully supported," "partially supported," and "unsupported"?
4. **Research task:** Look up "RAG faithfulness" and "citation faithfulness metrics." How do production RAG systems ensure the model only uses provided context?

---

### 🤔 Scenario 2: The Agent That Loops Forever

You build an AI agent with:
- A system prompt defining its goal
- Tools it can call (search, calculator, database)
- A "done" condition where it returns the answer

The agent works on simple tasks. Then you give it: "Find the best price for a flight to Tokyo next month."

The agent:
1. Searches for flights → gets 500 results
2. Tries to filter → needs to know dates → searches for "best time to visit Tokyo"
3. Searches again → gets prices for April
4. But user said "next month" and it's currently May, so April is wrong
5. Searches again → gets June prices
6. Compares → finds cheap option → presents it
7. But didn't check if user prefers window seat
8. Asks user → user doesn't respond immediately
9. Agent waits... and waits... and waits...

**14 minutes later, the agent is still running, has spent $2.37 in API calls, and the user gave up.**

**Questions:**
1. The agent prompt says "Complete the user's request thoroughly." How does "thoroughly" cause infinite loops?
2. What specific constraints would you add to prevent this? Be precise — "be efficient" is too vague.
3. Design a timeout system for agent prompts. What happens at timeout? How does the agent communicate "I couldn't finish"?
4. The agent can't read the human's mind — it doesn't know when "good enough" is enough. How do you encode "satisficing" (good enough) vs. "optimizing" (perfect) in an agent prompt?

---

### 🤔 Scenario 3: The Extraction That Hallucinated Fields

You use this prompt to extract structured data from resumes:

```
Extract the following fields from this resume:
- name
- email
- phone
- years_of_experience
- skills
```

A resume says: "John Doe has 5 years of Python experience." Your system extracts:
```json
{
  "name": "John Doe",
  "years_of_experience": 5,
  "skills": ["Python"]
}
```

Correct! But then another resume: "Jane Smith led engineering teams at Google for 3 years and at Amazon for 4 years." Your system extracts:
```json
{
  "name": "Jane Smith", 
  "years_of_experience": 7,
  "skills": ["engineering", "team leadership", "Google", "Amazon"]
}
```

**The "skills" field contains "Google" and "Amazon" — companies, not skills. The model confused job history with skills because they appeared near each other in the text.**

**Questions:**
1. Why does a clear extraction prompt produce semantically wrong output? The format is right, the tokens are wrong.
2. How would you make skills extraction more precise? What constraints would prevent company names from appearing as skills?
3. If you get JSON validation to pass but semantic validation to fail, where's the gap in your system? What level of validation is missing?
4. Design a post-processing step that catches this type of error. What heuristics would you use to detect "company classified as skill"?

---

**By the end of this file, you will:**
- Design RAG prompts that resist hallucination
- Build agent prompts with proper guardrails
- Create extraction prompts with validation
- Write code generation prompts that produce safe output
- Understand the pattern differences across these specialized domains

**⏱ Time Budget:** 2 hours

---

## 📖 The Specialization Spectrum

```
DIFFERENT TASKS → DIFFERENT PROMPT ARCHITECTURES

RAG:    constraining (narrow the model to specific context)
        "You are a precise reader. Only use provided text."

Agent:  directing (guide action selection and tool use)
        "You have these tools. Choose wisely. Know when to stop."

Code:   structuring (force specific output patterns)
        "Generate complete, safe, typed code. Plan first."

Classify: anchoring (fix output to specific categories)
         "Respond with EXACTLY one of: [A, B, C]"

Extract: isolating (pull specific fields, ignore everything else)
        "Find these fields. Do not invent values."
```

---

## 💻 RAG Prompt Patterns

```python
"""
RAG prompts are the most CRITICAL to get right.
A bad RAG prompt confidently hallucinates. A good one knows its limits.
"""


class RAGPromptBuilder:
    """
    Build prompts for retrieval-augmented generation.
    
    KEY DESIGN PRINCIPLES:
    1. Separate context from instructions (delimit clearly)
    2. Ground answers in specific citations
    3. Handle partial relevance explicitly
    4. Make "I don't know" the default, not an afterthought
    """
    
    @staticmethod
    def standard_rag() -> str:
        """
        Standard RAG prompt with citation.
        
        Forces the model to cite which part of context supports its answer.
        This is the MINIMUM viable RAG prompt.
        """
        return """You are a precise question-answering system.

## Rules
1. Answer ONLY using the information in the provided context below.
2. If the context fully answers the question, provide a concise answer.
3. If the context partially answers, state what IS supported and what is NOT.
4. If the context does NOT answer the question, say:
   "The provided information does not address this question."
5. NEVER use your own knowledge to fill gaps in the context.
6. For each claim in your answer, cite the relevant part of the context in [brackets].

## Context
{context}

## Question
{question}

## Answer
"""
    
    @staticmethod
    def citation_rag() -> str:
        """
        Citation-focused RAG prompt.
        
        Every statement must be traceable to the source.
        This is for HIGH-STAKES applications (medical, legal, financial).
        """
        return """You are a citation-focused analysis system.

CRITICAL INSTRUCTION: Every factual claim in your answer must include
a citation to the line number in the context that supports it.

CONTEXT (line-numbered):
{numbered_context}

USER QUESTION: {question}

FORMAT YOUR ANSWER AS:
Answer: <your answer with [line: X] citations>

If no line supports a claim you want to make, do NOT make that claim.
Instead say: "I cannot verify this from the provided context."

Uncited claims will be flagged as HALLUCINATIONS by our monitoring system.
"""
    
    @staticmethod
    def decompose_rag(top_k: int = 3) -> str:
        """
        Decomposition-based RAG for complex questions.
        
        Breaks down multi-part questions into sub-questions,
        answers each from context, then synthesizes.
        """
        return """You are a multi-step analysis system.

STEP 1: DECOMPOSE
Break the user's question into sub-questions that can each be answered
from the provided context.

STEP 2: ANSWER EACH SUB-QUESTION
For each sub-question, answer using ONLY the context.
Cite which document/paragraph supports each answer.

STEP 3: SYNTHESIZE
Combine your sub-answers into a complete response.
Flag any sub-question that could NOT be answered from context.

TOP {top_k} RELEVANT DOCUMENTS:
{context_docs}

USER QUESTION: {question}

FORMAT:
## Sub-questions
1. <sub-question 1>
2. <sub-question 2>

## Answers
1. <answer with citation>
2. <answer with citation>

## Synthesis
<combined answer>

## Unknowns
<what could not be answered>
"""

    @staticmethod
    def rag_with_verification() -> str:
        """
        RAG with built-in verification step.
        
        After generating an answer, the model checks each claim
        against the context. This catches hallucination before output.
        """
        return f"""You are a RAG system with built-in verification.

STEP 1: Generate a draft answer using only the context below.
STEP 2: For EACH claim in your draft, verify it exists in the context.
STEP 3: If a claim cannot be verified, remove it.
STEP 4: Output the verified answer.

CONTEXT:
{context}

QUESTION: {question}

FORMAT:
<verified_answer>
"""


# ── RAG Failure Mode Analysis ──

RAG_FAILURE_MODES = {
    "lost_in_middle": {
        "symptoms": "Model ignores middle context, uses only first/last docs",
        "fix": "Place most relevant docs FIRST and LAST. Use query re-writing.",
        "detection": "Compare answers with different doc orderings",
    },
    "premature_answer": {
        "symptoms": "Model stops reading context after first few sentences",
        "fix": "Force model to summarize/annotate context before answering",
        "detection": "Add 'useless' info in middle context — does the answer use it?",
    },
    "partial_overlap_hallucination": {
        "symptoms": "Context partially answers, model fills gaps from training",
        "fix": "Explicit 'unsupported' category + citation requirement",
        "detection": "Test with context that's missing specific details",
    },
    "citation_bloat": {
        "symptoms": "Model cites everything, even made-up line numbers",
        "fix": "Number lines explicitly. Verify citations exist post-hoc.",
        "detection": "Random sample: check cited line numbers actually exist",
    },
}
```

---

## 💻 Agent Prompt Patterns

```python
"""
Agent prompts are fundamentally different from RAG prompts.
Instead of CONSTRAINING, they need to DIRECT — give the model
agency while keeping it safe.
"""


class AgentPromptBuilder:
    """
    Build prompts for AI agents.
    
    THE AGENT PROMPT CHALLENGE:
    The agent must be:
    1. Proactive (take initiative when appropriate)
    2. Constrained (not go on infinite tangents)
    3. Tool-aware (know which tool to use when)
    4. Termination-aware (know when to stop)
    """
    
    @staticmethod
    def basic_agent(
        tools: list[dict],
        max_steps: int = 10,
    ) -> str:
        """
        Basic ReAct agent prompt.
        
        Uses the Thought → Action → Observation loop:
        1. Think: What do I need to do next?
        2. Action: Which tool to use and with what parameters
        3. Observation: What did the tool return?
        4. Repeat or Answer
        """
        tool_descriptions = "\n".join(
            f"  {t['name']}: {t['description']} — Args: {t.get('args', 'none')}"
            for t in tools
        )
        
        return f"""You are an AI agent with access to tools.

## Your Goal
Help the user accomplish their task. Use tools when you need 
information or actions beyond your knowledge.

## Available Tools
{tool_descriptions}

## Rules
1. ALWAYS use this format for each step:
   Thought: <what you're trying to accomplish>
   Action: <tool_name>(<args>)
   Observation: <result>

2. After observing the result, either:
   - Take another action (if you need more information)
   - Provide a FINAL ANSWER (if you have enough information)

3. You have MAXIMUM {max_steps} actions. Plan carefully.
   If you can't complete the task in {max_steps} steps, provide your
   best partial answer and explain what's missing.

4. If a tool errors or returns no results, try an alternative approach.
   If ALL approaches fail, explain what went wrong to the user.

## User Task
{{task}}

Begin!"""
    
    @staticmethod
    def stopping_rules() -> str:
        """
        Critically important: rules that prevent infinite loops.
        
        These are often MISSING from agent prompts.
        Adding them is the difference between an agent that works
        and one that burns $50 in API calls overnight.
        """
        return """## STOPPING RULES (These are mandatory)

1. If you've taken 3 actions without making progress
   → STOP. Report current state. Ask user for guidance.

2. If a tool returns the SAME error twice
   → STOP trying that approach. Try something different.
   If nothing works, explain the limitation to the user.

3. If the user's request is ambiguous
   → Do NOT guess. Ask ONE clarifying question.
   Wait for their response before proceeding.

4. Once you have enough information to answer
   → STOP taking actions. Provide your answer immediately.

5. If you detect the task cannot be completed
   → STOP trying. Explain WHY it can't be done.
   Suggest alternatives if possible.

REMEMBER: More actions = more cost and latency for the user.
Be efficient. Every action should bring you closer to the goal.
"""
    
    @staticmethod
    def tool_definition(name: str, description: str, args: dict) -> dict:
        """
        Standard tool definition format.
        
        Each tool needs:
        - name: Short, unique identifier
        - description: When to use this tool
        - args: What parameters the tool needs
        """
        return {
            "name": name,
            "description": description,
            "args": args,
        }


# ── Agent System Prompt: Complete Example ──

AGENT_SYSTEM_PROMPT = """You are a research assistant that can browse the web
and analyze documents to answer user questions.

AVAILABLE TOOLS:
1. web_search(query): Search the web for current information
2. read_url(url): Read and summarize a web page
3. analyze_text(text, instruction): Analyze text according to instruction
4. save_to_file(filename, content): Save research to a file

RULES:
1. For factual questions, ALWAYS verify with web_search before answering
2. Cite your sources (URL) in the final answer
3. If information is contradictory, present both sides
4. Be concise — answer the question directly, don't ramble
5. If you cannot find reliable information, say so

TERMINATION:
- Once you have a verified answer with sources → ANSWER
- If you can't find the answer after 3 searches → Report what you found
  and what you couldn't find
"""


# ── Agent Failure Mode Analysis ──

AGENT_FAILURE_MODES = {
    "tool_obsession": {
        "symptoms": "Model uses tools even when it knows the answer",
        "fix": "Add 'Answer directly if you know' to instructions",
        "example": "Model searches web for 'what is 2+2' instead of answering",
    },
    "loop_without_progress": {
        "symptoms": "Same action repeated with slight variations",
        "fix": "Implement progress detection (same action count threshold)",
        "example": "Search → too many results → search with slightly different terms → repeat",
    },
    "premature_termination": {
        "symptoms": "Model gives up too early, claims task impossible",
        "fix": "Provide 'retry with different approach' guidance",
        "example": "One search fails → 'I cannot find this information'",
    },
    "tool_misuse": {
        "symptoms": "Tool called with wrong arguments",
        "fix": "Better tool descriptions with clear 'when to use' guidance",
        "example": "Using 'save_to_file' instead of 'read_url'",
    },
}
```

---

## 💻 Code Generation Prompts

```python
"""
Code generation is a SPECIAL case because:
1. The output has strict syntax requirements
2. Errors are immediately visible (compilation fails)
3. Security vulnerabilities are invisible (but dangerous)
4. Models are trained on code and have strong priors

KEY INSIGHT: For code generation, the FORMAT of the request matters
more than the INSTRUCTION. Well-structured code prompts produce
better code than verbose instructions.
"""


class CodePromptBuilder:
    """
    Build prompts for code generation.
    
    Principles:
    1. Specify inputs, outputs, constraints (like a function signature)
    2. Require planning before coding
    3. Always include error handling requirements
    4. Specify security concerns explicitly
    """
    
    @staticmethod
    def function_generation(
        func_name: str,
        params: list[dict],
        return_type: str,
        constraints: list[str],
        examples: list[tuple[str, str]],
    ) -> str:
        """
        Generate a single function with full specifications.
        """
        params_str = "\n".join(f"  - {p['name']} ({p['type']}): {p['desc']}" for p in params)
        constraints_str = "\n".join(f"  - {c}" for c in constraints)
        examples_str = "\n".join(
            f"  Input: {inp} → Output: {out}" for inp, out in examples
        )
        
        return f"""Write a function `{func_name}`.

PARAMETERS:
{params_str}

RETURNS: {return_type}

CONSTRAINTS:
{constraints_str}

EXAMPLES:
{examples_str}

REQUIREMENTS:
1. Include complete type hints for all parameters and return type
2. Handle edge cases: empty inputs, None values, invalid types
3. Add a docstring explaining the function's purpose
4. Include input validation at the start of the function
5. Write test cases that verify the function works

Write ONLY the function implementation and tests. No explanations."""

    @staticmethod
    def review_prompt(code: str, focus_areas: list[str]) -> str:
        """
        Code review prompt focused on specific areas.
        """
        areas = "\n".join(f"- {area}" for area in focus_areas)
        return f"""Review this code for issues:

```python
{code}
```

FOCUS ON:
{areas}

For each issue found, report:
1. The exact line number
2. What the problem is
3. How to fix it
4. Severity (critical/major/minor)

If no issues found, state: "Code review passed — no issues detected.""""

    @staticmethod
    def security_review(code: str) -> str:
        """
        Security-focused code review.
        """
        return f"""SECURITY REVIEW

Review this code for security vulnerabilities:

```python
{code}
```

CHECK FOR:
1. SQL injection (string concatenation in queries)
2. Command injection (shell=True, os.system, eval)
3. Path traversal (unsanitized file paths)
4. Insecure deserialization (pickle, yaml.load)
5. Hardcoded secrets (API keys, passwords)
6. XXE (XML processing without disabling external entities)
7. SSRF (making requests to user-supplied URLs)
8. Insecure direct object references (IDOR)

For each vulnerability found:
- Line number
- Vulnerability type
- Risk level
- Fix recommendation

If no vulnerabilities found, state: "Security review passed.""""
```

---

## 💻 Classification Prompts

```python
"""
Classification prompts need to be PRECISE AND CONSISTENT.
The goal is not creativity — it's reliability.
"""


class ClassificationPromptBuilder:
    """
    Build prompts for classification tasks.
    
    Key design:
    - Define categories explicitly (with descriptions)
    - Include a "not sure" option (prevents forced misclassification)
    - Use strict format control (only output the category)
    - Handle multi-label vs single-label explicitly
    """
    
    @staticmethod
    def single_label(
        categories: dict[str, str],
        input_text: str,
        include_unsure: bool = True,
    ) -> str:
        """
        Single-label classification prompt.
        
        Args:
            categories: {"category_name": "description of when to use this"}
            input_text: The text to classify
            include_unsure: Whether to include an "unsure" option
        """
        cat_list = "\n".join(f"  - {name}: {desc}" for name, desc in categories.items())
        
        if include_unsure:
            cat_list += "\n  - UNSURE: Cannot confidently determine category"
        
        return f"""Classify the following text into EXACTLY ONE category.

CATEGORIES:
{cat_list}

TEXT: {input_text}

RULES:
- Respond with ONLY the category name. No explanations. No punctuation.
- If multiple categories apply, choose the PRIMARY one.
- If you cannot confidently classify, respond with "UNSURE".

CATEGORY:"""

    @staticmethod
    def multi_label(
        categories: dict[str, str],
        input_text: str,
        max_labels: int = 3,
    ) -> str:
        """
        Multi-label classification.
        
        A text can belong to multiple categories.
        """
        cat_list = "\n".join(f"  {i+1}. {name}: {desc}" for i, (name, desc) in enumerate(categories.items()))
        
        return f"""Classify the following text. It may belong to MULTIPLE categories.

CATEGORIES:
{cat_list}

TEXT: {input_text}

INSTRUCTIONS:
1. Select ALL categories that apply (0 to {max_labels} categories)
2. Respond only as a comma-separated list: "cat1, cat2"
3. If no category fits, respond with: "NONE"

CATEGORIES:"""

    @staticmethod
    def confidence_scored(
        categories: dict[str, str],
        input_text: str,
    ) -> str:
        """
        Classification with confidence scoring.
        
        Useful when you need to know HOW SURE the model is.
        """
        cat_list = "\n".join(f"  {name}: {desc}" for name, desc in categories.items())
        
        return f"""Classify AND provide confidence.

CATEGORIES:
{cat_list}

TEXT: {input_text}

Respond ONLY in this exact JSON format:
{{"category": "<category_name>", "confidence": <0.0-1.0>, "reasoning": "<one sentence>"}}

Examples:
{{"category": "complaint", "confidence": 0.85, "reasoning": "Strong negative language about poor service experience"}}
{{"category": "inquiry", "confidence": 0.95, "reasoning": "Direct question about product availability with neutral tone"}}
{{"category": "UNSURE", "confidence": 0.3, "reasoning": "Text is too short/ambiguous to classify confidently"}}

OUTPUT:"""
```

---

### 🤔 Application Question: The RAG-Agent Hybrid

Your system needs to BOTH retrieve information (RAG) AND take actions (agent). A user asks: "Find my last order and process a return."

The RAG part needs to find the order in the database.
The Agent part needs to actually PROCESS the return.
The prompt needs to handle BOTH — but RAG prompts are constraining and Agent prompts are directive. They're fundamentally different.

**Questions:**
1. Would you use ONE prompt for both tasks, or separate prompts? What are the tradeoffs?
2. If you use separate prompts, how does the system decide which mode to use? Does the model decide, or does your code route based on intent?
3. Design a prompt structure that handles BOTH retrieval and action. Where does the "constraint" mindset end and the "directive" mindset begin?
4. What happens if the model finds the order (RAG success) but can't process the return (agent failure)? How does your prompt handle partial completion?

---

## 💻 Extraction Prompts

```python
"""
Extraction prompts need to be AGNOSTIC to format and PRECISE about fields.
The model should extract only what's there — never invent.
"""


class ExtractionPromptBuilder:
    """
    Build prompts for data extraction.
    
    Challenges:
    - Model invents values for missing fields
    - Model confuses similar fields (company → skill)
    - Format is usually correct but content is wrong
    """
    
    @staticmethod
    def field_extraction(
        fields: dict[str, str],
        input_text: str,
        strict: bool = True,
    ) -> str:
        """
        Extract specific fields from text.
        
        Args:
            fields: {"field_name": "description of what to extract"}
            input_text: Text to extract from
            strict: If True, require exact field match
        """
        field_list = "\n".join(f'  "{name}": "{desc}"' for name, desc in fields.items())
        
        strict_rule = (
            "STRICT MODE: Every field must be EXACTLY as found in the text. "
            "Do NOT paraphrase, summarize, or infer. Extract verbatim where possible."
        ) if strict else ""
        
        return f"""Extract the specified fields from the text.

FIELDS TO EXTRACT:
{field_list}

TEXT:
{input_text}

RULES:
- Extract values EXACTLY as they appear in the text
- If a field is NOT present in the text, use null (do NOT invent values)
- Do NOT reformat or normalize values
- {strict_rule}

Respond in valid JSON:
{{"""

    @staticmethod
    def strict_json_extraction(schema: dict) -> str:
        """
        Structured extraction with JSON Schema validation hints.
        
        Compatible with OpenAI's response_format parameter.
        """
        schema_str = json.dumps(schema, indent=2)
        return f"""Extract the requested information from the text.

JSON SCHEMA:
{schema_str}

RULES:
1. Extract ONLY information present in the input text
2. Use null for any field not found in the text
3. Do NOT infer, guess, or fill in missing values
4. Values must match the specified types in the schema
5. If a field asks for a list and nothing is found, return an empty list []

INPUT TEXT:
{{input}}

OUTPUT:"""

    @staticmethod
    def extraction_with_examples(
        fields: dict[str, str],
        examples: list[tuple[str, dict]],
    ) -> str:
        """
        Extraction with few-shot examples.
        
        Examples help the model understand:
        - How to handle missing fields
        - What counts as a valid extraction
        - Edge cases in interpretation
        """
        field_list = "\n".join(f'  "{name}": "{desc}"' for name, desc in fields.items())
        
        examples_str = "\n".join(
            f"Input: {inp}\nOutput: {json.dumps(out)}"
            for inp, out in examples
        )
        
        return f"""Extract fields from text. Examples show the expected format.

FIELDS:
{field_list}

EXAMPLES:
{examples_str}

RULES:
- Extract EXACT values found in text
- Use null for missing fields
- Do NOT invent or infer values

Input: {{input}}
Output:"""
```

---

## ✅ Good Output Examples

### RAG: Good vs Bad

```
BAD RAG OUTPUT (confident but wrong):
Q: "What is the return policy for electronics?"
Context: "Our general return policy allows returns within 30 days."
A: "Electronics can be returned within 30 days, but opened items 
    may be subject to a restocking fee of 15%."

Problem: Context didn't mention restocking fees. Model invented it.

GOOD RAG OUTPUT (faithful to context):
Q: "What is the return policy for electronics?"
Context: "Our general return policy allows returns within 30 days."
A: "Based on the provided information, our return policy allows returns 
    within 30 days. The available context doesn't specify any special 
    conditions for electronics or restocking fees [uncited claim excluded]."
```

### Agent: Good vs Bad

```
BAD AGENT BEHAVIOR:
Action: web_search("best restaurants in Tokyo") → 50 results
Action: web_search("Tokyo restaurants Michelin") → 20 results  
Action: web_search("best ramen Tokyo") → 30 results
Action: web_search("Tokyo restaurant reviews") → 40 results
...continues forever...

GOOD AGENT BEHAVIOR:
Action: web_search("best restaurants in Tokyo") → 50 results
Thought: I have enough results. Let me summarize the top recommendations.
Final Answer: Based on search results, the top-rated restaurants in Tokyo are...
```

---

## 📊 Pattern Selection Guide

```
TASK TYPE      | RECOMMENDED PATTERN          | KEY CONSTRAINT
---------------|------------------------------|--------------------------------------
QA from docs   | Citation RAG                  | "Cite line numbers for every claim"
Complex QA     | Decomposition RAG             | "Break question into sub-questions"
Multi-step     | Agent with stopping rules     | "Max N actions, detect loops"
Single tool    | Basic ReAct                   | "Thought → Action → Observation"
Code gen       | Spec-first generation         | "Type hints, error handling, tests"
Classification | Strict single-label           | "Only output the category name"
Multi-label    | Multi-label with max          | "Select all that apply (max N)"
Extraction     | Schema-driven strict          | "null for missing, exact values"
```

---

## 🧪 Drills

### Drill 1: RAG Faithfulness Audit

Take your gateway from Phase 1 and test its RAG capabilities:
1. Create 20 test cases where context partially answers the question
2. Run each through your current RAG prompt
3. Count how many times the model hallucinates (adds info not in context)
4. Fix your prompt using the patterns in this file
5. Re-test — measure the improvement

### Drill 2: Agent Stopping Challenge

Build an agent prompt with these tools: `web_search`, `calculator`, `read_database`. Create a task that's impossible to complete (e.g., "Find the population of Mars in 1800"). 

Does your agent:
- Loop forever trying different searches?
- Give up appropriately?
- Explain WHY it can't answer?

Iterate until the agent handles impossible tasks gracefully.

### Drill 3: Extraction Boundary Detection

Take 20 product descriptions. Create an extraction prompt that pulls: `product_name`, `price`, `color`, `size`, `material`.

Then INJECT adversarial text:
- "The size is not listed but is typically medium"
- "Price: free (no, just kidding, it's $29.99)"
- "Material: cotton (also available in polyester)"

Does your extraction handle these edge cases? Improve it until it does.

### Drill 4: Hybrid RAG-Agent

Design and implement a prompt that can BOTH answer from documents AND take actions. The task: "Using the documentation, find the API endpoint for user creation, then test it with a sample request."

Your prompt needs to:
1. Retrieve the endpoint info from docs (RAG mode)
2. Construct and explain the API call (Action mode)
3. Never confuse the two modes

---

## 🚦 Gate Check

- [ ] You can explain the difference between RAG, agent, and classification prompt architectures
- [ ] Your RAG prompt passes the "partial context" test (doesn't hallucinate)
- [ ] Your agent prompt includes proper stopping rules
- [ ] Your extraction prompt handles missing fields correctly (returns null)
- [ ] Your classification prompt can say "I don't know" instead of guessing
- [ ] You understand which pattern to use for which task without guessing
- [ ] **You thought through all 3 scenarios at the top of this file**

---

## 📚 Cross-References

- [Context Engineering](01-Context-Engineering.md) — Why patterns work differently
- [Techniques Deep Dive](02-Techniques-Deep-Dive.md) — CoT for agent reasoning
- [System Prompts](03-System-Prompts-Personas.md) — Building the base prompts
- [Red-Teaming](05-Red-Teaming-Antipatterns.md) — Testing specialized prompts
- [Phase 4: RAG Foundations](../04-RAG-Foundations/) — Deep RAG prompt optimization
- [Phase 6: AI Agents](../06-AI-Agents/) — Advanced agent prompt design
