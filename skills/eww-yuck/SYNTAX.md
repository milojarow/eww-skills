# Yuck Language Syntax Reference

Complete syntax reference for all yuck constructs.

---

## defwindow

```yuck
(defwindow name [?arg1 ?arg2]
  :monitor 0
  :geometry (geometry :x "0%"
                      :y "0%"
                      :width "100%"
                      :height "30px"
                      :anchor "top center")
  root-widget)
```

### geometry sub-form

| Property | Description | Values |
|---|---|---|
| `:x` | Horizontal position relative to anchor | `"0%"`, `"10px"` |
| `:y` | Vertical position relative to anchor | `"0%"`, `"20px"` |
| `:width` | Window width | `"100%"`, `"400px"` |
| `:height` | Window height | `"30px"`, `"5%"` |
| `:anchor` | Anchor point | see below |

Anchor values:

```
"top left"      "top center"    "top right"
"center left"   "center"        "center right"
"bottom left"   "bottom center" "bottom right"
```

> **CRITICAL — sign convention for right-anchored windows:** `:x` is always
> in screen coordinates (positive = rightward). For `anchor "top right"` or
> `"bottom right"`, a **positive** `:x` value places the window's right edge
> that many pixels **inside** the screen. A negative value pushes it outside.
> Always use positive values to keep the window within the screen bounds.
> The same positive-means-inward rule applies to all anchors:
>
> ```lisp
> ; All four corners, 20px from each edge — all use positive "20px"
> :geometry (geometry :x "20px" :y "20px" :anchor "top left")
> :geometry (geometry :x "20px" :y "20px" :anchor "top right")
> :geometry (geometry :x "20px" :y "20px" :anchor "bottom left")
> :geometry (geometry :x "20px" :y "20px" :anchor "bottom right")
> ```

### monitor property

| Value | Meaning |
|---|---|
| `0` | Monitor index 0 |
| `"HDMI-A-1"` | Monitor by name |
| `"<primary>"` | Primary monitor (may fail on Wayland) |
| `'["<primary>", "HDMI-A-1", 0]'` | JSON fallback array — tries in order |

### X11-specific properties

| Property | Values | Description |
|---|---|---|
| `:stacking` | `"fg"` `"bg"` | Window stack layer |
| `:wm-ignore` | `true` `false` | WM ignores window (for dashboards) |
| `:reserve` | `(struts :distance "40px" :side "top")` | Reserve space in WM layout |
| `:windowtype` | `"normal"` `"dock"` `"toolbar"` `"dialog"` `"desktop"` | WM window type hint; defaults to `"dock"` if `:reserve` is set |

`:reserve` struts sides: `"top"` `"bottom"` `"left"` `"right"`

### Wayland-specific properties

| Property | Values | Description |
|---|---|---|
| `:stacking` | `"fg"` `"bg"` `"overlay"` `"bottom"` | Layer shell layer |
| `:exclusive` | `true` `false` | Compositor auto-reserves space; requires `:anchor` to include `center` |
| `:focusable` | `"none"` `"exclusive"` `"ondemand"` | Keyboard focus policy; must be non-`"none"` for text input |
| `:namespace` | any string | Wayland layersurface namespace |

### Window arguments

```yuck
(defwindow my-bar [arg1 ?arg2]
  :monitor arg1
  :geometry (geometry :height { arg2 ?: "30px" })
  :stacking "fg"
  (bar))
```

```bash
eww open my-bar --id primary --arg arg1=eDP-1
eww open my-bar --id secondary --arg arg1=HDMI-A-1 --arg arg2=40px

# open-many: per-window args
eww open-many my-bar:primary my-bar:secondary \
  --arg primary:arg1=eDP-1 \
  --arg secondary:arg1=HDMI-A-1

# open-many: shared arg (no id prefix)
eww open-many my-bar:primary my-bar:secondary --arg arg2=small
```

Special built-in args: `id` (set by `--id`), `screen` (set by `--screen`).

---

## defwidget

```yuck
(defwidget name [param1 ?optional-param]
  body)
```

- Required param: `[name]`
- Optional param: `[?name]` — defaults to `""` when not provided
- Body must contain **exactly one** root element

