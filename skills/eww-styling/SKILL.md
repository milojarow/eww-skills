---
name: eww-styling
description: Style eww widgets with GTK CSS/SCSS. Use when writing eww.scss, applying CSS to widgets, using GTK selectors, setting up SCSS variables or theming, using dynamic CSS classes, debugging styles with eww inspector, or applying the required * { all: unset; } reset.
---

# eww-styling — GTK CSS/SCSS

eww uses **GTK CSS** for styling. You can write vanilla CSS or **SCSS** (auto-compiled by eww on load).

---

## CRITICAL: Always Start With `* { all: unset; }`

**GTK applies its own opinionated default styles to every widget.** Without the global reset, your styles will fight GTK's defaults — and often lose. Buttons will have unexpected padding, labels will inherit theme fonts, boxes will have invisible margins. Always put this at the very top of `eww.scss`.

❌ WRONG — no reset, unexpected GTK styles bleed through:

```scss
/* eww.scss — missing reset */

.bar {
  background-color: #1e1e2e;
  /* GTK default padding, borders, and font may still appear */
}

button {
  color: #cba6f7;
  /* GTK button styles (borders, relief, padding) override your color */
}
```

✅ CORRECT — reset first, then style from a clean slate:

```scss
/* eww.scss */
* { all: unset; }

.bar {
  background-color: #1e1e2e;
  color: #cdd6f4;
  padding: 4px 8px;
}

button {
  color: #cba6f7;
  /* Fully predictable — no GTK defaults interfering */
}
```

The reset sets every CSS property to its initial (browser-like default) value on every widget. Your rules then build on top of a clean foundation.

---

## Quick Start

A minimal, working `eww.scss` for a status bar:

```scss
/* eww.scss */
* { all: unset; }

$bg:     #1e1e2e;
$fg:     #cdd6f4;
$accent: #cba6f7;
$muted:  #6c7086;
$surface: #313244;

.bar {
  background-color: $bg;
  color: $fg;
  padding: 4px 8px;
  font-family: "JetBrainsMono Nerd Font";
  font-size: 13px;
}

/* Workspace buttons */
.workspaces button {
  color: $muted;
  padding: 0 6px;
  border-radius: 4px;
}

.workspaces button.focused {
  color: $accent;
  background-color: $surface;
}

.workspaces button.urgent {
  color: #f38ba8;
}

/* Slider (e.g. volume) */
.metric scale trough highlight {
  background-color: $accent;
  border-radius: 10px;
  min-height: 4px;
}

.metric scale trough {
  background-color: $surface;
  min-height: 4px;
  min-width: 80px;
}
```

This covers the three most common patterns: a container class, state-based button classes, and a GTK sub-element selector (`scale trough highlight`).

---

## File Setup

```
~/.config/eww/
├── eww.yuck          <- widget definitions
├── eww.scss          <- styles (SCSS — auto-compiled)
└── eww.css           <- alternative (vanilla CSS — also valid)
```

- Use `eww.scss` for SCSS features (variables, nesting, `@import`)
- Use `eww.css` if you prefer plain CSS and don't need SCSS
- eww automatically compiles `eww.scss` to CSS when it loads; no manual compile step
- For modular projects, `eww.scss` can be just a series of `@import` statements pointing to other `.scss` files

Modular layout example:

```
~/.config/eww/
├── eww.yuck
├── eww.scss
├── themes/
│   └── catppuccin-mocha.scss
└── styles/
    ├── bar.scss
    ├── workspaces.scss
    └── powermenu.scss
```

```scss
/* eww.scss — entry point only */
* { all: unset; }

@import "./themes/catppuccin-mocha";
@import "./styles/bar";
@import "./styles/workspaces";
@import "./styles/powermenu";
```

---

## GTK CSS Selectors

GTK CSS uses the same selector syntax as web CSS, but the type names are GTK widget class names, not HTML tags.

### Type selectors — GTK widget type names

