---
name: vnc-service:setup
description: >-
  Set up a persistent virtual display + VNC server on a headless Linux server. Installs Xvfb, x11vnc,
  creates systemd user services, sets a VNC password, and configures display :99. Requires Debian/Ubuntu
  (uses apt-get). Use this skill when a tool needs browser interaction on a headless server — OAuth login,
  CAPTCHA solving, 2FA prompts. Trigger on: "vnc setup", "set up vnc", "headless browser", "no display",
  "DISPLAY not set", "CAPTCHA requires browser", "OAuth needs browser", or when any other skill calls
  /vnc-service:setup as a prerequisite.
---

# VNC Service Setup

Install and configure a persistent virtual display (Xvfb) + VNC server (x11vnc) on a headless
Linux server. This enables browser-based interactions (OAuth, CAPTCHA, 2FA) that normally require
a physical monitor.

## Requirements

- **Debian/Ubuntu-based Linux** (uses apt-get for package installation)
- **systemd** with user session support
- **sudo** access for package installation (or pre-installed Xvfb + x11vnc)

## What This Does

1. Detects if a display is already available (skip if graphical desktop)
2. Checks sudo access and installs Xvfb + x11vnc (if not present)
3. Checks for port 5999 conflicts
4. Creates a VNC password (stored in `~/.vnc/passwd`)
5. Creates systemd user services for Xvfb on `:99` + x11vnc on port `5999`
6. Enables services to start on boot (with lingering)
7. Verifies everything works

After setup, any process that needs a browser sets `DISPLAY=:99` and renders on the virtual
display. The user connects via VNC (over SSH tunnel) to interact.

## Workflow

### Step 1: Environment Detection

Check if a display is already available:

```bash
if [ -n "$DISPLAY" ] && xdpyinfo >/dev/null 2>&1; then
  echo "Graphical display detected — VNC service not needed"
  exit 0
fi
```

If a display exists, inform the user and skip setup — they can use their existing display
for browser interactions.

If no display: proceed with setup.

### Step 2: Check Sudo and Install Dependencies

**First check sudo availability:**

```bash
if ! command -v sudo >/dev/null 2>&1 || ! sudo -n true 2>/dev/null; then
  echo "sudo is not available or requires a password."
  echo "Please install these packages manually, then re-run setup:"
  echo "  apt-get install -y xvfb x11vnc"
  # Wait for user to confirm they've installed the packages
fi
```

**Then install missing packages:**

```bash
which Xvfb >/dev/null 2>&1 || sudo apt-get install -y xvfb
which x11vnc >/dev/null 2>&1 || sudo apt-get install -y x11vnc
```

**Verify both are now available:**

```bash
which Xvfb >/dev/null 2>&1 && which x11vnc >/dev/null 2>&1 || {
  echo "ERROR: Xvfb or x11vnc not available after install attempt."
  echo "On non-Debian systems, install the equivalent packages:"
  echo "  Fedora/RHEL: dnf install xorg-x11-server-Xvfb x11vnc"
  echo "  Arch: pacman -S xorg-server-xvfb x11vnc"
  exit 1
}
```

### Step 3: Check Port Availability

```bash
if ss -tlnp | grep -q ':5999 '; then
  echo "WARNING: Port 5999 is already in use:"
  ss -tlnp | grep ':5999 '
  echo ""
  echo "Options:"
  echo "  1. Kill the process using port 5999"
  echo "  2. Choose a different VNC port"
  echo "  3. Cancel setup"
  # Wait for user decision
fi
```

### Step 4: VNC Password

Check if `~/.vnc/passwd` exists. If so, ask before overwriting:

```bash
mkdir -p ~/.vnc
if [ -f ~/.vnc/passwd ]; then
  echo "VNC password already exists at ~/.vnc/passwd"
  echo "Keep existing password or generate a new one? (keep/new)"
  # If keep: skip password creation
  # If new: generate below
fi

# Generate new password
VNC_PASS=$(python3 -c "import secrets; print(secrets.token_urlsafe(12))")
x11vnc -storepasswd "$VNC_PASS" ~/.vnc/passwd
chmod 600 ~/.vnc/passwd
```

Display the password to the user:
```
VNC Password: <generated password>
Save this — you'll need it to connect from your PC.
```

### Step 5: Create Systemd User Services

**Check if service files already exist (idempotency guard):**

```bash
mkdir -p ~/.config/systemd/user

if [ -f ~/.config/systemd/user/virtual-display.service ]; then
  echo "Service files already exist."
  echo "  1. Overwrite with fresh config (resets any customizations)"
  echo "  2. Keep existing service files"
  # If keep: skip to Step 6
  # If overwrite: proceed
fi
```

**Create virtual-display.service:**

```ini
[Unit]
Description=Persistent Virtual Display (Xvfb)
After=network.target

[Service]
Type=simple
ExecStartPre=/bin/bash -c '\
  if [ -f /tmp/.X99-lock ]; then \
    pid=$(cat /tmp/.X99-lock 2>/dev/null | tr -d " "); \
    if [ -n "$pid" ] && kill -0 "$pid" 2>/dev/null; then \
      echo "Display :99 owned by live PID $pid"; exit 1; \
    else \
      rm -f /tmp/.X99-lock; \
    fi; \
  fi'
ExecStart=/usr/bin/Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

**Create vnc-server.service** (with display readiness check):

```ini
[Unit]
Description=VNC Server on Virtual Display :99
After=virtual-display.service
Requires=virtual-display.service

