---
name: eww-expressions
description: Write and debug eww expressions. Use when writing ${} expressions, using operators (ternary ?:, elvis ?:, safe access ?.), calling functions (round, jq, formattime, formatbytes, strlength, replace), accessing JSON/object/array data, or filtering with regex =~.
---

# FUNCTIONS.md — Complete Function Reference

---

## Math Functions

### `round(number, decimal_digits)`

Rounds a number to the specified number of decimal places.

- `number`: any numeric expression
- `decimal_digits`: integer, number of decimal places (0 = integer result)
- Returns: rounded float

```yuck
{round(3.14159, 0)}   ; 3
{round(3.14159, 2)}   ; 3.14
{round(73.7, 0)}      ; 74

; Most common usage — integer percent for display
{round(EWW_RAM.used_mem_perc, 0)}
{round(EWW_CPU.avg, 0)}
{round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}
```

---

### `floor(number)`

Rounds a number DOWN to the nearest integer (toward negative infinity).

- Returns: integer (as float type)

```yuck
{floor(3.9)}    ; 3
{floor(3.1)}    ; 3
{floor(-1.2)}   ; -2

; Use when you want to truncate without rounding up
{floor(EWW_RAM.used_mem_perc)}
```

---

### `ceil(number)`

Rounds a number UP to the nearest integer (toward positive infinity).

- Returns: integer (as float type)

```yuck
{ceil(3.1)}    ; 4
{ceil(3.9)}    ; 4
{ceil(-1.8)}   ; -1
```

---

### `min(a, b)`

Returns the smaller of two numbers.

```yuck
{min(5, 3)}                ; 3
{min(EWW_CPU.avg, 100)}   ; clamp to 100 max by taking min of... wait, use max for floor
{min(value, max_val)}     ; useful to cap a value
```

---

### `max(a, b)`

Returns the larger of two numbers.

```yuck
{max(5, 3)}               ; 5
{max(value, 0)}           ; clamp to minimum of 0 (no negatives)
{max(EWW_RAM.used_mem_perc, 1)}  ; ensure at least 1 for progress bar
```

---

### `powi(num, n)`

Raises `num` to the power `n`. The exponent `n` must be an integer (i32).

```yuck
{powi(2, 8)}    ; 256
{powi(3, 3)}    ; 27
{powi(10, 3)}   ; 1000
```

---

### `powf(num, n)`

Raises `num` to the power `n`. Both arguments are floats (f64). Use for fractional exponents.

```yuck
{powf(2.0, 0.5)}    ; 1.4142... (square root of 2)
{powf(9.0, 0.5)}    ; 3.0 (square root)
{powf(27.0, 0.333)} ; ~3.0 (cube root)
```

---

### `log(num, n)`

Calculates the base-`n` logarithm of `num`. Both arguments and the return type are f64.

```yuck
{log(100, 10)}   ; 2.0 (log base 10 of 100)
{log(8, 2)}      ; 3.0 (log base 2 of 8)
{log(2.718, 2.718)}  ; ~1.0 (natural log approximation)
```

---

### `sin(number)`, `cos(number)`, `tan(number)`, `cot(number)`

Trigonometric functions. Input is in radians.

```yuck
{sin(0)}             ; 0
{cos(0)}             ; 1
{sin(degtorad(90))}  ; 1.0 (sin of 90 degrees)
{tan(degtorad(45))}  ; ~1.0
```

---

### `degtorad(number)`

Converts degrees to radians. Equivalent to `degrees * π / 180`.

```yuck
{degtorad(180)}   ; 3.14159...
{degtorad(90)}    ; 1.5707...
{degtorad(360)}   ; 6.2831...
```

---

### `radtodeg(number)`

Converts radians to degrees. Equivalent to `radians * 180 / π`.

```yuck
{radtodeg(3.14159)}   ; ~180
{radtodeg(1.5707)}    ; ~90
```

---

## String Functions

### `strlength(value)`

Returns the number of characters in a string.

- Returns: integer

```yuck
{strlength("hello")}          ; 5
{strlength(player_title)}     ; length of current track title
{strlength(workspace.name) > 8 ? substring(workspace.name, 0, 8) : workspace.name}
```

---

### `substring(string, start, length)`

Returns a substring starting at `start` index, with the given `length`.

- `start`: 0-indexed character position
- `length`: number of characters to return
- Returns: string

