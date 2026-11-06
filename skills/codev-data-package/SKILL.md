---
name: codev-data-package
description: How to work with data packages (packages marked as type data in sfdx-project.json) in a Flxbl project, loaded by sfp through sfdmu. A data package is built into an immutable artifact that loads the same records into every environment in turn, including production, so every operation must be safe to run again. Covers when to use one, export.json and per-record files, test data for review orgs, external IDs, environment-specific data, what an install does and how it fails, the dev loop, and what to leave to a person. Use before creating or changing a data package, writing an export.json, loading reference or configuration data, or when a data package install fails.
---

# codev-data-package

A data package carries records, not metadata: CSV files, or one file per record, plus an `export.json` that tells sfdmu how to load them. sfp loads it with sfdmu, which it bundles, in its turn with the other packages.

| Task | Reference |
|---|---|
| Create or change a data package: export.json, external IDs, per-record files, data that differs by environment | [references/authoring.md](references/authoring.md) |
| Per-record YAML files: the format, editing records, and what not to do | [references/per-record-files.md](references/per-record-files.md) |
| Test data for testers and review orgs, kept out of production | [references/test-data.md](references/test-data.md) |
| What an install does, how it fails, and fixing a failed load | [references/deploying.md](references/deploying.md) |

## You are not running a data load

A one-off data load says: "put these rows into this org, now". A data package works differently:

1. **The repository is the source of truth** for these records.
2. **A build makes one artifact**, which holds the whole package, not just the rows you changed.
3. **The same artifact loads into every environment in turn**, including production, and again on a retry or a redeploy.
4. **Each environment already has data of its own**, and production has the most.

So every operation in `export.json` must be safe to run again, in any environment, against records you can't see.

## What to keep in mind

- **Upsert on an external ID.** It's the only operation that gives the same result on every run. An `Insert` creates duplicates on every redeploy.
- **Upsert overwrites.** A matched record gets the package's values, replacing edits made in that environment. Only put records in a data package that the repository should own.
- **A load isn't transactional.** If some rows fail, the install fails, but the rows that loaded stay loaded. There's no rollback.
- **Deletes are unguarded.** A `Delete` operation or `deleteOldData` removes records in every environment, production included, with no prompt and no check.
- **No check-only run.** Data packages can't be validated without loading. PR validation loads them into the review environment for real.
- **No tests run** for data packages.
- **Order matters.** Packages deploy in `sfdx-project.json` order. The objects, fields and record types the data needs must come from a package earlier in the list. sfp doesn't check this.

## When to use a data package

| Use a data package for | Don't use one for |
|---|---|
| Reference and configuration records the repository should own: lookup tables, product catalogue entries, settings records | Business records users create and edit |
| Records that must be identical in every environment, or differ in a controlled way per environment | Test records mixed into a reference-data package |
| **Test data** for testers and review orgs, in its own package that can't reach production. See [references/test-data.md](references/test-data.md) | Custom metadata or custom settings that metadata can deploy: put those in a source package |
| | A one-off data fix: a deployment step, or a script that checks before acting |

## Ask a person first

- **Adding a `Delete` operation or `deleteOldData`.** It deletes records in production.
- **Adding records that users edit**, because every deploy overwrites their changes.
- **Changing an `externalId`, or a record's external ID value.** Existing records stop matching, so the next deploy creates new ones and leaves the old ones in place. With per-record files, renaming a record file does this.
- **Retrying a failed load in production**, after checking what was already written.

## CLI

Use `codev` where it's installed, otherwise `sfp`, with the same commands and flags. Run `$CLI <command> --help` rather than guessing a flag. Always pass the target org with `-o`. `--json` writes JSON to stdout and logs to stderr, so never add `2>&1` when parsing it. Never print tokens or auth URLs.

| Command | What it does with a data package |
|---|---|
| `push -p <package> -o <org>` | Loads the package's records into the org through sfdmu. Builds nothing |
| `pull -p <package> -o <org>` | Exports the org's records into the package's files. `--preview` shows what would change |
| `build -p <package> --branch <b> -v <devhub>` | Writes a versioned artifact to `artifacts/`. `-v` is required even though no scratch org is used |
| `install -o <org> --artifactdir artifacts` | Loads the artifact and records the installed version in the org |
| `package data convert` | Converts between CSV and per-record files, offline |
| `package create data` | Registers the package with the server. It builds nothing |

`sfp sfdmu run` exists in standalone `sfp` only. Deployment scripts must call it as `sfp sfdmu run`, not `sf sfdmu run`.
