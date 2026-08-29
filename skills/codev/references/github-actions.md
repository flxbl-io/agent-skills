# ci-agent — codev inside GitHub Actions

Running as an AI agent in the `ghcr.io/flxbl-io/codev` container on GitHub Actions (typically triggered by `@claude` on an issue/PR, or a workflow_dispatch). This reference covers what differs from a developer machine; combine it with [`implement.md`](implement.md) for the dev-loop itself. Everything here was field-verified end-to-end on 2026-08-29: a bare "@claude use codev to work on this" comment on an issue produced sandbox fetch → push → apex tests (5/5 pass) → unassign → an agent-opened PR with real evidence (flxbl-io/sf-core-dev#555).

## What's different in CI

- **Auth is the application token**: `SFP_SERVER_TOKEN` + `SFP_SERVER_URL` are preset. Never ask for credentials; if they're missing, the workflow is misconfigured — say so and stop.
- **No human in the loop mid-run — but there ARE turns.** The interactive plan-confirmation step from implement.md does not apply: state your plan in your output and proceed. Tactical choices you'd normally ask about (run tests? which package? naming?) — make the conservative call and record it; the PR review is where a human redirects you, cheaply.
- **Ask only when the gap is load-bearing.** Some gaps you cannot fill conservatively: undefined business rules (thresholds, tier boundaries, rounding), contradictory requirements, or choices that are costly to reverse. Inventing those wastes a sandbox cycle and a review round; a question costs one turn. When you hit one, put the clarifying question(s) at the end of your output — it lands on the issue as your comment — and stop **before fetching any org** (never park a sandbox behind an open question).
- **Every `@claude` comment is a fresh run with the full issue/PR thread as context** — that thread IS the conversation. On every run, read the whole thread as the current task state: an earlier turn of yours may have asked a question that the newest comment answers; resume from there rather than starting over.
- **The PR is the deliverable** (this inverts implement.md's "do NOT create a PR" rule, which is for interactive runs where the user reviews the sandbox first). Commit to a fresh branch, push, open a PR to the default branch. The PR body must contain the REAL command outputs you observed — pool fetch result (alias/username), push summary, apex test results. Never fabricate output; if a step failed, quote the exact error and mark it clearly.
- **Always return the org** with `codev pool unassign` before finishing, pass or fail. A CI run that strands a sandbox starves the pool.
- **The container ships a curated codev command set** (pool, pull/push, apextests, package create/install, org login/list/open, project status, analyze). Out-of-scope commands exit 2 with a pointer — that's by design, not an error to work around.

## Proven command shapes

```bash
# assignment-id: derive from the issue (e.g. issue-554) — stable across turns,
# so a task resumed after a clarifying question reuses the SAME sandbox
codev pool fetch --repository "$SFP_REPOSITORY" --tag "$POOL_TAG" \
  --alias dev-org --assignment-id "issue-<number>" --json

codev push --package <pkg> --targetusername dev-org --json

codev apextests trigger --targetusername dev-org \
  --testlevel RunSpecifiedTests --specifiedtests <TestClass> --wait 30 --json
# (RunAllTestsInPackage --package <pkg> for the package-wide signal)

codev pool unassign --repository "$SFP_REPOSITORY" --tag "$POOL_TAG" \
  --assignment-id "<stable-task-slug>" --json
```

Note: with application-token auth, pool assignments are recorded under the token owner's identity as `app:<email>` — expect that form when listing/filtering assignments.

## Creating the PR — the token trap

GitHub Enterprise commonly forbids the Actions installation token from creating PRs (`GraphQL: GitHub Actions is not permitted to create or approve pull requests`). Two facts to navigate it:

1. The `GH_TOKEN`/`GITHUB_TOKEN` visible in your session is the **installation token** — the claude-code-action overrides these env vars even when the workflow set something else. Branch pushes work with it; `gh pr create` does not.
2. The workflow should therefore inject a PAT under a **different env name** (convention: `AGENT_GH_PAT`). Use it explicitly:

```bash
GH_TOKEN="$AGENT_GH_PAT" gh pr create --base main --title "..." --body-file /tmp/pr-body.md
```

If no PAT is present and PR creation is refused: push the branch, write the full evidence into your final output together with the compare URL (`https://github.com/<repo>/compare/main...<branch>?expand=1`), and say exactly why the PR could not be opened. Never claim a PR exists when it doesn't.

## Workflow template (for repo maintainers)

A generic `@claude` workflow that runs the agent inside the codev container. Repo/org **secrets** needed: `CLAUDE_CODE_OAUTH_TOKEN` (Claude subscription OAuth token — NOT an `sk-ant-api…` key; an API key would instead go to the `anthropic_api_key` input), `SFP_SERVER_TOKEN` (sfp application token), `GHA_TOKEN` (PAT: GHCR pull + repo scope for PR creation). Repo/org **Actions variables** needed: `SFP_SERVER_URL` (the tenant, e.g. `https://dev6.flxbl.io`) and `SFP_POOL_TAG` (the dev pool) — variables, not literals, so one workflow file serves every repo/tenant. Note the token is only valid on the server that issued it: changing `SFP_SERVER_URL` without swapping `SFP_SERVER_TOKEN` yields 401s.

The ready-to-copy workflow lives at [`../assets/claude.yml`](../assets/claude.yml) — drop it into `.github/workflows/claude.yml` verbatim. It is the exact file proven on the live e2e runs (container credentials, gh install, `safe.directory`, `issues: write` for the tracking comment, `claude_code_oauth_token`, and the `AGENT_GH_PAT` convention), parameterized entirely through the secrets and Actions variables listed above.

With this skill committed in the repo (`.claude/skills/codev/`), the comment can simply be: *"@claude use codev to work on this"* — the skill supplies the rest.
