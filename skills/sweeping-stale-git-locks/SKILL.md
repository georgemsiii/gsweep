---
name: sweeping-stale-git-locks
description: Use when git fails with "Unable to create '.git/index.lock': File exists" (or HEAD.lock, config.lock, objects/maintenance.lock) with no git process running — especially in repos linked to Claude Cowork or another mounted/synced folder, when lock files keep reappearing over days, or when a cloud/mounted session's git cannot delete files it created.
---

# Sweeping Stale Git Locks

## Overview

On Cowork-style mounted folders, git can **create and rename files but cannot delete them**. So every cloud-session git write — including fully successful commits — can strand a `.git` lock file that nothing on the cloud side can ever remove. Git holds locks for sub-second writes by design, therefore: **any lock older than ~60 seconds on such a mount is permanently stale, and deleting it from the host side is safe.**

## Wrong mental models (these produce bad fixes)

| Assumption | Reality |
|---|---|
| "A crashed/interrupted session left this" | No crash needed. Deletes fail on the mount, so healthy operations strand locks too. Expect debris from any cloud git write, not from rare incidents. |
| "`lsof` shows nothing holding it, so it's safe" | The lock's creator is a remote process; `lsof` only sees local ones. **File age is the only reliable safety signal here.** |
| "Sweep `.git`'s top level / prune `objects/`" | Confirmed-in-the-wild strandings include `.git/objects/maintenance.lock` (auto-maintenance after a cloud commit) and branch locks under `refs/`. Sweep **all of `.git` recursively**. |

## Workflow

1. **Diagnose** (host side): `find .git -name '*.lock*' -type f -exec stat -f '%N | %Sm | %z bytes' {} +` — stale locks are typically zero-byte and minutes-to-days old (macOS `stat`; on Linux use `stat -c`). Confirm the repo is mounted/synced (e.g. Cowork-linked).
2. **Immediate safe sweep** (host side, substitute the real repo path):
   ```bash
   find /path/to/repo/.git \( -name '*.lock' -o -name '*.lock.old*' \) -type f -mmin +1 -print -delete
   ```
3. **If you ARE the cloud/mounted session** (deletes fail there): rename locks aside instead of deleting — `[ -f .git/index.lock ] && mv .git/index.lock .git/index.lock.old$(date +%s)` before git writes. The host-side sweep clears the `*.lock.old*` debris later; that is why the sweep pattern includes it.
4. **Install durable automation** (host side) — read and apply, substituting the user's real absolute paths:
   - **launchd-agent.md** (macOS, recommended primary): background sweep every 5 minutes, runs whether or not any agent session is open — which matters, because the locks appear while no local session is running.
   - **claude-code-hook.md** (optional complement): sweeps when a Claude Code session starts in the repo.
5. **Verify**: plant a backdated lock (`touch -t 202001010000 .git/gsweep-test.lock`), trigger the automation, confirm it is deleted and logged; plant a fresh lock, confirm it survives; remove it.

## Hard rules

- **Never delete a lock younger than 60 seconds.** That age guard is the entire safety argument — automation must keep `-mmin +1`.
- **The Claude Code hook goes in user-global `~/.claude/settings.json` — never in the repo's own `.claude/` settings.** Repo settings ride the mount into cloud sessions, where the delete fails noisily at every session start.
- **launchd must never execute anything stored inside the mounted repo.** A cloud-writable file executed automatically by the host is a privilege-escalation channel; keep the command embedded in the plist (the reference template does this).
- If the user would rather eliminate the cause than manage it: give the cloud tool its own clone instead of the live folder; locks then strand only in the clone.
