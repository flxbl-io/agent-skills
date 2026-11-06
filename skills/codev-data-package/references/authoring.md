# Authoring a data package

## Layout

```text
data/reference-data/        ← the package's path
  export.json
  Product2.csv
  Pricebook2.csv
```

```json
{ "path": "data/reference-data", "package": "reference-data", "type": "data", "versionNumber": "1.0.0.NEXT" }
```

A data package needs `export.json` and at least one CSV, or per-record files. Without them, it counts as empty. A data package with a `VlocityComponents.yaml` instead is a Vlocity package, which this skill doesn't cover.

## export.json

One entry per object, in the order sfdmu should load them, parents before children:

```json
{
  "objects": [
    {
      "query": "SELECT Name, IsActive FROM Pricebook2 WHERE IsStandard = false",
      "operation": "Upsert",
      "externalId": "Name"
    },
    {
      "query": "SELECT Name, ProductCode, Family, IsActive FROM Product2",
      "operation": "Upsert",
      "externalId": "ProductCode"
    }
  ]
}
```

- **`operation`:** use `Upsert`. `Insert` duplicates records on every deploy. `Delete` and `deleteOldData` remove records in every environment: ask a person first.
- **`externalId`:** a field whose value is the same in every environment, such as a code or unique name. Never a Salesforce record ID: IDs differ between orgs. A composite key is allowed, for example `Name;Family__c`.
- **Lookups:** reference the parent by its external ID, with a `Parent__r.ExternalField` column, and include the parent object in `export.json`, earlier in the list. A lookup sfdmu can't resolve fails with `Required field missing`.
- **Record types:** reference them by developer name, not by ID.
- **Only fields the repository should own.** Every field in the query overwrites what's in the target org.

## Per-record files

With `"dataFormat": "perRecord"`, the package stores one YAML file per record instead of CSVs, in beta. See [per-record-files.md](per-record-files.md) for the format and how to edit records safely.

## Data that differs by environment

Use an aliasfied package: one folder per org alias, each with its own `export.json` and files.

```text
data/env-data/
  default/        ← used by sandboxes with no folder of their own
  uat/
  prod/
```

- **Production needs an exact match.** If the production org's alias has no folder, the install fails. `default/` isn't used for production unless the package sets `aliasfy.mergeMode`.
- **`push` doesn't choose a folder:** it reads the package's top folder. Test an alias folder with `build` and `install` against a sandbox.

To keep a package out of some orgs entirely, use `skipDeployOnOrgs`.

## Before you hand back a change

1. Is every operation an `Upsert` on an external ID that's stable across orgs?
2. What happens to records that already exist in production with the same external ID?
3. Does anything the data needs (objects, fields, record types, parent records) come earlier in `sfdx-project.json`?
4. Did you load it twice into a sandbox, and did the second run change nothing?
