# Deleting components

Removing a file from `main/default` removes nothing from any org. **Verified:** an object deleted only in git stayed in the org after the deploy reported success.

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

## Flows are different

For both folders, flows are deactivated and all their versions deleted **before** the metadata deploy, and this is not part of the transaction. If the later deploy fails, the flow stays deleted. Anything that blocks the deletion must already be gone from the org. See the product documentation before relying on this.

## After the delete has reached every environment

Merge a follow-up change that removes the folder. Until then it ships in every build (**Verified**), so the delete runs again on each deploy.

Do this before anyone restores the component, or a build could delete it again.

## If the delete fails in some environments

Stop and report this. The environments have now diverged: those where the delete succeeded are ahead, and the others hold the build start back. Don't restore the component in git as a quick fix, because that's the failing case in [reverting.md](reverting.md). A person should decide whether to fix the failing environments and retry, or restore the component on the environment that deleted it.

A Salesforce commit-step error with no component or test failures means the org rejected the change as a whole, so read the org's deployment status page too.
