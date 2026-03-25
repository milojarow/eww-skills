# eww CLI Commands

All eww CLI commands with exact syntax and usage notes.

---

## eww daemon — start daemon

```bash
eww daemon                        # start daemon in background (normal usage)
eww daemon --no-daemonize         # run in foreground — output goes to terminal
eww daemon --no-daemonize --debug # foreground with verbose log output
```

The daemon must be running before any other eww command will work. It reads the config and holds widget state.

---

## eww open — open a window

```bash
eww open <window>                             # open window by name
eww open bar --id primary                     # open with explicit ID (multi-instance)
eww open bar --screen 0                       # open on specific monitor (by index)
eww open bar --toggle                         # open if closed, close if already open
eww open bar --arg key=value                  # pass a window argument
eww open bar --arg key=value --arg key2=val2  # multiple arguments
eww open bar --id primary --arg monitor=eDP-1 # ID + argument combined
```

Window `--id` defaults to the window name if not specified. Use `--id` to run multiple instances of the same window config.

### open-many — open multiple windows

```bash
eww open-many bar:primary bar:secondary
eww open-many bar:primary bar:secondary --arg size=small         # shared arg (no id prefix)
eww open-many bar:primary --arg primary:monitor=eDP-1            # per-window arg (id prefix)
eww open-many bar:primary bar:secondary \
  --arg primary:monitor=eDP-1 \
  --arg secondary:monitor=HDMI-A-1
```

The `--arg` option must come after all window names in the `open-many` command.

---

## eww close — close a window

```bash
eww close <window>     # close by window name or ID
eww close bar          # close window named "bar"
eww close primary      # close window with ID "primary"
eww close-all          # close all open windows
```

---

## eww toggle — toggle a window

```bash
eww toggle <window>    # open if closed, close if open
```

Equivalent to `eww open <window> --toggle`. Useful for popup panels triggered by buttons.

---

## eww update — update a variable

```bash
eww update <var>=<value>
eww update my-var="hello world"
eww update count=42
eww update active=true
eww update active=false
eww update items='["a","b","c"]'
eww update data='{"key":"value"}'
```

Works on both `defvar` and `defpoll` variables. Updates take effect immediately in all open widgets using that variable.

---

## eww state — show all variable values

```bash
eww state                    # print all variable names and current values
eww state | grep battery     # filter to specific variable
```

Use to verify what value a variable currently holds, especially when debugging why a widget is showing unexpected content.

---

## eww get — print one variable's value

```bash
eww get <variable>
eww get count
eww get workspaces
```

Prints the current value of a single variable. Useful in scripts to read state before updating:

```bash
v=$(eww get count)
eww update count=$(( v + 1 ))
```

---

## eww poll — force-poll a variable

```bash
eww poll <variable>
eww poll weather
```

Immediately triggers a poll for a `defpoll` variable, outside its normal interval. Also works when `:run-while` is currently false.

---

## eww debug — show widget tree

```bash
eww debug
```

Prints the current widget tree structure with internal state. Useful for understanding how eww is interpreting your config.

---

## eww logs — tail daemon logs

```bash
eww logs
```

Streams live log output from the eww daemon. Shows parse errors, variable update events, and runtime errors. This is the first command to run when something is not working.

---

## eww inspector — GTK visual debugger

```bash
eww inspector
```

Opens the GTK inspector — a visual tool for inspecting the widget tree, examining CSS classes, and testing style changes live. The GTK inspector is the fastest way to debug styling issues.

---

## eww reload — hot-reload config

```bash
eww reload
```

Reloads the yuck and scss config without restarting the daemon. Attempts to re-open all previously open windows.

**Important behavior:**
- If a window that was previously open no longer exists in the config, reload exits with code 1
- This means `eww reload && eww open my-window` may fail if any old window is gone
- Use `;` instead of `&&` when chaining with reload:

```bash
eww reload ; eww open my-window    # correct — continues even if reload has warnings
eww reload && eww open my-window   # risky — stops if reload returns non-zero
```

Try `eww reload` before reaching for `eww kill`. Only kill when the config fails to parse entirely.

---

## eww kill — stop daemon

```bash
eww kill
```

Stops the daemon and clears all open window state. After `eww kill`, no windows are remembered — you must open them explicitly after starting a new daemon.

---

## eww list-windows — list defined windows

```bash
eww list-windows
```

Lists all windows defined in the config, showing which are currently open. Useful for confirming window names and open state.

---

## eww ping — check daemon alive

```bash
eww ping
```

Returns `pong` if the daemon is running. Use in scripts to check daemon state before sending commands.

---

## --config flag — use alternate config directory

```bash
eww --config ~/eww-bar daemon
eww --config ~/eww-bar open bar
eww --config ~/eww-bar reload
eww --config ~/eww-bar logs
eww --config ~/eww-bar kill
```

Runs a separate eww daemon instance that uses a different config directory. Every command in that session must include `--config` to target the right daemon. The two instances have completely separate state and logs.

---

## Common workflows

### Normal start

```bash
eww daemon
eww open bar
```

### After editing config

```bash
eww reload
# or if reload fails (parse error):
eww kill && eww daemon && eww open bar
```

### Debug loop — investigate a broken config

```bash
eww kill
eww daemon --no-daemonize    # see all errors in terminal
# (in another terminal)
eww logs                     # stream errors
eww state                    # check variable values
eww debug                    # check widget tree
eww inspector                # check CSS live
```

### Full restart (clean slate)

```bash
eww kill && eww daemon && eww open my-window
```

### Verify variable before updating (shell)

```bash
v=$(eww get count)
eww update count=$(( v + 1 ))
```

---

## onclick patterns — calling eww from widgets

```yuck
; Open / close / toggle windows
(button :onclick "eww open powermenu")
(button :onclick "eww close powermenu")
(button :onclick "eww open panel --toggle")

; Update variables
(button :onclick "eww update active=true")
(button :onclick "eww update active=${!active}")

; Increment with shell arithmetic
(button :onclick "v=$(eww get count); eww update count=$(( v + 1 ))")

; Reset counter at maximum
(button :onclick "v=$(eww get fill); [ \"$v\" -ge 100 ] && eww update fill=0 || eww update fill=$(( v + 10 ))")

; Chain with other commands
(button :onclick "notify-send 'Hello' && eww update notified=true")

; Run a script
(button :onclick "~/.config/eww/scripts/toggle-panel.sh")
```
