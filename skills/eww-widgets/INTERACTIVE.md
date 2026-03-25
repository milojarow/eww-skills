---
name: eww-widgets
description: Choose and configure eww widgets. Use when picking between box/centerbox/overlay/scroll/stack, using button/checkbox/input/scale, displaying data with label/image/progress/circular-progress/graph, using systray or revealer animations, or accessing EWW_RAM/EWW_CPU/EWW_BATTERY/EWW_DISK/EWW_NET magic variables.
---

# Interactive Widgets

Deep reference for `button`, `checkbox`, `input`, `scale`, `combo-box-text`, `color-button`, and `color-chooser`.

---

## button — Clickable Area, Triggered on Release

A container widget that makes its child clickable. Events fire on **mouse release**, not press.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:onclick` | string | — | Command on left click (or keyboard activation) |
| `:onmiddleclick` | string | — | Command on middle click |
| `:onrightclick` | string | — | Command on right click |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Behavior

- `button` wraps any widget as its child. The entire child area is the clickable region.
- Events fire on release, not on press. This matches standard GTK button behavior.
- Has default GTK button styling (border, padding). Override with CSS or `:style` if you want a flat appearance.
- Unlike `eventbox`, `button` does not support hover/scroll/drag events.
- Keyboard navigation activates `:onclick`.

### Examples

```yuck
; Simple text button
(button :onclick "eww open powermenu"
        :onrightclick "eww close powermenu"
  "Power")
```

```yuck
; Icon button that opens a window
(button :onclick "${EWW_CMD} open calendar-popup"
        :class "calendar-btn"
  (label :text {formattime(EWW_TIME, "%H:%M")}))
```

```yuck
; Button with middle-click behavior
(button :onclick      "xdg-open https://example.com"
        :onmiddleclick "xdg-open https://example.com/new-tab"
        :onrightclick  "scripts/context-menu.sh"
  (label :text "Link"))
```

```yuck
; Flat button (no GTK decoration)
(button :onclick "do-thing"
        :style "all: unset; cursor: pointer;"
  (label :text ""))
```

> Use `button` when you want clickable behavior with default button styling. Use `eventbox` when you need hover events, scroll handling, or want zero styling overhead.

---

## checkbox — Toggle Widget

A standard GTK checkbox that fires commands when checked or unchecked.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:checked` | bool | `false` | Initial checked state |
| `:onchecked` | string | — | Command to run when the user checks it |
| `:onunchecked` | string | — | Command to run when the user unchecks it |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Behavior

- `:checked` sets the **initial** state when the widget is created. To keep state in sync with your own variable, bind `:checked` to that variable.
- `onchecked` and `onunchecked` are mutually exclusive per interaction — only one fires per toggle.
- No `{}` placeholder — the commands do not receive the new value as an argument. Use the command to update a `defvar` if needed.

### Examples

```yuck
(defvar notifications-enabled true)

(checkbox :checked notifications-enabled
          :onchecked   "eww update notifications-enabled=true"
          :onunchecked "eww update notifications-enabled=false")
```

```yuck
; Checkbox controlling a service
(checkbox :checked {service-active}
          :onchecked   "systemctl --user start my-service"
          :onunchecked "systemctl --user stop my-service")
```

---

## input — Text Input Field

A text field the user can type into. Requires the window to have `:focusable "ondemand"` set.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:value` | string | — | Current content of the field |
| `:onchange` | string | — | Command run on every keystroke; `{}` replaced by current value |
| `:onaccept` | string | — | Command run when user presses Enter; `{}` replaced by current value |
| `:timeout` | duration | `"200ms"` | Timeout for triggered commands |
| `:password` | bool | `false` | Obscure the input (password mode) |

Plus all universal properties.

### Behavior

- The `{}` placeholder in `:onchange` and `:onaccept` is replaced by the full current field value as a string.
- `:onchange` fires on every character change — useful for live search/filter.
- `:onaccept` fires only when Enter is pressed — useful for "run command" patterns.
- The window containing this widget must declare `:focusable "ondemand"` or the user cannot click into the field and type.
- `:value` bound to a variable allows you to programmatically set or clear the field.

### Examples

```yuck
(defwidget search-bar []
  (input :value    {search-query}
         :onchange "eww update search-query={}"
         :onaccept "scripts/search.sh {}"
         :placeholder "Search..."))
