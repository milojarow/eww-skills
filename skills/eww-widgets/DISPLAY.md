---
name: eww-widgets
description: Choose and configure eww widgets. Use when picking between box/centerbox/overlay/scroll/stack, using button/checkbox/input/scale, displaying data with label/image/progress/circular-progress/graph, using systray or revealer animations, or accessing EWW_RAM/EWW_CPU/EWW_BATTERY/EWW_DISK/EWW_NET magic variables.
---

# Display & Output Widgets

Deep reference for `label`, `image`, `progress`, `circular-progress`, `graph`, `systray`, `revealer`, `expander`, `transform`, `literal`, `tooltip`, and `calendar`.

---

## label — Text Display

The primary text widget. Supports plain text, Pango markup, rotation, alignment, and truncation.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:text` | string | — | Plain text to display (supports `${expr}` interpolation) |
| `:markup` | string | — | Pango markup string; use instead of `:text` for rich text |
| `:truncate` | bool | — | Enable truncation when text overflows (show ellipsis) |
| `:truncate-left` | bool | `false` | Truncate from the left side instead of the right |
| `:limit-width` | int | — | Maximum character count before truncation |
| `:show-truncated` | bool | `true` | Show ellipsis indicator; set false to disable dynamic truncation |
| `:wrap` | bool | `false` | Wrap text to multiple lines (works best with a fixed `:width`) |
| `:wrap-mode` | string | `"word"` | How text wraps: `"word"` `"char"` `"wordchar"` |
| `:lines` | int | `-1` | Max line count (requires `:limit-width`; `-1` = unlimited) |
| `:angle` | float | `0.0` | Rotation angle in degrees (0–360) |
| `:gravity` | string | `"auto"` | Text gravity: `"south"` `"east"` `"west"` `"north"` `"auto"` |
| `:xalign` | float | `0.5` | Text alignment on x axis (0=left, 0.5=center, 1=right) |
| `:yalign` | float | `0.5` | Text alignment on y axis (0=bottom, 0.5=center, 1=top) |
| `:justify` | string | `"left"` | Justify multi-line text: `"left"` `"right"` `"center"` `"fill"` |
| `:unindent` | bool | `false` | Remove leading whitespace |

Plus all universal properties.

### Examples

```yuck
; Simple text with expression
(label :text "CPU: ${round(EWW_CPU.avg, 0)}%")
```

```yuck
; Truncated window title — max 30 chars
(label :text {active-window-title}
       :limit-width 30
       :truncate true
       :class "window-title")
```

```yuck
; Pango markup for rich text (bold, colored)
(label :markup "<b>Status:</b> <span color='#a6e3a1'>OK</span>")
```

```yuck
; Rotated label for a vertical bar
(label :text "SYSTEM"
       :angle 270
       :class "vertical-label")
```

```yuck
; Nerd Font icon glyph
(label :text "" :class "icon nf-battery")
```

> `:text` and `:markup` are mutually exclusive — use one or the other. `:text` escapes HTML entities automatically; `:markup` does not.

---

## image — Image and Icon Display

Displays raster images (PNG, JPEG), SVG files, or GTK theme icons.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:path` | string | — | Absolute path to image file |
| `:image-width` | int | — | Width to render the image (px) |
| `:image-height` | int | — | Height to render the image (px) |
| `:preserve-aspect-ratio` | bool | `true` | Keep aspect ratio when resizing (does not work for all formats) |
| `:fill-svg` | string | — | Fill color for SVG images (hex string, e.g., `"#cba6f7"`) |
| `:icon` | string | — | GTK theme icon name (alternative to `:path`) |
| `:icon-size` | string | — | Size of theme icon: `"menu"` `"small-toolbar"` `"toolbar"` `"large-toolbar"` `"button"` `"dnd"` `"dialog"` |

Plus all universal properties.

### Examples

```yuck
; Raster image with fixed dimensions
(image :path "${EWW_CONFIG_DIR}/icons/avatar.png"
       :image-width 48
       :image-height 48)
```

