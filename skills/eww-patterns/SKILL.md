---
name: eww-patterns
description: Build real-world eww widgets using proven patterns. Use when building a status bar, power menu, sidebar, notification popup, workspace indicator, music player, or system monitor, asking for real eww examples, dotfile patterns, or architectural guidance, or when positioning, centering, aligning, or spacing elements within a widget layout.
---

# eww-patterns — Real-World Patterns

Proven patterns from: elkowar/eww examples, elkowar/dots-of-war, owenrumney/eww-bar, druskus20/eugh, Saimoomedits/eww-widgets, dharmx powermenu guide, iSparsh/gross, Nycta/eww_activate-linux.

---

## Pattern Selection Guide

What are you building? Start here.

| Goal | File to read |
|---|---|
| Horizontal top/bottom bar | [BAR_PATTERNS.md](./BAR_PATTERNS.md) — Pattern 1 |
| Vertical left/right sidebar bar | [BAR_PATTERNS.md](./BAR_PATTERNS.md) — Pattern 2 |
| Multi-monitor bar | [BAR_PATTERNS.md](./BAR_PATTERNS.md) — Pattern 3 |
| Hover-reveal slider or content | [BAR_PATTERNS.md](./BAR_PATTERNS.md) — Pattern 4 |
| Power menu / logout screen | [POPUP_PATTERNS.md](./POPUP_PATTERNS.md) — Pattern 1 |
| Calendar popup | [POPUP_PATTERNS.md](./POPUP_PATTERNS.md) — Pattern 2 |
| Volume OSD popup | [POPUP_PATTERNS.md](./POPUP_PATTERNS.md) — Pattern 3 |
| Slide-in notification panel | [POPUP_PATTERNS.md](./POPUP_PATTERNS.md) — Pattern 4 |
| Live music info + controls | [DATA_PATTERNS.md](./DATA_PATTERNS.md) — deflisten section |
| Workspace indicator (Sway/bspwm) | [DATA_PATTERNS.md](./DATA_PATTERNS.md) — deflisten section |
| Battery icon with charge levels | [COMMUNITY.md](./COMMUNITY.md) — Snippet 1 |
| Network strength icon | [COMMUNITY.md](./COMMUNITY.md) — Snippet 2 |
| Volume slider (scale widget) | [COMMUNITY.md](./COMMUNITY.md) — Snippet 5 |
| Dynamic list with `for` loop | [DATA_PATTERNS.md](./DATA_PATTERNS.md) — JSON array section |
| Split config into multiple files | [COMMUNITY.md](./COMMUNITY.md) — Snippet 8 |
| Floating watermark / overlay label | [COMMUNITY.md](./COMMUNITY.md) — Snippet 6 |

---

## Core Architectural Principles

These six principles appear across every well-structured eww config. Violating them is the most common source of messy, hard-to-maintain configs.

### 1. Separate Data from UI

Scripts and variables produce data. Widgets consume it. Never embed a shell command inside a widget body — put it in a `defpoll` or `deflisten` and reference the variable name.

```yuck
; Good: data lives in a variable
(defpoll volume :interval "1s" "scripts/getvol")
(label :text "${volume}%")

; Avoid: logic embedded in the widget
(label :text {`amixer get Master | grep -o '[0-9]*%' | head -1`})
```

### 2. Use `include` to Split Configs

Large configs become unmanageable. Split by concern: variables, listeners, pollers, windows, widgets. The entry-point `eww.yuck` then contains only `include` calls.

```yuck
; eww.yuck — entry point only
(include "./variables.yuck")
(include "./listeners.yuck")
(include "./pollers.yuck")
(include "./widgets/bar.yuck")
(include "./widgets/powermenu.yuck")
```

For SCSS, use `@import`:
```scss
@import "./themes/catppuccin";
@import "./bar";
@import "./powermenu";
```

### 3. `centerbox` for 3-Zone Bars

The `centerbox` widget guarantees that its three children occupy left, center, and right zones with true centering — the center child is geometrically centered even if left/right differ in width. Use it as the top-level layout for any bar.

```yuck
(defwidget bar []
  (centerbox :orientation "h"
    (left-section)    ; left zone
    (clock)           ; center zone — always truly centered
    (right-section))) ; right zone
```

For vertical bars, use `:orientation "v"` with top/middle/bottom zones.

### 4. Conditional CSS Classes for State

Rather than hiding/showing widgets with `:visible`, apply dynamic CSS classes and let SCSS handle the visual difference. This is cleaner and supports transitions.

```yuck
; Workspace button — active workspace gets the "focused" class
(button :class {ws.focused ? "focused" : ""}
        :onclick "swaymsg workspace ${ws.name}"
  {ws.name})

; Battery — critical level gets a warning class
(label :class `battery ${EWW_BATTERY.BAT0.capacity < 20 ? "critical" : "ok"}`
       :text battery-icon)
```

```scss
.focused { background-color: #cba6f7; }
.battery.critical { color: #f38ba8; }
```

### 5. `eventbox` for Hover Interactions

The `eventbox` widget wraps any content and fires `onhover`/`onhoverlost` shell commands. Combined with `defvar` and `revealer`, this is the standard pattern for hover-reveal modules (volume sliders, music controls, etc.).

