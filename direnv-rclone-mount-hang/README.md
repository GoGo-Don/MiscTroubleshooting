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
| kernel  | 6.18.26-2-lts |
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
8. Applied `setsid + </dev/null &>/dev/null & disown` → mount succeeded, but terminal still hung and Ctrl-C had no effect.
9. Diagnosed: FDs 0/1/2 closed but direnv's internal pipe lives at FD 3+; rclone inherited it, never closed it, direnv waited forever.
10. Switched to `systemd-run --user --no-block` → direnv completed immediately. Confirmed working.

## Root Cause

direnv sources `.envrc` in a bash subprocess and waits for **all file descriptors inherited by child processes** to close before exiting. FUSE mount processes (rclone, encfs, etc.) inherit open FDs from the parent shell session. Even with `&`, `disown`, and `setsid`, the hang persisted because:

- `</dev/null &>/dev/null` only closes FDs 0, 1, 2.
- direnv opens an internal pipe at **FD 3** (first available above stderr) to capture environment output.
- rclone inherits FD 3, holds it open indefinitely, and direnv never sees EOF on that pipe.

The `--daemon` flag doesn't help — rclone initialises (auth, network) before forking, inheriting FDs in the process. `setsid` detaches the process group but does not close inherited FDs.

The Ctrl-C unresponsiveness indicates the bash subprocess entered an **uninterruptible sleep (D state)** while waiting on the pipe — signals cannot wake it.

## Solution

Use `systemd-run --user --no-block` to launch rclone as a transient systemd user service. systemd creates the process with a **fresh, clean file descriptor table** — no inherited FDs from the shell session reach rclone at all.

```bash
if ! grep -qF "$PWD/Shared_Dir" /proc/mounts; then
  fusermount -uz "$PWD/Shared_Dir" 2>/dev/null || true
  systemd-run --user --no-block \
    rclone mount MyDrive: "$PWD/Shared_Dir" \
    --drive-root-folder-id <folder-id>
fi
```

Key points:
- `grep -qF "$PWD/Shared_Dir" /proc/mounts` — checks mount without stat'ing the directory (avoids hang on stale FUSE mount).
- `systemd-run --user` — launches rclone as transient user unit; systemd owns the process, not the shell.
- `--no-block` — `systemd-run` registers the service and exits immediately; direnv does not wait for rclone to complete.
- No FD inheritance: systemd starts processes with a clean descriptor table, so direnv's pipe (FD 3+) never reaches rclone.
- Mount status visible via `systemctl --user status` for debugging.

### Why `setsid + FD close` was not enough

`setsid` detaches the process group (prevents terminal signals from reaching rclone). Redirecting FDs 0/1/2 to `/dev/null` only closes the standard streams. Neither action touches FD 3+. The only reliable fixes are:

1. `systemd-run` — systemd resets the FD table (recommended).
2. Explicitly close every FD above 2 before exec: `3>&- 4>&- 5>&- ...` on the `if` block (fragile; must guess the FD range).

## Prevention

Use `systemd-run --user --no-block` when launching any long-lived FUSE daemon from direnv. Do not rely on `--daemon`, `& disown`, `setsid`, or FD redirection alone — none of these close FDs above stderr, and direnv's pipe will keep the process anchored.

Use `/proc/mounts` (or `grep /proc/self/mountinfo`) instead of `mountpoint`/`findmnt` for idempotency checks — avoids blocking stat on unresponsive mounts.

## References

- [direnv#755 — hangs when subprocess doesn't exit](https://github.com/direnv/direnv/issues/755)
- [direnv#503 — option to load .envrc in background](https://github.com/direnv/direnv/issues/503)
- [rclone mount docs](https://rclone.org/commands/rclone_mount/)
- `man systemd-run`, `man fusermount`
