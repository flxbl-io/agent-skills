---
name: codev-review-environments
description: How review environments work in a Flxbl project validated by codev, and how an agent should use them. A review environment is an org assigned to a pull request, where codev deploys the PR's artifact for real to validate it. Covers how a PR gets one, leases and expiry, what unassign, renew and merge do to the org, PR comment commands, syncing an environment that is behind, opening it to inspect a change, and what to leave to a person. Use when a PR's validation fails or waits, when asked to open, extend, sync, renew or release a PR's environment, or before doing any work in one.
---

# codev-review-environments

A **review environment** is an org assigned to a pull request. When the PR is validated, codev builds the PR's artifact and deploys it to this org with a real deployment: deployment scripts run, and data-dependent failures show up, unlike a check-only run.

| Task | Reference |
|---|---|
| PR comment commands and CLI commands, with their exact effect | [references/commands.md](references/commands.md) |
| Validation fails, waits, or the environment is behind, expired or missing | [references/troubleshooting.md](references/troubleshooting.md) |

## Where the org comes from

- **Assignment rules** pick it, by target branch, domain and priority. The target can be a **sandbox pool**, a **scratch org pool** or a **dedicated environment**.
- **One assignment per domain:** a PR that changes two domains can hold two environments.
- **Dedicated environments are shared** between PRs, so one PR's validation can affect another's.
- **Review pools aren't dev pools.** Never take an instance from a review pool with `codev pool fetch`. Pools are taken by validation, and an agent's fetch can leave a PR waiting.

## Lifecycle

1. **A PR event triggers validation:** opening it, pushing to it, reopening it or marking it ready. Draft PRs are skipped. A project can instead trigger validation from checks, comments or the API.
2. **Validation assigns an environment**, or reuses the PR's existing one, then takes an exclusive **usage lock** for the run. When the run ends it releases the lock, not the org: the PR keeps its environment.
3. **The assignment has a lease.** The pool's lease length applies, and `extend` adds time. Scratch org pools can't be extended.
4. **When the PR ends:**

| Event | What happens to the org |
|---|---|
| PR merged | Unassigned. Returned to the pool only if the pool's `returnToPoolOnUnassign` is on, which is off by default. Otherwise it's marked expired and left to the pool's refresh-or-delete strategy |
| PR closed without merging | **Kept**, so reopening the PR resumes on the same org |
| `unassign` comment | Unassigned and **returned to the pool**, unless `--no-return-to-pool` |
| `renew` comment | The org is **expired**, and validation runs again on a fresh one |

Running validations are cancelled when a PR is merged or closed. A scratch org that isn't returned to the pool is deleted.

## How to use one well

- **Treat it as a copy of where your PR is going, not a workspace.** A change you make by hand in a review org isn't in the repository: the next validation can overwrite it, reviewers can't see it, and it won't reach any other environment. Fix the PR, and let validation redeploy.
- **Use it to look, test and reproduce.** Open it to check how the change behaves, run Apex tests, or record a session for reviewers.
- **Before debugging a strange failure, check whether the org is behind.** A sandbox cloned before the latest release is missing commits that no PR carries, so it fails for reasons unrelated to the PR. Look at its baseline state, and sync it if it's behind.
- **Extend before long work.** An expired environment can't be extended. `renew` replaces it, and anything only in the old org is lost.
- **One PR, one environment.** Don't point another PR's work at it.

## Shape the PR so its environment means something

Validation deploys **this PR's change** to **this PR's environment**. It shows whether the change works on its own, on top of its target branch. How a change is split into PRs decides whether that result means anything:

| PR shape | What goes wrong |
|---|---|
| **Depends on another open PR**, for example the object in one PR and the permission set that uses it in another | This PR's environment doesn't have the other PR's change. Validation fails for a reason the PR can't fix, or passes on a state no environment will ever be in |
| **Very large**, many unrelated changes or many domains | Validation is slow and holds one environment per domain. One failure blocks everything, failures mix together, and the lease can run out before it's fixed |
| **Open for a long time** | Other merges move the baseline on, so the environment falls behind, and its lease expires. The result drifts away from what will actually be released |
| **Too granular**, a feature split into many small PRs | Every PR takes its own environment and validation run, so pool capacity and time go on states that never ship. Pieces that are only valid once the rest lands fail, or prove nothing |

**The rule: one coherent change per PR.** It must be:

- **Deployable** on its own, on top of its target branch, with everything it needs in the same PR: the metadata, the components that reference it, tests and any deployment script.
- **Reviewable** by a person or an agent in one sitting.
- **Testable** in its review environment: there's something to check once it's deployed.

Split a change only when each part still passes all three. A smaller PR that fails one of them costs an environment and a validation run without telling anyone anything.

Also:

- **Merge dependencies first.** If a change needs another one, merge that one, then sync this PR's environment with `/codev sync-review-env` before trusting its validation.
- **Keep PRs short-lived.** Merge, or sync, before the environment falls far behind.
- **Prefer one domain per PR** where the change allows it.

## Ask a person first

- **`renew`**: it throws away the current org.
- **Unassigning an environment you didn't take**, or with `--no-return-to-pool`, which expires it. Unassigning a scratch org without returning it deletes it.
- **`extend --immortal`**: it takes an org out of the pool indefinitely.
- **Syncing a dedicated environment**: other PRs share it.
- **Changing assignment rules or pool settings.**

## CLI

The restricted `codev` CLI doesn't include review-environment commands. Agents work with review environments through PR comments and the codev web app. Standalone `sfp` has `sfp review-envs ...`, described in [references/commands.md](references/commands.md).

Never print an environment's auth URL or a frontdoor login URL. Open an org through the codev web app, or let a person do it.
