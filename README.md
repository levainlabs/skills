# Levain skills

Skills and plugins for building [Levain](https://levainlabs.com) agents
from a coding agent.

A Levain agent is a **recipe**: a `graph.tla` spec declaring the node
graph and state, plus a Python package with one function per node.
The `levain` plugin teaches Claude how to author one — fork a draft
version, edit the recipe, validate it locally, verify, and publish —
so the iteration runs on your Claude subscription rather than metered
platform tokens.

## Install

```
/plugin marketplace add levainlabs/skills
/plugin install levain
```

Or use the skill on its own, without the plugin:

```
cp -r plugins/levain/skills/levain-authoring ~/.claude/skills/
```

The skill uses only the portable Agent Skills frontmatter fields, so
it works on claude.ai and through the API as well as in Claude Code.

## Configure

Two sign-ins, both OAuth, no API key:

1. Connect the **Levain MCP server** in your client.
2. Install the CLI and sign in:

```bash
uv tool install levain-cli
levain login
```

`levain login` opens a device-code flow — visit the URL it prints,
enter the code, done. After that the CLI acts as git's credential
helper and mints short-lived, repo-scoped credentials on demand, so
no secret is stored in `.git/config`, kept in your environment, or
shown to the agent.

## How it's put together

`SKILL.md` carries the workflow. The detail lives in `references/`
and is read only when it's needed — the recipe format and the
`SPEC.md` discipline the platform's reviewer depends on, the
pre-flight checklist, and model-selection guidance.

The network is used only for what the bundle can't know: where your
agent's source lives, the current model catalog and prices, and the
platform's own verification and publish steps. Local checks run
through [`levain-cli`](https://pypi.org/project/levain-cli/):

```bash
uvx --from levain-cli levain validate-package .
```

Passing locally is necessary but not sufficient — the platform
re-validates every push and gates publishing on its own review.

## Contributing

This repository is generated, so edits here are overwritten. Open an
issue instead, or reach us at [levainlabs.com](https://levainlabs.com).

## License

Apache-2.0. See [LICENSE](LICENSE).
