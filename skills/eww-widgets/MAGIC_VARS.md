---
name: eww-widgets
description: Choose and configure eww widgets. Use when picking between box/centerbox/overlay/scroll/stack, using button/checkbox/input/scale, displaying data with label/image/progress/circular-progress/graph, using systray or revealer animations, or accessing EWW_RAM/EWW_CPU/EWW_BATTERY/EWW_DISK/EWW_NET magic variables.
---

# Magic Variables (EWW_*)

Complete reference for all built-in eww variables. No `defpoll` or `defvar` needed — they exist automatically.

**Verify live data with:** `eww state | grep EWW_`

**Update rates:**
- `EWW_TIME` — every **1 second**
- All others — every **2 seconds**

---

## EWW_TIME — Unix Timestamp

**Type:** integer (Unix timestamp in seconds)
**Update rate:** 1 second

### Access

```yuck
{EWW_TIME}  ; → 1741234567
```

### Common Patterns

```yuck
; Time display
(label :text {formattime(EWW_TIME, "%H:%M")})         ; 14:32
(label :text {formattime(EWW_TIME, "%H:%M:%S")})      ; 14:32:05
(label :text {formattime(EWW_TIME, "%I:%M %p")})      ; 02:32 PM

; Date display
(label :text {formattime(EWW_TIME, "%A, %B %d")})     ; Thursday, March 06
(label :text {formattime(EWW_TIME, "%Y-%m-%d")})      ; 2026-03-06
(label :text {formattime(EWW_TIME, "%-d %b %Y")})     ; 6 Mar 2026

; Timezone-aware time
(label :text {formattime(EWW_TIME, "%H:%M", "America/Mexico_City")})

; Short day name
(label :text {formattime(EWW_TIME, "%a")})            ; Thu

; Day of month (no leading zero)
(label :text {formattime(EWW_TIME, "%-d")})           ; 6
```

### Gotchas

- `formattime` format strings follow `strftime` conventions.
- `%-d`, `%-m`, `%-H` strip leading zeros (Linux only).
- `EWW_TIME` is an integer. Do not treat it as a string directly.
- For calendar widget month: `{formattime(EWW_TIME, "%-m") - 1}` (GTK calendar is 0-indexed).

---

## EWW_CPU — CPU Usage

**Type:** object
**Update rate:** 2 seconds
**Platform note:** No macOS support

### Structure

```json
{
  "cores": [
    {"core": 0, "freq": 2400, "usage": 12.5},
    {"core": 1, "freq": 2400, "usage": 8.3},
    {"core": 2, "freq": 3200, "usage": 45.1},
    {"core": 3, "freq": 3200, "usage": 22.7}
  ],
  "avg": 22.15
}
```

- `freq` — frequency in MHz
- `usage` — percentage (0.0–100.0)
- `avg` — average usage across all cores

### Access Patterns

```yuck
; Overall average
{round(EWW_CPU.avg, 0)}              ; → "22"
{round(EWW_CPU.avg, 1)}              ; → "22.2"

; Specific core
{round(EWW_CPU.cores[0].usage, 0)}   ; core 0 usage %
{EWW_CPU.cores[0].freq}              ; core 0 frequency MHz

; In a progress bar
(progress :value {EWW_CPU.avg})

; In a graph
(graph :value {EWW_CPU.avg}
       :time-range "60s"
       :min 0 :max 100)

; Iterate all cores (for a per-core display)
(for core in {EWW_CPU.cores}
  (progress :value {core.usage}
            :orientation "v"))
```

### Gotchas

- `avg` is a float. Use `round(EWW_CPU.avg, 0)` for integer display.
- Core count varies by machine — do not hardcode index counts.
- Usage is sampled, not accumulated — it reflects the last 2-second interval.

---

## EWW_RAM — Memory and Swap

**Type:** object
**Update rate:** 2 seconds

### Structure

