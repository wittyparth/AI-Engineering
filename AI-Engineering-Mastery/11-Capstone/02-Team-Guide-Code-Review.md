# 🏗️ Team-Guided Build: AI Code Review Assistant

> **Your Role:** Junior Engineer — you write all the code.
> **Our Role:** Product Manager + Tech Lead + Senior Engineer + QA — we define everything, review everything, set every standard.
> **Goal:** Ship a production-grade AI Code Review Assistant that competes with CodeRabbit and Qodo.

---

## 📋 Project Brief (Product Manager)

**Product Name:** `PR-Buddy`
**Market:** AI Code Review ($12.8B market in 2026, growing at 27% CAGR)
**Real competitors:** CodeRabbit ($100M+ ARR), Qodo, GitHub Copilot Code Review
**Problem:** Developers merge 30-50% of PRs without meaningful review. AI-generated code (22% of all merged code) introduces ~1.7× more issues. Teams need automated, line-level code review that catches real bugs before they reach production.

### User Stories

```
As a senior engineer, I want every PR to be automatically reviewed for:
  - Security vulnerabilities (OWASP Top 10)
  - Common bugs (null pointers, race conditions, resource leaks)
  - Code style violations (against team standards)
  - Missing error handling
  - Test coverage gaps
So that I can focus on architectural feedback instead of nitpicks.

As a junior engineer, I want specific, line-level feedback with code suggestions
So that I learn best practices while I code.

As a CTO, I want a quality gate that blocks PRs with critical issues
So that production incidents from preventable bugs drop to near zero.
```

### Acceptance Criteria

```
✅ Reviews are posted as line-level GitHub PR comments within 2 minutes
✅ Each review has a severity tag: CRITICAL / MAJOR / MINOR / NIT
✅ CRITICAL issues block merge (via GitHub status check)
✅ False positive rate stays below 20%
✅ Supports Python, JavaScript, TypeScript (extensible to more)
✅ Every review includes a summary comment at the top of the PR
✅ System cost stays under $0.02 per review
✅ All reviews are logged in Langfuse for quality monitoring
```

### Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Bug catch rate | >30% of bugs in changed code | Compare against human review findings |
| False positive rate | <20% | Manual audit of 100 reviews |
| Review time | <90 seconds per PR | Average across 100 PRs |
| Developer satisfaction | >4.0/5.0 | NPS survey after 1 week |

---

## 🏗️ System Architecture (Tech Lead)

### High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                        PR-Buddy System                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  GitHub Webhook ──→ FastAPI Server ──→ Agent Pipeline        │
│       (pr.opened)          │                  │              │
│                            │                  ▼              │
│                            │           ┌──────────────┐     │
│                            │           │ Orchestrator  │     │
│                            │           │ (LangGraph)   │     │
│                            │           └──────┬───────┘     │
│                            │                  │              │
│                            │     ┌────────────┼──────────┐  │
│                            │     ▼            ▼          ▼  │
│                            │  [Security]  [Bug]     [Style] │
│                            │   Agent      Agent      Agent  │
│                            │     │            │          │  │
│                            │     └────────────┼──────────┘  │
│                            │                  ▼              │
│                            │           ┌──────────────┐     │
│                            │           │  Summary     │     │
│                            │           │  Agent       │     │
│                            │           └──────┬───────┘     │
│                            │                  │              │
│                            │                  ▼              │
│                            │           GitHub API Post      │
│                            │           (comments + check)   │
└─────────────────────────────────────────────────────────────┘
```

### Tech Stack

| Component | Choice | Why |
|-----------|--------|-----|
| API Framework | FastAPI | You already know it. Async. Python. |
| Agent Framework | LangGraph | Stateful, checkpointable, observable |
| LLM | GPT-4o-mini | Cheap ($0.15/1M input), fast, good at code |
| Vector DB | ChromaDB | Local, no infra needed for MVP |
| Embeddings | text-embedding-3-small | $0.02/1M tokens, 1536 dimensions |
| Storage | SQLite → PostgreSQL | Start simple, upgrade later |
| Deployment | Docker + Railway/Vercel | Zero-infra deployment |
| Observability | Langfuse | Open-source, self-hostable |
| CI/CD | GitHub Actions | Runs eval suite on push |

### Data Model

```python
# pr_buddy/models.py
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class PullRequest(BaseModel):
    repo: str
    pr_number: int
    title: str
    description: str
    author: str
    base_branch: str
    head_branch: str
    changed_files: list[str]
    diff: str  # Full diff text
    created_at: datetime

class ReviewComment(BaseModel):
    pr_number: int
    file_path: str
    line_number: int
    severity: str  # CRITICAL | MAJOR | MINOR | NIT
    category: str  # security | bug | style | performance | best_practice
    title: str
    body: str
    suggested_code: Optional[str] = None

class ReviewResult(BaseModel):
    pr_number: int
    repo: str
    comments: list[ReviewComment]
    summary: str
    passed: bool  # False if any CRITICAL issue
    cost: float
    latency_ms: int
    reviewed_at: datetime
```

---

## 📅 Sprint Plan (Tech Lead)

### Week 1: Foundation (Days 1-3)

| Day | What to Build | Senior Engineer Note |
|-----|---------------|---------------------|
| 1 | Project setup + GitHub webhook receiver | FastAPI app that listens for PR events |
| 2 | PR diff fetcher + file change analyzer | Parse GitHub's diff format into structured data |
| 3 | Basic single-agent review + GitHub comment poster | Agent reads a file, analyzes it, posts a comment |

### Week 2: Multi-Agent System (Days 4-5)

| Day | What to Build | Senior Engineer Note |
|-----|---------------|---------------------|
| 4 | LangGraph agent pipeline (3 specialized agents) | Security, Bug, Style agents |
| 5 | Summary agent + severity classification + quality gate | Aggregates all findings, decides pass/fail |

### Week 3: Production Polish (Days 6-7)

| Day | What to Build | Senior Engineer Note |
|-----|---------------|---------------------|
| 6 | Vector DB for best practices + RAG-based review | Index OWASP top 10, team style guide |
| 7 | Langfuse observability + eval suite | Trace every review, measure pass/fail |

---

## 👨‍💻 Day 1: Project Setup + Webhook Receiver

### Problem (Senior Engineer)

We need a FastAPI server that receives GitHub webhook events when PRs are opened or updated. This is the entry point for everything.

### What to Build

```
pr-buddy/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app + webhook endpoint
│   ├── config.py            # Settings + env vars
│   ├── github/
│   │   ├── __init__.py
│   │   ├── client.py        # GitHub API client
│   │   └── webhook.py       # Webhook signature verification
│   └── models.py            # Pydantic models
├── .env
├── requirements.txt
├── Dockerfile
└── README.md
```

### Step-by-Step Instructions

**Step 1.1: Project skeleton**

Create the directory structure above. Initialize a Python project with `pyproject.toml` (or `requirements.txt` — your call).

Requirements needed:
```
fastapi==0.115.0
uvicorn[standard]
httpx
pydantic==2.0
python-dotenv
pydantic-settings
```

**Step 1.2: Config**

```python
# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    github_token: str  # GitHub PAT with repo scope
    github_webhook_secret: str  # Webhook secret for verification
    openai_api_key: str
    langfuse_public_key: str = ""
    langfuse_secret_key: str = ""
    langfuse_host: str = "https://cloud.langfuse.com"
    database_url: str = "sqlite:///pr_buddy.db"
    model_name: str = "gpt-4o-mini"
    max_review_cost: float = 0.02  # Max $ per review

    class Config:
        env_file = ".env"

