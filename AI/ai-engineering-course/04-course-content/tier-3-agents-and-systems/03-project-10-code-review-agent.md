# Project 10: GitHub Code Review Agent

- **Tier:** 3 — Real Constraints
- **Project #:** 10 of 12
- **Tech Stack:** LangGraph (create_react_agent), GitHub API (via PyGithub/httpx), AST parsing (ast module), Langfuse tracing
- **Concepts:** Tool use (GitHub API), structured code analysis, agent evaluation with precision/recall, false positive minimization
- **Quality Gate:** ✅ APPROVED when false positive rate < 10% on a curated test set of 50 PRs

---

## Phase 1: Brief (Priya)

> *Priya: "This one's for another dev team. They're drowning in PR reviews. The agent doesn't write code — it reviews it."*

**Client:** **CodeBridge** — a SaaS company with 40 engineers shipping 200+ PRs/week. Their senior engineers spend 3+ hours/day reviewing code.

**Their problem:** Most PR reviews catch the same types of issues: style violations, missing error handling, security anti-patterns, logic errors that a linter won't catch. Seniors are bored reviewing the same patterns. Juniors miss issues because they don't know what to look for.

**What we're building:** An AI agent that watches GitHub PRs and automatically reviews them. The agent:
1. Fetches the PR diff
2. Analyzes changed files for common issues (error handling, null safety, security, style, logic)
3. Posts inline comments on specific lines
4. Provides an overall summary: ✅ Looks good / 🔄 Minor issues / ❌ Major issues found

**Non-negotiable:**
- Must NOT produce false positives on code that follows the team's conventions. False alarms destroy trust
- Must understand the language context (Python/TypeScript primarily). Different languages have different pitfalls
- Must integrate with GitHub via the API — inline PR comments, not a separate dashboard
- Must be faster than a human reviewer — under 30 seconds for a typical PR (10 files, 200 lines changed)

**Deadline:** 10 days. They're scaling the team and PR review bottlenecks are getting worse.

**Definition of done:** Install the GitHub app on a test repo → open a PR → agent posts inline comments within 30 seconds. Test set of 50 PRs: false positive rate < 10%.

---

## Phase 2: Learning Path (Maya)

> *Maya: "This project teaches you two things: (1) how to design tools that interact with external systems (GitHub API), and (2) how to evaluate an agent when 'correctness' is subjective."*

### Why This Is Hard

"Three unique challenges:

1. **False positives are worse than false negatives.** If the agent misses a bug, the human reviewer might catch it. If the agent flags CORRECT code as wrong, the human has to waste time investigating — and they stop trusting the agent.

2. **Code analysis requires context.** A variable named `data` might be fine in one file and terrible in another. A try/except that swallows an exception might be intentional or might be a bug. The agent needs FILE-LEVEL context, not just line-level.

3. **Evaluation is subjective.** What counts as a 'valid' review comment? Two senior engineers might disagree. You need a test set with curated ground truth and a clear definition of false positive.

### Learning Order (Scratch-First)

**Step 1: Build the diff parser.**
GitHub PRs come as diffs — `@@ -line,col +line,col @@` format. Parse this into structured file changes. Understand what changed, not just the raw text.

**Step 2: AST parsing (Python).**
Use the `ast` module to parse Python files. Find functions, classes, try/except blocks, variable assignments. This gives you STRUCTURAL understanding beyond raw text.

**Step 3: Single-file review (no LLM).**
Write rule-based checkers for common patterns:
- Bare `except:` clauses (no exception type)
- Hardcoded secrets/API keys
- Functions without docstrings
- Too many arguments (>5)
These rule-based checks are FAST and DETERMINISTIC — zero false positives for well-defined rules.

**Step 4: LLM-powered review.**
Add an LLM that reviews the diff with file-level context. This catches logic issues that rules can't. But this is where false positives come from — the LLM invents issues.

**Step 5: Precision/recall evaluation.**
Build a test set of 50 PRs with known issues. Measure: does your agent find them? Does it flag anything it shouldn't?

### Memory Triage

**Memorize cold:**
- GitHub API PR review flow: `get PR diff` → `create review comments` → `submit review`
- AST node types for Python: `FunctionDef`, `Try`, `ExceptHandler`, `Call`, `Name`
- The precision/recall formula for agent eval: `precision = true_positives / (true_positives + false_positives)`

**Understand deeply:**
- Why precision matters more than recall for code review — *"a review agent with 60% recall but 95% precision is useful. A review agent with 95% recall but 60% precision is useless (too many false alarms)."*
- The limits of LLM code review — *"LLMs are good at style issues and obvious bugs. They're bad at architectural issues ('this function doesn't belong in this module') and performance implications. Know the boundary."*

### First Concrete Step

> "Install PyGithub. Write a script that authenticates with a GitHub token, fetches a PR diff from a public repo, and prints the changed files with line numbers. Get the raw diff format working before you analyze anything."

