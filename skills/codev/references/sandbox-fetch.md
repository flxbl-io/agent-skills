# sandbox-fetch

Fetch one sandbox from the configured sfp pool and report enough for the user to open it. Nothing else — no worktree, no edits, no deploy. If the user wants the full implement/deploy/test loop, use [`implement.md`](implement.md) instead.

Shared CLI / config / `--json` conventions live in [`../SKILL.md`](../SKILL.md).

## Workflow

### 1. Decide assignment-id and alias

The user's args may include a slug they want to reuse (e.g. `poc-flow-test`). If they passed one, use it verbatim — re-running with the same slug reuses the same sandbox (sfp's `--assignment-id` is stable). If they didn't, generate a fresh one:

```bash
ASSIGNMENT_ID="${1:-adhoc-$(date +%Y%m%d-%H%M)}"
ALIAS="sbx-$ASSIGNMENT_ID"
```

An auto-generated `adhoc-…` id always pulls a new sandbox; a user-supplied slug pins to the same one across runs.

### 2. Resolve repo + pool from config

```bash
REPO=$(git remote get-url origin | sed -E 's|\.git$||; s|^.*[:/]([^/]+/[^/]+)$|\1|')
POOL_TAG=$(jq -r '."pool-tag" // empty' .sfp-pro/config.json)

[ -z "$POOL_TAG" ] && { echo "pool-tag missing from .sfp-pro/config.json" >&2; exit 1; }
```

If `.sfp-pro/config.json` doesn't exist in the current directory (e.g. you're in a worktree), surface that to the user rather than silently failing — they likely want to run this from the main repo checkout.

### 3. Fetch the sandbox

```bash
sfp pool fetch \
  --repository "$REPO" \
  --tag "$POOL_TAG" \
  --alias "$ALIAS" \
  --assignment-id "$ASSIGNMENT_ID" \
  --json
```

Notes:
- No `--set-default` — this mode is report-only and must not stomp on the user's current default org.
- Parse JSON from stdout. Capture `username` for the report.
- Auth is handled by the fetch; no separate `sfp org login` is needed.
- If the fetch errors on missing auth, surface the message — don't invent values.

### 4. Open the sandbox in the browser

```bash
sfp org open --targetusername "$ALIAS"
```

Drop `--json` deliberately. With `--json`, sfp returns the frontdoor URL (which carries a live session credential) and skips opening the browser. Without `--json`, sfp opens the browser locally; the URL stays on the user's machine and never enters your output, the transcript, or logs.

**Never capture, paste, or print the frontdoor URL yourself**, even in a report. If the user later wants to open the sandbox in a different browser, tell them to rerun `sfp org open --targetusername $ALIAS` themselves.

### 5. Report

Output a compact summary:

- **Sandbox:** opened in browser. Reopen anytime with `sfp org open --targetusername $ALIAS`.
- **Alias:** `$ALIAS` (use this with subsequent `sfp ... --targetusername $ALIAS` calls)
- **Username:** from the fetch response
- **Assignment ID:** `$ASSIGNMENT_ID` (pass this same value to re-fetch the same sandbox later)
- **Pool tag:** `$POOL_TAG`
- **Expiry / lease info:** if the fetch JSON includes it

Do **not** include the frontdoor URL anywhere in the report.

That's it. Do not enter a worktree, do not edit files, do not run `sfp push`. If the user later asks to build something on this sandbox, switch to the **implement** mode.
