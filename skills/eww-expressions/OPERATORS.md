---
name: eww-expressions
description: Write and debug eww expressions. Use when writing ${} expressions, using operators (ternary ?:, elvis ?:, safe access ?.), calling functions (round, jq, formattime, formatbytes, strlength, replace), accessing JSON/object/array data, or filtering with regex =~.
---

# OPERATORS.md — Complete Operator Reference

---

## Arithmetic Operators

| Operator | Name | Example | Result |
|---|---|---|---|
| `+` | Addition | `{3 + 2}` | `5` |
| `-` | Subtraction | `{10 - 4}` | `6` |
| `*` | Multiplication | `{5 * 3}` | `15` |
| `/` | Division | `{10 / 4}` | `2.5` |
| `%` | Modulo | `{10 % 3}` | `1` |

### Type behavior notes

- Division of two integers produces a float: `{10 / 3}` → `3.3333...`
- Use `round()`, `floor()`, or `ceil()` to convert to integer for display.
- Modulo works on integers. Using it with floats may produce unexpected results.
- String concatenation is done with `+`: `{"foo" + "bar"}` → `"foobar"`

```yuck
; Disk percent computed from bytes
{round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}

; Megabytes per second from bytes per second
{round(EWW_NET.wlan0.up / 1000000, 2)}

; Combine arithmetic and round for clean integer display
{round(EWW_RAM.used_mem_perc, 0)}
```

---

## Comparison Operators

| Operator | Name | Example | Result |
|---|---|---|---|
| `==` | Equal | `{a == b}` | `true` / `false` |
| `!=` | Not equal | `{a != b}` | `true` / `false` |
| `>` | Greater than | `{a > b}` | `true` / `false` |
| `<` | Less than | `{a < b}` | `true` / `false` |
| `>=` | Greater or equal | `{a >= b}` | `true` / `false` |
| `<=` | Less or equal | `{a <= b}` | `true` / `false` |

```yuck
; Compare numbers
{EWW_BATTERY.BAT0.capacity < 20}   ; true when battery is low
{EWW_CPU.avg > 80}                  ; true when CPU is under load
{count == 0}                        ; true when count is zero

; Compare strings (lexicographic)
{formattime(EWW_TIME, "%H") < "12"}  ; true before noon (string comparison)
{status == "Charging"}               ; exact string match
```

---

## Boolean Operators

| Operator | Name | Example |
|---|---|---|
| `&&` | Logical AND | `{a && b}` |
| `\|\|` | Logical OR | `{a \|\| b}` |
| `!` | Logical NOT | `{!a}` |

```yuck
; AND: both conditions must be true
{EWW_BATTERY.BAT0.capacity < 20 && EWW_BATTERY.BAT0.status == "Discharging"}

; OR: either condition is true
{cpu_high || memory_high}

; NOT: invert a boolean variable
{!paused}
{!muted}

; Combined
{!(status == "Charging") && EWW_BATTERY.BAT0.capacity < 30}
```

---

## Ternary Operator

Syntax: `condition ? value_if_true : value_if_false`

```yuck
; Basic usage
{online ? "Connected" : "Offline"}
{muted ? "󰖁" : "󰕾"}

; Ternary in attribute (no quotes — expression context)
:active {count > 0}
:class {active ? "active" : "inactive"}

; Ternary in string interpolation
(label :text "Status: ${online ? "online" : "offline"}")

; Multi-level nested ternary (chain the false branch)
{battery < 10  ? "󰁺" :
 battery < 25  ? "󰁼" :
 battery < 50  ? "󰁾" :
 battery < 75  ? "󰂀" :
                 "󰁹"}

; Ternary for dynamic CSS class in backtick string
:class `battery ${EWW_BATTERY.BAT0.capacity < 20 ? "critical" :
                  EWW_BATTERY.BAT0.capacity < 50 ? "low" : "ok"}`

; Ternary to conditionally show content
{music != "" ? "🎵 ${music}" : ""}
```

### Nesting patterns

For many branches, chain the false arm (right of `:`) rather than nesting inside the true arm.
This keeps expressions readable:

```yuck
; ✅ CORRECT: chain the false branch
{a < 10 ? "low" :
 a < 50 ? "mid" :
 a < 90 ? "high" :
          "max"}

; ❌ HARDER TO READ: nesting true branch
{a < 10 ? "low" : {a < 50 ? "mid" : {a < 90 ? "high" : "max"}}}
```

---

## Elvis Operator `?:`

Syntax: `value ?: default`

Returns `default` if `value` is `""` (empty string) or JSON `null`.
Otherwise returns `value` unchanged.

```yuck
; Variable that might be empty string
{player_title ?: "Nothing playing"}

; Network interface that may not exist
{EWW_NET.eth0.up ?: 0}

; Custom poll that returns "" on failure
{weather_temp ?: "N/A"}

; JSON field that may be null
{user.nickname ?: user.username}
```

### Elvis vs Ternary

