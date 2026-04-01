---
name: eww-widgets
description: Choose and configure eww widgets. Use when picking between box/centerbox/overlay/scroll/stack, using button/checkbox/input/scale, displaying data with label/image/progress/circular-progress/graph, using systray or revealer animations, accessing EWW_RAM/EWW_CPU/EWW_BATTERY/EWW_DISK/EWW_NET magic variables, or understanding widget alignment (halign, valign, hexpand, vexpand, centering, spacing, tug-of-war expand behavior, push to edge).
metadata:
  priority: 5
  pathPatterns: ["**/eww/widgets/**/*.yuck"]
  bashPatterns: []
---

# eww-widgets — Widget Reference & Magic Variables

---

## Quick Start

Pick the right widget for your task, then configure it. Every widget inherits the universal properties table below.

```yuck
; Minimal bar example using the three most common widgets
(defwindow bar
  :monitor 0
  :geometry (geometry :width "100%" :height "32px" :anchor "top center")
  :stacking "fg"
  (centerbox :orientation "h"
    (workspaces)
    (clock)
    (systray :spacing 4 :icon-size 18)))

(defwidget clock []
  (label :text {formattime(EWW_TIME, "%H:%M")}
         :class "clock"))
```

---

## Widget Selection Guide

"What do you want to do?" — find the right widget immediately.

| Goal | Widget |
|---|---|
| Arrange things horizontally or vertically | `box` |
| Left / center / right zones (bar layout) | `centerbox` |
| Stack widgets on top of each other | `overlay` |
| Scrollable area | `scroll` |
| Show one of several children at a time | `stack` |
| Clickable button | `button` |
| Capture hover, scroll, or drag events | `eventbox` |
| Display text | `label` |
| Display an image or theme icon | `image` |
| Linear progress bar | `progress` |
| Circular progress ring | `circular-progress` |
| Time-series graph | `graph` |
| System tray icons | `systray` |
| Slide-in / slide-out animation | `revealer` |
| Collapsible section | `expander` |
| Volume / brightness slider | `scale` |
| User text input (keyboard) | `input` |
| Toggle on/off | `checkbox` |
| Select from a dropdown list | `combo-box-text` |
| Pick a color | `color-button` or `color-chooser` |
| Custom tooltip widget (not just text) | `tooltip` |
| Render dynamic yuck from a variable | `literal` |
| 2D transformation (rotate, scale, translate) | `transform` |
| Date picker | `calendar` |

---

## Universal Widget Properties

These properties apply to **every** widget without exception.

| Property | Type | Default | Description |
|---|---|---|---|
| `:class` | string | — | CSS class name |
| `:halign` | string | `"fill"` | Horizontal alignment: `"fill"` `"baseline"` `"center"` `"start"` `"end"` |
| `:valign` | string | `"fill"` | Vertical alignment: `"fill"` `"baseline"` `"center"` `"start"` `"end"` |
| `:hexpand` | bool | `false` | Expand to fill available horizontal space |
| `:vexpand` | bool | `false` | Expand to fill available vertical space |
| `:width` | int | — | Width in px (cannot shrink below content size) |
| `:height` | int | — | Height in px (cannot shrink below content size) |
| `:active` | bool | `true` | Whether the widget can be interacted with |
| `:tooltip` | string | — | Plain-text tooltip shown on hover |
| `:visible` | bool | `true` | Show or hide the widget |
| `:style` | string | — | Inline SCSS style: `"color: red; font-size: 14px;"` |
| `:css` | string | — | Full SCSS block: `"button { color: red; }"` |

> `:style` applies to the widget element itself. `:css` lets you target child elements by type.

> See [ALIGNMENT.md](./ALIGNMENT.md) for the full behavioral model of `:halign`, `:valign`, `:hexpand`, and `:vexpand` — including the tug-of-war rule and common mistakes.

---

## Magic Variables Quick Reference

All EWW_* variables are built-in — no `defpoll` or `defvar` needed. Check live values with `eww state | grep EWW_`.

