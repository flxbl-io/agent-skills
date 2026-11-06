---
name: codev-diff-package
description: How diff packages (packages marked as type diff in sfdx-project.json) behave in a Flxbl project, and how to change them safely. Covers registering a package, what goes into the artifact, baselines across environments, deleting components, reverting or restoring changes when environments are at different commits, test selection including codev-suggested test lists, and diagnosing a component that is missing from an artifact or a deploy that fails on it. Use before reverting, deleting or restoring metadata in a diff package, when an environment is ahead of or behind another, when a PR is waiting on a test decision, or when asked why a change did not reach an org. Not for source, unlocked or data packages, which always build in full.
---

# codev-diff-package

A diff package contains only what changed between a **baseline commit** and the build commit. That makes it behave differently from every other package type, in ways that are easy to get wrong. Read the matching reference before acting:

| Task | Reference |
|---|---|
| Add or register a diff package, or check whether a change will be included | [references/setup.md](references/setup.md) |
| Remove a component from orgs | [references/deleting-components.md](references/deleting-components.md) |
| Revert a change, restore something deleted, or undo a merged PR | [references/reverting.md](references/reverting.md) |
| Which tests run, codev-suggested test lists, or a PR waiting on a test decision | [references/test-selection.md](references/test-selection.md) |
| A component is missing from an artifact or an org, or a deploy fails | [references/diagnosing.md](references/diagnosing.md) |

Check `"type": "diff"` on the package's entry in `sfdx-project.json` first. If it isn't there, none of this applies.

## What to keep in mind

1. **A diff compares two end states, not history.** A change that was made and then undone shows no difference, so it isn't in the artifact, even if an environment still holds the intermediate state.
2. **One artifact goes to every environment.** The build starts from the baseline of the active environment furthest behind on the branch. An environment that is ahead receives the same artifact, which doesn't always carry what that environment needs.
3. **A git delete removes nothing from an org.** Removal goes through the `pre-destructive/` and `post-destructive/` folders.
4. **A successful deploy can hide a gap.** The environment's baseline and artifact record move to the build commit even when a file in that environment was not updated. Nothing flags it afterwards.
5. **There is no rollback.** Diff packages move forward with a new change.
6. **Sandboxes only.** Diff packages don't work with scratch orgs.

## Ask a person first

These change what later builds contain, or change an org outside the normal pipeline. Propose them and let a person decide:

- **Editing a baseline.** Later builds won't deploy the changes it skips. Only an `owner` can do this, in the codev web app.
- **Running Sync Org to Branch** on a shared environment.
- **Deactivating an environment** to change where builds start.
- **Accepting or rejecting a test list** with `/tests`. It sets what guards production.

## CLI

- **Which binary.** Use `codev` where it's installed, otherwise `sfp`. Commands and flags are the same:
  ```bash
  CLI=codev; command -v codev >/dev/null 2>&1 || CLI=sfp
  ```
  Run `$CLI <command> --help` rather than guessing a flag. `sf` and `sfdx` aren't the tools for these workflows.
- **Auth.** In CI and containers, `SFP_SERVER_TOKEN` and `SFP_SERVER_URL` are already set, so don't ask for credentials. On a developer machine, `codev` uses the signed-in codev app session. Exit code 78 means the user must sign in to the app.
- **Repository.** Use `SFP_REPOSITORY` if it's set, otherwise derive it from `origin`:
  ```bash
  REPO=${SFP_REPOSITORY:-$(git remote get-url origin | sed -E 's|\.git$||; s|^.*[:/]([^/]+/[^/]+)$|\1|')}
  ```
- **`--json` output.** JSON goes to stdout and logs to stderr. Never add `2>&1` when parsing it.
- **Secrets.** Never print tokens, `sfdxAuthUrl` values or frontdoor URLs, and never write them to files.

The `codev` CLI has `build`, `install`, `package create diff` and `package diff`. `sync org` and `project preflight` exist only in standalone `sfp`, or as operations in the codev app.

## Further reading

Product documentation: [Diff Package](https://docs.flxbl.io/sfp/concepts/supported-package-types/diff-package), [Destructive Changes](https://docs.flxbl.io/sfp/concepts/destructive-changes).
