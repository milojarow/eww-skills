# eww systemd Service Hardening

---

## Preventing Duplicate Windows on Daemon Restart

### The Problem

When the eww daemon dies (crash, `eww reload` failure, SIGKILL), orphan `eww open` processes can survive. These hold Wayland connections and render layer-shell surfaces that persist visually even though the new daemon doesn't know about them.

`pkill -x eww` sends SIGTERM, but eww processes blocked on Wayland I/O may not respond to SIGTERM. They survive the kill, and the new daemon opens fresh windows — resulting in duplicate bars.

### The Fix: SIGTERM + SIGKILL Escalation

```ini
[Service]
ExecStartPre=-/bin/sh -c 'pkill -x eww 2>/dev/null; sleep 2; pkill -KILL -x eww 2>/dev/null; sleep 0.5'
```

This sequence:
1. **SIGTERM** all eww processes (graceful shutdown attempt)
2. **Wait 2 seconds** for processes to die
3. **SIGKILL** any survivors (cannot be caught or ignored)
4. **Wait 0.5 seconds** for sway to destroy orphan layer-shell surfaces

### Why SIGKILL is Necessary

- `eww open <window>` spawns a persistent manager process per window
- These managers can become orphans (PPID=1) when the daemon crashes
- Orphan managers blocked on Wayland I/O ignore SIGTERM
- Only SIGKILL guarantees death — the kernel forcibly terminates the process

---

## Single Source of Truth for Window Opening

**RULE:** Always use `systemctl --user restart eww.service` to restart eww. Never run the startup window-opening script manually.

### Why

If your unit's `ExecStartPost` runs a window-opening script (here called `open-windows.sh`) after the daemon starts, then ALSO running that script manually — or via `eww reload` + a manual open — gives two sources opening windows, creating duplicates.

```bash
# ❌ WRONG — two sources of window opening
eww reload && ~/.config/eww/scripts/open-windows.sh

# ❌ WRONG — eww reload crashes, systemd restarts, you also open manually
eww reload  # crashes
sleep 5     # systemd restarts daemon + ExecStartPost opens windows
~/.config/eww/scripts/open-windows.sh  # DUPLICATE

# ✅ CORRECT — single source of truth
systemctl --user restart eww.service
```

### When You Need to Open a Single Window

For opening one-off windows (e.g. an on-demand popup or overlay), use `eww open` directly — it's safe for single windows that the startup script doesn't open:

```bash
eww open my-popup   # fine — not in open-windows.sh
eww close my-popup  # clean close
```
