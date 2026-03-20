---
name: vnc-service:run
description: >-
  Ensure the persistent virtual display + VNC is running and print connection info. Call this skill
  whenever a browser interaction is needed on a headless server — before OAuth login, CAPTCHA solve,
  or 2FA prompt. If the service isn't set up yet, directs to /vnc-service:setup. Trigger on:
  "start vnc", "vnc run", "I need to solve a CAPTCHA", "browser login needed", "OAuth prompt",
  or when any other skill needs browser interaction on a headless server.
---

# VNC Service Run

Ensure the virtual display and VNC server are running, and provide connection details.
This is the skill other plugins call when they need browser interaction on a headless server.

## Workflow

### Step 1: Check if Setup Has Been Done

```bash
test -f ~/.config/systemd/user/virtual-display.service && echo "SETUP" || echo "NOT_SETUP"
```

If not set up: inform the user and invoke `/vnc-service:setup`. Do not proceed until setup completes.

### Step 2: Ensure Services Are Running

Start the display first, wait for it, then start VNC:

```bash
# Start display if not running
if ! systemctl --user is-active virtual-display.service >/dev/null 2>&1; then
  systemctl --user start virtual-display.service
fi

# Wait for display :99 to be ready (up to 15 seconds)
for i in $(seq 1 30); do
  DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && break
  sleep 0.5
done

# Start VNC if not running
if ! systemctl --user is-active vnc-server.service >/dev/null 2>&1; then
  systemctl --user start vnc-server.service
fi
```

### Step 3: Verify Display

```bash
DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && echo "OK" || echo "FAIL"
```

If display check fails:
1. Check if Xvfb process exists: `pgrep -f "Xvfb :99" | head -1`
2. Check for stale lock: inspect PID in `/tmp/.X99-lock`, remove if dead, then restart service
3. Check service logs: `journalctl --user -u virtual-display.service --no-pager -n 20`

### Step 4: Verify VNC

```bash
ss -tlnp | grep -q 5999 && echo "OK" || echo "FAIL"
```

If VNC check fails, restart the vnc-server service.

### Step 5: Report Connection Info

Determine the server's IP address:
```bash
SERVER_IP=$(hostname -I | awk '{print $1}')
```

Print:
```
VNC Ready (localhost only — SSH tunnel required)
  1. From your PC: ssh -L 5999:localhost:5999 <user>@<server-ip>
  2. Connect VNC client to: localhost:5999
  Display: :99 (set DISPLAY=:99 for browser commands)
```

### Step 6: Wait for User

If called as a prerequisite for another operation (e.g., CAPTCHA solve or OAuth):
- Print the connection info
- Ask: "Connect to VNC now (SSH tunnel + VNC client). Let me know when you're connected."
- Wait for user confirmation before returning

## Usage by Other Skills

Other skills should call `/vnc-service:setup` as a prerequisite check and `/vnc-service:run`
before browser operations:

```markdown
### Prerequisite: Virtual Display

If the environment is headless (no DISPLAY set):
1. Invoke /vnc-service:setup to ensure VNC is installed
2. Invoke /vnc-service:run to ensure running and get connection info
3. Wait for user to confirm VNC connection
4. Set DISPLAY=:99 before running browser commands
```
