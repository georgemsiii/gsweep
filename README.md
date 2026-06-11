# gsweep

A Claude Code plugin that permanently fixes stale `.git` lock files in repos used with Claude Cowork (or any mounted/synced folder where git can create files but not delete them).

## The symptom

```
fatal: Unable to create '/path/to/repo/.git/index.lock': File exists.
```

…with **no git process running**, recurring every few days, ever since you linked the folder to Cowork. Sometimes it's `HEAD.lock`, `config.lock`, or `objects/maintenance.lock` instead.

## Why it happens

Git on a Cowork-style mount can **create and rename files but cannot delete them**. Git's locking works by creating a lock file and deleting it sub-seconds later — so on the mount, even *fully successful* cloud-session commits strand locks that nothing on the cloud side can ever remove. They sit in your local `.git` until a write trips over them.

Two non-obvious consequences (both observed in the wild):

- No crash or interruption is required — expect debris from ordinary, healthy cloud git activity.
- Locks strand **deep inside `.git`** too (`objects/maintenance.lock`, `refs/heads/<branch>.lock`), so top-level `rm .git/*.lock` under-cleans.

Because git holds locks only for sub-second writes, any lock **older than ~60 seconds** on such a mount is permanently stale — which makes automated, age-guarded deletion from the host side safe.

## What the skill does

One skill, `gsweep:sweeping-stale-git-locks`, that Claude triggers when you hit the error (or you invoke directly). It:

1. Diagnoses the locks in your repo (recursive, with ages).
2. Sweeps them safely (only files older than 60 seconds).
3. Installs durable automation, parameterized to your repo paths:
   - a **macOS launchd agent** (recommended) — background sweep every 5 minutes, no session needed;
   - and/or a **Claude Code SessionStart hook** — sweeps whenever you open a session in the repo (placed in user-global settings, deliberately *not* in the repo's own `.claude/`, which cloud sessions would also load).
4. Verifies the install with a planted backdated lock, and confirms fresh locks survive.

It also teaches the cloud-side workaround (rename locks aside, since deletion is impossible there) and the safety rules that make all of this non-destructive.

## Install

**As a plugin (Claude Code):**

```
/plugin marketplace add georgemsiii/gsweep
/plugin install gsweep@gsweep
```

**Manual (any agent supporting [Agent Skills](https://agentskills.io), e.g. into `~/.claude/skills/`):**

```bash
git clone https://github.com/georgemsiii/gsweep.git
cp -R gsweep/skills/sweeping-stale-git-locks ~/.claude/skills/
```

## Use

Just bring Claude the error — the skill triggers on it. Or ask directly: *"set up gsweep for this repo"*. Everything the skill installs is inspectable first: the templates live in [skills/sweeping-stale-git-locks/](skills/sweeping-stale-git-locks/).

## Safety properties

- **Age guard:** nothing younger than 60 seconds is ever deleted; live git operations are untouchable by construction.
- **Scoped:** only repos you list are swept — never a blanket filesystem sweep.
- **Auditable:** every deleted path is printed/logged.
- **No remote execution surface:** the launchd job embeds its command in the plist; it never executes anything stored inside the (cloud-writable) repo.

## Origin

Built June 2026 after watching a Cowork-linked repo strand a zero-byte `HEAD.lock` whose mtime matched a cloud session's commit to the second — and finding a second lock hiding in `.git/objects/`. License: MIT.
