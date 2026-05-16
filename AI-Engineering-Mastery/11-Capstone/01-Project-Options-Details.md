# Capstone Project Options — Details

## Option 1: AI Customer Support System

**Scenario**: A SaaS company gets 500 support tickets/day. Build an AI system that handles Level 1 support autonomously and escalates complex issues to humans.

### Required Components
| Phase | Component |
|-------|-----------|
| 1 | Multi-provider LLM gateway |
| 2 | Prompt templates for different query types |
| 3 | Vector DB for product knowledge |
| 4 | RAG over support docs |
| 5 | Query categorization + routing |
| 6 | Multi-agent (triage, research, respond) |
| 8 | Production deployment |
| 9 | Observability + quality monitoring |
| 10 | Guardrails (don't make promises, escalate sensitive topics) |

### Evaluation Metrics
- Auto-resolution rate: >60%
- Customer satisfaction: >4.0/5.0
- Escalation accuracy: >90%
- Response time: <30s
- Cost per ticket: <$0.10

## Option 2: AI Code Review Assistant

**Scenario**: A development team needs automated code review that catches bugs, security issues, and style violations.

### Required Components
| Phase | Component |
|-------|-----------|
| 1 | LLM integration for code analysis |
| 2 | Prompt patterns for code review |
| 3 | Embeddings for code search |
| 4 | RAG over codebase + best practices |
| 6 | Agent that reads files, runs linters, suggests fixes |
| 7 | Fine-tuned code review model (optional) |
| 8 | GitHub App deployment |
| 9 | Eval: does it catch real bugs? precision/recall |
| 10 | Security: prevent malicious code suggestions |

### Evaluation Metrics
- Bug catch rate: >30% of real bugs
- False positive rate: <20%
- Review time: <2min per PR
- Developer satisfaction: survey

## Option 3: AI Research Platform

**Scenario**: Build a system that researches any topic and produces a structured, cited report.

### Required Components
| Phase | Component |
|-------|-----------|
| 1-2 | Prompt system for research planning |
| 3 | Vector DB for storing research |
| 4-5 | RAG over documents + web search |
| 6 | Multi-agent (research, analyze, write, review) |
| 8 | API deployment |
| 9 | Eval: factuality, coverage, citation accuracy |
| 10 | Safety: prevent misuse for harmful topics |

### Evaluation Metrics
- Factual accuracy: >90%
- Citation quality: >95% valid
- Report quality: human rating >4/5
- Research time: <5min per topic
- Cost per report: <$0.50
