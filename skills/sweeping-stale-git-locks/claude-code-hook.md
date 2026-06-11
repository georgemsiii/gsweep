# gsweep Claude Code hook (sweep at session start)

A `SessionStart` hook that sweeps stale git locks the moment a **local** Claude Code session opens in a swept repo. Optional complement to the launchd agent — it guarantees a clean `.git` at session start, but locks still accumulate between local sessions (cloud sessions create them while no local session is open), so don't make it the only mechanism.

## Placement rule (the one people get wrong)

**Target file: the user-global `~/.claude/settings.json` on the host machine.**

Never put this in the repo's `.claude/settings.json` or `.claude/settings.local.json`: those files live inside the mounted folder, so **cloud sessions load them too** — and there the delete fails, producing a failing hook at the start of every cloud session. The user-global file never rides the mount.

Because user-global hooks fire in *every* project, the command guards itself by directory name and does nothing elsewhere.

## The hook

Readable form (replace `your-repo-name` with the repo's folder name):

```bash
case "$PWD" in
  */your-repo-name)
    if [ -d .git ]; then
      find .git \( -name '*.lock' -o -name '*.lock.old*' \) -type f -mmin +1 -print -delete
    fi ;;
esac
```

Multiple repos: `case "$PWD" in */repo-a|*/repo-b)`.

Same safety properties as the launchd agent: recursive over all of `.git` (don't prune `objects/`), only files older than 60 seconds (`-mmin +1` — keep it), deleted paths printed.

JSON to merge into `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "case \"$PWD\" in */your-repo-name) if [ -d .git ]; then find .git \\( -name '*.lock' -o -name '*.lock.old*' \\) -type f -mmin +1 -print -delete; fi ;; esac",
            "timeout": 10,
            "statusMessage": "gsweep: clearing stale git locks"
          }
        ]
      }
    ]
  }
}
```

## Install

1. **Read the existing `~/.claude/settings.json` first — merge, never replace.**
   - No `"hooks"` key → add the whole block alongside existing keys.
   - `"hooks"` exists, no `"SessionStart"` → add the `"SessionStart": [...]` entry inside it.
   - `"SessionStart"` already has entries → append this object to that array.
2. Validate (a malformed settings.json silently disables everything in that file):
   ```bash
   jq -e '.hooks.SessionStart[].hooks[] | select(.type == "command") | .command' ~/.claude/settings.json
   # expected: prints the command, exit 0
   ```
3. Pipe-test the command with a planted stale lock:
   ```bash
   cd /path/to/your-repo
   touch -t 202001010000 .git/gsweep-test.lock
   echo '{}' | bash -c 'case "$PWD" in */your-repo-name) if [ -d .git ]; then find .git \( -name "*.lock" -o -name "*.lock.old*" \) -type f -mmin +1 -print -delete; fi ;; esac'
   # expected: prints .git/gsweep-test.lock, and the file is gone
   ```
4. Activate: settings apply to **new** sessions; for an already-open one, run `/hooks` once (reloads hook config) or restart.

## Manage

- `/hooks` inside Claude Code shows, edits, and disables hooks.
- Remove by deleting the entry from `~/.claude/settings.json`; re-run the `jq` check after editing.

## Limitations

- Fires only when a local session starts **in the repo root** (the `$PWD` guard matches the folder name); sessions started in subdirectories skip it. New sessions, `/clear`, and resumes all count as session starts.
- Does nothing for plain terminal `git` between sessions — the launchd agent covers that gap.

## References

- Claude Code hooks: https://docs.claude.com/en/docs/claude-code/hooks
- Claude Code settings: https://docs.claude.com/en/docs/claude-code/settings
