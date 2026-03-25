---
name: eww-styling
description: Style eww widgets with GTK CSS/SCSS. Use when writing eww.scss, applying CSS to widgets, using GTK selectors, setting up SCSS variables or theming, using dynamic CSS classes, debugging styles with eww inspector, or applying the required * { all: unset; } reset.
---

# SCSS_PATTERNS — SCSS Patterns for eww

Practical SCSS patterns drawn from real eww community configs. Each section shows the yuck side (when relevant) and the SCSS side together.

---

## all: unset — The Required GTK Reset

Every eww config that renders correctly starts with this. GTK widgets carry heavy default styling — without the reset, borders, backgrounds, padding, and focus rings appear where you don't want them.

### Global reset (druskus20 pattern)

The most common placement: a bare `* { all: unset; }` after variable declarations, before any class rules. From simpler-bar/eww.scss exactly as written:

```scss
// Source: druskus20 — simpler-bar/eww.scss
$base: #1e1e2e;
$text: #cdd6f4;
// ... rest of variables ...

* {
  all: unset;
}

window {
  font-family: "NotoSans Nerd Font", sans-serif;
  background-color: $base;
}
```

### Global reset with shared font-family (isparsh pattern)

isparsh combines `all: unset` and `font-family` in the same `*` rule — both declarations apply to all elements. From eww/eww.scss exactly as written:

```scss
// Source: isparsh — eww/eww.scss
* {
  all: unset;
  font-family: "JetBrains Mono";
}
```

### Global reset with multiple font-family declarations (saimoomedits pattern)

saimoomedits declares two `font-family` lines in the same rule — the second one wins. From leftbar/eww.scss exactly as written:

```scss
// Source: saimoomedits — eww/leftbar/eww.scss
*{
	all: unset;
	font-family: feather;
	font-family: JetBrainsMono NF;
}
```

### Per-element reset inside GTK sub-nodes (owenrumney pattern)

The global `* { all: unset; }` does not fully reach GTK internal sub-nodes like scale trough and highlight. owenrumney applies `all: unset` again on every trough and highlight rule individually. From eww.scss exactly as written:

```scss
// Source: owenrumney — eww.scss
// Note: all: unset appears on EVERY trough and highlight rule, not just the global reset

.metric scale trough highlight {
  all: unset;
  background-color: $green;
  color: $black;
  border-radius: 10px;
}

.metric scale.warning trough highlight {
  all: unset;
  background-color: $yellow;
  color: $black;
  border-radius: 10px;
}

.metric scale.error trough highlight {
  all: unset;
  background-color: $error;
  color: $black;
  border-radius: 10px;
}

.metric scale.volume trough highlight {
  all: unset;
  background-color: $lightmagenta;
  color: $black;
  border-radius: 10px;
}

.metric scale trough {
  all: unset;
  background-color: $background-alt;
  border-radius: 50px;
  min-height: 5px;
  min-width: 100px;
  margin-left: 10px;
  margin-right: 20px;
}
```

saimoomedits does the same for each named bar in leftbar/eww.scss:

```scss
// Source: saimoomedits — eww/leftbar/eww.scss
.storage_bar scale trough highlight {
	all: unset;
    background-image: linear-gradient(to top, #91ab7a 30%, #afbea2 50%, #ccefad 100% *50);
	border-radius: 24px;
}
.bright_bar scale trough highlight {
	all: unset;
    background-image: linear-gradient(to top, #e0b089 30%, #e4c9af 50%, #ebd0b7 100% *50);
	border-radius: 24px;
}
.volume_bar scale trough highlight {
	all: unset;
    background-color: #bfc9db;
	border-radius: 24px;
}
.mic_bar scale trough highlight {
	all: unset;
    background-color: #bfc9db;
	border-radius: 24px;
}
scale trough {
	all: unset;
	background-color: #232232;
	border-radius: 24px;
	min-height: 130px;
	min-width: 14px;
	margin : 10px 0px 0px 0px;
}
```

---

## @import — Split Theme from Main SCSS

owenrumney's config is the canonical example of separating the color palette into its own file. From eww.scss exactly as written:

```scss
// Source: owenrumney — eww.scss
@import "gruvbox.scss";

* {
  all: unset;
}

// ... all widget rules follow, using $background, $green, $error etc. from gruvbox.scss
```

The pattern:
1. `@import` is the very first line
2. `* { all: unset; }` comes immediately after the import
3. Widget rules follow — they can use any variable defined in the imported file

