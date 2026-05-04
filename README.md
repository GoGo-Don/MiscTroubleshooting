# MiscTroubleshooting

Personal knowledge base of Linux and system software problems — documented for future reference and to help others hitting the same issues.

## Machine

| Field | Value |
|-------|-------|
| Host | ThinkPad P14s Gen 6 |
| OS | EndeavourOS x86_64 |
| Kernel | Linux 6.x LTS |
| CPU | Intel Core Ultra 7 255H |
| WM | i3 (X11) |
| Shell | zsh |

## Structure

Each problem lives in its own directory:

```
<problem-slug>/
├── README.md       # problem statement, investigation, root cause, solution
├── logs/           # raw logs, journal exports, dmesg output
├── configs/        # relevant config files or diffs
└── scripts/        # diagnostic or fix scripts
```

## Entries

| Problem | Summary |
|---------|---------|
| [direnv-rclone-mount-hang](direnv-rclone-mount-hang/README.md) | direnv hangs indefinitely when `.envrc` launches a rclone FUSE mount |
| [hdmi-xrandr-mode-not-found](hdmi-xrandr-mode-not-found/README.md) | xrandr reports `mode not found` when adding a custom modeline for HDMI output |

## Notes for readers

- Remote names (e.g. `MyDrive:`) and resource IDs (e.g. `<your-drive-folder-id>`) are placeholders — substitute your own.
- Absolute paths use `~` throughout; no real usernames are present.
- Configs are stripped of credentials before committing.