settings = Settings()
```

**Step 1.3: GitHub client**

```python
# app/github/client.py
import httpx
from app.config import settings

class GitHubClient:
    def __init__(self):
        self.headers = {
            "Authorization": f"Bearer {settings.github_token}",
            "Accept": "application/vnd.github.v3.diff",
        }
        self.base_url = "https://api.github.com"
    
    async def get_pr_diff(self, repo: str, pr_number: int) -> str:
        """Fetch the unified diff for a PR"""
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{self.base_url}/repos/{repo}/pulls/{pr_number}",
                headers=self.headers
            )
            resp.raise_for_status()
            return resp.text
    
    async def get_pr_files(self, repo: str, pr_number: int) -> list[dict]:
        """Get list of files changed in the PR"""
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{self.base_url}/repos/{repo}/pulls/{pr_number}/files",
                headers={**self.headers, "Accept": "application/vnd.github.v3+json"}
            )
            resp.raise_for_status()
            return resp.json()
    
    async def post_comment(self, repo: str, pr_number: int, body: str):
        """Post a PR review comment"""
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{self.base_url}/repos/{repo}/pulls/{pr_number}/comments",
                headers={**self.headers, "Accept": "application/vnd.github.v3+json"},
                json={"body": body}
            )
            resp.raise_for_status()
    
    async def post_line_comment(
        self, repo: str, pr_number: int, 
        file_path: str, line_number: int, body: str
    ):
        """Post a line-level PR comment"""
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{self.base_url}/repos/{repo}/pulls/{pr_number}/comments",
                headers={**self.headers, "Accept": "application/vnd.github.v3+json"},
                json={
                    "body": body,
                    "path": file_path,
                    "line": line_number
                }
            )
            resp.raise_for_status()
    
    async def set_commit_status(
        self, repo: str, sha: str, 
        state: str, description: str
    ):
        """Set a commit status check (success/failure/pending)"""
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{self.base_url}/repos/{repo}/statuses/{sha}",
                headers={**self.headers, "Accept": "application/vnd.github.v3+json"},
                json={
                    "state": state,
                    "description": description,
                    "context": "PR-Buddy / code-review"
                }
            )
            resp.raise_for_status()
```

**Step 1.4: Webhook handler**

```python
# app/github/webhook.py
import hmac
import hashlib
from fastapi import Request, HTTPException
from app.config import settings

