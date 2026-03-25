---
name: eww-widgets
description: Choose and configure eww widgets. Use when picking between box/centerbox/overlay/scroll/stack, using button/checkbox/input/scale, displaying data with label/image/progress/circular-progress/graph, using systray or revealer animations, or accessing EWW_RAM/EWW_CPU/EWW_BATTERY/EWW_DISK/EWW_NET magic variables.
---

# Container & Layout Widgets

Deep reference for `box`, `centerbox`, `overlay`, `scroll`, `stack`, and `eventbox`.

---

## box — Primary Layout Container

The fundamental building block. Lays out children in a line, horizontally or vertically.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:orientation` | string | `"h"` | Direction: `"h"` `"horizontal"` `"v"` `"vertical"` |
| `:spacing` | int | `0` | Pixels of space between children |
| `:space-evenly` | bool | `true` | Distribute available space evenly across children |
| `:homogeneous` | bool | — | Force all children to the same size (not documented, inherits from GTK Box) |

Plus all universal properties (`:class`, `:halign`, `:valign`, `:hexpand`, `:vexpand`, `:width`, `:height`, `:style`, `:css`, `:visible`, `:active`, `:tooltip`).

### Behavior

- Children are placed in document order, left-to-right (h) or top-to-bottom (v).
- `:space-evenly true` divides free space equally. `:space-evenly false` packs children tightly from the start.
- `:hexpand` / `:vexpand` on the box itself causes it to fill its parent's available space.
- `:hexpand` / `:vexpand` on a child causes that child to consume remaining space in the box's main axis.

### Examples

```yuck
; Horizontal bar section — tight packing
(box :orientation "h"
     :spacing 8
     :space-evenly false
     :halign "start"
  (label :text "CPU: ${round(EWW_CPU.avg, 0)}%")
  (label :text "RAM: ${round(EWW_RAM.used_mem_perc, 0)}%"))
```

```yuck
; Vertical stack of items with equal distribution
(box :orientation "v"
     :spacing 4
     :space-evenly true
     :vexpand true
  item-a item-b item-c)
```

```yuck
; One item fills all remaining space, others stay minimal
(box :orientation "h" :spacing 0
  (label :text "left")
  (box :hexpand true)     ; spacer — expands to fill
  (label :text "right"))
```

---

## centerbox — Three-Zone Layout

Lays out exactly three children at start, center, and end of the container. The center child is always truly centered relative to the container, not just the remaining space. This is the correct widget for bar layouts.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:orientation` | string | `"h"` | Direction: `"h"` `"horizontal"` `"v"` `"vertical"` |

Plus all universal properties.

### Behavior

- Requires **exactly 3 children**. More or fewer is an error.
- The center child is geometrically centered in the total container width/height.
- The start and end children are positioned at their respective edges.
- If start/end content is large enough to overlap the center zone, content can overlap.

### Examples

```yuck
; Standard horizontal bar
(centerbox :orientation "h"
  (workspaces)    ; left zone
  (clock)         ; center zone — always centered on screen
  (sidestuff))    ; right zone
```

```yuck
; When a zone needs multiple widgets, wrap in a box
(centerbox :orientation "h"
  (box :orientation "h" :spacing 8
    (workspaces)
    (window-title))
  (clock)
  (box :orientation "h" :spacing 6
    (network)
    (battery)
    (volume)
    (systray :icon-size 16)))
```

```yuck
; Vertical centerbox (e.g., side bar)
(centerbox :orientation "v"
  (top-section)
  (middle-section)
  (bottom-section))
```

> CRITICAL: `centerbox` must have exactly 3 children. A common mistake is trying to make it work like `box` with variable child counts. Always wrap groups in a `box` to meet the 3-child requirement.

```yuck
; ❌ WRONG — 4 children
(centerbox :orientation "h"
  (left) (center) (extra) (right))

; ✅ CORRECT — always 3, group as needed
(centerbox :orientation "h"
  (left)
  (center)
  (box :orientation "h" (extra) (right)))
```

