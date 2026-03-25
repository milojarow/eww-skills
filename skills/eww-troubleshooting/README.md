# eww-troubleshooting skill

Debug and fix eww widget system problems.

## Files

**SKILL.md** — Start here. Contains the step-by-step debug workflow, top 5 common issues quick reference, how to read `eww logs` / `eww state` / `eww debug`, how to use `eww inspector`, and how to test scripts independently.

**DEBUGGING.md** — Deep dive on each debug tool: log format and patterns, state output interpretation, widget tree reading, GTK inspector walkthrough, `eww reload` vs kill+restart, and script buffering/PATH issues.

**COMMON_ERRORS.md** — Error catalog with exact symptom, cause, and fix for 12 specific errors:
- Window not appearing
- Config parse error
- CSS not applying
- Variable not updating
- Literal `${var}` text in output
- deflisten not receiving data
- Wayland window not appearing
- X11 window not reserving space
- Multiple daemon instances
- Script not found (PATH issue)
- Compilation errors
- `eww reload` not working

## Related skills

- **eww-yuck** — syntax for defwindow, defwidget, defpoll, deflisten
- **eww-expressions** — expression operators, functions, type coercion
- **eww-styling** — GTK CSS properties, specificity, selectors
- **eww-widgets** — built-in widget properties and behavior
- **eww-patterns** — complete working examples to reference