async def verify_webhook(request: Request, body: bytes):
    """Verify that the webhook came from GitHub"""
    signature = request.headers.get("X-Hub-Signature-256")
    if not signature:
        raise HTTPException(400, "Missing signature")
    
    expected = hmac.new(
        settings.github_webhook_secret.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(f"sha256={expected}", signature):
        raise HTTPException(400, "Invalid signature")
```

**Step 1.5: Main app**

```python
# app/main.py
from fastapi import FastAPI, Request
from app.github.client import GitHubClient
from app.github.webhook import verify_webhook
from app.config import settings

app = FastAPI(title="PR-Buddy")
github = GitHubClient()

@app.post("/webhook/github")
async def github_webhook(request: Request):
    body = await request.body()
    await verify_webhook(request, body)
    
    event = request.headers.get("X-GitHub-Event")
    if event != "pull_request":
        return {"status": "ignored", "event": event}
    
    payload = await request.json()
    action = payload.get("action")
    
    if action in ("opened", "synchronize"):
        repo = payload["repository"]["full_name"]
        pr_number = payload["pull_request"]["number"]
        
        # Fetch diff
        diff = await github.get_pr_diff(repo, pr_number)
        
        # For now: print it. We'll process it tomorrow.
        print(f"PR #{pr_number} in {repo}: {len(diff)} bytes of diff")
        
        return {"status": "received", "pr": pr_number}
    
    return {"status": "ignored", "action": action}

@app.get("/health")
async def health():
    return {"status": "ok", "model": settings.model_name}
```

### ✅ Check: Day 1 Gate

Before proceeding to Day 2, verify:

```
1. ❓ Can you start the server?   uvicorn app.main:app --reload
2. ❓ Can you receive a webhook?  Use `ngrok http 8000` → configure GitHub webhook
3. ❓ Does the health endpoint work?  curl http://localhost:8000/health
4. ❓ Does the webhook handler print PR info when you open a test PR?
```

**If all 4 pass → proceed. If not → debug and fix before moving on.**

---

## 👨‍💻 Day 2: PR Diff Analyzer

### Problem (Senior Engineer)

The GitHub diff format is a raw string. We need to parse it into structured file-level data: which files changed, what lines were added/removed, and categorizing the change type (code, config, test, docs).

### What to Build

Create `app/analyzer.py` that:
1. Takes a raw diff string
2. Parses it into structured per-file diffs
3. Detects file type: `code`, `test`, `config`, `docs`
4. Filters out irrelevant files (auto-generated, lock files, etc.)

### Implementation

```python
# app/analyzer.py
import re
from typing import Optional
from pydantic import BaseModel

class ChangedLine(BaseModel):
    line_number: int      # Line number in the new file
    content: str          # Line content
    type: str             # "added" | "removed" | "context"

class FileDiff(BaseModel):
    file_path: str
    change_type: str      # "added" | "modified" | "deleted" | "renamed"
    file_category: str    # "code" | "test" | "config" | "docs" | "generated"
    language: Optional[str]
    added_lines: list[ChangedLine]
    removed_lines: list[ChangedLine]
    hunks: list[str]      # Diff hunks for LLM context

class ParsedDiff(BaseModel):
    repo: str
    pr_number: int
    title: str
    files: list[FileDiff]
    raw_diff: str

# File type detection
EXTENSION_MAP = {
    ".py": ("code", "python"),
    ".js": ("code", "javascript"),
    ".ts": ("code", "typescript"),
    ".tsx": ("code", "typescriptreact"),
    ".jsx": ("code", "javascriptreact"),
    ".go": ("code", "go"),
    ".rs": ("code", "rust"),
    ".java": ("code", "java"),
    ".rb": ("code", "ruby"),
    ".sh": ("code", "bash"),
    ".yaml": ("config", "yaml"),
    ".yml": ("config", "yaml"),
    ".json": ("config", "json"),
    ".toml": ("config", "toml"),
    ".md": ("docs", "markdown"),
    ".rst": ("docs", "markdown"),
}

IRRELEVANT_PATTERNS = [
    r"package-lock\.json$",
    r"yarn\.lock$",
    r"pnpm-lock\.yaml$",
    r"poetry\.lock$",
    r"\.pyc$",
    r"__pycache__",
    r"\.next/",
    r"node_modules/",
    r"\.gitignore$",
]

def classify_file(file_path: str) -> tuple[str, str]:
    """Returns (file_category, language)"""
    for pattern in IRRELEVANT_PATTERNS:
        if re.search(pattern, file_path):
            return ("generated", None)
    
    import os
    ext = os.path.splitext(file_path)[1].lower()
    
    if ext in EXTENSION_MAP:
        return EXTENSION_MAP[ext]
    
    # Check for test files by path
    if "/test/" in file_path or file_path.startswith("test_"):
        ext = os.path.splitext(file_path)[1].lower()
        lang = EXTENSION_MAP.get(ext, (None, None))[1]
        return ("test", lang)
    
    return ("code", None)


def parse_diff(repo: str, pr_number: int, title: str, raw_diff: str) -> ParsedDiff:
    """
    Parse a GitHub unified diff into structured file-level data.
    
    Format:
        diff --git a/file.py b/file.py
        index abc123..def456 100644
        --- a/file.py
        +++ b/file.py
        @@ -10,6 +10,8 @@ class SomeClass:
         context line
        +added line
        -removed line
    """
    files = []
    current_file = None
    current_hunk = []
    
    for line in raw_diff.split("\n"):
        # New file section
        if line.startswith("diff --git"):
            if current_file and current_hunk:
                current_file.hunks.append("\n".join(current_hunk))
            current_hunk = []
            
            # Extract old/new filenames
            match = re.search(r"b/(.+)$", line)
            file_path = match.group(1) if match else "unknown"
            
            cat, lang = classify_file(file_path)
            current_file = FileDiff(
                file_path=file_path,
                change_type="modified",
                file_category=cat,
                language=lang,
                added_lines=[],
                removed_lines=[],
                hunks=[]
            )
            files.append(current_file)
        
        elif line.startswith("--- /dev/null"):
            if current_file:
                current_file.change_type = "added"
        
        elif line.startswith("+++ /dev/null"):
            if current_file:
                current_file.change_type = "deleted"
        
        elif line.startswith("@@"):
            if current_hunk:
                if current_file:
                    current_file.hunks.append("\n".join(current_hunk))
                current_hunk = [line]
            else:
                current_hunk = [line]
        
        elif line.startswith("+") and current_file and current_file.file_category != "generated":
            if current_hunk is not None:
                current_hunk.append(line)
            current_file.added_lines.append(ChangedLine(
                line_number=0,  # We'll compute this later
                content=line[1:],
                type="added"
            ))
        
        elif line.startswith("-") and current_file and current_file.file_category != "generated":
            if current_hunk is not None:
                current_hunk.append(line)
            current_file.removed_lines.append(ChangedLine(
                line_number=0,
                content=line[1:],
                type="removed"
            ))
        
        elif current_hunk is not None and current_file and current_file.file_category != "generated":
            current_hunk.append(line)
    
    # Don't forget the last hunk
    if current_file and current_hunk:
        current_file.hunks.append("\n".join(current_hunk))
    
    # Filter out generated files
    files = [f for f in files if f.file_category != "generated"]
    
    return ParsedDiff(
        repo=repo,
        pr_number=pr_number,
        title=title,
        files=files,
        raw_diff=raw_diff
    )
```

### ✅ Check: Day 2 Gate

```
Create test_diff.py with sample diffs and verify:
1. ❓ Can you parse a simple 1-file PR diff?
2. ❓ Are added/removed lines correctly separated?
3. ❓ Are generated files (package-lock.json) filtered out?
4. ❓ Are test files correctly classified as "test"?
5. ❓ Can you handle a multi-file PR diff?

If all 5 work → proceed. If not → fix before moving on.
```

---

## 👨‍💻 Day 3: Single-Agent Reviewer + GitHub Comment Poster

### Problem (Senior Engineer)

Now we need an LLM agent that analyzes a file diff and produces specific, actionable review comments. Start simple: one file at a time, one agent.

### What to Build

Create `app/reviewer.py` and `app/poster.py`:
1. `reviewer.py` — Takes a `FileDiff`, calls GPT-4o-mini, returns `list[ReviewComment]`
2. `poster.py` — Takes `ReviewComment` list, posts them as GitHub PR comments

### Implementation

```python
# app/reviewer.py
import json
from openai import AsyncOpenAI
from app.config import settings
from app.analyzer import FileDiff
from app.models import ReviewComment

client = AsyncOpenAI(api_key=settings.openai_api_key)

REVIEW_SYSTEM_PROMPT = """You are a senior software engineer conducting a code review.

Review the provided diff for:
1. **Security vulnerabilities** — SQL injection, XSS, hardcoded secrets, CSRF, auth bypass
2. **Logic bugs** — Null pointer dereferences, off-by-one errors, race conditions, infinite loops
3. **Code quality** — Error handling, logging, premature optimization, code duplication
4. **Performance** — N+1 queries, unnecessary allocations, blocking calls in async code

For each issue, return a JSON object with:
- severity: "CRITICAL" | "MAJOR" | "MINOR" | "NIT"
- category: "security" | "bug" | "performance" | "style" | "best_practice" | "test_coverage"
- file_path: the file this issue is in
- line_number: the line number this issue is on (approximate is fine)
- title: short, actionable title (max 60 chars)
- body: 2-3 sentence explanation of the issue and why it matters
- suggested_code: if applicable, show the fixed version of the code

Return a JSON array. If no issues found, return an empty array.
Focus on issues that could actually cause problems in production.
Don't be pedantic about style preferences unless they affect correctness.
"""

async def review_file(file_diff: FileDiff, pr_context: str = "") -> list[dict]:
    """
    Review a single file's changes and return structured issues.
    
    Args:
        file_diff: Parsed diff for one file
        pr_context: PR title/description for context
    
    Returns:
        List of review comments as dicts (to be validated into ReviewComment)
    """
    # Build the context for the LLM
    diff_text = "\n".join(file_diff.hunks) if file_diff.hunks else ""
    
    if not diff_text.strip():
        return []
    
    review_prompt = f"""
PR Context: {pr_context}

File: {file_diff.file_path}
Language: {file_diff.language or 'unknown'}
Change type: {file_diff.change_type}

Diff to review:
```{file_diff.language or ''}
{diff_text}
```

Review this diff for issues. Return JSON array.
"""
    
    response = await client.chat.completions.create(
        model=settings.model_name,
        messages=[
            {"role": "system", "content": REVIEW_SYSTEM_PROMPT},
            {"role": "user", "content": review_prompt}
        ],
        response_format={"type": "json_object"},
        temperature=0.1,
        max_tokens=2000
    )
    
    try:
        result = json.loads(response.choices[0].message.content)
        issues = result.get("issues", result if isinstance(result, list) else [result])
        return issues if isinstance(issues, list) else [issues]
    except json.JSONDecodeError:
        return [{
            "severity": "MAJOR",
            "category": "best_practice",
            "file_path": file_diff.file_path,
            "line_number": 1,
            "title": "Review parsing failed",
            "body": "Could not parse LLM review output as JSON. Manual review recommended.",
            "suggested_code": None
        }]
```

```python
# app/poster.py
from app.github.client import GitHubClient
from app.models import ReviewComment

github = GitHubClient()

def format_comment_body(comment: ReviewComment) -> str:
    """Format a review comment with severity badge and suggestion"""
    severity_icons = {
        "CRITICAL": "🔴",
        "MAJOR": "🟠",
        "MINOR": "🟡",
        "NIT": "⚪"
    }
    icon = severity_icons.get(comment.severity, "⚪")
    
    body = f"{icon} **[{comment.severity}] {comment.title}**\n\n{comment.body}"
    
    if comment.suggested_code:
        body += f"\n\n**Suggested fix:**\n```\n{comment.suggested_code}\n```"
    
    body += f"\n\n*PR-Buddy review · {comment.category}*"
    
    return body


async def post_review(
    repo: str, 
    pr_number: str, 
    comments: list[ReviewComment],
    sha: str
) -> dict:
    """
    Post all review comments to the PR and set status check.
    """
    if not comments:
        await github.set_commit_status(
            repo, sha, "success", 
            "No issues found ✅"
        )
        return {"status": "success", "comments_posted": 0}
    
    critical_count = sum(1 for c in comments if c.severity == "CRITICAL")
    major_count = sum(1 for c in comments if c.severity == "MAJOR")
    
    # Post line-level comments
    for comment in comments:
        if comment.file_path and comment.line_number:
            try:
                await github.post_line_comment(
                    repo, pr_number,
                    comment.file_path,
                    comment.line_number,
                    format_comment_body(comment)
                )
            except Exception as e:
                # Fall back to file-level comment
                await github.post_comment(
                    repo, pr_number,
                    f"**File: {comment.file_path}**\n\n{format_comment_body(comment)}"
                )
    
    # Post summary comment at the top
    summary = f"""## PR-Buddy Review Summary

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | {critical_count} |
| 🟠 MAJOR | {major_count} |
| 🟡 MINOR | {sum(1 for c in comments if c.severity == 'MINOR')} |
| ⚪ NIT | {sum(1 for c in comments if c.severity == 'NIT')} |

**Total issues found: {len(comments)}**
"""
    
    await github.post_comment(repo, pr_number, summary)
    
    # Set commit status
    if critical_count > 0:
        await github.set_commit_status(
            repo, sha, "failure",
            f"{critical_count} critical issues found — fix before merge"
        )
    else:
        await github.set_commit_status(
            repo, sha, "success",
            f"Review complete: {len(comments)} issues found"
        )
    
    return {
        "status": "success",
        "comments_posted": len(comments),
        "critical": critical_count,
        "major": major_count
    }
```

Now update `app/main.py` to wire everything together:

```python
@app.post("/webhook/github")
async def github_webhook(request: Request):
    body = await request.body()
    await verify_webhook(request, body)
    
    event = request.headers.get("X-GitHub-Event")
    if event != "pull_request":
        return {"status": "ignored", "event": event}
    
    payload = await request.json()
    action = payload.get("action")
    
    if action in ("opened", "synchronize"):
        repo = payload["repository"]["full_name"]
        pr_number = payload["pull_request"]["number"]
        title = payload["pull_request"]["title"]
        sha = payload["pull_request"]["head"]["sha"]
        
        # Tell GitHub we're reviewing
        await github.set_commit_status(repo, sha, "pending", "PR-Buddy is reviewing...")
        
        # 1. Fetch + parse diff
        raw_diff = await github.get_pr_diff(repo, pr_number)
        parsed = parse_diff(repo, pr_number, title, raw_diff)
        
        # 2. Review each file
        all_comments = []
        for file_diff in parsed.files:
            issues = await review_file(file_diff, title)
            for issue in issues:
                all_comments.append(ReviewComment(
                    pr_number=pr_number,
                    file_path=issue.get("file_path", file_diff.file_path),
                    line_number=issue.get("line_number", 1),
                    severity=issue.get("severity", "MINOR"),
                    category=issue.get("category", "best_practice"),
                    title=issue.get("title", "Review finding"),
                    body=issue.get("body", ""),
                    suggested_code=issue.get("suggested_code")
                ))
        
        # 3. Post results
        result = await post_review(repo, pr_number, all_comments, sha)
        
        return {"status": "reviewed", "pr": pr_number, "result": result}
    
    return {"status": "ignored", "action": action}
```

### ✅ Check: Day 3 Gate

```
1. ❓ Create a test PR in a test repo with intentional bugs (unhandled exception, 
    hardcoded API key, SQL injection pattern, missing error handling)
2. ❓ Does PR-Buddy catch at least 3 out of 4 issues?
3. ❓ Are comments posted as line-level GitHub comments?
4. ❓ Is a summary comment posted?
5. ❓ Is the commit status set correctly (failure for critical)?
6. ❓ Does the review complete in under 60 seconds for a 5-file PR?

If all 6 pass → proceed. If not → debug and fix.
```

---

## 👨‍💻 Day 4: Multi-Agent Pipeline with LangGraph

### Problem (Senior Engineer)

A single agent reviewing everything misses specialized issues. Security bugs need different analysis than style problems. We need **specialized agents** coordinated by a supervisor.

### What to Build

Create `app/agents/` directory with:
1. `security_agent.py` — Focuses only on OWASP patterns, secrets, auth issues
2. `bug_agent.py` — Focuses only on logic bugs, null pointers, race conditions
3. `style_agent.py` — Focuses only on code quality, best practices, error handling
4. `orchestrator.py` — LangGraph state graph that runs all agents and aggregates results

### Implementation

```python
# app/agents/orchestrator.py
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from openai import AsyncOpenAI
import json
import operator

from app.config import settings
from app.analyzer import ParsedDiff, FileDiff
from app.models import ReviewComment

client = AsyncOpenAI(api_key=settings.openai_api_key)

# ── State Definition ──

class AgentState(TypedDict):
    """State flowing through the review graph"""
    parsed_diff: ParsedDiff
    pr_context: str
    security_findings: Annotated[list[dict], operator.add]
    bug_findings: Annotated[list[dict], operator.add]
    style_findings: Annotated[list[dict], operator.add]
    all_findings: list[dict]
    review_summary: str
    passed: bool
    cost: float
    files_reviewed: int

# ── Agent Prompts ──

SECURITY_PROMPT = """You are a security-focused code reviewer. Review the diff for:
- SQL injection (user input in raw SQL queries)
- Hardcoded API keys, passwords, tokens
- Authentication/authorization bypasses
- Cross-site scripting (XSS) in web apps
- Path traversal (user input in file paths)
- Insecure deserialization
- Command injection (user input in system()/exec() calls)

Be strict. Security issues are the most important to catch.
Only flag REAL issues — not theoretical ones.
Return JSON array of findings with severity, line, explanation, and suggested fix."""

BUG_PROMPT = """You are a bug-focused code reviewer. Review the diff for:
- Null/undefined reference errors
- Off-by-one errors in loops
- Race conditions in shared state
- Incorrect error handling (bare except, swallowed exceptions)
- Async/await misuse (blocking calls in async, unawaited coroutines)
- Type errors (assuming a value is a type it might not be)
- Off-by-one or fencepost errors
- Missing return values
- Incorrect comparison operators (= vs ==)

Focus on things that WILL break in production, not style preferences.
Return JSON array of findings."""

STYLE_PROMPT = """You are a code quality reviewer. Review the diff for:
- Missing error handling on fallible operations
- Functions that are too long/have too many responsibilities
- Magic numbers/strings that should be constants
- Dead code or commented-out code
- Violations of common patterns for the language
- Missing type hints in Python (if applicable)
- Logging that exposes sensitive data
- Inefficient data structures (list instead of set for lookups)

These are important but less critical than security or bugs.
Return JSON array of findings."""

# ── Agent Functions ──

async def run_review_agent(
    file_diffs: list[FileDiff], 
    pr_context: str, 
    system_prompt: str,
    temperature: float = 0.1
) -> list[dict]:
    """Run a specialized review agent across all files"""
    all_findings = []
    
    for file_diff in file_diffs:
        if not file_diff.hunks:
            continue
        
        diff_text = "\n".join(file_diff.hunks)
        
        user_prompt = f"""PR Context: {pr_context}

File: {file_diff.file_path}
Language: {file_diff.language or 'unknown'}

Diff:
```{file_diff.language or ''}
{diff_text}
```

Review for issues. Return JSON array of {{"file_path", "line_number", "severity", "title", "body", "suggested_code", "category"}}.
Category should be one of: security, bug, performance, style, best_practice.
"""
        
        response = await client.chat.completions.create(
            model=settings.model_name,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt}
            ],
            response_format={"type": "json_object"},
            temperature=temperature,
            max_tokens=2000
        )
        
        try:
            result = json.loads(response.choices[0].message.content)
            issues = result if isinstance(result, list) else result.get("findings", result.get("issues", [result]))
            if isinstance(issues, list):
                all_findings.extend(issues)
        except json.JSONDecodeError:
            pass
    
    return all_findings


