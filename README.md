# VNC Service Plugin

Persistent virtual display + VNC server for headless Linux servers. Enables browser-based
auth flows (OAuth, CAPTCHA, 2FA) on machines without a physical monitor.

## Requirements

- **Debian/Ubuntu-based Linux** (uses apt-get for package installation)
- **systemd** with user session support
- **sudo** access for package installation (or pre-installed Xvfb + x11vnc)

## The Problem

Many developer tools need browser interaction that is impossible on headless servers:
- **NotebookLM** needs Google OAuth via browser (`notebooklm login`)
- **Google AI Mode** needs CAPTCHA solving on first use
- **GitHub OAuth** sometimes requires browser confirmation
- **Any MCP server** with browser-based auth

Without a display, these tools either fail silently or crash with "No X-Server detected."

## The Solution

This plugin installs a persistent virtual display (Xvfb) + VNC server (x11vnc) as systemd
user services:

```
Xvfb on :99 (virtual display, 1280x1024)
  └── x11vnc on port 5999 (all interfaces, password auth)
      └── Access restricted by ufw firewall (LAN only)
          └── VNC client connects from local network
              └── Any browser with DISPLAY=:99 renders here
```

Once set up, it starts on boot and runs permanently. Any tool that needs a browser just
sets `DISPLAY=:99`.

## Installation

Remove any stale marketplace entry, add fresh, install, and reload:

```bash
claude plugin marketplace remove vnc-service
claude plugin marketplace add 3D-Stories/vnc-service
claude plugin install vnc-service@vnc-service
```

Then reload plugins in your session:

```
/reload-plugins
```

## Commands

### `/vnc-service:setup`

One-time install and configuration. Idempotent -- asks before overwriting existing service
files or VNC passwords.

1. **Environment detection** -- skips if graphical desktop is detected
2. **Sudo check** -- if sudo is unavailable or requires a password, lists the `apt-get install`
   commands for the user to run manually
3. **Install dependencies** -- installs Xvfb + x11vnc via apt-get (skips already-installed)
4. **Port conflict detection** -- checks if port 5999 is already in use, offers alternatives
5. **VNC password** -- generates and stores in `~/.vnc/passwd` (chmod 600). Asks before
   overwriting an existing password.
6. **Systemd user services** -- creates `virtual-display.service` and `vnc-server.service`.
   Asks before overwriting existing service files (idempotency guard).
   - **Race condition fix:** vnc-server has an `ExecStartPre` that waits up to 15 seconds
     for display :99 to become ready before x11vnc starts. This prevents the race where
     x11vnc launches before Xvfb has initialized.
   - **VNC binds to all interfaces** (`0.0.0.0`), not localhost. Access is restricted by ufw
     firewall rules (Step 6b), not by binding address.
6. **Enable and start** -- enables auto-start on boot. Handles `loginctl enable-linger` for
   services to persist without an active login session. Warns if lingering fails (common in
   containers and WSL).
6. **Firewall (ufw)** -- auto-detects the LAN CIDR from the host's network interfaces and
   restricts port 5999 to the local subnet only:
   ```bash
   LAN_CIDR=$(ip -4 addr show | grep 'inet ' | grep -v '127.0.0.1' | head -1 | awk '{print $2}' | sed 's|\.[0-9]*/|.0/|')
   sudo ufw allow from "$LAN_CIDR" to any port 5999 proto tcp
   ```
   If ufw is not available, warns that port 5999 is accessible from any network and suggests
   installing ufw or configuring the firewall manually.
7. **Verify** -- checks display :99 health, port 5999 listening, service status
8. **Report** -- prints connection info with server IP

### `/vnc-service:run`

Ensure running and print connection info. Call before any browser operation.

1. Checks if setup has been done (directs to `/vnc-service:setup` if not)
2. Starts display service, waits for :99 to be ready (up to 15 seconds)
3. Starts VNC service
4. Verifies display + VNC port
5. Prints connection info
6. Waits for user to confirm VNC connection

