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

```bash
systemctl --user is-active virtual-display.service >/dev/null 2>&1 || systemctl --user start virtual-display.service
systemctl --user is-active vnc-server.service >/dev/null 2>&1 || systemctl --user start vnc-server.service
```

### Step 3: Verify Display

```bash
DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && echo "OK" || echo "FAIL"
```

If display check fails:
1. Check if Xvfb process exists: `pgrep -f "Xvfb :99"`
2. Check for stale lock: `rm -f /tmp/.X99-lock` then restart service
3. Check service logs: `journalctl --user -u virtual-display.service --no-pager -n 20`

### Step 4: Verify VNC

```bash
ss -tlnp | grep 5999 && echo "OK" || echo "FAIL"
```

If VNC check fails, restart the vnc-server service.

### Step 5: Report Connection Info

Determine the server's IP address:
```bash
hostname -I | awk '{print $1}'
```

Print:
```
VNC Ready
  Connect: <server-ip>:5999
  Display: :99 (set DISPLAY=:99 for browser commands)

  Waiting for you to connect via VNC...
```

### Step 6: Wait for User

If called as a prerequisite for another operation (e.g., CAPTCHA solve or OAuth):
- Print the connection info
- Ask: "Connect to VNC now. Let me know when you're connected and I'll proceed."
- Wait for user confirmation before returning

## Usage by Other Skills

Other skills should call `/vnc-service:run` like this in their SKILL.md:

```markdown
### Prerequisite: Virtual Display

If the environment is headless (no DISPLAY set):
1. Invoke /vnc-service:run to ensure VNC is available
2. Wait for user to confirm VNC connection
3. Set DISPLAY=:99 before running browser commands
```
