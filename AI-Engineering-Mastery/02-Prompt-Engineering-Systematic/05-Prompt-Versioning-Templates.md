# Prompt Versioning & Templates

## Prompts Are Code

Treat prompts with the same rigor as source code:
- Version controlled (git)
- Code reviewed (PRs)
- Tested (CI pipeline)
- Deployed (canary → staged → full rollout)
- Rolled back (when quality drops)

## Prompt Template System

```python
from string import Template
import hashlib
from datetime import datetime

class PromptTemplate:
    def __init__(self, name: str, content: str, version: str,
                 author: str, description: str = ""):
        self.name = name
        self.content = content
        self.version = version
        self.author = author
        self.description = description
        self.created_at = datetime.utcnow()
        self.hash = hashlib.sha256(content.encode()).hexdigest()[:12]

    def render(self, **kwargs) -> str:
        return Template(self.content).safe_substitute(**kwargs)

class PromptRegistry:
    def __init__(self, storage_path: str = "prompts/"):
        self.storage_path = Path(storage_path)
        self.templates: dict[str, list[PromptTemplate]] = {}

    def get(self, name: str, version: str = "latest") -> PromptTemplate:
        versions = self.templates.get(name, [])
        if version == "latest":
            return versions[-1]
        return next(v for v in versions if v.version == version)

    def register(self, template: PromptTemplate):
        self.templates.setdefault(template.name, []).append(template)
        self._save(template)
```

## File Structure

```
prompts/
├── classifier/
│   ├── v1.0.0.yaml
│   ├── v1.1.0.yaml
│   └── v2.0.0.yaml
├── summarizer/
│   ├── v1.0.0.yaml
│   └── v1.0.1.yaml
└── registry.json
```

## Prompt Template YAML

```yaml
name: email_classifier
version: 2.0.0
author: "user@example.com"
description: "Classifies emails as spam or not-spam"
model: gpt-4o-mini
temperature: 0.0
max_tokens: 50

system: |
  You are an email classifier. Classify the following email
  as either "spam" or "not_spam". Return only the label.

template: |
  Email: {email_text}
  Classification:

tests:
  - input: {email_text: "Buy now! Limited offer!"}
    expected: "spam"
  - input: {email_text: "Meeting at 3pm tomorrow"}
    expected: "not_spam"
```

## 🔴 Senior: Deployment Strategy

```
dev → canary (5% traffic) → staged (50%) → prod (100%)
                                    ↓
                              If metrics drop → auto-rollback
```

## Drill: Prompt Registry API

Build a FastAPI service that:
1. Stores prompt templates with versions
2. Renders prompts with variables
3. Executes test suites on registration
4. Has a rollback endpoint
5. Logs which prompt version was used per request (for audit)