```json
{
  "total_mem":     16284536,
  "free_mem":       8432120,
  "used_mem":       7852416,
  "available_mem":  9000000,
  "used_mem_perc":  48.2,
  "total_swap":     8000000,
  "free_swap":      7000000,
  "used_swap_perc": 12.5
}
```

- All byte values are in **KB** (not bytes, not MiB)
- `used_mem_perc` and `used_swap_perc` are percentages (0.0–100.0)
- `free_mem` — unused memory
- `available_mem` — memory available for new allocations (includes reclaimable cache)

### Access Patterns

```yuck
; Percentage display
{round(EWW_RAM.used_mem_perc, 0)}     ; → "48"

; Human-readable used memory
; Values are in KB, multiply by 1024 to get bytes for formatbytes
{formatbytes(EWW_RAM.used_mem * 1024)}       ; → "7.5 GiB"
{formatbytes(EWW_RAM.total_mem * 1024)}      ; → "15.5 GiB"

; Swap usage
{round(EWW_RAM.used_swap_perc, 0)}    ; → "12"
{formatbytes(EWW_RAM.free_swap * 1024)}

; Progress bar
(progress :value {EWW_RAM.used_mem_perc})

; Tooltip with detail
(tooltip
  (box :orientation "v"
    (label :text "Used: ${formatbytes(EWW_RAM.used_mem * 1024)}")
    (label :text "Available: ${formatbytes(EWW_RAM.available_mem * 1024)}")
    (label :text "Swap: ${round(EWW_RAM.used_swap_perc, 0)}%"))
  (label :text "󰍛" :class "icon"))
```

### Gotchas

- Values are in **KB**, not bytes. Always multiply by 1024 before passing to `formatbytes`.
- `free_mem` will be very small on a healthy Linux system (Linux uses free RAM for cache). Use `available_mem` to assess actual memory pressure.
- `used_mem_perc` uses `used_mem`, not `available_mem`. This may look higher than htop's "used" number.

---

## EWW_DISK — Disk Usage

**Type:** object keyed by mount point
**Update rate:** 2 seconds

### Structure

```json
{
  "/": {
    "name":     "sda1",
    "total":    500000000000,
    "free":     200000000000,
    "used":     300000000000,
    "used_perc": 60.0
  },
  "/home": {
    "name":     "sda2",
    "total":    1000000000000,
    "free":     600000000000,
    "used":     400000000000,
    "used_perc": 40.0
  }
}
```

- All byte values are in **bytes** (unlike EWW_RAM which uses KB)
- `used_perc` is a percentage (0.0–100.0)

### Access Patterns

```yuck
; Percentage by mount point
{round(EWW_DISK["/"].used_perc, 0)}          ; → "60"
{round(EWW_DISK["/home"].used_perc, 0)}

; Human-readable sizes (values already in bytes)
{formatbytes(EWW_DISK["/"].used)}             ; → "279.4 GiB"
{formatbytes(EWW_DISK["/"].total)}            ; → "465.8 GiB"
{formatbytes(EWW_DISK["/"].free)}             ; → "186.3 GiB"

; Compute free percentage manually (used_perc may round)
{round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}

; Progress bar
(progress :value {EWW_DISK["/"].used_perc})
```

### Gotchas

- Keys are mount point strings with the leading `/`. Use bracket notation: `EWW_DISK["/"]`.
- To discover your mount points: `eww state | grep DISK` or `df -h`.
- Some filesystems (btrfs, zfs) may report inaccurate values due to copy-on-write accounting.
- EWW_DISK is a map — if you access a mount point that does not exist, you get null.

---

## EWW_BATTERY — Battery Status

**Type:** object keyed by battery name
**Update rate:** 2 seconds

### Structure

```json
{
  "BAT0": {
    "status":   "Discharging",
    "capacity": 78
  },
  "total_avg": 78.0
}
```

- `status` — one of: `"Charging"` `"Discharging"` `"Full"` `"Not charging"`
- `capacity` — integer percentage (0–100)
- `total_avg` — average capacity across all batteries

