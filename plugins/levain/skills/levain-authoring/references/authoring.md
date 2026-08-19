# Recipe authoring in depth

Read this before your first change to a recipe. It covers what a
recipe is made of, how the pieces have to agree, and the SPEC.md
discipline the platform's reviewer depends on.

## What a recipe contains

```
graph.tla            the node graph and state, in PlusCal
SPEC.md              what this agent does and why — the intent record
pyproject.toml       package metadata and dependencies
src/<module>/
  <node>.py          one function per node
  _state.py          GENERATED — never hand-edit
```

`graph.tla`, the node functions, and `_state.py` must agree: every
node in the graph has a function, every state key a node touches is
declared in the spec, and `_state.py` is regenerated whenever the
spec's state changes (`levain generate-state .`).

## The graph

Nodes are annotated in the PlusCal source. A node declares the
function that implements it, the model it runs on when it calls one,
the tools it may use, and any MCP servers it needs:

```
@node {"function": "my_agent.summarize:run", "model": "claude-sonnet-5"}
```

Design rules that survive review:

- **One responsibility per node.** A node that fetches, decides, and
  publishes is hard to review, hard to debug, and expensive to retry.
- **Every loop terminates**, and the exit condition is visible in the
  spec — the platform model-checks this, so an unbounded loop fails
  validation rather than running forever.
- **State is declared, not improvised.** Each field is used by at
  least one node, and list-typed fields declare a reducer (the
  default "last write wins" is rarely what a list wants).
- **Integrations referenced by the graph must be attached to the
  agent**, or publishing is refused. Check before verifying.

The validator's messages are the authoritative reference for the
format; the worked examples in the platform's starter recipes are
the best thing to read alongside them (MCP resource `recipes://`).

## Node functions

Each node function takes the graph state and a config, and returns a
*partial* state update — only the keys it changed:

```python
async def run(state: GraphState, config: RunnableConfig) -> GraphState:
    ...
    return {"summary": text}
```

Keep I/O inside the node that owns it. Orchestration, retries, and
scheduling belong to the platform — a node that implements its own
retry loop is fighting the harness.

## SPEC.md is the intent channel

The platform's reviewer is deliberately independent: it sees the
diff and `SPEC.md`, and nothing else. It never sees the conversation
that produced the change.

So `SPEC.md` carries the *why*:

- What the agent is for, and the shape of its flow.
- Decisions that would otherwise look wrong: an unusual model choice,
  a node kept deliberately simple, a suggestion previously rejected
  and why.
- Anything a reader would otherwise flag as an oversight.

Update it in the same commit as the change it explains. A documented
decision reads as intent; the same decision undocumented reads as a
mistake and comes back as a review comment.

## Working with review feedback

Verification runs the checks and an independent review over the
pushed commit. Treat each comment as one of:

- **Valid** — fix it and push.
- **Overkill for this recipe** — reply in `SPEC.md` with the
  reasoning, then push; the next pass sees the rationale.
- **Contradicts what the user asked for** — the user's explicit
  choice wins. Record it in `SPEC.md` so the next pass stops
  re-raising it.

Resolving a thread client-side is not a verdict. The next
verification re-judges the current code, and only the platform's
verdict gates publishing.
