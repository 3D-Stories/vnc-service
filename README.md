# VNC Service Plugin

Persistent virtual display + VNC server for headless Linux servers. Enables browser-based auth flows (OAuth, CAPTCHA, 2FA) on machines without a physical monitor.

## The Problem

Many developer tools need browser interaction that's impossible on headless servers:
- **NotebookLM** needs Google OAuth via browser (`notebooklm login`)
- **Google AI Mode** needs CAPTCHA solving on first use
- **GitHub OAuth** sometimes requires browser confirmation
- **Any MCP server** with browser-based auth

Without a display, these tools either fail silently or crash with "No X-Server detected."

## The Solution

This plugin installs a persistent virtual display (Xvfb) + VNC server (x11vnc) as systemd services:

```
Xvfb on :99 (virtual display, 1280x1024)
  └── x11vnc on port 5999 (VNC access with password)
      └── Any browser launched with DISPLAY=:99 renders here
          └── User connects via VNC client to interact
```

Once set up, it starts on boot and runs permanently. Any tool that needs a browser just sets `DISPLAY=:99`.

## Quick Start

```bash
# In Claude Code:
/vnc-service:setup    # One-time install (asks for VNC password)
/vnc-service:run      # Ensure running + print connection info
/vnc-service:status   # Check health
```

## Commands

### `/vnc-service:setup`
One-time setup:
1. Detects environment (skips if graphical desktop)
2. Installs Xvfb + x11vnc
3. Creates VNC password
4. Creates systemd user services
5. Enables auto-start on boot
6. Verifies everything works

### `/vnc-service:run`
Before any browser operation:
1. Checks if setup has been done
2. Starts services if stopped
3. Verifies display + VNC
4. Prints connection info
5. Waits for user to confirm VNC connection

### `/vnc-service:status`
Diagnostics:
- Service status (active/inactive/not-installed)
- Process IDs
- Display :99 health
- Port 5999 listening
- Connection info

## For Other Plugin Authors

If your plugin needs browser interaction, declare `/vnc-service:setup` as a prerequisite:

```markdown
### Prerequisite: Virtual Display (headless only)
If no display detected, call /vnc-service:setup then /vnc-service:run before browser operations.
```

## How It Works

```
┌──────────────────────────────────────────┐
│  Your PC (VNC Client)                    │
│  Connect to: server-ip:5999              │
└──────────┬───────────────────────────────┘
           │ VNC Protocol
┌──────────▼───────────────────────────────┐
│  Headless Server                         │
│  ┌─────────────────────────────────────┐ │
│  │  x11vnc (port 5999, password auth)  │ │
│  └──────────┬──────────────────────────┘ │
│             │ X11 Protocol               │
│  ┌──────────▼──────────────────────────┐ │
│  │  Xvfb :99 (virtual display)         │ │
│  │  ┌───────────────────────────────┐  │ │
│  │  │  Chrome / Playwright browser  │  │ │
│  │  │  (OAuth, CAPTCHA, 2FA)        │  │ │
│  │  └───────────────────────────────┘  │ │
│  └─────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

## Security

- VNC password stored in `~/.vnc/passwd` (obfuscated, chmod 600)
- VNC listens on `0.0.0.0:5999` (LAN accessible)
- For tighter security: use SSH tunnel (`ssh -L 5999:localhost:5999 server`)
- Services run as the current user, not root
- `loginctl enable-linger` required for services to persist without login session

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "No X-Server detected" | Run `/vnc-service:run` to start the display |
| VNC connection refused | Check: `systemctl --user status vnc-server.service` |
| Display :99 locked | `rm -f /tmp/.X99-lock` then restart service |
| Services don't survive reboot | Run `loginctl enable-linger $(whoami)` |
| Multiple displays (:99, :100...) | Kill stale Xvfb processes, restart service |
| Black screen in VNC | Correct display — run `/vnc-service:status` to verify |

## Version History

- **0.1.0** — Initial release. Xvfb + x11vnc systemd services, setup/run/status skills.
