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
brew install akgandlur/tap/crestore
crestore install          # adds a small hook to ~/.zshrc (once)
```

Restoring sends `Cmd+N` / `Cmd+D` to Ghostty through AppleScript, so Ghostty needs Accessibility access:
System Settings → Privacy & Security → Accessibility → enable Ghostty.

To uninstall: `brew uninstall crestore`, delete the `# --- crestore` block from `~/.zshrc`, and optionally
`rm -rf ~/.claude-snapshots`.

## How it works

- Live sessions are read from Claude Code's own session files (`~/.claude/sessions/<pid>.json`): session id,
  working directory and name. Only interactive sessions owned by you are included.
- Sessions are saved in terminal-tty order, so panes you opened together stay neighbours and share a window
  when restored.
- Restore never types into a terminal (keystrokes can land in the wrong pane). It queues one
  `cd <dir> && claude --resume <id>` per session in `~/.claude-snapshots/queue/`, and the zsh hook makes each
  **new** shell claim one entry at its first prompt. crestore only presses `Cmd+N` / `Cmd+D`, waiting for
  each pane to claim its session before moving on, and retrying a keystroke once if it didn't.
  Queue entries older than two minutes are ignored.
- `close` sends SIGTERM to each session, waits for it to exit, then hangs up its shell so the pane closes.
  Claude Code writes its transcript as it goes, so nothing is lost and `--resume` picks up where you were.
  If you run it from inside a Claude session, that one is closed last.

## Configuration

| Variable              | Default                 | Meaning                                  |
| --------------------- | ----------------------- | ---------------------------------------- |
| `CRESTORE_DIR`        | `~/.claude-snapshots`   | where snapshots and the queue live       |
| `CRESTORE_PER_WINDOW` | `2`                     | sessions per Ghostty window              |
| `CRESTORE_DELAY`      | `1.2`                   | seconds between keystrokes               |

## Requirements

macOS, Ghostty with the default `super+n` (new window) / `super+d` (split right) bindings, zsh as your
shell, `jq`.

The session-file format is a Claude Code internal, not a documented API. If an update changes it, `save`
will find no sessions until crestore is updated.
