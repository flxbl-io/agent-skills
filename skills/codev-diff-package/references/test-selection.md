# Which tests a diff package runs

codev-suggested lists need server 53.26.0 and CLI 52.37.0 or later. The product documentation may still describe a suggestion as only advice. In these versions an accepted list does change the test level.

## Order of precedence

For a diff package with Apex in its payload, highest first:

1. An artifact override or explicit test level for this run.
2. `skipTesting`, which is never honoured on a production release.
3. `isOptimizedDeployment: false` on the package: `RunLocalTests`.
4. A bound list: `RunSpecifiedTests`, with the classes from `config/sfp_testsuites.yaml` (`deploy_with`) and any codev-suggested list combined. Both are fixed at build time.
5. Otherwise: `RunRelevantTests`, where Salesforce picks the tests. In a large org that can queue thousands.

With no Apex in the payload, sandboxes run no tests. A release deploy is all or nothing.

## codev-suggested lists

A project opts in through **Test selection for diff packages** in the codev Workflow Settings (`testSelection.diffPackages`). It applies to the whole project. Signs that it's on: a `codev: Test selection` check and a test selection comment on the PR.

| Stage | What happens |
|---|---|
| PR opened or updated | Detection finds the tests for the Apex the PR changes, in the package's full source at the head. It follows the classes that use a changed class, up to the configured depth, and posts the list with a reason per class. It also lists changed Apex that no test reaches |
| Before validation | **Validation waits until every list for that head is decided.** Manual runs are refused with `Validation waits for a test decision on <package>` |
| Validation | Runs this PR's accepted list plus any classes bound in yaml, and checks 75% coverage on the Apex the PR changes |
| Build after merge | Binds the union of the accepted lists for every Apex commit in the artifact's range, plus yaml classes |
| Baseline moves on | Lists for commits behind the new build start drop out of later builds |

The waiting state is normal. It isn't a stuck run, so don't retry it.

### Deciding a list

A person decides, on the PR or on the codev validation page. The commenter must be an sfp `member` or `owner`.

```text
/tests accept [package]
/tests reject [package]
/tests add [package:]ClassA,ClassB remove [package:]ClassC
```

**Don't accept or reject a list yourself unless a person has told you to, for that PR.** It sets what guards production. Do this instead:

- Read the list and the changes marked as not mapped to tests. Detection follows Apex references, so it misses tests that reach code only through DML and triggers, and changes to fields, flows and other metadata.
- Propose which classes to add or remove, and say why.
- Tell the person the PR is waiting on their decision.

If the PR's Apex changes after a decision, a changed list goes back to pending. A list that hasn't changed keeps its decision.

### When a build ignores the lists

If **any** Apex commit in the artifact's range can't be resolved, the whole package falls back: to its yaml-bound suite if it has one, otherwise to `RunRelevantTests`. A partial list would under-test without anyone noticing. The build logs:

```text
codev-suggested tests not bound to <package>: <reasons>
```

| Reason in the log | Cause |
|---|---|
| `#12 has no test decision` | A PR in the range was never decided |
| `#12 was rejected` | A person rejected its list |
| `a1b2c3d was pushed without a pull request` | A direct push changed Apex |
| `belongs to more than one pull request`, `couldn't be found` | The commit can't be tied to one PR |
| `more than 500 commits changed its Apex` | The range is too long |
| `its baseline … isn't an ancestor of the built commit` | The range doesn't match the payload |

The block lasts until the baseline moves past that commit, so a single direct push changing Apex can affect every build until then. Before pushing Apex without a PR on such a project, warn the person.

### Things that catch people out

- **The range is set by the environment furthest behind.** Every environment runs the union for the whole range, including tests for changes it already has. While that environment stays behind, the list keeps growing.
- **A green PR doesn't guarantee a green release.** The PR ran its own list. The release runs the union against the whole payload, and Salesforce requires coverage from the tests it ran.
- **Reverts count.** A revert PR that changes Apex is a commit in the range and needs its own decision. The range is built from commits, so a list can still run for a change that nets out of the payload. See [reverting.md](reverting.md).
- **A class deleted later** is left off with a warning. A class added by hand that doesn't exist fails validation.
- **Packages that never bind:** `isOptimizedDeployment: false`, no Apex in the payload, or an empty range.
- **`skipTesting` on:** validation binds nothing and reports `NoTestRun`, so a green run hasn't tested the list.

## Previewing the list

```bash
sfp validate suggest-tests --basebranch <branch> --repository <owner/repo>
```

This prints the lists for your branch. It's in standalone `sfp` only. Don't pass `--pr-number`, which publishes the lists to the PR. To see what a built artifact will run, check the build log for this line:

```text
Package '<package>' deploys with <n> codev-suggested test classes from #12, #15
```

## Component checksum skip and tests

When the skip prunes a payload, a package without a bound list moves to `RunRelevantTests`. Bound packages keep their list.
