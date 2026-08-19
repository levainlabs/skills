---
name: levain-authoring
description: Build and change Levain agents — create a draft version, edit the recipe, validate it, verify, and publish. Use when the user asks to create, edit, debug, or publish a Levain agent or recipe, mentions graph.tla, a recipe node, or a Levain draft or version, or asks to change what one of their Levain agents does. Not for running or monitoring existing agents.
license: Apache-2.0
compatibility: Requires a workspace-scoped Levain API key (LEVAIN_API_KEY), git, and uv/uvx. Needs network access to your Levain deployment.
metadata:
  author: Levain
  version: 0.1.0
  mcp-server: levain
---

# Authoring Levain agents

A Levain agent is a **recipe**: a `graph.tla` spec declaring the node
graph and state, plus a Python package with one function per node.
Versions are the unit of change — `draft` versions are editable,
`active` and `archived` ones are immutable.

Verification is the authority. Local checks catch mistakes early, but
the platform re-validates and reviews every push, and that verdict is
what gates publishing.

## Instructions

### Step 1: Confirm the target

List the workspace's agents and versions before touching anything:

```bash
curl -sS -H "Authorization: Bearer $LEVAIN_API_KEY" \
  "${LEVAIN_API:-https://api.levainlabs.com}/api/v1/agents/"
```

Agree with the user on which agent, and whether to edit an existing
draft or fork a new version. If the Levain MCP server is connected,
`list_agents` and `list_agent_versions` do the same thing.

### Step 2: Create a draft

`POST /api/v1/agents/{agent}/versions` (MCP: `create_agent_version`).
Published versions reject pushes — always work on a draft.

### Step 3: Get a working copy

Ask the platform for the remote; never construct the URL yourself:

```bash
curl -sS -H "Authorization: Bearer $LEVAIN_API_KEY" \
  "${LEVAIN_API:-https://api.levainlabs.com}/api/v1/agents/{agent}/versions/{n}/source"
```

It returns `remote` and `branch`. Clone with the API key as the
password, any username:

```bash
git clone "https://x:$LEVAIN_API_KEY@<remote-host>/<path>.git" <dir>
cd <dir> && git checkout <branch>
```

### Step 4: Edit

Read `references/authoring.md` before your first change to this
recipe — it covers the graph, state, node design, and the SPEC.md
discipline the reviewer depends on.

The single rule worth repeating here: **record decisions in
`SPEC.md`**, in the same commit as the change. The reviewer sees only
the diff and `SPEC.md`, never this conversation, so an undocumented
deliberate choice reads as a mistake and gets flagged.

For model choices, read `references/model-selection.md`, and fetch
the live catalog for what is actually available and at what price
(MCP resource `guide://models-catalog`).

### Step 5: Check locally

```bash
uvx --from "${LEVAIN_CLI_FROM:-levain-cli}" levain validate-package .
```

This validates the spec, model-checks the graph, and regenerates
`_state.py`. Read its errors literally — they are the format's
documentation. Expected output on success: `VALIDATION PASSED` plus a
node and transition summary.

Without a Java runtime it skips model checking and says so; the
platform runs the full pipeline server-side regardless.

### Step 6: Push

```bash
git push origin <branch>
```

### Step 7: Verify

`POST /api/v1/agents/{agent}/versions/{n}/verify` (MCP:
`verify_version`). It returns immediately; poll the run until it
finishes.

Before calling it, walk `references/review-checklist.md` — each issue
you catch is a verification round you don't pay for.

Then read the feedback:
`GET /api/v1/agents/{agent}/versions/{n}/review-threads` (MCP:
`get_version_review_threads`). Fix what is valid, push, verify again.
Marking a thread resolved does not resolve it — the next pass
re-judges the code either way.

Note that verification may itself commit to the branch (it
regenerates `_state.py`, and its review loop can apply fixes). Pull
before you continue editing.

### Step 8: Publish

`PATCH /api/v1/agents/{agent}/versions/{n}` with `{"status":"active"}`
(MCP: `publish_version`), or pass `publish: true` to `verify_version`
to publish automatically on a green verdict.

Show the user the diff and what verification said before publishing.

## Examples

**Example 1: change a prompt**
User says: "make the summarizer less verbose"
Actions: confirm agent → fork draft → clone → edit the node's prompt
file and note the intent in SPEC.md → validate → push → verify →
publish.
Result: new active version, one round of verification.

**Example 2: add a node**
User says: "have it post the digest to Slack too"
Actions: read `references/authoring.md` → add the node to `graph.tla`
with its state fields → write the node function → regenerate
`_state.py` via `levain generate-state .` → confirm the Slack
integration is attached to the agent → validate → push → verify.
Result: extended graph; publishing fails until the integration is
attached, so check that before verifying.

## Troubleshooting

**401 on an API call** — `LEVAIN_API_KEY` is unset, wrong, or
revoked. Org-scoped keys work for the API but not for git; use a
workspace-scoped key.

**`Not allowed to push to protected branch`** — that version is
published and immutable. Fork a new draft and re-apply the change.

**409 on verify: "a build turn is currently running"** — a build or
verification is already in flight for that agent. Wait for it, then
retry.

**Validation fails on `_state.py`** — it is generated, never
hand-edited. Run `levain generate-state .` and commit the result.

**Publish rejected on a green-looking draft** — publishing needs a
recorded verdict from a verification pass on the *current* commit,
and every integration the graph references must be attached to the
agent. Re-verify after your last push.

**`uvx` cannot find `levain-cli`** — set `LEVAIN_CLI_FROM` to a
version spec or a local checkout of the toolchain.
