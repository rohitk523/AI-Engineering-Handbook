# SupportDesk: The Build Plan

> One real project, built level by level, by hand.
> Status: **DRAFT, waiting for approval.** Nothing below is final until you approve it.

---

## 1. What we are building

**SupportDesk** (working name, rename it if you like) is a customer-support platform. It starts as a plain
REST API and grows into a multi-agent system running on Kubernetes.

Why a helpdesk? It gives every level a real reason to exist:

| Need | Why the helpdesk needs it |
|------|---------------------------|
| CRUD, validation, pagination | Tickets, comments, knowledge-base articles |
| A state machine | Ticket status: `open → in_progress → waiting → resolved → closed` |
| Real authorization | Customers see only their own tickets, agents see assigned ones, admins see everything |
| Background jobs | SLA breach checks, email notifications, document ingestion |
| Caching and rate limits | Public KB search gets hammered |
| An LLM | Summarize a ticket, classify priority, draft a reply |
| Chat | A customer asks questions in a chat window |
| RAG | Answers must come from the knowledge base, with citations |
| One agent | Look up the ticket, search the KB, draft a reply, propose a status change |
| Many agents | Triage agent → research agent → writer agent → critic agent |
| Frameworks | Rebuild the agents in LangGraph / LlamaIndex and compare against your hand-written version |
| Deployment | Docker → AWS ECS → Kubernetes (local kind cluster, then EKS) |

By the end you have one repository that covers most of what a senior AI/backend engineer is expected to know in 2026.

---

## 2. The rules (the honor code)

These are what make this work. If you break them, you are only fooling yourself.

1. **No AI writes or completes your code.** Turn off Copilot, Claude inline suggestions, Cursor Tab and similar
   tools for this workspace. Language-server autocomplete (Pylance/Pyright: names, signatures, imports) is allowed.
   That is the "autocomplete like the editor has" and nothing more.
2. **Allowed sources:** official docs, library source code, books, `--help`, `python -m pydoc`, Stack Overflow,
   GitHub issues. Skip AI-generated search summaries.
3. **What Claude does in this project.** Claude is the instructor and examiner, not the pair programmer:
   - writes the **spec** for each level,
   - writes the **gate tests** (black-box tests you must pass and may not edit),
   - gives **hints when you ask**, in escalating levels (see section 4), never a full solution,
   - **quizzes you** before a level unlocks,
   - **reviews** your code only *after* the gate passes.
4. **You cannot skip a level.** The next level's spec is written only after the current gate is passed.
5. **Stuck rule.** Struggle for 45 minutes first: read the error, read the docs, read the source. Then ask for a hint.
6. **Write it down.** Every level ends with an entry in `learning-documents/learning-log.md`: what broke,
   what you didn't know, what you would do differently.

---

## 3. How a level works (the check system)

Every level has the same structure and the same gate. A level is **passed** only when all six checks are green.

```
spec ──► you build ──► your own tests ──► gate tests ──► break-it drills ──► quiz ──► PR merged + tag
```

| # | Check | How it's verified |
|---|-------|-------------------|
| 1 | **Gate tests pass** | `make gate LEVEL=NN` runs the tests in `gates/level_NN/` against your running app. 100% pass, no edits to gate files (CI checks their hash). |
| 2 | **Your own tests** | You write unit and integration tests. Coverage ≥ 80%, enforced in CI. |
| 3 | **Quality checks** | `ruff` (lint + format), `pyright` (types), all in CI. No `# type: ignore` without a comment explaining why. |
| 4 | **Break-it drills** | Each level lists failures to cause on purpose (kill the DB mid-request, send a 10 MB body, expire a token…). You write down what happened and fix whatever needs fixing. |
| 5 | **Quiz** | Claude asks 5–8 questions about the level. You answer without looking anything up. A weak answer sends you back to read, not forward. |
| 6 | **Shipped** | A PR per level, CI green, merged to `main`, tagged `level-NN`. From level 6 on, it is also deployed. |

The gate tests are contract tests: they talk to your API over HTTP, so they don't care how you wrote the code,
only that it behaves correctly. They will include nasty cases (wrong types, missing auth, someone else's ticket,
concurrent requests). That is where the gaps in your fundamentals will show up.

---

## 4. Hint ladder

When you're stuck, ask for a hint at a specific level. You start at 1.

| Level | What you get |
|-------|--------------|
| 1 | Which concept is involved ("this is about transaction isolation") |
| 2 | Which doc page or source file to read |
| 3 | What the shape of the solution looks like, in words |
| 4 | Pseudocode for the tricky part only |

