---
name: diff-packages
description: How diff packages (packages marked as type diff in sfdx-project.json) behave in a Flxbl project, and how to change them safely. Covers registering a package, what goes into the artifact, baselines across environments, deleting components, reverting or restoring changes when environments are at different commits, and diagnosing a component that is missing from an artifact or a deploy that fails on it. Use before reverting, deleting or restoring metadata in a diff package, when an environment is ahead of or behind another, or when asked why a change did not reach an org. Not for source, unlocked or data packages, which always build in full.
---

# diff-packages

A diff package contains only what changed between a **baseline commit** and the build commit. That makes it behave differently from every other package type, in ways that are easy to get wrong. Read the matching reference before acting:

| Task | Reference |
|---|---|
| Add or register a diff package, or check whether a change will be included | [references/setup.md](references/setup.md) |
| Remove a component from orgs | [references/deleting-components.md](references/deleting-components.md) |
| Revert a change, restore something deleted, or undo a merged PR | [references/reverting.md](references/reverting.md) |
| A component is missing from an artifact or an org, a deploy fails, or tests run unexpectedly | [references/diagnosing.md](references/diagnosing.md) |

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

## CLI

Use the conventions in the [codev skill](../codev/SKILL.md) if it's installed: `codev` where available, otherwise `sfp`, with the same commands and flags. Run `<cli> <command> --help` rather than guessing a flag.

The `codev` CLI has `build`, `install`, `package create diff` and `package diff`. `sync org` and `project preflight` exist only in standalone `sfp`, or as operations in the codev app.

## How far this has been verified

Items marked **Verified** were run on 2026-11-04 with sfp 52.29.2 against a Salesforce scratch org, using a stand-in for the sfp server that held the baselines. The Sync Org to Branch dry runs used an artifact record inserted by hand, with the same fields a successful deploy writes. The rest comes from the product's source and documentation. Newer sfp versions may behave differently, so check the output of a dry-run build rather than relying on this alone.

Product documentation: [Diff Package](https://docs.flxbl.io/sfp/concepts/supported-package-types/diff-package), [Destructive Changes](https://docs.flxbl.io/sfp/concepts/destructive-changes).
