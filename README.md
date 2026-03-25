# eww-skills

**Expert Claude Code skills for building [eww](https://github.com/elkowar/eww) widgets**

## What is this?

This repository contains **6 complementary Claude Code skills** that teach AI assistants how to build production-ready eww widgets for Wayland desktops.

### Why These Skills Exist

Building eww widgets can be challenging:
- GTK CSS differs significantly from web CSS
- The yuck configuration language has unique syntax
- Expression language has non-obvious behaviors
- Debugging requires specific tools and techniques

These skills solve these problems by teaching Claude:
- Correct GTK CSS/SCSS patterns (and what doesn't work)
- Yuck syntax for windows, widgets, and data sources
- Expression operators, functions, and JSON data access
- Proven widget patterns from real dotfiles
- Systematic debugging workflow and common error fixes

## The 6 Skills

| Skill | Description |
|-------|-------------|
| **eww-yuck** | Core configuration syntax — defwindow, defwidget, defvar, defpoll, deflisten |
| **eww-widgets** | Widget reference — containers, display, interactive, alignment, magic variables |
| **eww-expressions** | Expression language — ternary, elvis, safe access, functions, data access |
| **eww-styling** | GTK CSS/SCSS — selectors, properties, theming, gotchas (box-shadow, cursor) |
| **eww-patterns** | Real-world patterns — bars, popups, positioning, overlap detection |
| **eww-troubleshooting** | Debug workflow, common errors, shell pitfalls, systemd hardening |

## Installation

Add this marketplace in Claude Code:

```
/plugin → Marketplaces → Add Marketplace → milojarow/eww-skills
```

Then install:

```
/plugin → Discover → eww-skills → Install
```

## Requirements

- [eww](https://github.com/elkowar/eww) installed
- A Wayland compositor (sway, Hyprland, etc.)
- Claude Code

## License

MIT
