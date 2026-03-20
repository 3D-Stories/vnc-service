---
name: vnc-service:status
description: >-
  Show the status of the virtual display and VNC service — whether running, display number,
  port, and connection info. Use when checking if VNC is available or troubleshooting display issues.
  Trigger on: "vnc status", "is vnc running", "display status", "check vnc".
---

# VNC Service Status

Display the current status of the virtual display and VNC server.

## Workflow

Run these checks and present a status report:

```bash
# Service status
VD_STATUS=$(systemctl --user is-active virtual-display.service 2>/dev/null || echo "not-installed")
VNC_STATUS=$(systemctl --user is-active vnc-server.service 2>/dev/null || echo "not-installed")

# Process check
XVFB_PID=$(pgrep -f "Xvfb :99" 2>/dev/null || echo "none")
VNC_PID=$(pgrep -f "x11vnc.*:99" 2>/dev/null || echo "none")

# Port check
VNC_PORT=$(ss -tlnp 2>/dev/null | grep 5999 | head -1 || echo "not listening")

# Display check
DISPLAY_OK=$(DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && echo "OK" || echo "FAIL")

# Server IP
SERVER_IP=$(hostname -I | awk '{print $1}')
```

## Report Format

```
VNC Service Status
  Virtual Display:  active/inactive/not-installed (PID: XXXX)
  VNC Server:       active/inactive/not-installed (PID: XXXX)
  Display :99:      OK / FAIL
  Port 5999:        listening / not listening
  Connect:          <ip>:5999

  Service enabled:  yes/no (survives reboot)
  Lingering:        enabled/disabled (runs without login session)
```

If anything is wrong, provide the specific fix command.
