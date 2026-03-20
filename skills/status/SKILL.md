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
# Service installation check
if ! systemctl --user cat virtual-display.service >/dev/null 2>&1; then
  VD_STATUS="not-installed"
else
  VD_STATUS=$(systemctl --user is-active virtual-display.service 2>/dev/null)
fi

if ! systemctl --user cat vnc-server.service >/dev/null 2>&1; then
  VNC_STATUS="not-installed"
else
  VNC_STATUS=$(systemctl --user is-active vnc-server.service 2>/dev/null)
fi

# Process check
XVFB_PID=$(pgrep -f "Xvfb :99" | head -1 || echo "none")
VNC_PID=$(pgrep -f "x11vnc.*:99" | head -1 || echo "none")

# Port check
VNC_PORT_OK=$(ss -tlnp | grep -q 5999 && echo "listening" || echo "not listening")

# Display check
DISPLAY_OK=$(DISPLAY=:99 xdpyinfo >/dev/null 2>&1 && echo "OK" || echo "FAIL")

# Service enabled check
VD_ENABLED=$(systemctl --user is-enabled virtual-display.service 2>/dev/null || echo "n/a")
VNC_ENABLED=$(systemctl --user is-enabled vnc-server.service 2>/dev/null || echo "n/a")

# Lingering check
LINGER=$(loginctl show-user $(whoami) -p Linger 2>/dev/null | cut -d= -f2 || echo "unknown")

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
  Connect:          ssh -L 5999:localhost:5999 <user>@<ip>, then VNC to localhost:5999

  Auto-start:       enabled/disabled (virtual-display + vnc-server)
  Lingering:        yes/no (services persist without login session)
```

If anything is wrong, provide the specific fix command:
- Not installed → "Run /vnc-service:setup"
- Inactive → "Run /vnc-service:run"
- Display FAIL → "Check: journalctl --user -u virtual-display.service -n 20"
- Port not listening → "Check: journalctl --user -u vnc-server.service -n 20"
- Lingering off → "Run: loginctl enable-linger $(whoami)"