[Service]
Type=simple
Environment=DISPLAY=:99
ExecStartPre=/bin/bash -c '\
  for i in $(seq 1 30); do \
    DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && exit 0; \
    sleep 0.5; \
  done; \
  echo "Timed out waiting for display :99"; exit 1'
ExecStart=/usr/bin/x11vnc -display :99 -rfbport 5999 -forever -shared -rfbauth %h/.vnc/passwd -o %h/.vnc/x11vnc.log
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

**Key design decisions:**
- Two separate services so they can be managed independently
- vnc-server has an `ExecStartPre` that waits up to 15 seconds for display :99 to be ready
  (fixes the race condition where x11vnc starts before Xvfb is initialized)
- The lock file removal checks if the PID is alive before deleting
  (prevents killing another user's Xvfb instance)
- VNC binds to all interfaces — access is restricted by ufw firewall rules (Step 6b)
- `Environment=DISPLAY=:99` ensures child processes inherit the display

### Step 6: Enable and Start

```bash
systemctl --user daemon-reload
systemctl --user enable virtual-display.service vnc-server.service
systemctl --user start virtual-display.service
```

Wait for display :99 to be ready before starting VNC:
```bash
for i in $(seq 1 30); do
  DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && break
  sleep 0.5
done
systemctl --user start vnc-server.service
```

**Lingering** — for services to run without an active login session and survive reboots:
```bash
if ! loginctl enable-linger $(whoami) 2>/dev/null; then
  echo "WARNING: Could not enable lingering. Services will stop when you log out."
  echo "This may happen in containers or WSL. Services will need manual restart after reboot."
fi
```

### Step 6b: Configure Firewall (ufw)

Restrict VNC port to the local network only:

```bash
if command -v ufw >/dev/null 2>&1; then
  # Auto-detect LAN subnet
  LAN_CIDR=$(ip -4 addr show | grep 'inet ' | grep -v '127.0.0.1' | head -1 | awk '{print $2}' | sed 's|\.[0-9]*/|.0/|')

  echo "Detected LAN: $LAN_CIDR"
  echo "Configuring ufw to allow VNC (port 5999) from local network only..."

  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  sudo ufw allow from "$LAN_CIDR" to any port 5999 proto tcp
  sudo ufw allow ssh
  sudo ufw --force enable

  echo "Firewall configured: port 5999 allowed from $LAN_CIDR only"
else
  echo "WARNING: ufw not available. Port 5999 is accessible from any network."
  echo "Install ufw (sudo apt install ufw) or configure your firewall manually."
fi
```

### Step 7: Verify

```bash
# Check Xvfb is running on :99
DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && echo "Display :99 OK" || echo "Display :99 FAILED"

# Check VNC is listening on port 5999
ss -tlnp | grep 5999 && echo "VNC port 5999 OK" || echo "VNC port 5999 FAILED"

# Check services
systemctl --user is-active virtual-display.service
systemctl --user is-active vnc-server.service
```

### Step 8: Report

Determine the server's IP address for the SSH tunnel command:
```bash
SERVER_IP=$(hostname -I | awk '{print $1}')
```

```
VNC Service Setup Complete
  Display:  :99
  VNC Port: 5999
  Password: <shown above>
  Firewall: port 5999 allowed from {LAN_CIDR} only (or "ufw not available" warning)
  Service:  virtual-display.service + vnc-server.service (enabled, starts on boot)

To connect: Use any VNC client → <server-ip>:5999 (from local network)
To use:     Set DISPLAY=:99 before running browser commands
To check:   /vnc-service:status
To stop:    /vnc-service:stop
```

## Error Handling

- **sudo not available:** List the `apt-get install` commands for the user to run manually.
  Also show equivalents for dnf (Fedora/RHEL) and pacman (Arch).
- **systemd user services not supported:** Fall back to manual Xvfb + x11vnc startup instructions
- **Port 5999 already in use:** Show what's using it, offer to kill or pick a different port
- **Xvfb display :99 already locked:** Check if the locking PID is alive. If dead, remove lock.
  If alive, inform user another process owns the display.
- **loginctl enable-linger fails:** Warn that services won't survive logout/reboot.
  Common in containers and WSL.
- **Service files already exist:** Ask before overwriting to protect customizations.

## Security Notes

- VNC port 5999 is **restricted to the local network** via ufw firewall rules
- If ufw is unavailable, VNC binds to all interfaces with password auth only — consider
  adding firewall rules manually or using SSH tunnel: `ssh -L 5999:localhost:5999 user@server`
- VNC password is stored in `~/.vnc/passwd` (VNC's obfuscated format, chmod 600)
- The virtual display runs as the current user, not root
- The Unix socket at `/tmp/.X11-unix/X99` is accessible to local users on the same machine.
  On single-user systems this is fine. For multi-user servers, consider adding `-auth` with
  an Xauthority file for isolation.