```

> Note: `placeholder` is not listed in the official docs but works as a GTK property via `:placeholder`.

```yuck
; Password entry
(input :value    ""
       :password true
       :onaccept "scripts/unlock.sh {}"
       :class "password-input")
```

> CRITICAL: If the input field does not respond to keyboard input, the window is missing `:focusable "ondemand"`. Add it to your `defwindow`:
>
> ```yuck
> (defwindow launcher
>   :focusable "ondemand"
>   ...)
> ```

### Real Usage

**Rofi-style app search launcher** — `input :onchange` pipes every keystroke to a search script whose output updates a `deflisten` variable; that variable feeds a `literal` for dynamic results. The `eventbox :onhoverlost` auto-closes the window when the mouse leaves, removing the need for an explicit close button.

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

## scale — Slider

A draggable slider for selecting a numeric value within a range.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:value` | float | — | Current value |
| `:min` | float | — | Minimum value |
| `:max` | float | — | Maximum value |
| `:onchange` | string | — | Command on value change; `{}` replaced by new value |
| `:orientation` | string | `"h"` | Direction: `"h"` `"horizontal"` `"v"` `"vertical"` |
| `:flipped` | bool | `false` | Flip the direction (low value at end instead of start) |
| `:draw-value` | bool | `false` | Display the numeric value next to the slider |
| `:value-pos` | string | `"top"` | Position of drawn value: `"left"` `"right"` `"top"` `"bottom"` |
| `:marks` | string | — | Draw tick marks at specified positions |
| `:round-digits` | int | — | Decimal places to round the value to before passing to `:onchange` |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Behavior

- `:onchange` `{}` receives the new value as a float string (e.g., `"72.0"`) unless `:round-digits 0` is set, in which case it receives an integer string.
- Scroll wheel on the slider changes the value. Wrap in `eventbox :onscroll` if you want scroll to control the value without the user dragging the handle.
- CSS classes `scale`, `scale trough`, `scale highlight`, `scale slider` are available for styling.

### Examples

```yuck
; Volume slider
(scale :min 0 :max 100
       :value volume
       :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"
       :round-digits 0
       :orientation "h"
       :class "volume-slider")
```

```yuck
; Brightness slider (vertical)
(scale :min 0 :max 100
       :value brightness
       :onchange "brightnessctl set {}%"
       :orientation "v"
       :flipped true      ; 0 at bottom, 100 at top
       :round-digits 0)
```

```yuck
; Slider with scroll support via eventbox wrapper
(eventbox :onscroll "scripts/volume.sh {}"  ; {} = "up" or "down"
  (scale :min 0 :max 100
         :value volume
         :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"
         :round-digits 0))
```

### Real Usage

**Metric component** — reusable widget pairing an icon/label with a scale. The `:active {onchange != ""}` trick disables interaction when no command is provided, so the same widget serves as both a read-only meter and an interactive slider depending on the call site. CSS class is dynamic: `"error"` above 75, `"warning"` above 50, `"normal"` otherwise.

```yuck
; Source: owenrumney — metrics.yuck
(defwidget metric [icon value ?onchange ?onclick ?class ?height ?width]
  (box :orientation "h"
       :class "metric"
       :space-evenly false
    (termbutton :command "${onclick}" :height "1000" :width "1000" :text "${icon}")
    (scale :class {class != "" ? class : value > 50 ? value > 75 ? "error" : "warning" : "normal"}
           :min 0
           :max 101
           :active {onchange != ""}
           :value value
           :onchange onchange)))
```

**Vertical audio panel** — two `scale` widgets side by side, one active (volume with `amixer` onchange), one read-only (mic, `:active "false"`). Both are `:flipped true :orientation "v"` so 0 sits at the bottom, matching natural audio level expectations.

```yuck
; Source: saimoomedits — eww/leftbar/eww.yuck
(defwidget audio []
  (box :vexpand "false" :hexpand "false"
    (box :orientation "h" :spacing 25 :halign "center" :valign "center"
         :space-evenly "false"
      (box :class "volume_bar" :orientation "v" :spacing 20 :space-evenly "false"
        (scale :flipped "true" :orientation "v"
               :min 0 :max 101
               :value volume_percent
               :onchange "amixer -D pulse sset Master {}%")
        (label :text "" :class "vol_icon"))
      (box :class "mic_bar" :orientation "v" :spacing 20 :space-evenly "false"
        (scale :flipped "true" :orientation "v"
               :min 0 :max 101
               :value mic_percent
               :active "false")       ; read-only meter
        (label :text "" :class "mic_icon")))))
```