async def security_agent(state: AgentState) -> dict:
    """Run security review"""
    findings = await run_review_agent(
        state["parsed_diff"].files,
        state["pr_context"],
        SECURITY_PROMPT
    )
    return {"security_findings": findings}


async def bug_agent(state: AgentState) -> dict:
    """Run bug review"""
    findings = await run_review_agent(
        state["parsed_diff"].files,
        state["pr_context"],
        BUG_PROMPT
    )
    return {"bug_findings": findings}


async def style_agent(state: AgentState) -> dict:
    """Run style review"""
    findings = await run_review_agent(
        state["parsed_diff"].files,
        state["pr_context"],
        STYLE_PROMPT
    )
    return {"style_findings": findings}


async def aggregate_agent(state: AgentState) -> dict:
    """Aggregate all findings, deduplicate, classify severity"""
    all_raw = (
        state.get("security_findings", []) +
        state.get("bug_findings", []) +
        state.get("style_findings", [])
    )
    
    # Dedup by (file_path, title)
    seen = set()
    deduped = []
    for f in all_raw:
        key = (f.get("file_path", ""), f.get("title", ""))
        if key not in seen:
            seen.add(key)
            deduped.append(f)
    
    # Classify final severity (most strict wins)
    critical = any(f.get("severity") == "CRITICAL" for f in deduped)
    
    return {
        "all_findings": deduped,
        "passed": not critical,
        "files_reviewed": len(state["parsed_diff"].files)
    }