```yuck
{substring("hello world", 0, 5)}   ; "hello"
{substring("hello world", 6, 5)}   ; "world"

; Truncate long titles for display
{strlength(title) > 20
  ? "${substring(title, 0, 20)}..."
  : title}
```

---

### `replace(string, regex, replacement)`

Replaces all matches of `regex` in `string` with `replacement`.

- Uses Rust regex syntax
- Replaces ALL occurrences (not just first)
- Returns: modified string

```yuck
{replace("foo bar baz", "bar", "qux")}       ; "foo qux baz"
{replace("hello  world", "\\s+", " ")}        ; collapse multiple spaces
{replace(path, "^/home/[^/]+", "~")}          ; abbreviate home path
{replace(title, "[^a-zA-Z0-9 ]", "")}         ; strip special characters
```

---

### `search(string, regex)`

Searches for all non-overlapping matches of `regex` in `string`.

- Returns: JSON array of match strings

```yuck
{search("one two three", "\\w+")}   ; ["one", "two", "three"]
{search(output, "[0-9]+")}           ; find all numbers in string
```

---

### `matches(string, regex)`

Tests whether `string` matches `regex`. Returns a boolean.

- Returns: `true` or `false`

```yuck
{matches("hello", "^h")}              ; true
{matches("Hello", "^[a-z]")}          ; false
{matches(workspace.name, "^special")} ; true for workspaces named "special*"

; Use in ternary
{matches(status, "^(Running|Active)$") ? "green" : "red"}
```

> NOTE: `matches()` takes `(string, regex)` order — string first, regex second.
> The `=~` operator uses reversed order: `regex =~ string`.

---

### `captures(string, regex)`

Returns the capture groups from the first match of `regex` in `string`.

- Returns: JSON array of captured group strings (index 0 = full match, 1+ = groups)

```yuck
{captures("2024-03-15", "(\\d{4})-(\\d{2})-(\\d{2})")}
; ["2024-03-15", "2024", "03", "15"]

{captures("foo=bar", "(\\w+)=(\\w+)")[1]}  ; "foo" (first capture group)
{captures(output, "volume: (\\d+)%")[1]}   ; extract volume number
```

---

## Array and Object Functions

### `arraylength(value)`

Returns the number of elements in a JSON array.

- `value`: a JSON array string (from a variable or expression)
- Returns: integer

```yuck
{arraylength('[1,2,3]')}            ; 3
{arraylength(stringArray)}          ; length of defvar array
{arraylength(workspaces)}           ; number of workspaces

; Check if array is non-empty
{arraylength(items) > 0 ? items[0] : "No items"}
```

---

### `objectlength(value)`

Returns the number of keys in a JSON object.

- `value`: a JSON object string
- Returns: integer

```yuck
{objectlength('{"a":1,"b":2}')}     ; 2
{objectlength(EWW_BATTERY)}         ; number of battery devices
{objectlength(my_object)}           ; keys in a defvar object
```

---

## Utility Functions

### `jq(json_string, filter)`

Runs a jq-style filter on a JSON string. Uses jaq internally (jq-compatible).

- `json_string`: a valid JSON string value
- `filter`: a jq filter expression as a string
- Returns: the filtered value (as JSON or raw depending on flags)

```yuck
; Access a field
{jq(data, ".name")}
{jq(api_response, ".temperature")}

; Access nested field
{jq(data, ".user.profile.avatar")}

; Array index
{jq(data, ".items[0]")}
{jq(data, ".items[-1]")}  ; last element

; Array length
{jq(data, ".items | length")}

; Filter and transform
{jq(data, ".items[] | select(.active == true) | .name")}

; Combine fields
{jq(data, '".name=" + .name + " .age=" + (.age | tostring)')}
```

### `jq(json_string, filter, flags)`

Same as above, but with flags to modify output format.

Supported flags:
- `"r"` — raw output. If the result is a string, it is returned without JSON quotes. Equivalent to jq's `--raw-output`.

```yuck
; Without "r" flag: returns `"hello"` (with quotes)
{jq(data, ".name")}

; With "r" flag: returns `hello` (without quotes)
{jq(data, ".name", "r")}

; Practical: extract a string for display without extra quotes
(label :text {jq(track_data, ".title", "r")})
```

---

### `get_env(variable_name)`

Gets the value of an environment variable.

