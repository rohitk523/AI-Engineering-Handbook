# 02 — Tool Calling and the Agent Loop ⭐

[← 01 LLM fundamentals](01-llm-fundamentals.md) · [Index](00-INDEX.md) · Next: [03 Architectures →](03-agent-architectures.md)

> **About your cafe-billing assignment:** its README says the round must be written without AI assistance, so this file doesn't solve `agent.py`. It teaches the pattern with a different example (a weather/unit-conversion assistant). Once you understand this file, you can write that loop yourself in a few minutes.

---

## 1. The mental model

The model **cannot run code**. It can only emit text. "Tool calling" means:

1. You describe tools to the model (name, description, JSON Schema of the arguments).
2. The model, instead of answering, emits a structured request: *"call `get_weather` with `{"city": "Pune"}`"*.
3. **Your code** runs the function and sends the result back as a new message.
4. The model reads the result and either asks for another tool or answers in plain text.

That loop **is** an agent. Anthropic's definition: *an agent is an LLM using tools in a loop, deciding its own next step based on environment feedback.*

```
           ┌──────────────────────────────────────────┐
           ▼                                          │
 messages ──► LLM(messages, tools) ──► tool_calls? ── yes ──► run each tool ──► append results
                                         │
                                         no
                                         ▼
                                  return final text
```

---

## 2. The message protocol (OpenAI-compatible "Chat Completions" format)

This is the format used by OpenAI, Groq, Together, vLLM, Ollama and most others. Anthropic's Messages API and OpenAI's newer Responses API use the same ideas with different field names.

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Weather in Pune in Fahrenheit?"},
]
```

The model replies with an **assistant message that contains `tool_calls`** and usually no `content`:

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {"id": "call_abc", "type": "function",
     "function": {"name": "get_weather", "arguments": "{\"city\": \"Pune\"}"}}
  ]
}
```

Key details that trip people up:
- `arguments` is a **JSON string**, not a dict. You must `json.loads` it, and it can be malformed.
- You must append **that assistant message itself** to `messages` before the tool results. The API rejects tool results that don't follow a matching assistant `tool_calls` message.
- Every tool result is a message with `role: "tool"`, the matching **`tool_call_id`**, and string `content`.
- A model can request **several tool calls in one turn** (parallel tool calls). Answer **all** of them before calling the model again.

```json
{"role": "tool", "tool_call_id": "call_abc", "content": "31°C, humid"}
```

---

## 3. A reference implementation (different domain from your assignment)

```python
import json
from openai import OpenAI

client = OpenAI()  # or OpenAI(base_url=..., api_key=...) for Groq/vLLM/etc.
MODEL = "your-model-id"
MAX_STEPS = 8


def get_weather(city: str) -> str:
    fake = {"pune": 31, "delhi": 36}
    temp = fake.get(city.strip().lower())
    return f"unknown city: {city}" if temp is None else str(temp)


def celsius_to_fahrenheit(celsius: float) -> str:
    return f"{celsius * 9 / 5 + 32:.1f}"


TOOLS = [
    {"type": "function", "function": {
        "name": "get_weather",
        "description": "Current temperature in Celsius for a city.",
        "parameters": {"type": "object",
                       "properties": {"city": {"type": "string"}},
                       "required": ["city"]}}},
    {"type": "function", "function": {
        "name": "celsius_to_fahrenheit",
        "description": "Convert a Celsius temperature to Fahrenheit.",
        "parameters": {"type": "object",
                       "properties": {"celsius": {"type": "number"}},
                       "required": ["celsius"]}}},
]

REGISTRY = {"get_weather": get_weather, "celsius_to_fahrenheit": celsius_to_fahrenheit}


def execute_tool(name: str, raw_args: str) -> str:
    """Never let a tool error crash the loop: return the error to the model as text."""
    func = REGISTRY.get(name)
    if func is None:
        return f"error: unknown tool {name}"
    try:
        args = json.loads(raw_args or "{}")
        return str(func(**args))
    except Exception as exc:  # bad JSON, wrong args, tool bug
        return f"error: {type(exc).__name__}: {exc}"


def run(user_text: str) -> str:
    messages = [
        {"role": "system", "content": "Use tools for facts. Be concise."},
        {"role": "user", "content": user_text},
    ]
    for _ in range(MAX_STEPS):
        response = client.chat.completions.create(
            model=MODEL, messages=messages, tools=TOOLS,
        )
        msg = response.choices[0].message

        if not msg.tool_calls:          # plain text, so we're done
            return msg.content or ""

        messages.append(msg)            # the assistant turn that requested the tools
        for call in msg.tool_calls:     # handle every parallel call
            result = execute_tool(call.function.name, call.function.arguments)
            print(f"  tool {call.function.name}({call.function.arguments}) -> {result}")
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": result,
            })
    return "Stopped: too many steps."
```

