---
name: eww-styling
description: Style eww widgets with GTK CSS/SCSS. Use when writing eww.scss, applying CSS to widgets, using GTK selectors, setting up SCSS variables or theming, using dynamic CSS classes, debugging styles with eww inspector, or applying the required * { all: unset; } reset.
---

# CSS_REFERENCE — GTK CSS Complete Reference for eww

This file is a comprehensive reference for CSS in eww. GTK CSS is a subset of web CSS with some GTK-specific extensions. Not everything from web CSS works.

---

## Selector Types

### Type selector — matches GTK widget class name

```css
box         { }
label       { }
button      { }
scale       { }
progressbar { }
image       { }
window      { }
entry       { }
checkbutton { }
revealer    { }
menu        { }
menuitem    { }
separator   { }
```

Type selectors in GTK CSS use the lowercase GTK widget type name. These are NOT HTML tags. Use `eww inspector` → CSS Nodes to find the exact name for any widget.

### Class selector

```css
.myclass        { }
.bar            { }
.workspaces     { }
.battery.critical { }   /* element with BOTH classes */
```

### ID selector

```css
#my-widget-id   { }     /* matches :id "my-widget-id" in yuck */
```

IDs in eww are set with the `:id` attribute on any widget. They have the highest specificity of any selector short of `!important`.

### Descendant selector — any level deep

```css
box label               { }   /* label anywhere inside box */
.bar label              { }
.workspaces button      { }
```

### Child selector — direct children only

```css
box > label             { }   /* label that is a direct child of box */
window > box            { }
```

### Combined selectors

```css
button.active           { }   /* button element with class "active" */
label.critical          { }
.module button.focused  { }
.workspaces button.urgent { }
```

### Multiple selectors — comma-separated

```css
.warning, .critical     { color: #f38ba8; }
button, label           { font-family: "Nerd Font"; }
```

---

## Pseudo-Classes

```css
button:hover            { }   /* pointer is over the widget */
button:active           { }   /* widget is being clicked/pressed */
button:focus            { }   /* widget has keyboard focus */
button:disabled         { }   /* widget is insensitive/disabled */
checkbutton:checked     { }   /* checkbox is checked */
widget:first-child      { }   /* first sibling in its parent */
widget:last-child       { }   /* last sibling in its parent */
widget:nth-child(2)     { }   /* second sibling (1-based) */
widget:nth-child(odd)   { }
widget:nth-child(even)  { }
widget:not(.myclass)    { }   /* does not have class "myclass" */
```

Pseudo-class chaining:

```css
button.active:hover     { }   /* active button being hovered */
```

---

## GTK Widget Type Names and Sub-Elements

These are the GTK internal types. Sub-elements shown below are styled with descendant selectors.

### box

No sub-elements. Style directly:

```css
box { background-color: #1e1e2e; padding: 4px; }
```

### label

No sub-elements. Style directly:

```css
label { color: #cdd6f4; font-size: 13px; }
```

### button

No required sub-elements for basic use. Has a label child:

```css
button              { padding: 4px 8px; border-radius: 4px; }
button label        { color: #cdd6f4; }   /* text inside button */
button:hover        { background-color: #313244; }
button:active       { background-color: #45475a; }
```

### scale (slider)

Three sub-elements: `trough`, `highlight` (child of trough), `slider`:

```css
scale trough           { background-color: #313244; min-height: 4px; min-width: 80px; }
scale trough highlight { background-color: #cba6f7; border-radius: 10px; }
scale slider           { background-color: #cdd6f4; border-radius: 50%; min-width: 10px; min-height: 10px; }
```

Note: `scale trough` matches both the trough element AND the highlight (which is inside it). Always define `scale trough highlight` after `scale trough` to override the highlight-specific color.

### progressbar

Two sub-elements: `trough` and `progress`:

```css
progressbar trough   { background-color: #313244; min-height: 6px; border-radius: 3px; }
progressbar progress { background-color: #cba6f7; border-radius: 3px; }
```

### image

No CSS sub-elements. Use `-gtk-icon-source` for icons or style the container.

### entry (text input)

```css
entry                { background-color: #313244; color: #cdd6f4; padding: 4px; border-radius: 4px; }
entry:focus          { border: 1px solid #cba6f7; }
entry selection      { background-color: #cba6f7; color: #1e1e2e; }
```

### checkbutton

```css
checkbutton              { color: #cdd6f4; }
checkbutton check        { background-color: #313244; border: 1px solid #6c7086; }
checkbutton:checked check { background-color: #cba6f7; }
```

