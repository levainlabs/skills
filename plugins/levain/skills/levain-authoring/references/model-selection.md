# Model selection

For every LLM-backed node, choose an engine and `model` — and judge
those choices in review — from the catalog, not memory:

```
python -m levain_sdk.catalog
```

It lists the engines, every routable model with list prices, and each
model's lane (`included` = platform lane; `byok` = needs the
customer's provider key), versioned with the SDK so it matches what
this workspace can route. Never rely on memorized ids, prices, or
rankings.

Principles:

- **The customer's explicit choice wins.** A model named in chat or
  `SPEC.md` is a requirement — apply it, record it in the spec, never
  relitigate it. Flag only picks that cannot run (unsupported engine,
  unroutable lane).
- **No vendor defaults.** Pick by the node's job against the catalog.
  A specialist beats a flagship in its lane — and only there. Ties go
  to `included` availability.
- **Escalate on evidence.** Start each node on the cheapest plausible
  model; move up only when its checks or evals actually fail, not
  because the task sounds hard. Nodes inside checked loops get
  quality from the loop; only unguarded one-shot nodes need the model
  as the quality floor.
- **Cost concentrates in loops** — scrutinize in proportion to
  invocation count.
- **Reasoning is task-gated.** Worth it for math, hard multi-step
  code, and complex analysis; on writing, summarization, extraction,
  classification, and translation it adds cost, latency, and room to
  confabulate. Deep effort means slow first tokens — background nodes
  only, never interactive ones.
- **Tune the engine before jumping tiers** (Claude `effort`, Hermes
  `max_iterations`) on work that benefits from reasoning at all.
- **Benchmarks eliminate, they don't rank.** Saturated or
  harness-dependent scores rule out weak models but can't split
  leaders. Decisions that matter run on the customer's own schemas
  and documents; subjective quality (writing, voice) on human
  preference, not autograders.
- **Usable context is well short of advertised context**, and long
  requests can bill higher — prefer retrieval or chunking to
  window-stuffing; never design a node that needs the full limit.
- **High-stakes output needs verification, not a bigger model.**
  Citations, medical/legal claims, money movement: add a grounding or
  verification step, or a human checkpoint. A stronger model shrinks
  the error rate; it never replaces the check.
- **Unroutable picks fail loudly at call time** — make a `byok` pick
  deliberate, with a preflight when it guards expensive work.

In review: flag over-powered models on trivial steps, under-powered
models on unguarded judgment steps, expensive models in loops,
thinking on nodes it doesn't help, and missing verification on
high-stakes output. Never flag the customer's explicit choice.
