# Troubleshooting Knowledge Base

Academic documentation of Linux/system software problems and solutions.

## Machine

| Field | Value |
|-------|-------|
| Host | ThinkPad P14s Gen 6 |
| OS | EndeavourOS x86_64 |
| Kernel | Linux 6.18.23-1-lts |
| CPU | Intel Core Ultra 7 255H — 16 cores @ 5.10 GHz |
| GPU | Intel Graphics (integrated) |
| RAM | ~31 GiB |
| Disk | ext4 on / |
| Display 1 | 3072x1920 @ 1.5x — 14" built-in |
| Display 2 | 1920x1080 @ 60 Hz — HDMI external |
| WM | i3 4.25.1 (X11) |
| Shell | zsh 5.9 |
| Terminal | WezTerm — JetBrains Mono |
| Packages | pacman (~1455) |
| Theme | Arc-Dark / Qogir-Dark icons |
| Locale | en_GB.UTF-8 |

## Directory Structure

Each problem gets its own directory named descriptively:

```
troubleshooting/
├── CLAUDE.md
└── <problem-slug>/
    ├── README.md       # problem statement, analysis, solution, references
    ├── logs/           # raw logs, journal exports, dmesg output
    ├── configs/        # relevant config files or diffs
    └── scripts/        # any scripts written to diagnose or fix
```

## Privacy Rules (for public repo safety)

Before committing any file, verify:

- **No real credentials**: No OAuth tokens, API keys, passwords, session cookies
- **No internal remote names**: Use generic placeholders (`MyDrive:`, `myremote:`) not org-specific names
- **No real folder/resource IDs**: Replace Drive folder IDs, bucket names, project IDs with `<placeholder>`
- **No hostnames or internal URLs**: Replace with `hostname.example.com`
- **No absolute home paths**: Use `~` or `$HOME`, never `/home/<username>/...`
- **No EDID blocks**: Strip `xrandr --verbose` EDID hex before committing; use non-verbose output only
- **No journal dumps with PII**: Scrub usernames, emails, internal service names from journal exports before committing to `logs/`
- **Machine table**: Keep generic (model name OK, no MTM codes or serial numbers)
- **Kernel versions**: Fine to include; update when kernel changes

## README.md Template (per solution)

Each `README.md` should document:

- **Date**: when the problem occurred
- **Problem**: exact symptoms and context
- **Environment**: relevant package versions, kernel, hardware state
- **Investigation**: commands run, logs examined, hypotheses tested
- **Root Cause**: what actually caused it
- **Solution**: exact steps, commands, config changes applied
- **Prevention**: how to avoid recurrence
- **References**: man pages, bug trackers, forums, upstream issues