Import rules:
- The `.scss` extension is optional in `@import`
- Paths are relative to the importing file
- SCSS `@import` inlines the file at compile time (unlike CSS `@import` which is a runtime network request)
- Variables must be defined before use — import theme before any rule that references its variables

---

## Workspace Status Classes (druskus20 pattern)

druskus20's workspace-indicator uses four named status classes set dynamically from the yuck side. From workspace-indicator/eww.scss exactly as written:

```scss
// Source: druskus20 — workspace-indicator/eww.scss

* {
    all: unset;
}

.top {
    background-color: #202020;
}

.ws {
    .small-ws {
        &.status-empty {
            font-family: "Font Awesome 6 Free Solid";
            color: rgba(237, 112, 112, 0.3);
        }
        &.status-occupied {
            font-family: "Font Awesome 6 Free Solid";
            color: rgba(237, 112, 112, 0.5);
        }
        &.status-focused {
            font-family: "Font Awesome 6 Free Solid";
            color: rgba(237, 112, 112, 1.0);
        }
        &.status-urgent {
            font-family: "Font Awesome 6 Free Solid";
            color: rgba(255, 255, 255, 1.0)
        }
    }

    .big-ws {
        font-size: 20px;
        font-weight: bold;
        color: white;
        margin: 4px;
        background-image: url("res/rhombus.png");
        background-size: 100% 100%;
        background-repeat: no-repeat;
        background-position: center;
    }
}
```

Key observations:
- All four states use the same `rgba(237, 112, 112, ...)` hue — only the alpha changes (0.3 / 0.5 / 1.0 / override to white for urgent)
- The `font-family` is repeated in every state because `all: unset` on `*` strips it — each state must declare it explicitly
- `.big-ws` uses `background-image: url()` for a decorative shape behind the workspace name
- Class names are `status-empty`, `status-occupied`, `status-focused`, `status-urgent` — set from the yuck side via `deflisten` on sway IPC events

Matching yuck pattern (how the class is assigned):

```yuck
; The class string is built from the workspace state coming from IPC
(label :class "small-ws status-${ws.status}"
       :text ws.icon)
```

---

## Metric Scale Threshold Coloring (owenrumney pattern)

The class that determines the color lives on the `scale` widget itself, not on any parent. The selector chain is: `.metric scale.warning trough highlight`. From eww.scss exactly as written:

```scss
// Source: owenrumney — eww.scss

// Default (normal/ok) — green highlight
.metric scale trough highlight {
  all: unset;
  background-color: $green;
  color: $black;
  border-radius: 10px;
}

// Warning threshold — yellow highlight
.metric scale.warning trough highlight {
  all: unset;
  background-color: $yellow;
  color: $black;
  border-radius: 10px;
}

// Error threshold — red highlight
.metric scale.error trough highlight {
  all: unset;
  background-color: $error;
  color: $black;
  border-radius: 10px;
}

// Volume — distinct color (lightmagenta) regardless of level
.metric scale.volume trough highlight {
  all: unset;
  background-color: $lightmagenta;
  color: $black;
  border-radius: 10px;
}

// Trough (track background) — same for all states
.metric scale trough {
  all: unset;
  background-color: $background-alt;
  border-radius: 50px;
  min-height: 5px;
  min-width: 100px;
  margin-left: 10px;
  margin-right: 20px;
}
```

The yuck side computes which class to assign:

```yuck
; The scale widget gets class "metric warning", "metric error", or just "metric"
; depending on the current value
(scale :class {cpu > 90 ? "metric error" : cpu > 75 ? "metric warning" : "metric"}
       :value cpu
       :max 100)
```

druskus20 uses a simpler two-state variant — `.over` class when volume exceeds 100%:

```scss
// Source: druskus20 — simpler-bar/eww.scss
scale trough highlight {
  background-color: $text;
  border-radius: 2px;
}

scale.over trough highlight {
  background-color: $peach;
  border-radius: 2px;
}
```

isparsh uses a gradient instead of a solid color — no threshold classes, single gradient for all values:

```scss
// Source: isparsh — eww/eww.scss
.metric scale trough highlight {
  all: unset;
  color: #000000;
  background: linear-gradient(90deg, #88c0d0 0%, #81a1c1 50%, #5E81ac 100%);
}
```

---

## Circular/Tall Progress Bars (saimoomedits pattern)