### window (eww window container)

```css
window { background-color: transparent; }   /* for transparent bars/windows */
```

### menu and menuitem (system tray context menus)

```css
menu                     { background-color: #1e1e2e; padding: 4px; border-radius: 4px; }
menuitem                 { padding: 3px 8px; font-size: 13px; color: #cdd6f4; }
menuitem:hover           { background-color: #313244; }
menuitem:disabled label  { color: #6c7086; }
separator                { background-color: #313244; min-height: 1px; margin: 2px 0; }
```

---

## CSS Properties That WORK in GTK

### Layout and sizing

```css
padding: 4px 8px;            /* top-bottom left-right */
padding-top: 4px;
padding-right: 8px;
padding-bottom: 4px;
padding-left: 8px;

margin: 2px;
margin-top: 4px;
margin-right: 0;
margin-bottom: 4px;
margin-left: 0;

min-width: 100px;            /* use this instead of width */
min-height: 20px;            /* use this instead of height */

spacing: 8px;                /* GTK-specific: gap between children in box */
```

### Typography

```css
font-family: "JetBrainsMono Nerd Font";
font-size: 13px;
font-size: 1em;
font-size: large;            /* keywords work: small, medium, large, x-large */
font-weight: bold;
font-weight: 600;            /* numeric weights work */
font-style: italic;
font-style: normal;
letter-spacing: 1px;
```

### Color and background

```css
color: #cdd6f4;
color: rgba(205, 214, 244, 0.8);
color: rgb(205, 214, 244);

background-color: #1e1e2e;
background-color: transparent;
background-color: rgba(30, 30, 46, 0.9);

background-image: url("/path/to/image.png");
background-image: linear-gradient(to right, #cba6f7, #89b4fa);

opacity: 0.9;                /* 0.0–1.0 */
```

### Border

```css
border: 1px solid #313244;
border: none;
border-radius: 8px;
border-top-left-radius: 4px;
border-top-right-radius: 4px;
border-bottom-left-radius: 4px;
border-bottom-right-radius: 4px;

border-top: 2px solid #cba6f7;
border-bottom: 1px solid #313244;
border-left: none;
border-right: none;
```

### Box shadow

```css
box-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
box-shadow: inset 0 1px 2px rgba(0, 0, 0, 0.3);
```

### Transition (animation)

```css
transition: 200ms ease;
transition: color 150ms ease, background-color 150ms ease;
transition: all 200ms linear;
```

GTK supports basic transitions on color, background-color, opacity, and border-radius.

### Outline (focus ring)

```css
outline: none;              /* remove focus ring */
outline: 1px solid #cba6f7;
outline-offset: 2px;
```

### Text decoration

```css
text-decoration: underline;
text-decoration: none;
```

---

## CSS Properties That DO NOT WORK or Are Limited

| Property | Status | Alternative |
|---|---|---|
| `width` | Ignored on most widgets | Use `min-width` |
| `height` | Ignored on most widgets | Use `min-height` |
| `display: flex` | Not supported | Use eww `box` with `:orientation` and `:space-evenly` |
| `display: grid` | Not supported | Use nested eww `box` widgets |
| `float` | Not supported | Use eww layout attributes |
| `position: absolute` | Not supported | Use eww `overlay` widget |
| `position: relative` | Not supported | — |
| `z-index` | Not supported | Use `overlay` widget |
| `overflow` | Limited support | — |
| `cursor` | Partially works | — |
| `content` | Not supported | Use eww `label` with `:text` |
| `@keyframes` | **Supported** — see grass limitations below | — |
| `gap` | Not supported by grass | Use `margin` on children or `:spacing` in yuck |
| `calc()` | Limited support | Avoid in GTK CSS |
| `var()` | Not supported | Use SCSS variables `$name` instead |
| `clamp()` | Not supported | — |
| `flexbox properties` | Not supported | Use eww box widget attributes |

---

## grass SCSS Compiler Limitations

eww uses **grass** (a Rust SCSS compiler) to compile `eww.scss` before passing the result to GTK. Grass is not 100% spec-compliant — some valid CSS/SCSS that works in `sassc` or `dart-sass` will fail in grass.

**Critical:** a grass compile error stops stylesheet loading at that line. Every rule after the error disappears silently — **ALL widgets lose their styles**, not just the one being edited. Always check `journalctl --user -u eww.service` after any SCSS edit.

### Known grass limitations (confirmed)

#### 1. `@keyframes` works — but no comma-separated keyframe selectors

