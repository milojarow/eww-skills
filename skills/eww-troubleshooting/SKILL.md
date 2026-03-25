---
name: eww-troubleshooting
description: Debug and fix eww problems. Use when eww isn't working, windows won't open, config fails to load, styles aren't applying, variables aren't updating, getting compilation errors, or needing to understand eww logs, eww state, or eww debug output.
---

# eww-troubleshooting — Debug & Fix

---

## Debug Workflow

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

## Top 5 Most Common Issues

| # | Issue | One-Line Fix |
|---|-------|-------------|
| 1 | Window won't open | Check `eww logs` for parse errors; verify `defwindow` name matches `eww open <name>` |
| 2 | Styles not applying | Add `* { all: unset; }` at top of `eww.scss`; use `eww inspector` to find correct GTK selector |
| 3 | Variable not updating | Test the script manually in terminal; add `:run-while true` to `defpoll` |
| 4 | Script not found | Use absolute paths in `defpoll`/`deflisten`; eww does not inherit shell PATH |
| 5 | Config parse error | Count unmatched parentheses; check `eww logs` for "parse error at line X" |

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
/home/milo/.config/eww/scripts/getvol

# Simulate the minimal environment eww runs scripts in
env -i HOME=$HOME PATH=/usr/bin:/bin bash -c '/home/milo/.config/eww/scripts/getvol'
```

The script must print exactly one value to stdout and exit 0.

### Testing a deflisten script

```bash
# Run the listener directly — it should print one line per event
/home/milo/.config/eww/scripts/swayspaces

# Or with Python explicitly
python3 /home/milo/.config/eww/scripts/swayspaces.py
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
