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

---

## Rapid `eww reload` Wedges the Daemon IPC

Firing several `eww reload` in quick succession while iterating on a config can wedge the daemon's IPC socket. Once wedged, `eww get` / `eww update` / `eww open` / `eww state` return empty output or fail:

```
ERROR eww::error_handling_ctx > Error while forwarding command to server
Caused by:
  0: Error reading response from server
  1: Resource temporarily unavailable (os error 11)
```

The nastier failure mode is **silent**: some commands return exit code 0 while *not* applying — `eww update VAR=x` exits 0 but `eww get VAR` keeps returning empty. Each `eww reload` re-opens every managed window (an Opening/Closing cascade visible in `journalctl --user -u eww.service`); stacking reloads is the strongly-correlated trigger.

### The misdiagnosis trap

During the wedge, an **existing, known-good** variable also returns empty from `eww get`. That is the tell: the problem is the daemon/IPC, not your newly-added `defvar`/`defwindow`. Don't conclude "my new definition failed to load" when get/update are failing globally.

**Always run a control:** `eww get <a-pre-existing-var>`. If that is also empty, it's the daemon — stop debugging your config and recover the daemon first. (Note `eww state` is not a reliable existence test for new defs — see DEBUGGING.md, "Variables that never appear in `eww state`".)

### Recovery

On a setup where eww runs as a systemd user unit (`Restart=always`), one clean service restart recovers the IPC immediately and reloads the config from scratch:

```bash
systemctl --user restart eww.service
```

Never recover with manual `eww kill` / `eww daemon` here — that spawns a second daemon and duplicate windows (see "Preventing Duplicate Windows" above).

### Best practice after a config edit

- Do **one** reload (or a service restart when you've added a brand-new `defwindow`/`defvar`), then let the daemon settle before issuing IPC commands. Don't stack reloads.
- A service restart is the unambiguous path for new **structural** defs (new windows/vars); frequent **theme/style** reloads should prefer `eww reload` to stay under the systemd `StartLimit` (5/min).
- Gate verification on a condition, not a blind `sleep`:
  ```bash
  until [ -n "$(eww get <existing-var>)" ]; do sleep 0.3; done
  # daemon is responsive again — now test the new defs
  ```