### Multiple children

```yuck
; Must wrap in a container
(defwidget my-widget [label]
  (box :orientation "h"   ; single root
    (label :text label)
    (button "click")))
```

### Children placeholder

```yuck
; Receive children passed at call site
(defwidget wrapper [title]
  (box :class "wrapper"
    (label :text title)
    (children)))

; Receive a specific child by index
(defwidget two-col []
  (box
    (box :class "left"  (children :nth 0))
    (box :class "right" (children :nth 1))))
```

Usage:

```yuck
(wrapper :title "My Section"
  (button :onclick "..." "Click"))

(two-col
  (label :text "Left")
  (label :text "Right"))
```

---

## defvar

```yuck
; String
(defvar my-str "hello")

; Number
(defvar my-num 0)

; Boolean
(defvar my-bool false)

; JSON string (must be a valid JSON string value)
(defvar my-json `["a", "b", "c"]`)
(defvar my-obj `{"key": "value"}`)
```

Update from CLI:

```bash
eww update my-str="new value"
eww update my-num=42
eww update my-bool=true
eww update my-json='["x","y"]'
```

Update from a widget onclick:

```yuck
(button :onclick "eww update my-bool=true" "Enable")
(button :onclick "eww update my-bool=${!my-bool}" "Toggle")
```

---

## defpoll

```yuck
(defpoll varname
  :interval "1s"
  :initial "fallback-value"
  :run-while some-variable
  `shell command`)
```

| Property | Required | Description |
|---|---|---|
| `:interval` | Yes | Polling interval. Units: `ms`, `s`, `m`, `h` |
| `:initial` | No | Value before first poll result. Avoids blank display on startup. |
| `:run-while` | No | Variable/expression; polling pauses while false. Default: `true`. |

Interval examples: `"100ms"` `"500ms"` `"1s"` `"5s"` `"1m"` `"1h"`

Shell command uses backtick syntax. Command runs in `sh -c`. JSON output is parsed automatically — access fields with dot notation.

```yuck
; Simple string
(defpoll time :interval "1s" :initial "00:00"
  `date +%H:%M`)

; JSON — access with dot notation in expressions
(defpoll disk :interval "30s" :initial "{}"
  `df -h / | awk 'NR==2{print "{\"used\":\"" $3 "\",\"avail\":\"" $4 "\"}"}'`)
; Usage: {disk.used} / {disk.avail}

; Conditional polling
(defvar show-stats false)
(defpoll cpu-usage :interval "2s" :run-while show-stats
  `top -bn1 | grep "Cpu(s)" | awk '{print $2}'`)
```

Force immediate poll: `eww poll varname`
Update externally: `eww update varname=value` (works on defpoll same as defvar)

---

## deflisten

```yuck
(deflisten varname
  :initial "fallback"
  `long-running-command`)
```

| Property | Required | Description |
|---|---|---|
| `:initial` | No | Value before first stdout line arrives |
| `:onchange` | No | Shell command run each time value updates; `{}` is replaced with the new value |

### Script stdout requirements

The script must:
- Stay running (not exit after one output)
- Print each new value as a separate line to stdout
- Flush stdout after each write (critical for scripts that buffer output)

For Python scripts, use `sys.stdout.flush()` or `python -u`:

```python
#!/usr/bin/env python3
import sys, time
while True:
    print(get_value())
    sys.stdout.flush()
    time.sleep(1)
```

For shell scripts, `echo` always flushes. For tools that buffer, pipe through `stdbuf -oL`.

### Examples

```yuck
; Music player
(deflisten music :initial "Nothing playing"
  `playerctl --follow metadata --format '{{ artist }} - {{ title }}' || true`)

; Workspace list from hyprland/sway IPC
(deflisten workspaces :initial "[]"
  `~/.config/eww/scripts/workspaces.sh`)

; File watcher
(deflisten watched :initial ""
  `tail -F /tmp/state-file`)

; With onchange side effect
(deflisten volume :initial "50"
  :onchange "notify-send 'Volume' '{}%'"
  `pactl subscribe | grep --line-buffered "sink" | while read -r _; do pactl get-sink-volume @DEFAULT_SINK@ | awk '{print $5}' | tr -d '%'; done`)
```