| Variable | Update rate | What it contains | Common access pattern |
|---|---|---|---|
| `EWW_TIME` | 1 second | Unix timestamp (int) | `{formattime(EWW_TIME, "%H:%M")}` |
| `EWW_CPU` | 2 seconds | CPU cores + average usage | `{round(EWW_CPU.avg, 0)}` |
| `EWW_RAM` | 2 seconds | Memory/swap stats in KB | `{round(EWW_RAM.used_mem_perc, 0)}` |
| `EWW_DISK` | 2 seconds | Per-mount-point disk stats (bytes) | `{round(EWW_DISK["/"].used_perc, 0)}` |
| `EWW_BATTERY` | 2 seconds | Per-battery capacity + status | `{EWW_BATTERY.BAT0.capacity}` |
| `EWW_NET` | 2 seconds | Per-interface bytes up/down per second | `{round(EWW_NET.wlan0.down / 1000000, 2)}` |
| `EWW_TEMPS` | 2 seconds | Per-sensor temperature in Celsius | `{round(EWW_TEMPS["coretemp Package id 0"], 0)}` |
| `EWW_CONFIG_DIR` | static | Path to eww config directory | `"${EWW_CONFIG_DIR}/icons/foo.png"` |
| `EWW_CMD` | static | eww command for this config | `"${EWW_CMD} update foo=bar"` |
| `EWW_EXECUTABLE` | static | Full path to eww binary | `"${EWW_EXECUTABLE} close my-window"` |

> BEST PRACTICE: Use `EWW_TIME` with `formattime()` for clocks — never a `defpoll` that polls `date`. `EWW_TIME` is more efficient (built-in, no subprocess) and always accurate.

> CRITICAL: Magic variables update every **2 seconds**, except `EWW_TIME` which updates every **1 second**. Do not expect sub-second or even sub-2-second precision from EWW_CPU, EWW_RAM, etc.

---

## Common Widget Patterns

### Clock with date tooltip

```yuck
(defwidget clock []
  (tooltip
    (label :text {formattime(EWW_TIME, "%A, %B %d %Y")})
    (label :text {formattime(EWW_TIME, "%H:%M")}
           :class "clock")))
```

### CPU + RAM status strip

```yuck
(defwidget sysinfo []
  (box :orientation "h" :spacing 12 :class "sysinfo"
    (box :orientation "h" :spacing 4
      (label :text "CPU")
      (progress :value {EWW_CPU.avg}
                :orientation "h"
                :class "cpu-bar"))
    (box :orientation "h" :spacing 4
      (label :text "RAM")
      (progress :value {EWW_RAM.used_mem_perc}
                :orientation "h"
                :class "ram-bar"))))
```

### Battery with charging indicator

```yuck
(defwidget battery []
  (box :orientation "h" :spacing 4 :class "battery"
    (label :text {if EWW_BATTERY.BAT0.status == "Charging" then "+" else ""}
           :class "bat-icon")
    (label :text "${EWW_BATTERY.BAT0.capacity}%"
           :class "bat-label")))
```

### Slide-in panel toggled by a button

```yuck
(defvar show-panel false)

(defwidget panel-toggle []
  (button :onclick "eww update show-panel=${!show-panel}"
    "Menu"))

(defwidget panel []
  (revealer :reveal show-panel
            :transition "slidedown"
            :duration "300ms"
    (box :orientation "v" :class "panel"
      ; panel content here
      )))
```

### Volume slider using eventbox for scroll

```yuck
(defpoll volume :interval "1s"
  "pactl get-sink-volume @DEFAULT_SINK@ | grep -oP '\\d+(?=%)' | head -1")

(defwidget vol []
  (eventbox :onscroll "scripts/volume {}"  ; {} = "up" or "down"
    (box :orientation "h" :spacing 6
      (label :text "" :class "vol-icon")
      (scale :min 0 :max 100
             :value volume
             :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"
             :orientation "h"))))
```

---

## Critical Callouts

> CRITICAL: `centerbox` requires **exactly 3 children** — no more, no fewer. Wrapping child groups in a `box` is the standard way to meet this constraint while having complex content in each zone.

