# eww Common Errors Catalog

---

## Error: Window not appearing ("No windows opened")

**Symptom:** `eww open bar` runs without error but no window appears on screen.

**Cause:** One of several silent failures — name mismatch, parse error in config, daemon not running, wrong monitor index, or missing Wayland stacking.

**Fix:**
```bash
# 1. Verify the daemon is running
eww daemon   # if not running, start it

# 2. Check for parse errors
eww logs     # look for "Failed to parse" or "ERROR"

# 3. Verify the window name matches exactly
#    If your defwindow uses "topbar", you must run "eww open topbar"
eww list-windows

# 4. For Wayland: add :stacking to defwindow
# ❌ WRONG — window exists but may not be visible on Wayland
(defwindow bar
  :geometry (geometry :width "100%" :height "30px"))

# ✅ CORRECT
(defwindow bar
  :geometry (geometry :width "100%" :height "30px")
  :stacking "fg")
```

---

## Error: Config not loading (parse error in yuck)

**Symptom:** `eww logs` shows a parse error, or all `eww open` commands fail silently.

**Cause:** `eww.yuck` contains a syntax error. eww cannot load any windows until the config parses cleanly.

**Fix:**
```bash
# Find the error
eww logs   # shows "parse error at line X" or "unexpected token"
```

Common yuck parse errors:

```yuck
; ❌ WRONG — unmatched parentheses
(defwindow bar
  :geometry (geometry :width "100%"
  (box))   ; missing closing ) for geometry

; ✅ CORRECT
(defwindow bar
  :geometry (geometry :width "100%")
  (box))

; ❌ WRONG — typo in property name
(defwindow bar :exclusve true)

; ✅ CORRECT
(defwindow bar :exclusive true)

; ❌ WRONG — comma between children (yuck uses spaces)
(box child1, child2)

; ✅ CORRECT
(box child1 child2)
```

---

## Error: CSS/styles not applying

**Symptom:** Widget renders but has no custom styling, or only some styles apply.

**Cause:** Missing `all: unset`, wrong CSS selector, or a parse error earlier in `eww.scss` stopping all rules below it from loading.

**Fix:**

```scss
// ❌ WRONG — GTK defaults override custom styles
label.time {
  color: #cdd6f4;
}

// ✅ CORRECT — reset GTK defaults first (at top of eww.scss)
* {
  all: unset;
}

label.time {
  color: #cdd6f4;
}
```

For wrong selector — use `eww inspector` to find the exact GTK type:
```bash
eww inspector   # crosshair → click widget → CSS Nodes tab
```

For CSS parse error:
```bash
# Eww stops parsing CSS at first error — comment out sections to bisect
# Check eww logs for CSS-related warnings
eww logs | grep -i css
```

GTK CSS does not support `width`/`height` directly:
```scss
/* ❌ WRONG */
.my-widget { width: 100px; }

/* ✅ CORRECT */
.my-widget { min-width: 100px; }
```

---

## Error: Yuck layout attribute used as CSS property (global style breakage)

**Symptom:** After adding or editing a `.scss` file, **all widgets lose their custom styles** — not just the one being worked on. The eww inspector shows no custom CSS rules applying anywhere. `journalctl --user -u eww.service` shows a line like:

```
error: 'spacing' is not a valid property name
    ┌─ /home/milo/.config/eww/eww.scss:NNN:1
```

**Cause:** A yuck widget attribute (`spacing`, `halign`, `valign`, `hexpand`, `vexpand`, `space-evenly`, `orientation`) was written as a CSS property in a `.scss` file. GTK CSS does not recognize these as valid properties. eww stops compiling the stylesheet at the first invalid property, so every rule defined after that line silently disappears.

**Why it's dangerous:** The error is in one widget's `.scss` file but breaks all other widgets. The error message names the property and the line number but does NOT say which widget is affected — it only shows the compiled line number in the final `eww.scss`.

**Fix:**

