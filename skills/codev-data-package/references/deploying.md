# Deploying a data package

## What an install does

1. Pre-deployment permission sets, then the pre-deployment script.
2. sfdmu loads the records from the artifact into the org, following `export.json`.
3. sfp reads sfdmu's results and **fails the install if any row failed.**
4. Post-deployment permission sets and script, then the installed version is recorded in the org.

The artifact is the whole package, every time. A data package is rebuilt when its files change, and then every record in it loads again.

| Behaviour | Data packages |
|---|---|
| Diff or partial payload | No: always the whole package |
| Component checksum skip | Doesn't apply |
| Check-only release and quick deploy | Not supported |
| Tests | None run |
| PR validation | Loads the records into the review environment |
| Skip if already installed | Yes, when installs skip installed versions, unless `alwaysDeploy` is set |

## When a load fails

sfdmu writes the rows it loaded before sfp checks the results. **A failed install can leave some records loaded** and others not.

1. Read the failed rows in the deploy log. sfdmu's `target/*_target.csv` reports hold the error for each row.
2. Fix the cause in the repository, then deploy again. With `Upsert` on a stable external ID, a second run completes the load without duplicating what's there.
3. With `Insert`, a retry duplicates every row that loaded the first time. Clean up first, and switch the operation to `Upsert`.

| Error | Likely cause |
|---|---|
| `has no mandatory external Id field definition` | `externalId` missing for that object |
| `Required field missing: [Field__c]` | A lookup the load couldn't resolve: add the parent object to `export.json`, or reference it by external ID |
| A field or object not found | The metadata package that adds it deploys later in `sfdx-project.json`, or isn't deployed to that org |
| `does not have a '<alias>' directory` | An aliasfied package with no folder for this org. Production needs an exact match |

## Dev loop

- **`push` and `pull`** move records between the package's files and an org through sfdmu. They don't build an artifact or record a version.
- **Before raising a PR,** push twice to a sandbox. The second run should change nothing. If it does, the operations aren't idempotent.
- **`pull` overwrites** the package's files with the org's records. Review the diff before committing, so test records from your sandbox don't end up in the package.
