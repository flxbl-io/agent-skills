# Deleting components

Removing a file from `main/default` removes nothing from any org. The deploy succeeds and the component stays.

## How to delete

Move the component's metadata file into a folder beside `main/default`, keeping the type folder:

```text
<package-path>/
  main/default/...
  pre-destructive/objects/Child__c/Child__c.object-meta.xml
  post-destructive/objects/Master__c/Master__c.object-meta.xml
```

| Folder | Runs | Use for |
|---|---|---|
| `pre-destructive/` | Before the deploy, in the same metadata transaction | Components that block the deploy, and the child side of a relationship |
| `post-destructive/` | After the deploy, in the same metadata transaction | Components still referenced until this deploy removes the reference, and the master side |

Remove references from other metadata in the same change, such as permission sets, layouts and Apex. Check other packages too, since this package's diff won't show them.

### Example: removing an LWC used on a Lightning page

The change deletes an LWC, removes it from the FlexiPage that shows it, and removes the `@AuraEnabled` Apex method it calls:

| Where the LWC goes | Result |
|---|---|
| `post-destructive/` | **One deployment succeeded.** The Apex and FlexiPage deployed, then the LWC was deleted |
| `pre-destructive/` | Failed: `Referenced by a component instance inside the Lightning page`. The delete runs before the page is updated |
| Deleted in git only | The deploy succeeded, but the LWC stayed in the org |

Put a component in `post-destructive/` when the same change removes what references it. Salesforce also allowed the Apex method to be removed while the LWC that imports it still existed, so that order didn't matter.

## Flows are different

For both folders, flows are handled **before** the metadata deploy, and this isn't part of the transaction:

1. sfp checks whether any flow interview, whether paused, failed or running, still uses any version of the flow.
2. If none does, it deactivates the flow and deletes every version. If the later deploy fails, the flow stays deleted.
3. If one does, the flow is **skipped**: it stays active, with all its versions, and nothing about it changes.

**A skipped flow doesn't fail the install.** The deploy reports success, and the only signs are this row in the destructive flow table and a warning that lists the blocked versions:

```text
Skipped: in use (<n> interviews)
```

Read that table in every environment's deploy log. Don't assume the flow is gone.

**The install never deletes interviews.** Deleting one drops the work it was waiting to do, so a person decides. They can clear them under Setup › Paused and Failed Flow Interviews, or run:

```bash
sfp org flow cleanup -f <flow> -o <org> --dry-run             # list what blocks it
sfp org flow cleanup -f <flow> -o <org> --delete-interviews   # delete paused and failed interviews, then old versions
```

That command is in standalone `sfp` only. Running interviews are never deleted, and the active version is always kept. While the flow remains in the destructive folder, each later deploy tries to delete it again.

Anything else that blocks the deletion must already be gone from the org. Removing it in the same deploy doesn't help, because the flow step runs first.

### Old flow versions can block deleting a field

Salesforce won't delete a field while any flow version references it, including inactive ones. To delete those versions first, set `flowVersionCleanup` on the package, which deletes old versions of the package's own flows before each deploy:

```json
"flowVersionCleanup": { "keepVersions": 5, "deleteInterviews": false }
```

A person can also do it once, with `sfp org flow cleanup -f <flow> -o <org> --keep <n>`. Versions that interviews still use are kept, with a `FLOW_VERSION_IN_USE` warning, unless `deleteInterviews` is `true`. Setting it to `true` is a person's decision, for the same reason as above.

Flow interview handling needs server 53.30.0 and CLI 52.40.0 or later. Older versions deactivate a flow before checking, and can leave it partly deleted.

## After the delete has reached every environment

First confirm it really reached each environment. For a flow, that means no `Skipped` row in the log. Then merge a follow-up change that removes the folder. Until then it ships in every build, so the delete runs again on each deploy.

Do this before anyone restores the component, or a build could delete it again.

## If the delete fails in some environments

Stop and report this. The environments have now diverged: those where the delete succeeded are ahead, and the others hold the build start back. Don't restore the component in git as a quick fix, because that's the failing case in [reverting.md](reverting.md). A person should decide whether to fix the failing environments and retry, or restore the component on the environment that deleted it.

A Salesforce commit-step error with no component or test failures means the org rejected the change as a whole, so read the org's deployment status page too.
