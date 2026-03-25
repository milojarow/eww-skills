---
name: eww-expressions
description: Write and debug eww expressions. Use when writing ${} expressions, using operators (ternary ?:, elvis ?:, safe access ?.), calling functions (round, jq, formattime, formatbytes, strlength, replace), accessing JSON/object/array data, or filtering with regex =~.
---

# DATA_ACCESS.md — Data Access Patterns

---

## Simple Variable Access

Variables declared with `defvar`, `defpoll`, or `deflisten` are accessed by name.

```yuck
; In attribute (bare expression context)
:value {my_var}
:active {is_active}

; In string interpolation
(label :text "Value: ${my_var}")
(label :text "${greeting}, ${name}!")
```

---

## Object Field Access

When a variable holds a JSON object string, access its fields with dot notation or bracket notation.

### Dot notation

```yuck
{EWW_RAM.used_mem_perc}
{weather.temperature}
{status.current}
{time_data.hour}
```

### Bracket notation

Required when field keys contain spaces, special characters, slashes, or are stored in a variable.

```yuck
; Key with spaces
{EWW_TEMPS["coretemp Package id 0"]}

; Key with slashes (disk mount points)
{EWW_DISK["/"].used_perc}
{EWW_DISK["/home"].free}

; Dynamic key from a variable
(defvar selected `🦝`)
{object[selected]}   ; looks up object["🦝"]

; Network interface by name
{EWW_NET["wlan0"].up}
```

---

## Array Index Access

When a variable holds a JSON array string, access elements by 0-based integer index.

```yuck
{items[0]}              ; first element
{items[1]}              ; second element
{EWW_CPU.cores[0].usage}  ; first core's usage field
{workspaces[0].name}    ; name of first workspace
```

---

## Nested (Chained) Access

Combine object and array access in any order.

```yuck
; Object → Object
{EWW_BATTERY.BAT0.capacity}

; Object → Array → Object
{EWW_CPU.cores[0].usage}
{data.users[0].name}
{response.items[2].title}

; Object → Bracket → Object (spaces in intermediate key)
{EWW_DISK["/home"].used_perc}

; Variable key at any level
{servers[active_server].status}
```

---

## EWW_* Magic Variables

These are provided automatically by eww and updated in real time. No `defpoll` needed.

### EWW_TIME

Unix timestamp (integer), updated every second.

```yuck
{EWW_TIME}
{formattime(EWW_TIME, "%H:%M")}
{formattime(EWW_TIME, "%A, %B %d")}
{formattime(EWW_TIME, "%H:%M", "UTC")}
```

### EWW_RAM

```yuck
{EWW_RAM.used_mem_perc}     ; float, percent used (0-100)
{EWW_RAM.used_mem}          ; bytes used
{EWW_RAM.total_mem}         ; total bytes
{EWW_RAM.free_mem}          ; bytes free
{EWW_RAM.available_mem}     ; bytes available (free + reclaimable)

; Display examples
(label :text "RAM: ${round(EWW_RAM.used_mem_perc, 0)}%")
(label :text "${formatbytes(EWW_RAM.used_mem, false, "iec")} / ${formatbytes(EWW_RAM.total_mem, false, "iec")}")
```

### EWW_CPU

```yuck
{EWW_CPU.avg}               ; float, average usage across all cores (0-100)
{EWW_CPU.cores[0].usage}    ; float, per-core usage (0-100)
{EWW_CPU.cores[1].usage}
; ... index matches CPU core number

; Display examples
(label :text "CPU: ${round(EWW_CPU.avg, 0)}%")
(scale :value {EWW_CPU.avg} :min 0 :max 100)
```

### EWW_DISK

Keyed by mount point. Use bracket notation because mount points contain slashes.

```yuck
{EWW_DISK["/"].used_perc}     ; float, percent used (0-100)
{EWW_DISK["/"].free}          ; bytes free
{EWW_DISK["/"].total}         ; total bytes
{EWW_DISK["/home"].used_perc}
{EWW_DISK["/home"].free}

; Compute percent manually from bytes
{round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}

; Display
(label :text "Disk: ${round(EWW_DISK["/"].used_perc, 0)}%")
```

### EWW_BATTERY

Keyed by battery device name (e.g., `BAT0`, `BAT1`).

```yuck
{EWW_BATTERY.BAT0.capacity}    ; integer, percent (0-100)
{EWW_BATTERY.BAT0.status}      ; string: "Charging", "Discharging", "Full", "Unknown"

; Multiple batteries
{EWW_BATTERY.BAT1.capacity}

; Display
(label :text "${EWW_BATTERY.BAT0.capacity}%")
:class `battery ${EWW_BATTERY.BAT0.capacity < 20 ? "critical" : "ok"}`
```