async def summary_agent(state: AgentState) -> dict:
    """Generate a human-readable summary"""
    findings = state["all_findings"]
    counts = {"CRITICAL": 0, "MAJOR": 0, "MINOR": 0, "NIT": 0}
    categories = {}
    
    for f in findings:
        sev = f.get("severity", "MINOR")
        cat = f.get("category", "other")
        counts[sev] = counts.get(sev, 0) + 1
        categories[cat] = categories.get(cat, 0) + 1
    
    summary = f"""## PR-Buddy Review Summary

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | {counts.get('CRITICAL', 0)} |
| 🟠 MAJOR | {counts.get('MAJOR', 0)} |
| 🟡 MINOR | {counts.get('MINOR', 0)} |
| ⚪ NIT | {counts.get('NIT', 0)} |

### By Category
{chr(10).join(f"- **{k.capitalize()}**: {v}" for k, v in sorted(categories.items()))}

**Total: {len(findings)} issues across {state.get('files_reviewed', 0)} files**
"""
    
    return {"review_summary": summary}


# ── Build Graph ──

def build_review_graph():
    """Construct the LangGraph for multi-agent code review"""
    workflow = StateGraph(AgentState)
    
    # Add nodes
    workflow.add_node("security", security_agent)
    workflow.add_node("bug", bug_agent)
    workflow.add_node("style", style_agent)
    workflow.add_node("aggregate", aggregate_agent)
    workflow.add_node("summary", summary_agent)
    
    # Set entry point
    workflow.set_entry_point("security")
    
    # Connect: all 3 agents run in parallel → aggregate → summary
    workflow.add_edge("security", "aggregate")
    workflow.add_edge("bug", "aggregate")
    workflow.add_edge("style", "aggregate")
    workflow.add_edge("aggregate", "summary")
    workflow.add_edge("summary", END)
    
    # Compile
    return workflow.compile()


