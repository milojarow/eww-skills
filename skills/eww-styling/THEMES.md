---
name: eww-styling
description: Style eww widgets with GTK CSS/SCSS. Use when writing eww.scss, applying CSS to widgets, using GTK selectors, setting up SCSS variables or theming, using dynamic CSS classes, debugging styles with eww inspector, or applying the required * { all: unset; } reset.
---

# THEMES — Color Palettes for eww

Copy-paste ready SCSS variable blocks sourced directly from real community dotfiles. Every color block here is the actual code from the referenced repo — not invented or approximated.

**Usage pattern for all themes:**

1. Save the theme block as `~/.config/eww/themes/<theme-name>.scss`
2. `@import "./themes/<theme-name>";` at the top of `eww.scss`
3. Use variables directly in widget styles

---

## Catppuccin Mocha (dark)

// Source: druskus20 — simpler-bar/eww.scss

Exact variable declarations as they appear in the file — no aliasing, no semantic renames. The variable names are already semantic (Catppuccin's own naming).

```scss
// Source: druskus20 — simpler-bar/eww.scss
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

The full bar rules as they appear in simpler-bar/eww.scss — `* { all: unset; }` comes after the variable block, before any class rules:

```scss
// Source: druskus20 — simpler-bar/eww.scss

* {
  all: unset;
}

window {
  font-family: "NotoSans Nerd Font", sans-serif;
  background-color: $base;
}

.workspace {
  margin: 3px;
  padding: 0px;
}

.icon {
  margin: 0 5px;
  color: $text;
}

.icon.single-icon {
  margin: 0;
}

.mic {
  .icon.running {
    color: $green;
  }
}

.workspace {
  .icon {
    margin: 0 3px;
    font-size: 10px;
  }
  .name {
    background-color: $base;
  }
}

.workspace.focused {
  .icon {
    color: $text;
  }
}

.battery, .time, .date .tray .speakers .mic .wifi {
  color: $text;
  font-size: 10px;
}

.battery {
  .icon.critical {
    color: $red;
  }
}

.tooltip {
  background-color: $surface0;
  color: $text;
  font-size: 10px;
}

.camera {
  .icon.connected {
    color: $green;
  }
}

.icon.muted {
  color: $red;
}

.wifi {
  .icon.connected {
    color: $green;
  }
}

.tray image {
  -gtk-icon-transform: scale(0.5);
}

// $crust trough, $text highlight, $peach when volume exceeds 100% (.over class)
scale trough {
  background-color: $crust;
  border-radius: 2px;
  min-height: 5px;
  min-width: 50px;
}

scale trough highlight {
  background-color: $text;
  border-radius: 2px;
}

scale.over trough highlight {
  background-color: $peach;
  border-radius: 2px;
}
```

Key observations from druskus20's approach:
- `* { all: unset; }` appears after variable declarations, before any class rules
- Catppuccin's own names (`$green`, `$red`, `$peach`) are already semantic — no `$ok`/`$urgent` aliases needed
- The `.over` class on the scale widget signals volume > 100%; the SCSS turns the highlight `$peach`
- `$base` for window background, `$text` for all labels, `$crust` for scale troughs

---

## Gruvbox Dark (owenrumney)

// Source: owenrumney — gruvbox.scss

This is a custom Gruvbox variant. Background is `#0d1117` (darker than stock Gruvbox `bg0-hard` of `#1d2021`) and foreground is desaturated to `#b0b4bc`. Exact file as deployed:

```scss
// Source: owenrumney — gruvbox.scss
$background: #0d1117;
$foreground: #b0b4bc;

$background-alt: #232320;

$black: #000;
$white: #fff;

$green: #98971a;
$lightgreen: #b8bb26;
$yellow: #d79921;
$lightyellow: #fabd2d;
$red: #cc241d;
$lightred: #fb4934;
$blue: #0db7ed;
$lightblue: #83a598;
$magenta: #b16286;
$lightmagenta: #d3869b;
$cyan: #689d6a;
$lightcyan: #8ec07c;
$gray: #928374;
$lightgray: #a89984;

$warning: $yellow;
$error: $red;

$urgent: $red;
```

owenrumney's eww.scss `@import`s gruvbox.scss and then applies the palette. The full eww.scss as deployed — note the `@import` is first, then `* { all: unset; }`, then rules:

```scss
// Source: owenrumney — eww.scss

@import "gruvbox.scss";

* {
  all: unset; // Unsets everything so you can style everything from scratch
}

.bar {
  background-color: $background;
  color: $foreground;
  padding: 5px;
  font-size: 1em;
  font-family: Hack Regular;
}

.bottombar {
  background-color: $background;
  color: $foreground;
  padding: 5px;
  font-size: 1em;
  font-family: Hack Regular;
}

.icon-module {
  margin: 0 5px;
  & > &__icon {
    margin-right: 5px;
    font-family: "Font Awesome 5 Free Solid";
  }
}

.sidestuff slider {
  all: unset;
  color: #ffd5cd;
}

// Metric scale — four named variants, each with all: unset on trough/highlight
// The class (.warning, .error, .volume) is on the scale widget itself, not its parent
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

// Network activity — five named classes, color-coded by direction and speed
.uplink       { color: $lightmagenta; }
.veryuplink   { color: $red; }
.noactive     { color: $white; }
.downlink     { color: $lightcyan; }
.verydownlink { color: $green; }
```

Critical detail: `all: unset` appears *inside each individual trough and highlight rule*, not only in the global reset. GTK's default scale sub-element rendering needs a targeted reset per element — the global `* { all: unset; }` does not fully reach nested widget nodes.

---

## Nord (isparsh)

// Source: isparsh — eww/eww.scss

isparsh uses raw Nord hex values directly rather than named SCSS variables. The standout technique is a `linear-gradient` on the trough highlight spanning all three Frost blues. Full eww.scss as deployed:

```scss
// Source: isparsh — eww/eww.scss

* {
  all: unset;
  font-family: "JetBrains Mono";
}

.bg {
  background-color: #1E222A;
  border: 2px solid #80A0C0;
}

.notes {
  padding: 10px 20px 10px 5px;
}

window {
  background: #2e3440;
  color: #ebcb8b;
  border: 3px solid #80A0C0;
}

// Gradient scale: Frost 2 (#88c0d0) -> Frost 3 (#81a1c1) -> Frost 4 (#5E81ac)
// Light to deep — fill visually "deepens" as the metric value climbs
.metric scale trough highlight {
  all: unset;
  color: #000000;
  background: linear-gradient(90deg, #88c0d0 0%, #81a1c1 50%, #5E81ac 100%);
}

.metric scale trough {
  all: unset;
  background-color: #545454;
  min-height: 10px;
  min-width: 100px;
  margin-left: 10px;
  margin-right: 5px;
}

// Aurora Purple (#B48EAD) for metric labels
.metric .label {
  color: #B48EAD;
  font-size: 1.5rem;
}

.quote-text {
  padding: 5px;
  font-size: 1rem;
  font-style: italic;
  font-weight: 600;
  color: #BF616A;   // Aurora Red
}

.cal {
  color: #8fbcbb;   // Frost 1 / teal
  padding: 5px;
  padding-bottom: 1px;
}
```

Key color values from isparsh's file and their Nord identities:
- `#2e3440` — Polar Night 1 (main background)
- `#80A0C0` — between Frost 3 and 4 (border color)
- `#ebcb8b` — Aurora Yellow (used as foreground text)
- `#88c0d0` — Frost 2 (sky blue, gradient start)
- `#81a1c1` — Frost 3 (medium blue, gradient mid)
- `#5E81ac` — Frost 4 (deep blue, gradient end)
- `#B48EAD` — Aurora Purple (metric labels)
- `#BF616A` — Aurora Red (quote text)
- `#8fbcbb` — Frost 1 / teal (calendar)

Note: `font-family` and `all: unset` share the same `*` rule — both declarations coexist on the global selector.

Full canonical Nord palette for reference when building a complete theme file:

```scss
// Canonical Nord palette — nordtheme.com

// Polar Night — dark backgrounds
$polar-night-1: #2e3440;   // darkest, main background
$polar-night-2: #3b4252;
$polar-night-3: #434c5e;
$polar-night-4: #4c566a;   // lightest dark, borders/inactive

// Snow Storm — light foreground
$snow-storm-1:  #d8dee9;   // primary text
$snow-storm-2:  #e5e9f0;
$snow-storm-3:  #eceff4;   // brightest white

// Frost — blue-cyan accents
$frost-1:       #8fbcbb;   // teal
$frost-2:       #88c0d0;   // sky blue (most used accent)
$frost-3:       #81a1c1;   // medium blue
$frost-4:       #5e81ac;   // deep blue

// Aurora — colorful accents
$aurora-red:    #bf616a;
$aurora-orange: #d08770;
$aurora-yellow: #ebcb8b;
$aurora-green:  #a3be8c;
$aurora-purple: #b48ead;
```

---

## rxyhn (Tokyo Night variant)

// Source: rxyhn — config/eww/dashboard/src/scss/variables.scss

rxyhn uses two separate variable files — one for the dashboard, one for the bar. The hues are shared but some names differ.

**Dashboard palette** (variables.scss — exact file contents):

```scss
// Source: rxyhn — config/eww/dashboard/src/scss/variables.scss
$background: #1A1B26;
$background-alt: #16161E;
$background-alt2: #1E202E;
$foreground: #a9b1d6;
$red: #F7768E;
$yellow: #E0AF68;
$orange: #FF9E64;
$green: #9ECE6A;
$blue: #7AA2F7;
$blue2: #88AFFF;
$magenta: #BB9AF7;
$cyan: #73DACA;
```

**Bar palette** — defined inline at the top of bar/eww.scss:

```scss
// Source: rxyhn — config/eww/bar/eww.scss (variable section)
$background: #1A1B26;
$foreground: #A9B1D6;

$black:   #24283B;   // surface/card color
$gray:    #565F89;   // inactive/muted
$red:     #F7768E;
$green:   #73DACA;   // rxyhn uses teal as "green" — intentional
$yellow:  #E0AF68;
$blue:    #7AA2F7;
$magenta: #BB9AF7;
$cyan:    #7DCFFF;
$white:   $foreground;
```

Notable quirk: `$green` in rxyhn's bar is `#73DACA` — a teal, not a green. In the dashboard variables.scss `$green` is `#9ECE6A` (actual green) and `$cyan` is `#73DACA`. The bar swaps these names.

rxyhn's bar is **vertical** — one of the few real-world vertical eww bar examples. Key vertical bar patterns from bar/eww.scss:

```scss
// Source: rxyhn — config/eww/bar/eww.scss

* {
  all: unset;
}

.eww_bar {
  background-color: $background;
  padding: .3rem;
}

// Vertical scale: tall (80px) and narrow (10px)
scale trough {
  all: unset;
  background-color: $background;
  border-radius: 5px;
  min-height: 80px;   // tall for vertical orientation
  min-width: 10px;    // narrow
  margin: .3rem 0 .3rem 0;
}

// Named per-control highlights
.bribar trough highlight { background-color: $yellow; border-radius: 5px; }
.volbar trough highlight { background-color: $green;  border-radius: 5px; }

// Workspace container
.works {
  font-family: "Font Awesome 6 Pro";
  padding: .2rem .7rem .2rem .7rem;
  background-color: $black;
  border-radius: 5px;
}

// Workspace icons: unoccupied/occupied share $gray, focused gets $foreground
.0, .01, .02, .03, .04, .05, .06,
.011, .022, .033, .044, .055, .066 {
  margin: .55rem 0 .55rem 0;
}
.0 { color: $gray; }
.01, .02, .03, .04, .05, .06 { color: $gray; }
.011, .022, .033, .044, .055, .066 { color: $foreground; }

// Each workspace icon gets a different font-size for visual variety
.01, .011, .03, .033 { font-size: 1.5em; }
.02, .022            { font-size: 1.4em; }
.04, .05, .044, .055 { font-size: 1.6em; }
.06, .066            { font-size: 1.7em; }

calendar:selected      { color: $blue; }
calendar.header        { color: $blue; font-weight: bold; }
calendar.button        { color: $magenta; }
calendar.highlight     { color: $magenta; font-weight: bold; }
calendar:indeterminate { color: $background; }
```

For a vertical bar: use `padding: Xrem 0` (top/bottom only) on items, `min-height: 80px; min-width: 10px` on scale troughs, `margin: .Xrem 0 .Xrem 0` (no horizontal margins) on stacked elements.

---

## Comparison Table — 4 Main Themes

| Property        | Catppuccin Mocha (druskus20) | Gruvbox Dark (owenrumney) | Nord (isparsh)    | rxyhn (Tokyo Night) |
|---|---|---|---|---|
| Background      | `#1e1e2e`                    | `#0d1117`                 | `#2e3440`         | `#1A1B26`           |
| Surface/card    | `#313244`                    | `#232320`                 | `#434c5e`         | `#24283B`           |
| Primary text    | `#cdd6f4`                    | `#b0b4bc`                 | `#d8dee9`         | `#A9B1D6`           |
| Muted/inactive  | `#6c7086`                    | `#928374`                 | `#4c566a`         | `#565F89`           |
| Main accent     | `#cba6f7` (mauve)            | `#fabd2d` (lightyellow)   | `#88c0d0` (frost) | `#7AA2F7` (blue)    |
| Error/critical  | `#f38ba8` (red)              | `#cc241d` (red)           | `#bf616a`         | `#F7768E`           |
| Warning         | `#f9e2af` (yellow)           | `#d79921` (yellow)        | `#ebcb8b`         | `#E0AF68`           |
| OK/success      | `#a6e3a1` (green)            | `#b8bb26` (lightgreen)    | `#a3be8c`         | `#9ECE6A`           |
| Scale style     | solid `$text`, `$peach` over | solid per class name       | gradient 3 frosts | per-control named   |
| Bar orientation | horizontal                   | horizontal                | horizontal        | vertical            |
| Workspace style | `.focused` class             | `.focused` class          | N/A in source     | numeric `.011` etc. |
| Font            | NotoSans NF                  | Hack Regular               | JetBrains Mono    | Comic Mono / FA6    |

---

## "I want X's colors" — Quick Reference

| You say...                   | Use these variables                                                              |
|---|---|
| "I want druskus20's colors"  | Catppuccin Mocha: `$base: #1e1e2e`, `$text: #cdd6f4`, `$mauve: #cba6f7`        |
| "I want owenrumney's colors" | Gruvbox custom: `$background: #0d1117`, `$foreground: #b0b4bc`, `$lightmagenta: #d3869b` for volume |
| "I want isparsh's colors"    | Nord: `#2e3440` bg, Frost gradient `#88c0d0 → #81a1c1 → #5E81ac`, `#ebcb8b` fg |
| "I want rxyhn's bar colors"  | Tokyo Night bar: `$background: #1A1B26`, `$green: #73DACA` (teal!), `$cyan: #7DCFFF` |
| "I want rxyhn's dashboard"   | Tokyo Night dashboard: `$background: #1A1B26`, `$green: #9ECE6A`, `$cyan: #73DACA` |

---

## Generic Minimal Template (Custom Theme)

Starting point when building a custom palette. Define raw colors first, then alias them semantically so you can swap themes by changing one `@import` line.

```scss
// themes/my-theme.scss

// Raw palette
$my-dark1:  #111118;
$my-dark2:  #1a1a22;
$my-dark3:  #252530;
$my-light1: #c8cde0;
$my-light2: #9298b0;
$my-accent: #89b4fa;
$my-red:    #f38ba8;
$my-yellow: #f9e2af;
$my-green:  #a6e3a1;

// Semantic aliases — use these in all widget styles
$bg:      $my-dark1;
$bg-alt:  $my-dark2;
$surface: $my-dark3;
$fg:      $my-light1;
$fg-muted: $my-light2;
$accent:  $my-accent;
$urgent:  $my-red;
$warning: $my-yellow;
$ok:      $my-green;
```

Import in eww.scss:

```scss
// eww.scss
@import "./themes/my-theme";

* { all: unset; }

.bar {
  background-color: $bg;
  color: $fg;
}
```

Switching themes later is a one-line `@import` change. All widget rules using `$bg`, `$fg`, `$accent`, etc. automatically follow.
