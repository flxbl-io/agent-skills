# Troubleshooting review environments

## Validation fails on things the PR didn't change

The review org is probably **behind**. A pool sandbox is cloned at one point, then only ever receives PR artifacts. After a release moves the baseline on, the commits in between are in neither the sandbox nor any PR.

**How to check:** the validation report gives the org's installed commit against the baseline, for example `<org> at X, behind the baseline Y, so the diff starts at Z`. Pool and assignment pages show a baseline state:

| State | Meaning |
|---|---|
| `current` | At the branch head |
| `behind-head` | Behind the head but not the baseline. Validation works, with a larger payload |
| `behind-baseline` | Missing commits that no PR will carry. Validation widens the diff to cover them, which can surface unrelated failures |
| `unknown` | Its commit couldn't be placed. The page gives the reason |

**Fix:** comment `/codev sync-review-env`, or sync the sandbox from **My assignments**. Then validate again.

Thorough validation, the default, already starts the diff from the org's installed commit when that's older than the baseline. A sync still helps, because it deploys full packages rather than a diff.

## Validation is waiting

- **For the usage lock:** another run holds the environment, and validation waits. A lock that's never released times out after about 90 minutes. **Release hold** in the web app frees it, but only use it when no run is active.
- **For a test decision:** on projects using codev-suggested test lists, validation waits until every list is decided.
- **For capacity:** with no free instance, codev falls back to another pool where the rules allow it, or waits for one.

## The environment has expired

`extend` no longer works once the org has expired. Comment `/codev renew`, after asking a person, to validate on a fresh org. Anything that existed only in the old org is gone.

## The org looks different from the repository

Someone changed it by hand, or an earlier PR's artifact is still in it. Don't fix it by hand. Sync it to the branch with `/codev sync-review-env`, or renew it.

## Another PR's change shows up in this org

The PR is on a **dedicated environment**, which PRs share, or the org came back from the pool with an earlier PR's state. Sync it before relying on what you see.
