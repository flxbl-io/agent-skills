# ci-agent — codev inside GitHub Actions

Running as an AI agent in the `ghcr.io/flxbl-io/codev` container on GitHub Actions (typically triggered by `@claude` on an issue/PR, or a workflow_dispatch). This reference covers what differs from a developer machine; combine it with [`implement.md`](implement.md) for the dev-loop itself. Everything here was field-verified end-to-end (sandbox fetch → push → apex tests → unassign → branch push) on 2026-08-29.

## What's different in CI

- **Auth is the application token**: `SFP_SERVER_TOKEN` + `SFP_SERVER_URL` are preset. Never ask for credentials; if they're missing, the workflow is misconfigured — say so and stop.
- **No human in the loop mid-run.** The interactive plan-confirmation step from implement.md does not apply: state your plan in your output and proceed. Decisions you'd normally ask about (run tests? which package?) — make the conservative call and record it.
- **The PR is the deliverable** (this inverts implement.md's "do NOT create a PR" rule, which is for interactive runs where the user reviews the sandbox first). Commit to a fresh branch, push, open a PR to the default branch. The PR body must contain the REAL command outputs you observed — pool fetch result (alias/username), push summary, apex test results. Never fabricate output; if a step failed, quote the exact error and mark it clearly.
- **Always return the org** with `codev pool unassign` before finishing, pass or fail. A CI run that strands a sandbox starves the pool.
- **The container ships a curated codev command set** (pool, pull/push, apextests, package create/install, org login/list/open, project status, analyze). Out-of-scope commands exit 2 with a pointer — that's by design, not an error to work around.

## Proven command shapes

```bash
codev pool fetch --repository "$SFP_REPOSITORY" --tag "$POOL_TAG" \
  --alias dev-org --assignment-id "<stable-task-slug>" --json

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

A generic `@claude` workflow that runs the agent inside the codev container. Repo/org secrets needed: `CLAUDE_CODE_OAUTH_TOKEN` (Claude subscription OAuth token — NOT an `sk-ant-api…` key; an API key would instead go to the `anthropic_api_key` input), `SFP_SERVER_TOKEN` (sfp application token), `GHA_TOKEN` (PAT: GHCR pull + repo scope for PR creation).

```yaml
name: claude
on:
  issue_comment:
    types: [created]
permissions:
  contents: write
  pull-requests: write
  issues: read
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    timeout-minutes: 90
    container:
      image: ghcr.io/flxbl-io/codev:latest
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GHA_TOKEN }}
    env:
      SFP_SERVER_URL: https://<your-sfp-server>
      SFP_SERVER_TOKEN: ${{ secrets.SFP_SERVER_TOKEN }}
      SFP_REPOSITORY: ${{ github.repository }}
      SFP_POOL_TAG: <your-dev-pool-tag>
      AGENT_GH_PAT: ${{ secrets.GHA_TOKEN }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install GitHub CLI (the codev image does not ship gh)
        run: |
          set -euo pipefail
          curl -fsSL -o /tmp/gh.tgz https://github.com/cli/cli/releases/download/v2.76.2/gh_2.76.2_linux_amd64.tar.gz
          tar -xzf /tmp/gh.tgz -C /tmp
          install -m 0755 /tmp/gh_2.76.2_linux_amd64/bin/gh /usr/local/bin/gh
      - name: Mark workspace safe for git
        # container bind-mounts the workspace under a different uid; without
        # this, git discovery fails ("not in a git directory") inside the action
        run: |
          git config --global --add safe.directory "$GITHUB_WORKSPACE"
          git config --global --add safe.directory '*'
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # explicit token skips the action's GitHub-App OIDC exchange
          # (which needs id-token:write + the Claude GitHub App installed)
          github_token: ${{ secrets.GITHUB_TOKEN }}
          claude_args: --allowedTools "Bash,Read,Write,Edit,Glob,Grep"
```

With this skill committed in the repo (`.claude/skills/codev/`), the comment can simply be: *"@claude use codev to work on this"* — the skill supplies the rest.
