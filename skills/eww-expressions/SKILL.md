---
name: eww-expressions
description: Write and debug eww expressions. Use when writing ${} expressions, using operators (ternary ?:, elvis ?:, safe access ?.), calling functions (round, jq, formattime, formatbytes, strlength, replace), accessing JSON/object/array data, or filtering with regex =~.
metadata:
  priority: 5
  pathPatterns: ["**/eww/**/*.yuck"]
  bashPatterns: []
---

# eww-expressions — Expression Language

Yuck includes a small expression language for math, conditionals, JSON access, and function calls.
Expressions appear anywhere inside `{ }` in attributes, or inside `${ }` within strings.

---

## Quick Start: 10 Most Common Expressions

These cover the majority of real-world eww use cases.

```yuck
; Clock — format Unix timestamp
{formattime(EWW_TIME, "%H:%M")}

; RAM percent, rounded to integer
{round(EWW_RAM.used_mem_perc, 0)}

; Disk usage percent (computed from free/total bytes)
{round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}

; Battery capacity from magic variable
{EWW_BATTERY.BAT0.capacity}

; Conditional text based on variable
{battery < 20 ? "Low battery!" : "OK"}

; Conditional CSS class in attribute (no string context)
:class {active ? "active" : "inactive"}

; Conditional CSS class in backtick string (mixed context)
:class `base-class ${active ? "active" : ""}`

; Elvis — provide default when value is empty or null
{value ?: "N/A"}

; Safe access — null-safe field access, chained with elvis
{data?.field ?: "missing"}

; String interpolation in a label attribute
(label :text "CPU: ${round(EWW_CPU.avg, 0)}%")
```

---

## Syntax Rules: `${}` vs `{}`

This is the single most common source of errors. The rule is simple but absolute.

### The Rule

| Context | Syntax | Notes |
|---|---|---|
| Attribute value (not in a string) | `{ expr }` | Direct expression |
| Inside a quoted string `"..."` | `${ expr }` | String interpolation |
| Inside a backtick string `` `...` `` | `${ expr }` | Also supports interpolation |

### Examples

```yuck
; ✅ CORRECT: attribute value — bare expression, no quotes around the {}
(label :text {EWW_RAM.used_mem_perc})
(button :active {count > 0})
(revealer :reveal {show_panel})

; ✅ CORRECT: inside a string — use ${}
(label :text "RAM: ${round(EWW_RAM.used_mem_perc, 0)}%")
(label :text "Time: ${formattime(EWW_TIME, "%H:%M")}")

; ✅ CORRECT: backtick string for dynamic CSS classes
:class `module ${active ? "active" : ""}`
```

```yuck
; ❌ WRONG: using ${} as an attribute value (not inside a string)
(button :active "${count > 0}")
; Result: the string literal "true" or "false", not a boolean

; ❌ WRONG: using {} inside a regular string
(label :text "{EWW_TIME}")
; Result: the literal text "{EWW_TIME}" is displayed, no substitution happens

; ❌ WRONG: missing ${} entirely in string
(label :text "RAM: EWW_RAM.used_mem_perc%")
; Result: literal text, variable never evaluated
```

> CRITICAL: Attribute values that take a boolean, number, or expression must use `{ }` with no surrounding quotes. Wrapping in `"..."` turns the result into a string, breaking boolean/numeric attributes.

---

## Operators

### Arithmetic

```yuck
{a + b}    ; addition
{a - b}    ; subtraction
{a * b}    ; multiplication
{a / b}    ; division
{a % b}    ; modulo (remainder)
```

Note: Division between two integers produces a float in eww expressions.
Use `round()`, `floor()`, or `ceil()` to get integer output.

### Comparison

```yuck
{a == b}   ; equal
{a != b}   ; not equal
{a > b}    ; greater than
{a < b}    ; less than
{a >= b}   ; greater than or equal
{a <= b}   ; less than or equal
```

### Boolean

```yuck
{a && b}   ; logical AND
{a || b}   ; logical OR
{!a}       ; logical NOT
```

### Ternary — conditional expression

```yuck
; Basic form
{condition ? value_if_true : value_if_false}

; Multi-level ternary (battery icon selection)
{battery < 10  ? "󰁺" :
 battery < 25  ? "󰁼" :
 battery < 50  ? "󰁾" :
 battery < 75  ? "󰂀" :
                 "󰁹"}

; Ternary in string interpolation
(label :text "Status: ${online ? "connected" : "offline"}")

; Ternary for conditional CSS class
:class `battery ${EWW_BATTERY.BAT0.capacity < 20 ? "critical" :
                  EWW_BATTERY.BAT0.capacity < 50 ? "low" : "ok"}`
```

### Elvis `?:` — null/empty coalescing

