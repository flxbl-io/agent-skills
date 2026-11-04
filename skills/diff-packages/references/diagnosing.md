# Diagnosing a diff package

Start with the run's log. It names the build start, and `Source Version From` and `Source Version To` give the artifact's range.

## A change isn't in the artifact

| Check | Why |
|---|---|
| `git diff --name-status <start> <build-commit> -- <package-path>` | If the file isn't listed, the end states match. Was it reverted? See [reverting.md](reverting.md) |
| Is the file under this package's `path`? | Only this package's path is diffed |
| Is it a companion in another package? | Not copied. Look for a warning naming the owning package |
| Was it deleted in git? | Deletions need the destructive folders. See [deleting-components.md](deleting-components.md) |
| Was it pruned at deploy? | See "Component checksum skip" below |

## A change is in the artifact but not the org

- **A `Baseline check` warning at deploy:** the artifact starts after this environment's baseline, so the commits in between never reached it. The deploy continues, and the server holds the baseline at its old commit until a deployment starts at or before it. The next build on the branch starts from there while that environment is active.
- **Component checksum skip:** the component matched the previously installed artifact and was removed from the payload.

## A deploy fails on a reference to a missing component

For example, `In field: object - no CustomObject named X found`. The environment probably lacks something the artifact assumes it has:

1. Compare the artifact's start with the environment's baseline. If the baseline is ahead of the start, check whether the component changed between them.
2. Run the comparison in [reverting.md](reverting.md).
3. If it isn't that, look for the component in another package that deploys later.

Retrying won't help, because every build from the same start fails the same way.

## An org doesn't match git, but the deploy succeeded

Don't trust the baseline or the artifact record here, because both only show what was deployed. Retrieve the component or query the org, then compare. If it's a revert, see [reverting.md](reverting.md).

## Which tests run

See [test-selection.md](test-selection.md), including why a build ignored its codev-suggested list.

## Component checksum skip

On by default for diff packages. Components that match the installed previous artifact are removed from the payload. `Profile`, `PermissionSet`, `PermissionSetGroup` and `CustomLabels` are always kept. If everything would be skipped, the full set deploys.

If an org has drifted and a component needs redeploying, use `--no-checksum-skip` on `install`.

## Validation differs from release

- **Thorough validation** starts the diff from the review org's installed commit when that's older than the baseline, or when there's no baseline. Release never reads the org's commit, so a validation passing doesn't prove that a release to another environment will.
- Validation doesn't advance baselines. Nor does a check-only release.
- A review sandbox's baseline state can be `current`, `behind-head`, `behind-baseline` or `unknown`. `behind-baseline` means it's missing commits that no PR payload will carry.

## Sync Org to Branch

`sfp sync org --from-org` diffs from the org's installed artifact commit to the branch, using the same comparison of commits as a build. It therefore fixes environments whose **deploy failed**, but not those where a successful deploy left a gap ([reverting.md](reverting.md)). From the source, it doesn't advance the environment's baseline. Preview with `--dryrun`.
