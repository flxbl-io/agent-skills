# Setting up and previewing a diff package

## Declare and register

Set `"type": "diff"` on the entry in `sfdx-project.json`. Source package attributes also apply.

```json
{ "path": "unpackaged", "package": "unpackaged", "type": "diff" }
```

To create the package and register it with the server:

```bash
<cli> package create diff -n <name> -r <path> --repository <owner/repo>
```

If the entry already exists, a person can run `sfp project preflight --branch <branch>`, or a real build registers it. Both record the last commit that touched the package path as the **default baseline**, and neither overwrites an existing one.

## Where a build starts

For each package, on the build's branch:

| Situation | Build starts from |
|---|---|
| Active environments have recorded baselines | The one furthest behind |
| An active environment has no baseline | That environment is skipped |
| No active environment has a baseline | The default baseline |
| No recorded baseline is in the checkout | The default baseline |
| The branch isn't tracked | The merge-base with its nearest tracked parent |
| Start and build commit are the same | Nothing: the package builds empty |
| No baseline at all | The build fails |

Inactive environments never count. A shallow clone can leave baselines outside the checkout, so fetch full history before trusting a preview.

## What's in the artifact

- Files under the package `path` that differ between the start and the build commit.
- Companion files that must deploy with them, such as the parent object of a changed MasterDetail field, or the holder of a decomposed permission set part. Only files inside this package's `path` are added. For a companion in another package, the build logs a warning and copies nothing.
- The whole of `pre-destructive/` and `post-destructive/`, if they exist. **Verified:** they ship even when the build's range doesn't touch them.

## Preview before merging

```bash
codev build                                                      # current workspace, including uncommitted changes
codev build --no-local                                           # committed state
sfp build --dry-run --branch <branch> -p <package> --repository <owner/repo>
```

These make no durable changes. Look for this line, and open the zip in `artifacts/` to see the payload:

```text
<package>: using most-behind commit from 2 environment(s) → <start>
```

To see what the build should contain from that start:

```bash
git diff --name-status <start> HEAD -- <package-path>
```