Returns the right side if the left side evaluates to `""` (empty string) or JSON `null`.
Otherwise returns the left side unchanged.

```yuck
; Provide a fallback for a variable that might be empty
{variable ?: "default"}

; Network interface may not exist yet
{EWW_NET.eth0.up ?: 0}

; Custom poll that returns empty string on failure
{name ?: "Unknown"}

; Chain: safe access + elvis for full null protection
{data?.field ?: "fallback"}
```

> NOTE: The elvis operator `?:` is distinct from ternary `? :`. Elvis has no explicit condition — it only checks for empty/null on the left side.

### Safe Access `?.` and `?.[index]`

Returns `null` if the left side is empty or JSON null, instead of crashing.
Otherwise, performs the field or index access normally.

```yuck
; Safe field access
{data?.field}

; Safe array index access
{array?.[0]}

; Chained safe access
{obj?.nested?.deep}

; Combined with elvis for a safe fallback
{data?.field ?: "fallback"}
```

> CAUTION: Safe access still errors if the left side exists but is a number or a plain string (not a JSON object or array). It only protects against the null/empty case, not type mismatches.

### Regex Match `=~`

Tests whether a string matches a Rust-syntax regex pattern.
Returns a boolean.

```yuck
; Left = regex, right = string (reversed from most languages)
{workspace.name =~ "^special:.+$"}
{"Hello World" =~ "^Hello"}
{name =~ "^[A-Z]"}
```

> IMPORTANT: The operand order is `regex =~ string`, not `string =~ regex`. This is the opposite of most languages (Perl, Ruby, bash). The left side is always the pattern.

---

## Data Access

### Simple Variable

```yuck
; In attribute
:value {my_var}

; In string
(label :text "Value: ${my_var}")
```

### Object Field Access

```yuck
; Dot notation (most common)
{EWW_RAM.used_mem_perc}
{weather.temperature}

; Bracket notation (for keys with spaces, special chars, or dynamic keys)
{EWW_TEMPS["coretemp Package id 0"]}
{EWW_DISK["/home"].used_perc}
{object[selected]}  ; dynamic key from variable
```

### Array Index Access

```yuck
{items[0]}           ; first element
{EWW_CPU.cores[0].usage}  ; first core's usage field
{items[2]}           ; third element (0-indexed)
```

### Chained Access

```yuck
; Nested object then array then field
{EWW_BATTERY.BAT0.capacity}
{workspaces[0].name}
{data.users[0].name}
```

### EWW_* Magic Variables

These are provided automatically by eww at runtime.

```yuck
; Battery
{EWW_BATTERY.BAT0.capacity}      ; integer 0-100
{EWW_BATTERY.BAT0.status}        ; "Charging", "Discharging", "Full"

; CPU
{EWW_CPU.avg}                    ; average CPU usage percent
{EWW_CPU.cores[0].usage}         ; per-core usage

; RAM
{EWW_RAM.used_mem_perc}          ; percent used
{EWW_RAM.used_mem}               ; used in bytes
{EWW_RAM.total_mem}              ; total in bytes

; Disk
{EWW_DISK["/"].used_perc}        ; percent used for /
{EWW_DISK["/"].free}             ; free bytes
{EWW_DISK["/"].total}            ; total bytes

; Network
{EWW_NET.wlan0.up}               ; upload bytes/sec
{EWW_NET.wlan0.down}             ; download bytes/sec
{EWW_NET["wlan0"].up}            ; bracket notation equivalent

; Temperatures
{EWW_TEMPS["coretemp Package id 0"]}  ; bracket notation required for keys with spaces

; Time (Unix timestamp, updated every second)
{EWW_TIME}
{formattime(EWW_TIME, "%H:%M")}
```

See DATA_ACCESS.md for the full reference including jq patterns.

---

## Top 5 Most-Used Functions

### 1. `round(number, decimal_digits)`

Rounds a number to the given number of decimal places. Returns 0 decimal places for integers.

```yuck
{round(EWW_RAM.used_mem_perc, 0)}    ; 73
{round(3.14159, 2)}                   ; 3.14
{round(EWW_CPU.avg, 1)}              ; 45.7
```

### 2. `formattime(unix_timestamp, format_str)`

Formats a Unix timestamp using chrono format strings. Uses system local timezone.

```yuck
{formattime(EWW_TIME, "%H:%M")}       ; 14:30
{formattime(EWW_TIME, "%H:%M:%S")}    ; 14:30:05
{formattime(EWW_TIME, "%A, %B %d")}   ; Tuesday, March 04
{formattime(EWW_TIME, "%H:%M", "America/New_York")}  ; with explicit timezone
```

Common format codes: `%H` 24h hour, `%M` minutes, `%S` seconds, `%A` weekday, `%B` month name, `%d` day, `%Y` year.

### 3. `jq(json_string, filter)`