```scss
/* ❌ WRONG — 'spacing' is a yuck attribute, not a CSS property */
.my-box {
  spacing: 8px;
}

/* ✅ CORRECT — set spacing in yuck: (box :class "my-box" :spacing 8 ...) */

/* ✅ CORRECT — for CSS-side child spacing, use margin */
.my-box > * {
  margin: 0 4px;
}
```

Full list of yuck layout attributes that must **never** appear in SCSS:

| Yuck attribute | What to do instead in CSS |
|---|---|
| `:spacing N` | Set in yuck only; or use `margin` on children in CSS |
| `:halign` | No direct CSS equivalent for GTK box children; set in yuck |
| `:valign` | No direct CSS equivalent; set in yuck |
| `:hexpand` / `:vexpand` | No CSS equivalent; set in yuck |
| `:space-evenly` | No CSS equivalent; set in yuck |
| `:orientation` | No CSS equivalent; set in yuck |

**Diagnostic:**

```bash
# Find the exact invalid line immediately:
journalctl --user -u eww.service -n 50 --no-pager | grep "is not a valid property"
```

### Also breaks all styles: grass-unsupported CSS properties

The same "all widgets lose styles" symptom occurs for any property that **grass** (eww's embedded Rust SCSS compiler) does not recognize — even if the property is valid standard CSS that `sassc` or `dart-sass` accept without error:

| Property / construct | Error message | Fix |
|---|---|---|
| `gap` | `'gap' is not a valid property name` | Use `margin` on children or `:spacing` in yuck |
| `line-height` | `'line-height' is not a valid property name` | Remove it; use `padding: 0` on labels and rely on font metrics |
| Comma keyframe selectors (`0%, 100% { }`) | `Expected closing bracket after keyframes block` | Expand to one selector per line |

**Important:** `@keyframes` itself **does work** in grass — the bt-scan-pulse animation is proof. Only comma-separated selectors inside keyframe blocks fail.

```scss
/* ❌ FAILS in grass */
@keyframes my-anim {
  0%, 100% { opacity: 0.2; }
  50%      { opacity: 1;   }
}

/* ✅ CORRECT */
@keyframes my-anim {
  0%   { opacity: 0.2; }
  50%  { opacity: 1;   }
  100% { opacity: 0.2; }
}
```

```bash
# Catches ALL grass SCSS errors (property, keyframe, etc.):
journalctl --user -u eww.service -n 20 --no-pager | grep "error:"
```

---

## Error: Nested for loops (config fails to load)

**Symptom:** After adding an inner `(for ...)` loop inside an outer `(for ...)`, eww stops loading the config entirely. `eww open` produces no window, `eww reload` reports failure, and `eww logs` shows a parse or runtime error near the inner loop line.

**Cause:** eww does not support nested `for` iteration. The inner loop is invalid syntax at the config level.

**Fix:** Pre-compute all grouping in the data source script and emit flat named fields. Use a fixed number of sibling widgets in yuck instead of a dynamic inner loop.

```yuck
; WRONG — nested for causes a config load failure
(for ws in workspaces
  (box
    (label :text "${ws.name}")
    (for row in {ws.icon_rows}      ; NOT supported
      (label :text row))))

; CORRECT — script pre-computes icon_top/icon_mid/icon_bot fields
(for ws in workspaces
  (box
    (label :text "${ws.icon_top}")
    (label :text "${ws.icon_mid}" :visible {ws.has_mid})
    (label :text "${ws.icon_bot}" :visible {ws.has_bot})))
```

**Corollary — mixing a sibling label with a for loop inside the same box:**
Placing a plain `(label :visible {!condition} ...)` as a direct sibling of a `(for ...)` in the same box also causes rendering/parse issues. Encode all display cases (including fallback states) into the flat fields so yuck only ever has homogeneous fixed widgets.

**Diagnostic:**
```bash
journalctl --user -u eww.service -n 30 --no-pager | grep -E "error|parse"
```

---

## Error: Variable not updating

**Symptom:** `defpoll` value stays at its initial value; `eww state` shows a stale value.

**Cause:** Window is closed (polls only run while window is open by default), script fails, wrong interval format, or `:run-while` not set.

**Fix:**

```yuck
; ❌ WRONG — poll stops when window closes
(defpoll time :interval "5s"
  `date +%H:%M`)

; ✅ CORRECT — poll runs even when window is closed
(defpoll time :interval "5s" :run-while true
  `date +%H:%M`)
```

For wrong interval format:
```yuck
; ❌ WRONG — not valid interval syntax
(defpoll time :interval 5)
(defpoll time :interval "5sec")
(defpoll time :interval "5 s")

; ✅ CORRECT
(defpoll time :interval "5s")
(defpoll time :interval "500ms")
(defpoll time :interval "1m")
```

Test the script manually:
```bash
bash -c 'date +%H:%M'   # must print a value and exit 0
```

---

## Error: `${var}` showing literal text

**Symptom:** Widget displays the literal string `${variable}` or `{variable}` instead of the variable's value. Or a boolean property receives a string "true" instead of a boolean.

**Cause:** Wrong expression context. In eww, `{expr}` is an expression (evaluated), `"${expr}"` is a string interpolation. Using `"${expr}"` where a plain value is expected passes a string, not a boolean or number.

**Fix:**
```yuck
; ❌ WRONG — passes string "true", not boolean
(button :active "${count > 0}")

; ✅ CORRECT — expression returns actual boolean
(button :active {count > 0})

; ❌ WRONG — {name} in a text attribute loses the surrounding string context
(label :text {name})   ; works only if name is a complete value

; ✅ CORRECT — string interpolation embeds the value
(label :text "Hello, ${name}!")

; ❌ WRONG — unquoted words in ternary are treated as variable references
:class {active ? active : inactive}

; ✅ CORRECT — quote string literals
:class {active ? "active" : "inactive"}
```

---

## Error: deflisten not receiving data

**Symptom:** A `deflisten` variable never updates; the variable stays at the `defvar` initial value.

**Cause:** Script is buffering output (writing to stdout but not flushing), script is not outputting to stdout at all, or script exits immediately instead of running continuously.

**Fix:**

Test the script manually first:
```bash
# Script should print lines continuously as events occur
/home/milo/.config/eww/scripts/swayspaces
```

For Python buffering:
```python
# ❌ WRONG — output is buffered, eww gets nothing until buffer fills
print(json.dumps(data))

# ✅ CORRECT
print(json.dumps(data), flush=True)

# Alternative: run Python with -u flag in defvar
```

```yuck
; ✅ CORRECT — force Python unbuffered mode
(deflisten workspaces `python3 -u /home/milo/.config/eww/scripts/swayspaces.py`)
```

If the script outputs to stderr instead of stdout, redirect it:
```bash
# In the script, use stdout not stderr
echo "data"          # ✅ stdout
echo "data" >&2      # ❌ stderr — eww does not read this
```

**Cause: variable not referenced in any widget**

**Symptom:** `eww state` does not show the variable at all — not stuck at `:initial`, but completely absent from the output.

**Cause:** eww only starts a `deflisten` script when the variable it defines is referenced in an open window's widget tree. A side-effect-only deflisten — where the script calls `eww close`/`eww open` internally but the variable is never used in any widget attribute — is silently never started by the daemon.

**Fix:** Add a hidden reference in a persistent window (e.g., the main bar):

```yuck
; ❌ WRONG — variable defined but never used in a widget; script silently never runs
(deflisten fullscreen-active :initial "false"
  `~/.config/eww/scripts/fullscreen-subscribe.sh`)

; ✅ CORRECT — hidden label anchors the variable; script starts as long as bar is open
(label :visible false :text {fullscreen-active})
```

**Diagnostic tip:** distinguish the two failure modes:
- Variable **absent from `eww state`** → script is not running (not referenced in any widget)
- Variable **present but stuck at `:initial`** → script is running but has a buffering or stdout issue (see above)

---

## Error: Wayland window not appearing

**Symptom:** On Wayland, `eww open bar` runs but window is not visible. Window does appear on X11.

**Cause:** Missing `:stacking`, wrong anchor for `:exclusive`, or eww was built with X11 features instead of Wayland features.

**Fix:**

```yuck
; ❌ WRONG — no stacking set on Wayland
(defwindow bar
  :geometry (geometry :width "100%" :height "30px" :anchor "top center"))

; ✅ CORRECT
(defwindow bar
  :geometry (geometry :width "100%" :height "30px" :anchor "top center")
  :stacking "fg"
  :exclusive true)
```

For `:exclusive true`, the anchor must include `center`:
```yuck
; ❌ WRONG — exclusive with non-center anchor
:geometry (geometry :anchor "top left")
:exclusive true

; ✅ CORRECT
:geometry (geometry :anchor "top center")
:exclusive true
```

Verify Wayland build:
```bash
eww --version   # should mention wayland features
# If wrong, rebuild: cargo build --no-default-features --features=wayland
```

For windows that need keyboard input (like launchers):
```yuck
(defwindow launcher
  :focusable "ondemand"
  :stacking "overlay")
```

---

## Error: X11 window not reserving space

**Symptom:** On X11, a bar window does not push other windows down — maximized windows overlap the bar.

**Cause:** On X11, `:exclusive true` (a Wayland mechanism) does not work. X11 requires explicit struts.

**Fix:**
```yuck
; ❌ WRONG on X11 — :exclusive is a Wayland-only property
(defwindow bar
  :exclusive true
  :geometry (geometry :width "100%" :height "30px" :anchor "top center"))

; ✅ CORRECT for X11
(defwindow bar
  :reserve (struts :distance "30px" :side "top")
  :geometry (geometry :width "100%" :height "30px" :anchor "top center"))
```

---

## Error: Multiple daemon instances / stale state

**Symptom:** Opening windows behaves unexpectedly; `eww state` shows values from a previous session; commands seem to apply to a different window than expected.

**Cause:** More than one eww daemon is running. This can happen when eww crashes or when the daemon is started multiple times.

**Fix:**
```bash
# Kill all eww daemon processes
pkill -f "eww daemon"

# Verify no eww processes remain
pgrep -a eww

# Start a clean daemon
eww daemon

# Open your windows
eww open bar
```

---

## Error: Script not found from eww

**Symptom:** `defpoll` or `:onclick` command fails with "command not found" or "No such file or directory" even though the command works fine in terminal.

**Cause:** eww (when launched from a compositor or systemd service) inherits a minimal environment without the user's shell PATH. Directories like `~/.local/bin`, `~/.cargo/bin`, and `~/.npm-global/bin` are not present.

**Fix:**
```yuck
; ❌ WRONG — "waybar-audio" not found because ~/.local/bin is not in PATH
(defpoll vol :interval "1s"
  `waybar-audio`)

; ✅ CORRECT — absolute path always resolves regardless of PATH
(defpoll vol :interval "1s"
  `/home/milo/.local/bin/waybar-audio`)
```

Or export PATH at the top of the called script:
```bash
#!/bin/bash
export PATH="$HOME/.local/bin:$HOME/.cargo/bin:/usr/local/bin:/usr/bin:/bin"
# ... rest of script
```

This is documented in MEMORY.md as a confirmed issue for eww launched from Sway/systemd.

---

## Error: Compilation errors (eww won't build)

**Symptom:** `cargo build` fails with Rust compiler errors.

**Cause:** Rust version is too old, or required system dependencies are missing.

**Fix:**
```bash
# Update Rust to latest stable
rustup update

# Verify version
rustc --version   # should be recent stable

# For Wayland build, ensure correct feature flags
cargo build --release --no-default-features --features=wayland

# For X11 build
cargo build --release --no-default-features --features=x11
```

If the compiler reports a missing system library, install it via pacman:
```bash
# Read the compiler output — it names the missing library
# Example for GTK3
sudo pacman -S gtk3
```

---

## Error: `eww reload` not working

**Symptom:** `eww reload` runs but config does not update, or logs show "Failed to reload".

**Cause:** A syntax error in the updated config prevents eww from accepting it. eww will not apply a config that fails to parse, even partially.

**Fix:**
```bash
# Check what went wrong
eww logs | tail -20

# Look for parse errors after "Reloading config"
# Fix the syntax error in eww.yuck, then reload again
eww reload

# If reload keeps failing: hard reset
eww kill && eww daemon && eww open bar
```

If a previously-open window was renamed or removed from config:
```bash
# ❌ WRONG — reload fails if "old-bar" no longer exists in config
eww reload && eww open new-bar

# ✅ CORRECT — semicolon continues even if reload fails
eww reload ; eww open new-bar

# ✅ MOST RELIABLE — full reset
eww kill && eww daemon && eww open new-bar
```

---

## Error: Ghost text after label text change in `stacking "bg"` windows

**Symptom:** When a label's `:text` changes (e.g., via ternary expression), the OLD text remains faintly visible underneath the new text. Looks like double-vision or a shadow.

**Cause:** eww windows with `stacking "bg"` (layer-shell background layer) don't properly repaint the region when label text changes. The old frame's pixels persist in the compositor buffer.

**Fix:** Never change label text dynamically in bg-stacked windows. Use one of these alternatives:

```lisp
;; ❌ WRONG — ghost of old text persists
(label :text {active ? "━━━━━━" : "━━▶  ◀━━"})

;; ✅ CORRECT — static text, only change color
(label :class {active ? "tube active" : "tube"}
       :text "━━━━━━━━")

;; ❌ ALSO WRONG — :visible false doesn't clean render either
(label :visible {!active} :text "━━▶  ◀━━")
(label :visible {active}  :text "━━━━━━")
;; Ghost of the hidden label still appears
```

**Note:** This issue does NOT affect `stacking "fg"` or `stacking "overlay"` windows.

---

## Error: `eww active-windows` says 1 bar but 2 bars visible on screen

**Symptom:** You see two eww-bar surfaces on screen, but `eww active-windows` reports only one. `systemctl --user restart eww.service` doesn't fix it.

**Cause:** An orphan eww process (e.g., `eww open widget-name`) from a previous daemon instance survived the daemon restart. It has PPID=1 (adopted by init), maintains its own Wayland connection, and renders its own layer-shell surfaces. The new daemon doesn't know about it.

**Diagnosis:**
```bash
# Look for orphan eww processes with PPID=1
ps -eo pid,ppid,comm,args | grep eww | grep -v grep
# Orphans show ppid=1 and args like "eww open ..."
```

**Fix:** Kill ALL eww processes including orphans, not just the daemon:
```bash
# SIGTERM first, then SIGKILL for stubborn processes
pkill -x eww; sleep 2; pkill -KILL -x eww; sleep 0.5
systemctl --user start eww.service
```

See [SYSTEMD.md](./SYSTEMD.md) for the bulletproof ExecStartPre configuration.

---

## Error: `eww reload` crashes the daemon

**Symptom:** Running `eww reload` returns "Error while forwarding command to server" and all eww windows disappear. After 3 seconds, eww restarts (via systemd Restart=always) but may have duplicate windows.

**Cause:** `eww reload` sends a reload command via IPC. If the config has certain errors or the daemon is in a bad state, the reload crashes the daemon process.

**Fix:** Never use `eww reload` as a casual operation. For reliable restarts:
```bash
# ✅ SAFE — systemd handles full lifecycle
systemctl --user restart eww.service

# ❌ RISKY — can crash daemon and create orphan surfaces
eww reload
```

If `eww reload` crashes, do NOT manually run `open-windows.sh` — let systemd's Restart=always + ExecStartPost handle it automatically. Running both creates duplicate windows.