```yuck
; SVG icon with theme-matching fill color
(image :path "${EWW_CONFIG_DIR}/icons/battery.svg"
       :image-width 20
       :image-height 20
       :fill-svg "#cba6f7")
```

```yuck
; GTK theme icon (uses system icon theme)
(image :icon "audio-volume-high"
       :icon-size "large-toolbar")
```

> Use `EWW_CONFIG_DIR` for all image paths to keep configs portable. Hardcoded `/home/user/...` paths break when the config is used on a different machine.

---

## progress — Linear Progress Bar

A horizontal or vertical bar showing a value from 0 to 100.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:value` | float | — | Current value (0–100) |
| `:orientation` | string | `"h"` | Direction: `"h"` `"horizontal"` `"v"` `"vertical"` |
| `:flipped` | bool | `false` | Flip fill direction |

Plus all universal properties.

### CSS Note

`progress` maps to the GTK `progressbar` widget. To control the width of a horizontal bar via CSS or `:width`, you must also set `min-width` on the inner trough:

```scss
progressbar > trough {
  min-width: 100px;
}
```

### Examples

```yuck
(progress :value {EWW_RAM.used_mem_perc}
          :orientation "h"
          :class "ram-bar")
```

```yuck
; Vertical progress bar (e.g., volume level)
(progress :value {volume}
          :orientation "v"
          :flipped true   ; fill from bottom
          :class "vol-bar")
```

---

## circular-progress — Ring Meter

A circular arc showing progress from 0 to 100. Can have a child widget centered inside it.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:value` | float | — | Current value (0–100) |
| `:start-at` | float | `0` | Starting angle as percentage (0 = top, 25 = right, 50 = bottom, 75 = left) |
| `:thickness` | float | — | Stroke width in px. Very high values produce a filled disc (pie chart effect) |
| `:clockwise` | bool | `true` | Fill direction |

Plus all universal properties. Accepts one optional child (rendered centered inside the ring).

### Examples

```yuck
; CPU ring with centered label
(overlay
  (circular-progress :value {EWW_CPU.avg}
                     :thickness 4
                     :clockwise true
                     :start-at 75    ; start at left (270 degrees)
                     :class "cpu-ring"
                     :width 48
                     :height 48)
  (label :text "${round(EWW_CPU.avg, 0)}%"
         :class "ring-label"
         :halign "center"
         :valign "center"))
```

```yuck
; Battery ring starting at the top
(circular-progress :value {EWW_BATTERY.BAT0.capacity}
                   :thickness 6
                   :clockwise true
                   :start-at 0
                   :class "bat-ring")
```

> `circular-progress` does not natively center its child — wrap it in `overlay` with the child having `:halign "center" :valign "center"` to achieve a centered label inside the ring.

### Real Usage

**Filled-disc system meters** — using `:thickness 100` produces a fully filled disc (pie) rather than a ring. Three meters (battery, CPU, memory) are grouped vertically with a label below each one. The child label inside `circular-progress` acts as an overlay; `:show_truncated false` prevents ellipsis on single-character labels.

```yuck
; Source: saimoomedits — eww/leftbar/eww.yuck
(defwidget system []
  (box :class "sys_win" :orientation "h" :space-evenly "false" :spacing 13
    (box :class "sys_bat_box" :orientation "v" :space-evenly "false"
      (circular-progress :value battery
                         :class "sys_bat"
                         :thickness 100
        (label :text " "
               :class "cc_cc"
               :limit-width 2
               :show_truncated false
               :wrap false))
      (label :text "BAT" :class "sys_icon_bat" :limit-width 4))
    (box :class "sys_cpu_box" :orientation "v" :space-evenly "false"
      (circular-progress :value cpu
                         :class "sys_cpu"
                         :thickness 100
        (label :text " " :class "cc_cc" :limit-width 2 :show_truncated false :wrap false))
      (label :text "CPU" :class "sys_icon_cpu" :limit-width 4))
    (box :class "sys_mem_box" :orientation "v" :space-evenly "false"
      (circular-progress :value memory
                         :class "sys_mem"
                         :thickness 100
        (label :text " " :class "cc_cc" :limit-width 2 :show_truncated false :wrap false))
      (label :text "MEM" :class "sys_icon_mem" :limit-width 4))))
```