### Access Patterns

```yuck
; Capacity and status
{EWW_BATTERY.BAT0.capacity}              ; → 78
{EWW_BATTERY.BAT0.status}               ; → "Discharging"

; Average across batteries (multi-battery systems)
{round(EWW_BATTERY.total_avg, 0)}

; Conditional icon based on status
(label :text {if EWW_BATTERY.BAT0.status == "Charging" then ""
              else if EWW_BATTERY.BAT0.capacity > 80 then ""
              else if EWW_BATTERY.BAT0.capacity > 50 then ""
              else if EWW_BATTERY.BAT0.capacity > 20 then ""
              else ""}
       :class "bat-icon")

; Progress bar
(progress :value {EWW_BATTERY.BAT0.capacity})

; Dynamic CSS class for low-battery warning
(label :text "${EWW_BATTERY.BAT0.capacity}%"
       :class "bat-label ${if EWW_BATTERY.BAT0.capacity < 20 then "low" else ""}")
```

### Finding Your Battery Name

```bash
ls /sys/class/power_supply/
# or
eww state | grep BATTERY
```

Common names: `BAT0`, `BAT1`, `BATT`, `battery`.

### Gotchas

- Battery name is hardware-dependent. Never hardcode `BAT0` without checking.
- On desktop machines with no battery, `EWW_BATTERY` is an empty object `{}`. Guard with `{if EWW_BATTERY != {} then ...}`.
- `"Not charging"` means the charger is connected but not charging (e.g., battery is full and not reporting `"Full"`). Handle it the same as `"Full"`.

---

## EWW_NET — Network Interface Throughput

**Type:** object keyed by interface name
**Update rate:** 2 seconds

### Structure

```json
{
  "wlan0": {
    "up":   512,
    "down": 4096
  },
  "eth0": {
    "up":   1024,
    "down": 102400
  },
  "lo": {
    "up":   0,
    "down": 0
  }
}
```

- Values are in **bytes per second** (not bits per second, not KB/s)
- Reflects the average rate over the last 2-second update interval

### Access Patterns

```yuck
; Bytes per second
{EWW_NET.wlan0.down}               ; raw bytes/s

; KB/s
{round(EWW_NET.wlan0.down / 1000, 1)}

; MB/s
{round(EWW_NET.wlan0.down / 1000000, 2)}

; Human-readable with formatbytes (per second, not total)
; formatbytes expects bytes, not bytes/s — use it as a formatter only
{formatbytes(EWW_NET.wlan0.down)}   ; → "4.0 KiB/s" (formatbytes adds /s suffix? — check eww state)

; Both upload and download
(box :orientation "h" :spacing 4
  (label :text "↑ ${round(EWW_NET.wlan0.up / 1000, 0)} KB/s")
  (label :text "↓ ${round(EWW_NET.wlan0.down / 1000, 0)} KB/s"))
```

### Finding Your Interface Names

```bash
ip link show
# or
eww state | grep NET
```

Common names: `wlan0`, `wlp3s0`, `eth0`, `enp2s0`, `lo`.

### Gotchas

- Interface names are hardware-dependent (predictable interface names vary by system).
- `lo` (loopback) is included — filter it out if you are iterating all interfaces.
- Values reset to 0 if the interface goes down. Guard with `{if EWW_NET.wlan0 != null then ...}`.
- Values are bytes/second, not cumulative bytes. A zero means no traffic in the last 2 seconds.

---

## EWW_TEMPS — Component Temperatures

**Type:** object keyed by sensor name
**Update rate:** 2 seconds

### Structure

```json
{
  "coretemp Package id 0": 52.0,
  "coretemp Core 0":       48.0,
  "coretemp Core 1":       51.0,
  "acpitz":                35.0,
  "nvme Composite":        38.0
}
```

- Values are in degrees Celsius (float)
- Sensor names vary by hardware

### Access Patterns

