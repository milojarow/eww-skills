---
name: eww-styling
description: Style eww widgets with GTK CSS/SCSS. Use when writing eww.scss, applying CSS to widgets, using GTK selectors, setting up SCSS variables or theming, using dynamic CSS classes, debugging styles with eww inspector, or applying the required * { all: unset; } reset.
---

# eww-styling skill

GTK CSS/SCSS reference for eww widget styling.

## Files

| File | What it covers |
|---|---|
| `SKILL.md` | Start here. Core concepts, quick start, GTK reset, dynamic classes, inspector usage, common mistakes. |
| `CSS_REFERENCE.md` | Complete GTK CSS reference: all selector types, pseudo-classes, widget sub-elements, working vs. non-working properties, specificity, parse error behavior. |
| `SCSS_PATTERNS.md` | SCSS-specific patterns with real community examples: variables, nesting, @import, @mixin, full yuck+SCSS pairs for battery/workspaces/metrics/tray. |
| `THEMES.md` | Copy-paste color palettes: Catppuccin Mocha, Catppuccin Latte, Gruvbox Dark, Nord, Tokyo Night. Each includes the full SCSS variable block and semantic alias mapping. |

## The one rule that matters most

```scss
* { all: unset; }  /* first line of eww.scss, always */
```

Without it, GTK default styles override everything you write.

## Related skills

- `eww-yuck` — widget definitions and the `:class` attribute
- `eww-widgets` — widget type reference for selector targeting
- `eww-expressions` — expressions used inside `:class {}`
- `eww-troubleshooting` — when styles still won't apply
