# Fingerprint Authentication via fprintd + PAM

**Date:** 2026-05-06

## Problem

Fingerprint reader (Synaptics 06cb:00f9) present on ThinkPad P14s Gen 6 but unused. Goal: enable fingerprint authentication system-wide — sudo, screen locker (xss-lock + i3lock), LightDM login, and polkit-gnome dialogs.

## Environment

| Component | Details |
|-----------|---------|
| Kernel | Linux 6.18.26-2-lts |
| Fingerprint reader | Synaptics USB 06cb:00f9 |
| Display manager | LightDM |
| Screen locker | xss-lock + blur-lock script (i3lock) |
| Polkit agent | polkit-gnome-authentication-agent-1 |
| fprintd | 1.94.5-2 |
| libfprint | 1.94.10-1 |

## Investigation

All four target auth surfaces chain through `/etc/pam.d/system-auth`:

```
sudo              → system-auth
i3lock            → /etc/pam.d/i3lock   → system-auth
LightDM           → system-login        → system-auth
polkit-gnome      → PAM                 → system-auth
```

Single edit to `system-auth` propagates fingerprint auth to all surfaces. No other PAM files need changing.

## Root Cause

fprintd not installed; `pam_fprintd.so` not referenced in PAM stack.

## Solution

### 1. Install fprintd

```bash
sudo pacman -S fprintd
```

libfprint installs as a dependency.

### 2. Enable fprintd service

```bash
sudo systemctl enable --now fprintd.service
```

### 3. Enroll fingerprint

```bash
fprintd-enroll
```

Follow prompts — swipe finger multiple times. Default finger: right index. To enroll a specific finger:

```bash
fprintd-enroll -f right-index-finger $USER
```

### 4. Edit `/etc/pam.d/system-auth`

**Before:**
```
auth       required                    pam_faillock.so      preauth
-auth      [success=2 default=ignore]  pam_systemd_home.so
auth       [success=1 default=bad]     pam_unix.so          try_first_pass nullok
auth       [default=die]               pam_faillock.so      authfail
auth       optional                    pam_permit.so
auth       required                    pam_env.so
auth       required                    pam_faillock.so      authsucc
```

**After:**
```
auth       required                    pam_faillock.so      preauth
-auth      [success=3 default=ignore]  pam_systemd_home.so
auth       sufficient                  pam_fprintd.so
auth       [success=1 default=bad]     pam_unix.so          try_first_pass nullok
auth       [default=die]               pam_faillock.so      authfail
auth       optional                    pam_permit.so
auth       required                    pam_env.so
auth       required                    pam_faillock.so      authsucc
```

**Changes:**
1. `success=2` → `success=3` on the `pam_systemd_home` line (accounts for the new fprintd line in the jump count)
2. Insert `auth sufficient pam_fprintd.so` after `pam_systemd_home`

`sufficient` means: fingerprint success → immediate auth pass; fingerprint failure or timeout → falls through to password prompt.

> **Warning:** Keep a second terminal open before testing. If PAM breaks, switch to TTY2 (Ctrl+Alt+F2) and revert both changes manually.

## Verification

Test each surface:

```bash
# sudo
sudo ls

# polkit dialog
pkexec ls /root

# screen locker — trigger via keybind or:
loginctl lock-session

# LightDM — log out and log back in
```

For each: touch sensor when prompted. Should fall back to password on timeout (~10s).

Check logs on failure:

```bash
journalctl -xe | grep -E 'pam|fprintd'
```

## Prevention

- Keep fprintd enabled: `systemctl is-enabled fprintd`
- Re-enroll after hardware changes or fprintd major version bumps
- After kernel/libfprint updates, test `fprintd-list $USER` to confirm enrollment persists

## References

- [Arch Wiki — fprint](https://wiki.archlinux.org/title/Fprint)
- [Arch Wiki — PAM](https://wiki.archlinux.org/title/PAM)
- [freedesktop.org — libfprint supported devices](https://fprint.freedesktop.org/supported-devices.html)
- `man pam_fprintd`, `man fprintd-enroll`