---

## Phase 3: The Build

> *Every false positive is a loss of trust. Optimize for precision.*

### Milestone 1: GitHub API Integration

```python
from github import Github
import requests

class PRReviewer:
    def __init__(self, token: str):
        self.github = Github(token)
    
    def get_pr_diff(self, repo_name: str, pr_number: int) -> list[dict]:
        """Fetch PR diff and parse into file changes."""
        repo = self.github.get_repo(repo_name)
        pr = repo.get_pull(pr_number)
        
        # Get diff
        diff_url = pr.diff_url
        response = requests.get(diff_url, headers={"Authorization": f"token {self.github.oauth_token}"})
        diff_text = response.text
        
        # Parse diff into file changes
        files = self._parse_diff(diff_text)
        return files
    
    def _parse_diff(self, diff_text: str) -> list[dict]:
        """Parse unified diff format into structured changes."""
        files = []
        current_file = None
        
        for line in diff_text.split("\n"):
            if line.startswith("+++ b/"):
                current_file = line[6:]  # filename
                files.append({"filename": current_file, "additions": [], "deletions": []})
            elif line.startswith("@@"):
                # @@ -start,count +start,count @@
                # Parse line numbers
                ...
            elif current_file:
                if line.startswith("+"):
                    files[-1]["additions"].append(line[1:])
                elif line.startswith("-"):
                    files[-1]["deletions"].append(line[1:])
        
        return files
```

### Milestone 2: Rule-Based Checkers (Zero False Positives)

```python
# These are DETERMINISTIC. If they flag something, it's always a real issue.

def check_bare_except(source_code: str) -> list[dict]:
    """Find bare 'except:' clauses that catch ALL exceptions."""
    tree = ast.parse(source_code)
    issues = []
    for node in ast.walk(tree):
        if isinstance(node, ast.ExceptHandler):
            if node.type is None:  # bare except
                issues.append({
                    "type": "bare_except",
                    "line": node.lineno,
                    "message": "Bare except clause catches ALL exceptions, including SystemExit and KeyboardInterrupt. Use `except Exception as e:` instead.",
                    "severity": "medium"
                })
    return issues

def check_hardcoded_secrets(source_code: str) -> list[dict]:
    """Find potential hardcoded API keys and passwords."""
    patterns = [
        (r'(?i)(api[_-]?key|secret|password|token)\s*=\s*["\'][A-Za-z0-9_\-]{16,}["\']', "Possible hardcoded secret"),
        (r'(?i)sk-[A-Za-z0-9]{32,}', "Possible OpenAI API key"),
    ]
    issues = []
    for pattern, message in patterns:
        for match in re.finditer(pattern, source_code):
            issues.append({
                "type": "hardcoded_secret",
                "line": source_code[:match.start()].count("\n") + 1,
                "message": message,
                "severity": "high"
            })
    return issues
```

### Milestone 3: LLM-Powered Review

```python
def llm_review_diff(file_diff: dict, repo_context: str) -> list[dict]:
    """Use LLM to review a file diff with context."""
    prompt = f"""Review this code diff for issues. Consider:
1. Logic bugs (off-by-one, missing edge cases, race conditions)
2. Error handling (unhandled failures, swallowed exceptions)
3. Security (input validation, injection risks, auth bypass)
4. Code quality (readability, maintainability, consistency with patterns)

IMPORTANT: Only flag issues you are CONFIDENT are real problems.
If you are unsure, do NOT flag it. False positives destroy trust.

Repository context: {repo_context}

Diff for {file_diff['filename']}:
```diff
{format_diff(file_diff)}
```

Return issues as structured JSON with: line_number, severity (low/medium/high), message, confidence (0.0-1.0)"""
    
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",  # cheaper model for code review
        messages=[{"role": "user", "content": prompt}],
        response_format=ReviewIssues,
        temperature=0.0
    )
    
    issues = response.choices[0].message.parsed
    
    # Confidence filter: only include high-confidence issues
    return [i for i in issues if i.confidence >= 0.8]
```

**Expected stuck point:** The LLM flags obvious false positives — "function `get_data` could be renamed to `fetch_data`" or "consider adding type hints" on a file that clearly follows the project's existing patterns.

**Maya's Socratic question:**
> *"Your LLM reviewer flagged 5 issues. 2 are real bugs. 3 are style preferences that aren't in the team's conventions. How do you teach the LLM what the team's ACTUAL conventions are?"*

> They should: provide the team's `.editorconfig`, `pyproject.toml`, or a `CONTRIBUTING.md` as context. Add a system prompt section: "Only flag issues that violate team conventions OR are clear logic bugs. Do not suggest style changes that are not enforced by the project's tooling."

### Milestone 4: Precision/Recall Evaluation

Build the test framework:

