# VNC Service Plugin

Persistent virtual display + VNC server for headless Linux servers. Enables browser-based auth flows (OAuth, CAPTCHA, 2FA) on machines without a physical monitor.

## Requirements

- **Debian/Ubuntu-based Linux** (uses apt-get for package installation)
- **systemd** with user session support
- **sudo** access for package installation (or pre-installed Xvfb + x11vnc)

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
  └── x11vnc on port 5999 (localhost only, password auth)
      └── SSH tunnel from your PC
          └── VNC client connects to localhost:5999
              └── Any browser with DISPLAY=:99 renders here
```

Once set up, it starts on boot and runs permanently. Any tool that needs a browser just sets `DISPLAY=:99`.

## Quick Start

```bash
# In Claude Code:
/vnc-service:setup    # One-time install (installs packages, creates services, sets password)
/vnc-service:run      # Ensure running + print connection info
/vnc-service:stop     # Stop services to free resources
/vnc-service:status   # Check health and diagnostics
```

## Installation

### From GitHub

```
/plugin marketplace add 3D-Stories/vnc-service
/plugin install vnc-service@vnc-service
/reload-plugins
```

### Updating

If updating to a newer version:

```
/plugin marketplace remove vnc-service
/plugin marketplace add 3D-Stories/vnc-service
/plugin install vnc-service@vnc-service
/reload-plugins
```

## Commands

### `/vnc-service:setup`
One-time setup:
1. Detects environment (skips if graphical desktop)
2. Checks sudo access, installs Xvfb + x11vnc
3. Checks port 5999 availability
4. Creates VNC password (asks before overwriting existing)
5. Creates systemd user services (asks before overwriting existing)
6. Enables auto-start on boot (with lingering)
7. Verifies everything works

### `/vnc-service:run`
Before any browser operation:
1. Checks if setup has been done (directs to `/vnc-service:setup` if not)
2. Starts display service, waits for :99 to be ready
3. Starts VNC service
4. Verifies display + VNC
5. Prints SSH tunnel + connection info
6. Waits for user to confirm VNC connection

### `/vnc-service:stop`
Stop services to free resources:
1. Stops VNC server and virtual display
2. Optionally disables auto-start on boot
3. Services can be restarted anytime with `/vnc-service:run`
4. Browser profiles and CAPTCHA cookies are preserved

### `/vnc-service:status`
Diagnostics:
- Service status (active/inactive/not-installed) with PIDs
- Display :99 health check
- Port 5999 listening check
- Auto-start and lingering status
- Connection info
- Specific fix commands for any issues found

## For Other Skill Authors

If your skill needs browser interaction, declare `/vnc-service:setup` as a prerequisite:

```markdown
### Prerequisite: Virtual Display (headless only)
If no display detected:
1. Call /vnc-service:setup to install the virtual display service
2. Call /vnc-service:run before browser operations
3. Wait for user to confirm VNC connection
4. Set DISPLAY=:99 for browser commands
```

## How It Works

```
┌──────────────────────────────────────────┐
│  Your PC                                 │
│  1. ssh -L 5999:localhost:5999 server    │
│  2. VNC client → localhost:5999          │
└──────────┬───────────────────────────────┘
           │ SSH Tunnel (encrypted)
┌──────────▼───────────────────────────────┐
│  Headless Server                         │
│  ┌─────────────────────────────────────┐ │
│  │  x11vnc (localhost:5999, password)  │ │
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

- VNC binds to **localhost only** (`-localhost` flag) — not directly accessible from the network
- Access requires an **SSH tunnel**: `ssh -L 5999:localhost:5999 user@server`
- VNC password stored in `~/.vnc/passwd` (obfuscated, chmod 600)
- Services run as the current user, not root
- `loginctl enable-linger` required for services to persist without login session
- On multi-user systems, the X11 Unix socket at `/tmp/.X11-unix/X99` is accessible to local users

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "No X-Server detected" | Run `/vnc-service:run` to start the display |
| VNC connection refused | Set up SSH tunnel first: `ssh -L 5999:localhost:5999 server` |
| Display :99 locked | `/vnc-service:status` checks if PID is alive; removes stale locks |
| Services don't survive reboot | Run `loginctl enable-linger $(whoami)` |
| Services don't survive logout | Same — lingering must be enabled |
| Multiple displays (:99, :100...) | Kill stale Xvfb processes: `pkill -f "Xvfb :"`; restart service |
| Black screen in VNC | Check `/vnc-service:status` for display health |
| sudo not available | Install packages manually, then re-run `/vnc-service:setup` |
| Port 5999 in use | `/vnc-service:setup` detects this and offers alternatives |
| Non-Debian system | Install Xvfb + x11vnc via your package manager, then run setup |

## Version History

- **0.2.0** — Security: VNC binds localhost only (SSH tunnel required). Fixes: race condition (display readiness wait), idempotent setup (asks before overwriting), sudo detection, port conflict detection, loginctl error handling, safe lock file removal. Added `/vnc-service:stop` skill. Added Debian/Ubuntu requirement docs.
- **0.1.0** — Initial release. Xvfb + x11vnc systemd services, setup/run/status skills.