```scss
box         { }   /* eww box widget */
label       { }   /* eww label widget */
button      { }   /* eww button widget */
scale       { }   /* eww scale (slider) widget */
progressbar { }   /* eww progressbar widget */
image       { }   /* eww image widget */
entry       { }   /* text input */
checkbutton { }   /* checkbox */
window      { }   /* top-level eww window */
menu        { }   /* system tray context menus */
menuitem    { }   /* items inside menus */
```

### Class selectors

```scss
.my-class       { }
.bar .label     { }   /* label descendant of .bar */
box > label     { }   /* label direct child of box */
```

### ID selectors

```scss
#my-widget-id   { }   /* matches :id "my-widget-id" in yuck */
```

### Combined selectors

```scss
button.active           { }   /* button with class "active" */
label.critical          { }   /* label with class "critical" */
.module button          { }   /* button anywhere inside .module */
.workspaces button.focused { } /* focused button inside .workspaces */
```

### Pseudo-classes

```scss
button:hover            { }   /* mouse over */
button:active           { }   /* being clicked */
checkbutton:checked     { }   /* checked state */
widget:focus            { }   /* has keyboard focus */
widget:disabled         { }   /* disabled state */
widget:first-child      { }   /* first child in parent */
widget:last-child       { }   /* last child in parent */
widget:nth-child(2)     { }   /* second child */
```

### Finding the right selector — use the inspector

```bash
eww inspector
```

Click the crosshair icon in the top left, then click any widget on screen. Select "CSS Nodes" from the dropdown in the top right. You will see the exact GTK type name, all applied CSS classes, and the widget hierarchy. This is the fastest way to determine why a selector is not matching.

---

## Widget Type Names and Sub-Elements

Some eww widgets have internal GTK sub-elements that you style with descendant selectors. These sub-elements are not eww concepts — they are part of GTK's internal widget structure and only visible in the inspector.

### scale (slider widget)

```scss
/* The filled/active portion — reflects current value */
scale trough highlight {
  background-color: #cba6f7;
  border-radius: 10px;
  min-height: 4px;
}

/* The full trough track (background behind highlight) */
scale trough {
  background-color: #313244;
  min-height: 4px;
  min-width: 80px;
}

/* The draggable handle knob */
scale slider {
  background-color: #cdd6f4;
  border-radius: 50%;
  min-width: 10px;
  min-height: 10px;
}
```

Note: `scale trough` styles both the track and the highlight unless you also target `scale trough highlight` with higher specificity. Always style `highlight` after `trough`.

### progressbar

```scss
/* The background track */
progressbar trough {
  background-color: #313244;
  min-height: 6px;
  border-radius: 3px;
}

/* The filled progress bar */
progressbar progress {
  background-color: #cba6f7;
  border-radius: 3px;
}
```

### circular-progress

The `circular-progress` eww widget is drawn with Cairo — it does not have CSS sub-elements like `scale` does. Control its appearance through the widget's own yuck attributes (`:value`, `:thickness`, `:start-at`, `:clockwise`) and color via the `:style` attribute or a wrapper class for background/padding.

### window (eww window container)

```scss
window {
  background-color: transparent;  /* for transparent bars */
}
```

### menu and menuitem (system tray)

```scss
menu {
  padding: 5px;
  background-color: #1e1e2e;
}

menuitem {
  padding: 2px 5px;
  font-size: 14px;
}

menuitem:hover {
  background-color: #313244;
}

menuitem:disabled label {
  color: #6c7086;
}

separator {
  padding-top: 1px;
}
```

---

## Dynamic Classes (State-Based Styling)

Dynamic classes are the primary way to change widget appearance based on data. You set the class in yuck using an expression, then style the resulting class combinations in SCSS.

### Pattern in yuck

```yuck
; Simple binary toggle
(button :class {active ? "active" : "inactive"})

; Base class + conditional extra class using backtick string
(label :class `battery ${capacity < 20 ? "critical" : capacity < 50 ? "low" : "ok"}`)

; Workspace focused state
(button :class {ws.focused ? "focused" : "unfocused"}
        :onclick "swaymsg workspace ${ws.name}"
  (label :text ws.name))

; Network activity level
(label :class {speed > 5 ? "veryactive" : speed > 0.1 ? "active" : "idle"}
       :text speed-text)
```