Runs a jq-style filter on a JSON string. Uses jaq internally (compatible with jq syntax).

```yuck
{jq(my_json_var, ".field")}
{jq(my_json_var, ".items[0].name")}
{jq(my_json_var, ".items | length")}
{jq(my_json_var, ".name", "r")}       ; "r" flag = raw output, strips JSON quotes
```

### 4. `formatbytes(bytes, short, format_mode)`

Converts a byte count to a human-readable string.

```yuck
{formatbytes(EWW_RAM.used_mem, false, "iec")}   ; "7.5 GiB" (default)
{formatbytes(EWW_RAM.used_mem, true, "iec")}    ; "7.5G" (short form)
{formatbytes(EWW_RAM.used_mem, false, "si")}    ; "8.1 GB" (SI units)
```

### 5. `replace(string, regex, replacement)`

Replaces all matches of a regex pattern in a string.

```yuck
{replace("foo bar baz", "bar", "qux")}    ; "foo qux baz"
{replace(title, "\\s+", "_")}             ; replace whitespace with underscore
{replace(path, "^/home/[^/]+", "~")}      ; shorten home path
```

See FUNCTIONS.md for the complete function reference.

---

## Common Mistakes

### Mistake 1: `${}` in attribute context

```yuck
; ❌ WRONG: attribute value wrapped in quotes — produces a string, not a boolean
(button :active "${count > 0}")
(revealer :reveal "${show_panel}")

; ✅ CORRECT: attribute value is a bare expression
(button :active {count > 0})
(revealer :reveal {show_panel})
```

### Mistake 2: `{}` inside a string

```yuck
; ❌ WRONG: {} is not substituted inside double-quoted strings
(label :text "Time: {formattime(EWW_TIME, "%H:%M")}")
; Displays: "Time: {formattime(EWW_TIME, "%H:%M")}" literally

; ✅ CORRECT: use ${} for string interpolation
(label :text "Time: ${formattime(EWW_TIME, "%H:%M")}")
```

### Mistake 3: Reversed operands for `=~`

```yuck
; ❌ WRONG: string on the left, regex on the right (other-language habit)
{workspace.name =~ "^special:.+$"}  ; this is actually correct in eww
; Wait — in eww, left IS the regex. So this example is already correct.

; ❌ WRONG: thinking left = string like in bash/Perl
{"^special:.+$" =~ workspace.name}  ; backwards — this tests if workspace.name matches "^special:.+$" as the pattern... no, this is wrong syntax

; ✅ CORRECT: left = regex pattern, right = string to test
{"^special:.+$" =~ workspace.name}   ; NO — keep it as:
{workspace.name =~ "^special:.+$"}   ; regex on left, string on right
```

> To be explicit: `{pattern =~ string}` — the regex pattern is on the LEFT, the string being tested is on the RIGHT. This is the opposite of bash's `[[ string =~ pattern ]]`.

### Mistake 4: Accessing array/object data without a JSON variable

```yuck
; ❌ WRONG: items is not defined as a JSON array variable
{items[0]}  ; error if `items` is not a defvar/defpoll returning a JSON array string

; ✅ CORRECT: define the variable with a JSON array value first
(defvar items `["a", "b", "c"]`)
; Then access works
{items[0]}  ; "a"
```

### Mistake 5: Using safe access on a non-object/non-array

```yuck
; ❌ WRONG: safe access when the value is a number or plain string (not JSON object)
; If `my_num` = "42" (a string), then:
{my_num?.field}  ; error: cannot index a number/string

; ✅ CORRECT: only use ?. when the left side might be null/empty JSON
; Safe access guards against null, not against type errors
{data?.field}    ; safe when data might be null or ""
```

### Mistake 6: Forgetting to use `round()` for display

```yuck
; ❌ WRONG: raw division produces a long float
(label :text "Disk: ${EWW_DISK["/"].used_perc}%")
; Might display: "Disk: 73.47291827364%"

; ✅ CORRECT: round for clean display
(label :text "Disk: ${round(EWW_DISK["/"].used_perc, 0)}%")
; Displays: "Disk: 73%"
```

---

## Real-World Patterns

### Battery Icon with Multi-Level Ternary

```yuck
(defwidget battery []
  (box :class "battery-module"
    (label :text {EWW_BATTERY.BAT0.capacity < 10  ? "󰁺" :
                  EWW_BATTERY.BAT0.capacity < 25  ? "󰁼" :
                  EWW_BATTERY.BAT0.capacity < 50  ? "󰁾" :
                  EWW_BATTERY.BAT0.capacity < 75  ? "󰂀" :
                                                    "󰁹"})
    (label :text "${EWW_BATTERY.BAT0.capacity}%")
    :class `battery ${EWW_BATTERY.BAT0.capacity < 20 ? "critical" :
                      EWW_BATTERY.BAT0.capacity < 50 ? "low" : "ok"}`))
```

