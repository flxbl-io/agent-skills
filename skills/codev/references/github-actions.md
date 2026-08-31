# github-actions

Running as an AI agent in the `ghcr.io/flxbl-io/codev` container on GitHub Actions, typically triggered by `@claude` on an issue or PR. This reference covers only what differs from a developer machine; combine it with [`implement.md`](implement.md) for the dev loop itself. Workflow setup (secrets, variables, the ready-to-copy workflow file) is documented in the repository README — it is maintainer configuration, not agent context.

## What's different in CI

- **Auth is the application token.** `SFP_SERVER_TOKEN` and `SFP_SERVER_URL` are preset. Never ask for credentials; if they are missing, the workflow is misconfigured — say so and stop.
- **The PR is the deliverable.** This inverts implement.md's "do not create a PR" rule, which applies to interactive runs where the user reviews the sandbox first. Commit to a fresh branch, push, and open a PR to the default branch. The PR body must contain the real command outputs you observed — pool fetch result, push summary, test results. Never fabricate output; if a step failed, quote the exact error and mark it clearly.
- **Always return the org** with `codev pool unassign` before finishing, pass or fail — a run that keeps its sandbox reduces the pool for everyone.
- **The container ships the curated codev command set.** Out-of-scope commands exit 2 with a pointer message — that is by design, not an error to work around.
- **The workflow establishes telemetry context.** `CODEV_AGENT_TASK_ID` identifies this issue/PR thread. Each curated `codev` command reports a best-effort counter against it; do not unset or replace it. The image does not contain `sfp`, and this skill never invokes it.

## Turns and clarifying questions

- Each `@claude` comment is a fresh run carrying the full issue/PR thread — the thread is the conversation. On every run, read the whole thread as the current task state: an earlier turn of yours may have asked a question that the newest comment answers; resume from there rather than starting over.
- Default to proceeding. The interactive plan-confirmation step from implement.md does not apply: state your plan in your output, make the conservative call on tactical choices (which package, whether to run the full suite), and record it — the PR review is where a human redirects you, cheaply.
- Ask only when the gap cannot be filled conservatively: undefined business rules, contradictory requirements, or choices that are costly to reverse. Put the question(s) at the end of your output — it lands on the issue as your comment — and stop **before fetching any org**, so no sandbox sits behind an open question.
- Derive the assignment-id from the issue (for example `issue-554`). It stays stable across turns, so a task resumed after a question reuses the same sandbox instead of taking a fresh one.

## Proven command shapes

```bash
codev pool fetch --repository "$SFP_REPOSITORY" --tag "$POOL_TAG" \
  --alias dev-org --assignment-id "issue-<number>" --json

codev push --package <pkg> --targetusername dev-org --json

codev apextests trigger --targetusername dev-org \
  --testlevel RunSpecifiedTests --specifiedtests <TestClass> --wait 30 --json
# (RunAllTestsInPackage --package <pkg> for the package-wide signal)

codev pool unassign --repository "$SFP_REPOSITORY" --tag "$POOL_TAG" \
  --assignment-id "issue-<number>" --json
```

With application-token auth, pool assignments are recorded under the token owner's identity as `app:<email>` — expect that form when listing or filtering assignments.

## Creating the pull request

GitHub Enterprise policy commonly forbids the Actions installation token from creating PRs (`GraphQL: GitHub Actions is not permitted to create or approve pull requests`). Two facts matter:

1. The `GH_TOKEN`/`GITHUB_TOKEN` visible in your session is the installation token — the action sets these itself, regardless of what the workflow configured. Branch pushes work with it; `gh pr create` does not.
2. The workflow provides a personal access token under `AGENT_GH_PAT` for exactly this step. Use it explicitly:

```bash
PR_URL=$(GH_TOKEN="$AGENT_GH_PAT" gh pr create --base main --title "..." --body-file /tmp/pr-body.md)
PR_NUMBER=$(GH_TOKEN="$AGENT_GH_PAT" gh pr view "$PR_URL" --json number --jq .number)
codev agent link-pr --number "$PR_NUMBER" --repository "$SFP_REPOSITORY"
```

Link the PR immediately after creation. The server verifies that its repository, branch, and head SHA match the checkout before recording it. The workflow also retries this link after the agent step, so a transient telemetry failure does not invalidate otherwise successful development work.

If `AGENT_GH_PAT` is absent and PR creation is refused: push the branch, put the full evidence in your final output together with the compare URL (`https://github.com/<repo>/compare/main...<branch>?expand=1`), and state exactly why the PR could not be opened. Never claim a PR exists when it does not.