Grass supports `@keyframes` (the bt-scan-pulse animation in bt-widget.scss is proof). However, grass **does not support comma-separated selectors inside a `@keyframes` block**:

```scss
/* ❌ FAILS in grass — "Expected closing bracket after keyframes block" */
@keyframes my-anim {
  0%, 100% { opacity: 0.2; }
  50%      { opacity: 1;   }
}

/* ✅ CORRECT — expand each selector to its own line */
@keyframes my-anim {
  0%   { opacity: 0.2; }
  50%  { opacity: 1;   }
  100% { opacity: 0.2; }
}
```

Note: `sassc` accepts the comma form without error. Grass does not. This is a grass bug.

#### 2. `gap` is not recognized

The `gap` CSS property (used for flex/grid spacing) is not accepted by grass:

```scss
/* ❌ FAILS — "'gap' is not a valid property name" */
.my-row {
  gap: 4px;
}

/* ✅ CORRECT — use margin on children */
.my-row > * {
  margin: 0 2px;
}

/* ✅ ALSO CORRECT — use :spacing in yuck */
/* (box :class "my-row" :spacing 4 :orientation "h" ...) */
```

### Diagnostic

```bash
# After any SCSS edit, immediately check for errors:
journalctl --user -u eww.service -n 20 --no-pager | grep "error:"

# Keyframe comma-selector error:
# error: Expected closing bracket after keyframes block
#     476 │   0%, 100% {

# Invalid property error:
# error: 'gap' is not a valid property name
```

---

## Specificity Rules in GTK CSS

GTK follows standard CSS specificity:

```
!important > inline style > ID > class/pseudo-class > type
```

Specificity weight (simplified):

| Selector | Specificity |
|---|---|
| `*`                | 0,0,0 |
| `label`            | 0,0,1 |
| `.myclass`         | 0,1,0 |
| `#myid`            | 1,0,0 |
| `button.active`    | 0,1,1 |
| `.bar button`      | 0,1,1 |
| `#id .class label` | 1,1,1 |

When two rules have equal specificity, the **later rule wins** (cascade order). This is why `* { all: unset; }` must come first — it has the lowest possible specificity (0,0,0), so any rule you write afterward will override it.

---

## Parse Error Behavior

eww stops loading the stylesheet at the **first parse error**. Every rule after the error is silently ignored.

Symptoms:
- Styles that "suddenly stopped working" after an edit
- Some widgets are styled but others are not
- Inspector shows no custom rules on widgets you expect to be styled

Debugging:
1. Run `eww inspector` and click a widget that should have styles
2. In the CSS tab, check if any of your rules appear
3. If no rules appear at all, there is a parse error near the top of the file
4. Bisect: comment out the bottom half of `eww.scss`, reload, check; continue until you find the error

Common parse errors:
- Missing semicolon: `color: #cba6f7`  (missing `;`)
- Unclosed brace: `.bar { background: $bg;`  (missing `}`)
- Invalid `@import` path (file does not exist)
- SCSS variable used before it is defined
- Wrong SCSS syntax (CSS written where SCSS expects something different)
- **Invalid web CSS property used** (see Gotchas below)

---

## GTK CSS Gotchas — Properties That Don't Work

These web CSS properties either don't exist in GTK CSS or behave differently. Using them can cause silent failures or break ALL styling.

### `box-shadow` is always rectangular

GTK CSS renders `box-shadow` as a **rectangle** regardless of `border-radius`. A circular element (`border-radius: 50%`) will have a square shadow.

```scss
// ❌ WRONG — shadow is a square behind a circle
.orb {
  border-radius: 50%;
  box-shadow: 0 0 14px rgba(158, 206, 106, 0.45); // renders as rectangle
}

// ✅ CORRECT — simulate glow with thick semi-transparent border
.orb {
  border-radius: 50%;
  border: 6px solid rgba(158, 206, 106, 0.30); // follows the circle
}
```

### `cursor: pointer` is not valid GTK CSS

GTK CSS does not support the `cursor` property. Using it causes a **parse error that breaks ALL SCSS** — every widget in every window loses its styling.

```scss
// ❌ WRONG — breaks ALL eww styling globally
.my-button {
  cursor: pointer;
}

// ✅ CORRECT — set cursor in yuck, not CSS
// (eventbox :cursor "pointer" (button ...))
```

Valid cursor values in yuck `(eventbox :cursor ...)`: `"pointer"`, `"crosshair"`, `"text"`, `"default"`, `"help"`, `"wait"`, `"progress"`.