There is no level 5. You never get the code.

---

## 5. The stack (decided, with reasons)

| Concern | Choice | Why |
|---------|--------|-----|
| Language | Python 3.13+ | Check library support before moving to newer versions |
| Packaging | `uv` | Fast, lockfile, the current standard |
| Lint / format / types | `ruff`, `pyright` | Industry default |
| Web | FastAPI + Pydantic v2 | Covered in handbook 06 |
| Server | Uvicorn | Plain and standard |
| Database | PostgreSQL 17 + pgvector | One database for relational data and vectors |
| ORM / migrations | SQLAlchemy 2.0 (async) + asyncpg, Alembic | What most teams use |
| Cache / queue / rate limits | Redis | One tool, three jobs |
| Background jobs | Hand-written Redis queue first, then `arq` | Build it once so you understand what the library does |
| Tests | pytest, pytest-asyncio, httpx, testcontainers | Real Postgres in tests, not SQLite |
| LLM | Raw HTTP first (`httpx`), then official SDKs. Ollama locally for free runs | Provider-agnostic interface |
| Observability | structured logs, OpenTelemetry | Vendor-neutral |
| Containers | Docker, docker compose | Local dev for levels 2–5 |
| Infra as code | Terraform | Most widely used, same tool for ECS and EKS |
| AWS | ECR, ECS Fargate, RDS, ElastiCache, ALB, Secrets Manager, then EKS | |
| Kubernetes | kind (local) → EKS, Helm | Learn on your laptop for free, then pay for EKS |
| CI/CD | GitHub Actions | Lint, types, tests, gate, build, deploy |

---

## 6. The levels

The order is deliberate: **backend foundations → ship to the cloud early → LLM → chat → RAG → agents → evals →
multi-agent → frameworks → Kubernetes.** Deploying at level 6, before any AI, means every later level is shipped
to a real environment instead of being "done on my laptop".

Each level lists: what you build, the fundamentals it forces you to relearn, the gate, and the break-it drills.
Time estimates assume evenings and weekends without AI help, so expect to go over them.

---

### Phase A: Backend foundations

#### Level 0: Repository and tooling (≈1 evening)
- **Build:** repo layout, `uv` project, `ruff`, `pyright`, `pytest`, `pre-commit`, a `Makefile`
  (`make dev`, `make test`, `make lint`, `make gate`), `.env.example`, GitHub Actions running lint + types + tests on every PR,
  branch protection on `main`.
- **Fundamentals:** virtual environments and how imports resolve, `pyproject.toml`, git branching and PRs.
- **Gate:** CI green on an empty PR; `make` targets work on a fresh clone.
- **Break it:** push a lint error and watch CI block the merge.

#### Level 1: First endpoints (≈1 weekend)
- **Build:** `GET /health`. Tickets CRUD with an **in-memory** store: create, get, list, update (partial), delete.
  List supports filtering (`status`, `priority`), sorting and offset pagination. Ticket status follows the state machine
  (invalid transitions return `409`).
- **Fundamentals:** HTTP methods and idempotency, status codes (`200/201/204/400/404/409/422`), path vs query vs body,
  Pydantic validation, `PUT` vs `PATCH`, OpenAPI.
- **Gate:** contract tests for every endpoint, including bad input and illegal transitions.
- **Break it:** send a string where an int is expected, a negative page, a 1 MB title. Restart the server and see the data vanish (why that's a problem is level 3).

#### Level 2: Structure, config, errors, Docker (≈1 weekend)
- **Build:** split into routers / services / schemas. Settings with `pydantic-settings`. A single error format for the
  whole API. Request-ID middleware. Structured JSON logs. Dependency injection for the store. Dockerfile
  (multi-stage, non-root user, small image) and `docker compose up`.
- **Fundamentals:** layering and why business logic stays out of routes, 12-factor config, middleware order,
  how `Depends` works, Docker layers and caching.
- **Gate:** every error response matches the envelope; every response has `X-Request-ID`; image under a size budget; runs in compose.
- **Break it:** start with a missing env var (should fail fast at startup, not on the first request); raise an
  unexpected exception (no stack trace should leak to the client).

#### Level 3: A real database (≈1–2 weekends)
- **Build:** Postgres in compose. SQLAlchemy async models for users, tickets, comments, tags (many-to-many).
  Alembic migrations. Repository pattern. **Cursor pagination** replacing offset. Indexes for your list queries.
  Tests against a real Postgres via testcontainers.
- **Fundamentals:** SQL joins, constraints, indexes and `EXPLAIN ANALYZE`, transactions and isolation levels,
  the N+1 problem, connection pools, async sessions, why offset pagination degrades.
- **Gate:** contract tests from level 1 still pass on Postgres; pagination is stable while rows are inserted; a
  migration round-trips (`upgrade → downgrade → upgrade`).
- **Break it:** two requests change the same ticket's status at the same moment (lost update: fix it with row locks
  or optimistic versioning). Kill Postgres during a request. Load 100k tickets and find the slow query.

