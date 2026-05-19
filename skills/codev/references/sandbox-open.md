# sandbox-open

List the sandboxes the user already owns in the configured pool and open the one they pick. No new fetch from the pool — this mode is for sandboxes that are already `ASSIGNED` to the user. If they want a fresh one, use [`sandbox-fetch.md`](sandbox-fetch.md). If they want to build/deploy, use [`implement.md`](implement.md).

Shared CLI / config / `--json` conventions live in [`../SKILL.md`](../SKILL.md).

## Workflow

### 1. Resolve repo, pool, identity

```bash
REPO=$(git remote get-url origin | sed -E 's|\.git$||; s|^.*[:/]([^/]+/[^/]+)$|\1|')
POOL_TAG=$(jq -r '."pool-tag" // empty' .sfp-pro/config.json)
MY_EMAIL=$(jq -r '."server-email" // empty' .sfp-pro/config.json)

[ -z "$POOL_TAG" ]  && { echo "pool-tag missing from .sfp-pro/config.json" >&2; exit 1; }
[ -z "$MY_EMAIL" ]  && { echo "server-email missing from .sfp-pro/config.json" >&2; exit 1; }
```

`server-email` is the user's identity — used to filter assigned sandboxes to "mine". If `.sfp-pro/config.json` isn't present (e.g. running from a worktree), surface that — don't guess.

### 2. List sandboxes assigned to me

```bash
sfp pool list \
  --repository "$REPO" \
  --tag "$POOL_TAG" \
  --status ASSIGNED \
  --json
```

Returns `{ "instances": [...] }`. Filter client-side to instances where `assignedToUserEmail == $MY_EMAIL`. The server doesn't expose a `--me` flag — the filter is on you.

Key fields per instance:
- `assignmentId` — the slug used at fetch time (this is what the user will recognise)
- `sandboxName` — e.g. `354711`
- `expiresAt` — ISO timestamp; sort ascending so closest-to-expiry shows first
- `instanceUrl`, `salesforceOrgId`, `sfdxAuthUrl` — for opening / auth

### 3. Pick a sandbox

- **0 instances** — tell the user they have nothing assigned in this pool, suggest **sandbox-fetch** mode to grab one. Stop.
- **1 instance** — use it directly; don't ask.
- **2+ instances** — call `AskUserQuestion` with one option per sandbox. Label each with `assignmentId` (or `sandboxName` if no meaningful slug) plus expiry, e.g. `platform-validator-url — expires in 12h`. Description line can include the sandbox name and instance URL host.

### 4. Establish a local alias

`sfp org open` needs a locally-authed alias. Check what already exists:

```bash
DERIVED_ALIAS="sbx-$(echo "$ASSIGNMENT_ID" | tr -c '[:alnum:]-' '-' | sed 's/-\+/-/g; s/^-//; s/-$//')"

sfp org list --local-only --json \
  | jq -r --arg url "$INSTANCE_URL" '.result.nonScratchOrgs[]? // .nonScratchOrgs[]? // empty | select(.instanceUrl == $url) | .alias // .username' \
  | head -1
```

If an existing alias/username is found, reuse it. Otherwise auth fresh from the `sfdxAuthUrl` already returned by step 2:

```bash
echo "$SFDX_AUTH_URL" | sfp org login --url-stdin - --alias "$DERIVED_ALIAS" --json
```

`sfdxAuthUrl` is a session credential — pass via stdin, don't write to disk or echo to logs. Don't pass `--set-default` (this mode must not stomp on the user's current default org).

### 5. Open it

```bash
sfp org open --targetusername "$ALIAS" --json | jq -r '.result.url // .frontdoorUrl'
```

On most setups `sfp org open` also launches the browser as a side effect, so the user lands in the org without clicking. Still print the URL — they may want to paste it into a different browser/profile.

### 6. Report

Compact summary:

- **Sandbox (open in browser):** frontdoor URL on its own line
- **Assignment ID:** so the user can recognise which sandbox this is
- **Sandbox name:** e.g. `354711`
- **Alias:** the local alias they can use with subsequent `sfp ... --targetusername <alias>` calls
- **Expires:** the `expiresAt` value, plus a humanised "in Xh" if helpful
- **Pool tag:** the pool the sandbox came from

Frontdoor URLs and sfdxAuthUrls are short-lived session secrets — print once in the report, never persist.