---

## for loop

```yuck
(for element in array-variable
  widget-body)
```

`array-variable` must hold a JSON array string. Each element is bound to `element` for use in `widget-body`.

### Simple array

```yuck
(defvar tags `["web", "code", "media", "misc"]`)

(box :orientation "h"
  (for tag in tags
    (button :onclick "swaymsg workspace ${tag}"
      (label :text tag))))
```

### Array of objects

```yuck
(defvar windows `[{"title": "Firefox", "id": 1}, {"title": "Terminal", "id": 2}]`)

(box :orientation "v"
  (for win in windows
    (button :onclick "swaymsg '[con_id=${win.id}] focus'"
      (label :text "${win.title}"))))
```

### Nested for — NOT SUPPORTED

eww does **not** support nested `for` loops. An inner `(for ...)` inside an outer
`(for ...)` causes a parse/runtime error that prevents the config from loading entirely.

```yuck
; WRONG — nested for causes a config load failure
(for group in groups
  (box :class "group"
    (for item in {group.items}   ; NOT supported — breaks the entire config
      (label :text item))))

; CORRECT — pre-compute grouping in the data source script,
; emit flat named fields, use fixed widgets in yuck
(for ws in workspaces
  (box
    (label :text "${ws.icon_top}")
    (label :text "${ws.icon_mid}" :visible {ws.has_mid})
    (label :text "${ws.icon_bot}" :visible {ws.has_bot})))
```

**Rule:** If you need to render N items per parent item, do the grouping in the
data source script and emit a fixed set of named fields. Never nest `for` in yuck.

**Corollary:** Placing a plain `(label :visible {!condition} ...)` as a direct sibling
of a `(for ...)` inside the same `(box ...)` also causes rendering issues. Avoid mixing
static sibling widgets with a `for` loop — encode all display states into the flat fields.

---

## include

```yuck
; Relative to current file location
(include "./variables.yuck")
(include "./widgets/bar.yuck")
(include "./widgets/powermenu.yuck")
```

- Paths are relative to the file containing the `include` directive
- Included files can themselves include other files
- All definitions (defwindow, defwidget, defvar, etc.) become globally available

Recommended structure:

```
~/.config/eww/
├── eww.yuck             ; root: only include directives
├── eww.scss             ; root: only @import directives
├── variables.yuck
├── widgets/
│   ├── bar.yuck
│   └── powermenu.yuck
└── styles/
    ├── bar.scss
    └── powermenu.scss
```

---

## literal

```yuck
(literal :content variable-or-expression)
```

Renders a string as a yuck widget tree. The string must contain a single valid yuck widget expression.

```yuck
(defvar popup-content "(label :text \"Hello\")")
(literal :content popup-content)

; With deflisten for script-driven structures
(deflisten notif-widgets :initial "(box)"
  `~/.config/eww/scripts/notifications.sh`)
(literal :content notif-widgets)
```

Use sparingly — re-parses and re-renders on every value change. Prefer `for` for lists.

---

## String interpolation

### `${ }` — inside a string

Use when embedding an expression within a larger string:

```yuck
(label :text "Hello ${name}")
(label :text "Battery: ${battery}%")
(label :text "Time: ${formattime(EWW_TIME, "%H:%M")}")
(button :onclick "notify-send 'Hello' 'Hi, ${name}'"
  "Greet")
```

### `{ }` — standalone expression

Use when the entire attribute value is an expression (not embedded in a string):

```yuck
(scale :value {EWW_RAM.used_mem_perc})
(button :active {count > 0})
(label :text {battery})
(box :class {active ? "on" : "off"})
```

### Common mistake

```yuck
; Wrong — string where number is expected
(scale :value "${battery}")

; Correct
(scale :value {battery})
```

---

## Comments

```yuck
; This is a line comment
; Comments use semicolons, same as Lisp/Scheme

(defwidget bar []
  ; This widget is the main bar
  (box :orientation "h"
    (label :text "hello")))   ; inline comment
```

There are no block comments in yuck.
