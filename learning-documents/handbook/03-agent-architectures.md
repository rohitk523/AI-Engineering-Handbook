# 03 — Agent Architectures and Context Engineering ⭐

[← 02 Tool calling](02-tool-calling-and-agent-loop.md) · [Index](00-INDEX.md) · Next: [04 MCP →](04-mcp-and-protocols.md)

---

## 1. The most important distinction: workflows vs agents

| | **Workflow** | **Agent** |
|---|---|---|
| Who decides the steps? | Your code (predefined path) | The model (dynamic path) |
| Predictability | High | Lower |
| Cost / latency | Lower, bounded | Higher, variable |
| Good for | Well-understood tasks: extraction, classification, fixed pipelines | Open-ended tasks: coding, research, support with many branches |

**Senior rule: use the simplest thing that works.** Start with one LLM call plus retrieval. Add a workflow if needed. Add an agent loop only when the path truly can't be known in advance. Most "agent" products in production are mostly workflows with a small agentic core.

---

## 2. The building-block patterns

These patterns (popularized by Anthropic's *"Building effective agents"* essay) are the vocabulary of design reviews.

| Pattern | Shape | Example |
|---|---|---|
| **Augmented LLM** | LLM + retrieval + tools + memory | Basis of everything |
| **Prompt chaining** | Step 1 → check → step 2 → step 3 | Draft → critique → rewrite |
| **Routing** | Classifier picks a specialized path or model | Simple FAQ to a small model, complex billing issue to a reasoning model |
| **Parallelization** | Split into independent subtasks (sectioning) or run N times and vote | Check a document for 5 policy types in parallel |
| **Orchestrator–workers** | A lead LLM breaks the task down and spawns workers dynamically | Deep research, multi-file code changes |
| **Evaluator–optimizer** | Generator + critic in a loop until the critic passes | Translation, code that must pass tests |
| **Autonomous agent** | Tool loop with environment feedback until done | Coding agents, computer-use agents |

### ReAct (Reason + Act)
The original agent prompting pattern: *Thought → Action → Observation → Thought ...* Native tool calling plus reasoning models have absorbed it. You no longer parse "Action:" strings, but the idea is the same loop.

### Plan-and-execute
Make a plan first (a todo list), then execute steps, re-planning when observations contradict the plan. Modern coding agents keep an explicit todo list in context for exactly this reason: it keeps long tasks on track.

---

## 3. Multi-agent systems

**What it is:** several LLM "agents" with separate contexts, prompts and tools, coordinated by code or by a lead agent.

**When it helps:**
- The task is **breadth-first and parallelizable** (research across many sources, searching a large codebase).
- Subtasks produce lots of noisy intermediate context that the main agent doesn't need. A **subagent** explores, then returns a *compressed summary*, which keeps the lead agent's context clean.
- You need different permissions per role (a reader agent with no write tools, a writer agent with no internet).

**When it hurts:**
- Tightly coupled tasks where agents need shared, constantly-changing state (most coding edits). Agents disagree, duplicate work or overwrite each other.
- Cost: multi-agent runs can use many times the tokens of a single agent.

**Coordination topologies:** orchestrator–worker (most common), pipeline/hand-off (agent A passes to B), group chat/debate (rarely worth it), and a blackboard (shared state store).

---

## 4. Context engineering (the core senior skill)

> Prompt engineering is choosing the words. **Context engineering is deciding what information goes into the context window at each step**, and what stays out.

The context window is a scarce, expensive resource whose quality degrades as it fills. Every token competes for the model's attention.

### What lives in context
```
[system prompt] [tool definitions] [retrieved docs / memory] [conversation + tool results] [current task]
 ^ stable, cacheable ───────────────────────────────────────────────►  ^ changes every step
```

### Techniques
| Technique | What it does |
|---|---|
| **Just-in-time retrieval** | Give the agent tools (`search`, `read_file`, `grep`) to fetch what it needs instead of stuffing everything up front. This is how coding agents work |
| **Compaction / summarization** | When history gets long, summarize old turns into a short state description and continue |
| **Tool-result clearing** | Drop or shorten old, bulky tool outputs once they've been used |
| **Structured note-taking / memory files** | Agent writes progress notes or a todo list to a file and re-reads it, which survives compaction and new sessions |
| **Subagents** | Isolate exploration in a separate context and return only the summary |
| **Progressive disclosure ("skills")** | Show only a short index of capabilities. Load full instructions only when relevant (see below) |
| **Prompt caching** | Keep stable content first and unchanged so the provider caches it |
| **Fewer tools** | 100 tool definitions cost tokens and confuse selection. Load tools on demand (tool search) |

### Memory types
| Type | Meaning | Implementation |
|---|---|---|
| Working memory | Current context window | The `messages` list |
| Episodic | What happened in past sessions | Saved transcripts or summaries, retrieved by similarity or recency |
| Semantic | Facts about the user or domain | Key-value store, vector DB, or plain markdown memory files |
| Procedural | How to do things | System prompts, skills, playbooks |

Simple beats clever: plain files and a few well-named markdown notes often outperform elaborate vector-memory systems.

---

## 5. Agent harnesses, skills and the "agent as a product" stack

The **harness** is everything around the model: the loop, tool execution, permissions, context management, memory, sandboxes and UI. Products like Claude Code, Codex, Cursor and Devin-style agents are mostly harness engineering on top of frontier models.

Recent architectural ideas worth knowing (as of 2026, verify specifics):

- **Skills**: folders of instructions + scripts + resources that an agent loads on demand (a short description in context, full content only when needed). They're a portable way to give agents procedural knowledge without bloating every prompt.
- **Agent SDKs**: providers ship the harness as a library (for example Claude Agent SDK, OpenAI Agents SDK, Google ADK), so you get the loop, tools, subagents, hooks and permissions out of the box.
- **Hooks**: deterministic code that runs at lifecycle points (before a tool call, after an edit, on stop) to enforce rules the model might forget: lint, block dangerous commands, require tests.
- **Sandboxed code execution**: instead of 50 bespoke tools, give the agent a sandbox (container/VM) with a shell and let it write code. "Code as the universal tool" is a big 2025–26 trend. It also cuts token use: the agent can process data in code instead of pulling it into context.
- **Computer use / browser agents**: the model sees screenshots and emits mouse/keyboard actions. This is a vision problem at heart.
- **Long-running / background agents**: run for hours in the cloud, checkpoint state, and report back (PRs, reports). These need durable execution (see [08](08-serving-llm-apps.md)).
- **Human-in-the-loop**: approval gates on risky actions, interrupt/resume, and the ability to edit the agent's plan.

---

## 6. Frameworks: what to know and when to skip them

| Framework | Good at | Watch out for |
|---|---|---|
| **Raw SDK loop** | Full control, easy to debug, interview-standard | You build retries, tracing and memory yourself |
| **LangGraph** | Explicit state graphs, checkpointing, human-in-the-loop, durable workflows | Learning curve; overkill for simple loops |
| **Pydantic AI** | Type-safe agents, structured outputs, feels like FastAPI | Smaller ecosystem |
| **OpenAI Agents SDK / Claude Agent SDK / Google ADK** | Provider-optimized loops, handoffs, built-in tools | Some vendor coupling |
| **CrewAI / AutoGen-family** | Role-based multi-agent prototypes | Easy to build demos that are hard to make reliable |
| **LlamaIndex** | Data ingestion and RAG pipelines | Large abstraction surface |
| **DSPy** | Programmatic prompt optimization against a metric | Different mental model |

**Senior take:** understand the raw loop first ([02](02-tool-calling-and-agent-loop.md)). Pick a framework for **state management, durability and observability**, not for "making agents". Avoid frameworks that hide the prompt and messages from you.

---

## 7. Check yourself

- A PM asks for "a multi-agent system" to answer HR policy questions. What do you build instead and why?
- Explain context rot and three techniques to fight it.
- When does a subagent save tokens, and when does it waste them?
- Design the memory for a personal assistant that should remember preferences across months.
- What's the difference between a tool and a skill?
- Why might "give the agent a code sandbox" beat "give the agent 40 tools"?