---

## overlay — Stack Children on Top of Each Other

Places all children at the same position, on top of each other. The first child determines the size of the overlay widget. Subsequent children are rendered on top within that bounding box.

### Properties

No widget-specific properties beyond universal properties.

### Behavior

- Size is determined by the **first child**. Later children do not expand the bounding box.
- Children are painted in document order: first child at the bottom, last child on top (highest z-order).
- Children that are larger than the first child will be clipped.
- Useful for icon overlays, badge notifications, background + foreground patterns.

### Examples

```yuck
; Progress ring with centered label
(overlay
  (circular-progress :value {EWW_CPU.avg}
                     :thickness 4
                     :clockwise true
                     :width 48 :height 48)
  (label :text "${round(EWW_CPU.avg, 0)}%"
         :halign "center"
         :valign "center"))
```

```yuck
; Notification badge on an icon
(overlay
  (image :path "${EWW_CONFIG_DIR}/icons/mail.svg"
         :image-width 24 :image-height 24)
  (label :text "3"
         :class "badge"
         :halign "end"
         :valign "start"))
```

> Limitation: `overlay` has no z-index control beyond document order. If you need to reorder layers dynamically, use separate windows or restructure your widget tree.

---

## scroll — Scrollable Container

Wraps a single child and allows it to be scrolled when it overflows.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:hscroll` | bool | `false` | Allow horizontal scrolling |
| `:vscroll` | bool | `true` | Allow vertical scrolling |

Plus all universal properties.

### Behavior

- Accepts exactly **one child**.
- The scroll container itself must have a constrained size (via `:height`, `:width`, `:vexpand false`, or parent constraints) for scrolling to activate. If the container expands to fit its child, there is nothing to scroll.
- Both axes can be enabled simultaneously.

### Examples

```yuck
; Vertical scroll list with fixed height
(scroll :vscroll true
        :hscroll false
        :height 200
  (box :orientation "v" :spacing 2
    (for item in items
      (label :text {item}))))
```

```yuck
; Horizontal scroll for long content
(scroll :hscroll true
        :vscroll false
        :width 300
  (label :text very-long-text))
```

> If scrolling is not working, check that the scroll widget has a constrained dimension. Without a height or width limit, it expands freely and never scrolls.

---

## stack — Show One Child at a Time

Displays exactly one child at a time, switching between them with an optional animated transition.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:selected` | int | `0` | Index of the visible child (0-based) |
| `:transition` | string | `"none"` | Animation: `"slideright"` `"slideleft"` `"slideup"` `"slidedown"` `"crossfade"` `"none"` |
| `:same-size` | bool | `true` | Force all children to occupy the same size as the largest child |

Plus all universal properties.

### Behavior

- Children are referenced by index. Index 0 is the first child in document order.
- `:same-size true` prevents the container from resizing when switching between children of different sizes — avoids layout shifts.
- The transition animation plays when `:selected` changes.
- A common pattern is binding `:selected` to a variable that reflects application state (e.g., current view, mode).

### Examples

```yuck
; Tab-like view switcher
(defvar current-view 0)

(stack :selected current-view
       :transition "crossfade"
       :same-size true
  (home-view)      ; index 0
  (stats-view)     ; index 1
  (settings-view)) ; index 2
```

```yuck
; Show different battery icons based on level
(stack :selected {if EWW_BATTERY.BAT0.capacity > 75 then 0
                  else if EWW_BATTERY.BAT0.capacity > 50 then 1
                  else if EWW_BATTERY.BAT0.capacity > 25 then 2
                  else 3}
       :transition "none"
  (label :text "")  ; index 0: full
  (label :text "")  ; index 1: high
  (label :text "")  ; index 2: low
  (label :text "")) ; index 3: critical
```

---

## eventbox — Event-Sensitive Container

