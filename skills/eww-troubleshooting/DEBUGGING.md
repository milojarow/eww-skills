# eww Debugging Tools — Deep Dive

---

## `eww logs`

### Location

eww writes logs to a file in the runtime directory:

```bash
$XDG_RUNTIME_DIR/eww/logs   # typically /run/user/1000/eww/logs
```

### Tailing logs

```bash
# Stream in real time (preferred while reproducing issues)
eww logs

# View full log without streaming
cat $XDG_RUNTIME_DIR/eww/logs

# View last 50 lines
tail -50 $XDG_RUNTIME_DIR/eww/logs
```

### Log format

Each line is structured as:

```
LEVEL  timestamp  message
```

Example entries:

```
INFO   2024-01-15 14:32:01  Starting eww daemon
INFO   2024-01-15 14:32:01  Loading config from /home/milo/.config/eww
ERROR  2024-01-15 14:32:01  Failed to parse config: unexpected ')' at line 42
INFO   2024-01-15 14:32:05  Opening window: bar
WARN   2024-01-15 14:32:06  Script exited with non-zero exit code: 1
INFO   2024-01-15 14:32:10  Variable updated: time = "14:32"
```

### What WARNING looks like

```
WARN   Script exited with non-zero exit code: 1
WARN   Could not convert value "" to type i32
WARN   Expression evaluation failed: variable 'battery_level' not found
```

Warnings do not stop eww but indicate something is wrong. A variable that always shows its initial value and generates warnings on every poll cycle points to a broken script or wrong variable reference.

### What ERROR looks like

```
ERROR  Failed to parse config: unexpected token '(' at line 17, col 3
ERROR  No window named "topbar" found
ERROR  Failed to start script: No such file or directory
```

Errors are hard failures. After a parse error, no windows from that config can be opened.

### Common log patterns and what they mean

| Pattern | Meaning |
|---------|---------|
| `Failed to parse config` + line number | Syntax error in `eww.yuck` |
| `No window named "X" found` | `eww open X` but no `defwindow X` in config |
| `Script exited with non-zero` | `defpoll`/`deflisten` script failed — test it manually |
| `Could not convert value "" to i32` | Script returned empty string, expected integer |
| `variable 'X' not found` | Expression references `X` but no `defvar`/`defpoll`/`deflisten` named `X` |
| `Reloading config` | `eww reload` was called |
| `Failed to reload` | Reload failed — syntax error or missing window prevents it |

---

## `eww state`

### Running it

```bash
eww state                        # all variables
eww state | grep battery         # filter
eww state | grep -E "^(vol|time)" # filter by prefix
```

### Output format

```
variable_name: value
```

For simple types:
```
time:            14:32
volume:          65
brightness:      80
battery_percent: 87
```

For JSON/array types, the full JSON string is printed on one line:
```
workspaces: [{"id":1,"name":"1","focused":true},{"id":2,"name":"2","focused":false}]
```

### Checking if a variable is updating

1. Run `eww state` once, note the value
2. Wait for the poll interval to expire
3. Run `eww state` again

If the value changed, the variable is updating correctly. If it did not change:
- Check whether the window is open (defpoll only runs while window is open)
- Check `eww logs` for script warnings
- Test the script manually

### Variables that never appear in `eww state`

If a variable is defined with `defvar`/`defpoll`/`deflisten` but does not appear in `eww state` output, it was not loaded — meaning the config failed to parse before reaching that definition. Check `eww logs`.

### JSON variables

If a variable holds JSON, `eww state` shows the raw JSON string. To verify the structure matches what your expression expects:

```bash
eww state | grep workspaces | cut -d' ' -f2- | python3 -m json.tool
```

---

## `eww debug`

### Running it

```bash
eww debug
```

### Widget tree format

`eww debug` prints the fully resolved widget tree — after all `defwidget` substitutions, expression evaluations, and conditional renders:

```
window: bar
  box [orientation=h, class="bar-box"]
    box [orientation=h, class="left"]
      label [text="14:32", class="time"]
      label [text=" ", class="separator"]
    box [orientation=h, class="right"]
      button [onclick="pactl set-sink-mute @DEFAULT_SINK@ toggle", class="vol-btn"]
        label [text="65%"]
```

### How to read it

- **Indentation** = parent/child relationship (nesting depth)
- **Widget type** = the GTK widget created (label, box, button, etc.)
- **Property values in brackets** = the actual computed values at the time of inspection

### Identifying layout issues