Things a reviewer will look for (and the things interviewers probe):

| Concern | What good looks like |
|---|---|
| Termination | Stop on plain text, **and** keep a `MAX_STEPS` guard against infinite loops |
| History | Append the assistant tool-call message, then one `tool` message per call, with matching ids |
| Parallel calls | Loop over **all** `tool_calls`, not just `[0]` |
| Errors | Tool errors become strings returned to the model, which can then correct itself. Don't crash |
| Argument parsing | `json.loads` inside try/except. Validate with Pydantic in real systems |
| Tool output | Always a string. Keep it short and informative (the model reads every byte, and you pay for it) |
| Determinism of business logic | Arithmetic, pricing, totals happen **in code**, never "in the model's head" |
| Observability | Log every tool name, args, result and latency |

The last-but-one row is the reason tools like `calculate_bill` exist: **LLMs are bad at reliable arithmetic and good at deciding what to compute.** Put the computation in a deterministic function and let the model orchestrate it.

---

## 4. Writing good tools (the "agent-computer interface")

Tool design affects agent quality more than prompt tweaks do.

- **Name and describe tools for a new colleague.** The description is a prompt. Say what it does, when to use it, units, and what it returns.
- **Fewer, higher-level tools beat many tiny ones.** `calculate_bill(items)` (one call for the whole order) is better than making the model call `get_price` N times and add up results.
- **Use tight schemas:** enums, required fields, integer vs number. Turn on *strict* schema mode if the provider supports it.
- **Return errors that tell the model how to recover:** `"unknown item: latte. Valid items: masala chai, ..."` is better than `"error"`.
- **Make tools idempotent where you can**, especially ones with side effects (payments, emails). Agents retry.
- **Paginate or truncate large outputs.** Never dump 50K tokens of logs into context.
- **Separate read tools from write tools** so you can require human approval for writes ([10](10-security-for-agents.md)).

---

## 5. Variations you should know

| Variation | What it is |
|---|---|
| `tool_choice` | `"auto"` (model decides), `"required"` (must call some tool), a specific tool, or `"none"` |
| Streaming tool calls | Arguments arrive as partial JSON chunks. Accumulate by index until the stream ends |
| Anthropic Messages API | Tool calls are `tool_use` content blocks. Results go back as `tool_result` blocks inside a **user** message |
| OpenAI Responses API | Server-side conversation state, built-in tools (web search, file search, code interpreter), items instead of messages |
| Server-side / hosted tools | Provider runs the tool (web search, code execution) and you only see results |
| Provider "tool runners" / agent SDKs | The SDK runs the loop for you. Use them in production, but **know how to write it raw** (interviews test exactly this) |
| MCP | A standard way to expose tools to any agent. The loop stays the same, only tool discovery and execution go over a protocol ([04](04-mcp-and-protocols.md)) |

---

## 6. Common failure modes and fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| API 400 "tool messages must follow tool_calls" | Forgot to append the assistant message, or ids mismatch | Append `msg` before tool results and copy `call.id` exactly |
| Model loops calling the same tool | Tool result unclear or an error string unhelpful | Better error text, a step limit, and dedupe identical calls |
| Model answers without calling a tool | Weak description, or the model "knows" a guess | Sharpen description, use system prompt "always use tools for prices", or `tool_choice` |
| Hallucinated tool or arguments | Too many tools, vague schema | Fewer tools, enums, strict mode, validation |
| Wrong totals | Model did arithmetic itself | Move the math into a tool |
| Slow and expensive | Huge tool outputs, history growing every step | Truncate outputs, summarize/compact history, cache the prefix |

---

## 7. Check yourself

- Write the loop from memory in under 10 minutes, with a step limit and parallel calls handled.
- Why must the assistant `tool_calls` message be appended before the `tool` messages?
- What happens if `arguments` is invalid JSON? How do you recover?
- Why put arithmetic in a tool instead of trusting the model?
- How would you add human approval before a `send_email` tool runs?
- How does the loop change when streaming?
