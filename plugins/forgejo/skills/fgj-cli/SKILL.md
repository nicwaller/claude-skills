---
name: fgj-cli
description: Use when the user asks about the "fgj" CLI (a GitHub-CLI-style `gh` equivalent for Forgejo instances, including Codeberg) — authenticating, and managing pull requests, issues, repositories, releases, labels, milestones, and Forgejo Actions from the terminal. Covers command structure, common flags, and the raw `fgj api` escape hatch for anything without a dedicated command.
---

# fgj CLI

`fgj` brings pull requests, issues, and other Forgejo concepts to the terminal —
the same role `gh` plays for GitHub. It talks to any Forgejo instance (self-hosted
or Codeberg).

```
fgj version 0.5.0
```

## Global flags

Available on every command:

- `--hostname string` — Forgejo instance hostname (e.g. `codeberg.org`). Falls
  back to the auth'd/default instance if omitted.
- `--config string` — config file (default `$HOME/.config/fgj/config.yaml`)

Most repo-scoped commands also take `-R, --repo owner/name` — if omitted, fgj
auto-detects the repo from the current git working directory.

## Auth

```bash
fgj auth login --hostname codeberg.org -t <personal-access-token>
fgj auth login --hostname codeberg.org   # prompts for token if -t omitted
fgj auth status                          # show configured instances and their state
fgj auth token                           # print the stored token
fgj auth logout --hostname codeberg.org
```

Multiple instances can be authenticated at once; `--hostname` on any command
picks which one to use.

## Pull requests (`fgj pr`, alias `pull-request`)

```bash
fgj pr create -t "Title" -b "Body" -H feature-branch -B main
fgj pr create -t "Title" -H feature-branch -a @me         # self-assign
fgj pr list -s all                                        # open|closed|all, default open
fgj pr list --json
fgj pr view 42
fgj pr status                                              # PRs for current branch / created by you / assigned to you
fgj pr checkout 42                                          # fetches refs/pull/42/head, works for fork PRs too
fgj pr checkout 42 -b local-name --force
fgj pr comment 42 -b "LGTM"
fgj pr edit 42 --add-label bug --remove-label wip --add-reviewer alice
fgj pr merge 42 --merge-method squash                       # merge|rebase|squash, default merge
fgj pr close 42
fgj pr reopen 42
```

## Issues (`fgj issue`)

```bash
fgj issue create -t "Title" -b "Body" -l bug -l priority/high
fgj issue list -s open --json
fgj issue list -l bug
fgj issue view 7
fgj issue comment 7 -b "Reproduced on 0.5.0"
fgj issue edit 7 --add-label bug --remove-label wontfix -s closed
fgj issue close 7 -c "Fixed in 1a2b3c4"
fgj issue reopen 7
```

## Repositories (`fgj repo`)

```bash
fgj repo create myrepo -d "description" --private --add-readme -g Go -l MIT
fgj repo create myorg/myrepo -c            # create under an org, clone immediately
fgj repo clone owner/name [destination] -p ssh   # protocol: https (default) or ssh
fgj repo list                              # your own repos
fgj repo view owner/name --json
fgj repo fork owner/name
```

## Releases (`fgj release`, alias `releases`)

```bash
fgj release create v1.2.3 ./dist/*.tar.gz -n "release notes" --target main
fgj release create v1.2.3 --draft --prerelease
fgj release list --limit 50 --json
fgj release view v1.2.3          # or "latest"
fgj release upload v1.2.3 ./more-assets.zip --clobber
fgj release delete v1.2.3 -y
```

## Labels (`fgj label`)

```bash
fgj label create bug -c ff0000 -d "Something broken"
fgj label create priority/high -c ff9900 --exclusive
fgj label list --json
fgj label edit bug -n defect -c cc0000
fgj label delete wontfix -y
```

## Milestones (`fgj milestone`)

```bash
fgj milestone create "v1.0" -d "First stable release" --due 2026-12-31
fgj milestone list -s all
fgj milestone edit "v1.0" --state closed
fgj milestone delete "v1.0" -y
```

## Forgejo Actions (`fgj actions`, alias `action`)

Nested command tree for CI: `run`, `runner`, `secret`, `variable`, `workflow`.

```bash
# Workflow runs
fgj actions run list -L 20
fgj actions run view <run-id> --log-failed
fgj actions run view <run-id> -j <job-id> --log
fgj actions run watch <run-id> -i 5s     # poll until the run completes
fgj actions run rerun <run-id>
fgj actions run cancel <run-id>

# Workflows
fgj actions workflow list
fgj actions workflow run <workflow-file> -r main -f env=prod
fgj actions workflow enable <workflow-file>
fgj actions workflow disable <workflow-file>

# Runners
fgj actions runner list
fgj actions runner register my-runner --ephemeral
fgj actions runner delete <id>

# Secrets (value read from stdin)
echo -n "secretvalue" | fgj actions secret create MY_SECRET
fgj actions secret list
fgj actions secret delete MY_SECRET

# Variables
fgj actions variable create MY_VAR "value"
fgj actions variable update MY_VAR "new-value"
fgj actions variable get MY_VAR
fgj actions variable list
fgj actions variable delete MY_VAR
```

## Escape hatch: `fgj api`

For anything without a dedicated command, hit the Forgejo REST API directly
through the stored auth:

```bash
fgj api repos/owner/name/branch_protections
fgj api repos/owner/name/commits/SHA/status

# POST with fields (auto-becomes POST when fields/--input given)
fgj api repos/owner/name/issues -f title="bug" -f body="details"

# typed fields: true/false/null/integers become JSON literals; @file reads a value
fgj api repos/owner/name/issues/1 -X PATCH -F state=closed

# raw body from file or stdin
fgj api repos/owner/name/contents/file.txt --input ./file.txt -X PUT

# pagination
fgj api repos/owner/name/issues -p --slurp

# include response headers, custom method/headers
fgj api repos/owner/name -i -X GET -H "Accept: application/json"
```

Notes:
- A leading `/api/v1` is added automatically if omitted; an absolute
  `https://...` URL is used as-is.
- Owner/repo are never auto-filled here — write them out explicitly (unlike
  the dedicated `pr`/`issue`/`repo` commands, which auto-detect from git).
- `-f/--raw-field` sends plain strings; `-F/--field` sends typed JSON values
  and supports `@filename` / `@-` to read a value from a file or stdin.
- Repeat a key with `[]` suffix (`-f 'items[]=a' -f 'items[]=b'`) to build an
  array.

## Other

```bash
fgj completion zsh > ~/.zsh/completions/_fgj   # bash|zsh|fish|powershell
fgj manpages --dir ./man
```
