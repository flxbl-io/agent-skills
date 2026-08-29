# flxbl-io / agent-skills

Agent skills for [Flxbl](https://flxbl.io) — reusable, agent-loadable workflows for AI coding assistants (Claude Code, Cursor, OpenCode, Codex, and [50+ others](https://github.com/vercel-labs/skills)) working in Flxbl Salesforce projects.

Built on the [open agent skills standard](https://skills.sh) and installed via [`npx skills`](https://github.com/vercel-labs/skills).

## Skills in this repo

| Skill | What it does |
|---|---|
| **[codev](skills/codev/)** | Salesforce dev workflows backed by the [sfp](https://docs.flxbl.io/sfp) dev pool, driven by the **codev CLI**. Four modes: end-to-end implement/deploy, ad-hoc sandbox fetch, reopen an assigned sandbox, and a CI agent mode for `@claude` runs inside the `ghcr.io/flxbl-io/codev` container on GitHub Actions. |

## Install

Install all skills in this repo:

```bash
npx skills add flxbl-io/agent-skills
```

Or pick one skill:

```bash
npx skills add flxbl-io/agent-skills/codev
```

The CLI auto-detects your agent (Claude Code, Cursor, etc.) and drops the skill into the right location (e.g. `.claude/skills/codev/` for Claude Code).

## Prerequisites

These skills are designed for teams using the Flxbl framework. To use `codev`, you need **one** of:

- **The codev desktop app** with its command-line tools installed (Settings → App Configuration → Command-Line Tools) and a signed-in session — this is the developer-machine setup. The `codev` command carries the full workflow surface; `sf` / `sfdx` are not used.
- **The codev container image** (`ghcr.io/flxbl-io/codev`) with an sfp **application token** in the environment (`SFP_SERVER_TOKEN` + `SFP_SERVER_URL`) — this is the CI / GitHub Actions setup; see the skill's `references/github-actions.md` for a ready workflow template.
- **Legacy: standalone sfp-pro V3** (`sfp --version` reporting 51.x+) authenticated to the server, with `.sfp-pro/config.json` at the repo root:
  ```json
  {
    "server-url": "https://your-sfp-server.example.com",
    "server-email": "you@yourcompany.com",
    "pool-tag": "your-dev-pool-tag"
  }
  ```
  Every command in the skill is identical between `codev` and `sfp` — only the binary name differs.

Plus, in all setups: **sfp server access with a configured dev pool** and **a Flxbl-style repo** with `sfdx-project.json` describing packages and (typically) domains. Without a server and a dev pool, the skill has nothing to fetch sandboxes from. If you're a Flxbl customer and need access, contact your Flxbl admin or reach out via [flxbl.io](https://flxbl.io).

## Skill design

Each skill follows the slim-router pattern:

```
skills/<name>/
  SKILL.md           ← always loaded by the agent; routes to references
  references/        ← detail files, loaded on demand
    <mode-1>.md
    <mode-2>.md
```

`SKILL.md` carries the shared operating context (CLI conventions, config resolution, `--json` stdout/stderr handling, secret-hygiene rules). Each `references/*.md` is a self-contained workflow for one mode. This keeps the always-on context small and lets agents pull detail only when they need it.

## License

[MIT](LICENSE) — use freely, modify freely, no warranty.