# Singleton graph instance
review_graph = build_review_graph()


async def run_multi_agent_review(
    parsed_diff: ParsedDiff,
    pr_context: str
) -> dict:
    """Run the full multi-agent review pipeline"""
    
    initial_state = {
        "parsed_diff": parsed_diff,
        "pr_context": pr_context,
        "security_findings": [],
        "bug_findings": [],
        "style_findings": [],
        "all_findings": [],
        "review_summary": "",
        "passed": True,
        "cost": 0.0,
        "files_reviewed": 0
    }
    
    result = await review_graph.ainvoke(initial_state)
    return result
```

Now update the webhook handler to use the multi-agent pipeline:

```python
# In app/main.py — replace the review_file loop with:
from app.agents.orchestrator import run_multi_agent_review

result = await run_multi_agent_review(parsed, title)

# Convert to ReviewComment objects
all_comments = []
for finding in result["all_findings"]:
    all_comments.append(ReviewComment(
        pr_number=pr_number,
        file_path=finding.get("file_path", ""),
        line_number=finding.get("line_number", 1),
        severity=finding.get("severity", "MINOR"),
        category=finding.get("category", "best_practice"),
        title=finding.get("title", ""),
        body=finding.get("body", ""),
        suggested_code=finding.get("suggested_code")
    ))

review_result = await post_review(repo, pr_number, all_comments, sha)
```

### ✅ Check: Day 4 Gate

```
1. ❓ Can you run the LangGraph pipeline locally?
2. ❓ Does the security agent catch a hardcoded API key in a test PR?
3. ❓ Does the bug agent catch a null pointer / NoneType error?
4. ❓ Does the style agent catch missing error handling?
5. ❓ Are results properly deduplicated (same issue from 2 agents)?
6. ❓ Does the graph complete in under 2 minutes for a 3-file PR?
7. ❓ Run it against a real open-source PR (pick any). Does it produce reasonable output?

If all 7 pass → proceed.
```

---

## 👨‍💻 Day 5: Summary + Quality Gate + Testing

### Problem (Senior Engineer)

Reviews are running, but we need to:
1. Block PRs with CRITICAL issues (quality gate via GitHub status check)
2. Track false positives (not every agent finding is correct)
3. Build an eval suite to measure our actual performance

### What to Build

1. **Quality gate** — Already partially built in `post_review()`. Enhance it.
2. **Eval suite** — Create `tests/test_evals.py` with known-bug test cases
3. **False positive tracker** — Simple JSON log of all reviews

### Eval Suite

```python
# tests/test_evals.py
"""
Test suite for PR-Buddy.
Each test case has a known set of bugs. We check if PR-Buddy finds them.
"""
import pytest
from app.analyzer import parse_diff
from app.agents.orchestrator import run_multi_agent_review

# Test Case 1: Hardcoded API key
HARDCODED_KEY_DIFF = """diff --git a/app/config.py b/app/config.py
index abc..def 100644
--- a/app/config.py
+++ b/app/config.py
@@ -1,5 +1,10 @@
 import os
+import requests
 
 class Config:
     SECRET_KEY = os.environ.get("SECRET_KEY")
+    
+    def send_to_analytics(self, data):
+        # TODO: fix this
+        api_key = "sk-abc123def456ghi789jkl"
+        requests.post("https://analytics.example.com", json={"data": data}, headers={"Authorization": f"Bearer {api_key}"})
"""

EXPECTED_HARDCODED_KEY = {
    "should_find": True,
    "min_severity": "CRITICAL",
    "category": "security",
    "keywords": ["hardcoded", "api_key", "sk-"]
}

# Test Case 2: Null pointer / NoneType bug
NULL_POINTER_DIFF = """diff --git a/app/user.py b/app/user.py
index abc..def 100644
--- a/app/user.py
+++ b/app/user.py
@@ -1,3 +1,8 @@
 def get_user_email(user):
+    if user is None:
+        return None
     return user.get("email", "")
 
+def send_welcome_email(user):
+    # Bug: doesn't check if user is None
+    name = user["name"]  # Potential KeyError + None
+    print(f"Sending welcome email to {name}")
+    # Also: no actual email sending logic
"""

EXPECTED_NULL_POINTER = {
    "should_find": True,
    "min_severity": "MAJOR",
    "category": "bug",
    "keywords": ["None", "KeyError", "null"]
}

# Test Case 3: SQL injection
SQL_INJECTION_DIFF = """diff --git a/app/db.py b/app/db.py
index abc..def 100644
--- a/app/db.py
+++ b/app/db.py
@@ -1,3 +1,14 @@
 import sqlite3
 
+def search_products(search_term: str):
+    conn = sqlite3.connect("store.db")
+    cursor = conn.cursor()
+    # BUG: SQL injection — user input directly in query
+    query = f"SELECT * FROM products WHERE name LIKE '%{search_term}%'"
+    cursor.execute(query)
+    results = cursor.fetchall()
+    conn.close()
+    return results
+
 def get_product(product_id: int):
     conn = sqlite3.connect("store.db")
"""

EXPECTED_SQL_INJECTION = {
    "should_find": True,
    "min_severity": "CRITICAL",
    "category": "security",
    "keywords": ["SQL", "injection", "injection"]
}

# Test Case 4: Merge with no issues
CLEAN_DIFF = """diff --git a/app/utils.py b/app/utils.py
index abc..def 100644
--- a/app/utils.py
+++ b/app/utils.py
@@ -1 +1,5 @@
-def old_name():
-    pass
+def format_date(year: int, month: int, day: int) -> str:
+    \"\"\"Format a date as YYYY-MM-DD\"\"\"
+    if not (1 <= month <= 12):
+        raise ValueError(f"Invalid month: {month}")
+    return f"{year:04d}-{month:02d}-{day:02d}"
"""

EXPECTED_CLEAN = {
    "should_find": False
}