### Corresponding SCSS

```scss
/* Battery */
.battery         { color: $fg; }
.battery.low     { color: $yellow; }
.battery.critical {
  color: $red;
  animation: pulse 1s infinite;  /* transitions work in GTK */
}

/* Workspace buttons */
.workspaces button.focused   { color: $accent; background-color: $surface; }
.workspaces button.unfocused { color: $muted; }
.workspaces button.urgent    { color: $red; }

/* Network */
label.active     { color: $green; }
label.veryactive { color: $yellow; }
label.idle       { color: $muted; }
```

### Scale state classes (from owenrumney pattern)

```yuck
(scale :class {metric-class}   ; e.g. "metric warning" or "metric error"
       :value cpu-percent)
```

```scss
/* Default metric scale */
.metric scale trough highlight {
  background-color: $green;
  border-radius: 10px;
}

/* Warning threshold */
.metric scale.warning trough highlight {
  background-color: $yellow;
}

/* Error/critical threshold */
.metric scale.error trough highlight {
  background-color: $red;
}
```

---

## SCSS Features

### Variables

```scss
$bg:      #1e1e2e;
$fg:      #cdd6f4;
$accent:  #cba6f7;
$red:     #f38ba8;
$yellow:  #f9e2af;
$green:   #a6e3a1;
$surface: #313244;
$muted:   #6c7086;

.bar   { background-color: $bg; color: $fg; }
.alert { color: $red; }
```

### Nesting

```scss
.workspaces {
  background-color: $surface;
  border-radius: 15px;
  padding: 0 5px;

  button {
    color: $muted;
    padding: 0 6px;
    border-radius: 4px;
  }

  /* & refers to the parent selector */
  button.focused {
    color: $accent;
    background-color: $bg;
  }

  button.urgent {
    color: $red;
  }
}

/* Compiles to: .workspaces { } .workspaces button { } .workspaces button.focused { } */
```

### SCSS & parent reference

```scss
.workspaces {
  button {
    color: $muted;
    &.focused  { color: $accent; }   /* .workspaces button.focused */
    &.urgent   { color: $red; }      /* .workspaces button.urgent */
    &:hover    { color: $fg; }       /* .workspaces button:hover */
  }
}
```

### @import — modular file structure

```scss
/* eww.scss */
@import "./themes/catppuccin-mocha";   /* no .scss extension needed */
@import "./styles/bar";
@import "./styles/workspaces";
@import "./styles/powermenu";
```

Import order matters: variables must be imported before the files that use them. Always import your theme file first.

### @mixin and @include — reusable style blocks

```scss
@mixin pill($bg, $fg) {
  background-color: $bg;
  color: $fg;
  border-radius: 20px;
  padding: 2px 10px;
}

.tag-active   { @include pill($accent, $bg); }
.tag-inactive { @include pill($surface, $muted); }
.tag-urgent   { @include pill($red, $bg); }
```

---

## eww Inspector

```bash
eww inspector
```

The GTK inspector is the primary debugging tool for eww styles.

**How to use:**

1. Run `eww inspector` while your bar/window is open
2. Click the crosshair icon (top left) to enter "pick" mode
3. Click any widget on screen to select it
4. In the dropdown (top right), select **CSS Nodes** to see:
   - The exact GTK type name of the widget (e.g., `button`, `label`, `scale`)
   - All CSS classes currently applied (including dynamic ones)
   - The parent hierarchy of widget types
5. In the dropdown, select **CSS** to see:
   - Which CSS rules are currently active on the selected widget
   - Which file and line each rule comes from
   - Which rules are overridden and why

**Common inspection tasks:**

- "Why is my selector not matching?" — check the exact type name and classes in CSS Nodes
- "Where is this color coming from?" — CSS tab shows rule sources
- "Is my dynamic class being applied?" — CSS Nodes shows live classes including dynamic ones
- "What are the sub-elements of this widget?" — expand the CSS Nodes tree

**Parse error note:** If the inspector shows no custom CSS rules at all, there is likely a SCSS parse error earlier in the file. eww stops loading the stylesheet at the first error. Debug by commenting out sections until styles start appearing again.

