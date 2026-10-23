# SupportDesk

A customer-support platform built from scratch, one level at a time: a FastAPI service that grows into
a multi-agent AI system on AWS and Kubernetes.

This is a learning project with one rule: **no AI writes the code.** Editor autocomplete is fine.
Specs, tests and reviews come from an instructor; the implementation is mine.

## Where things are

| Path | What it is |
|------|------------|
| [learning-documents/PROJECT-PLAN.md](learning-documents/PROJECT-PLAN.md) | The full plan: rules, the gate system, all 17 levels |
| [learning-documents/handbook/](learning-documents/handbook/00-INDEX.md) | Reference reading: LLMs, agents, FastAPI, auth, RAG, evals, security, system design |
| `learning-documents/levels/` | One spec per level, added when the level unlocks |
| `learning-documents/learning-log.md` | What I learned and what broke, one entry per level |

## How it works

Each level has a spec, gate tests I can't edit, my own tests (80%+ coverage), break-it drills and a quiz.
A level is done when all of those pass and the PR is merged and tagged `level-NN`.
The next level doesn't unlock until then.

## Progress

**Phase A: Backend foundations**
- [ ] Level 0: Repository and tooling
- [ ] Level 1: First endpoints
- [ ] Level 2: Structure, config, errors, Docker
- [ ] Level 3: A real database
- [ ] Level 4: Authentication and authorization
- [ ] Level 5: Production backend concerns (Redis, rate limits, jobs, idempotency)

**Phase B: Ship it**
- [ ] Level 6: First deployment to AWS on ECS

**Phase C: LLMs, by hand**
- [ ] Level 7: Calling an LLM properly
- [ ] Level 8: Chat, no framework
- [ ] Level 9: RAG over the knowledge base

**Phase D: Agents, by hand**
- [ ] Level 10: One agent from scratch
- [ ] Level 11: Evals and observability
- [ ] Level 12: Multiple agents
- [ ] Level 13: Expose tools over MCP

**Phase E: Frameworks**
- [ ] Level 14: Rebuild with LangGraph and LlamaIndex, compare with evals

**Phase F: Kubernetes**
- [ ] Level 15: Kubernetes locally (kind, Helm)
- [ ] Level 16: EKS in production
- [ ] Level 17: Capstone

## Stack

Python · FastAPI · Pydantic · PostgreSQL + pgvector · SQLAlchemy · Alembic · Redis · pytest ·
Docker · Terraform · AWS (ECS, RDS, ElastiCache, EKS) · Kubernetes · Helm · GitHub Actions ·
OpenTelemetry · LangGraph · LlamaIndex

## Getting started

Setup instructions arrive with Level 0.