@pytest.mark.asyncio
async def test_hardcoded_key_detection():
    parsed = parse_diff("test/repo", 1, "Add analytics client", HARDCODED_KEY_DIFF)
    result = await run_multi_agent_review(parsed, "Add analytics client")
    findings = result["all_findings"]
    
    has_critical_security = any(
        f.get("severity") == "CRITICAL" and 
        f.get("category") == "security" and
        any(k in str(f).lower() for k in EXPECTED_HARDCODED_KEY["keywords"])
        for f in findings
    )
    
    assert has_critical_security, (
        f"Expected to find hardcoded API key (CRITICAL/security). "
        f"Findings: {json.dumps(findings, indent=2)}"
    )


@pytest.mark.asyncio
async def test_null_pointer_detection():
    parsed = parse_diff("test/repo", 2, "Add welcome email", NULL_POINTER_DIFF)
    result = await run_multi_agent_review(parsed, "Add welcome email")
    findings = result["all_findings"]
    
    has_bug = any(
        f.get("severity") in ("CRITICAL", "MAJOR") and
        any(k in str(f).lower() for k in EXPECTED_NULL_POINTER["keywords"])
        for f in findings
    )
    
    assert has_bug, f"Expected to find null pointer bug. Findings: {json.dumps(findings, indent=2)}"


@pytest.mark.asyncio
async def test_sql_injection_detection():
    parsed = parse_diff("test/repo", 3, "Add product search", SQL_INJECTION_DIFF)
    result = await run_multi_agent_review(parsed, "Add product search")
    findings = result["all_findings"]
    
    has_sql_injection = any(
        f.get("severity") == "CRITICAL" and
        f.get("category") == "security" and
        any(k.lower() in str(f).lower() for k in EXPECTED_SQL_INJECTION["keywords"])
        for f in findings
    )
    
    assert has_sql_injection, (
        f"Expected to find SQL injection. Findings: {json.dumps(findings, indent=2)}"
    )


@pytest.mark.asyncio
async def test_clean_diff_no_false_positives():
    parsed = parse_diff("test/repo", 4, "Refactor date formatting", CLEAN_DIFF)
    result = await run_multi_agent_review(parsed, "Refactor date formatting")
    findings = result["all_findings"]
    
    critical_findings = [f for f in findings if f.get("severity") == "CRITICAL"]
    assert len(critical_findings) == 0, (
        f"Expected 0 CRITICAL findings for clean diff. Got: {len(critical_findings)}"
    )
```

### ✅ Check: Day 5 Gate

```
1. ❓ Do ALL eval tests pass? (bug, security, sql injection, clean diff)
2. ❓ Does a PR with CRITICAL findings get a "failure" status check?
3. ❓ Does a PR with only minor findings pass the quality gate?
4. ❓ Can you run:  pytest tests/test_evals.py -v  and see 4/4 pass?

If all 4 pass → proceed.
```

---

## 👨‍💻 Day 6: RAG-Enhanced Review (Best Practices Vector DB)

### Problem (Senior Engineer)

The agents are smart, but they don't know YOUR team's specific standards. We need a vector DB of best practices that the agents can query during review.

### What to Build

1. Seed a ChromaDB collection with common best practices (OWASP, error handling patterns, async patterns)
2. Add a RAG step before agent review: retrieve relevant best practices, inject into agent context
3. Create `seed_knowledge_base.py` to initialize the DB

### Implementation

```python
# app/knowledge_base.py
import chromadb
from chromadb.config import Settings
from openai import OpenAI
from app.config import settings

client = OpenAI(api_key=settings.openai_api_key)
chroma_client = chromadb.PersistentClient(path="./knowledge_db")

def get_or_create_collection():
    return chroma_client.get_or_create_collection(
        name="code-review-best-practices",
        metadata={"hnsw:space": "cosine"}
    )


def embed_text(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding


def seed_knowledge_base():
    """Seed the vector DB with best practices for code review"""
    collection = get_or_create_collection()
    
    practices = [
        # Security
        "Always use parameterized queries instead of string formatting for SQL. SQL injection via f-strings or concatenation is a CRITICAL vulnerability.",
        "Never hardcode API keys, passwords, or tokens in source code. Use environment variables or a secrets manager.",
        "Validate and sanitize all user inputs. Never trust user input directly in file paths, URLs, or system commands.",
        "Use HTTPS for all external API calls. Never send sensitive data over unencrypted connections.",
        
        # Error handling
        "Always handle exceptions explicitly. Never use bare 'except:' clauses — they catch SystemExit and KeyboardInterrupt too.",
        "Log errors with context (traceback, input values) — not just 'something went wrong'. Never log passwords or PII.",
        "Return proper HTTP error codes from APIs: 400 for bad input, 401 for auth, 403 for forbidden, 404 for not found, 500 for server errors.",
        
        # Async patterns
        "Never use blocking calls (time.sleep(), requests.get()) in async code. Use asyncio.sleep(), httpx.AsyncClient() instead.",
        "Always await coroutines. Calling an async function without await returns a coroutine object, not the result.",
        "Use asyncio.gather() for parallel async tasks. Sequential awaits waste the benefit of async.",
        
        # Python-specific
        "Use type hints for all function parameters and return values. This catches bugs at the type-checking stage.",
        "Use pathlib over os.path for file operations. It's cleaner, cross-platform, and composable.",
        "Use dataclasses or Pydantic models instead of raw dicts for structured data. They validate and document.",
        
        # Performance
        "Use sets for membership tests (O(1)) instead of lists (O(n)) when order doesn't matter.",
        "Avoid N+1 queries in database access. Use eager loading or batch queries.",
        "Use connection pooling for database and HTTP connections. Creating a new connection per request is slow.",
    ]
    
    texts = [p["text" if isinstance(p, dict) else "p"] for p in practices]
    # Fix the above
    texts = [p if isinstance(p, str) else p.get("text", str(p)) for p in practices]
    
    # Generate IDs and embeddings
    ids = [f"bp-{i:03d}" for i in range(len(practices))]
    embeddings = embed_text(texts)
    # Actually batch embed
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    embeddings = [item.embedding for item in response.data]
    
    # Add to collection
    collection.add(
        documents=texts,
        embeddings=embeddings,
        ids=ids
    )
    
    print(f"✓ Seeded {len(practices)} best practices into knowledge base")
    return collection


def query_relevant_practices(code_context: str, n_results: int = 3) -> list[str]:
    """Find best practices relevant to the code being reviewed"""
    collection = get_or_create_collection()
    
    query_embedding = client.embeddings.create(
        model="text-embedding-3-small",
        input=code_context
    ).data[0].embedding
    
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=n_results
    )
    
    return results['documents'][0] if results['documents'] else []
```

Now inject relevant practices into agent prompts:

```python
# In each agent's user_prompt, add:
from app.knowledge_base import query_relevant_practices

