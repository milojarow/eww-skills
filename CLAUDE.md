# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **eww-skills** repository — a collection of Claude Code skills for building eww widgets on Wayland desktops.

**Repository**: https://github.com/milojarow/eww-skills

**Purpose**: 6 complementary skills that provide expert guidance on eww widget development — from yuck syntax to GTK CSS styling, expression language, widget patterns, and troubleshooting.

## Repository Structure

```
eww-skills/
├── .claude-plugin/        # Claude Code plugin configuration
├── CLAUDE.md              # This file
├── README.md              # Project overview
├── LICENSE                # MIT License
└── skills/                # Individual skill implementations
    ├── eww-expressions/   # Expression language (${}, operators, functions)
    ├── eww-patterns/      # Real-world widget patterns and positioning
    ├── eww-styling/       # GTK CSS/SCSS theming and gotchas
    ├── eww-troubleshooting/ # Debugging, common errors, systemd hardening
    ├── eww-widgets/       # Widget reference, alignment, magic variables
    └── eww-yuck/          # Core yuck syntax, defwindow, deflisten, defpoll
```

## The 6 Skills

### 1. eww-yuck
Core configuration syntax — defwindow, defwidget, defvar, defpoll, deflisten, includes.

### 2. eww-widgets
Widget selection and properties — containers, display, interactive, systray, alignment.

### 3. eww-expressions
Expression language — ternary, elvis, safe access, functions, JSON data access.

### 4. eww-styling
GTK CSS/SCSS — selectors, properties, theming, SCSS patterns, GTK-specific gotchas.

### 5. eww-patterns
Real-world patterns — bar, popup, data subscription, positioning, overlap detection.

### 6. eww-troubleshooting
Debug workflow, common errors catalog, shell script pitfalls, systemd service hardening.

## Skill Activation

Skills activate automatically when queries match their description triggers:
- Editing `.yuck` files → eww-yuck
- Writing SCSS for eww → eww-styling
- Debugging eww issues → eww-troubleshooting
- Building a bar or widget → eww-patterns

## Cross-Skill Integration

Skills reference each other:
- eww-patterns references eww-widgets for alignment details
- eww-troubleshooting references eww-styling for CSS gotchas
- eww-expressions references eww-yuck for variable definitions
