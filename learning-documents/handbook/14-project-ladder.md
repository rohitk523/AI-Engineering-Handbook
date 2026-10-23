# 14 — Hands-On Project Ladder

[← 13 System design](13-system-design-for-ai.md) · [Index](00-INDEX.md)

Reading isn't enough. Each project builds on the previous one, so by the end you have one portfolio repo that shows the whole stack. Put each in Git with a README, tests and a short "what I learned".

---

### Project 1: Raw agent loop (≈3 hrs) → [02](02-tool-calling-and-agent-loop.md)
- An OpenAI-compatible client (Groq free tier or a local Ollama model), 3–4 tools in any domain (expenses, unit conversions, a to-do list).
- Handles parallel tool calls, tool errors, `MAX_STEPS`, logging.
- **Stretch:** port the same loop to the Anthropic Messages API and note the differences.
- **Done when** you can rewrite it from a blank file in 10 minutes.

### Project 2: Agent as a FastAPI service (≈5 hrs) → [06](06-fastapi-fundamentals.md)
- `POST /chat` with SSE streaming, `lifespan`-managed async client, Pydantic schemas, settings via env.
- pytest with a mocked LLM client and `httpx.AsyncClient`.
- Docker + docker compose.

### Project 3: Auth, persistence and limits (≈6 hrs) → [07](07-fastapi-auth.md), [08](08-serving-llm-apps.md)
- Users with Argon2 hashes, JWT access + rotating refresh tokens, API keys (hashed).
- Postgres via SQLAlchemy async + Alembic: users, conversations, messages, tool_calls.
- Redis rate limiting (per user, token-aware). Object-level authorization tests.

### Project 4: Your own MCP server (≈4 hrs) → [04](04-mcp-and-protocols.md)
- Expose project 3's domain (or a CV model!) as MCP tools + resources with FastMCP.
- Test with MCP Inspector; connect it to Claude Desktop/Code or your own agent.
- **Stretch:** remote HTTP transport behind OAuth.

### Project 5: Hybrid RAG with evals (≈8 hrs) → [05](05-rag-and-retrieval.md)
- Ingest 50–200 PDFs (use a layout/VLM parser for scanned ones), pgvector + BM25, RRF, reranker, citations.
- A 50-question labeled set: measure recall@k and faithfulness. Compare chunking strategies with numbers.

### Project 6: Evals + tracing for everything (≈5 hrs) → [09](09-evals-observability.md)
- Eval suite for projects 1–5 with multiple trials, run in GitHub Actions on PRs.
- OpenTelemetry/Langfuse tracing; a dashboard for latency, cost and tool errors.

### Project 7: Multimodal agent (your CV edge) (≈10 hrs) → [11](11-genai-landscape-2026.md)
Pick one:
- **Invoice/receipt agent**: VLM + a detector/OCR extract line items → validate with deterministic tools → store → answer questions.
- **Visual inspection agent**: your detection model as a tool; the LLM explains defects, queries history and files tickets (with human approval).
- **Browser agent** for a narrow task, with screenshots + DOM, in a sandbox.

### Project 8: Background multi-step agent (≈10 hrs) → [03](03-agent-architectures.md), [08](08-serving-llm-apps.md), [10](10-security-for-agents.md)
- `POST /runs` → queue → worker with durable checkpoints (LangGraph or Temporal), progress over SSE.
- Orchestrator + 2–3 subagents (for example a research agent with web search, a writer, a critic).
- Budgets, approval gates, a prompt-injection test case in your eval suite.

---

## Portfolio framing

One repo, one architecture diagram, one README that says:
*"Production-style agent platform: FastAPI + Postgres/pgvector + Redis + queue, JWT/API-key auth, MCP tools, hybrid RAG, multimodal document parsing, evals in CI, OpenTelemetry tracing."*
That one line covers most of what senior AI engineer job descriptions ask for in 2026.