saimoomedits uses vertical scales with `min-height: 130px` and `min-width: 14px` to create tall narrow bars that visually resemble circular-progress columns. Each bar type gets its own gradient. From leftbar/eww.scss exactly as written:

```scss
// Source: saimoomedits — eww/leftbar/eww.scss

// Storage bar — green gradient (bottom to top)
.storage_bar scale trough highlight {
	all: unset;
    background-image: linear-gradient(to top, #91ab7a 30%, #afbea2 50%, #ccefad 100% *50);
	border-radius: 24px;
}

// Brightness bar — warm gradient
.bright_bar scale trough highlight {
	all: unset;
    background-image: linear-gradient(to top, #e0b089 30%, #e4c9af 50%, #ebd0b7 100% *50);
	border-radius: 24px;
}

// Volume and mic bars — solid color
.volume_bar scale trough highlight {
	all: unset;
    background-color: #bfc9db;
	border-radius: 24px;
}
.mic_bar scale trough highlight {
	all: unset;
    background-color: #bfc9db;
	border-radius: 24px;
}

// Shared trough for all bars — tall and narrow
scale trough {
	all: unset;
	background-color: #232232;
	border-radius: 24px;
	min-height: 130px;
	min-width: 14px;
	margin : 10px 0px 0px 0px;
}
```

saimoomedits also styles a horizontal music progress bar with different dimensions:

```scss
// Source: saimoomedits — eww/leftbar/eww.scss
.song_prog scale trough highlight {
	all: unset;
    background-color: #bfc9db;
	border-radius: 8px;
}
.song_prog scale trough {
	all: unset;
	background-color: #232232;
	border-radius: 8px;
	min-height: 5px;
	min-width: 220px;
	margin : 15px;
}
```

The pattern: name each bar with a BEM-style parent class (`.storage_bar`, `.bright_bar`, `.volume_bar`) and target `scale trough highlight` under it. All bars share a single bare `scale trough` rule for the track.

---

## Nesting

SCSS nesting keeps related rules together and reduces repetition.

### Basic nesting with button states

From owenrumney's eww.scss — workspaces with inline button state rules:

```scss
// Source: owenrumney — eww.scss
.workspaces {
  background-color: $background-alt;
  border-radius: 15px;
  margin-left: 5px;
  margin-top: 5px;
  padding-left: 10px;
  padding-right: 5px;
  font-family: "Hack NF Regular";
  font-size: 28px;
  margin-right: 20px;

  button         { color: #a89984; margin: 0px 3px; }
  button.focused { color: $green; }
  button.urgent  { color: $urgent; }
}
```

### & parent reference for BEM sub-elements

From owenrumney's eww.scss — the `.icon-module__icon` BEM pattern:

```scss
// Source: owenrumney — eww.scss
.icon-module {
  margin: 0 5px;
  & > &__icon {
    margin-right: 5px;
    font-family: "Font Awesome 5 Free Solid";
  }
}
```

### Deep nesting with & for state classes

From druskus20's simpler-bar/eww.scss — mic, camera, and wifi active states:

```scss
// Source: druskus20 — simpler-bar/eww.scss
.mic {
  .icon.running {
    color: $green;
  }
}

.camera {
  .icon.connected {
    color: $green;
  }
}

.wifi {
  .icon.connected {
    color: $green;
  }
}

.battery {
  .icon.critical {
    color: $red;
  }
}
```

---

## Dynamic Class Pattern — Full Yuck + SCSS Pair

The most important pattern in eww styling. The yuck side sets the class, the SCSS side styles each resulting class combination.

### Battery

```yuck
; yuck
(label :class `battery ${battery < 20 ? "critical" : battery < 50 ? "low" : "ok"}`
       :text battery-text)
```

```scss
.battery         { color: $fg; }
.battery.low     { color: $warning; }
.battery.critical { color: $urgent; }
```

### Workspaces — four status states (druskus20)

```yuck
; yuck — deflisten on sway IPC, ws.status is "empty"/"occupied"/"focused"/"urgent"
(label :class "small-ws status-${ws.status}"
       :text ws.icon)
```

```scss
// Source: druskus20 — workspace-indicator/eww.scss
.ws .small-ws {
  &.status-empty    { color: rgba(237, 112, 112, 0.3); }
  &.status-occupied { color: rgba(237, 112, 112, 0.5); }
  &.status-focused  { color: rgba(237, 112, 112, 1.0); }
  &.status-urgent   { color: rgba(255, 255, 255, 1.0); }
}
```