> CRITICAL: `eventbox` accepts **exactly one child**. Wrap multiple elements in a `box` first.

> CRITICAL: `input` requires the window to have `:focusable "ondemand"` set, otherwise the user cannot type into it.

> CRITICAL: `progress` width may not respond to `:width` alone. You must set `min-width` on `progressbar > trough` in your CSS.

> CRITICAL: `overlay` takes the **size of its first child**. All other children are overlaid within that bounding box. If the first child is small, overlapping children will be clipped.

> CRITICAL: Magic variable field names are exact. `EWW_RAM.used_mem_perc` exists; `EWW_RAM.usedPercent` does not. Verify field names with `eww state | grep EWW_`.

---

## Best Practices

**Use `centerbox` for bar layouts, not nested `box` with `hexpand`.**

```yuck
; ❌ WRONG — fragile, center element drifts
(box :orientation "h"
  (left-widgets)
  (box :hexpand true :halign "center" (clock))
  (right-widgets))

; ✅ CORRECT — three zones, center is always centered
(centerbox :orientation "h"
  (left-widgets)
  (clock)
  (right-widgets))
```

**Wrap `scale` in `eventbox` for scroll-to-adjust behavior.**

```yuck
; ✅ CORRECT — scroll on the whole widget area, not just the slider handle
(eventbox :onscroll "handle-scroll {}"
  (scale :min 0 :max 100 :value {vol} :onchange "set-vol {}"))
```

**Use `EWW_CMD` in onclick handlers instead of hardcoding `eww`.**

```yuck
; ❌ WRONG — breaks if eww binary is not in PATH at event time
(button :onclick "eww open panel")

; ✅ CORRECT — always resolves to the correct eww instance
(button :onclick "${EWW_CMD} open panel")
```

**Use `EWW_CONFIG_DIR` for image paths instead of absolute paths.**

```yuck
; ❌ WRONG — breaks if config is moved or used by another user
(image :path "/home/milo/.config/eww/icons/avatar.png")

; ✅ CORRECT — portable
(image :path "${EWW_CONFIG_DIR}/icons/avatar.png")
```

**Prefer `label :markup` for icons from Nerd Fonts, not Unicode escape sequences in expressions.**

```yuck
; ✅ CORRECT — Nerd Font glyph as literal string in text property
(label :text "" :class "icon")
```

---

## Integration with Other Skills

| Topic | Skill |
|---|---|
| Widget layout containers in depth | `CONTAINERS.md` (this skill) |
| Interactive widgets in depth | `INTERACTIVE.md` (this skill) |
| Display/output widgets in depth | `DISPLAY.md` (this skill) |
| Magic variable structures and gotchas | `MAGIC_VARS.md` (this skill) |
| Yuck syntax, defwidget, defwindow, defvar | `eww-yuck` skill |
| Expressions: field access, operators, functions | `eww-expressions` skill |
| CSS selectors for widget types | `eww-styling` skill |
| Real-world bar and popup patterns | `eww-patterns` skill |
| Debugging widget issues | `eww-troubleshooting` skill |

---

## Summary

- **Layout**: `box` (flex), `centerbox` (3-zone), `overlay` (stack), `scroll`, `stack` (one-of-N)
- **Events**: `button` (click), `eventbox` (hover/scroll/drag/click)
- **Input**: `scale` (slider), `input` (text), `checkbox` (toggle), `combo-box-text` (dropdown)
- **Display**: `label`, `image`, `progress`, `circular-progress`, `graph`, `systray`, `revealer`, `expander`
- **Dynamic**: `literal` (dynamic yuck), `tooltip` (custom), `transform` (2D transforms), `calendar`
- **Magic vars**: `EWW_TIME` (1s), everything else (2s) — access with dot notation or bracket notation
- **Universal props**: `:class` `:style` `:css` `:visible` `:active` `:tooltip` `:halign` `:valign` `:hexpand` `:vexpand` `:width` `:height`

See `CONTAINERS.md`, `INTERACTIVE.md`, `DISPLAY.md`, and `MAGIC_VARS.md` for deep-dive references on each category.
