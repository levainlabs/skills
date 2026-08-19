# Pre-flight checklist

What the platform's review looks at. Walk it before verifying —
every issue caught here is a verification round you don't pay for.

This is the shape of the review, not its judgment: the reviewer runs
server-side against your pushed commit and decides for itself.

- **Intent** — does the change do what `SPEC.md` says, all of it, and
  nothing that wasn't asked for?
- **Coherence** — `graph.tla`, the node functions, and `_state.py`
  agree; `_state.py` is freshly generated.
- **Node design** — one responsibility per node; the model on each
  node matches the work it does.
- **Loops** — every cycle terminates, with the exit condition visible
  in the spec.
- **Boundaries** — nodes do their own work; orchestration, retries,
  and scheduling are the platform's job.
- **Prompts** — specific, scoped to the node's task, consistent with
  the tools that node is given.
- **State** — every field documented and actually used; list fields
  declare a reducer.
- **Python** — signatures match the node contract, returns are
  partial state updates, no dead code or unused imports.
- **Package** — `pyproject.toml` correct, dependencies declared,
  nothing referenced that isn't there.
- **Integrations** — everything the graph references is attached to
  the agent (publishing is refused otherwise).

Deliberately deviating from any of these is fine — say so in
`SPEC.md`. The reviewer treats a documented decision as intent and an
undocumented one as an oversight.
