---
name: eww-expressions
description: Write and debug eww expressions. Use when writing ${} expressions, using operators (ternary ?:, elvis ?:, safe access ?.), calling functions (round, jq, formattime, formatbytes, strlength, replace), accessing JSON/object/array data, or filtering with regex =~.
---

# eww-expressions — Navigation Guide

Reference files for the eww expression language.

## Files

**SKILL.md** — Start here. Quick start examples, syntax rules, common mistakes, and real-world patterns. Covers the `${}` vs `{}` distinction, the top 5 functions, and integration with other eww skills.

**OPERATORS.md** — Complete operator reference. Arithmetic, comparison, boolean, ternary (`? :`), elvis (`?:`), safe access (`?.`), regex match (`=~`), precedence table, and complex multi-operator examples.

**FUNCTIONS.md** — All functions grouped by category: math (`round`, `floor`, `ceil`, `min`, `max`, `powi`, `powf`, `log`, trig), string (`strlength`, `substring`, `replace`, `search`, `matches`, `captures`), array/object (`arraylength`, `objectlength`), and utility (`jq`, `get_env`, `formattime`, `formatbytes`). Each entry includes signature, return type, and practical examples.

**DATA_ACCESS.md** — Data access patterns: simple variables, dot and bracket notation, array indexing, nested chaining, all EWW_* magic variables with their fields, defining JSON variables with `defvar`/`defpoll`/`deflisten`, and jq patterns for complex transformations.

## Quick Lookup

| Need | File |
|---|---|
| `${}` vs `{}` syntax rule | SKILL.md — Syntax Rules |
| Ternary, elvis, safe access | OPERATORS.md |
| `formattime` format codes | FUNCTIONS.md |
| `EWW_BATTERY`, `EWW_RAM` fields | DATA_ACCESS.md |
| `jq()` filter examples | DATA_ACCESS.md, FUNCTIONS.md |
| Common mistakes | SKILL.md — Common Mistakes |