---

## graph — Time-Series Graph

Plots a value over time, maintaining a history automatically. Useful for CPU, RAM, or network sparklines.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:value` | float | — | Current value (0–100 by default, or use `:min`/`:max`) |
| `:thickness` | float | — | Line stroke width (px) |
| `:time-range` | duration | — | How far back to show (e.g., `"30s"`, `"2m"`) |
| `:min` | float | `0` | Minimum y-axis value |
| `:max` | float | — | Maximum y-axis value |
| `:dynamic` | bool | `false` | Auto-scale y-axis to fit actual data range |
| `:line-style` | string | `"miter"` | Edge style: `"miter"` `"round"` `"bevel"` |
| `:flip-x` | bool | `false` | Reverse x-axis direction (oldest on left vs right) |
| `:flip-y` | bool | `false` | Invert y-axis |
| `:vertical` | bool | `false` | Exchange x and y axes |

Plus all universal properties.

### Examples

```yuck
; CPU usage over last 60 seconds
(graph :value {EWW_CPU.avg}
       :time-range "60s"
       :min 0 :max 100
       :thickness 2
       :class "cpu-graph"
       :width 100 :height 40)
```

```yuck
; Network download with dynamic scale
(graph :value {round(EWW_NET.wlan0.down / 1024, 0)}
       :time-range "30s"
       :dynamic true
       :thickness 1.5
       :line-style "round"
       :class "net-graph")
```

> `graph` accumulates history from the moment eww starts. On first open, the graph will be partially empty until `:time-range` worth of data has been collected.

---

## systray — System Tray Icon Container

Embeds the system notification area (tray icons) from running applications.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:spacing` | int | `0` | Pixels between tray icons |
| `:orientation` | string | `"h"` | Layout direction: `"h"` `"v"` |
| `:space-evenly` | bool | `true` | Distribute icons evenly |
| `:icon-size` | int | — | Size of icons in pixels |
| `:prepend-new` | bool | `false` | Add new icons at the start (left/top) instead of end |

Plus all universal properties.

### Examples

```yuck
(systray :spacing 4
         :orientation "h"
         :space-evenly false
         :icon-size 18
         :prepend-new false
         :class "systray")
```

> Only one `systray` widget should be active at a time. Having multiple tray widgets can cause icon duplication or missing icons.

---

## revealer — Animated Show/Hide

Wraps a single child and reveals or hides it with an animation.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:reveal` | bool | `false` | Whether the child is currently revealed |
| `:transition` | string | `"none"` | Animation type: `"slideright"` `"slideleft"` `"slideup"` `"slidedown"` `"crossfade"` `"none"` |
| `:duration` | duration | `"500ms"` | Length of the reveal/hide animation |

Plus all universal properties.

### Behavior

- When `:reveal` changes from false to true, the enter animation plays; from true to false, the exit animation plays (reverse).
- The `slidedown` transition slides content down into view — good for dropdown panels.
- The `crossfade` transition fades in/out without a positional movement.
- Content is not destroyed when hidden — it is just not rendered. Variables still update.

### Examples

```yuck
(defvar show-panel false)

; Toggle button
(button :onclick "eww update show-panel=${!show-panel}"
  (label :text {if show-panel then "" else ""}))

; Revealed panel
(revealer :reveal show-panel
          :transition "slidedown"
          :duration "300ms"
  (box :orientation "v" :class "panel"
    (label :text "Panel content")))
```

```yuck
; Hover-triggered reveal
(eventbox :onhover     "eww update hover=true"
          :onhoverlost "eww update hover=false"
  (box :orientation "v"
    (label :text "Hover over me")
    (revealer :reveal hover
              :transition "slidedown"
              :duration "200ms"
      (label :text "I appear on hover!"))))
