# 11 — The GenAI Landscape in Late 2026

[← 10 Security](10-security-for-agents.md) · [Index](00-INDEX.md) · Next: [12 Python & DSA →](12-python-and-dsa.md)

> Everything here is time-sensitive. Treat names and rankings as **(as of 2026, verify)**. The trends are more durable than the names.

---

## 1. Big trends (2025 → 2026)

1. **From chatbots to agents.** Value moved from "answer a question" to "complete a task": coding agents, research agents, support agents that take actions, browser/computer-use agents, and background agents that run for hours.
2. **Reasoning models are the default for hard work.** RL on verifiable rewards (math, code, tool use) made models much better at multi-step tasks. Effort/thinking budgets are now an API parameter you tune.
3. **Coding agents are the breakout use case.** Terminal and IDE agents (Claude Code, OpenAI Codex, Cursor, GitHub Copilot agent mode, Gemini CLI and others) changed how engineers work. Engineers increasingly *specify, review and orchestrate* rather than type every line.
4. **Context engineering > prompt engineering.** Compaction, memory, subagents, skills and tool search ([03](03-agent-architectures.md)).
5. **Protocol standardization.** MCP for tools, A2A for agent-to-agent, both under neutral foundations. The MCP 2026-07-28 spec went stateless ([04](04-mcp-and-protocols.md)).
6. **Code execution as the universal tool.** Agents get sandboxes and write code to act, instead of calling hundreds of bespoke tools.
7. **Open-weight models are close behind.** Strong open models (from Meta, Mistral, Alibaba's Qwen, DeepSeek, Google's Gemma, OpenAI's gpt-oss and others) make self-hosting and fine-tuning realistic. Your cafe assignment uses `gpt-oss-20b` on Groq.
8. **Inference got much cheaper and faster.** Price per token keeps falling. Specialized hardware (Groq LPUs, Cerebras, TPUs, Trainium) and serving engines (vLLM, SGLang) matter.
9. **Multimodal and real-time.** Native image, audio and video input/output; real-time voice agents; video generation.
10. **Physical AI / embodied agents.** Vision-language-action (VLA) models for robotics, and agents coordinating machines. **This is where CV skills are scarce and valuable.**
11. **Evals, governance and regulation.** The EU AI Act obligations are phasing in; enterprises demand audit trails, evals and data controls.

---

## 2. Model landscape (shape, not rankings)

| Category | Examples (verify current versions) | Typical use |
|---|---|---|
| Frontier closed models | Anthropic Claude (Opus / Sonnet / Haiku tiers), OpenAI GPT and o-series, Google Gemini (Pro / Flash) | Hard reasoning, coding, agents |
| Small/fast closed tiers | Haiku, Flash, mini/nano variants | Routing, classification, high-volume steps |
| Open-weight | Llama, Qwen, DeepSeek, Mistral, Gemma, gpt-oss, Kimi and GLM families | Self-hosting, fine-tuning, data privacy |
| Embeddings / rerankers | OpenAI, Voyage, Cohere, Jina, BGE/E5/GTE | Retrieval |
| VLMs | Most frontier models + open VLMs (Qwen-VL family, etc.) | Documents, screenshots, video |
| Speech | Whisper-family ASR, realtime speech APIs, TTS | Voice agents |

**How seniors choose:** build an eval for your task ([09](09-evals-observability.md)), then compare 3–4 candidates on quality, latency, cost and data/compliance constraints. Re-run when new models launch. Leaderboards are a starting point, not an answer.

---

## 3. Architectural advances to understand (at a conceptual level)

| Topic | One-liner |
|---|---|
| Mixture of Experts | Huge total parameters, small active parameters per token |
| Long context techniques | RoPE scaling, sliding-window/sparse attention, better KV cache handling |
| Linear-attention / state-space hybrids | Mamba-style layers mixed with attention for cheaper long sequences |
| Multi-head latent attention (MLA) | Compresses the KV cache (popularized by DeepSeek) |
| Test-time compute scaling | Accuracy improves with more thinking tokens or sampling + verification |
| RLVR / GRPO | RL from verifiable rewards, the training method behind open reasoning models |
| Distillation | Big model teaches small model, which is how strong small models are made |
| Speculative decoding, FP8/FP4 inference | Faster, cheaper serving |
| Native multimodality | One model trained across text/image/audio tokens rather than bolted-on encoders |
| Diffusion LMs | Experimental: generate text by iterative denoising (familiar to you from image diffusion) |

---

## 4. How engineering work is changing

- **AI-native workflow**: engineers drive coding agents with specs, tests and reviews. Skills that matter more now: writing clear specs, decomposing work, code review, testing and system design.
- **"AI engineer" as a role**: sits between ML engineering and backend. It's about building products on top of models (agents, RAG, evals, infra), less about training models from scratch.
- **Evaluation and reliability work** is what separates seniors from people who only build demos.
- **Full-stack expectations**: a senior AI engineer is expected to ship the FastAPI service, the auth, the DB schema, the queue, the evals and the dashboard, not only the model call.
- **Cost awareness**: unit economics per task is part of design reviews.

---

## 5. Where your CV background fits

High-value overlaps where few LLM-only engineers are strong:
- **Document AI**: layout, tables, OCR, VLM parsing for RAG pipelines ([05](05-rag-and-retrieval.md)).
- **Computer-use and browser agents**: screen understanding, UI grounding, element detection.
- **Multimodal agents**: inspection, retail, medical imaging, video analytics with LLM reasoning on top of detectors.
- **VLM fine-tuning and evaluation**: you already know data curation, augmentation and metrics.
- **Edge deployment**: quantization and on-device inference carry over directly to small LLMs/VLMs.
- **Robotics / physical AI**: VLA models, simulation, perception-to-action pipelines.

**Positioning pitch:** *"I build agentic systems that see." Vision + agents + solid backend.*

---

## 6. How to keep up (without drowning)

**Weekly (1–2 hours):**
- A curated newsletter or two (for example Latent Space, The Batch, Simon Willison's blog, Import AI, Interconnects). Pick 2, not 10.
- Changelogs and engineering blogs of the 3 providers you use (Anthropic, OpenAI, Google), plus vLLM and FastAPI release notes.

**Monthly:**
- Read 1–2 engineering deep dives (Anthropic Engineering blog posts on agents, context engineering, multi-agent research; OpenAI cookbook; Google ADK docs).
- Try one new tool or model on **your own eval set**.

**Quarterly:**
- Rebuild one small project with the newest approach and compare.
- Skim the MCP spec changelog and OWASP LLM guidance.

**Rule:** read primary sources (docs, papers, changelogs) over hype threads. If a claim can't be traced to an official announcement, don't rely on it.

---

## 7. Check yourself

- Name three ways agents changed in 2025–26 compared with 2023-era chatbots.
- How would you pick a model for a new product feature?
- Explain why reasoning models cost more and when they're worth it.
- Pitch your CV background for an agentic AI role in 30 seconds.