---

## Common Mistakes (Why Styles Don't Apply)

**1. Missing `* { all: unset; }`**
GTK defaults override your rules. Add the reset as the very first rule.

**2. Wrong widget type name**
`div`, `span`, `p` are HTML — they don't exist in GTK. Use `box`, `label`, `button`. Use the inspector to find exact names.

**3. Parse error blocking all subsequent rules**
One typo stops all CSS loading from that point onward. Open the inspector and check if any of your rules appear. If none appear, search upward from the end for syntax errors.

**4. Using `width`/`height` instead of `min-width`/`min-height`**
GTK CSS ignores `width` and `height` on most widgets. Use `min-width` and `min-height` instead.

**5. Class typo between yuck and SCSS**
`:class "actve"` in yuck and `.active` in SCSS will never match. Copy-paste class names.

**6. Selector specificity**
`button.active` has higher specificity than `button`. If your base `button` rule uses `!important`, it will block the `.active` override. Avoid `!important`; use specificity correctly.

**7. Forgetting that `scale trough` matches the highlight too**
`scale trough` targets both the trough background and the highlight child. Define `scale trough highlight` after `scale trough` to override only the highlight.

❌ WRONG — highlight rule defined before trough, gets overridden:

```scss
scale trough highlight { background-color: $accent; }
scale trough           { background-color: $surface; min-height: 4px; }
```

✅ CORRECT — trough background first, then the more specific highlight:

```scss
scale trough           { background-color: $surface; min-height: 4px; min-width: 80px; }
scale trough highlight { background-color: $accent; border-radius: 10px; }
```

**8. Using yuck layout attributes as CSS properties**

`spacing`, `halign`, `valign`, `hexpand`, `vexpand`, `space-evenly`, and `orientation` are **yuck widget attributes** — they control layout in `.yuck` files. They are not GTK CSS properties. Writing them in `.scss` causes a parse error that stops all CSS loading from that line onward, silently breaking every widget that relies on styles defined after it.

❌ WRONG — `spacing` is a yuck attribute, not CSS:

```scss
.my-box {
  spacing: 4px;   /* invalid — GTK CSS parse error */
}
```

✅ CORRECT — set `:spacing` in the yuck file; use `margin` on children for CSS-side spacing:

```yuck
(box :class "my-box" :spacing 8 ...)
```

```scss
/* If you need CSS-side spacing, use margins on children */
.my-box > * {
  margin: 0 4px;
}
```

The full list of yuck-only layout attributes that must **never** appear in SCSS:
`spacing` · `halign` · `valign` · `hexpand` · `vexpand` · `space-evenly` · `orientation`

---

**9. Using standard CSS properties that GTK does not implement**

GTK3's CSS engine implements a subset of the CSS spec. Using an unsupported property causes a parse error that stops **all CSS loading from that line onward**, silently breaking every widget styled below it in the file. The symptom is identical to any other parse error: all widget styling disappears at once.

Two confirmed-unsupported properties that look valid but will break everything:

| Property | Error in `eww logs` / `journalctl` | GTK-native alternative |
|---|---|---|
| `overflow: hidden` | `'overflow' is not a valid property name` | `overlay` widget — GTK clips overlay children to the overlay's allocation natively |
| `transform: translateX()` | `No property named 'transform'` | `margin-left` / `margin-top` in `@keyframes` |

❌ WRONG — both cause parse errors that kill all styles below them:

```scss
.clip-box {
  overflow: hidden;               /* GTK CSS parse error */
}
.slide {
  transform: translateX(-100px);  /* GTK CSS parse error */
}
```

✅ CORRECT — GTK-native alternatives:

For **clipping** (replacing `overflow: hidden`): use the `overlay` widget. GTK clips
all overlay children to the overlay's own allocated size — no CSS required.

```yuck
(overlay
  (box :width 170 :height 30 :class "clip-bg")  ; main child — sets size, acts as clip
  (box :class "scrolling-strip" ...))            ; overlay child — natural width, clipped to 170px
```

For **positional animation** (replacing `transform`): use `margin-left` or `margin-top`
in `@keyframes`. These ARE supported in GTK CSS.