| Situation | Use |
|---|---|
| Handle empty string or JSON null | `value ?: default` (elvis) |
| Handle any boolean condition | `cond ? a : b` (ternary) |
| Handle 0 (zero) as a valid value | Use ternary: `val == "" ? "N/A" : val` |

> NOTE: Elvis does NOT treat `0`, `false`, or `[]` as falsy. Only `""` and JSON `null` trigger the default. If `0` is a valid meaningful value, elvis will return `0` correctly.

---

## Safe Access Operator `?.` and `?.[index]`

Syntax: `value?.field` or `value?.[index]`

If the left side is empty string or JSON `null`, returns `null` instead of erroring.
Otherwise, performs the normal field/index access.

```yuck
; Safe field access — protects when data might not be available yet
{sensor_data?.temperature}
{api_response?.items}

; Safe array access
{items?.[0]}
{json_array?.[index]}

; Chained safe access
{obj?.nested?.deep}
{response?.data?.users}

; Best pattern: combine with elvis for a safe fallback value
{sensor_data?.temperature ?: "--"}
{api_response?.items?.[0]?.name ?: "Unknown"}
```

### When to use safe access

- When data comes from a `defpoll` or `deflisten` that might not have fired yet
- When accessing JSON from an external script that may return `null` for some fields
- When an object may not always have a particular key

### When safe access is NOT enough

```yuck
; ❌ Still errors: the left side exists but is a number, not an object
; If `my_count` = "42" (a plain string/number):
{my_count?.field}   ; error: cannot index a number

; Safe access only guards null/empty, not type mismatches
; ✅ Use elvis instead when the value might be wrong type:
{my_count ?: "0"}
```

---

## Regex Match Operator `=~`

Syntax: `regex_pattern =~ string_to_test`

Returns `true` if the string matches the regex pattern, `false` otherwise.
Uses Rust regex syntax (similar to PCRE but without lookaheads).

> IMPORTANT: The LEFT side is the regex, the RIGHT side is the string. This is the OPPOSITE of bash (`[[ string =~ pattern ]]`) and Perl/Ruby.

```yuck
; Test workspace name against a pattern
{workspace.name =~ "^special:.+$"}

; Test if a string starts with uppercase
{"Hello" =~ "^[A-Z]"}

; Test for a specific suffix
{filename =~ "\\.log$"}

; Used in ternary
{workspace.name =~ "^special" ? "special-class" : "normal-class"}

; Case-insensitive with (?i) flag (Rust regex inline flag)
{title =~ "(?i)^music"}
```

### Rust regex vs other languages

| Feature | Rust regex | bash `[[ =~ ]]` | Perl/Ruby |
|---|---|---|---|
| Operand order | `pattern =~ string` | `string =~ pattern` | `string =~ /pattern/` |
| Lookaheads | Not supported | Not supported | Supported |
| Named captures | `(?P<name>...)` | Not supported | `(?<name>...)` |
| Unicode | Full support | Partial | Full support |

---

## Operator Precedence

From highest (evaluated first) to lowest:

| Precedence | Operators | Notes |
|---|---|---|
| 1 (highest) | `!`, unary `-` | Logical NOT, negation |
| 2 | `*`, `/`, `%` | Multiplicative |
| 3 | `+`, `-` | Additive |
| 4 | `>`, `<`, `>=`, `<=` | Relational comparison |
| 5 | `==`, `!=` | Equality comparison |
| 6 | `=~` | Regex match |
| 7 | `&&` | Logical AND |
| 8 | `\|\|` | Logical OR |
| 9 | `?:` | Elvis (null coalescing) |
| 10 | `?.`, `?.[n]` | Safe access |
| 11 (lowest) | `? :` | Ternary conditional |

Use parentheses to override precedence or to make complex expressions readable.

---

## Complex Examples Combining Multiple Operators

### Network status with fallback

```yuck
; Show speed or "offline" if interface data is missing
{EWW_NET?.wlan0?.up ?: 0 > 0
  ? "Up: ${round(EWW_NET.wlan0.up / 1000000, 2)} MB/s"
  : "Offline"}
```

### Battery with charging indicator

```yuck
{EWW_BATTERY.BAT0.status == "Charging"
  ? "⚡ ${EWW_BATTERY.BAT0.capacity}%"
  : EWW_BATTERY.BAT0.capacity < 20
    ? "! ${EWW_BATTERY.BAT0.capacity}%"
    : "${EWW_BATTERY.BAT0.capacity}%"}
```

### Temperature with alert threshold

```yuck
; CSS class based on temperature range and availability
:class `temp ${EWW_TEMPS["coretemp Package id 0"] ?: 0 > 90
              ? "critical"
              : EWW_TEMPS["coretemp Package id 0"] ?: 0 > 70
                ? "warm"
                : "normal"}`
```

### CPU usage with regex-based workspace class

```yuck
; Combine CPU check and workspace name pattern
{EWW_CPU.avg > 80 && workspace.name =~ "^[0-9]+$" ? "busy" : "idle"}
```
