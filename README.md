# carboncopy

Copies the *shape* of a private [Gitea](https://about.gitea.com/) instance's
commit history onto a GitHub contribution graph, and nothing else about it.

Work that lands on a self-hosted forge is invisible on GitHub. `bin/carboncopy`
walks every non-mirror repository the configured token can see, and for each
commit on its default branch authored by you, appends one line to `ledger.txt`
and makes one commit stamped with that commit's original author date. GitHub
attributes a contribution by author date and author email, so the graph ends up
with the same shape as the private instance's — a day at a time, without that
instance being reachable from, or known to, GitHub.

## What this repository does not contain

No repository names, commit messages, file names, branch names, or source commit
hashes. A ledger line is

```
sha256(<owner>/<name> + NUL + <commit hash>) truncated to 16 hex characters
```

— enough to recognise a commit already copied, useless to anyone who cannot
already guess both halves. That is the whole state: the ledger on the push
remote is the only record of what has been copied, which is what makes a failed
push self-healing and a lost machine a non-event.

If you add anything to what gets written here, keep it on that side of the line.

## Requirements

- bash 4+, `git`, `curl`, `jq`.
- A Gitea access token with read scope.
- The author email on the copies must be **verified on the GitHub account**.
  Commits with an unverified address land correctly and contribute nothing.
- The copies must land on this repository's **default branch**, and the
  repository must not be a fork — GitHub counts neither otherwise.
- If this repository is private, turn on *Settings → Public profile → Include
  private contributions on my profile*, or the graph stays empty for everyone
  but you.

## Configuration

All of it is environment, so nothing about the private instance has to be
written down in a public repository.

| Variable | |
| --- | --- |
| `CARBONCOPY_GITEA_URL` | Base URL of the instance. Required. |
| `CARBONCOPY_GITEA_TOKEN` | Gitea access token, read scope. Required. |
| `CARBONCOPY_AUTHORS` | Author emails to copy, comma or space separated. Required, and deliberately has no default — an empty allowlist meaning "everyone" would put dependency-bot merges and other people's commits on your graph as your own work. |
| `CARBONCOPY_EXCLUDE` | `owner/name` entries to skip. |
| `CARBONCOPY_REMOTE` | Push remote. Default `origin`. |
| `CARBONCOPY_BRANCH` | Branch to commit on. Default: the current one. |
| `CARBONCOPY_NAME`, `CARBONCOPY_EMAIL` | Identity on the copies. Default: this checkout's `git config`, `commit.gpgsign` included — a checkout that already signs your commits keeps signing these. |

```sh
export CARBONCOPY_GITEA_URL=https://gitea.example.com/
export CARBONCOPY_GITEA_TOKEN=…
export CARBONCOPY_AUTHORS=you@example.com

bin/carboncopy --dry-run   # what it would copy, by day
bin/carboncopy             # copy and push
```

## Running it once a day

The checkout is machine-owned: every run resets it to the push remote before
reading the ledger, so it must not be a checkout you also work in. A systemd
timer on a box that can reach both the instance and GitHub is enough — see
`carboncopy.service` / `carboncopy.timer` in the Ansible repository that
deploys this one.

Nothing about the schedule is load-bearing. The copies carry the source's
timestamps rather than the run's, so a missed day, a catch-up run, or a first
run against ten years of history all land on exactly the same days.

## What it deliberately does not do

**Mirrors are skipped.** A mirror's commits already exist wherever it mirrors
from, which for a homelab instance usually means GitHub — already counted, and
copying them would count them twice. Forks are *not* skipped: a fork's history
is mostly upstream's, and the author allowlist already leaves all of it behind.

**Only the default branch is walked.** Under `--all`, every commit on an open
branch would count whether or not it ever landed, and a squash merge would count
the branch's commits and then the commit that replaced them. The cost, stated
rather than hidden: work in progress is invisible until it merges, and a
squash-merged pull request contributes exactly one dot — on the day it merged,
since that is the author date the forge stamps a squash with.

**It is never mirrored back onto the instance it reads.** As a mirror it would
be skipped anyway, but pushed there as an ordinary repository it would feed on
its own output, every run copying the copies the last one made. The script skips
any repository whose name matches its own push remote as a safety belt; the rule
it belts is simpler — this one goes one way.

**A rewritten source history is copied again.** A rebase gives a commit a new
hash, and a new hash is a new ledger line. The alternative — keying on author
date and message — would dedupe rebases at the cost of collapsing genuinely
distinct commits that happen to share both.

**The ledger never shrinks.** A deleted repository's copied commits stay copied.
Removing them would mean rewriting this repository's history to take days off a
graph that did happen.

**It never reads the instance over anything but the API,** and never clones from
it. The token is the whole access boundary, and read scope is all it needs.