### `/vnc-service:stop`

Stop services to free resources.

1. Stops VNC server and virtual display
2. Optionally disables auto-start on boot (asks the user)
3. Services can be restarted anytime with `/vnc-service:run`
4. Browser profiles and CAPTCHA cookies are preserved in the filesystem

### `/vnc-service:status`

Health check and diagnostics.

- Service status (active/inactive/not-installed) with PIDs
- Display :99 health check
- Port 5999 listening check
- Auto-start and lingering status
- Connection info (server IP)
- Specific fix commands for any issues found

## For Other Skill Authors

If your skill needs browser interaction on a headless server, declare vnc-service as a
prerequisite:

```markdown
### Prerequisite: Virtual Display (headless only)
If no display detected:
1. Call /vnc-service:setup to install the virtual display service
2. Call /vnc-service:run before browser operations
3. Wait for user to confirm VNC connection
4. Set DISPLAY=:99 for browser commands
```

Skills auto-install dependencies via the `claude` CLI in Bash. The user only needs to run
`/reload-plugins` when prompted.

## How It Works

```
┌──────────────────────────────────────────┐
│  Your PC                                 │
│  VNC client → <server-ip>:5999           │
│  (from local network only, via ufw)      │
└──────────┬───────────────────────────────┘
           │ TCP (LAN restricted by ufw)
┌──────────▼───────────────────────────────┐
│  Headless Server                         │
│  ┌─────────────────────────────────────┐ │
│  │  x11vnc (0.0.0.0:5999, password)   │ │
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

If ufw is not available, use an SSH tunnel instead:
`ssh -L 5999:localhost:5999 user@server`, then connect VNC client to `localhost:5999`.

## Security

- VNC port 5999 is **restricted to the local subnet** via ufw firewall rules (auto-detected
  LAN CIDR). VNC binds to all interfaces -- firewall provides the access control, not binding.
- If ufw is unavailable, port 5999 is open to all networks with password auth only. Install ufw
  or use SSH tunnel: `ssh -L 5999:localhost:5999 user@server`
- VNC password stored in `~/.vnc/passwd` (VNC obfuscated format, chmod 600)
- Services run as the current user, not root
- `loginctl enable-linger` required for services to persist without login session
- On multi-user systems, the X11 Unix socket at `/tmp/.X11-unix/X99` is accessible to local
  users on the same machine

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "No X-Server detected" | Run `/vnc-service:run` to start the display |
| VNC connection refused | Check ufw allows your IP: `sudo ufw status` |
| Display :99 locked | `/vnc-service:status` checks if PID is alive; removes stale locks |
| Services don't survive reboot | Run `loginctl enable-linger $(whoami)` |
| Services don't survive logout | Same -- lingering must be enabled |
| Multiple displays (:99, :100...) | Kill stale Xvfb processes: `pkill -f "Xvfb :"`; restart service |
| Black screen in VNC | Check `/vnc-service:status` for display health |
| sudo not available | Install packages manually, then re-run `/vnc-service:setup` |
| Port 5999 in use | `/vnc-service:setup` detects this and offers alternatives |
| Non-Debian system | Install Xvfb + x11vnc via your package manager, then run setup |

## Version History

- **0.2.2** -- Current release.
- **0.2.x** -- Incremental fixes and refinements.
- **0.2.0** -- Security: VNC binds to all interfaces with ufw firewall restricting port 5999
  to local subnet (auto-detected LAN CIDR). Fixes: race condition (vnc-server ExecStartPre
  waits for display readiness), idempotent setup (asks before overwriting service files and
  VNC password), sudo detection, port conflict detection, loginctl linger error handling,
  safe lock file removal. Added `/vnc-service:stop` skill. Added Debian/Ubuntu requirement.
- **0.1.0** -- Initial release. Xvfb + x11vnc systemd services, setup/run/status skills.
