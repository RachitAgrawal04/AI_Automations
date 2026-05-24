# AI Automations

A personal showcase of my **AI automation projects**—from **lead generation workflows** and **MCP servers** to **RAG + evaluations**, and other practical systems that combine LLMs with real-world tooling.

## Goals

- Build end-to-end, production-minded AI automations (not just demos)
- Learn reliable patterns for tool use, orchestration, and evaluation
- Create reusable templates for future projects (agents, RAG, pipelines)
- Document what works (and what doesn’t) so the repo stays useful over time

## What I’m building here

- **Lead generation automations**
  - Enrichment, scoring, outreach drafts, CRM-ready outputs
  - Human-in-the-loop flows and safety guardrails
- **MCP servers / tool providers**
  - Exposing data/tools to agents in a structured way
  - Local + remote integrations (APIs, files, DBs)
- **RAG (Retrieval-Augmented Generation)**
  - Ingestion pipelines, chunking, embeddings, retrieval strategies
  - Structured citations + traceability
- **Evals & quality gates**
  - Regression tests for prompts and pipelines
  - Automated evaluation + error analysis
- **Agentic workflows**
  - Multi-step planning/execution loops
  - Tool calling, memory patterns, and observability

## What I expect to learn

- When to use **agents** vs. **workflows** vs. traditional scripts
- How to design **tool interfaces** that are reliable and testable
- RAG fundamentals: chunking, retrieval, reranking, grounding, and caching
- How to build **evals** that prevent regressions as prompts/tools evolve
- Debugging + observability techniques for LLM apps (traces, logs, metrics)
- Secure handling of secrets, API keys, and data privacy constraints

## Tech stack (typical)

This repo may contain projects using some combination of:

- **Python** (automation scripts, pipelines, evals)
- **TypeScript/Node.js** (servers, MCP tooling, integrations)
- **LLM providers** (OpenAI / others depending on project)
- **RAG tooling** (vector DBs / embedding pipelines, retrievers, rerankers)
- **Workflow orchestration** (scripts, queues, cron, or pipeline tools)
- **Testing & evals** (pytest, custom eval harnesses, golden tests)
- **Dev tooling**
  - `pre-commit`, formatting/linting (ruff/black/eslint), CI
  - Docker where reproducibility matters

## How to navigate

- Each project should live in its own folder with:
  - a small README explaining *what it does*
  - setup instructions
  - example inputs/outputs
  - eval methodology (when applicable)

## Conventions

- Prefer **shared components/utilities** over copy-pasting repeated logic
- Keep examples small but realistic
- Add short notes when a decision is non-obvious (tradeoffs, pitfalls)

## Roadmap (high level)

- [ ] Add first end-to-end lead gen workflow (with evals)
- [ ] Add an MCP server example with at least 2 tool integrations
- [ ] Add a RAG project with traceable citations and regression evals
- [ ] Add CI checks (lint/format/tests) for consistency

---

If you’re interested in any specific project, open an issue or reach out—happy to share details and learnings.
