# Per-record YAML files (beta)

A data package can store one YAML file per record instead of one CSV per object. Opt in per package with `"dataFormat": "perRecord"` on its entry in `sfdx-project.json`.

**Why:** two people editing different records edit different files, so they don't conflict. Reviews show only the records that changed. And because Salesforce IDs are left out, pulling the same data from any org gives identical files.

## Layout

```text
data/marketing-leads/
  export.json
  records/
    Lead/
      _schema.yaml                 ← column order, for rebuilding the CSV
      ada@acme.example.yaml        ← one record, named from its external ID
      grace@acme.example.yaml
    EmailTemplate/
      Welcome.yaml
      _content/
        Welcome.HTMLBody__c.html   ← long or HTML field values
```

## A record file

A flat list of fields:

```yaml
Email: ada@acme.example
Company: ACME Robotics
FirstName: Ada
LastName: Lovelace
LeadSource: Web
Title: Chief Technology Officer
```

- **What's in it:** every field in the `export.json` query, the external ID fields, and lookup references (`Parent__r.Name`).
- **What's left out:** the Salesforce `Id`, audit fields such as `CreatedDate`, empty values, and raw lookup ID columns that have a `__r.Name` reference. `excludeIdsFromCSVFiles: true` in `export.json` does this, and conversion sets it.
- **Long or HTML values:** fields over about 2 KB, or HTML, live in `_content/` and are referenced as `$file: _content/<key>.<field>.html`.

Build, install and `push` flatten the records back into CSVs, so sfdmu loads exactly the same input as before.

## Editing records

- **Change a value:** edit the YAML and commit. No org round-trip is needed.
- **Add a record:** add a file whose fields include the external ID. Leave the filename to `pull` or the converter where you can: they name it from the external ID.
- **Don't rename a file, or change its external ID value, to rename a record.** The external ID is the upsert key, so the next deploy creates a **new** record, and the old one stays in every environment, because nothing deletes it. Treat a key change like changing the `externalId`: ask a person first.
- **Don't delete a file to delete a record.** Removing the file stops the package loading it, but the record stays in every org. Deleting records needs a `Delete` operation or `deleteOldData`, which a person decides on.
- **Don't edit `_schema.yaml` by hand.** It records the column order. Columns follow the `export.json` query.

## Commands

```bash
$CLI package data convert -p <package> --to records   # CSV to per-record files
$CLI package data convert --all --to records          # every data package
$CLI package data convert -p <package> --to csv       # preview the CSV a build will produce
$CLI pull -o <org> -p <package>                       # refresh the files from an org, then git diff
$CLI push -o <org> -p <package>                       # load the records into an org
```

- **Conversion is offline.** It deletes the source CSVs unless you pass `--keep-csv`. A CSV left behind is overwritten on every build, so don't edit it.
- **`--keep-ids`** keeps Salesforce IDs in the records. Avoid it: IDs differ between orgs.
- **A new package:** add the descriptor with `"dataFormat": "perRecord"` and an `export.json`, then `pull` from an org to create the record files.
- **Opting out:** remove `"dataFormat"` from the descriptor.

## Errors

| Error | Cause |
|---|---|
| `Required field missing: [Foo__c]` | A `__r.Name` reference the load couldn't resolve. Add the parent object to `export.json`, or make sure a matching parent exists |
| `duplicate field selected` | The `externalId` field is also listed explicitly in the query. Remove the duplicate |
| `has no mandatory external Id field definition` | An object in `export.json` has no `externalId` |

If a row's external ID can't be read, the file is named from `Name`, and clashes get `__2`, `__3` suffixes. Names like that mean the `externalId` needs fixing.