#### Level 4: Authentication and authorization (≈1–2 weekends)
- **Build:** registration and login, Argon2 password hashing, JWT access tokens plus rotating refresh tokens
  (stored, revocable), roles `customer` / `agent` / `admin`, **object-level** rules (customers only see their own
  tickets), hashed API keys for machine clients.
- **Fundamentals:** authentication vs authorization, hashing vs encryption, how JWTs are signed and why they can't
  be "logged out", token storage, timing attacks, OWASP API Top 10 (BOLA/IDOR). Handbook 07.
- **Gate:** a full permission matrix (every role × every endpoint × own/other's resource); reused refresh token
  revokes the whole token family; tampered or expired tokens are rejected.
- **Break it:** change a ticket ID in the URL to someone else's. Replay an old refresh token. Brute-force the login (leads into level 5).

#### Level 5: Production backend concerns (≈2 weekends)
- **Build:** Redis in compose. **Token-bucket rate limiter written by you** (per user and per IP). Cache for public KB
  reads with invalidation on write. **Idempotency keys** on `POST /tickets`. A **hand-written Redis job queue** and
  worker for notifications and SLA checks (retries, backoff, a dead-letter list), then swap to `arq` and compare.
  KB article tags get a **prefix autocomplete endpoint** backed by a trie you implement. Liveness and readiness
  probes. Graceful shutdown.
- **Fundamentals, including the DSA that shows up naturally:** token bucket vs sliding window, LRU/TTL caching,
  cache stampede, at-least-once delivery and why jobs must be idempotent, heaps (SLA ordering), tries, signals and
  graceful shutdown.
- **Gate:** limiter returns `429` with `Retry-After`; duplicate idempotent POST returns the original response;
  a job that fails 3 times lands in the dead-letter list; readiness fails when the DB is down.
- **Break it:** kill the worker mid-job. Restart Redis. Fire 500 concurrent requests and check the limiter is still correct.

---

### Phase B: Ship it

#### Level 6: First deployment to AWS on ECS (≈2 weekends)
- **Build:** set an **AWS Budget alarm first.** Terraform for VPC, ECR, ECS Fargate (API service and worker service),
  RDS Postgres, ElastiCache Redis, ALB with HTTPS, Secrets Manager. GitHub Actions: build → push to ECR → run
  migrations → deploy, using OIDC (no long-lived AWS keys in GitHub). CloudWatch logs.
- **Fundamentals:** VPCs, subnets, security groups, IAM roles vs users, task roles, how a load balancer health check
  works, rolling deploys, why migrations run as a separate step.
- **Gate:** the gate suite passes against the deployed URL; a merge to `main` deploys with zero downtime;
  `terraform destroy` and `terraform apply` rebuild everything.
- **Break it:** deploy a version that fails its health check (ECS should roll back). Rotate the DB password.
- **Cost note:** ALB, RDS and NAT gateways cost money even when idle. Destroy the stack when you aren't using it.
  Verify current pricing before you start.

From this level on, **every level ends with a deploy.**

---

### Phase C: LLMs, by hand

#### Level 7: Calling an LLM properly (≈1 weekend)
- **Build:** a provider interface (a `Protocol`) with two implementations: one written with **raw `httpx`** calls to
  an HTTP API, and one local (Ollama). Features: `POST /tickets/{id}/summary`, auto-classify priority on create
  with **structured output** validated by Pydantic (retry on invalid JSON). Timeouts, retries with jittered backoff,
  token and cost accounting per request, stored in the DB. A fake provider for tests.
- **Fundamentals:** what's in an API request (messages, roles, temperature, max tokens), tokens and context limits,
  why LLM output must be validated like any untrusted input, exponential backoff. Handbook 01.
- **Gate:** tests pass with the fake provider and no network; malformed model output never reaches the DB;
  a provider timeout returns a clean `503`/`504`, not a hang.
- **Break it:** point the provider at a server that sleeps 60 s. Return invalid JSON. Exceed the rate limit.

#### Level 8: Chat, no framework (≈1–2 weekends)
- **Build:** conversations and messages tables. `POST /conversations/{id}/messages` that **streams** the reply with
  Server-Sent Events. Context-window management: a token budget, truncation, then rolling summarization of old turns.
  Cancel on client disconnect. A small vanilla HTML/JS chat page (no frontend framework) to see it work.
- **Fundamentals:** async generators, SSE vs WebSockets, backpressure, what happens to a running request when the
  client disconnects, why conversation history is state you own.
- **Gate:** first token arrives within a time budget; disconnect stops generation and is logged; long conversations
  stay under the token budget; users can't read each other's conversations.
- **Break it:** close the tab mid-stream. Send 200 messages to one conversation.

#### Level 9: RAG over the knowledge base (≈2 weekends)
- **Build:** ingestion job (chunking with overlap), embeddings into pgvector, Postgres full-text search, **hybrid
  retrieval with Reciprocal Rank Fusion that you implement**, answers with citations to KB article IDs. A labeled
  set of ~30 questions and a script that reports recall@k.
- **Fundamentals:** embeddings and cosine similarity, ANN indexes (HNSW) vs exact search, BM25 intuition, RRF,
  chunking trade-offs, measuring retrieval separately from generation. Handbook 05.
- **Gate:** recall@5 above a threshold you set and justify; every answer cites sources that exist; re-ingesting an
  article replaces its chunks instead of duplicating them.
- **Break it:** ask something the KB doesn't cover (it should say so, not invent). Put an instruction inside a KB
  article ("ignore previous instructions…") and see what happens.

---

### Phase D: Agents, by hand

#### Level 10: One agent from scratch (≈2 weekends)
- **Build:** the tool-calling loop yourself. Tools: `search_kb`, `get_ticket`, `list_ticket_history`,
  `draft_reply`, `propose_status_change`. Tool schemas generated from Pydantic models. Max steps, per-run token
  budget, tool error handling (errors go back to the model, not up as 500s), parallel tool calls. **Approval
  gate:** anything that changes data needs a human `approve` call. Every step persisted as a trace.
- **Fundamentals:** the agent loop (handbook 02), tool design, least privilege, why the loop needs a stop
  condition, how to make a run resumable.
- **Gate:** the agent resolves a set of scripted scenarios using a fake model that returns tool calls; it stops at
  max steps; it never changes a ticket without approval; a customer's agent can't read another customer's tickets
  through a tool.
- **Break it:** a tool that throws, a tool that hangs, a model that calls a tool that doesn't exist, a prompt
  injection hidden in a ticket body.

#### Level 11: Evals and observability (≈1–2 weekends)
- **Build:** an eval set for the chat, RAG and agent features. Deterministic checks first, an LLM-as-judge with a
  written rubric second, multiple trials per case. OpenTelemetry tracing across API → LLM → tools → DB.
  A GitHub Actions job runs evals on PRs that touch prompts or agent code and fails on regression.
- **Fundamentals:** why evals replace unit tests for model behavior, judge bias, variance and trials,
  traces vs logs vs metrics. Handbook 09.
- **Gate:** a deliberately worsened prompt fails CI; a trace shows every step of one agent run with latency and cost.
- **Break it:** change the model and compare scores; find a case where the judge is wrong.

#### Level 12: Multiple agents (≈2–3 weekends)
- **Build:** `POST /runs` creates a background run handled by the worker. An orchestrator delegates to a triage
  agent, a research agent (RAG), a writer and a critic. The workflow is a **DAG you execute yourself** (topological
  order, parallel branches, retries per node). Durable checkpoints so a crashed worker resumes. Progress streamed
  over SSE. Budgets per run and per agent. Human approval before anything is sent to a customer.
- **Fundamentals:** topological sort, fan-out/fan-in, idempotent steps, checkpointing, when multi-agent is worth it
  and when one agent is better. Handbooks 03, 08, 10.
- **Gate:** kill the worker mid-run and it resumes without repeating finished steps; budget exhaustion ends the
  run cleanly; evals show whether multi-agent beats the level 10 agent (either answer is fine, with numbers).
- **Break it:** make the critic reject everything (infinite loop?). Two workers pick up the same run.

#### Level 13: Expose tools over MCP (≈1 weekend)
- **Build:** an MCP server for SupportDesk tools and KB resources, with auth. Connect it to your own agent and to an
  MCP client.
- **Fundamentals:** MCP transports, tools vs resources vs prompts, auth for remote servers. Handbook 04.
- **Gate:** your agent works the same through MCP as through direct tool calls; unauthenticated calls are refused.

---

### Phase E: Frameworks

#### Level 14: Rebuild with LangGraph and LlamaIndex (≈2 weekends)
- **Build:** reimplement level 12's workflow in LangGraph and level 9's retrieval in LlamaIndex, behind the same
  interfaces, switchable by config.
- **The point:** you now know what these frameworks do under the hood. Run the **same eval set** on your version
  and the framework version and write a short decision record: quality, latency, cost, lines of code, how easy
  it is to debug.
- **Gate:** both implementations pass the gate suite; the decision record exists and uses numbers.

---

### Phase F: Kubernetes

#### Level 15: Kubernetes locally (≈2 weekends)
- **Build:** a kind cluster. Deployments for API and worker, Services, ConfigMaps, Secrets, liveness/readiness
  probes, resource requests and limits, a migration Job, an Ingress, HPA on the API. Package it as a Helm chart.
- **Fundamentals:** pods vs deployments vs services, how probes decide traffic, requests vs limits and OOMKills,
  rolling updates, why the worker scales differently from the API.
- **Gate:** the gate suite passes against the kind cluster; `kubectl delete pod` on a running API pod causes no
  failed requests; a bad image rolls back.

#### Level 16: EKS in production (≈2–3 weekends)
- **Build:** EKS with Terraform, AWS Load Balancer Controller, IAM for pods (Pod Identity or IRSA), External
  Secrets, the Helm chart from level 15, a GitHub Actions deploy (or Argo CD for GitOps). Autoscaling, dashboards,
  alerts.
- **Fundamentals:** the EKS control plane vs nodes, Fargate vs managed node groups, pod-level IAM, cluster costs.
- **Gate:** gate suite passes on EKS; a load test (k6 or Locust) at a target RPS meets your latency SLO;
  a written runbook for "API is down" and "costs spiked".
- **Cost note:** EKS charges for the control plane per hour plus nodes. Destroy it between sessions.

#### Level 17: Capstone (open-ended)
Security review against OWASP API and LLM Top 10, a system-design write-up of the whole thing (handbook 13), an
architecture diagram, and a README that would get you an interview.

---

## 7. Fundamentals coverage map

These are the gaps this project is designed to expose. If you get through all 17 levels, each one has come up
in a context where it mattered.

| Area | Where it comes up |
|------|-------------------|
| HTTP and REST semantics | 1, 2, 5 |
| Python: typing, async, generators, context managers, Protocols | 1, 2, 7, 8 |
| SQL, indexes, transactions, concurrency | 3, 4, 9 |
| Security: hashing, tokens, authorization, injection | 4, 9, 10, 13 |
| Data structures and algorithms | token bucket (5), trie (5), heap (5), cursor pagination (3), RRF ranking (9), topological sort (12) |
| Distributed systems: retries, idempotency, at-least-once, checkpoints | 5, 7, 12 |
| Testing: unit, integration, contract, evals | every level, 11 |
| Cloud and infra: networking, IAM, containers, orchestration | 2, 6, 15, 16 |
| LLM engineering: context, tools, RAG, agents, evals | 7–14 |

---

## 8. Repository layout (target)

```
supportdesk/
├── README.md
├── Makefile
├── pyproject.toml / uv.lock
├── docker-compose.yml
├── Dockerfile
├── src/supportdesk/        # your code (you write all of it)
├── tests/                  # your tests
├── gates/level_NN/         # gate tests from Claude (do not edit)
├── migrations/             # Alembic
├── infra/terraform/        # level 6+
├── deploy/helm/            # level 15+
├── .github/workflows/
└── learning-documents/
    ├── PROJECT-PLAN.md     # this file
    ├── learning-log.md     # your notes, one entry per level
    ├── levels/             # one spec per level, added as you unlock them
    └── handbook/           # reference reading (00–14)
```

---

## 9. Open questions for you before we start

1. **Name:** keep "SupportDesk" or choose your own?
2. **LLM provider for levels 7+:** a paid API (which one?), local only (Ollama), or both? The plan assumes both.
3. **AWS:** do you already have an account you can use, and a monthly budget you want to stay under?
4. **Pace:** how many hours a week? This sets how big each level spec is.
5. **Gate tests:** are you happy for Claude to write them (you never edit them), or do you want to write them yourself from the spec?

Once you approve (with changes), the next step is the **Level 0 spec** in `learning-documents/levels/level-00.md`.