### Scale with warning/error thresholds (owenrumney)

```yuck
; yuck — compute the scale class from the value
(scale :class {cpu > 90 ? "metric error" : cpu > 75 ? "metric warning" : "metric"}
       :value cpu
       :max 100)
```

```scss
// Source: owenrumney — eww.scss (variable names from gruvbox.scss)
.metric scale trough highlight      { all: unset; background-color: $green;      border-radius: 10px; }
.metric scale.warning trough highlight { all: unset; background-color: $yellow;   border-radius: 10px; }
.metric scale.error   trough highlight { all: unset; background-color: $error;    border-radius: 10px; }
.metric scale.volume  trough highlight { all: unset; background-color: $lightmagenta; border-radius: 10px; }
```

### Scale over 100% (druskus20 .over class)

```yuck
; yuck — the .over class is added when volume exceeds 100
(scale :class {vol > 100 ? "over" : ""}
       :value vol
       :max 150)
```

```scss
// Source: druskus20 — simpler-bar/eww.scss
scale trough highlight      { background-color: $text;  border-radius: 2px; }
scale.over trough highlight { background-color: $peach; border-radius: 2px; }
```

### Network activity (owenrumney)

```yuck
; yuck
(label :class {netspeed > 5 ? "veryuplink" : netspeed > 0.1 ? "uplink" : "noactive"}
       :text net-text)
```

```scss
// Source: owenrumney — eww.scss
.uplink       { color: $lightmagenta; }
.veryuplink   { color: $red; }
.noactive     { color: $white; }
.downlink     { color: $lightcyan; }
.verydownlink { color: $green; }
```

---

## Variables

Define all colors at the top of the theme file. From the real source files:

```scss
// Source: druskus20 — simpler-bar/eww.scss (full variable block)
$rosewater: #f5e0dc;
$flamingo: #f2cdcd;
$pink: #f5c2e7;
$mauve: #cba6f7;
$red: #f38ba8;
$maroon: #eba0ac;
$peach: #fab387;
$yellow: #f9e2af;
$green: #a6e3a1;
$teal: #94e2d5;
$sky: #89dceb;
$sapphire: #74c7ec;
$blue: #89b4fa;
$lavender: #b4befe;
$text: #cdd6f4;
$subtext1: #bac2de;
$subtext0: #a6adc8;
$overlay2: #9399b2;
$overlay1: #7f849c;
$overlay0: #6c7086;
$surface2: #585b70;
$surface1: #45475a;
$surface0: #313244;
$base: #1e1e2e;
$mantle: #181825;
$crust: #11111b;
```

owenrumney adds semantic aliases at the bottom of gruvbox.scss for use in widget rules:

```scss
// Source: owenrumney — gruvbox.scss (semantic alias section)
$warning: $yellow;
$error: $red;
$urgent: $red;
```

---

## @mixin — Reusable Style Blocks

Use `@mixin` for style patterns you apply to multiple selectors.

```scss
// Pill-shaped label/button
@mixin pill($bg-color, $fg-color) {
  background-color: $bg-color;
  color: $fg-color;
  border-radius: 20px;
  padding: 2px 10px;
}

.tag-active   { @include pill($accent, $bg); }
.tag-inactive { @include pill($surface, $fg-muted); }
.tag-urgent   { @include pill($urgent, $bg); }
```

---

## State-Based Styling Patterns

### Binary toggle (button active/inactive)

```yuck
(button :class {active ? "active" : ""})
```

```scss
button        { color: $fg-muted; background-color: transparent; }
button.active { color: $accent; background-color: $surface; }
```

### Three-state (ok/warning/critical)

```yuck
(label :class `status ${val < 20 ? "critical" : val < 50 ? "warning" : "ok"}`)
```

```scss
.status          { }
.status.ok       { color: $ok; }
.status.warning  { color: $warning; }
.status.critical { color: $urgent; }
```

---

## System Tray Styling

The system tray menu is standard GTK. Style with `menu` and `menuitem` type selectors:

```scss
menu {
  padding: 5px;
  background-color: $bg;

  > menuitem {
    font-size: 14px;
    padding: 2px 5px;

    &:disabled label {
      color: $fg-muted;
    }

    &:hover {
      background-color: $surface;
    }
  }

  separator {
    padding-top: 1px;

    &:last-child {
      padding: unset;
    }
  }
}
```

saimoomedits's tooltip pattern from leftbar/eww.scss:

