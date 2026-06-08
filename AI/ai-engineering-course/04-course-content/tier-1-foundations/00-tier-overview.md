# Tier 1: Internal Studio Tools — Foundations

> *"Get comfortable with the AI engineering workflow. No client pressure. You're building tools for your own team."*

## Why Tier 1 Exists

Before you ship code for a client who'll yell at you if it breaks, you need a safe sandbox. Tier 1 is that sandbox. You're building tools for **NexaAI's internal team** — your coworkers. If something goes wrong, you don't lose a client. You just have to face Rohan at the next standup.

**Think of this as your flight simulator before the real flight.** Every project here teaches a skill you'll use every single day as an AI engineer:

| Project | The Real Skill | Why It's Not Toy |
|---|---|---|
| **Project 1** — Prompt Testing Harness | Structured outputs, eval, systematic testing | Production teams spend 30% of their time on prompt eval. You need this skill on day 1. |
| **Project 2** — Transcript Summarization | Pydantic validation, batch processing, cost tracking | Batch AI pipelines are everywhere — support tickets, email triage, content moderation |
| **Project 3** — Semantic Search | Embeddings, vector similarity, search quality | Every RAG system in the world starts with exactly this. Build it manually first so you understand what LangChain hides. |

## What You'll Be Able to Do After Tier 1

- ✅ Call LLM APIs with proper error handling, streaming, and retries
- ✅ Return structured JSON from LLMs using Pydantic (Schema-First approach — the 2026 standard)
- ✅ Evaluate and compare prompt versions systematically (not "feels better" — measured)
- ✅ Build batch processing pipelines that handle real-world edge cases
- ✅ Implement semantic search with embeddings and understand how vector similarity works
- ✅ Track token usage and estimate costs (you'll never ship a system that burns money)
- ✅ Deploy a basic AI API with FastAPI + Docker

**Portfolio:** 3 working internal tools you can put on GitHub

## How Tier 1 Works

There's no actual client. You're building tools that the NexaAI team needs internally. This means:
- **Lower stakes** — you can break things without consequences
- **Maya is very present** — she'll guide you step by step
- **Rohan's bar is lower** — he checks that it works correctly (Points 1-2 of his 7-point bar)
- **But don't coast** — everything you build here will come back harder in Tier 2

The golden thread through all 3 projects: **Structure. Cost. Evaluation.** 
- Project 1 teaches structured outputs + eval
- Project 2 teaches structure at scale + cost tracking
- Project 3 teaches structure + search quality measurement

## Before You Start

You need basic Python. If you've written functions, used classes, and know what a dictionary is, you're ready. If you need a refresher, Maya will point you to resources when you need them — not all at once.

**Let's go.** Priya has your first ticket.
