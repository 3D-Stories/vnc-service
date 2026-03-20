---
name: vnc-service
description: >-
  Persistent virtual display + VNC for headless servers. Four commands: /vnc-service:setup (install
  and configure), /vnc-service:run (ensure running + print connection info), /vnc-service:stop
  (stop services to free resources), /vnc-service:status (check health). Use whenever browser
  interaction is needed on a headless server — OAuth, CAPTCHA, 2FA. Other skills should call
  /vnc-service:setup as a prerequisite check and /vnc-service:run before browser operations.
---

# VNC Service Plugin

This plugin provides a persistent virtual display (Xvfb on `:99`) and VNC server (x11vnc on
port `5999`) for headless Linux servers. It enables browser-based interactions that normally
require a physical monitor.

## Skills

- `/vnc-service:setup` — One-time install and configuration
- `/vnc-service:run` — Ensure running + print connection info (call before browser operations)
- `/vnc-service:stop` — Stop services to free resources (restart anytime with /run)
- `/vnc-service:status` — Health check and diagnostics

## For Other Skill Authors

If your skill needs browser interaction on a headless server, add this to your prerequisites:

```markdown
### Prerequisite: Virtual Display (headless servers only)

Check if a display is available. If not, call /vnc-service:setup to install the virtual
display service, then /vnc-service:run before any browser operations.
```