```python
class ReviewEvaluator:
    def __init__(self, test_set: list[dict]):
        """
        test_set: [
            {
                "pr_url": "https://github.com/...",
                "known_issues": [
                    {"line": 42, "type": "bare_except", "message": "...", "severity": "medium"}
                ]
            }
        ]
        """
        self.test_set = test_set
    
    def evaluate(self, review_agent) -> dict:
        true_positives = 0
        false_positives = 0
        false_negatives = 0
        
        for test_case in self.test_set:
            agent_issues = review_agent.review_pr(test_case["pr_url"])
            known_issues = test_case["known_issues"]
            
            for agent_issue in agent_issues:
                matched = any(
                    self._issues_match(agent_issue, known)
                    for known in known_issues
                )
                if matched:
                    true_positives += 1
                else:
                    false_positives += 1
            
            for known in known_issues:
                matched = any(
                    self._issues_match(agent_issue, known)
                    for agent_issue in agent_issues
                )
                if not matched:
                    false_negatives += 1
        
        precision = true_positives / (true_positives + false_positives) if (true_positives + false_positives) > 0 else 0
        recall = true_positives / (true_positives + false_negatives) if (true_positives + false_negatives) > 0 else 0
        
        return {
            "precision": precision,
            "recall": recall,
            "f1": 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0,
            "true_positives": true_positives,
            "false_positives": false_positives,
            "false_negatives": false_negatives
        }
```

### Rohan's Mid-Build Interruption

> *Rohan looks at your precision numbers.*

"Your precision is 78%. That means 22% of your review comments are WRONG. If a human reviewer was wrong 22% of the time, the team would stop reading their reviews.

I need precision > 90%. That means you need to CUT 60% of your false positives. How will you identify which comments are false positives and stop generating them?"

### Priya's Requirement Change

> *Priya: "CodeBridge wants the agent to also detect SECURITY issues specifically. OWASP Top 10 for web apps. SQL injection, XSS, CSRF, insecure deserialization. They'll pay extra for security-specific review. Three extra days."*

---

## Phase 4: Review (Rohan)

> *You submit: GitHub-integrated PR review agent, rule-based + LLM checkers, precision/recall report on 50 PRs, security scanning.*

### Decision Documentation Required

1. Your false positive reduction strategy — what patterns you excluded and why
2. Your split between rule-based and LLM review — why some checks are deterministic and others use LLM
3. The precision/recall results — and what you did to get precision > 90%
4. How you handle different programming languages

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | Reviews PRs, posts inline comments, submits review. End-to-end. |
| 2 | **Edge cases?** | ✅ PASS | Handles PRs with no changes, binary files, deleted files. Handles rate limits. |
| 3 | **Cost-aware?** | ✅ PASS | Rule-based checkers cover 60% of issues for $0. LLM review costs $0.02/PR. Cached results for unchanged files across commits. |
| 4 | **Observable?** | ✅ PASS | Every review decision is traced. I can see why each issue was flagged. |
| 5 | **Right approach?** | ✅ PASS | Rule-based for deterministic checks. LLM for logic analysis. This is the production pattern. |
| 6 | **Decisions justified?** | ✅ PASS | You showed exactly which checks are rule-based, which are LLM, and why. |
| 7 | **Measurable quality?** | ✅ PASS | Precision: 92.3%. Recall: 68.1%. False positive rate: 7.7%. Under the 10% gate. |

### Verdict: ✅ APPROVED

*Rohan nods.*

"92.3% precision. Acceptable. Your false positive rate is under 10%. The team will trust the agent's comments because most of them are real issues.

**Note:** Your recall is only 68%. That means the agent misses 32% of real issues. That's fine — the human reviewer catches those. The agent is a force multiplier, not a replacement. Never optimize recall at the expense of precision."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "Code review agents are one of the hardest agent applications to get right. Here's why you succeeded:

- **High precision > high recall.** Every engineer worries about AI being a noisy reviewer. By optimizing for precision (never flag something unless you're sure), you built trust. The rest of your agent career: apply this principle.
- **Rule-based + LLM hybrid.** Deterministic checks for what you can define (bare except, hardcoded secrets). LLM for what requires judgment (logic bugs). This pattern applies to almost every agent system.
- **Evaluation with precision/recall.** Most agent projects skip formal eval. You measured it. You optimized it. You proved it.

**What you'll see again:**
- **Tool design (GitHub API)** — This pattern (fetch data → analyze → write back) appears in every automation agent.
- **Precision/recall evaluation** — Project 11 (NL2SQL) uses this for SQL accuracy measurement.
- **LLM reviewing LLM output** — You used LLM-as-judge for code review quality. This is the same pattern used in RLHF, self-critique, and constitutional AI. You just don't know it yet.

> *Maya: "Project 11 is text-to-SQL. The agent writes database queries from natural language. And the scariest part: it has to execute them safely without dropping tables."*
