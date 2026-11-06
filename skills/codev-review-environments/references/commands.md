# Review environment commands

## PR comment commands

Start the comment with `/codev`. The commenter's Git identity must be linked to an sfp member.

| Command | Effect |
|---|---|
| `/codev status` | Shows the PR's environments, leases and usage |
| `/codev extend [--hours N] [--domain d]` | Extends the lease, by 24 hours by default. Fails once the org has expired, and isn't available for scratch org pools |
| `/codev extend --immortal` | Removes the expiry. Ask a person first |
| `/codev unassign [--domain d] [--reason "…"]` | Unassigns and **returns the org to the pool**. `release` is an alias |
| `/codev unassign --no-return-to-pool` | Unassigns and expires the org. A scratch org is deleted. Ask a person first |
| `/codev renew` | Open PRs only. Expires the current org and validates again on a fresh one. Ask a person first |
| `/codev assign [--pool p] [--domain d] [--expires H] [--force]` | Assigns an environment by hand |
| `/codev sync-review-env [--domain a,b] [--packages p] [--dry-run]` | Brings the PR's environment up to the branch: it deploys, as full packages, everything the org is missing according to its installed artifacts. Needs no approval, and holds the environment while it runs |
| `/codev sync <environment>` | Syncs a **named environment**, not the review environment, behind an approval |
| `/codev validate` | Starts validation, when the project triggers it from comments. Needs the `member` or `owner` role |
| `/codev record [--ret-url /path]` / `/codev record stop` | Records a browser session in the review org for reviewers, without exposing a session id. Needs the project's remote browser integration, and stops after 60 minutes |
| `/codev help [command]` | Lists the commands, or explains one |

## codev web app

The review environment page has **Open**, which logs in through the app, **Extend** (+24 hours), **Release hold**, shown while a usage lock is held, and **Unassign**, which returns the org to the pool. A sandbox assigned to you is synced from **My assignments**, and a pool's own sync never touches assigned sandboxes.

## Standalone sfp

```bash
sfp review-envs list
sfp review-envs assign --type pr --id <pr> --domain <d> --branch <base>
sfp review-envs extend ...
sfp review-envs unassign ...
sfp review-envs acquire ... / release ...   # the usage lock that validation takes
sfp review-envs rules list | match
```

Run `sfp review-envs <command> --help` for the flags. These aren't in the restricted `codev` CLI.