### Network Speed Formatting

```yuck
; Convert bytes/sec to MB/s with 2 decimal places
(label :text "Up: ${round(EWW_NET.wlan0.up / 1000000, 2)} MB/s")
(label :text "Down: ${round(EWW_NET.wlan0.down / 1000000, 2)} MB/s")

; Or use formatbytes (shows unit automatically)
(label :text "Up: ${formatbytes(EWW_NET.wlan0.up, true, "iec")}/s")
```

### Time Formatting with EWW_TIME

```yuck
; Simple clock
(label :text {formattime(EWW_TIME, "%H:%M")})

; Full date
(label :text "${formattime(EWW_TIME, "%A")}, ${formattime(EWW_TIME, "%B %d")}")

; Time of day icon
{formattime(EWW_TIME, "%H") < "12" ? "🌅" : "🌙"}
```

### Disk Usage Percent (Computed)

```yuck
; EWW_DISK["/"].used_perc gives the percentage directly
(label :text "Disk: ${round(EWW_DISK["/"].used_perc, 0)}%")

; Or compute manually from free/total (from the official example)
:value {round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}
```

### Music Now Playing (Show Only When Active)

```yuck
(deflisten music :initial ""
  "playerctl --follow metadata --format '{{ artist }} - {{ title }}' || true")

(defwidget music []
  (box :class "music"
    {music != "" ? "🎵 ${music}" : ""}))
```

### Dynamic CSS Class from Variable (data-structures.yuck Pattern)

```yuck
(defvar selected `🦝`)

(eventbox
  :class `animal ${selected == emoji ? "selected" : ""}`
  :onhover "eww update selected=${emoji}"
  emoji)
```

### JSON Poll — Accessing Parsed Fields

```yuck
; defpoll returns a JSON object
(defpoll time_data :interval "10s"
  `date +'{"hour":"%H","min":"%M","sec":"%S"}'`)

; Access fields directly
(label :text "${time_data.hour}:${time_data.min}")
```

---

## Best Practices

1. **Always round floats for display.** Raw division and percentages produce long decimals. Use `round(val, 0)` for integers and `round(val, 1)` or `round(val, 2)` for one or two decimal places.

2. **Use the Elvis operator for variables that might not be set yet.** When eww first starts, polls and listeners may not have data. `{value ?: "default"}` prevents blank or broken widgets during initialization.

3. **Use safe access `?.` for optional nested fields.** When accessing JSON data from an external script, a missing key returns null instead of crashing the widget.

4. **Prefer `formatbytes()` over manual division for byte values.** It handles IEC vs SI, short vs long format, and negative sizes automatically.

5. **Use backtick strings for dynamic CSS classes.** The pattern `:class \`module ${condition ? "active" : ""}\`` is cleaner and less error-prone than building class strings with ternary in quotes.

6. **Test expressions with `eww state` and `eww update`.** Run `eww state` to see all current variable values, then test expressions against them before embedding in widgets.

---

## Integration with Other Skills

- **eww-variables** — Covers `defvar`, `defpoll`, `deflisten`, and how variable types (string, number, JSON) affect expression behavior. Essential for understanding what data is available in expressions.
- **eww-widgets** — Covers the full list of EWW_* magic variables and their field structures. Reference this when accessing `EWW_BATTERY`, `EWW_CPU`, `EWW_NET`, `EWW_DISK`, etc.
- **eww-yuck** — Covers the overall yuck file structure, `defwindow`, `defwidget`, and where expressions appear in context.
- **eww-css** — Covers how CSS class names set via expressions (`:class {expr}` or `:class \`...\``) connect to GTK theming.

---

## Summary

Eww expressions are used in two syntactic positions:

- **`{ expr }`** — standalone attribute value (boolean, number, string result)
- **`${ expr }`** — interpolated inside a string (`"..."` or `` `...` ``)

Key operators: arithmetic (`+`, `-`, `*`, `/`, `%`), comparison (`==`, `!=`, `>`, etc.), boolean (`&&`, `||`, `!`), ternary (`? :`), elvis (`?:`), safe access (`?.`), regex match (`=~`).

Key functions: `round()`, `formattime()`, `jq()`, `formatbytes()`, `replace()`, `strlength()`, `arraylength()`.

Data access: `{var}`, `{obj.field}`, `{arr[0]}`, `{obj["key"]}`, `{EWW_BATTERY.BAT0.capacity}`.

**Reference files in this skill:**
- `OPERATORS.md` — complete operator reference with precedence table
- `FUNCTIONS.md` — all functions grouped by category with signatures and examples
- `DATA_ACCESS.md` — data access patterns, EWW_* magic variables, jq patterns
