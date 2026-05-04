# HDMI xrandr "cannot find mode" failure on disconnected display

- **Date**: 2026-05-03
- **Status**: Temporarily resolved — Option C (autorandr) pending

---

## Problem

HDMI external display physically disconnected. Running xrandr (likely via i3 config or autostart script) fails with **exit code 1** and error:

```
xrandr: cannot find mode none/n
```

Internal display continues working. Error appears on login or i3 reload.

## Environment

| Field | Value |
|-------|-------|
| Kernel | 6.18.23-1-lts |
| X server | X11 (i3 4.25.1) |
| GPU driver | Intel i915 / modesetting (integrated) |
| Display 1 | eDP — 3072x1920 14" built-in |
| Display 2 | HDMI-1 — 1920x1080 external (disconnected) |

---

## Investigation

### Phase 1 — Gather Evidence

#### 1.1 Capture current xrandr state

```bash
xrandr --query
xrandr --listmonitors
xrandr --listactivemonitors
```

Look for:
- Exact output names (may be `eDP-1`, `eDP1`, `HDMI-1`, `HDMI-A-1`, `HDMI-2` — driver version affects naming)
- HDMI line should read `disconnected` with no modes listed
- Confirm eDP panel mode name (usually `3072x1920` but may differ)

Save output:
```bash
xrandr --query > logs/xrandr-query.txt
```

#### 1.2 Capture verbose xrandr (full mode database)

```bash
xrandr --verbose 2>&1 > logs/xrandr-verbose.txt
grep -nE "^[A-Za-z0-9_-]+ (connected|disconnected)" logs/xrandr-verbose.txt
grep -nE "Modeline|^\s+[0-9]{3,}x[0-9]{3,}" logs/xrandr-verbose.txt | head -40
```

Look for:
- Custom modelines with non-standard names (e.g. `1920x1080_60.00`) added by an earlier `--newmode`/`--addmode`
- HDMI output stuck with leftover attached modes from a previous session

#### 1.3 Reproduce the failure

```bash
grep -nE "exec|exec_always" ~/.config/i3/config
```

Run each exec line touching xrandr manually in a terminal to identify which one fails.

#### 1.4 Pull errors from logs

```bash
tail -100 ~/.xsession-errors 2>/dev/null > logs/xsession-errors.txt
journalctl --user -b 0 --no-pager | grep -iE "xrandr|mode|i3" | tail -60 > logs/journal-user.txt
journalctl -b 0 --no-pager | grep -iE "i915|drm|HDMI" | tail -40 > logs/journal-drm.txt
```

Look for: full `xrandr: cannot find mode <name>` line. The line before it in `.xsession-errors` usually names the calling script.

---

### Phase 2 — Locate the Offending Call

#### 2.1 Grep all startup locations

```bash
grep -rnE "xrandr|--output|--mode|--newmode|--addmode" \
  ~/.config/i3/ \
  ~/.config/autostart/ \
  ~/.xprofile \
  ~/.xinitrc \
  ~/.xsession \
  ~/.bash_profile \
  ~/.zprofile \
  ~/.profile \
  /etc/xdg/autostart/ \
  /etc/X11/xinit/xinitrc.d/ \
  2>/dev/null
```

Look for:
- `xrandr --output HDMI-1 --mode <something>` without a `connected` guard
- Lines sourcing an arandr-saved layout (e.g. `~/.screenlayout/dual.sh`)
- `exec_always` (re-fires on every i3 reload) vs `exec` (login only)

Save output:
```bash
# (run above grep, pipe to configs/xrandr-calls.txt)
```

#### 2.2 Inspect arandr saved layouts

```bash
ls -la ~/.screenlayout/ 2>/dev/null
for f in ~/.screenlayout/*.sh; do echo "===== $f ====="; cat "$f"; done
```

Look for: `--output HDMI-1 --mode 1920x1080` hardcoded without connection guard. Copy offending scripts to `configs/`.

#### 2.3 Check for autorandr

```bash
command -v autorandr && autorandr --version
ls -la ~/.config/autorandr/ 2>/dev/null
autorandr --current 2>&1
```

#### 2.4 Confirm output name drift

Kernel/driver updates can rename outputs (`HDMI-1` → `HDMI-A-1`):

```bash
ls /sys/class/drm/ | grep -i hdmi
```

#### 2.5 Run offending script with tracing

```bash
bash -x ~/.screenlayout/dual.sh 2>&1 | tail -20
```

First line printing `cannot find mode` is the culprit. Note the exact mode name in the error.

---

### Phase 3 — Diagnose Root Cause

Map evidence to one category:

