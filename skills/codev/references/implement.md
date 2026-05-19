# implement

End-to-end Salesforce change in this repo. Grab a sandbox, implement, deploy, validate, report. You do **not** open a PR — the user does that after reviewing.

Shared CLI / config / `--json` conventions live in [`../SKILL.md`](../SKILL.md). The points below are specific to implement.

## Operating context

- **Isolated environment.** Changes go to a sandbox; humans review in the PR before production. Be confident — make your best attempt, don't bail out on complexity.
- **Flxbl model:** packages are independently deployable units; domains group related packages; dependencies flow from shared/core → feature packages.
- **Salesforce specifics:** API version consistency, governor limits (prefer bulk patterns), explicit metadata dependencies, profile/perm-set reconciliation, flow activation rules, custom-setting vs custom-metadata deployment differences.

## Workflow

### 1. Understand the task

The user's args may be a Jira key, a GitHub issue URL, or a free-form description. Work from whatever they provide. If they passed an identifier and the relevant tooling is available (Jira MCP, `gh issue view`), pull context yourself — otherwise ask the user for what you need. Don't block on tracker access.

Settle on a **stable task slug** before moving on (e.g., the Jira key `PROJ-123`, or a 3-5 word kebab-case slug derived from the description like `vehicle-color-field`). You'll reuse this for both the worktree name and the sfp `--assignment-id`, so re-running codev on the same task picks up where it left off.

Then read the repo to locate the affected domain/package and identify existing patterns.

### 2. Present a plan and get confirmation

Before touching a worktree or fetching a sandbox, output a short plan and wait for the user to approve. Cover:

- **Task understanding** — 1-2 sentences on what you intend to build.
- **Task slug** — the stable slug from step 1.
- **Affected package(s)** — which packages from `sfdx-project.json` you expect to touch.
- **Components to add/modify** — one line each (e.g., "new field `Stale_Since__c` on Opportunity", "new class `StaleOpportunityFinder` in sales", "modify `SalesOpportunityTracker.cls`").
- **Notable considerations** — schema changes, cross-package dependencies, configurable values, governor-limit concerns.
- **Open questions / assumptions** — anything you're guessing at.

Then call `AskUserQuestion` with options like:
- **Proceed** — confirm the plan; move on to step 3.
- **Adjust** — let the user redirect; revise the plan and re-present.
- **Cancel** — stop the workflow.

Do not enter a worktree, fetch a sandbox, or edit any files until the user picks **Proceed**. If they pick **Adjust**, revise and re-present — don't act on a stale plan.

### 3. Enter an isolated worktree

Each codev run works in its own git worktree so the user's main checkout stays clean and parallel runs don't collide. Use the `EnterWorktree` tool with `name: codev-<task-slug>` — this creates `.claude/worktrees/codev-<task-slug>/` on a fresh branch and switches the session into it. All subsequent steps run from there.

If a worktree with that name already exists (re-running the same task), `EnterWorktree` will reuse it.

After entering the worktree, link `.sfp-pro/` from the main repo so sfp can resolve its config:

```bash
[ ! -e .sfp-pro ] && ln -s "$(dirname "$(git rev-parse --git-common-dir)")/.sfp-pro" .sfp-pro
```

`.sfp-pro/` is gitignored (user-local config) so it isn't present in a fresh worktree; symlinking inherits it from the main checkout.

### 4. Resolve repo + pool from config

```bash
REPO=$(git remote get-url origin | sed -E 's|\.git$||; s|^.*[:/]([^/]+/[^/]+)$|\1|')
POOL_TAG=$(jq -r '."pool-tag" // empty' .sfp-pro/config.json)

[ -z "$POOL_TAG" ] && { echo "pool-tag missing from .sfp-pro/config.json" >&2; exit 1; }
```

Don't try to auto-discover a pool. If `pool-tag` isn't set in config, surface that to the user and stop. The configured pool is the dev pool for ad-hoc work; review-env pools are managed separately by `sfp review-envs` and should never be poached.

### 5. Fetch the sandbox

Using the pool tag from config and the task slug as the stable assignment ID:

```bash
sfp pool fetch \
  --repository "$REPO" \
  --tag "$POOL_TAG" \
  --alias codev-dev \
  --set-default \
  --assignment-id "<task-slug>" \
  --json
```