```

### Real Usage

**Dual-revealer hovered-sign** — two revealers toggling opposite states of the same boolean. When `var` is false, children[0] shows; when `var` is true, children[1] shows. Both use fast 100ms `slideleft` transitions to create a swap effect without layout shift. Used inside `revealer-on-hover` to swap a collapsed icon for an expanded one.

```yuck
; Source: owenrumney / druskus20 — revealer-hover-module
(defwidget hovered-sign [var]
  (box :space-evenly false
    (revealer :reveal {!var}
              :duration "100ms"
              :transition "slideleft"
      (children :nth 0))   ; shown when NOT hovered
    (revealer :reveal {var}
              :duration "100ms"
              :transition "slideleft"
      (children :nth 1)))) ; shown when hovered
```

**Notification revealer driven by deflisten** — a `deflisten` variable carries a JSON object `{"show": bool, "content": "yuck-string"}`. The revealer's `:reveal` binds to the `.show` field; the inner `literal` renders the `.content` field as a widget. This decouples notification display from any polling cycle — the shell script pushes updates only when notifications arrive.

```yuck
; Source: druskus20 — notification-revealer/eww.yuck
(deflisten notifications_listen :initial '{"show": false, "content": ""}' "./notifications.sh")

(defwidget notification-revealer []
  (box :class "notification-revealer"
       :orientation "h"
       :space-evenly false
    "NOTIFICATIONS: "
    (revealer :reveal {notifications_listen.show}
              :transition "slideright"
      (box
        (literal :valign "center" :content {notifications_listen.content})))))
```

---

## expander — Collapsible Section

A native GTK expander widget with a clickable arrow to show/hide its child.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:name` | string | — | Label text shown on the expander header |
| `:expanded` | bool | `false` | Whether the child is visible |

Plus all universal properties.

### Examples

```yuck
(expander :name "Advanced Settings"
          :expanded false
  (box :orientation "v" :spacing 4
    (label :text "Option 1")
    (label :text "Option 2")))
```

> Unlike `revealer`, `expander` has its own built-in toggle UI (the arrow). You do not need a separate button. Use `revealer` when you want custom toggle controls; use `expander` for standard collapsible sections.

---

## transform — 2D Transformation

Applies CSS-like transformations to its child: rotation, translation, and scaling. Transformations are applied in order: rotate → translate → scale.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:rotate` | float | — | Rotation in degrees |
| `:translate-x` | string | — | X translation in `px` or `%` |
| `:translate-y` | string | — | Y translation in `px` or `%` |
| `:scale-x` | string | — | X scale factor in `px` or `%` |
| `:scale-y` | string | — | Y scale factor in `px` or `%` |
| `:transform-origin-x` | string | — | X coordinate of transformation origin in `px` or `%` |
| `:transform-origin-y` | string | — | Y coordinate of transformation origin in `px` or `%` |

Plus all universal properties.

### Examples

```yuck
; Rotate a Nerd Font arrow icon based on panel state
(transform :rotate {if show-panel then 180 else 0}
  (label :text "" :class "chevron"))
```

```yuck
; Scale and center an element
(transform :scale-x "150%"
           :scale-y "150%"
           :transform-origin-x "50%"
           :transform-origin-y "50%"
  (label :text "BIG"))
```

---

## literal — Dynamic Yuck from a Variable

Renders a yuck widget string stored in a variable. Useful for dynamically changing the entire widget structure, not just its data.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:content` | string | — | Yuck markup string to render as a widget |

Plus all universal properties.

### Examples

```yuck
(defvar current-widget "(label :text \"Loading...\")")

; Changes the entire rendered widget when current-widget changes
(literal :content current-widget)
```

```yuck
; Dynamic widget from a script
(defpoll widget-content :interval "5s"
  `scripts/get-widget.sh`)

(literal :content widget-content)
```

> `literal` is powerful but can be harder to debug. Prefer data-driven widgets (binding variables to `:text`, `:value`, etc.) when possible. Reserve `literal` for cases where the widget structure itself needs to change dynamically.

### Real Usage

