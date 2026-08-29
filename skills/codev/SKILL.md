---
name: codev
description: Salesforce development workflows backed by the sfp dev pool, driven by the codev CLI. Four modes — (1) implement, autonomous end-to-end build/fix on a fresh sandbox with deploy and report; (2) sandbox-fetch, grab a fresh sandbox for ad-hoc work like exploring, debugging, manual testing; (3) sandbox-open, reopen a sandbox already assigned to you; (4) CI agent runs inside the codev container on GitHub Actions (@claude on an issue/PR). Use when the user asks to build/fix in this repo, asks for a sandbox/org, asks to reopen one they already have, or when running as an agent in CI.
---

# codev

Salesforce development workflows for this repo, backed by the sfp dev pool and driven by the **codev CLI**. Pick the mode by the user's intent:

| User intent | Mode | Reference |
|---|---|---|
| "Build/fix X", "implement Y", end-to-end task | **implement** | [references/implement.md](references/implement.md) |
| "Grab me a sandbox", "give me an org for X", ad-hoc work | **sandbox-fetch** | [references/sandbox-fetch.md](references/sandbox-fetch.md) |
| "Open my sandbox", "reopen the one for X" | **sandbox-open** | [references/sandbox-open.md](references/sandbox-open.md) |
| Running in GitHub Actions / the codev container (`@claude` on an issue or PR) | **ci-agent** | [references/github-actions.md](references/github-actions.md) |

Read the matching reference file before acting. Each reference is self-contained; you do not need to read the others. In CI, read **github-actions.md first**, then the mode reference for the task itself (usually implement).

## Shared operating context (all modes)

- **CLI:** the command is **`codev`** — the developer CLI shipped by the codev desktop app ("Install command-line tools" in its settings) and by the `ghcr.io/flxbl-io/codev` container image. On legacy setups where only the standalone `sfp` binary exists, every command and flag below is identical — substitute the binary name once:
  ```bash
  CLI=codev; command -v codev >/dev/null 2>&1 || CLI=sfp
  ```
  `sf` and `sfdx` are NOT the tools for these workflows — don't fall back to them for pool/push/test operations even if present. When `codev` errors, run `codev <cmd> --help` rather than guessing flags.
- **Auth — two modes, detect before doing anything:**
  1. **Application token (CI, containers):** `SFP_SERVER_TOKEN` and `SFP_SERVER_URL` are preset in the environment; auth just works — never ask for credentials, never pass `--email`. `SFP_REPOSITORY`, when set, is the default `--repository`.
  2. **Desktop session (developer machines):** codev reuses the signed-in codev app session. Exit code **78** with "codev requires a signed-in codev app" means the user must open the app and sign in — surface that and stop; there is no workaround and you must not ask for passwords. (Legacy standalone `sfp` instead reads `.sfp-pro/config.json` — see below.)
- **Repo identifier:** honor `SFP_REPOSITORY` if set, else derive from `origin` — don't hardcode.
  ```bash
  REPO=${SFP_REPOSITORY:-$(git remote get-url origin | sed -E 's|\.git$||; s|^.*[:/]([^/]+/[^/]+)$|\1|')}
  # e.g. flxbl-io/sf-core-staging
  ```
- **Pool tag:** resolve in this order — `SFP_POOL_TAG` env, the task/user's instruction, then `.sfp-pro/config.json` (`pool-tag` key) when that file exists (developer machines):
  ```bash
  POOL_TAG=${SFP_POOL_TAG:-$(jq -r '."pool-tag" // empty' .sfp-pro/config.json 2>/dev/null)}
  [ -z "$POOL_TAG" ] && { echo "pool tag not resolvable (SFP_POOL_TAG env, task instruction, or .sfp-pro/config.json)" >&2; exit 1; }
  ```
  Don't auto-discover a pool from `codev pool list` unless the task explicitly names one or there is exactly one dev pool — review-env pools are managed separately — do not take instances from them. If `.sfp-pro/config.json` isn't present and no env/instruction supplies the tag, surface that rather than silently failing.
- **Data packages (`type: "data"`, sfdmu-based) deploy through `codev push` / `codev pull` in the dev loop** — the same commands as source, routed to sfdmu when `-p` names a data package (per-record packages included). Do not reach for `codev package create data` to "build" one: in sfp-pro V3 that command *registers* the package (sfdx-project entry + git-diff baseline), producing no installable artifact; and `codev package install` accepts only `name:04t…` subscriber versions (unlocked/2GP) and rejects data packages — that command pair does not apply to data packages.
  For the **versioned artifact lifecycle** (build once, install with skip-if-already-installed), use `codev build` / `codev install` instead — the sfpowerscripts artifact tooling, distinct from `package create`/`package install`. `codev build -p <pkg> --branch <branch> -v <devhub-alias>` writes a semver-versioned zip to `artifacts/`; `codev install -o <org> --artifactdir artifacts --skipifalreadyinstalled -b <org>` installs it via sfdmu and records the installed version against the org, so a second identical install reports `To be installed? No` / `0 artifacts installed`. `-v/--devhubalias` is required by `codev build` even for data packages (no scratch org is actually consumed) — if no devhub is authenticated locally, run `codev org login --server --default-devhub --set-default-dev-hub --alias devhub` first. Verified live 2026-08-29 (flxbl-io/sf-core-dev#561).
- **stdout/stderr with `--json`:** `codev ... --json` writes JSON to stdout and logs to stderr. Do **not** redirect stderr — it corrupts the JSON.
  ```bash
  # WRONG
  codev pool fetch ... --json 2>&1 | jq .
  # CORRECT
  codev pool fetch ... --json | jq .
  # CORRECT — capture JSON, stderr still visible
  result=$(codev pool fetch ... --json)
  echo "$result" | jq -r '.username'
  ```
- **Session secrets — never print to your output:** frontdoor URLs (the `https://<org>/secur/frontdoor.jsp?sid=...` form) and `sfdxAuthUrl` values contain live session credentials good for hours. Your output is captured in conversation transcripts, logs, and screenshots — printing a session URL "just once in the report" still leaks it. Open the browser via `codev org open --targetusername <alias>` **without** `--json` — codev opens the browser locally as a side effect and the URL never surfaces. If the user wants to open the org in a different browser later, tell them to rerun the same `codev org open` command themselves; do not capture or paste the URL on their behalf. Never write these values to files either. Pass `sfdxAuthUrl` to `codev org login` via stdin (`--url-stdin -`), never as an argument.