# Before sending to LLM:
relevant_practices = query_relevant_practices(diff_text)
if relevant_practices:
    user_prompt += f"""

Relevant best practices to consider:
{chr(10).join(f'- {p}' for p in relevant_practices)}
"""
```

### ✅ Check: Day 6 Gate

```
1. ❓ Run seed_knowledge_base.py — does it create the DB without errors?
2. ❓ Query with "SQL injection" — does it return the parameterized queries practice?
3. ❓ Query with "async performance" — does it return the gather() practice?
4. ❓ Does the agent prompt now include relevant practices?
5. ❓ Run the eval suite again — do all 4 tests still pass?

If all 5 pass → proceed.
```

---

## 👨‍💻 Day 7: Observability + Eval Dashboard

### Problem (Senior Engineer)

We have no idea how well PR-Buddy is actually performing. We need:
1. Full tracing of every review (Langfuse)
2. An eval dashboard showing pass/fail across test cases
3. Cost tracking per review

### What to Build

1. Add `@observe()` decorators to trace every review through Langfuse
2. Create `scripts/run_benchmark.py` that runs the eval suite and generates a report
3. Track cost per review

### Implementation

```python
# app/observability.py
from langfuse.decorators import observe, langfuse_context
from app.config import settings
from langfuse import Langfuse

langfuse = Langfuse(
    public_key=settings.langfuse_public_key,
    secret_key=settings.langfuse_secret_key,
    host=settings.langfuse_host
)

@observe(name="pr-buddy-review")
async def traced_review(parsed_diff, pr_context):
    """Traced version of the review pipeline"""
    from app.agents.orchestrator import run_multi_agent_review
    
    result = await run_multi_agent_review(parsed_diff, pr_context)
    
    # Add trace metadata
    langfuse_context.update_current_trace(
        name=f"PR-{parsed_diff.pr_number}",
        metadata={
            "repo": parsed_diff.repo,
            "pr_number": parsed_diff.pr_number,
            "files": len(parsed_diff.files),
            "findings": len(result["all_findings"]),
            "passed": result["passed"]
        },
        tags=["code-review", parsed_diff.repo]
    )
    
    return result


def score_review(trace_id: str, score: float, name: str = "review_quality"):
    """Manually score a trace"""
    langfuse.score(
        trace_id=trace_id,
        name=name,
        value=score
    )
```

```python
# scripts/run_benchmark.py
"""
Run the eval suite and generate a benchmark report.
Usage: python scripts/run_benchmark.py
"""
import json
import sys
import os
from pathlib import Path

# Add project to path
sys.path.insert(0, str(Path(__file__).parent.parent))

import pytest
import time
from datetime import datetime


def run_benchmark():
    """Run all eval tests and produce a report"""
    print("=" * 60)
    print("PR-Buddy Benchmark Suite")
    print(f"Date: {datetime.now().isoformat()}")
    print("=" * 60)
    
    start = time.time()
    
    # Run pytest programmatically
    exit_code = pytest.main([
        "-v",
        "--tb=short",
        "tests/test_evals.py"
    ])
    
    duration = time.time() - start
    
    passed = exit_code == 0
    
    report = {
        "timestamp": datetime.now().isoformat(),
        "duration_seconds": round(duration, 2),
        "passed": passed,
        "exit_code": exit_code,
        "version": "1.0.0"
    }
    
    # Save report
    report_path = Path("benchmark_results.json")
    with open(report_path, "w") as f:
        json.dump(report, f, indent=2)
    
    print(f"\n{'=' * 60}")
    print(f"Status: {'✅ PASSED' if passed else '❌ FAILED'}")
    print(f"Duration: {duration:.1f}s")
    print(f"Report saved to: {report_path}")
    
    return exit_code


if __name__ == "__main__":
    sys.exit(run_benchmark())
```

### ✅ Check: Day 7 Gate — THIS IS YOUR SHIP GATE

```
1. ❓ Run the benchmark:  python scripts/run_benchmark.py
    → All 4 eval tests must pass.
    
2. ❓ Set up Langfuse (docker compose or cloud account)
    → Traces should appear in the Langfuse dashboard.
    
3. ❓ Do a full end-to-end test:
    - Open a test PR with intentional bugs
    - PR-Buddy receives webhook
    - All 3 agents run
    - Comments posted on PR
    - Status check set
    - Trace visible in Langfuse
    - Cost tracked

4. ❓ Do a "clean PR" test:
    - Open a PR with no bugs (just refactoring)
    - No CRITICAL/MAJOR findings should be posted (MINOR/NIT is OK)
    - Status check should pass (green checkmark)

5. ❓ Edge cases:
    - Empty PR (no file changes)
    - PR with only generated files
    - Very large PR (100+ files — does it timeout?)
    - PR in unsupported language (markdown, yaml)
```

---

## 🚀 Post-Build: What We Built vs Real Products

| Feature | PR-Buddy (Your Build) | CodeRabbit | Qodo |
|---------|----------------------|------------|------|
| Line-level PR comments | ✅ | ✅ | ✅ |
| Multi-agent review (security, bug, style) | ✅ | ✅ | ✅ |
| Quality gate (block on critical) | ✅ | ✅ | ✅ |
| Best practices RAG | ✅ | ✅ | ✅ |
| Custom rules per repo | Coming next | ✅ | ✅ |
| Batch review across repos | Coming next | ✅ | ✅ |
| CI integration | ✅ | ✅ | ✅ |
| Eval suite | ✅ | Proprietary | Proprietary |
| Cost/review | ~$0.01-0.02 | Paid | Paid |

**What you built is a real competitor to products companies pay for.**

---

## 📚 Deliverables Checklist

```
[ ] Source code pushed to GitHub: pr-buddy/
[ ] README.md with architecture, setup, usage
[ ] Dockerfile + docker-compose.yml
[ ] All 4 eval tests passing
[ ] Langfuse dashboard showing traces
[ ] Demo video (2-3 min): open a PR, see review, show dashboard
[ ] Cost analysis: $/review, monthly projection for 1000 reviews
[ ] Post-mortem: what broke, what surprised you, what's next

🏆 Portfolio entry template:

  "I built PR-Buddy — an AI code review assistant that catches security
   vulnerabilities, logic bugs, and style issues before they merge.
   It uses a multi-agent LangGraph pipeline with RAG-enhanced review
   and costs ~$0.01 per PR. It caught 3/4 intentional bugs in my test
   suite with 0 critical false positives. Deployed as a GitHub App
   with Langfuse observability."
```

---

**You're not just doing a tutorial project. You built something that competes in a $12.8B market. Ship it.**
