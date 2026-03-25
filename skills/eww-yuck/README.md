# eww-yuck skill

Core eww yuck configuration syntax — windows, widgets, variables, and the CLI.

## What this skill covers

- Writing `eww.yuck` config files from scratch
- Defining windows with `defwindow` (geometry, X11 vs Wayland properties)
- Creating reusable widgets with `defwidget`
- Declaring variables: `defvar`, `defpoll`, `deflisten`
- Iterating data with `for` loops
- Splitting config across files with `include`
- Rendering dynamic markup with `literal`
- Running eww from the command line

## File index

| File | Description |
|---|---|
| `SKILL.md` | Main reference: all constructs with examples, decision guides, and critical rules |
| `SYNTAX.md` | Quick-lookup syntax tables for every yuck construct |
| `CLI.md` | Every eww CLI command with exact syntax and common workflows |

## Related skills

- **eww-expressions** — `${}` expression language, operators, functions, JSON access
- **eww-widgets** — built-in widgets and magic variables (EWW_RAM, EWW_CPU, etc.)
- **eww-styling** — GTK CSS/SCSS, theming, eww inspector
- **eww-patterns** — complete working examples: bars, popups, dashboards
- **eww-troubleshooting** — debugging with eww logs, eww state, eww debug