### EWW_NET

Keyed by network interface name. Use dot notation if the name has no special characters,
or bracket notation to be safe.

```yuck
{EWW_NET.wlan0.up}       ; bytes uploaded per second
{EWW_NET.wlan0.down}     ; bytes downloaded per second
{EWW_NET.eth0.up}
{EWW_NET.eth0.down}

; Bracket notation (equivalent, required for names with hyphens etc.)
{EWW_NET["wlan0"].up}
{EWW_NET["enp2s0"].down}

; Display in MB/s
(label :text "Up: ${round(EWW_NET.wlan0.up / 1000000, 2)} MB/s")
(label :text "Down: ${formatbytes(EWW_NET.wlan0.down, true, "si")}/s")
```

### EWW_TEMPS

Keyed by thermal zone label. Keys typically contain spaces, so bracket notation is required.

```yuck
{EWW_TEMPS["coretemp Package id 0"]}   ; CPU package temperature (°C, float)
{EWW_TEMPS["acpitz"]}                  ; ACPI thermal zone

; Display
(label :text "CPU: ${round(EWW_TEMPS["coretemp Package id 0"], 0)}°C")
```

> NOTE: The exact key names depend on your hardware. Run `eww state` to see the
> actual EWW_TEMPS keys on your system.

---

## Defining Variables with JSON Data

For custom JSON data, define variables that produce JSON strings.

### `defvar` — static JSON

```yuck
; JSON array
(defvar stringArray `["apple", "banana", "cherry"]`)

; JSON object
(defvar config `{"theme": "dark", "font_size": 14}`)

; Array of objects (from data-structures.yuck example)
(defvar object `{
  "🦝": "racoon",
  "🐱": "cat",
  "🦊": "fox"
}`)
```

### `defpoll` — JSON from a script

```yuck
; Script outputs JSON object
(defpoll time_data :interval "10s"
  `date +'{"hour":"%H","min":"%M","day":"%A"}'`)

; Access fields
{time_data.hour}:{time_data.min}
{time_data.day}

; Script outputs JSON array
(defpoll workspaces :interval "500ms"
  "swaymsg -t get_workspaces")

{workspaces[0].name}
{arraylength(workspaces)}
```

### `deflisten` — JSON from long-running process

```yuck
(deflisten active_window :initial `{"title":"","app_id":""}`
  "swaymsg -t subscribe -m '[ \"window\" ]' | jq -c '.container'")

{active_window.title}
{active_window.app_id}
```

---

## Common Mistake: Bracket vs Dot Notation

```yuck
; ❌ WRONG: dot notation for keys with special characters
{EWW_DISK[/].used_perc}      ; syntax error — slash is not valid in dot path
{EWW_TEMPS.coretemp Package id 0}  ; syntax error — spaces break parsing

; ✅ CORRECT: bracket notation for special keys
{EWW_DISK["/"].used_perc}
{EWW_TEMPS["coretemp Package id 0"]}

; ❌ WRONG: bracket notation with unquoted string
{EWW_DISK[/].used_perc}   ; / is interpreted as division, not a string

; ✅ CORRECT: always quote the key in bracket notation
{EWW_DISK["/"].used_perc}
```

---

## jq Patterns for Complex JSON

Use `jq()` when direct field access is not sufficient — filtering, transforming, or extracting from deeply nested or dynamic structures.

```yuck
; Extract a single field
{jq(json_var, ".name")}
{jq(json_var, ".user.email", "r")}   ; raw (no JSON quotes around string)

; Array operations
{jq(json_var, ".items | length")}
{jq(json_var, ".items[0].title", "r")}
{jq(json_var, ".items[-1].id")}       ; last element

; Filtering arrays
{jq(json_var, "[.items[] | select(.active)] | length")}

; Transform and extract
{jq(json_var, ".scores | add / length")}   ; average of a numbers array
{jq(json_var, ".tags | join(\", \")", "r")} ; join array to comma-separated string

; Conditional in jq
{jq(json_var, "if .status == \"ok\" then .value else 0 end")}

; Multiple fields (returns JSON object string)
{jq(json_var, "{name: .user.name, age: .user.age}")}
```

### jq vs direct access

| Situation | Prefer |
|---|---|
| Simple field access | Direct: `{obj.field}` |
| Array index | Direct: `{arr[0]}` |
| Filter array by condition | jq: `{jq(arr, "[.[] | select(.active)]")}` |
| Compute from array (sum, avg) | jq: `{jq(arr, "add")}` |
| Transform data structure | jq: `{jq(data, "{name, score}")}` |
| Strip quotes from string result | jq with `"r"` flag |