```scss
@keyframes scroll-left {
  from { margin-left: 0px; }
  to   { margin-left: -200px; }
}
.scrolling-strip {
  animation: scroll-left 8s linear infinite;
}
```

> IMPORTANT: `margin-left` on a widget in the normal layout flow will shift adjacent
> widgets (it affects layout, not just rendering). To animate position without affecting
> surrounding elements, the widget must be an `overlay` child — overlay children are
> outside the normal layout flow.

---

## CSS Animations in GTK

GTK3 supports `@keyframes` and the `animation` shorthand. These work reliably in eww.

### What works

```scss
@keyframes my-anim {
  0%   { margin-top: 0px; opacity: 1; }
  50%  { margin-top: -30px; opacity: 0.5; }
  100% { margin-top: 0px; opacity: 1; }
}

.animated {
  animation: my-anim 2s ease-in-out infinite;
}
```

**Confirmed animatable properties in GTK CSS:**
`margin-top` · `margin-left` · `margin-bottom` · `margin-right` · `opacity` ·
`background-color` · `color` · `min-height` · `min-width` · `padding` · `font-size`

### What does NOT work

`transform` and `overflow` are **not implemented** in GTK3's CSS engine. Using either
causes a parse error that breaks all styles below that line in the file.

### Horizontal marquee pattern (confirmed working in eww)

Use `overlay` for clipping + `margin-left` animation on the overlay child:

```yuck
; The overlay's main child sets the visible window size (e.g. 170px).
; GTK clips overlay children to that size automatically.
(overlay
  (box :width 170 :height 30 :class "marquee-bg")
  ; Strip gets its natural (wider) width; overlay clips it to 170px
  (box :class "marquee-strip"
       :orientation "h"
       :space-evenly false
       :halign "start"
       :hexpand false
    (label :class "marquee-item" :text "first item")
    (label :class "marquee-item" :text "second item")
    ; Duplicate for seamless loop:
    (label :class "marquee-item" :text "first item")
    (label :class "marquee-item" :text "second item")))
```

```scss
.marquee-strip {
  animation: marquee-scroll 8s linear infinite;
}

@keyframes marquee-scroll {
  from { margin-left: 0px; }
  to   { margin-left: -<one-cycle-width>px; }  /* width of one [first][second] pair */
}
```

The loop is seamless: at `-<one-cycle-width>px` the visible content is identical to
`0px` (the duplicate pair). For monospace fonts, calculate pixel width as:
`chars × (font-size × 0.6) + padding`.

---

## Integration with Other Skills

- **eww-yuck** — use `:class` attribute patterns; the yuck side of dynamic classes
- **eww-widgets** — full reference of every eww widget type and its yuck attributes
- **eww-expressions** — write the expressions used inside `:class {}` and `:class \`\``
- **eww-troubleshooting** — deeper debugging: styles not applying, eww not reloading, parse errors

---

## Summary

1. Always start with `* { all: unset; }` — no exceptions
2. eww uses GTK CSS type names (`box`, `label`, `button`), not HTML tags
3. SCSS auto-compiles — use variables, nesting, and `@import` freely
4. Style sub-elements with descendant selectors: `scale trough highlight`, `progressbar progress`
5. Dynamic classes bridge eww state (variables) to CSS appearance
6. Use `eww inspector` to find exact selectors and debug rule application
7. Parse errors stop all CSS loading from that line onward
8. Use `min-width`/`min-height`, not `width`/`height`
9. Never write yuck layout attributes (`spacing`, `halign`, `valign`, etc.) in SCSS — they are not CSS properties; one invalid property breaks all styles below it
10. `overflow` and `transform` are also not GTK CSS properties — they cause the same parse error; use `overlay` widget for clipping and `margin-left`/`margin-top` in `@keyframes` for positional animation

**Reference files in this skill:**
- `CSS_REFERENCE.md` — complete GTK CSS property and selector reference
- `SCSS_PATTERNS.md` — SCSS patterns with real community examples
- `THEMES.md` — copy-paste color palettes (Catppuccin, Gruvbox, Nord, Tokyo Night)