| Category | Symptom | Confirmation |
|----------|---------|--------------|
| **3a** Hardcoded mode against disconnected output (most likely) | Error names real mode like `1920x1080` | `xrandr --query \| awk '/^HDMI/ {print $1, $2}'` shows `disconnected` |
| **3b** Empty variable expansion | Error says `none` or single char | Script has `--mode "$MODE"` where `$MODE` unset |
| **3c** Stale custom modeline | Error names non-standard mode like `1920x1080_60.00` | Mode missing from `xrandr --verbose` output |
| **3d** Output name drift | Cascading errors after `cannot find output` | `ls /sys/class/drm/` shows different name than script uses |

---

## Root Cause

xrandr state had HDMI-1 still active (not `--off`) despite monitor being physically disconnected. Any call attempting to set a mode on HDMI-1 fails immediately because a disconnected output has no mode list.

**Immediate fix**: ran `xrandr --output HDMI-1 --off` once in terminal. No scripts modified. Fix is session-scoped — will recur on next login until Option C (autorandr) is configured.

---

## Solution

**Applied fix**: `xrandr --output HDMI-1 --off` — one-off terminal command, session-scoped only. No scripts written or modified.

**Permanent fix**: Option C below (autorandr). Not yet applied.

---

Choose the option matching the diagnosis. Option C (autorandr) is recommended for a laptop with frequent dock/undock cycles.

### Option A — Inline guard in failing script (minimal change)

```bash
#!/bin/sh
# Replace body of ~/.screenlayout/dual.sh (or equivalent) with:
if xrandr --query | grep -q "^HDMI-1 connected"; then
    xrandr --output eDP-1 --mode 3072x1920 --pos 0x0 --rotate normal \
           --output HDMI-1 --mode 1920x1080 --pos 3072x0 --rotate normal
else
    xrandr --output eDP-1 --mode 3072x1920 --pos 0x0 --rotate normal \
           --output HDMI-1 --off
fi
```

Substitute actual output names from Step 1.1.

### Option B — Two scripts + i3 key binding (manual switching)

Keep `~/.screenlayout/dual.sh` for docked. Create `~/.screenlayout/single.sh` for undocked. In `~/.config/i3/config`:

```
exec_always --no-startup-id ~/.screenlayout/single.sh
bindsym $mod+Shift+d exec --no-startup-id ~/.screenlayout/dual.sh
bindsym $mod+Shift+s exec --no-startup-id ~/.screenlayout/single.sh
```

### Option C — autorandr with udev hotplug (recommended)

```bash
# Install
sudo pacman -S autorandr

# With HDMI unplugged, save undocked profile
autorandr --save mobile

# Plug HDMI in, arrange with arandr, then save
autorandr --save docked

# Set undocked as fallback when no fingerprint matches
autorandr --default mobile

# Enable hotplug via systemd user unit
systemctl --user enable --now autorandr.service
```

In `~/.config/i3/config`, replace any xrandr/screenlayout exec with:

```
exec_always --no-startup-id autorandr --change --default mobile
```

`--change` picks matching saved profile by EDID fingerprint, falls back to `mobile` when nothing matches.

### Option D — Generic safe-output wrapper

For scripts that can't be replaced:

```bash
xrandr_safe_output() {
    out="$1"; shift
    if xrandr --query | grep -q "^${out} connected"; then
        xrandr --output "$out" "$@"
    else
        xrandr --output "$out" --off 2>/dev/null || true
    fi
}
xrandr_safe_output HDMI-1 --mode 1920x1080 --right-of eDP-1
```

### Option E — Prevent exec_always re-fire

Use `exec` (not `exec_always`) for layout scripts. `exec_always` re-fires on every `i3 reload`, re-triggering the failure. Reserve `exec_always` for status bars and keybinding daemons.

---

## Verification

```bash
# 1. Reload i3 with HDMI unplugged — check for errors
i3-msg reload
tail -30 ~/.xsession-errors

# 2. Check exit status of layout command directly
~/.screenlayout/single.sh; echo "exit=$?"
# or
autorandr --change; echo "exit=$?"

# 3. (Option C) Verify hotplug — plug HDMI in, watch journal
journalctl --user -f -u autorandr.service
# expect: "applying profile docked"
```

Success: exit code 0, no `cannot find mode` in `~/.xsession-errors`, both layouts work across reload and hotplug cycles.

---

## Prevention

- Never hardcode `--mode` against an output without first checking `xrandr | grep "^OUT connected"`
- Prefer `autorandr --change` over saved arandr scripts — fingerprint-matched, hotplug-aware, idempotent
- Use `exec` not `exec_always` for one-shot display config in i3
- After kernel upgrades, verify output names via `ls /sys/class/drm/` — naming can change
- Keep a `mobile` undocked profile as `--default` so fallback is always safe

---

## References

- `man xrandr`
- `man autorandr`
- [autorandr on GitHub](https://github.com/phillipberndt/autorandr)
- [ArchWiki — xrandr](https://wiki.archlinux.org/title/xrandr)
- [ArchWiki — autorandr](https://wiki.archlinux.org/title/autorandr)
- [ArchWiki — multihead](https://wiki.archlinux.org/title/Multihead)
