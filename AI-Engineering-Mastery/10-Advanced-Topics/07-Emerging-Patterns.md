# Emerging Patterns

## A2A (Agent-to-Agent) Protocol

Google's open protocol for agents communicating with each other:

```python
# Agent A sends a task to Agent B
agent_a.send_task(
    agent_b_url="https://agent-b.example.com/a2a",
    task={
        "id": "task-123",
        "type": "research",
        "query": "Find latest AI news",
        "callback_url": "https://agent-a.example.com/callback",
    }
)
# Agent B processes and returns result asynchronously
```

## AI Governance

Patterns for managing AI systems at scale:

```python
class AIGovernance:
    """Centralized governance for multiple AI systems."""

    def __init__(self):
        self.policies = {
            "max_cost_per_query": 0.50,
            "allowed_models": ["gpt-4o", "claude-4", "llama-3.2"],
            "blocked_topics": ["medical diagnosis", "legal advice"],
            "retention_days": 90,
            "require_human_review": ["loan_approval", "content_moderation"],
        }

    async def check_policy(self, request: dict) -> bool:
        # Check model is allowed
        if request["model"] not in self.policies["allowed_models"]:
            return False
        # Check topic
        if self._matches_blocked(request["query"]):
            return False
        # Check cost budget
        if request["estimated_cost"] > self.policies["max_cost_per_query"]:
            return False
        return True
```

## Prompt Compression (LLMLingua)

```python
from llmlingua import PromptCompressor

compressor = PromptCompressor()

# Compress prompts while keeping key information
compressed = compressor.compress(
    prompt,  # Original prompt
    rate=0.5,  # Compress to 50% of original
    condition_in_instruction="Answer the question",
    force_tokens=["\n", ".", "!"],  # Keep these
)
# Result: compressed_prompt
# Saves 50% on tokens, loses <5% quality
```

## Speculative Decoding

Use a small "draft" model to predict tokens, then verify with the large model:

```python
# Draft model (fast, cheap): predicts 5 tokens
# Target model (slow, quality): verifies all 5 in one forward pass
# Result: 2-3x speedup with same quality

from vllm import LLM

llm = LLM(
    model="meta-llama/Llama-3.2-70B",
    draft_model="meta-llama/Llama-3.2-3B",  # Draft model
    speculative_model_mode="VLLM",
)
```

## 🔴 Senior: What to Track

| Pattern | Maturity | Should You Learn? |
|---------|----------|-------------------|
| MCP | Early production | Yes |
| A2A | Experimental | Maybe |
| Speculative decoding | Production | Yes (if self-hosting) |
| Prompt compression | Early production | Yes (cost savings) |
| AI governance | Early production | Yes (enterprise) |
| Agent evaluation | Research | Yes (hardest problem) |
