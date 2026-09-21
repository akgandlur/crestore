# crestore

Snapshot your running [Claude Code](https://claude.com/claude-code) sessions and bring them all back in
[Ghostty](https://ghostty.org), two per window using splits. macOS only.

You have a handful of Claude sessions open across Ghostty panes, then you reboot or quit Ghostty. Getting
them back means remembering each directory and session id for `claude --resume`. crestore does that for you.

```
crestore save [label]      snapshot all live interactive Claude sessions
crestore close [label]     snapshot, then quit every session and close its Ghostty pane
crestore list              list snapshots, newest first (index, saved-at, count, names)
crestore show [snap]       print the sessions inside a snapshot
crestore [snap]            reopen a snapshot (default: latest); sessions already running are skipped
crestore install           add the zsh hook that lets new panes pick up restored sessions
crestore help
```

`<snap>` is nothing or `latest`, an index from `crestore list`, a filename prefix, or a path.
Snapshots are plain TSV files in `~/.claude-snapshots/`, named by save time so they sort by date.

```
$ crestore save
saved 3 session(s) -> ~/.claude-snapshots/2026-09-19_15-27-59.tsv
  f2d77065  api-3f                           ttys000  ~/code/api
  ebf413ca  api-c1                           ttys001  ~/code/api
  c2543a62  web-7a                           ttys002  ~/code/web

$ crestore list
#    saved at               n     sessions
1    2026-09-19_15-27-59    3     api-3f, api-c1, web-7a
2    2026-09-18_18-02-10    2     api-3f, web-7a
```

## Install

```
brew trust akgandlur/tap  # Homebrew 7+ refuses formulae from third-party taps until you trust them (once)
brew install akgandlur/tap/crestore
crestore install          # adds a small hook to ~/.zshrc (once), and checks Accessibility
```

If `brew install` says `Refusing to load formula akgandlur/tap/crestore from untrusted tap`, that first line
is what's missing. Older Homebrew doesn't have `brew trust` and doesn't need it.

Restoring sends `Cmd+N` / `Cmd+D` to Ghostty through AppleScript, and macOS only allows that if the
terminal app you run crestore from is trusted for Accessibility:
System Settings → Privacy & Security → Accessibility → enable Ghostty.

macOS drops those keystrokes *silently* when the grant is missing — no error, no new pane — so crestore
checks for it rather than letting you find out through a confusing failure. `crestore install` tells you
where you stand, and a restore refuses up front instead of queueing sessions it cannot open. If Ghostty is
already listed under Accessibility, switch it off and on and restart it: the grant only reaches processes
started afterwards.

Upgrade with `brew upgrade crestore`. To uninstall: `brew uninstall crestore`, delete the `# --- crestore` block from `~/.zshrc`, and optionally
`rm -rf ~/.claude-snapshots`.

## How it works

- Live sessions come from `claude agents --json`: session id, working directory, name and whether it's busy.
  Only interactive sessions owned by you are included. Older Claude Code without that subcommand falls back
  to its session files (`~/.claude/sessions/<pid>.json`).
- Sessions are saved in terminal-tty order, so panes you opened together stay neighbours and share a window
  when restored.
- Restore never types into a terminal (keystrokes can land in the wrong pane). It queues one
  `cd <dir> && claude --resume <id>` per session in `~/.claude-snapshots/queue/`, and the zsh hook makes each
  **new** shell claim one entry at its first prompt. crestore only presses `Cmd+N` / `Cmd+D`, waiting for
  each pane to claim its session before moving on, and retrying a keystroke once if it didn't.
  Queue entries older than two minutes are ignored.
- If a pane still hasn't claimed after the retry, crestore checks whether any new shell appeared at all.
  None means the keystroke never landed and it points you at Accessibility; one that appeared but didn't
  claim means the zsh hook isn't running, which is a different fix.
- `close` sends SIGTERM to each session, waits for it to exit, then hangs up its shell so the pane closes.
  Claude Code writes its transcript as it goes, so nothing is lost and `--resume` picks up where you were.
  If any session is busy (mid-task) it lists them and asks first; non-interactive runs just warn.
  If you run it from inside a Claude session, that one is closed last.

## Configuration

| Variable              | Default                 | Meaning                                  |
| --------------------- | ----------------------- | ---------------------------------------- |
| `CRESTORE_DIR`        | `~/.claude-snapshots`   | where snapshots and the queue live       |
| `CRESTORE_PER_WINDOW` | `2`                     | sessions per Ghostty window              |
| `CRESTORE_DELAY`      | `1.2`                   | seconds between keystrokes               |

## Requirements

macOS, Claude Code, Ghostty with the default `super+n` (new window) / `super+d` (split right) bindings, zsh as
your shell, `jq`, and Accessibility access for the terminal app you run crestore from.