**Widget missing from tree entirely:**
The widget is wrapped in a conditional `{condition ? widget : ""}` and the condition evaluated to false, or the variable is undefined. Check `eww state` for the relevant variable.

**Property shows wrong value:**
The expression evaluated to something unexpected. Check `eww state` for the variables the expression uses. A common case: `{vol}` shows `0` in debug because `volume` is still `0` from initialization.

**Unexpected nesting:**
A `defwidget` expanded in a way you didn't expect. Check the widget definition in `eww.yuck` for extra wrapper boxes.

---

## `eww inspector`

### Launching

```bash
eww inspector
```

The GTK inspector window opens. Your eww window must be visible.

### Selecting an element

1. Click the **crosshair/target icon** in the top-left of the inspector
2. Move your cursor to the eww window — a highlight box shows the widget under the cursor
3. Click to select it

The inspector now shows information for that specific widget.

### CSS Nodes panel

The CSS Nodes tab shows the widget's GTK type hierarchy:

```
window.background
  decoration
  box#bar-box.bar-box:dir(ltr)
    box.left:dir(ltr)
      label.time:dir(ltr)
```

- The first part before `.` is the **GTK type name** (e.g., `label`, `box`, `button`)
- The part after `.` is the **CSS class** (e.g., `.time`, `.bar-box`)
- Pseudo-states like `:dir(ltr)`, `:hover`, `:active` are shown as state flags

Use the type name and class to write accurate CSS selectors:
```scss
label.time { color: #cdd6f4; }   // targets label with class "time"
box.left > label { ... }          // targets labels directly inside box.left
```

### CSS panel

Shows which CSS rules are being applied to the selected element. Rules that are overridden by more specific rules appear crossed out. Rules that failed to parse appear missing.

### Checking computed styles

If a property is not being applied:
1. Open CSS panel after selecting the widget
2. Look for your property — if it's crossed out, a more specific rule is overriding it
3. If it's not listed at all, either the selector did not match or there is a parse error earlier in `eww.scss` that prevented parsing from reaching that rule

---

## `eww reload` vs Kill and Restart

### `eww reload`

```bash
eww reload
```

- Re-reads `eww.yuck` and `eww.scss` without stopping the daemon
- Re-opens all windows that were previously open
- Faster than kill+restart for iteration

**When it works:** config is valid, no windows have been renamed or removed.

**When it fails (silently or with error in logs):**
- Syntax error in the new config
- A previously-open window no longer exists in config
- Stale variable state can sometimes cause unexpected behavior

### Kill and restart

```bash
eww kill && eww daemon && eww open bar
```

- Terminates daemon completely, clearing all state
- Required when: daemon is stuck, `eww reload` keeps failing, stale state is suspected
- Use this as the definitive reset when troubleshooting

### Pattern for iterating quickly

```bash
# During development — keep reloading until it breaks
eww reload

# When reload fails — hard reset
eww kill && eww daemon && eww open bar
```

---

## Testing defpoll Scripts

A `defpoll` script must:
1. Print exactly one value to stdout
2. Exit with code 0
3. Not require interactive input

```bash
# Test exactly as eww would run it
bash -c 'date +%H:%M'

# Test a script file
/home/milo/.config/eww/scripts/getvol

# Simulate minimal PATH environment
env -i HOME=$HOME PATH=/usr/bin:/bin:/usr/local/bin bash -c '/home/milo/.config/eww/scripts/getvol'

# Check exit code
/home/milo/.config/eww/scripts/getvol; echo "Exit: $?"
```

If the script outputs nothing, or exits non-zero, eww will not update the variable and will log a warning.

---

## Testing deflisten Scripts

A `deflisten` script must:
1. Run continuously (do not exit)
2. Print one line to stdout per update event
3. Flush stdout after each line (not buffer it)

```bash
# Run the listener — trigger events and watch for output lines
/home/milo/.config/eww/scripts/swayspaces

# Test a Python listener
python3 /home/milo/.config/eww/scripts/swayspaces.py
```

If you trigger an event (switch workspace, change volume, etc.) and nothing is printed, the script is not detecting the event correctly.

---

## Common Script Issues

### Output buffering

When a script writes to a pipe (as eww uses), the OS may buffer output and eww only receives updates in large chunks instead of line by line.

**Python:**
```python
# ❌ WRONG — buffered, eww sees updates in batches
import json, sys
while True:
    data = get_workspaces()
    print(json.dumps(data))

# ✅ CORRECT — flush=True forces immediate write
import json, sys
while True:
    data = get_workspaces()
    print(json.dumps(data), flush=True)
```