```scss
// Source: saimoomedits — eww/leftbar/eww.scss
tooltip.background {
    background-color: #0f0f17;
    font-size: 18;
    border-radius: 10px;
    color: #bfc9db;
}

tooltip label {
    margin: 6px;
}
```

---

## Transition Animations

GTK supports CSS transitions but not keyframe animations. Use for smooth color changes:

```scss
.workspaces button {
  color: $fg-muted;
  transition: color 150ms ease, background-color 150ms ease;
}

.workspaces button.focused {
  color: $accent;
  background-color: $surface;
}

.battery {
  color: $fg;
  transition: color 300ms ease;
}

.battery.critical {
  color: $urgent;
}
```

---

## Vertical Bar Patterns (rxyhn)

For vertical bars, every dimension convention flips. From bar/eww.scss:

```scss
// Source: rxyhn — config/eww/bar/eww.scss

// Vertical scale: tall (80px) and narrow (10px)
scale trough {
  all: unset;
  background-color: $background;
  border-radius: 5px;
  min-height: 80px;
  min-width: 10px;
  margin: .3rem 0 .3rem 0;   // vertical margin only
}

// Named per-control highlights — different colors per control
.bribar trough highlight { background-color: $yellow; border-radius: 5px; }
.volbar trough highlight { background-color: $green;  border-radius: 5px; }

// Items use top/bottom padding only
.launcher_icon {
  padding: 1rem 0 1rem 0;
}

// Stacked items use vertical margin only
.0, .01, .02, .03, .04, .05, .06,
.011, .022, .033, .044, .055, .066 {
  margin: .55rem 0 .55rem 0;
}
```

---

## Complete Minimal eww.scss (Reference Config)

```scss
// eww.scss — minimal reference

// Variables
$bg:      #1e1e2e;
$bg-alt:  #181825;
$fg:      #cdd6f4;
$muted:   #6c7086;
$accent:  #cba6f7;
$surface: #313244;
$red:     #f38ba8;
$yellow:  #f9e2af;
$green:   #a6e3a1;

* { all: unset; }

// Bar container
.bar {
  background-color: $bg;
  color: $fg;
  padding: 4px 8px;
  font-family: "JetBrainsMono Nerd Font";
  font-size: 13px;
}

// Workspaces
.workspaces {
  background-color: $surface;
  border-radius: 15px;
  padding: 0 5px;

  button {
    color: $muted;
    padding: 0 6px;
    border-radius: 4px;
    font-family: "Hack NF Regular";
    font-size: 20px;

    &.focused { color: $accent; background-color: $bg; }
    &.urgent  { color: $red; }
    &:hover   { color: $fg; }
  }
}

// System metrics (CPU, RAM, etc.) — owenrumney threshold pattern
.metric {
  background-color: $bg-alt;
  padding: 5px 2px;
}

.metric scale trough {
  all: unset;
  background-color: $surface;
  border-radius: 50px;
  min-height: 5px;
  min-width: 100px;
  margin: 0 10px;
}

.metric scale trough highlight             { all: unset; background-color: $green;  border-radius: 10px; }
.metric scale.warning trough highlight     { all: unset; background-color: $yellow; border-radius: 10px; }
.metric scale.error   trough highlight     { all: unset; background-color: $red;    border-radius: 10px; }

// System tray
menu {
  padding: 5px;
  background-color: $bg-alt;

  menuitem {
    padding: 2px 5px;
    font-size: 13px;
    color: $fg;

    &:hover   { background-color: $surface; }
    &:disabled label { color: $muted; }
  }
}

---

## Rounded Corners — Transparent Corners Require rgba alpha < 1.0

GTK only enables RGBA compositing when it detects `alpha < 1.0` in the background color.
With a fully opaque color (`#2E3440` or `rgba(..., 1.0)`), the area outside `border-radius`
renders **black** instead of transparent, even if the compositor supports transparency.

### Wrong (black corners)

```scss
.my-widget {
  background: #2E3440;   // fully opaque — GTK skips RGBA compositing
  border-radius: 14px;   // rounded corners, but outer area is black
}
```

### Correct (transparent corners)

```scss
.my-widget {
  background: rgba(46, 52, 64, 0.99);  // alpha < 1.0 triggers RGBA compositing
  border-radius: 14px;                  // corners are now truly transparent
}
```

`rgba(46, 52, 64, 0.99)` is visually identical to `#2E3440`. The `0.99` alpha is the minimum
change needed — it enables compositing without any visible transparency effect.
```
