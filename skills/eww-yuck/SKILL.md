---
name: eww-yuck
description: Core eww yuck configuration syntax. Use when writing eww.yuck config files, defining windows with defwindow, creating widgets with defwidget, declaring variables with defvar/defpoll/deflisten, structuring layouts with include or for loops, or setting up a new eww installation.
metadata:
  priority: 7
  pathPatterns: ["**/*.yuck", "**/eww/*.yuck"]
  bashPatterns: ["eww\\s+(daemon|reload|open|close|kill|list-windows|active-windows|state)"]
---

# eww-yuck — Core Configuration

eww (ElKowar's Wacky Widgets) is a widget system written in Rust, independent of your window manager. Config uses **yuck** (S-expressions) + **CSS/SCSS**.

---

## Quick Start

The minimal path from zero to a working bar:

```bash
# 1. Start the daemon
eww daemon

# 2. Open a window by name
eww open bar

# 3. Reload after edits (no restart needed)
eww reload

# 4. Stop daemon
eww kill
```

Minimal working bar — create `~/.config/eww/eww.yuck`:

```yuck
(defwidget bar []
  (centerbox :orientation "h"
    (label :text "Left")
    (label :text "Center")
    (label :text "Right")))

(defwindow bar
  :monitor 0
  :geometry (geometry :x "0%" :y "0%"
                      :width "100%" :height "30px"
                      :anchor "top center")
  :stacking "fg"
  :exclusive true
  (bar))
```

Also create `~/.config/eww/eww.scss` (can be empty to start):

```scss
/* styles go here */
```

Then run:

```bash
eww daemon && eww open bar
```

---

## Installation

### CachyOS / Arch (recommended)

```bash
paru -S eww-wayland   # Wayland
paru -S eww-x11       # X11
```

### Build from source

Prerequisites (Arch package names):

- `gtk3` — libgdk-3, libgtk-3
- `gtk-layer-shell` — Wayland only
- `pango`, `gdk-pixbuf2`, `libdbusmenu-gtk3`
- `cairo`, `glib2`, `gcc-libs`, `glibc`
- `rustup` (for `rustc` + `cargo`)

```bash
git clone https://github.com/elkowar/eww
cd eww

# Wayland
cargo build --release --no-default-features --features=wayland

# X11
cargo build --release --no-default-features --features=x11
```

Make the binary executable and place it in PATH:

```bash
chmod +x target/release/eww
cp target/release/eww ~/.local/bin/eww
```

### Config path

```
~/.config/eww/
├── eww.yuck     # main config (required)
└── eww.scss     # styles (required, can be empty)
```

Eww reads `$XDG_CONFIG_HOME/eww`, which defaults to `~/.config/eww`.

---

## defwindow

`defwindow` declares a top-level window. Each window has a name, geometry, display properties, and exactly one root widget.

```yuck
(defwindow name
  :monitor 0
  :geometry (geometry :x "0%"
                      :y "0%"
                      :width "100%"
                      :height "30px"
                      :anchor "top center")
  ; properties depend on X11 or Wayland (see table below)
  root-widget-call)
```

### geometry sub-form

| Property | Description | Example |
|---|---|---|
| `:x` | Horizontal position relative to anchor | `"0%"`, `"10px"` |
| `:y` | Vertical position relative to anchor | `"0%"`, `"20px"` |
| `:width` | Window width | `"100%"`, `"400px"` |
| `:height` | Window height | `"30px"`, `"5%"` |
| `:anchor` | Anchor point for position | `"top center"` |

Anchor values: `"top left"` `"top center"` `"top right"` `"center left"` `"center"` `"center right"` `"bottom left"` `"bottom center"` `"bottom right"`

### monitor property

| Value | Meaning |
|---|---|
| `0`, `1` | Monitor index |
| `"eDP-1"`, `"HDMI-A-1"` | Monitor name |
| `"<primary>"` | Primary display (may fail on Wayland) |
| `'["<primary>", "HDMI-A-1", 0]'` | JSON fallback array — eww tries in order |

### X11 vs Wayland properties

| Property | X11 | Wayland | Description |
|---|---|---|---|
| `:stacking` | `"fg"` or `"bg"` | `"fg"` `"bg"` `"overlay"` `"bottom"` | Window layer |
| `:exclusive` | — | `true`/`false` | Compositor auto-reserves space; anchor must include `center` |
| `:focusable` | — | `"none"` `"exclusive"` `"ondemand"` | Keyboard focus policy |
| `:namespace` | — | string | Wayland layersurface namespace |
| `:wm-ignore` | `true`/`false` | — | WM ignores this window (dashboards) |
| `:reserve` | `(struts :distance "30px" :side "top")` | — | Reserve space in WM layout |
| `:windowtype` | `"normal"` `"dock"` `"toolbar"` `"dialog"` `"desktop"` | — | Hint to WM; defaults to `"dock"` if `:reserve` is set |

### defwindow with window arguments

Windows can declare arguments the same way defwidget does:

```yuck
(defwindow my-bar [monitor-name ?size]
  :monitor monitor-name
  :geometry (geometry :width "100%"
                      :height { size ?: "30px" }
                      :anchor "top center")
  :stacking "fg"
  :exclusive true
  (bar))
```

```bash
eww open my-bar --id bar-primary --arg monitor-name=eDP-1
eww open my-bar --id bar-secondary --arg monitor-name=HDMI-A-1 --arg size=40px
```

Arguments are **constant** — they cannot be updated after the window opens.

---

## defwidget

`defwidget` defines a reusable widget. Widgets are S-expressions that can take parameters and call other widgets or built-ins.

```yuck
(defwidget name [required-param ?optional-param]
  body)
```

- `required-param` — must be provided at call site
- `?optional-param` — optional; defaults to `""` if omitted
- Body must be **exactly one** root widget — wrap multiple children in `box`

### Basic example

```yuck
(defwidget greeter [?text name]
  (box :orientation "h" :halign "center"
    text
    (button :onclick "notify-send 'Hello' 'Hello, ${name}'"
      "Greet")))

; Usage in defwindow or another widget:
(greeter :text "Say hello!"
         :name "Tim")
```

### Widget accepting children

Use `(children)` as a placeholder to receive nested content at call site:

```yuck
(defwidget labeled-container [label]
  (box :class "container"
    label
    (children)))

; Usage:
(labeled-container :label "My Section"
  (button :onclick "notify-send hey" "Click me"))
```

Access a specific child by index with `:nth`:

```yuck
(defwidget two-col []
  (box
    (box :class "left"  (children :nth 0))
    (box :class "right" (children :nth 1))))

; Usage:
(two-col
  (label :text "Left content")
  (label :text "Right content"))
```

### Referencing parameters

Parameters are referenced by name — no `${}` needed inside the widget body when used as a direct value:

```yuck
(defwidget my-label [text]
  (label :text text))   ; direct reference, not "${text}"
```

Inside a string attribute, use `${}` interpolation:

```yuck
(defwidget my-button [name]
  (button :onclick "notify-send 'Hi' 'Hello, ${name}'"
    "Greet"))
```

---

## Variable Types — Decision Guide

| Situation | Use |
|---|---|
| Value changes rarely, updated by external scripts or buttons | `defvar` |
| Value needs to be refreshed on a schedule (time, battery, weather) | `defpoll` |
| Value changes based on events from a long-running process (music, workspaces, volume) | `deflisten` |
| System data (CPU, RAM, battery) already provided | magic variable (`EWW_CPU`, `EWW_RAM`, etc.) |

**Prefer `deflisten` when you have an event-driven data source** — it is the most efficient option. Use `defpoll` only when no such source exists and you must resort to polling.

---

## defvar

`defvar` declares a static variable with an initial value. It never changes automatically — you update it externally.

```yuck
(defvar my-var "initial value")
(defvar count 0)
(defvar panel-open false)
(defvar items `["a", "b", "c"]`)
```

Update from the shell:

```bash
eww update my-var="new value"
eww update count=42
eww update panel-open=true
```

Update from a widget button:

```yuck
(button :onclick "eww update panel-open=true"
  "Open Panel")
```

**Common pattern — toggle:**

```yuck
(defvar panel-open false)

(button :onclick "eww update panel-open=${!panel-open}"
  "Toggle")
```

---

## defpoll

`defpoll` runs a shell command on a fixed interval and updates the variable with the output.

```yuck
(defpoll varname
  :interval "1s"
  :initial "fallback"
  :run-while some-condition
  `shell command here`)
```

### All properties

| Property | Required | Description |
|---|---|---|
| `:interval` | Yes | How often to run. Units: `ms`, `s`, `m`, `h`. Examples: `"500ms"`, `"1s"`, `"10m"` |
| `:initial` | No | Value shown before the first poll completes. Speeds up startup significantly. |
| `:run-while` | No | Variable or expression — polling only runs while this evaluates to true. Defaults to `true`. |

### Examples

> CRITICAL: For displaying the current time, always use `EWW_TIME` with `formattime()` — **not** a `defpoll` that polls `date`. `EWW_TIME` is a built-in magic variable updated every 1 second with no script required. A `defpoll` that runs `date` every second is wasteful and unnecessary.
>
> ✅ CORRECT: `(label :text {formattime(EWW_TIME, "%H:%M")})`
> ❌ INEFFICIENT: `(defpoll time :interval "1s" \`date +%H:%M\`)`

```yuck
; Clock — poll every second
(defpoll time :interval "1s"
              :initial "00:00"
  `date +%H:%M`)

; Battery — poll every minute with initial value
(defpoll battery :interval "1m"
                 :initial "100"
  `cat /sys/class/power_supply/BAT0/capacity`)

; JSON output — access fields with dot notation
(defpoll weather :interval "10m"
                 :initial "{}"
  `curl -s "wttr.in/?format=j1" | jq '{temp: .current_condition[0].temp_C}'`)
; Usage: {weather.temp}

; Run only while a panel is open
(defvar panel-visible false)

(defpoll panel-data :interval "5s"
                    :run-while panel-visible
  `fetch-panel-data.sh`)
```

> **Note:** `defpoll` only runs while a window using it is open. To poll even when no widget is visible, use `:run-while true`. `eww poll varname` triggers an immediate poll outside the normal schedule.

---

## deflisten

`deflisten` runs a script once and reads its stdout continuously. Each new line becomes the new value of the variable.

```yuck
(deflisten varname
  :initial "fallback"
  `long-running-command`)
```

### All properties

| Property | Required | Description |
|---|---|---|
| `:initial` | No | Value before the first line is received from the script |
| `:onchange` | No | Shell command to run each time the value updates. Use `{}` as placeholder for the new value. |

### Script requirements

The script must:
- Run as a long-lived process (not exit immediately)
- Print new values to **stdout**, one value per line
- **Flush stdout after each write** (critical — eww reads lines, not buffered output)

If the script exits, the variable stops updating. Use `|| true` or a wrapper to handle restarts.

### Examples

```yuck
; Music player — updates whenever track changes
(deflisten music :initial ""
  `playerctl --follow metadata --format '{{ artist }} - {{ title }}' || true`)

; Workspaces from a subscription script
(deflisten workspaces :initial "[]"
  `~/.config/eww/scripts/subscribe-workspaces`)

; Tail a file for live log output
(deflisten log-line :initial ""
  `tail -F /tmp/app.log`)

; Trigger side effect on every update
(deflisten music :initial ""
  :onchange "notify-send 'Now playing' '{}'"
  `playerctl --follow metadata --format '{{ title }}' || true`)
```

### Why deflisten is preferred over defpoll

`deflisten` is event-driven — it reacts instantly when data changes and uses no CPU between events. `defpoll` polls on an interval even when nothing has changed. For values like volume, brightness, workspaces, and currently playing music, always prefer `deflisten` if you have an event-driven data source.

---

## for — Iterate over JSON array

`for` generates a widget for each element in a JSON array variable.

```yuck
(for element in array-variable
  widget-body)
```

### Iterating a simple array

```yuck
(defvar items `["alpha", "beta", "gamma"]`)

(box :orientation "v"
  (for item in items
    (label :text item)))
```

### Iterating an array of objects

```yuck
(defvar workspaces `[{"name": "1", "focused": true}, {"name": "2", "focused": false}]`)

(box
  (for ws in workspaces
    (button :class {ws.focused ? "ws-focused" : "ws-inactive"}
            :onclick "swaymsg workspace ${ws.name}"
      (label :text "${ws.name}"))))
```

### for vs literal

`for` is the preferred way to render dynamic lists. Use `literal` only when you need to generate an entirely different widget structure at runtime (not just different values within the same structure).

❌ WRONG — using literal for a list:
```yuck
(defvar items-yuck "(box (label :text 'a') (label :text 'b'))")
(literal :content items-yuck)
```

✅ CORRECT — using for for a list:
```yuck
(defvar items `["a", "b"]`)
(box
  (for item in items
    (label :text item)))
```

---

## include — Modular config

Split your config across multiple files as it grows. `include` imports a yuck file at that point in the config.

```yuck
; ~/.config/eww/eww.yuck — root entry point
(include "./variables.yuck")
(include "./widgets/bar.yuck")
(include "./widgets/powermenu.yuck")
(include "./widgets/dashboard.yuck")
```

Recommended directory layout:

```
~/.config/eww/
├── eww.yuck             ; root — only include directives
├── eww.scss             ; root — only @import directives
├── variables.yuck       ; all defvar / defpoll / deflisten
├── widgets/
│   ├── bar.yuck
│   ├── powermenu.yuck
│   └── dashboard.yuck
└── styles/
    ├── bar.scss
    └── powermenu.scss
```

SCSS mirrors the same pattern:

```scss
/* eww.scss */
@import "./styles/bar";
@import "./styles/powermenu";
@import "./styles/dashboard";
```

---

## literal — Dynamic widget markup

`literal` renders a string as a yuck widget tree at runtime. Use when the widget structure itself must change dynamically (not just the values).

```yuck
(defvar dynamic-content "(label :text \"hello\")")
(literal :content dynamic-content)
```

When `dynamic-content` changes, the entire widget is re-rendered from the new string.

**Combine with deflisten for script-driven structures:**

```yuck
(deflisten notification-widgets :initial "(box)"
  `~/.config/eww/scripts/notification-listener`)

(literal :content notification-widgets)
```

> Use sparingly — `literal` re-parses and re-renders on every update. Prefer `for` for lists of the same widget type.

---

## String interpolation

Two interpolation forms exist in yuck:

| Form | Context | Example |
|---|---|---|
| `${ }` | Inside a string literal | `"Time: ${time}"` |
| `{ }` | As a standalone attribute value (not in a string) | `:value {EWW_RAM.used_mem_perc}` |

```yuck
; ${ } — embed expression inside a string
(label :text "Hello, ${name}! Battery: ${battery}%")

; { } — pure expression as attribute value
(scale :value {EWW_RAM.used_mem_perc}
       :min 0
       :max 100)

; Ternary inside { }
(button :class {active ? "active" : "inactive"} "Click")

; JSON field access
(label :text {EWW_BATTERY["BAT0"].capacity})
```

❌ WRONG — `${}` as standalone value:
```yuck
(scale :value "${battery}")    ; string, not a number
```

✅ CORRECT — `{}` as standalone expression:
```yuck
(scale :value {battery})
```

---

## CRITICAL Rules

1. **Every `defwindow` must have exactly one root widget.** Wrap multiple children in `box`.

2. **`${ }` is for interpolation inside strings. `{ }` is for pure expressions as attribute values.** Mixing them causes silent wrong behavior or type errors.

3. **`defpoll` only runs while the window is open.** Add `:run-while true` explicitly if you need it to keep polling when the window is not visible.

4. **`deflisten` scripts must flush stdout.** If the script buffers output (e.g., a Python script without `sys.stdout.flush()`), the variable will never update. Add explicit flushes or run with `python -u`.

5. **Window arguments are constants.** They are set once at `eww open` and cannot be changed with `eww update`. Use `defvar` for anything that needs to change after the window is open.

6. **Both `deflisten` and `defpoll` only run when their variable is referenced in an open window's widget tree.** A variable whose name never appears in any widget attribute is treated as dead code — its script will never start. This is the silent failure mode for side-effect-only deflistens (scripts that call `eww close`/`eww open` internally but whose variable is never displayed anywhere). Diagnostic: run `eww state` — if the variable is completely absent, the script is not running at all.

```yuck
; ❌ WRONG — no widget references fullscreen-active; script never starts
(deflisten fullscreen-active :initial "false"
  `~/.config/eww/scripts/fullscreen-subscribe.sh`)

; ✅ CORRECT — hidden label in a persistent window anchors the variable
(deflisten fullscreen-active :initial "false"
  `~/.config/eww/scripts/fullscreen-subscribe.sh`)
; In the always-open bar widget:
(label :visible false :text {fullscreen-active})
```

---

## Integration with Other Skills

- **eww-expressions** — writing `${}` expressions, operators, functions like `formattime()`, `round()`, JSON access, ternary expressions, and the full expression language reference
- **eww-widgets** — all built-in widgets (`box`, `label`, `button`, `scale`, `centerbox`, etc.) and magic variables (`EWW_RAM`, `EWW_CPU`, `EWW_BATTERY`, `EWW_TEMPS`, etc.)
- **eww-styling** — GTK CSS/SCSS, theming, `all: unset`, font setup, the eww inspector, and what CSS properties GTK supports
- **eww-patterns** — complete working examples: bars, power menus, popups, workspace indicators, notification centers, multi-monitor setups
- **eww-troubleshooting** — debugging with `eww logs`, `eww debug`, `eww state`, fixing parse errors, variable not updating, window not appearing

---

## Summary

eww uses three building blocks:

1. **`defwindow`** — creates a window with position, size, and display properties
2. **`defwidget`** — defines reusable widget components
3. **Variables** — provide dynamic data: `defvar` (static), `defpoll` (interval), `deflisten` (event-driven)

The entry point is `~/.config/eww/eww.yuck`. Split with `include` as config grows. Style with `~/.config/eww/eww.scss`.

Reference files in this skill:
- [SYNTAX.md](SYNTAX.md) — complete yuck syntax with all properties
- [CLI.md](CLI.md) — all eww CLI commands and workflows
