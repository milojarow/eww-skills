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

**RULE:** Always use `systemctl --user restart eww.service` to restart eww. Never run `open-windows.sh` manually.

### Why

The systemd service ExecStartPost automatically calls `open-windows.sh` after the daemon starts. If you ALSO run `open-windows.sh` manually (or via `eww reload` + manual open), two sources open windows — creating duplicates.

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

For opening individual windows (like the arrow widget), use `eww open` directly — it's safe for single windows that aren't in `open-windows.sh`:

```bash
eww open widget-de-las-flechas   # fine — not in open-windows.sh
eww close widget-de-las-flechas  # clean close
```
