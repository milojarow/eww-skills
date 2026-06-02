---
name: eww-troubleshooting
description: Debug and fix eww problems. Use when eww isn't working, windows won't open, config fails to load, styles aren't applying, variables aren't updating, getting compilation errors, or needing to understand eww logs, eww state, or eww debug output. Not for debugging non-eww processes, or general systemd/Wayland compositor troubleshooting unrelated to eww.
metadata:
  priority: 6
  pathPatterns: []
  bashPatterns: ["eww\\s+(logs|state|debug|inspector|kill|daemon|reload)"]
---

# eww-troubleshooting — Debug & Fix

> **🫟 ACTIVE-SKILL MARKER:** Prefix your reply with 🫟 **only on turns where the work touches the `eww-troubleshooting` domain** — debugging eww (daemon logs, a widget that won't appear, broken layout, `eww log`) — regardless of the layer/project (frontend, backend, a local script — all count); what matters is whether *this turn* touches the domain. On turns that do NOT touch it (typecheck, build, deploy, git ops, editing or curl in other domains), **omit 🫟** even if the skill loaded earlier in the session. If other active skills also apply to the same turn, **stack their emojis** in the prefix.

---

## Debug Workflow

> **⚠️ First — is eww managed by systemd?** If a user service runs eww with `Restart=always` (check `systemctl --user status eww.service`), do **NOT** use `eww kill` / `eww daemon` (Steps 1–2) or any manual daemon start/kill. The instant you kill it, systemd relaunches **its own** daemon while your manual one also starts — two daemons each open every window, so you get **duplicate / double bars**. Instead:
> - **Config change:** `eww reload`
> - **Full restart / clear stale state:** `systemctl --user restart eww.service`
> - **Verbose logs:** `journalctl --user -u eww.service -f` (or `eww logs`)
>
> The `eww kill` / `eww daemon` sequence below applies **only** to standalone eww with no systemd manager. Root cause: [SYSTEMD.md](SYSTEMD.md).

Always follow this sequence. Each step narrows the problem. Do not skip to advanced steps before completing the basics.

```bash
# Step 1: Kill the daemon to clear all stale state
eww kill

# Step 2: Start daemon in foreground — watch for immediate parse errors
eww daemon

# Step 3: Open the target window — watch terminal output for errors
eww open <window>

# Step 4: Stream live logs in a second terminal
eww logs

# Step 5: Check all variable values
eww state
eww state | grep -i <keyword>   # filter to relevant variable

# Step 6: Inspect widget tree structure
eww debug

# Step 7: Visual GTK inspector — for style/selector issues
eww inspector

# Step 8: Check which windows are defined and open
eww list-windows
```

When you need maximum verbosity, run the daemon in foreground debug mode:

```bash
eww kill && eww daemon --no-daemonize --debug
```

This prints all events, expression evaluations, and script invocations to stdout.

---

## Top 6 Most Common Issues

| # | Issue | One-Line Fix |
|---|-------|-------------|
| 1 | Window won't open | Check `eww logs` for parse errors; verify `defwindow` name matches `eww open <name>` |
| 2 | Styles not applying | Add `* { all: unset; }` at top of `eww.scss`; use `eww inspector` to find correct GTK selector |
| 3 | Variable not updating | Test the script manually in terminal; add `:run-while true` to `defpoll` |
| 4 | Script not found | Use absolute paths in `defpoll`/`deflisten`; eww does not inherit shell PATH |
| 5 | Config parse error | Count unmatched parentheses; check `eww logs` for "parse error at line X" |
| 6 | Bar lags / misses updates | The `deflisten` script does too much work per event (classic: scans all of `/proc`) — keep the event path cheap (see "Laggy bar" below) |

---

## Reading `eww logs`

`eww logs` tails the daemon log file in real time. Run it in a separate terminal while you reproduce the problem.

### Log levels

- `INFO` — normal operation: daemon started, window opened, variable updated
- `WARN` — recoverable issue: script returned non-zero, expression produced unexpected type
- `ERROR` — hard failure: parse error, window not found, script not found

### Patterns to look for

```
ERROR  Failed to parse config: unexpected token ')' at line 42
```
Points to a syntax error in `eww.yuck` at the given line number. Count parentheses around that line.

```
WARN   Script exited with non-zero exit code: 1
```
A `defpoll` or `deflisten` script failed. Test it manually in a terminal.

```
ERROR  No window named "bar" found
```
The window name in `eww open bar` does not match any `defwindow` in the config.

```
WARN   Could not convert value "" to type i32
```
A variable is empty when eww expects a number. Script returned nothing.

```
INFO   Reloading config
```
`eww reload` was triggered. Lines after this show whether reload succeeded.

### Log file location

```bash
# Default location
$XDG_RUNTIME_DIR/eww/logs   # usually /run/user/1000/eww/logs

# View without streaming
cat $XDG_RUNTIME_DIR/eww/logs
```

---

## Reading `eww state`

`eww state` prints all defined variables and their current values.

### Output format

```
battery_percent: 87
time:            14:32
workspaces:      [{"id":1,"name":"1"},{"id":2,"name":"2"}]
volume:          65
```

### How to spot variable problems

**Variable is empty string:**
```
volume:
```
The `defpoll` script returned nothing. Run the script manually to confirm it outputs a value.

**Variable is stuck at initial value:**
```
time: 0
```
The poll may not be running. Check whether the window is open (polls only run while window is open unless `:run-while true` is set).

**Variable is JSON but widget shows raw text:**
The widget is receiving the JSON string but not indexing into it. Use `{var.key}` or `{var[0]}` in the expression.

**Variable updates in `eww state` but widget doesn't change:**
Check expression syntax in `eww debug`. The widget may reference a different variable name.

---

## Reading `eww debug`

`eww debug` prints the full widget tree as eww has resolved it — after all `defwidget` substitutions and expression evaluations.

### Widget tree format

```
window: bar
  box (orientation: horizontal)
    label (text: "14:32")
    box (orientation: horizontal)
      button (onclick: "notify-send hello")
        label (text: "click me")
```

### How to read it

- **Indentation** = nesting (parent/child relationships)
- **Property values** = what eww actually computed, not what you wrote in yuck
- **Wrong value in a property** means the expression evaluated to the wrong thing — check `eww state` to see the variable's actual value

### Identifying layout issues

If a widget is missing from the tree entirely, the condition evaluating it is false or the variable is undefined. If properties show default/empty values, the expression failed to evaluate.

---

## Using `eww inspector`

The GTK inspector is the definitive tool for CSS issues.

```bash
eww inspector
```

### How to use it

1. Click the **crosshair icon** in the top-left of the inspector window
2. Click on any widget in your eww window
3. The inspector selects that widget and shows:
   - **CSS Nodes** tab — the GTK type hierarchy, applied CSS classes, and state flags (`:hover`, `:active`, etc.)
   - **CSS** tab — which CSS rules are currently applied and from which file
   - **Properties** tab — GTK object properties (not usually needed)

### Finding the right selector

The CSS Nodes tab shows entries like:

```
window.background
  box
    label.my-label:dir(ltr)
```

The GTK type name (`label`, `box`, `button`) is what you target in CSS. If your rule targets `.my-label` but the node shows `label.my-label`, a more specific rule like `label.my-label` may be needed due to specificity.

### Checking why a style isn't applied

In the **CSS** tab, look for your property in the list. If it appears crossed out (overridden), a more specific rule is winning. If it does not appear at all, the selector did not match or there is a parse error above it in `eww.scss`.

---

## Testing Scripts Independently

Before assuming eww is broken, verify that your scripts work in isolation.

### Testing a defpoll script

```bash
# Run the exact command from your defpoll
bash -c 'date +%H:%M'

# For a script file
~/.config/eww/scripts/getvol

# Simulate the minimal environment eww runs scripts in
env -i HOME=$HOME PATH=/usr/bin:/bin bash -c '~/.config/eww/scripts/getvol'
```

The script must print exactly one value to stdout and exit 0.

### Testing a deflisten script

```bash
# Run the listener directly — it should print one line per event
~/.config/eww/scripts/swayspaces

# Or with Python explicitly
python3 ~/.config/eww/scripts/swayspaces.py
```

The script must output lines continuously to stdout as events occur. If nothing appears after triggering an event, the issue is with the script, not eww.

### Common script failure patterns

**Output buffering (Python):**
```python
# ❌ WRONG — output may be buffered, eww never sees updates
print(data)

# ✅ CORRECT — force flush on every write
print(data, flush=True)
```

**PATH not inherited:**
```bash
# ❌ WRONG — eww cannot find "pactl" because /usr/bin may not be in PATH
(defpoll vol :interval "1s" `pactl get-sink-volume @DEFAULT_SINK@`)

# ✅ CORRECT — use absolute path
(defpoll vol :interval "1s" `/usr/bin/pactl get-sink-volume @DEFAULT_SINK@`)
```

See the PATH section in COMMON_ERRORS.md for full details.

---

## `eww reload` vs Kill and Restart

```bash
# eww reload — re-reads config, re-opens all previously-open windows
eww reload

# Kill and restart — full clean slate
eww kill && eww daemon && eww open bar
```

### When to use `eww reload`

Use `eww reload` for fast iteration when you know the config is valid. It is faster than kill+restart.

### When `eww reload` fails

`eww reload` fails silently or with an error when:
- There is a syntax error in the new config (check `eww logs`)
- A previously-open window name no longer exists in config

If `eww reload` fails, always fall back to `eww kill && eww daemon`.

### Handling windows removed from config

```bash
# ❌ WRONG — if "old-window" was open and no longer in config, this fails
eww reload && eww open bar

# ✅ CORRECT — semicolon ignores reload failure
eww reload ; eww open bar
```

---

## Valid yuck Syntax That Looks Like a Bug (But Isn't)

### Variable references without braces `{}`

Both bare references and `{}` form are valid for direct attribute values:

```yuck
; ✅ BOTH are correct and equivalent:
(label :text battery)       ; bare reference — valid
(label :text {battery})     ; expression form — also valid

(button :active panel-open) ; valid
(scale :value volume)       ; valid
```

**When `{}` is required:** Only when there are operators, functions, or field access:

```yuck
; ✅ These DO need {}:
(label :text {round(battery, 0)})
(button :class {battery < 20 ? "critical" : "ok"})
(label :text {EWW_BATTERY.BAT0.capacity})
```

> CRITICAL: If you are diagnosing a widget that doesn't update and your first suspicion is "missing `{}` on the attribute", verify first with `eww state` that the variable has the expected value. `(label :text battery)` is not a bug — it is perfectly valid yuck syntax.

---

## Laggy bar / missed updates — profile the event path

A `deflisten`-backed widget (workspaces, window title — anything driven by a
WM event stream) can fall behind: you switch workspace and the bar updates
late or not at all, **worst under memory pressure or with many processes
running**, while a native C bar (waybar) reacts instantly. The daemon isn't
slow — your **listener script is doing too much work per event**.

`deflisten` re-runs its emit path on *every* event, and WMs fire several
events per action (one workspace switch emits `workspace` + `window` focus +
more). Anything O(processes) or O(disk) on that path is paid every single
time, several times per keypress.

**The canonical offender: scanning `/proc` on every event.** Walking every
`/proc/<pid>` (to map process trees, detect a TUI like ranger inside a
terminal, etc.) costs ~30 ms with a few hundred processes — and balloons to
hundreds of ms once those reads start hitting swap. That is exactly why the
bar dies *only when the machine is loaded*.

**Diagnose — time the emit path in isolation** (import the listener, call its
build/emit function in a loop, print per-call ms):

```bash
python3 - <<'PY'
import time, importlib.util
spec = importlib.util.spec_from_file_location("m", "scripts/<your-listener>.py")
m = importlib.util.module_from_spec(spec); spec.loader.exec_module(m)
# set up args the way main() does (e.g. swaymsg get_tree / get_workspaces), then:
for _ in range(5):
    t = time.perf_counter(); m.build_output(...); print(f"{(time.perf_counter()-t)*1000:.2f} ms")
PY
```

Time each piece separately — **and measure under real load, not idle.** That
distinction is everything. Idle, a `swaymsg -t get_tree` looks cheap (~3 ms),
which is misleading: that call is a `fork()`+`execve()` spawning the WM CLI. A
full `/proc` scan is ~30 ms+. But under memory pressure (CPU contention + the
CLI binary paged out to swap) the *spawn itself* was measured at ~14 ms and
spikes to **seconds** when `execve` has to read the binary back from disk.
Spawning a process per event is as deadly as the `/proc` scan — and a query
over a persistent socket is ~1 ms with none of that risk.

**Fix — keep the event path cheap:**

- Cut work that only feeds a cosmetic detail. Removing a per-event `/proc`
  scan that existed solely to pick one icon took a real listener from ~32 ms
  to ~0.1 ms per event (≈300×) and stopped it degrading under load.
- If you genuinely must inspect a process subtree, walk only it via
  `/proc/<pid>/task/<tid>/children` — never scan all of `/proc`.
- Cache values that rarely change instead of recomputing them every event.
- Target O(1) or O(windows) per event, never O(system processes).
- **Hold a persistent IPC socket; never spawn the WM CLI (`swaymsg`/`i3-msg`)
  per event.** Spawning the CLI is a `fork`+`exec` every time; a long-lived
  UNIX socket to `$SWAYSOCK` — subscribe to events on one, reuse another for
  queries — does the same work with none of the process-creation cost. This is
  the architectural reason a native bar like waybar stays smooth, not its
  language. (i3-ipc framing: 6-byte magic `i3-ipc` + u32 length + u32 type in
  native byte order; event messages set the high bit on the type, which also
  lets you tell a workspace event from a window event.)
- **Update incrementally — don't rebuild all state on every event.** Classify
  the event by message type and `change`. A *focus* change moves no windows, so
  window-derived state (per-workspace app icons, etc.) cannot have changed:
  refresh only the cheap flag that did (which thing is focused, from the small
  `get_workspaces`) and reuse the cached tree. Re-read the full tree only on
  *structural* changes (window new/close/move). "Rebuild everything on every
  event" is the trap; "touch only what changed" is the fix.

Rule of thumb: a `deflisten` emit should take a few ms. If it doesn't, that is
why the bar feels dead next to waybar.

---

## Integration with Other Skills

- **eww-yuck** — use when the parse error is in `defwindow`, `defwidget`, or `defpoll` syntax; covers parenthesis rules, property names, and widget nesting
- **eww-expressions** — use when the expression inside `{}` or `"${}"` is wrong; covers operators, type coercion, and expression context rules
- **eww-styling** — use when styles are partially working but selectors or GTK CSS properties are behaving unexpectedly; covers GTK CSS specifics and `all: unset`
- **eww-widgets** — use when a specific built-in widget (slider, chart, calendar, etc.) is not behaving as expected; covers widget-specific properties
- **eww-patterns** — use when looking for complete working examples of bars, dashboards, and system monitors to reference against your config

---

## Summary

When eww is broken, follow the workflow in order:

1. `eww kill` then `eww daemon` — clears stale state and shows parse errors immediately
2. `eww open <window>` — watch terminal for errors
3. `eww logs` — stream logs to find the exact error
4. `eww state` — check variable values are what you expect
5. `eww debug` — verify widget tree is structured correctly
6. `eww inspector` — for CSS/style issues, find the real GTK selector
7. Test scripts manually in terminal — confirm scripts output the right value

Reference files:
- `DEBUGGING.md` — deep dive on each debug tool, script buffering, PATH issues
- `COMMON_ERRORS.md` — catalog of specific errors with exact causes and fixes