**Search results rendered as dynamic widgets** — a search script writes yuck markup to stdout; the result is stored in `search_listen` (a `deflisten` variable). `literal` renders whatever widget structure the script returns, allowing the results list to change shape entirely (e.g., zero results vs. a list of buttons) without reloading eww.

```yuck
; Source: isparsh — eww/eww_widgets.yuck
(deflisten search_listen "./scripts/search-listen.sh")

(defwidget searchapps []
  (eventbox :onhoverlost "eww close searchapps"
    (box :orientation "v" :space-evenly false :class "search-win"
      (box :orientation "h" :space-evenly false :class "searchapps-bar"
        (label :class "search-label" :text "")
        (input :class "search-bar" :onchange "~/.config/eww/scripts/search.sh {}"))
      (literal :halign "center" :valign "center"
               :class "app-container"
               :content search_listen))))
```

**Notification content from deflisten** — the `.content` field of a JSON object is a yuck string produced by the notification daemon script. `literal` renders it inline inside a `revealer`, so the notification widget can display arbitrary markup (icon + text, multi-line, etc.) without any fixed structure in the yuck file.

```yuck
; Source: druskus20 — notification-revealer/eww.yuck
(deflisten notifications_listen :initial '{"show": false, "content": ""}' "./notifications.sh")

(revealer :reveal {notifications_listen.show}
          :transition "slideright"
  (box
    (literal :valign "center" :content {notifications_listen.content})))
```

---

## tooltip — Custom Tooltip Widget

Wraps content with a custom widget tooltip (instead of just plain text). The first child is the tooltip content; the second child is the widget that triggers the tooltip.

### Properties

No widget-specific properties beyond universal properties.

### Examples

```yuck
; Custom tooltip showing detailed RAM info
(tooltip
  (box :orientation "v" :spacing 2 :class "tooltip-box"
    (label :text "Used: ${round(EWW_RAM.used_mem_perc, 0)}%")
    (label :text "Free: ${formatbytes(EWW_RAM.free_mem * 1024)}"))
  (label :text "󰍛" :class "icon"))  ; this is what you see normally
```

> The first child is the tooltip content (hidden until hover). The second child is the visible widget. Both are required. If you only need a plain text tooltip, use the universal `:tooltip` property on any widget instead.

---

## calendar — Date Picker

A GTK calendar widget with optional date selection callback.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:day` | float | — | Selected day (1–31) |
| `:month` | float | — | Selected month (0-indexed: 0=January, 11=December) |
| `:year` | float | — | Selected year |
| `:show-details` | bool | `false` | Show day details |
| `:show-heading` | bool | `true` | Show month/year heading |
| `:show-day-names` | bool | `true` | Show day-of-week column headers |
| `:show-week-numbers` | bool | `false` | Show ISO week numbers |
| `:onclick` | string | — | Command when user selects a date; `{0}`=day, `{1}`=month, `{2}`=year |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Examples

```yuck
(defvar cal-day   {formattime(EWW_TIME, "%-d")})
(defvar cal-month {formattime(EWW_TIME, "%-m") - 1})  ; 0-indexed
(defvar cal-year  {formattime(EWW_TIME, "%Y")})

(calendar :day   {cal-day}
          :month {cal-month}
          :year  {cal-year}
          :show-heading true
          :show-day-names true
          :show-week-numbers false
          :onclick "scripts/open-event.sh {0} {1} {2}")
```

> CRITICAL: The `:month` property is **0-indexed** (0 = January, 11 = December) to match GTK convention. If you pass the current month from `formattime("%m")`, subtract 1.

---

## Integration with Other Skills

- Container widgets that hold display widgets: `CONTAINERS.md`
- Interactive widgets (sliders, inputs, buttons): `INTERACTIVE.md`
- Magic variables used as `:value` sources: `MAGIC_VARS.md`
- Expressions for formatting text and computing values: `eww-expressions` skill
- CSS selectors for `progressbar`, `label`, `image`, etc.: `eww-styling` skill
- Real-world examples combining these widgets: `eww-patterns` skill
