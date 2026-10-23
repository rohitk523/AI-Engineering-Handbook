# 09 — Evals, Observability and Guardrails ⭐

[← 08 Serving](08-serving-llm-apps.md) · [Index](00-INDEX.md) · Next: [10 Security →](10-security-for-agents.md)

> In CV you had a test set and mAP. In LLM apps there is often no single metric, outputs are free text, and they vary run to run. **Evals are how you get your test set back.** Teams that ship reliable agents are the ones with good evals.

---

## 1. Types of evals

| Level | What you check | Example |
|---|---|---|
| **Unit / deterministic** | Code assertions on output | Valid JSON? Called `calculate_bill` exactly once? Total equals expected? |
| **Model-graded (LLM-as-judge)** | A judge model scores against a rubric | "Is the answer grounded in the provided sources? 1–5" |
| **Human review** | Experts label samples | Weekly review of 50 random production traces |
| **Trajectory evals** | Was the *path* reasonable? | Right tools, right order, no wasted steps, no unsafe actions |
| **Outcome / end-to-end** | Did the environment reach the goal state? | Tests pass after the coding agent finishes; ticket resolved |
| **Online / production** | Live signals | Thumbs down rate, escalation rate, retries, cost per task |

**Prefer deterministic checks wherever possible**, and use judges for what can't be checked by code.

---

## 2. Building an eval set (eval-driven development)

1. Start with **20–50 real cases** (from users, support tickets, domain experts). Include edge cases and failures you have already seen.
2. For each case: input, expected outcome or rubric, and tags (category, difficulty).
3. Run the suite on every prompt, model or tool change, like CI.
4. Every production bug becomes a new eval case (a regression test).
5. Grow it to hundreds of cases over time. Keep a held-out set you don't tune against.

```python
CASES = [
    {"input": "Weather in Pune in F?", "expect_tools": ["get_weather", "celsius_to_fahrenheit"],
     "expect_in_answer": "87.8"},
    {"input": "Weather on Mars?", "expect_tools": ["get_weather"], "expect_in_answer": "unknown"},
]

def run_eval(agent, cases, trials: int = 3):
    results = []
    for case in cases:
        passes = 0
        for _ in range(trials):              # non-determinism → multiple trials
            answer, trace = agent(case["input"])
            tools_ok = [t.name for t in trace.tool_calls] == case["expect_tools"]
            answer_ok = case["expect_in_answer"].lower() in answer.lower()
            passes += tools_ok and answer_ok
        results.append({"input": case["input"], "pass_rate": passes / trials})
    return results
```

Metrics like **pass@k** (at least one success in k trials) and **pass^k** (all k succeed, i.e. reliability) matter for agents. Users experience pass^k.

---

## 3. LLM-as-judge, done properly

- Give the judge a **specific rubric** with criteria and examples, not "rate quality 1–10".
- Prefer **binary or small scales** (pass/fail, 1–3) and ask for reasoning *before* the verdict.
- **Calibrate** against human labels on 50–100 samples. Measure agreement before trusting it.
- Known biases: position bias (in pairwise comparisons, swap order), verbosity bias, self-preference (a model favors its own outputs, so use a different judge model when you can).

---

## 4. Observability: tracing every run

A **trace** = one request/agent run. **Spans** = each LLM call, tool call and retrieval inside it.
Capture: prompts, outputs, tool args/results, model, tokens, cost, latency, errors, user/session IDs.

- Standard: **OpenTelemetry** with the GenAI semantic conventions, so traces are portable across tools.
- Tools (as of 2026, verify): Langfuse (open source), LangSmith, Arize Phoenix, Braintrust, Weights & Biases Weave, Helicone, Datadog/Honeycomb LLM features.
- Dashboards: p50/p95 latency, error rate, cost per task, tool error rates, eval scores over time.
- Sample production traces into review queues and into your eval set.

---

## 5. Guardrails

Checks around the model, in code:

| Stage | Examples |
|---|---|
| Input | PII detection/redaction, topic filter, prompt-injection classifiers, size limits |
| Tool layer | Argument validation (Pydantic), allow-lists, permission checks, human approval for risky calls |
| Output | Schema validation, grounding/citation check, toxicity/policy filter, "no secrets" check |
| Budget | Max steps, tokens, time, dollars per run |

Guardrails reduce risk. They don't make a system safe on their own (see [10](10-security-for-agents.md)).

---

## 6. Testing LLM code in CI

- **Unit tests**: mock the LLM client and test your loop, tools, parsing and error handling deterministically.
- **Recorded fixtures**: record real responses once, replay in CI (VCR-style).
- **Eval suite**: run on PRs that touch prompts/models/tools, against a threshold (e.g. "pass rate must not drop more than 2%").
- **Smoke test** against the live model nightly.

---

## 7. Check yourself

- How do you evaluate an agent whose correct answer can be phrased many ways?
- Why run each eval case multiple times? What's pass^k?
- How would you validate that your LLM judge is trustworthy?
- What would you trace in a RAG + tools agent, and which dashboards would you build?
- A prompt change improved your eval by 5%. How do you know it isn't noise or overfitting?
