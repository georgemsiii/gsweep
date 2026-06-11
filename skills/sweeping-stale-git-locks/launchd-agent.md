# gsweep launchd agent (macOS background sweep)

A LaunchAgent that deletes stale git locks from listed repos every 5 minutes, whether or not any agent session is open. Recommended primary mechanism: the locks are stranded by cloud sessions *while no local session is running*, so the cleanup can't depend on one.

## Customize these three things

1. **Repo path(s)** — each swept repo's `.git` directory, as an **absolute** path (one `<string>` each).
2. **Log path** — absolute path; `~` and `$HOME` are **not expanded** in launchd plists.
3. Nothing else. The label stays `com.gsweep.agent`; one agent covers all repos.

## Design constraints (do not "improve" these away)

- **Command lives inline in the plist — never point launchd at a script inside the swept repo.** The repo is writable by cloud sessions; a cloud-writable file executed automatically by the host is a privilege-escalation channel. The plist lives in `~/Library/LaunchAgents/`, outside any mount.
- **`-mmin +1` is the safety guard** (only files >60s old; git holds locks for sub-second writes, and on a can't-delete mount anything older is permanently stale). Keep it.
- **The sweep is recursive over all of `.git` — do not prune `objects/`** (real strandings include `objects/maintenance.lock` and `refs/heads/<branch>.lock`).
- `*.lock.old*` is included because cloud-side sessions can only rename locks aside (`index.lock.old<epoch>`); the host sweep is what finally removes that debris.

## The plist

Save as `~/Library/LaunchAgents/com.gsweep.agent.plist` (replace `/Users/YOURUSER/path/to/your-repo` and the log paths):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.gsweep.agent</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/find</string>
        <string>/Users/YOURUSER/path/to/your-repo/.git</string>
        <!-- more repos? add each .git path as another <string> right here -->
        <string>(</string>
        <string>-name</string>
        <string>*.lock</string>
        <string>-o</string>
        <string>-name</string>
        <string>*.lock.old*</string>
        <string>)</string>
        <string>-type</string>
        <string>f</string>
        <string>-mmin</string>
        <string>+1</string>
        <string>-print</string>
        <string>-delete</string>
    </array>

    <key>StartInterval</key>
    <integer>300</integer>

    <key>RunAtLoad</key>
    <true/>

    <key>ProcessType</key>
    <string>Background</string>

    <key>StandardOutPath</key>
    <string>/Users/YOURUSER/Library/Logs/gsweep.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOURUSER/Library/Logs/gsweep.log</string>
</dict>
</plist>
```

`-print -delete` logs every deleted path — a free audit trail.

## Install

```bash
plutil -lint ~/Library/LaunchAgents/com.gsweep.agent.plist        # must print: OK
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.gsweep.agent.plist
launchctl print gui/$(id -u)/com.gsweep.agent | head -15          # confirm loaded
```

On older macOS where `bootstrap` errors: `launchctl load -w ~/Library/LaunchAgents/com.gsweep.agent.plist`.

## Test (proves the sweep AND the safety guard)

```bash
cd /Users/YOURUSER/path/to/your-repo
touch -t 202001010000 .git/gsweep-test.lock              # backdated → instantly "stale"
launchctl kickstart gui/$(id -u)/com.gsweep.agent        # force a run now
ls .git/gsweep-test.lock                                 # expected: No such file
cat ~/Library/Logs/gsweep.log                            # expected: shows the deleted path

touch .git/gsweep-fresh.lock                             # fresh → must SURVIVE
launchctl kickstart gui/$(id -u)/com.gsweep.agent
ls .git/gsweep-fresh.lock                                # expected: still there
rm .git/gsweep-fresh.lock                                # clean up
```

## Manage

```bash
launchctl disable gui/$(id -u)/com.gsweep.agent          # pause
launchctl enable  gui/$(id -u)/com.gsweep.agent          # resume
launchctl bootout gui/$(id -u)/com.gsweep.agent && rm ~/Library/LaunchAgents/com.gsweep.agent.plist   # uninstall
# after editing the plist: bootout, then bootstrap again (launchd reads it at load time)
```

## Gotchas

- LaunchAgents run only while logged in and awake; missed intervals coalesce into one run on wake. Fine — locks only matter when the machine is in use.
- If a repo path moves, the job logs `No such file or directory`; update the plist and reload.
- If the test deletes nothing and logs nothing, check System Settings → Privacy & Security → Full Disk Access (repos under TCC-protected folders like Documents/Desktop need it; most dev folders don't).
- Log grows one line per deleted lock; truncate anytime: `: > ~/Library/Logs/gsweep.log`

## References

- Apple, Creating Launch Daemons and Agents: https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html
- `man launchd.plist`, `man launchctl`, `man find`
