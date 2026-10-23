# AI Engineering Handbook — From Computer Vision to Agentic Systems

> Written October 2026 for a senior AI engineer coming from computer vision.
> Goal: understand how agents, GenAI apps and their backends are built today, and
> know which fundamentals (Python, DSA, backend) to refresh.

---

## How to use this folder

Read it in order the first time. Each file has three parts:

1. **Mental model**: how the thing works, in a few paragraphs.
2. **How engineers actually do it**: the production patterns, code and trade-offs.
3. **Check yourself**: questions you should be able to answer in an interview or design review.

Files marked ⭐ are the core of "agentic engineering". If you are short on time, read those first.

---

## Table of contents

### Part 1: Foundations
| # | File | What you get |
|---|------|--------------|
| 01 | [LLM fundamentals for a CV engineer](01-llm-fundamentals.md) | Tokens, sampling, context windows, reasoning models, inference (KV cache, quantization, vLLM), mapped to ideas you already know from CV |
| 02 ⭐ | [Tool calling and the agent loop](02-tool-calling-and-agent-loop.md) | The single most important pattern in agentic AI. The message protocol, the loop and its failure modes |

### Part 2: Agentic systems
| # | File | What you get |
|---|------|--------------|
| 03 ⭐ | [Agent architectures and context engineering](03-agent-architectures.md) | Workflows vs agents, ReAct, orchestrator–workers, multi-agent, memory, subagents, skills, harnesses |
| 04 ⭐ | [MCP, A2A and agent protocols](04-mcp-and-protocols.md) | How tools are plugged into agents in a standard way, and what changed in 2026 |
| 05 | [RAG and retrieval, 2026 edition](05-rag-and-retrieval.md) | Embeddings, hybrid search, reranking, agentic RAG, when long context replaces RAG |

### Part 3: Backend engineering for AI
| # | File | What you get |
|---|------|--------------|
| 06 ⭐ | [FastAPI: how it actually works](06-fastapi-fundamentals.md) | ASGI, async, Pydantic v2, dependency injection, lifespan, streaming, testing, project layout |
| 07 ⭐ | [Authentication and authorization in FastAPI](07-fastapi-auth.md) | OAuth2, JWT, password hashing, API keys, scopes/RBAC, refresh tokens, the common mistakes |
| 08 | [Serving LLM apps in production](08-serving-llm-apps.md) | Streaming, queues, retries, rate limits, caching, cost control, deployment |

### Part 4: Making it reliable
| # | File | What you get |
|---|------|--------------|
| 09 ⭐ | [Evals, observability and guardrails](09-evals-observability.md) | How to know if an agent works: eval sets, LLM-as-judge, tracing, regression testing |
| 10 | [Security for agents](10-security-for-agents.md) | Prompt injection, the "lethal trifecta", sandboxing, least privilege, OWASP LLM Top 10 |

### Part 5: Career and keeping up
| # | File | What you get |
|---|------|--------------|
| 11 | [The GenAI landscape in late 2026](11-genai-landscape-2026.md) | Model families, frameworks, architectural trends, where CV fits, and how to keep up |
| 12 | [Python and DSA brush-up for backend and AI roles](12-python-and-dsa.md) | What DSA actually matters, Python internals (GIL, asyncio, free-threading), a 4-week drill plan |
| 13 | [System design for AI systems](13-system-design-for-ai.md) | A template plus three worked designs (support agent, document QA, coding agent) |
| 14 | [Hands-on project ladder](14-project-ladder.md) | Eight projects from a 50-line agent loop up to a deployed multi-agent service |

---

## Suggested 8-week plan (about 8–10 hours a week)

| Week | Read | Build |
|------|------|-------|
| 1 | 01, 02 | Project 1: raw tool-calling loop with no framework (any domain) |
| 2 | 06, 12 (Python part) | Project 2: wrap the agent in a FastAPI service with streaming |
| 3 | 07, 08 | Project 3: add JWT auth, API keys, rate limiting and Postgres |
| 4 | 03, 04 | Project 4: write an MCP server and connect it to an agent |
| 5 | 05 | Project 5: hybrid-search RAG over your own documents, with a reranker |
| 6 | 09, 10 | Project 6: an eval suite and tracing for projects 1–5 |
| 7 | 13, 12 (DSA part) | Two system-design mocks and 15 LeetCode mediums from the list in 12 |
| 8 | 11, 14 | Project 7 or 8: multimodal (your CV advantage) or multi-agent |

---

## The one-page summary (if you read nothing else)

1. **An agent is a loop**: call model → model asks for a tool → you run it → feed the result back → repeat until it answers. Everything else (frameworks, MCP, multi-agent) is built around this loop. See [02](02-tool-calling-and-agent-loop.md).
2. **Context engineering matters more than prompt engineering**: what goes into the context window (tools, retrieved docs, memory, previous steps) and what stays out decides quality, cost and latency. See [03](03-agent-architectures.md).
3. **Standard protocols won**: MCP is how tools and data plug into agents. Know how it works and how to write a server. See [04](04-mcp-and-protocols.md).
4. **The backend is ordinary, and it has to be good**: FastAPI + Pydantic + Postgres + Redis + a queue is the default Python stack. Auth, idempotency, retries and streaming are where AI products break. See [06](06-fastapi-fundamentals.md)–[08](08-serving-llm-apps.md).
5. **Evals are the new unit tests**: you can't ship an agent you can't measure. Seniors are expected to design eval sets and tracing from day one. See [09](09-evals-observability.md).
6. **Security is mostly about tools**: an agent that reads untrusted content, has access to private data, and can send data out can be hijacked by prompt injection. Design permissions around that. See [10](10-security-for-agents.md).
7. **Your CV background is an asset**: vision-language models, document AI, computer-use agents and robotics/physical AI all need people who understand vision. See [11](11-genai-landscape-2026.md).

---

## A note on freshness

This field moves monthly. Concepts (the loop, context engineering, auth, evals, system design) stay stable for years.
Specific model names, versions and benchmark numbers go stale fast, so this handbook avoids them where possible.
When a fact is time-sensitive it is marked **(as of 2026, verify)**. See the "How to keep up" section of [11](11-genai-landscape-2026.md).
