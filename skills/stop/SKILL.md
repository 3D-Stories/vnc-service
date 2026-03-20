---
name: vnc-service:stop
description: >-
  Stop the virtual display and VNC server services. Use when you no longer need browser interaction
  and want to free resources. The services can be restarted anytime with /vnc-service:run.
  Trigger on: "stop vnc", "vnc stop", "disable vnc", "turn off vnc", "shut down display".
---

# VNC Service Stop

Stop the Xvfb virtual display and x11vnc VNC server. Services remain installed and can be
restarted with `/vnc-service:run` at any time.

## Workflow

### Step 1: Stop Services

```bash
systemctl --user stop vnc-server.service virtual-display.service
```

### Step 2: Verify Stopped

```bash
if ! systemctl --user is-active virtual-display.service >/dev/null 2>&1; then
  echo "Display stopped"
else
  echo "WARNING: Display is still running"
fi

if ! systemctl --user is-active vnc-server.service >/dev/null 2>&1; then
  echo "VNC stopped"
else
  echo "WARNING: VNC is still running"
fi
```

### Step 3: Optionally Disable Auto-Start

Ask the user:
```
Services stopped. Do you also want to disable auto-start on boot?
  1. Keep auto-start (services restart on reboot) — recommended if you use VNC regularly
  2. Disable auto-start (services stay off until you run /vnc-service:run)
```

If disable:
```bash
systemctl --user disable virtual-display.service vnc-server.service
```

### Step 4: Report

```
VNC Service Stopped
  Display :99: stopped
  VNC port 5999: closed
  Auto-start: enabled/disabled

  Restart anytime with: /vnc-service:run
```

## Notes

- Stopping the display will disconnect any active VNC sessions
- Any browser processes using DISPLAY=:99 will lose their display but won't crash
  (they'll error on next render attempt)
- The Chrome browser profile and CAPTCHA cookies are NOT affected by stopping —
  they persist in the filesystem
