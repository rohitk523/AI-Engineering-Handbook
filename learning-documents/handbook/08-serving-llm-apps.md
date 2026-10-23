# 08 — Serving LLM Apps in Production

[← 07 Auth](07-fastapi-auth.md) · [Index](00-INDEX.md) · Next: [09 Evals →](09-evals-observability.md)

LLM calls are **slow (seconds), expensive, rate-limited, non-deterministic and sometimes fail**. Normal backend patterns still apply, and a few matter much more than usual.

---

## 1. Reference architecture

```
Client ──► API gateway / LB (TLS, auth, rate limit)
              │
              ▼
        FastAPI service  ──► Postgres (users, conversations, runs, pgvector)
          │   │    └──────► Redis (cache, rate limits, pub/sub, short-term state)
          │   └── SSE/WebSocket stream to client
          ▼
     Task queue (Celery / arq / Dramatiq / Temporal / cloud queue)
          │
          ▼
     Workers: agent runs, ingestion, embeddings ──► LLM gateway ──► providers / self-hosted vLLM
                                                         │
                                         tracing + cost + logs (OpenTelemetry, Langfuse...)
```

---

## 2. Sync, streaming or background?

| Task duration | Pattern |
|---|---|
| < ~30 s, user waiting | Direct call, **stream** tokens via SSE |
| 30 s – minutes (agent runs, doc ingestion) | `POST /runs` returns `202 Accepted` + `run_id`. A worker executes it. The client streams progress (SSE from Redis pub/sub) or polls `GET /runs/{id}` |
| Minutes – hours, multi-step with side effects | **Durable execution** (Temporal, Restate, LangGraph checkpointing, cloud workflows): each step is checkpointed so a crash resumes instead of restarting and repeating side effects |

---

## 3. Reliability patterns

- **Timeouts** on every LLM and tool call. Set a total budget per agent run (steps, tokens, wall-clock, dollars).
- **Retries with exponential backoff + jitter** on 429/5xx/timeouts (`tenacity` or the SDK's built-in retries). Never retry 4xx validation errors.
- **Fallbacks**: provider A fails → provider B or a smaller model.
- **Idempotency keys**: the client sends `Idempotency-Key` and the server stores the result per key, so a retried "create payment" doesn't charge twice. Critical when agents call write tools.
- **Circuit breakers** for flaky downstream tools.
- **Graceful degradation**: if the reranker is down, serve un-reranked results.

---

## 4. Rate limiting and quotas

Two directions:
1. **Your users → you**: per user/API key/tenant, in Redis (token bucket or sliding window). Limit **requests and tokens/cost**, not only requests. Return `429` with `Retry-After`.
2. **You → providers**: providers enforce RPM/TPM. Use a shared limiter or queue so 50 workers don't all hit 429 together.

---

## 5. Cost and latency control

| Lever | Effect |
|---|---|
| **Prompt caching** (stable prefix first) | Large input-cost reduction on repeated prefixes |
| **Model routing** | Small model for easy steps, frontier/reasoning model for hard ones |
| **Semantic / exact caching** | Cache answers to repeated questions (careful with personalization and freshness) |
| **Batch APIs** | Much cheaper for offline jobs (evals, backfills, ingestion) |
| **Trim context** | Shorter tool outputs, compaction ([03](03-agent-architectures.md)) |
| **Streaming** | Doesn't reduce cost, but makes perceived latency much lower |
| **Parallel tool calls / parallel subtasks** | Lower wall-clock time |
| **Self-hosting open models** | Cheaper at high, steady volume; worse at spiky low volume. Do the math including GPU idle time and ops staff |

Track **cost per request, per user and per feature**. Senior engineers are expected to know unit economics.

---

## 6. LLM gateway

A thin internal layer (or LiteLLM, Portkey, a cloud AI gateway) between your code and providers that handles one interface for many providers, keys, retries, fallbacks, rate limits, caching, cost tracking, logging and PII redaction. It also makes model migrations a config change.

---

## 7. Data you should persist

- `conversations`, `messages` (role, content, tool calls, token counts, model, latency)
- `runs` (status, steps, cost, error, trace_id) for agents
- `tool_calls` (name, args, result, duration, approved_by) as an audit trail
- feedback (thumbs up/down, corrections), which feeds your evals ([09](09-evals-observability.md))

Mind retention and PII rules (GDPR, India's DPDP Act). Users' prompts are personal data.

---

## 8. Deployment basics to be fluent in

- **Docker** (multi-stage, slim, non-root), **docker compose** locally.
- **Kubernetes** basics: Deployment, Service, Ingress, HPA, ConfigMap/Secret, liveness and readiness probes. GPU node pools for self-hosted models.
- Serverless containers (Cloud Run, AWS App Runner / ECS Fargate, Azure Container Apps) are often the simplest choice for the API layer.
- **CI/CD**: tests + evals on every PR, build image, deploy, canary.
- **12-factor config**: everything via env vars.
- **Postgres** (with pgvector) and **Redis** cover most AI startups' storage needs.

---

## 9. Check yourself

- Design `POST /agent-runs` so a 5-minute run survives a worker crash without sending the same email twice.
- Your provider returns 429s at peak. What do you change?
- How do you stream progress from a background worker to a browser?
- Estimate monthly cost: 10K users × 20 requests/day × 3K input + 500 output tokens. What levers cut it in half?
- Why should rate limits count tokens, not only requests?