- `variable_name`: string, the name of the environment variable
- Returns: the value as a string, or empty string if not set

```yuck
{get_env("HOME")}       ; "/home/milo"
{get_env("DISPLAY")}    ; ":0"
{get_env("USER")}       ; "milo"
{get_env("THEME")}      ; custom env var

; Use as fallback
{get_env("MY_ICON") ?: "default-icon"}
```

> NOTE: eww processes do not inherit the full interactive shell environment.
> Environment variables must be set in the process that launches eww, or exported system-wide.
> See eww-yuck skill for notes on environment variable availability.

---

### `formattime(unix_timestamp, format_str)`

Formats a Unix timestamp as a human-readable string using the system's local timezone.

- `unix_timestamp`: integer Unix timestamp (seconds since epoch)
- `format_str`: chrono format string
- Returns: formatted time string

### `formattime(unix_timestamp, format_str, timezone)`

Same as above, but uses the specified timezone instead of system local.

- `timezone`: string in chrono-tz format (e.g., `"America/New_York"`, `"Europe/Berlin"`, `"UTC"`)

```yuck
; Time (24-hour)
{formattime(EWW_TIME, "%H:%M")}          ; 14:30
{formattime(EWW_TIME, "%H:%M:%S")}       ; 14:30:05

; Time (12-hour with AM/PM)
{formattime(EWW_TIME, "%I:%M %p")}       ; 02:30 PM

; Date
{formattime(EWW_TIME, "%Y-%m-%d")}       ; 2024-03-15
{formattime(EWW_TIME, "%B %d, %Y")}      ; March 15, 2024
{formattime(EWW_TIME, "%A")}             ; Friday
{formattime(EWW_TIME, "%a, %b %d")}      ; Fri, Mar 15

; Combined
{formattime(EWW_TIME, "%A, %B %d · %H:%M")}   ; Friday, March 15 · 14:30

; With explicit timezone
{formattime(EWW_TIME, "%H:%M", "America/New_York")}
{formattime(EWW_TIME, "%H:%M", "Europe/Berlin")}
{formattime(EWW_TIME, "%H:%M", "UTC")}
```

### Common chrono format codes

| Code | Meaning | Example |
|---|---|---|
| `%H` | Hour (24h, zero-padded) | `14` |
| `%I` | Hour (12h, zero-padded) | `02` |
| `%M` | Minute | `30` |
| `%S` | Second | `05` |
| `%p` | AM/PM | `PM` |
| `%A` | Full weekday name | `Friday` |
| `%a` | Short weekday name | `Fri` |
| `%B` | Full month name | `March` |
| `%b` | Short month name | `Mar` |
| `%d` | Day of month (zero-padded) | `15` |
| `%Y` | 4-digit year | `2024` |
| `%y` | 2-digit year | `24` |
| `%j` | Day of year | `075` |
| `%Z` | Timezone abbreviation | `EST` |

---

### `formatbytes(bytes, short, format_mode)`

Converts a byte count to a human-readable string.

- `bytes`: i64 byte count (supports negative values)
- `short`: boolean — `true` for compact notation, `false` for full (default: `false`)
- `format_mode`: `"iec"` for binary (KiB, MiB, GiB) or `"si"` for decimal (KB, MB, GB) (default: `"iec"`)
- Returns: formatted string

```yuck
; Default: long IEC (binary, standard for memory)
{formatbytes(EWW_RAM.used_mem, false, "iec")}    ; "7.5 GiB"

; Short IEC (compact)
{formatbytes(EWW_RAM.used_mem, true, "iec")}     ; "7.5G"

; Long SI (decimal, common for network/storage)
{formatbytes(EWW_RAM.used_mem, false, "si")}     ; "8.1 GB"

; Short SI
{formatbytes(EWW_RAM.used_mem, true, "si")}      ; "8.1G"

; Network speed (bytes per second — add "/s" manually)
(label :text "${formatbytes(EWW_NET.wlan0.up, true, "si")}/s")

; RAM with label
(label :text "RAM: ${formatbytes(EWW_RAM.used_mem, false, "iec")} / ${formatbytes(EWW_RAM.total_mem, false, "iec")}")
```

> NOTE: `EWW_RAM.used_mem` is already in bytes. `EWW_NET` values are bytes/sec.
> Some magic variable fields may require multiplication if the unit differs — check the eww-widgets skill.