```yuck
(defvar show-vol false)

(eventbox :onhover     "eww update show-vol=true"
          :onhoverlost "eww update show-vol=false"
  (box :space-evenly false
    (label :text "")
    (revealer :reveal show-vol :transition "slideleft" :duration "300ms"
      (scale :min 0 :max 100 :value volume
             :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"))))
```

### 6. `deflisten` Over `defpoll` for Event-Driven Data

`defpoll` runs a command on an interval — wasteful for data that only changes on events. `deflisten` keeps a process open and reads each new line as an update. Use it for music, workspaces, and volume.

```yuck
; deflisten: process stays open, each stdout line updates the variable
(deflisten music :initial ""
  "playerctl --follow metadata --format '{{ artist }} - {{ title }}' || true")

; defpoll: runs every N seconds regardless of changes
(defpoll time :interval "10s"
  "date '+%H:%M'")
```

---

## Multi-Monitor Setup Pattern

Define one `defwindow` per monitor. Pass the monitor name or index as a widget parameter so workspace filtering and monitor-specific logic can differentiate.

```yuck
; One defwindow per physical monitor
(defwindow bar-primary
  :monitor '["<primary>", "DP-1", 0]'  ; array = first match wins
  :geometry (geometry :width "100%" :height "30px" :anchor "top center")
  :stacking "fg"
  :exclusive true
  (bar :screen "primary"))

(defwindow bar-secondary
  :monitor '[1, "HDMI-A-1"]'
  :geometry (geometry :width "100%" :height "30px" :anchor "top center")
  :stacking "fg"
  :exclusive true
  (bar :screen "secondary"))

; Widget accepts screen parameter for per-monitor logic
(defwidget bar [screen]
  (centerbox :orientation "h"
    (workspaces :screen screen)
    (clock)
    (sidestuff)))

; Open both at once
; eww open-many bar-primary bar-secondary
```

The dots-of-war config uses this pattern with a `deflisten` workspace script that emits a JSON object keyed by monitor name, so each bar only renders its own workspaces:

```yuck
; workspaces variable: {"DP-2": [...], "HDMI-A-1": [...]}
(deflisten workspaces :initial '{"DP-2": [], "HDMI-A-1": []}'
  "./swayspaces.py")

(defwidget workspaces [screen]
  (box :orientation "v" :class "workspaces"
    (for wsp in {workspaces[screen]}
      (button :class {wsp.focused ? "active" : "inactive"}
              :onclick "swaymsg workspace ${wsp.name}"
        {wsp.name}))))
```

---

## Reusable Widget Components

Define `defwidget` with parameters to create reusable building blocks. This is how the official eww example's `metric` widget works — one definition, used three times with different data.

```yuck
; Reusable metric: icon + scale bar + optional onchange handler
(defwidget metric [label value onchange]
  (box :orientation "h" :class "metric" :space-evenly false
    (box :class "label" label)
    (scale :min 0 :max 101
           :active {onchange != ""}
           :value value
           :onchange onchange)))

; Usage — each instance passes its own data
(metric :label "Vol"
        :value volume
        :onchange "amixer -D pulse sset Master {}%")
(metric :label "RAM"
        :value {EWW_RAM.used_mem_perc}
        :onchange "")
(metric :label "Disk"
        :value {round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}
        :onchange "")
```

The owenrumney config extends this with an `icon-module` wrapper that adds optional visibility:

```yuck
(defwidget icon-module [icon ?class ?visible]
  (box :class "${class} icon-module"
       :orientation "h" :halign "start" :space-evenly false
       :visible {visible ?: true}
    (label :class "icon-module__icon" :text "${icon}")
    (children)))
```

Note the `?` prefix for optional parameters — `?class` defaults to empty string, `?visible` defaults to false, so `{visible ?: true}` evaluates the default.

---

## Integration with Other Skills

This skill covers architectural patterns and real examples. For deeper reference:

- **eww-yuck** — `defwindow`, `defwidget`, `defpoll`, `deflisten`, `defvar` syntax in detail
- **eww-widgets** — full list of built-in widgets (`box`, `label`, `button`, `scale`, `revealer`, `centerbox`, `eventbox`, `systray`, `calendar`, `circular-progress`) and their properties
- **eww-styling** — SCSS structure, GTK CSS selectors, `* { all: unset }`, theming with variables
- **eww-troubleshooting** — `eww logs`, `eww state`, `eww inspector`, common error messages

---

## Summary

| File | What it covers |
|---|---|
| [BAR_PATTERNS.md](./BAR_PATTERNS.md) | Horizontal bar, vertical sidebar, multi-monitor, hover-reveal — complete yuck + scss |
| [POPUP_PATTERNS.md](./POPUP_PATTERNS.md) | Power menu, calendar popup, volume OSD, notification revealer — complete yuck + scss |
| [DATA_PATTERNS.md](./DATA_PATTERNS.md) | defpoll JSON, deflisten streams, shell scripts, `for` loops, dynamic class state |
| [COMMUNITY.md](./COMMUNITY.md) | Curated snippets with attribution: battery icon, network indicator, music player, workspace buttons, volume slider, activate-linux widget, modular file structure |
| [README.md](./README.md) | Navigation guide — start here if lost |
