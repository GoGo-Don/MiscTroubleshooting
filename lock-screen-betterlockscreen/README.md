# Lock Screen — betterlockscreen + Arc-Dark theme

**Date:** 2026-05-06

## Problem

Default i3lock via `blur-lock` script is visually bare — plain blurred screenshot, no clock, no widgets, no styling.

## Goal

Replace with `betterlockscreen` showing:
- Clock + date (native)
- Media info (playerctl snapshot at lock time)
- Battery state
- Username (greeter)
- Arc-Dark palette, JetBrains Mono font

## Environment

| Component | Details |
|-----------|---------|
| WM | i3 4.25.1 |
| Old locker | i3lock via `~/.config/i3/scripts/blur-lock` |
| New locker | betterlockscreen (AUR) |
| Trigger | xss-lock |
| Auth | PAM / pam_fprintd.so (unchanged) |
| Theme | Arc-Dark / #5294e2 accent |

## Solution

### Step 1 — Install betterlockscreen

```bash
yay -S betterlockscreen
```

`i3lock-color` installs as a dependency.

### Step 2 — Install config

```bash
cp configs/betterlockscreenrc ~/.config/betterlockscreenrc
```

### Step 3 — Install lock script

```bash
cp scripts/lock ~/.local/bin/lock
chmod +x ~/.local/bin/lock
```

### Step 4 — Install set-wallpaper script

```bash
cp scripts/set-wallpaper ~/.local/bin/set-wallpaper
chmod +x ~/.local/bin/set-wallpaper
```

Dependencies: `feh`, `fzf`, `chafa` (for image preview in picker), `betterlockscreen`.

```bash
yay -S fzf chafa
```

### Step 5 — Set up i3 wallpaper include file

Add one line to `~/.config/i3/config` (after existing `exec_always` blocks):

```
include ~/.config/i3/wallpaper.conf
```

`~/.config/i3/wallpaper.conf` is written and maintained by `set-wallpaper` — never hand-edit it. It contains a single line:

```
exec_always --no-startup-id feh --bg-fill "/path/to/current/wallpaper.jpg"
```

This persists the wallpaper across reboots and i3 reloads without touching `~/.config/i3/config` again.

### Step 6 — Cache initial wallpaper

```bash
set-wallpaper /path/to/wallpaper.jpg
```

Or use the picker (omit the path arg) — prompts for a directory, opens `fzf` with `chafa` side-by-side preview:

```bash
set-wallpaper
# Wallpaper directory: ~/Pictures/wallpapers
# → fzf opens, select image, Enter
```

Script applies wallpaper via `feh`, writes `wallpaper.conf`, and updates betterlockscreen cache in the background.

Verify cache: `ls ~/.cache/betterlockscreen/`

### Step 7 — Edit `~/.config/i3/config`

**Replace xss-lock line:**
```
# old:
exec --no-startup-id xss-lock -l ~/.config/i3/scripts/blur-lock

# new:
exec --no-startup-id xss-lock --transfer-sleep-lock -- ~/.local/bin/lock
```

**Add lock keybind** (after the focus keybinds block):
```
bindsym $mod+shift+l exec --no-startup-id ~/.local/bin/lock
```

Remove any `exec_always` line that manually calls `betterlockscreen -u` — `set-wallpaper` handles cache updates now.

### Step 8 — Reload i3

```bash
i3-msg reload
```

### Step 9 — Test

```bash
# Manual lock test
~/.local/bin/lock

# Confirm fingerprint unlocks
# Confirm password fallback (let fingerprint time out ~10s)
# Confirm keybind: Mod+Shift+L
# Confirm xss-lock triggers on idle
# Suspend and resume — confirm lock appears
# Change wallpaper and confirm lock screen updates after ~10s
```

## Root Cause

i3lock has no built-in widget/theming support. betterlockscreen wraps i3lock-color (which has extensive theming args) and pre-caches blur so locking is instant.

## Prevention

- Always change wallpaper via `set-wallpaper` — it keeps `feh`, `wallpaper.conf`, and betterlockscreen cache in sync automatically
- After betterlockscreen major version bumps, re-test lock and regenerate cache: `set-wallpaper $(cat ~/.cache/betterlockscreen/last-cached-wallpaper)`

## Security — set-wallpaper path validation

`set-wallpaper` writes the wallpaper path into `~/.config/i3/wallpaper.conf`, which is sourced by i3 config. A maliciously crafted filename (e.g. containing a newline + i3 exec directive) could inject arbitrary commands that run on next i3 reload.

Four-layer defense:

| Layer | Guard |
|-------|-------|
| `realpath --canonicalize-existing` | Resolve symlinks, confirm file exists |
| Newline / CR / NUL reject | Abort before any further processing |
| Character allowlist `^[a-zA-Z0-9/_. -]+$` | Block quotes, `$`, backticks, semicolons, etc. |
| Double-quote + escape in conf | Defense-in-depth — injected char can't form a parseable second directive |

## Notes

- `$mod+l` left as-is (focus right in hjkl layout) — lock uses `$mod+shift+l`
- Media info is a snapshot at lock time, not live-updated (i3lock grabs display exclusively)
- Auth unchanged — `pam_fprintd.so` in `/etc/pam.d/system-auth` applies to betterlockscreen/i3lock-color the same as i3lock

## References

- [betterlockscreen GitHub](https://github.com/betterlockscreen/betterlockscreen)
- [Arch Wiki — i3lock](https://wiki.archlinux.org/title/I3lock)
- [i3lock-color GitHub](https://github.com/Raymo111/i3lock-color)
