# Reverting or restoring a change

This is the most dangerous operation on a diff package. If the change has reached some environments but not the one furthest behind, an exact revert won't reach the environments that have it.

```text
                 A (production)     B (UAT)        HEAD
Modified file    v1            ──►  v2        ──►  v1 (revert)
Deleted object   exists        ──►  deleted   ──►  exists (restored)
```

The build starts at A. Neither file differs between A and HEAD, so neither is in the artifact.

| What was undone | Result in UAT (**Verified**) | Recovers by itself? |
|---|---|---|
| Modified file reverted | Deploy **succeeds**, UAT stays on v2, and its baseline moves to HEAD | **No, and nothing flags it** |
| Deleted component restored, while something that references it changed | Deploy **fails**: `no CustomObject named X found`. Baseline stays put | Yes, once the environment holding the build back deploys, because the next build then starts from UAT's baseline |
| New file reverted | The file stays in the org, as with any git delete | No |

## Before you revert

**1. Find the build start** from a dry-run build (see [setup.md](setup.md)).

**2. Get each active environment's baseline** from the codev app, or from the API:

```bash
curl -s -H "Authorization: Bearer $SFP_SERVER_TOKEN" \
  "$SFP_SERVER_URL/sfp/api/repository/package-baselines?repositoryIdentifier=<owner/repo>&branch=<branch>&packageName=<package>" \
  | jq '.value.environments | map_values(.baseline)'
```

Never print the token.

**3. With the revert on your branch, list what each environment needs and the artifact leaves out.** Run this for every environment ahead of the start:

```bash
comm -13 \
  <(git diff --name-only "$START" HEAD -- "$PKG_PATH" | sort) \
  <(git diff --name-only "$ENV_BASELINE" HEAD -- "$PKG_PATH" | sort)
```

- **Empty for every environment:** the revert is safe.
- **Any files listed:** those environments won't receive them. Continue below, and name the files and environments in your report.

## What to do when files are listed

**Modified file.** Make the content differ from the version at the build start, rather than restoring it exactly, so that the file is in the diff and reaches every environment. Prefer a real improvement. A comment works where the format allows it. **Verified** with an Apex class: the file was in the artifact. Say in the PR why the revert isn't exact.

**Restored component.** The same approach applies, but every file must differ, including each field of an object, so this rarely suits an object. Tell the person it will fail on that environment, and offer these options:

- Run Sync Org to Branch on that environment before or after merging. **Verified** in a dry run: it sees the changes, because the failed deploy never moved the org's artifact record.
- Wait until the environment holding the build back has deployed. The next build then carries the component.

**New file you want gone.** Move it to `post-destructive/`, as in [deleting-components.md](deleting-components.md).

## If the revert has already been deployed

Compare the org with git directly. A successful deploy tells you nothing here.

**Sync Org to Branch won't repair the silent case.** **Verified:** with the org on v2 and git on v1, `sfp sync org --from-org` reported `0 to sync, 1 up to date`, and `--sourcepackageoverride` didn't change that. It compares git commits starting from the org's artifact record, which the deploy already moved to HEAD. It doesn't read the org's metadata.

To repair it, change each affected file again so that it's in the next build. Don't edit baselines to force this without asking a person.

Report it plainly either way: which environment, which files, what the org holds and what git says.

## Tests

On a project using codev-suggested test lists, a revert PR that changes Apex needs its own test decision. If it's left undecided, or pushed without a PR, the package falls back to its yaml suite or `RunRelevantTests`. See [test-selection.md](test-selection.md).
