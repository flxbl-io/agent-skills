# The artifact model, for diff packages

## You are not building a change set

A change set says: "make this org look like this, now". A Flxbl project works differently, and most failed diff releases come from forgetting that:

1. **The repository is the source of truth.** Every environment exists to match it.
2. **A build makes one artifact.** It's versioned and doesn't change after it's built.
3. **That same artifact goes to every environment in turn**, such as review, test, UAT and production. That can take days, and each environment receives it in whatever state that environment is in.
4. **Environments converge on the repository.** They don't get individual fixes.

You're not writing a change for the org in front of you. You're writing something every environment will receive, in a state you can't see.

## What a diff artifact contains

Not just your change: **everything under the package path that differs between the build start and the build commit.**

- The build start is the baseline of the active environment furthest behind on the branch. So the artifact can carry other people's changes, from merges that haven't reached that environment yet.
- A diff compares two end states, not history. Something changed and then changed back isn't in it.
- An environment ahead of the build start receives this same artifact. It already has some of the changes, and may still hold intermediate versions the artifact doesn't correct. See [reverting.md](reverting.md).

## What the artifact will meet

Before finishing any change in a diff package, check that it works in every one of these:

| Environment state | Example |
|---|---|
| At the build start | The environment furthest behind, usually production |
| Ahead of the build start | UAT, which already has some of the changes in the artifact |
| Already has this version | A retried release, or a redeploy |
| Has a previous attempt half applied | A post-deployment script failed after the metadata deployed |
| Holds data the others don't | Production records, paused flow interviews, users and settings |
| Differs from git, after someone changed it by hand | Drift |

If the change only works in some of these, it isn't ready.

## What this means in practice

- **Scripts must be idempotent.** For a diff package they run on almost every release, in every environment, and again on retry. See [deployment-scripts.md](deployment-scripts.md).
- **Data changes must be idempotent.** Upsert on an external ID, never a blind insert or delete.
- **Each environment needs a guard.** Ask the org about its own state rather than assuming what the previous environment had.
- **Configure environment differences, don't branch on them in code.** Use aliasfied packages for metadata that differs by environment, string replacements for values, and `skipDeployOnOrgs` for a package some orgs shouldn't get.
- **Deletions persist.** Components in `pre-destructive/` or `post-destructive/` ship in every artifact, and are deleted on every deploy, until the folder is removed. See [deleting-components.md](deleting-components.md).
- **Undo means a new change.** You can't take an artifact back out of an environment, and an exact revert may not reach environments that are ahead. See [reverting.md](reverting.md).
- **Fix forward, through the repository.** Don't fix an org by hand and leave git behind: that's drift, and the next artifact can undo or break it. If an environment needs repairing, change the repository so the next artifact repairs it, or record a manual step that a person tracks.
- **Mind dependencies.** Packages deploy in the order of `sfdx-project.json`. Companion files from another package aren't copied into this one, so a change that needs another package's change must deploy there first.

## Before you hand back a change

Answer these in your summary or PR:

1. What does this change do in the environment at the build start, and in one ahead of it?
2. What does it do if it's deployed twice?
3. What does it do if it fails halfway, and is a retry safe?
4. What does it assume about data or settings, and how does it check them rather than assume?
5. Does anything need to happen outside the artifact? If so, who does it, and where is it tracked?