- `--set-default` makes the fetched org the default for subsequent `sfp` commands.
- `--assignment-id` is the stable task slug from step 1 — re-running codev for the same task reuses the same sandbox.
- Parse JSON from stdout. Capture `username` for the deploy step.
- Auth is handled by the fetch; no separate `sfp org login` is needed.
- If the fetch errors on missing auth, surface that to the user — don't invent values.

### 6. Implement

- Follow the package's existing patterns and naming.
- Update or add test classes to cover the change.
- Don't hardcode configurable values; don't bypass security or governor-limit constraints.
- Check for dependents before renaming/removing anything.

### 7. Identify impacted packages

```bash
# Preferred — for committed changes
sfp impact package --basebranch main --json

# Fallback — for uncommitted work
git diff --name-only HEAD
# Then map file path → package via sfdx-project.json:
cat sfdx-project.json | jq -r '.packageDirectories[] | select(.path | contains("<path-fragment>")) | .package'
```

### 8. Deploy

```bash
# Preferred — single package
sfp push --package <pkg> --targetusername <username-from-step-5> --json

# Multiple packages in same domain
sfp push --domain <domain> --targetusername <username> --json

# Last resort — all changes, no scope (slow, risky)
sfp push --ignore-conflicts --targetusername <username> --json
```

**Hard rules:**
- NEVER run `sfp push` with no flags — it deploys the entire source tree.
- NEVER combine `--package`/`--domain` with `--ignore-conflicts` — mutually exclusive.
- Use `sfp push`, NOT `sfp validate` — validate needs a DevHub the dev pool may not have.

### 9. Run Apex tests (if applicable)

After a successful deploy, check whether the deployed package contains Apex classes. If yes, **ask the user** whether to run tests — don't run them by default (tests take time and consume org resources).

```bash
PKG_PATH=$(jq -r --arg pkg "<package>" '.packageDirectories[] | select(.package == $pkg) | .path' sfdx-project.json)
HAS_APEX=$(find "$PKG_PATH" -name "*.cls" -type f 2>/dev/null | head -1)
```

If `$HAS_APEX` is non-empty, ask the user. If they say yes:

```bash
sfp apextests trigger \
  --targetusername codev-dev \
  --testlevel RunAllTestsInPackage \
  --package <package> \
  --json
```

- `RunAllTestsInPackage` scopes the run to tests in the deployed package — fastest signal.
- Add `--validatepackagecoverage` if the user wants a coverage gate.
- Add `--async` + `sfp apextests resume` for long test runs you don't want to block on.

Parse the JSON result for pass/fail counts and code coverage; include in the final report.

If the user declines, skip and note "tests skipped per user choice" in the report.

### 10. Handle failures

**Transient (retry up to 3× with 30s delay):** "Salesforce Edge", HTTP 400/502/503/504, ECONNRESET, ETIMEDOUT, "socket hang up".

```bash
for i in 1 2 3; do
  sfp push --package <pkg> --targetusername <u> --json && break
  echo "Attempt $i failed, retrying in 30s..." >&2
  sleep 30
done
```

**Application (fix and retry):** "Component not found" → add missing dependency; "Invalid field" → fix schema reference; test failure → fix code or test; "Duplicate value" → resolve data conflict.

Do not switch CLIs on failure — read the error and fix the root cause.

### 11. Report

Generate a frontdoor URL for the sandbox so the user can open it in one click:

```bash
sfp org open --targetusername codev-dev --json | jq -r '.frontdoorUrl'
```

Output a summary including:
- Task slug / identifier
- **Worktree (open in IDE):** absolute path to `.claude/worktrees/codev-<task-slug>/` — print it on its own line so the user can click/paste it into their editor
- **Sandbox (open in browser):** the frontdoor URL from `sfp org open --json` above — one-click login, no credentials needed
- Sandbox username + assignment ID + pool tag
- Files changed (brief one-line each)
- Packages deployed + deployment status with any notable warnings
- Apex test results (if tests were run): pass/fail counts; flag pre-existing vs newly-introduced failures separately when possible

Frontdoor URLs contain a session ID — short-lived and sensitive. Don't persist them anywhere.

**Do NOT create a PR.** The user opens it after reviewing in the sandbox.

## Quick reference

| Symptom | Wrong move | Correct move |
|---|---|---|
| `sfp push` returns 4xx/5xx | Try `sf project deploy` | Retry after 30s (transient) |
| Unknown flag | Guess | `sfp <command> --help` |
| JSON has logs mixed in | Add `2>&1` | Drop the redirect |
| Repeated deploy failures | Switch CLI | Read the error, fix the cause |
