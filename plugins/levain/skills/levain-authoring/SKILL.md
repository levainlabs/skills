---
name: levain-authoring
description: Build and change Levain agents — create a draft version, edit the recipe, validate it, verify, and publish. Use when the user asks to create, edit, debug, or publish a Levain agent or recipe, mentions graph.tla, a recipe node, or a Levain draft or version, or asks to change what one of their Levain agents does. Not for running or monitoring existing agents.
license: Apache-2.0
compatibility: Needs the Levain MCP server connected and the levain CLI signed in (both OAuth; no API key). Plus git, uv/uvx, and network access to your Levain deployment.
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

## Authentication

Two OAuth sign-ins, no API key anywhere, and **no credential ever
passes through this conversation**.

1. **The Levain MCP server** carries every step except cloning and
   pushing. If its tools aren't available, ask the user to connect
   it — that is the fix, not a key.
2. **The `levain` CLI** handles git. Check with `levain whoami`; if
   it fails, run `levain login` and tell the user to open the URL it
   prints and enter the code. The CLI then mints its own short-lived,
   repo-scoped credentials whenever git needs one.

You should never see, ask for, or handle a secret. If the user offers
a key, tell them it isn't needed.

## Instructions

### Step 1: Confirm the target

List the workspace's agents and versions before touching anything:
`list_agents`, then `list_agent_versions`.

Agree with the user on which agent, and whether to edit an existing
draft or fork a new version.

### Step 2: Create a draft

`create_agent_version`. Published versions reject pushes — always
work on a draft.

### Step 3: Get a working copy

Ask the platform where the source lives with
`get_version_source` — never construct the remote URL yourself. It
returns the `remote` and the version's `branch`.

Clone with the CLI as git's credential helper:

```bash
git clone -c credential.helper='!levain git-credential' -c credential.useHttpPath=true "<remote>" <dir>
cd <dir> && git checkout <branch>
```

git hands the helper the repository it wants, and the CLI mints a
credential scoped to exactly that, valid for about an hour. Only the
helper command is stored in the clone's config, so pushes keep
working and no secret is ever written to disk or shown to you.

Do **not** put a credential in the remote URL — git would persist it
in `.git/config` and echo it back from `git remote -v`.

If the environment has no git identity (fresh containers often
don't), set one on the clone before committing:
`git config user.name <name>` and `git config user.email <email>`,
with the user's details.

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

This validates the spec, model-checks the graph, and fails if
`_state.py` is missing or stale — regenerate it with
`levain generate-state .` after any change to the graph's state and
commit the result. Read its errors literally — they are the format's
documentation. Expected output on success: `VALIDATION PASSED` plus a
node and transition summary.

Without a Java runtime it skips model checking and says so; the
platform runs the full pipeline server-side regardless.

### Step 6: Push

```bash
git push origin <branch>
```

### Step 7: Server checks — free

`run_checks`. The platform lints the pushed tree exactly as
verification's first stage does (recipe validation, ruff, mypy) with
no reviewer and no cost, and never edits the branch. It returns
immediately; poll `get_session` with the returned `session_id` — the
round's `build_outcome` carries `checks_passed`, and
`failure_excerpt` holds the lint output when it fails.

Fix, push, and re-run until it passes. Only then is the paid
verification worth dispatching.

### Step 8: Verify

`verify_version`. It returns immediately; poll `get_session` until
the round finishes — its `build_outcome` is the verdict.

Before calling it, walk `references/review-checklist.md` — each issue
you catch is a verification round you don't pay for.

The pass is report-only: the reviewer records its verdict and
comments and never edits your branch. Read the feedback with
`get_version_review_threads`. Fix what is valid, push, and verify
again. Marking a thread resolved does not resolve it — the next pass
re-judges the code either way.

### Step 9: Publish

`publish_version`, or pass `publish: true` to `verify_version` to
publish automatically on a green verdict.

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

**Levain tools are missing** — the MCP server isn't connected. Ask
the user to connect it rather than falling back to an API key.

**`Authentication failed` on clone or push** — the CLI isn't signed
in, or its session expired. Run `levain login` again; the helper
mints fresh credentials on its own after that.

**`levain: command not found` from git** — the helper runs `levain`
from your PATH, so it needs a persistent install:
`uv tool install levain-cli`. A one-shot `uvx` invocation isn't on
PATH when git calls out.

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

**`404` on clone from a repo you can see** — credentials are scoped
to one repository, so a mismatch reads as not found rather than
denied.

**`uvx` cannot find `levain-cli`** — set `LEVAIN_CLI_FROM` to a
version spec or a local checkout of the toolchain.