**Hover-to-reveal scale** — a scale hidden inside a `revealer-on-hover` component. The icon is always visible; the slider slides in only when hovering over the icon. Demonstrates `:active {onchange != ""}` pattern for optional interactivity.

```yuck
; Source: druskus20 — collapsed-scales-concept/eww.yuck
(defvar reveal1 false)

(defwidget hover-module [vara varname icon ?class]
  (box :space-evenly false
       :class "hover-module ${class}"
    (revealer-on-hover :var vara
                       :varname "${varname}"
                       :transition "slideleft"
      (scale :value 5 :max 10 :min 0)   ; revealed on hover
      (label :class "icon" :text icon)))) ; always visible
```

---

## combo-box-text — Dropdown Selector

A dropdown widget that lets the user choose from a list of text items.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:items` | vec | — | List of items to display |
| `:onchange` | string | — | Command when selection changes; `{}` replaced by selected item as string |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Behavior

- `:items` expects a yuck vector literal or a variable containing a list.
- `:onchange` `{}` is replaced by the **string value** of the selected item, not its index.
- There is no built-in "current selection" binding — track state externally with a `defvar`.

### Examples

```yuck
(defvar selected-theme "dark")

(combo-box-text
  :items '["dark", "light", "nord", "gruvbox"]'
  :onchange "eww update selected-theme={}")
```

```yuck
; Dynamic list from a script
(defpoll audio-outputs :interval "5s"
  `pactl list short sinks | awk '{print $2}'`)

(combo-box-text
  :items {audio-outputs}
  :onchange "pactl set-default-sink {}")
```

> Note: Unlike `scale` and `input`, the `{}` placeholder in `combo-box-text :onchange` receives the selected **string**, not a numeric value.

---

## color-button — Color Picker Button

A button that opens a GTK color chooser dialog when clicked. Returns the selected color as a hex string.

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:use-alpha` | bool | `false` | Include alpha channel in the color picker and returned value |
| `:onchange` | string | — | Command when a color is selected; `{}` replaced by hex color string |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Examples

```yuck
(color-button :use-alpha false
              :onchange "scripts/set-accent.sh {}"
              :class "accent-picker")
```

- Without `:use-alpha`: `{}` receives `#rrggbb`
- With `:use-alpha true`: `{}` receives `#rrggbbaa`

---

## color-chooser — Inline Color Picker

An inline color chooser widget (no button, no dialog — the full color picker UI embedded in your layout).

### Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `:use-alpha` | bool | `false` | Include alpha channel |
| `:onchange` | string | — | Command when a color is selected; `{}` replaced by hex color string |
| `:timeout` | duration | `"200ms"` | Timeout for the triggered command |

Plus all universal properties.

### Examples

```yuck
; Inline color picker in a settings panel
(color-chooser :use-alpha true
               :onchange "scripts/apply-color.sh {}")
```

> `color-button` is compact (a single button). `color-chooser` embeds the full picker UI inline. Use `color-button` in toolbars; use `color-chooser` in settings panels or popups where space is available.

---

## Placeholder Usage Summary

Different interactive widgets use different placeholder conventions:

| Widget | Placeholder | Value received |
|---|---|---|
| `scale` | `{}` in `:onchange` | Float or rounded int (e.g., `"72.0"` or `"72"`) |
| `input` | `{}` in `:onchange` / `:onaccept` | Full string value of the field |
| `combo-box-text` | `{}` in `:onchange` | Selected item as string |
| `color-button` | `{}` in `:onchange` | Hex color string (e.g., `"#ff6600"`) |
| `color-chooser` | `{}` in `:onchange` | Hex color string |
| `eventbox` | `{}` in `:onscroll` | `"up"` or `"down"` |
| `eventbox` | `{}` in `:ondropped` | Dropped URI string |
| `calendar` | `{0}` `{1}` `{2}` in `:onclick` | Day, month, year as ints |
| `checkbox` | — | No placeholder; commands receive no argument |
| `button` | — | No placeholder; commands receive no argument |

---

## Integration with Other Skills

- Container widgets that wrap interactive widgets: `CONTAINERS.md`
- For the `calendar` widget's `{0}` `{1}` `{2}` placeholders: `DISPLAY.md`
- For expressions in `:value` bindings: `eww-expressions` skill
- For CSS styling of sliders, buttons, inputs: `eww-styling` skill
- For window `:focusable` property (required for `input`): `eww-yuck` skill