**Alternative for Python:** run with `python3 -u` (unbuffered mode):
```yuck
(deflisten workspaces `python3 -u /path/to/script.py`)
```

**Shell scripts** writing to stdout are line-buffered by default and generally do not need special handling.

### PATH not inherited

eww launched from a compositor or systemd service inherits a minimal environment. Shell PATH variables like `~/.local/bin` and `~/.cargo/bin` are NOT present.

```yuck
; ❌ WRONG — eww cannot find "waybar-audio" without $HOME/.local/bin in PATH
(defpoll vol :interval "1s" `waybar-audio`)

; ✅ CORRECT — absolute path always works
(defpoll vol :interval "1s" `/home/milo/.local/bin/waybar-audio`)
```

Or add PATH export at the top of your script:

```bash
#!/bin/bash
export PATH="$HOME/.local/bin:$HOME/.cargo/bin:/usr/local/bin:/usr/bin:/bin"
# ... rest of script
```

### Script not executable

```bash
# Check
ls -l /home/milo/.config/eww/scripts/getvol

# Fix
chmod +x /home/milo/.config/eww/scripts/getvol
```

eww runs script files directly. If the file is not executable and has no shebang, it will fail.

---

## Shell Script Pitfalls in eww Subscribe Scripts

### `trap '' SIGNAL` is inherited by child processes

POSIX mandates that `SIG_IGN` (ignore) disposition is inherited across `exec()`. If a subscribe script sets `trap '' INT` before launching a child process, that child **cannot be stopped via SIGINT**.

```bash
# ❌ WRONG — wf-recorder inherits SIG_IGN and ignores SIGINT
trap '' INT
wf-recorder -g "$area" --file="$file"
# pkill --signal SIGINT wf-recorder does NOTHING

# ✅ CORRECT — don't trap before launching children
wf-recorder -g "$area" --file="$file"
# pkill --signal SIGINT wf-recorder works as expected
```

### `pgrep -f` matches its own command line

When a subscribe script uses `pgrep -f "pattern"` to check if a process is running, pgrep's own command line contains the pattern, causing a **false positive** (always returns true).

```bash
# ❌ WRONG — always returns 0 (matches itself)
if pgrep -f "ssh.*-L 27017:localhost:27017" > /dev/null; then
    echo "connected"  # always reaches here
fi

# ✅ CORRECT — use a PID file
PIDFILE="/tmp/my-tunnel.pid"
is_alive() {
    [[ -f "$PIDFILE" ]] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null
}
```

---

## Widget Overlap Detection

**CRITICAL:** Before debugging click areas or visual artifacts on desktop widgets, ALWAYS check if another widget's invisible bounding box overlaps.

### Why this matters

eww desktop widgets have invisible bounding boxes that extend beyond their visible content. A widget with `width "787px"` creates a 787px-wide click-blocking rectangle even if only 200px of text is visible. Widgets on higher stacking layers block clicks on lower ones.

### Stacking layer order (bottom to top)

| Layer | Clickable? | Use case |
|-------|-----------|----------|
| `"bottom"` | No (below desktop) | Decorative (activate-linux) |
| `"bg"` | Yes (above desktop, below apps) | Desktop widgets |
| Apps | Yes | Terminal, browser, etc. |
| `"fg"` | Yes (above apps) | Bar, popups |
| `"overlay"` | Yes (above everything) | Lock screen, OSD |

### Coordinate system for overlap detection

Each widget has an anchor point. Coordinates are relative to that anchor corner:

| Anchor | 0,0 is... | X+ moves... | Y+ moves... |
|--------|-----------|-------------|-------------|
| `top left` | Top-left corner | Right | Down |
| `top right` | Top-right corner | Left | Down |
| `bottom left` | Bottom-left corner | Right | Up |
| `bottom right` | Bottom-right corner | Left | Up |

### How to check for overlaps

```bash
# 1. List all open windows
eww active-windows

# 2. For each, grep its geometry from the yuck file
grep -A5 "defwindow widget-name" ~/.config/eww/widgets/*.yuck

# 3. Map bounding boxes — anchor + x + y + width/height
# Look for rectangles that intersect with your widget
```

---

## Debugging Methodology

### Don't accumulate error over error

When a fix doesn't work, **STOP**. Don't patch on top of the failed fix. Instead:

1. Revert to the last known-good state
2. Re-examine the root cause with fresh eyes
3. Consider that your assumption about the cause might be wrong
4. Check for external factors (widget overlaps, orphan processes, stale state)
