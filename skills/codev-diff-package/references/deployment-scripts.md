# Deployment scripts in a diff package

A script in a diff package ships inside the artifact and runs on almost every release, because the package deploys whenever it has changes. Write it for every state it will meet, as in [artifact-model.md](artifact-model.md), not for the org in front of you.

## First: does this need a script?

Use a built-in setting when one exists. It's tested, visible in the project file and skipped correctly in check-only runs.

| You want to | Use instead of a script |
|---|---|
| Assign permission sets before or after deploying | `assignPermSetsPreDeployment` or `assignPermSetsPostDeployment` on the package |
| Load or update reference and configuration records | A data package |
| Delete metadata | `pre-destructive/` or `post-destructive/` folders |
| Activate flows, or delete old flow versions | `enableFlowActivation`, or `flowVersionCleanup` |
| Choose which tests run | Test suites bound in `config/sfp_testsuites.yaml` |
| Have a person do something once, such as a Setup change or approving a data fix | A deployment step in codev, which can hold the release until it's done. Without codev, a step in your pipeline or release notes |

Write a script only when none of these covers it: for example a setting that metadata can't deploy, or data that has to change in step with this deploy.

## How scripts run

- **Declaring one:** add the path, relative to the project root, on the package entry in `sfdx-project.json`:
  ```json
  { "path": "src/core", "package": "core", "preDeploymentScript": "scripts/core-pre.sh", "postDeploymentScript": "scripts/core-post.sh" }
  ```
- **What ships:** the single script file is copied into the artifact at build time, from the build commit. Nothing else from the repository comes with it.
- **Order:** pre-deployment permission sets → **pre script** → deploy → post-deployment permission sets → **post script** → the install is recorded.
- **Arguments:** `$1` package name, `$2` target org username, `$3` org alias (the username if there's none), `$4` artifact working directory, `$5` package directory within it.
- **Shell:** `bash -e` on Linux and macOS, so the first failing command stops the script. On Windows, sfp runs it with `cmd.exe`, where a bash script won't work, so a project that releases from Windows needs a batch script.
- **Environment:** the script inherits the environment of whatever runs sfp, with colour turned off so `sf ... --json | jq` parses cleanly.
  - In codev, sfp runs on codev's workers, which have `bash`, `sf`, `jq`, `curl` and `git`.
  - In your own pipeline, it's that runner's tools, so the sfp Docker image or the CI machine. Depend only on `bash`, `sf` and `jq`, and check with `command -v` if the script needs anything else.
- **Timeout:** 30 minutes.

## What a failure does

| Script | Effect of a non-zero exit |
|---|---|
| Pre | The package isn't deployed, and the install fails. Permission sets assigned before the script stay assigned |
| Post | The metadata is **already deployed**, but the install is marked failed and not recorded. A retry deploys the package again and runs both scripts again |

## Where scripts run

Every sfp command that installs the package runs its scripts, against that command's target org:

| Command | Runs scripts? |
|---|---|
| `sfp release`, `sfp install` | Yes |
| `sfp validate`, and codev change validation | Yes, on the review org, except for packages that validate check-only (`checkOnlyAgainst`) |
| `sfp sync org` | Yes, except `--diff-deploy`, which deploys a synthetic package without them |
| `sfp prepare`, which fills scratch-org pools | Yes, on every pool org |
| `sfp release --checkonly` | No |

A pool's own `postDeploymentScriptPath` in its pool definition is a separate script: it runs once per pool org, not per package.

## When scripts don't run

- **Check-only release:** neither script runs. A check-only pass doesn't prove the scripts work.
- **Quick deploy:** the pre script is skipped, and the post script runs.
- **Already installed:** when the same version is already installed and installs skip installed versions, nothing runs.



## Rules

1. **It must be safe to run more than once.** It runs on every deploy of the package, in every environment, and again after a failed install is retried. Check the state first, then act only if needed. For records, upsert on an external ID rather than insert.
2. **It must stand alone.** Only the script file reaches the artifact. Fetch anything else it needs, or inline it. Don't read files from the repository by relative path.
3. **Target the org explicitly.** Pass `-o "$2"` to every `sf` command. The runner's default org may be a different org.
4. **Expect production to differ.** Production has data, paused flow interviews and settings that sandboxes don't. Handle the states you can foresee with a clear message, and fail on anything you can't.
5. **Fail loudly and specifically.** Exit non-zero with a message that says what was wrong and how to fix it. Don't swallow errors to get a green deploy.
6. **Keep it short and fast.** It has 30 minutes, and every deploy waits for it.
7. **Never print secrets.** Read credentials from environment variables, and don't echo tokens, auth URLs or session IDs.
8. **Retire one-off scripts.** A script that has done its job in every environment should be removed from the package entry, not left to run forever.

## Template

```bash
#!/bin/bash
# Runs as: <script> <package> <username> <alias> <workdir> <packageDir>
set -euo pipefail

PACKAGE="$1"
TARGET="$2"   # use the username: the alias may be absent
log() { echo "[$PACKAGE] $*"; }

query() { sf data query -o "$TARGET" -q "$1" --json | jq -r "$2"; }

# 1. Decide whether this org needs the change at all.
already=$(query "SELECT COUNT() FROM My_Setting__c WHERE Name = 'Feature_X'" '.result.totalSize')
if [ "$already" -gt 0 ]; then
  log "Feature_X already configured, nothing to do"
  exit 0
fi

# 2. Make the change.
sf data create record -o "$TARGET" -s My_Setting__c -v "Name=Feature_X Enabled__c=true" --json >/dev/null

# 3. Confirm it took effect.
now=$(query "SELECT COUNT() FROM My_Setting__c WHERE Name = 'Feature_X'" '.result.totalSize')
[ "$now" -gt 0 ] || { log "Feature_X was not created"; exit 1; }
log "Feature_X configured"
```

To behave differently by environment, ask the org rather than relying on the alias:

```bash
is_sandbox=$(query "SELECT IsSandbox FROM Organization" '.result.records[0].IsSandbox')
```

## Test it before it reaches a release

1. **Run it directly against a sandbox or scratch org**, with the same arguments the installer passes:
   ```bash
   bash -e scripts/core-post.sh core <username> <alias> "$PWD" src/core
   ```
2. **Run it a second time.** It should report that there's nothing to do and exit 0.
3. **Run it against an org in the awkward state:** missing data, the change already half-applied, the edge case production is likely to have.
4. **Deploy the package to a review or sandbox org** with a real deployment, which runs the scripts: codev change validation, `sfp validate` or `sfp install`. A check-only release won't run them.
5. **If the project releases from more than one place**, such as codev and a CI pipeline, run it on each kind of runner, because their installed tools differ.

In your PR, say what the script does in each environment, how it behaves on a second run, and how you tested it.

## When a release fails inside a script

- **Pre script:** the package wasn't deployed. Fix the script or the org state, then retry.
- **Post script:** the metadata is already in the org. A retry deploys the package and runs both scripts again, so check that is safe before retrying.

Read the script's own output in the deploy log, under `Executing preDeployment script` or `Executing postDeployment script`.
