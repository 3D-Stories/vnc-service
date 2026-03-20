---
name: vnc-service:setup
description: >-
  Set up a persistent virtual display + VNC server on a headless Linux server. Installs Xvfb, x11vnc,
  creates a systemd user service, sets a VNC password, and configures display :99. Use this skill when
  a tool needs browser interaction on a headless server — OAuth login, CAPTCHA solving, 2FA prompts.
  Trigger on: "vnc setup", "set up vnc", "headless browser", "no display", "DISPLAY not set",
  "CAPTCHA requires browser", "OAuth needs browser", or when any other skill calls /vnc-service:setup
  as a prerequisite.
---

# VNC Service Setup

Install and configure a persistent virtual display (Xvfb) + VNC server (x11vnc) on a headless
Linux server. This enables browser-based interactions (OAuth, CAPTCHA, 2FA) that normally require
a physical monitor.

## What This Does

1. Detects if a display is already available (skip if graphical desktop)
2. Installs Xvfb and x11vnc (if not present)
3. Creates a VNC password (stored in `~/.vnc/passwd`)
4. Creates a systemd user service that starts Xvfb on `:99` + x11vnc on port `5999`
5. Enables the service to start on boot
6. Verifies everything works

After setup, any process that needs a browser sets `DISPLAY=:99` and renders on the virtual
display. The user connects via VNC to interact (solve CAPTCHAs, complete OAuth, etc.).

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

### Step 2: Install Dependencies

Check and install required packages:

```bash
which Xvfb >/dev/null 2>&1 || sudo apt-get install -y xvfb
which x11vnc >/dev/null 2>&1 || sudo apt-get install -y x11vnc
```

If `sudo` is not available or fails, inform the user what to install manually.

### Step 3: VNC Password

Check if `~/.vnc/passwd` exists. If not, create one:

```bash
mkdir -p ~/.vnc
VNC_PASS=$(python3 -c "import secrets; print(secrets.token_urlsafe(12))")
x11vnc -storepasswd "$VNC_PASS" ~/.vnc/passwd
chmod 600 ~/.vnc/passwd
```

Display the password to the user:
```
VNC Password: <generated password>
Save this — you'll need it to connect from your PC.
```

If `~/.vnc/passwd` already exists, ask: "VNC password already exists. Keep it or generate a new one?"

### Step 4: Create Systemd User Service

Create the service file:

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/virtual-display.service << 'EOF'
[Unit]
Description=Persistent Virtual Display (Xvfb + VNC)
After=network.target

[Service]
Type=simple
ExecStartPre=/bin/bash -c 'rm -f /tmp/.X99-lock'
ExecStart=/usr/bin/Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

cat > ~/.config/systemd/user/vnc-server.service << 'EOF'
[Unit]
Description=VNC Server on Virtual Display :99
After=virtual-display.service
Requires=virtual-display.service

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -display :99 -rfbport 5999 -forever -shared -rfbauth %h/.vnc/passwd -o %h/.vnc/x11vnc.log
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF
```

Two separate services so they can be managed independently. The VNC server depends on the
virtual display.

### Step 5: Enable and Start

```bash
systemctl --user daemon-reload
systemctl --user enable virtual-display.service vnc-server.service
systemctl --user start virtual-display.service vnc-server.service
```

**Lingering:** For the services to run without an active login session (and survive reboots):
```bash
loginctl enable-linger $(whoami)
```

### Step 6: Verify

```bash
# Check Xvfb is running on :99
DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && echo "Display :99 OK" || echo "Display :99 FAILED"

# Check VNC is listening
ss -tlnp | grep 5999 && echo "VNC port 5999 OK" || echo "VNC port 5999 FAILED"

# Check services
systemctl --user is-active virtual-display.service
systemctl --user is-active vnc-server.service
```

### Step 7: Report

```
VNC Service Setup Complete
  Display:  :99
  VNC Port: 5999
  Password: <shown above>
  Service:  virtual-display.service + vnc-server.service (enabled, will start on boot)

To connect: Use any VNC client → <server-ip>:5999
To use:     Set DISPLAY=:99 before running browser commands
To check:   /vnc-service:status
```

## Error Handling

- **sudo not available:** List the `apt-get install` commands for the user to run manually
- **systemd user services not supported:** Fall back to manual Xvfb + x11vnc startup instructions
- **Port 5999 already in use:** Check what's using it, offer to kill or pick a different port
- **Xvfb display :99 already locked:** Remove `/tmp/.X99-lock` (the service's ExecStartPre does this)

## Security Notes

- VNC password is stored in `~/.vnc/passwd` (VNC's obfuscated format, chmod 600)
- VNC listens on `0.0.0.0:5999` — accessible from the local network
- For production servers, consider SSH tunneling: `ssh -L 5999:localhost:5999 server`
- The virtual display runs as the current user, not root
