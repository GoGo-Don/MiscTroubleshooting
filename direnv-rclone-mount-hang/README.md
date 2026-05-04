# direnv hangs on rclone FUSE mount in .envrc

- **Date**: 2026-05-04

## Problem

Opening a project directory with direnv enabled caused the shell to hang indefinitely with the warning:

```
direnv: ([/usr/bin/direnv export zsh]) is taking a while to execute. Use CTRL-C to give up.
```

The `.envrc` attempted to mount a Google Drive remote via `rclone mount` in the background so the project's `Shared_Dir/` would be available on entry.

## Environment

| Package | Version |
|---------|---------|
| direnv  | system  |
| rclone  | system  |
| fuse3   | system  |
| kernel  | 6.18.23-1-lts |
| shell   | zsh 5.9 |

Multiple project directories each have their own `Shared_Dir/` FUSE mount via the same `MyDrive:` rclone remote with different `--drive-root-folder-id` values.

> **Note:** `MyDrive:` is a placeholder remote name. Replace with your own rclone remote. Folder IDs shown as `<folder-id>` — substitute your real Google Drive folder IDs.

## Investigation

1. Ran `rclone mount` manually without `--daemon` → blocks foreground (expected, not a bug).
2. Confirmed mount works: `ls ./Shared_Dir` showed files.
3. Added `--daemon` to `.envrc` → direnv still hung.
4. Replaced `--daemon` with `nohup ... &>/dev/null & disown` → still hung.
5. Moved entire block into subshell `() &>/dev/null & disown` → still hung.
6. Replaced `mountpoint -q` check (which stat's the directory) with `grep -qF "$PWD/Shared_Dir" /proc/mounts` → avoids stat on potentially stale FUSE mount, but still hung.
7. Searched upstream: [direnv#755](https://github.com/direnv/direnv/issues/755), [direnv#503](https://github.com/direnv/direnv/issues/503) — confirmed known unresolved bug.

## Root Cause

direnv sources `.envrc` in a bash subprocess and waits for **all file descriptors inherited by child processes** to close before exiting. FUSE mount processes (rclone, encfs, etc.) inherit open FDs from the parent shell session. Even with `&` and `disown`, the inherited FDs keep direnv's pipe open, so it never sees EOF and hangs indefinitely. The `--daemon` flag doesn't help because rclone still initialises (auth, network) before forking, and the fork inherits the same FDs.

## Solution

Use `setsid` to launch rclone in a new session (detaching from direnv's process group) and explicitly close stdin/stdout/stderr before exec:

```bash
if ! grep -qF "$PWD/Shared_Dir" /proc/mounts; then
  fusermount -uz "$PWD/Shared_Dir" 2>/dev/null || true
  setsid rclone mount MyDrive: "$PWD/Shared_Dir" \
    --drive-root-folder-id <folder-id> \
    </dev/null &>/dev/null &
  disown
fi
```

Key points:
- `grep -qF "$PWD/Shared_Dir" /proc/mounts` — checks mount without stat'ing the directory (avoids hang on stale FUSE mount).
- `setsid` — creates a new process session; rclone is no longer in direnv's process group.
- `</dev/null &>/dev/null` — closes all inherited FDs (stdin, stdout, stderr).
- `& disown` — shell doesn't wait; job removed from job table.

## Prevention

Always use `setsid` + FD closure when launching FUSE daemons from direnv. Never rely on `--daemon` alone or `& disown` without closing FDs — FUSE processes will still inherit open file descriptors from the parent.

Use `/proc/mounts` (or `grep /proc/self/mountinfo`) instead of `mountpoint`/`findmnt` for idempotency checks — avoids blocking stat on unresponsive mounts.

## References

- [direnv#755 — hangs when subprocess doesn't exit](https://github.com/direnv/direnv/issues/755)
- [direnv#503 — option to load .envrc in background](https://github.com/direnv/direnv/issues/503)
- [rclone mount docs](https://rclone.org/commands/rclone_mount/)
- `man setsid`, `man fusermount`