A container that captures mouse events — hover, scroll, clicks, drag and drop — and executes commands in response. Accepts exactly one child. Supports `:hover` and `:active` CSS pseudo-selectors.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:onclick` | string | — | Command on left click |
| `:onmiddleclick` | string | — | Command on middle click |
| `:onrightclick` | string | — | Command on right click |
| `:onhover` | string | — | Command when cursor enters widget |
| `:onhoverlost` | string | — | Command when cursor leaves widget |
| `:onscroll` | string | — | Command on scroll; `{}` replaced by `"up"` or `"down"` |
| `:ondropped` | string | — | Command when something is dropped onto this widget; `{}` replaced by the dropped URI |
| `:dragvalue` | string | — | URI to provide when dragging from this widget |
| `:dragtype` | string | — | Type of drag value: `"file"` or `"text"` |
| `:cursor` | string | — | GTK3 cursor name shown on hover (e.g., `"pointer"`, `"crosshair"`, `"text"`) |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Behavior

- Accepts exactly **one child**. Wrap multiple children in a `box`.
- `:onhover` / `:onhoverlost` are useful for showing/hiding tooltips or panels by updating a variable.
- `:onscroll` with `{}` placeholder receives the string `"up"` or `"down"`.
- Unlike `button`, `eventbox` does not have built-in button styling — use it when you need event capture without button appearance.
- CSS `:hover` selector works on `eventbox` without any extra configuration.

### Examples

```yuck
; Hover-to-reveal panel
(defvar hover-active false)

(eventbox :onhover     "eww update hover-active=true"
          :onhoverlost "eww update hover-active=false"
  (box :orientation "v"
    (icon-widget)
    (revealer :reveal hover-active
              :transition "slidedown"
              :duration "200ms"
      (expanded-content))))
```

```yuck
; Scroll to adjust volume
(eventbox :onscroll "scripts/volume.sh {}"
          :cursor "pointer"
  (box :orientation "h" :spacing 4
    (label :text "" :class "vol-icon")
    (label :text "${volume}%" :class "vol-label")))
```

```yuck
; Drag source
(eventbox :dragvalue "file:///home/milo/document.pdf"
          :dragtype "file"
          :cursor "grab"
  (label :text "Drag me"))
```

```yuck
; Drop target
(eventbox :ondropped "scripts/handle-drop.sh {}"
  (label :text "Drop here"))
```

### Real Usage

**Reusable revealer-on-hover component** — the eventbox drives the variable; children[0] is always visible, children[1] slides in on hover. The `?duration` and `?transition` optional args let callers override animation per site.

```yuck
; Source: owenrumney / druskus20 — revealer-hover-module
(defwidget revealer-on-hover [var varname ?class ?duration ?transition]
  (box :class "${class} revealer-on-hover"
       :orientation "h"
       :space-evenly false
    (eventbox :class "eventbox"
              :onhover "eww update ${varname}=true"
              :onhoverlost "eww update ${varname}=false"
      (box :space-evenly false
        (children :nth 0)
        (revealer :reveal var
                  :transition {transition ?: "slideright"}
                  :duration {duration ?: "500ms"}
          (children :nth 1))
        (children :nth 2)))))
```

**Close-on-hoverlost search window** — wraps the entire search panel so moving the mouse away auto-closes it. Combines `eventbox :onhoverlost` with an `input` for a launcher-style UX.

```yuck
; Source: isparsh — eww/eww_widgets.yuck
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

---

## Integration with Other Skills

- For the interactive widgets that go *inside* these containers: `INTERACTIVE.md`
- For display/output widgets: `DISPLAY.md`
- For magic variables used in `:selected`, `:reveal`, etc.: `MAGIC_VARS.md`
- For CSS targeting of these container elements: `eww-styling` skill
- For `defwidget`, `defwindow`, geometry, and monitor configuration: `eww-yuck` skill
