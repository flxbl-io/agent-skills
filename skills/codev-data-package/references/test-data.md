# Test data packages

A data package can carry test data, for example accounts, contacts and cases for testers and review orgs, as long as it is **marked as test data** and **can't reach production**.

## Mark it

- **Its own package.** Never mix test records into a reference-data package. Name it so nobody can mistake it, for example `test-data-cases` under `data/test-data/`.
- **Fake values only.** No real customer or personal data.
- **Recognisable external IDs**, such as a `TEST-` prefix, so test records are easy to find and clean up.
- **Upsert, not Insert or Delete.** Review orgs receive it again and again. Don't use `deleteOldData`: if the package ever reached the wrong org, it would delete real records.

## Keep it out of production

Use both guards:

```json
{
  "path": "data/test-data/cases",
  "package": "test-data-cases",
  "type": "data",
  "versionNumber": "1.0.0.NEXT",
  "ignoreOnStage": ["deploy"],
  "skipDeployOnOrgs": ["prod", "admin@acme.com"],
  "alwaysSync": true
}
```

**`ignoreOnStage: ["deploy"]`** skips the package in every command that runs as the `deploy` stage, and keeps it in the others:

| Command | Stage | Loads the test data? |
|---|---|---|
| `release`, `install` | `deploy` | No |
| `sync org`, including `/codev sync-review-env` and sandbox pool syncs | `deploy` | No |
| PR validation, `validate` | `validate` | Yes |
| `prepare`, which fills scratch org pools | `prepare` | Yes |

Use `"deploy"`, not `"release"`. The schema accepts `"release"`, but releases run as the `deploy` stage, so that value skips nothing.

**`skipDeployOnOrgs`** is a backstop. It skips the package when the target org given to the command exactly matches an entry. Depending on how the command is run, that's an alias or a username, so list both for production. It can only keep the package out of orgs you can name, so it can't aim it at pool orgs, whose names change.

If test data should also load into named test environments such as QA, leave out `ignoreOnStage` and rely on `skipDeployOnOrgs`, listing every org that must not get it, production first.

A release config can also leave the package out of a domain's releases, with `excludeArtifacts`.

## Getting test data into review orgs

| How | When it loads |
|---|---|
| **`alwaysSync: true`** | Validation normally loads only the packages a PR changed. With `alwaysSync`, the package is loaded whenever any package in its domain is changed. Put it in the domain whose PRs need the data |
| **Scratch org pools** | `prepare` installs the project's data packages into each pool org as it's created |
| **Sandbox pools** | A pool's sync runs as `deploy`, so `ignoreOnStage: ["deploy"]` skips it. Rely on `alwaysSync` in validation, or use `skipDeployOnOrgs` instead of `ignoreOnStage` |
| **The source sandbox** | Sandboxes cloned from a source that already holds the test data start with it |

`/codev sync-review-env` also runs as `deploy`, so it doesn't load a package with `ignoreOnStage: ["deploy"]`.

## Things that catch people out

- **Every deploy overwrites test records.** A tester's edits to a test record are replaced the next time the package loads.
- **Don't give test data a domain of its own.** If every package in a domain is skipped for the target org, the release of that domain fails. Keep the test-data package in a domain with packages that do deploy, or leave it out of the release config with `excludeArtifacts`.
- **Order matters.** Test data that refers to reference data or other test data must come later in `sfdx-project.json` than what it refers to.
- **A review org from the pool may hold a previous PR's test data.** Upserts on stable external IDs bring it back in line, but records the package no longer contains stay in the org.

## Before you hand back a change

1. Is the package clearly named and stored as test data, with fake values?
2. Does `ignoreOnStage` include `"deploy"`, or does `skipDeployOnOrgs` list every org that must not get it, production first?
3. Will it reach the review orgs that need it, through `alwaysSync`, pool preparation or the source sandbox?
4. Is every operation an upsert on a test-prefixed external ID?