```yuck
; CPU package temperature
{round(EWW_TEMPS["coretemp Package id 0"], 0)}

; Using string interpolation
(label :text "${round(EWW_TEMPS["coretemp Package id 0"], 0)}°C")

; Conditional color based on temperature
(label :text "${round(EWW_TEMPS["coretemp Package id 0"], 0)}°C"
       :style "color: ${if EWW_TEMPS["coretemp Package id 0"] > 80 then "#f38ba8" else "#a6e3a1"};")
```

### Finding Your Sensor Names

```bash
eww state | grep TEMPS
# or
sensors
```

### Gotchas

- Sensor names contain spaces — always use bracket notation: `EWW_TEMPS["coretemp Package id 0"]`.
- Names vary significantly between CPU vendors and kernel versions.
- Not all machines expose temperature sensors. If `EWW_TEMPS` is empty, your hardware may not have supported sensors.

---

## EWW_CONFIG_DIR — Config Directory Path

**Type:** string (static, never changes at runtime)

### Structure

```
"/home/milo/.config/eww"
```

### Access Patterns

```yuck
; Reference a file in the config
(image :path "${EWW_CONFIG_DIR}/icons/avatar.png")
(image :path "${EWW_CONFIG_DIR}/icons/battery.svg")

; In a script argument
(button :onclick "scripts/do-thing.sh ${EWW_CONFIG_DIR}")
```

> Always use `EWW_CONFIG_DIR` instead of hardcoded absolute paths. This makes the config portable and multi-user safe.

---

## EWW_CMD — eww Command for This Config

**Type:** string (static)

The eww command invocation that targets the current config instance. Resolves to something like `eww -c /home/milo/.config/eww`.

### Access Patterns

```yuck
; Open a popup window
(button :onclick "${EWW_CMD} open calendar-popup")

; Close current window
(button :onclick "${EWW_CMD} close my-window")

; Update a variable
(button :onclick "${EWW_CMD} update show-panel=true")
```

> Always prefer `EWW_CMD` over hardcoded `eww` in onclick handlers. If the binary is not on PATH at event time, commands will silently fail. `EWW_CMD` resolves to the absolute path used to launch eww.

---

## EWW_EXECUTABLE — Full Path to eww Binary

**Type:** string (static)

The absolute filesystem path to the eww binary.

### Access Patterns

```yuck
; When you need just the binary path (not config-specific)
(button :onclick "${EWW_EXECUTABLE} --version")
```

> For most window/variable operations, use `EWW_CMD` (which includes the config path). Use `EWW_EXECUTABLE` only when you need the raw binary path.

---

## Quick Lookup: Field Names by Variable

| Variable | Key fields |
|---|---|
| `EWW_CPU` | `.avg` `.cores[N].usage` `.cores[N].freq` `.cores[N].core` |
| `EWW_RAM` | `.used_mem_perc` `.used_mem` `.free_mem` `.available_mem` `.total_mem` `.used_swap_perc` `.free_swap` `.total_swap` |
| `EWW_DISK` | `["/"].used_perc` `["/"].used` `["/"].free` `["/"].total` `["/"].name` |
| `EWW_BATTERY` | `.BAT0.capacity` `.BAT0.status` `.total_avg` |
| `EWW_NET` | `.wlan0.up` `.wlan0.down` |
| `EWW_TEMPS` | `["sensor name"]` |
| `EWW_TIME` | direct (int) |
| `EWW_CONFIG_DIR` | direct (string) |
| `EWW_CMD` | direct (string) |
| `EWW_EXECUTABLE` | direct (string) |

---

## Integration with Other Skills

- Expression functions used with magic vars (`round`, `formattime`, `formatbytes`, `jq`): `eww-expressions` skill
- Widget properties that consume these values (`:value`, `:text`): `DISPLAY.md`, `CONTAINERS.md`
- Debugging unexpected variable values: `eww-troubleshooting` skill (`eww state`, `eww logs`)
