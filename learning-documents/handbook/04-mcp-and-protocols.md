# 04 — MCP, A2A and Agent Protocols ⭐

[← 03 Architectures](03-agent-architectures.md) · [Index](00-INDEX.md) · Next: [05 RAG →](05-rag-and-retrieval.md)

---

## 1. Why protocols?

Without a standard, every app × every tool needs custom glue (N × M integrations). **MCP (Model Context Protocol)**, introduced by Anthropic in late 2024 and now governed as an open standard adopted across major vendors, makes it N + M: any MCP client (Claude, ChatGPT, IDEs, your agent) can use any MCP server (GitHub, Postgres, Slack, your internal API).

The usual analogy is **USB-C for AI tools**. Another fair one: LSP (Language Server Protocol) for agents.

---

## 2. MCP architecture

```
 ┌──────────── Host app (Claude Desktop, IDE, your agent) ───────────┐
 │   LLM  ◄──►  MCP client ──┐   MCP client ──┐                      │
 └───────────────────────────┼────────────────┼──────────────────────┘
                             ▼                ▼
                      MCP server A      MCP server B
                     (GitHub, remote)   (local files, stdio)
```

- **Host**: the application the user interacts with.
- **Client**: one connection per server, inside the host.
- **Server**: exposes capabilities. It can be a local process (**stdio** transport) or a remote service (**Streamable HTTP** transport).
- Messages are **JSON-RPC 2.0**.

### What a server exposes ("primitives")
| Primitive | Controlled by | Meaning |
|---|---|---|
| **Tools** | Model | Functions the LLM can call (`create_issue`, `run_query`) |
| **Resources** | App | Read-only data the host can attach to context (files, DB rows) |
| **Prompts** | User | Reusable prompt templates (often surfaced as slash commands) |

Client-side features let servers ask the host for things (for example elicitation, which asks the user for input). Some older client features are being deprecated (see below).

### How a tool call flows
1. Host connects, discovers the server's capabilities, and calls `tools/list`.
2. Tool schemas get injected into the LLM's tool list.
3. LLM emits a tool call → client sends `tools/call` → server runs it → result returns to the LLM.

It's the same loop as [02](02-tool-calling-and-agent-loop.md). MCP only standardizes discovery and transport.

---

## 3. What changed in 2026 (as of Oct 2026, verify against the official spec)

The **2026-07-28 specification** is described as the biggest revision since launch. Reported highlights:
- **Stateless by design**: protocol-level sessions (the `initialize` handshake and `Mcp-Session-Id` header) are removed, and a discovery method lets clients fetch server capabilities up front. This makes MCP servers much easier to run behind load balancers and serverless platforms, like ordinary HTTP APIs.
- **Tasks**: first-class long-running operations (start, poll, get result) for work that takes minutes.
- **MCP Apps**: servers can ship interactive UI to hosts.
- **Formal extensions framework** and a **predictable deprecation policy**, with features like Roots, Sampling and Logging going through deprecation windows.
- **Authorization hardening**: OAuth 2.1-based authorization for remote servers, with the MCP server acting as an OAuth resource server.
- Cacheable discovery/list results and header-based routing.

Practical takeaway: if you build MCP servers now, design them **stateless**, put auth in front with OAuth, and avoid features marked as deprecated.

---

## 4. Writing an MCP server in Python (minimal)

The official Python SDK includes **FastMCP**, a decorator API that feels like FastAPI:

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("inventory")

STOCK = {"widget": 12, "gadget": 0}

@mcp.tool()
def check_stock(sku: str) -> str:
    """Units in stock for a SKU. Returns 'unknown sku' if not found."""
    qty = STOCK.get(sku.lower())
    return "unknown sku" if qty is None else str(qty)

@mcp.resource("inventory://skus")
def list_skus() -> str:
    """All known SKUs."""
    return ", ".join(STOCK)

if __name__ == "__main__":
    mcp.run()   # stdio by default; HTTP transport options are available
```

Type hints + docstring become the JSON Schema and description. Test it with the **MCP Inspector** before connecting it to a client. Check the SDK docs for the current API, since it has evolved quickly.

### MCP server design tips
- Treat tool descriptions as prompts (see [02 §4](02-tool-calling-and-agent-loop.md)).
- Don't auto-generate one tool per REST endpoint. Expose **task-level** tools.
- Return concise text. Paginate.
- Every tool runs with *someone's* credentials, so scope them tightly ([10](10-security-for-agents.md)).
- Remote servers: OAuth, per-user tokens, rate limits, audit logs.

---

## 5. Other protocols you'll hear about

| Protocol | Purpose |
|---|---|
| **A2A (Agent2Agent)** | Started by Google, now under the Linux Foundation. Lets *agents* talk to *agents* across vendors (task delegation, "agent cards" describing capabilities). MCP = agent↔tool; A2A = agent↔agent |
| **AG-UI and similar** | Event protocols for streaming agent state to frontends |
| **OpenAPI** | Still the main way to describe REST APIs. Many MCP servers wrap OpenAPI specs |
| **Agent payments / commerce protocols** | Emerging standards for agents making purchases with user authorization. Watch this space (verify current names) |
| **`llms.txt` / `AGENTS.md`** | Conventions for making websites and repos readable by agents (`AGENTS.md` = README for coding agents) |

---

## 6. Check yourself

- Explain MCP to a backend engineer in two sentences.
- Tools vs resources vs prompts: who controls each, and why does that matter?
- Why does removing sessions make MCP easier to deploy?
- How would you secure a remote MCP server that gives agents access to a company's CRM?
- When would you use A2A instead of MCP?
