# 13 — System Design for AI Systems

[← 12 Python & DSA](12-python-and-dsa.md) · [Index](00-INDEX.md) · Next: [14 Projects →](14-project-ladder.md)

---

## 1. A template for any AI system design interview (45–60 min)

1. **Clarify (5 min)**: users, core tasks, scale (QPS, documents, users), latency targets, accuracy bar, cost budget, compliance (PII, data residency), what an error costs.
2. **Define success (3 min)**: offline eval metrics + online metrics (resolution rate, CSAT, cost per task).
3. **Simplest baseline (5 min)**: one LLM call + retrieval. Explain why it may be enough, or why it isn't.
4. **High-level architecture (10 min)**: API, auth, orchestration (workflow or agent), tools, data stores, queue, model gateway.
5. **Deep dives (15 min)**: retrieval quality, the agent loop and tools, context management, state/durability, safety.
6. **Reliability and ops (5 min)**: evals in CI, tracing, fallbacks, rate limits, human escalation.
7. **Cost and scaling (5 min)**: back-of-envelope token math, caching, routing, batching.
8. **Risks and next steps (2 min)**.

---

## 2. Worked design A: customer-support agent for an e-commerce company

**Requirements:** answer policy questions, check order status, start returns; 50K conversations/day; < 2 s to first token; refunds over ₹5,000 need human approval.

```
Web/app chat ──SSE──► FastAPI (JWT auth, per-user rate limit)
                         │
                   Router (small model): FAQ | order-action | escalate
                     │            │                     │
              RAG over policies   Agent loop             Human handoff queue
              (hybrid + rerank)   tools: get_order(user-scoped), start_return,
                                         issue_refund (approval gate > ₹5K)
                         │
          Postgres (conversations, runs, audit) · Redis (sessions, limits) · Tracing + evals
```

Key decisions to talk through:
- **Tools act with the user's identity.** `get_order` only returns orders where `user_id = current_user` (prevents IDOR via the agent).
- **Refund tool**: idempotency key per conversation and order, human approval above the threshold, and audit log.
- **Prompt injection**: order notes and emails are untrusted. The agent can't send arbitrary outbound messages.
- **Evals**: 300 labeled conversations; deterministic checks on tool calls; judge on tone and policy accuracy; track escalation and recontact rates online.
- **Cost**: router sends ~60% of traffic to a cheap model; prompt caching on policies and tools.

---

## 3. Worked design B: document QA over 2 million enterprise PDFs (multi-tenant)

- **Ingestion pipeline** (queue + workers): parse (layout model / VLM for scanned pages, your CV edge), structure-aware chunking, contextual summaries, embed, write to pgvector/Qdrant **with `tenant_id` and ACL metadata**, plus a BM25 index.
- **Query**: auth → tenant/ACL filter at the DB → hybrid retrieve 100 → rerank to 8 → answer with citations → groundedness check.
- **Freshness**: change-data-capture or webhooks to re-ingest edited docs; versioned chunks.
- **Scale math**: 2M docs × ~30 chunks = 60M vectors × 1024 dims × 4 bytes ≈ 245 GB raw. Discuss quantization (int8/binary), sharding by tenant, HNSW memory.
- **Evals**: retrieval recall@k per tenant sample; faithfulness judge; human spot checks.

---

## 4. Worked design C: background coding agent that opens PRs

- Trigger: issue labeled `agent` → job queued → **isolated sandbox** (container/VM) with repo clone, no production secrets, egress allow-list (package registry only).
- Agent loop with tools: read/search files, edit, run tests, shell (sandboxed). Todo list + compaction for long runs. **Durable execution** with checkpoints.
- Done = tests pass + lint passes + self-review → opens PR with summary; humans review and merge.
- Guardrails: max runtime/cost, no force-push, no changes to CI config without approval.
- Metrics: PR merge rate, review rounds, time to merge, cost per merged PR.

---

## 5. Back-of-envelope cheat sheet

- 1 token ≈ 4 English characters ≈ 0.75 words. A page ≈ 500–800 tokens.
- Cost/day = requests × (input_tokens × in_price + output_tokens × out_price). Then apply cache discounts.
- Output tokens are slower and pricier than input. Decode speed is tens to hundreds of tokens/s depending on model and hardware.
- Agent runs multiply tokens: each step re-sends the growing history (unless cached).

---

## 6. Practice prompts

- Design a voice agent that books doctor appointments.
- Design a visual-inspection system where a VLM explains defects and files tickets.
- Design "chat with your data" over a company's Postgres warehouse (text-to-SQL, with safety).
- Design an internal MCP platform that exposes 30 company systems to agents with proper auth.
- Design an eval platform for a team shipping 10 LLM features.
