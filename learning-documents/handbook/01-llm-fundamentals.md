# 01 — LLM Fundamentals for a Computer Vision Engineer

[← Index](00-INDEX.md) · Next: [02 Tool calling →](02-tool-calling-and-agent-loop.md)

You already know convolutions, attention (from ViTs), losses, fine-tuning and deployment. This file maps what you know to how LLMs are used in products.

---

## 1. The mental model

An LLM is a **next-token predictor**. Given a sequence of tokens, it outputs a probability distribution over the vocabulary for the next token. Generation repeats that step:

```
prompt tokens ──► [Transformer decoder] ──► P(next token) ──► sample ──► append ──► repeat
```

| CV concept you know | LLM equivalent |
|---|---|
| Image patches (ViT) | Tokens (sub-word pieces, ~0.75 English words per token) |
| Patch embedding | Token embedding |
| Positional embedding | RoPE (rotary position embeddings), now standard |
| Backbone pre-training on ImageNet / self-supervised | Pre-training on trillions of tokens (next-token loss) |
| Fine-tuning a detection head | SFT (supervised fine-tuning) on instruction/chat data |
| — | Preference tuning (RLHF, DPO) and **RL on verifiable rewards** (math, code, tool use). This is what made "reasoning models" |
| Test-time augmentation | Test-time compute: the model "thinks" longer before answering |
| Model zoo backbones | Foundation models from labs, served through APIs or as open weights |

### Chat models are still just token sequences
"Messages" (`system`, `user`, `assistant`, `tool`) are turned into one token sequence using a **chat template** with special tokens. Tool calls are the model emitting a structured block (usually JSON) that the serving layer parses. Nothing magic happens. This matters when you debug open-weight models: a wrong chat template silently breaks tool calling.

---

## 2. Concepts you must be fluent in

### Tokens and context window
- Everything is billed and limited in tokens: input tokens, output tokens, and on reasoning models "thinking" tokens.
- **Context window** = max tokens (input + output) per request. Frontier models offer 200K to 1M+ tokens (as of 2026, verify).
- A large window does not mean the model uses all of it well. Quality drops with long, noisy contexts ("context rot"). That's why **context engineering** exists ([03](03-agent-architectures.md)).

### Sampling
- `temperature`: low means near-deterministic, high means more diverse. Use 0–0.3 for extraction and tool use, higher for creative work.
- `top_p` / `top_k`: truncate the distribution before sampling.
- Even at temperature 0, outputs aren't guaranteed to be reproducible (batching, floating point, MoE routing). **Design for non-determinism**: that's why evals use multiple runs.
- Many reasoning models fix or ignore some of these parameters. Check the provider docs.

### Structured outputs
Modern APIs can constrain output to a JSON Schema (constrained decoding / "structured outputs"). Use this instead of "please return JSON" prompts. In Python, define the schema with **Pydantic** and validate the result.

### Reasoning ("thinking") models
- Trained with RL to produce a long internal chain of thought before answering.
- Controls: a thinking budget or an effort level (`low / medium / high`).
- Trade-off: better on multi-step problems (math, code, planning, agentic tasks), but slower and costlier. Senior engineers **route**: cheap fast model for simple steps, reasoning model for hard ones.
- **Interleaved thinking**: the model thinks between tool calls, which is now common in agentic APIs.

### Multimodality (your advantage)
Vision-language models (VLMs) take images (and often video, audio, PDFs) as input. Typical architecture: a vision encoder (ViT-family) → projector → LLM. Uses: document understanding, screenshots for **computer-use agents**, charts, UI testing, robotics. See [11](11-genai-landscape-2026.md).

### Embeddings
A separate model maps text (or images) to a vector. This is the basis of semantic search and RAG ([05](05-rag-and-retrieval.md)). It works like CLIP or metric learning, which you already know.

---

## 3. Inference: what happens on the server

Know this well enough to talk to infra teams and to self-host open-weight models.

| Concept | One-line explanation |
|---|---|
| **Prefill vs decode** | Prefill processes the whole prompt in parallel (compute-bound). Decode generates one token at a time (memory-bandwidth-bound) |
| **KV cache** | Stores attention keys/values of previous tokens so each new token doesn't recompute them. Memory grows with context length × batch |
| **Prompt / prefix caching** | Reuse KV cache across requests that share a prefix (system prompt, tool list, documents). Providers discount cached tokens heavily. **Put stable content first, variable content last** |
| **Continuous batching** | Server adds and removes requests from a running batch on every step. Core of vLLM, SGLang and TGI |
| **PagedAttention** | vLLM's trick: KV cache in fixed-size pages, like OS virtual memory, so there is less fragmentation |
| **Quantization** | FP8 / INT8 / INT4 (AWQ, GPTQ, GGUF). Same idea as quantizing CV models for edge devices |
| **Speculative decoding** | A small draft model proposes tokens and the big model verifies them in parallel. Faster at the same quality |
| **MoE (Mixture of Experts)** | Only some "expert" layers run per token, so you get large total parameters with small active parameters. Most frontier and open models use it |
| **TTFT / TPOT** | Time to first token and time per output token. The two latency numbers users feel |

**Self-hosting stack (as of 2026, verify):** vLLM or SGLang for GPU serving, llama.cpp or Ollama for local/CPU, OpenAI-compatible HTTP endpoints as the de-facto interface. Groq, Together and Fireworks are hosted open-model providers with OpenAI-compatible APIs. Your cafe assignment uses Groq through the `openai` SDK in exactly this way.

---

## 4. Adapting a model: the decision ladder

Senior engineers climb this ladder in order and stop as early as possible:

1. **Better prompt + examples** (few-shot). Cheapest.
2. **Better context**: retrieval (RAG), tools, structured inputs.
3. **Better model / routing**: a stronger model for the hard parts.
4. **Fine-tuning**: LoRA/QLoRA SFT for format, style, domain jargon or a narrow task; distillation from a big model to a small one for cost and latency.
5. **RL fine-tuning** (RLVR/GRPO-style) for narrow verifiable tasks. Expensive, and needs a good reward signal.

Fine-tuning teaches **behavior and format** well and **new facts** poorly. Facts go in retrieval.

---

## 5. Check yourself

- Why does putting the system prompt and tool definitions at the start of the request reduce cost?
- Explain the KV cache, and why long context increases memory linearly.
- When would you pick a reasoning model over a standard one, and how would you contain the cost?
- Why can't you rely on temperature 0 for reproducibility?
- A client wants the model to "know" their 10,000-page policy manual. Fine-tune or RAG? Why?
- What is the difference between prefill and decode, and which one does a long prompt slow down?
