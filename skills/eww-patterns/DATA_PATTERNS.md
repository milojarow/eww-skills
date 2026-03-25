# DATA_PATTERNS.md — Connecting Data to eww

Patterns for how data flows from the system into eww widgets. Covers `defpoll` JSON, `deflisten` streams, shell scripts, JSON arrays, and dynamic class state. Examples are drawn from real community configs with full attribution.

---

## defpoll with JSON Output

`defpoll` runs a shell command on an interval and stores the stdout as the variable value. When the output is a JSON string, eww can access fields with dot notation.

```yuck
; Command outputs JSON — eww parses it automatically
(defpoll time :interval "5s"
  :initial `date +'{"hour":"%H","min":"%M","day":"%A"}'`
  `date +'{"hour":"%H","min":"%M","day":"%A"}'`)

; Access fields with dot notation in expressions
(label :text "${time.hour}:${time.min}")
(label :text "Today is ${time.day}")
```

**The `:initial` field** provides a value before the first poll completes so the widget renders immediately without showing empty/broken state.

```yuck
; Without :initial — widget shows blank until first poll fires
(defpoll vol :interval "1s" "scripts/getvol")

; With :initial — widget shows "50" immediately
(defpoll vol :interval "1s"
  :initial "50"
  "scripts/getvol")
```

**Accessing nested JSON** (EWW_BATTERY is a built-in magic variable):
```yuck
; EWW_BATTERY shape: {"BAT0": {"status": "Discharging", "capacity": 87}, "total_avg": 87.0}
(label :text "${EWW_BATTERY.BAT0.capacity}%")
(label :text {EWW_BATTERY.BAT0.status})

; EWW_DISK: {"/" : {"free": ..., "total": ..., "used": ..., "used_perc": ...}}
(label :text "${round(EWW_DISK["/"].used_perc, 0)}%")
```

**Bracket notation** (`["key"]`) is needed when the key contains special characters or is a variable. Dot notation (`var.key`) works for simple string keys.

---

## deflisten for Real-Time Streams

`deflisten` keeps a process open and reads each line of stdout as a new value update. Use it for anything event-driven — music changes, workspace switches, volume adjustments.

The key rule: **the script must never exit** (as long as updates are possible). A deflisten whose script exits stops receiving updates.

### Music via playerctl

```yuck
; playerctl --follow stays open and emits one line per track change
(deflisten music :initial ""
  "playerctl --follow metadata --format '{{ artist }} - {{ title }}' || true")

(deflisten song-status :initial "Stopped"
  "playerctl --follow status || echo Stopped")

(defwidget music []
  (box :class "music" :space-evenly false
    {music != "" ? "Now playing: ${music}" : ""}))
```

The `|| true` prevents the deflisten from dying if playerctl exits (no player running).

### deflisten with :onchange side effect

`deflisten` supports an `:onchange` hook that runs a shell command whenever the value updates, with `{}` replaced by the new value. Used by owenrumney for auto-reveal animations:

```yuck
; Source: owenrumney — github.com/owenrumney/dotfiles
(deflisten music
  :initial ""
  :onchange "eww update revealSpotify=true && sleep 5 && eww update revealSpotify=false"
  "playerctl --follow metadata --format '{{ artist }} - {{ title }}' || true")

(deflisten notifications
  :initial ""
  :onchange "eww update revealNotify=true && sleep 5 && eww update revealNotify=false"
  "notification-listener")
```

---

## Shell Script Patterns

Scripts should output one value per line with no trailing newlines or extra output.

### Script requirements summary

| Requirement | Why |
|---|---|
| One value per stdout line | eww reads lines; extra newlines create empty updates |
| Flush stdout immediately | Without flushing, eww may not see output until buffer fills |
| No interactive prompts | Scripts run non-interactively; any prompt hangs the script |
| Exit 0 on success | Non-zero exit kills a `deflisten` process |
| `--line-buffered` for grep pipes | Prevents grep from buffering and blocking eww |
| `flush=True` in Python | Python buffers stdout by default; required for deflisten |

### The flush=True requirement in Python deflisten scripts

Python buffers stdout when writing to a pipe. In a `deflisten`, the script's stdout IS a pipe — eww reads from it. Without `flush=True`, Python accumulates output in an internal 4KB buffer and eww sees nothing until that buffer fills or the process exits. Since a deflisten script is meant to run forever, the buffer never fills, and eww never gets any updates.

The fix is a single keyword argument on every `print` call in the event loop:

```python
# Bad — eww sees nothing until 4KB of JSON accumulates in the buffer
print(json.dumps(generate_workspace_data()))

# Correct — each print call flushes immediately to eww's pipe
print(json.dumps(generate_workspace_data()), flush=True)
```

This is the most common source of "my workspace/volume script does nothing" bugs in Python-based eww integrations. Both druskus20 and dots-of-war use `flush=True` in their workspace scripts.

---

## JSON Array for Dynamic Lists

Use `defvar` with a JSON array and the `for` keyword to render dynamic lists — workspace buttons, notification cards, emoji pickers.

```yuck
; Static array as defvar — useful for reference/picker widgets
(defvar emoji-list `["raccoon","cat","ape"]`)
(defvar selected "raccoon")

(defwidget emoji-picker []
  (box :class "picker" :orientation "h"
    (for item in emoji-list
      (eventbox :class `btn ${selected == item ? "selected" : ""}`
                :cursor "pointer"
                :onhover "eww update selected=${item}"
        item))))
```

**Dynamic workspace list** (from deflisten):
```yuck
(deflisten workspaces :initial "[]" "scripts/swayspaces.py")

(defwidget ws-buttons []
  (box :orientation "h" :spacing 4
    (for ws in workspaces
      (button :class `ws-btn ${ws.focused ? "focused" : ""} ${ws.urgent ? "urgent" : ""}`
              :onclick "swaymsg workspace ${ws.name}"
        {ws.name}))))
```

---

## Dynamic CSS Class State

The pattern for conditional styling: compute the class string in `:class`, style the classes in SCSS. Avoids the need for `:visible false` which causes layout collapse.

```yuck
; Pattern: base class always present, state classes appended conditionally
(label :class `battery ${EWW_BATTERY.BAT0.capacity < 10  ? "empty" :
                         EWW_BATTERY.BAT0.capacity < 25  ? "critical" :
                         EWW_BATTERY.BAT0.capacity < 50  ? "low" :
                                                            "ok"}`
       :text battery-icon)

; Network strength class
(label :class `network ${signal == "" ? "offline" :
                          signal < 26  ? "weak" :
                          signal < 51  ? "fair" :
                          signal < 76  ? "good" : "excellent"}`
       :text network-icon)
```

```scss
/* Battery states */
.battery.empty    { color: #f38ba8; animation: blink 1s infinite; }
.battery.critical { color: #fab387; }
.battery.low      { color: #f9e2af; }
.battery.ok       { color: #a6e3a1; }

/* Network states */
.network.offline  { color: #6c7086; }
.network.weak     { color: #f38ba8; }
.network.fair     { color: #f9e2af; }
.network.good     { color: #a6e3a1; }
.network.excellent{ color: #a6e3a1; }
```

---

## Key Patterns Summary

| Pattern | Use case | Example |
|---|---|---|
| `defpoll` + plain integer | Simple metrics that don't need realtime | `getvol`, `getram`, `wifi icon` |
| `defpoll` + JSON string | Multiple related fields at once | `battery icon + percent` as JSON |
| `deflisten` + event-driven stream | Realtime state — workspaces, volume, music | `listen-volume.sh`, `swayspaces.py` |
| `EWW_*` magic variables | CPU, RAM, disk, battery — no script needed | `EWW_RAM`, `EWW_BATTERY`, `EWW_DISK` |
| `defvar` + `eww update` | State changed only by user action | `audio_sink`, `volum`, `power` |
| `(literal :content ...)` | Script emits raw yuck markup | rxyhn's workspace script |
| Lockfile toggle | Popup open/close toggle from a button | rxyhn's `popup` script |
| Argument dispatch (`$1`) | One script, multiple modes | rxyhn's `battery`, `wifi`, `popup` |

---

## Real Script Gallery

Complete real scripts from community configs, attributed and annotated. Each subsection shows the script, the `defpoll`/`deflisten` line that invokes it, and the widget that consumes it.

---

### Workspace Listeners

Two approaches to the same problem: subscribe to Sway workspace events and emit the current state as JSON for eww to render.

---

**Approach A — druskus20: per-monitor dict, pre-computed CSS class and icon**

druskus20's script (which itself credits Elkowar as original author in its header comment) extends the base pattern by pre-computing `class` and `icon` fields in Python, so the yuck side just reads them directly.

```python
# Source: druskus20 — github.com/druskus20/simpler-bar
# (Original author credited in file: Elkowar — github.com/elkowar/dots-of-war)
#!/usr/bin/env python3

import subprocess
import json


def get_workspaces():
    output = subprocess.check_output(["swaymsg", "-t", "get_workspaces"])
    return json.loads(output.decode("utf-8"))


def generate_workspace_data() -> dict:
    data = {}
    for wsp in get_workspaces():
        if wsp["output"] not in data:
            data[wsp["output"]] = []
        i = { "name": wsp["name"],
              "monitor": wsp["output"],
              "focused": wsp["focused"],
              "visible": wsp["visible"],
            }
        if wsp["focused"]:
            i["class"] = "focused"
            i["icon"] = ""
        elif wsp["visible"]:
            i["class"] = "visible"
            i["icon"] = ""
        else:
            i["class"] = "hidden"
            i["icon"] = ""
        data[wsp["output"]].append(i)

    return data


if __name__ == "__main__":
    process = subprocess.Popen(
        ["swaymsg", "-t", "subscribe", "-m", '["workspace"]', "--raw"],
        stdout=subprocess.PIPE,
    )
    if process.stdout is None:
        print("Error: could not subscribe to sway events")
        exit(1)
    while True:
        print(json.dumps(generate_workspace_data()), flush=True)  # flush=True is mandatory
        line = process.stdout.readline().decode("utf-8")
        if line == "":
            break
```

How druskus20 uses it in `eww.yuck` — the dict is keyed by monitor name, and the widget receives the monitor as a parameter:

```yuck
; Source: druskus20 — github.com/druskus20/simpler-bar
(deflisten workspaces :initial '{"DP-2": [], "HDMI-A-1": [], "eDP-1": []}' "./modules/sway-workspaces.py")

(defwidget workspaces [monitor]
  (box :orientation "h" :class "workspaces"
    (for wsp in {workspaces[monitor]}
      (button :class "workspace ${wsp.class}"
              :onclick "swaymsg workspace ${wsp.name}"
        (box
          (label :class "icon" :text "${wsp.icon}")
          (label :class "name" :text "${wsp.name}"))))))
```

The `:initial` value must mirror the dict shape so the widget doesn't break before the first emit. The windows are created per-monitor:

```yuck
(defwindow win0 :monitor 0
               :geometry (geometry :x 0 :y 0 :height "15px" :width "100%" :anchor "center top")
               :stacking "fg" :exclusive true :focusable false
  (bar0 :monitor "eDP-1"))
```

---

**Approach B — dots-of-war (elkowar): same structure, no pre-computed class**

The original elkowar script omits `class` and `icon` from the output — it emits only the raw sway data. Class computation is done inline in yuck instead.

```python
# Source: elkowar — github.com/elkowar/dots-of-war
#!/usr/bin/env python3

import subprocess
import json


def get_workspaces():
    output = subprocess.check_output(["swaymsg", "-t", "get_workspaces"])
    return json.loads(output.decode("utf-8"))


def generate_workspace_data() -> dict:
    data = {}
    for wsp in get_workspaces():
        if wsp["output"] not in data:
            data[wsp["output"]] = []
        data[wsp["output"]].append(
            {
                "name": wsp["name"],
                "monitor": wsp["output"],
                "focused": wsp["focused"],
                "visible": wsp["visible"],
            }
        )
    return data


if __name__ == "__main__":
    process = subprocess.Popen(
        ["swaymsg", "-t", "subscribe", "-m", '["workspace"]', "--raw"],
        stdout=subprocess.PIPE,
    )
    if process.stdout is None:
        print("Error: could not subscribe to sway events")
        exit(1)
    while True:
        print(json.dumps(generate_workspace_data()), flush=True)  # flush=True is mandatory
        line = process.stdout.readline().decode("utf-8")
        if line == "":
            break
```

The dots-of-war repo also includes a commented-out alternative implementation that pre-declares a fixed workspace count per monitor (useful when you want workspace buttons to always appear, even for unoccupied workspaces):

```python
# Source: elkowar — github.com/elkowar/dots-of-war (commented-out variant in swayspaces.py)
# Approach: fixed WSP_COUNT per monitor, marks each as occupied/unoccupied
WSP_COUNT = 5
MONITOR_COUNT = 2

def generate_workspace_data_for_monitor(monitor: int) -> list[dict]:
    workspaces = {w["name"]: w for w in get_workspaces()}
    data = []
    for i in range(WSP_COUNT):
        name = f"{monitor+1}{i+1}"
        wsp_data = workspaces.get(name)
        entry = {
            "name": name,
            "monitor": monitor,
            "occupied": False,
            "focused": False,
            "visible": False,
        }
        if wsp_data is not None:
            entry["focused"] = wsp_data["focused"]
            entry["visible"] = wsp_data["visible"]
            entry["occupied"] = True
        data.append(entry)
    return data

def generate_workspace_data() -> dict:
    return {i: generate_workspace_data_for_monitor(i) for i in range(MONITOR_COUNT)}
```

This variant adds an `occupied` field so the yuck side can style unoccupied workspaces differently (dim, hide, etc.) without them disappearing from the list.

Used in `eww.yuck` — class is computed inline in yuck rather than pre-built in Python:

```yuck
; Source: elkowar — github.com/elkowar/dots-of-war
(deflisten workspaces :initial '{"DP-2": [], "HDMI-A-1": []}' "./swayspaces.py")

(defwidget workspaces [screen]
  (box :orientation "v" :class "workspaces"
    (for wsp in {workspaces[screen]}
      (button :class "${wsp.focused ? "active" : "inactive"}"
              :onclick "swaymsg workspace ${wsp.name}"
        {wsp.name}))))
```

---

**Comparison: Python workspace approaches**

| Aspect | druskus20 | elkowar (dots-of-war) |
|---|---|---|
| Output shape | Dict keyed by monitor name | Dict keyed by monitor name |
| Class/icon | Pre-computed in Python | Computed inline in yuck |
| Loop order | Emit first, then read event | Emit first, then read event |
| Extra fields | `class`, `icon` | None beyond raw sway data |
| Multi-monitor | Yes, via `workspaces[monitor]` | Yes, via `workspaces[screen]` |
| `flush=True` | Yes | Yes |

Both scripts use the same event loop structure: emit current state immediately, then block on `readline()` waiting for the next sway workspace event, then emit again. **`flush=True` on every `print` call is mandatory** — without it, Python buffers stdout in the pipe and eww never receives updates until the buffer fills (typically 4KB). This is the most common source of "my workspace script does nothing" bugs.

---

**Approach C — rxyhn: BSPWM, literal yuck output from shell**

rxyhn's bar was built for BSPWM (not Sway), and uses a fundamentally different approach: the shell script emits raw yuck widget markup as a string, and `(literal :content workspace)` renders it dynamically.

```bash
# Source: rxyhn — github.com/rxyhn/yoru
#!/bin/sh

workspaces() {
  ws1=1; ws2=2; ws3=3; ws4=4; ws5=5; ws6=6
  un="0"

  o1=$(bspc query -D -d .occupied --names | grep "$ws1")
  # ... (one line per workspace for occupied and focused state)
  f1=$(bspc query -D -d focused --names | grep "$ws1")
  # ...

  echo "(box :class \"works\" :orientation \"v\" ... \
    (button :onclick \"bspc desktop -f $ws1\" :class \"$un$o1$f1\" \"\") \
    ...)"
}

workspaces
bspc subscribe desktop node_transfer | while read -r _; do
  workspaces
done
```

```yuck
; Source: rxyhn — github.com/rxyhn/yoru
(defwidget workspaces []
  (literal :content workspace))
(deflisten workspace "scripts/workspace")
```

**Key difference from the Python approach:** `(literal :content ...)` lets the script generate arbitrary widget markup. This is powerful but harder to style consistently — the entire widget tree, including classes, is built in shell string concatenation. The Python + `for` loop approach is cleaner for maintenance.

---

### Volume / Audio

Three real approaches ranging from a one-liner `defpoll` to a full `deflisten` with JSON output and deduplication.

---

**owenrumney's getvol — simplest possible defpoll script**

```sh
# Source: owenrumney — github.com/owenrumney/dotfiles
#!/bin/sh
amixer -D pulse sget Master | grep 'Left:' | awk -F'[][]' '{ print $2 }' | tr -d '%' | head -1
```

```yuck
; Source: owenrumney — github.com/owenrumney/dotfiles
(defpoll volume :interval "1s" "scripts/getvol")
```

No JSON, no streaming — just an integer on stdout. Simple and correct for low-frequency polling. The awk `-F'[][]'` splits on `[` or `]` characters, extracting the percentage from amixer's `[50%]` format.

---

**dots-of-war's audio.sh — deflisten with mode switching via $1**

One script handles three modes: `symbol` (sink icon stream), `volume` (volume integer stream), and `toggle` (imperative sink switch). Each mode is invoked separately from yuck.

```bash
# Source: elkowar — github.com/elkowar/dots-of-war
#!/usr/bin/env bash

case "$1" in
  "symbol")
    pactl subscribe | grep --line-buffered "Event 'change' on client" | while read -r; do
      case "$(pactl get-default-sink)" in
        *Arctis_9*) echo "";;
        *)          echo "";;
      esac
    done
    ;;
  "volume")
    pamixer --get-volume
    pactl subscribe \
      | grep --line-buffered "Event 'change' on sink " \
      | while read -r evt; do
          pamixer --get-volume | cut -d " " -f1
        done
    ;;
  "toggle")
    speaker_sink_id=$(pamixer --list-sinks | grep "Komplete_Audio_6" | awk '{print $1}')
    game_sink_id=$(pamixer --list-sinks | grep "stereo-game"         | awk '{print $1}')
    case "$(pactl get-default-sink)" in
      *Arctis_9*)
        eww -c ~/.config/eww-bar update audio_sink=""
        pactl set-default-sink $speaker_sink_id
        ;;
      *)
        eww -c ~/.config/eww-bar update audio_sink=""
        pactl set-default-sink $game_sink_id
        ;;
    esac
    ;;
esac
```

Note `--line-buffered` on the `grep` calls — without it, grep's own output buffering would block the pipe and eww would see nothing until grep's internal buffer fills.

The `volume` mode emits the current volume immediately before entering the subscribe loop (line: `pamixer --get-volume`). This is the "initial emit before the loop" pattern — without it the widget shows `:initial` until the first sink event.

```yuck
; Source: elkowar — github.com/elkowar/dots-of-war
(defvar audio_sink "")
(deflisten volume :initial "0" "./audio.sh volume")

(defwidget volume []
  (box :class "volume-metric" :orientation "v" :space-evenly false
    (scale :orientation "h" :min 0 :max 100
           :onchange "pamixer --set-volume $(echo {} | sed 's/\\..*//g')"
           :value volume)
    (eventbox :onscroll "if [ '{}' == 'up' ]; then pamixer -i 5; else pamixer -d 5; fi"
      (box :orientation "v"
        (button :onclick "..." :onrightclick "./audio.sh toggle"
          (label :text audio_sink))
        (button :onclick "pavucontrol &"
          "${volume}%")))))
```

`audio_sink` is a `defvar` (not a deflisten) that gets updated imperatively by `audio.sh toggle` calling `eww update audio_sink=...`. This is the pattern for state that changes only on user action, not continuously.

---

**druskus20's listen-volume.sh — full deflisten with JSON output and deduplication**

```bash
# Source: druskus20 — github.com/druskus20/simpler-bar
#!/bin/bash

get_sink_volume_and_mute_status() {
    sink_id=$1
    # Extract volume percentage
    volume=$(pactl list sinks | grep -A 10 "Sink #$sink_id" | grep 'Volume:' | awk '{print $5}' | cut -d'%' -f1)
    # Extract mute status
    mute_status=$(pactl list sinks | grep -A 15 "Sink #$sink_id" | grep 'Mute:' | awk '{print $2}')

    if [[ -z "$volume" || -z "$mute_status" ]]; then
        return 1
    fi

    # Format the output
    echo "{\"volume\": $volume, \"muted\": $( [[ "$mute_status" == "yes" ]] && echo "true" || echo "false" ) }"
}

DEFAULT_SINK_NAME="$(pactl get-default-sink)"
DEFAULT_SINK_ID="$(pactl list sinks short | grep "$DEFAULT_SINK_NAME" | awk '{print $1}')"
CURR_STATUS="$(get_sink_volume_and_mute_status $DEFAULT_SINK_ID)"

echo "$CURR_STATUS"  # initial emit before the loop

pactl subscribe | while read -r line; do
    if echo "$line" | grep -q "Event 'change' on sink #$DEFAULT_SINK_ID"; then
        NEW_STATUS="$(get_sink_volume_and_mute_status $DEFAULT_SINK_ID)"
        # Avoids double events
        if [[ "$CURR_STATUS" != "$NEW_STATUS" ]]; then
            CURR_STATUS=$NEW_STATUS
            echo "$CURR_STATUS"
        fi
        # Update default sink in case it has changed
        DEFAULT_SINK_NAME="$(pactl get-default-sink)"
        DEFAULT_SINK_ID=$(pactl list sinks short | grep "$DEFAULT_SINK_NAME" | awk '{print $1}')
    fi
done
```

Key patterns in this script:

1. **Initial emit before the loop** — prints current state immediately before entering the subscribe loop. Without this, the widget shows `:initial` until the first real event.
2. **Deduplication** — `pactl subscribe` fires multiple events per single volume change (one per channel on some setups). The `CURR_STATUS != NEW_STATUS` guard prevents redundant updates.
3. **JSON with real booleans** — `muted` emits `true`/`false` (unquoted), so yuck can use `speakers.muted` as a boolean directly in ternary expressions.
4. **Sink ID tracking** — re-queries the default sink after each event so it follows sink switches at runtime.

```yuck
; Source: druskus20 — github.com/druskus20/simpler-bar
(deflisten speakers :initial '{ "volume": 0, "muted": "false" }' "./modules/listen-volume.sh")

(defwidget speakers []
  (tooltip
    (label :class "tooltip" :text "${ speakers.volume }%")
    (box :class "speakers" :space-evenly false
      (button :class "icon ${ speakers.volume > 100 ? 'over' : '' } ${ speakers.muted ? 'muted' : '' }"
              :onclick "pactl set-sink-mute @DEFAULT_SINK@ toggle"
        (label :text "${ speakers.muted ? '󰖁' :
                        speakers.volume > 70 ? '󰕾' :
                        speakers.volume > 35 ? '󰖀' :
                        '󰿿' }"))
      (scale :class "${ speakers.volume > 100 ? 'over' : '' }"
             :min 0 :max 100
             :value "${speakers.volume}"
             :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"))))
```

The JSON fields map directly to dot notation: `speakers.volume`, `speakers.muted`. The ternary chain on the icon is a common eww pattern — one expression, multiple thresholds.

druskus20 also applies the same structure to mic (source) monitoring, emitting a plain integer instead of JSON (0 = muted, 1 = idle, 2 = running) since it's a simple state machine with no need for multiple fields:

```yuck
; Source: druskus20 — github.com/druskus20/simpler-bar
(deflisten mic :initial 0 "./modules/listen-mic.sh")

(defwidget mic []
  (box :class "mic" :space-evenly false
    (button :class "mute-toggle"
            :onclick "pactl set-source-mute @DEFAULT_SOURCE@ toggle"
      ; 0: muted, 1: idle, 2: running
      (label :class "icon single-icon ${mic == 2 ? 'running' : mic == 0 ? 'muted' : ''}"
             :text "${mic == 0 ? '󰍭' : '󰍬' }"))))
```

The mic script emits a plain integer — the yuck side does all class and icon selection via ternary chains. Valid alternative to JSON for simple state machines with three or fewer states.

---

**rxyhn — defpoll inline, amixer**

rxyhn inlines the amixer command directly in the defpoll declaration rather than calling a script file:

```yuck
; Source: rxyhn — github.com/rxyhn/yoru
(defpoll current-volume :interval "1s"
  "amixer -D pulse sget Master | grep 'Left:' | awk -F'[][]' '{ print $2 }' | tr -d '%'")

(defwidget volum []
  (eventbox :onhover "${eww} update volum=true"
            :onhoverlost "${eww} update volum=false"
    (box :orientation "v" :space-evenly "false" :spacing "2"
      (revealer :transition "slideup" :reveal volum :duration "550ms"
        (scale :class "volbar"
               :value current-volume
               :orientation "v" :flipped true
               :tooltip "Volume: ${current-volume}%"
               :max 101 :min 0
               :onchange "amixer -D pulse sset Master {}%"))
      (button :onclick "scripts/popup audio" :class "volume-icon" ""))))
```

`volum` (the revealer state) is a `defvar` toggled by hover events — not a deflisten. The scale slider both reads and writes volume: `:value` reads from the poll, `:onchange` writes back via amixer.

---

### Battery

**rxyhn — argument-dispatched script, two defpolls**

```sh
# Source: rxyhn — github.com/rxyhn/yoru
#!/bin/sh

bat=/sys/class/power_supply/BAT0/
per="$(cat "$bat/capacity")"

icon() {
  [ $(cat "$bat/status") = Charging ] && echo "" && exit

  if   [ "$per" -gt "90" ]; then icon=""
  elif [ "$per" -gt "80" ]; then icon=""
  elif [ "$per" -gt "70" ]; then icon=""
  elif [ "$per" -gt "60" ]; then icon=""
  elif [ "$per" -gt "50" ]; then icon=""
  elif [ "$per" -gt "40" ]; then icon=""
  elif [ "$per" -gt "30" ]; then icon=""
  elif [ "$per" -gt "20" ]; then icon=""
  elif [ "$per" -gt "10" ]; then
    icon=""
    notify-send -u critical "Battery Low" "Connect Charger"
  elif [ "$per" -gt "0" ]; then
    icon=""
    notify-send -u critical "Battery Low" "Connect Charger"
  else
    echo && exit
  fi
  echo "$icon"
}

percent() { echo $per; }

[ "$1" = "icon"    ] && icon    && exit
[ "$1" = "percent" ] && percent && exit
```

The script reads from `/sys/class/power_supply/BAT0/` directly — no external tools. The `notify-send` calls inside `icon()` mean a low-battery notification fires every poll interval (1s), which could spam notifications. rxyhn accepts this trade-off; in production you'd add a debounce flag.

```yuck
; Source: rxyhn — github.com/rxyhn/yoru
(defpoll battery           :interval "1s" "scripts/battery icon")
(defpoll battery-cappacity :interval "1s" "scripts/battery percent")

(defwidget bat []
  (box :orientation "v" :space-evenly "false"
    (label :class "bat"
           :halign "end"
           :text battery
           :tooltip "Battery: ${battery-cappacity}%")))
```

Two separate defpolls invoke the same script with different arguments. The typo `battery-cappacity` is in the original source.

**druskus20 uses EWW_BATTERY instead** — the built-in magic variable, avoiding a script entirely:

```yuck
; Source: druskus20 — github.com/druskus20/simpler-bar
(defwidget battery []
  (box :class "battery" :space-evenly false
    (label :class "icon ${ EWW_BATTERY["BAT0"].capacity <= 10 ? 'critical' : '' }"
           :text "${  EWW_BATTERY["BAT0"].capacity > 95 ? '󰁹'
                    : EWW_BATTERY["BAT0"].capacity > 90 ? '󰂂'
                    : EWW_BATTERY["BAT0"].capacity > 80 ? '󰂁'
                    : EWW_BATTERY["BAT0"].capacity > 70 ? '󰂀'
                    : EWW_BATTERY["BAT0"].capacity > 60 ? '󰁿'
                    : EWW_BATTERY["BAT0"].capacity > 50 ? '󰁾'
                    : EWW_BATTERY["BAT0"].capacity > 40 ? '󰁽'
                    : EWW_BATTERY["BAT0"].capacity > 30 ? '󰁼'
                    : EWW_BATTERY["BAT0"].capacity > 20 ? '󰁻'
                    : EWW_BATTERY["BAT0"].capacity > 10 ? '󰁺'
                    : '󰂃' }")
    (label :text "${EWW_BATTERY["BAT0"].capacity}" :halign "center" :xalign 0.5 :justify "right")))
```

Prefer `EWW_BATTERY` over a custom script when you only need capacity and status — it's built in, updates automatically, and eliminates one polling process.

---

### WiFi

**druskus20 — defpoll, returns SSID string or "0"**

```bash
# Source: druskus20 — github.com/druskus20/simpler-bar
#!/bin/bash

wifi_ssid=$(nmcli -t -f ACTIVE,SSID dev wifi | awk -F':' '$1=="yes" {print $2}')

if [[ -z "$wifi_ssid" ]]; then
    echo "0"
else
    echo "$wifi_ssid"
fi
```

```yuck
; Source: druskus20 — github.com/druskus20/simpler-bar
(defpoll wifi :initial "0" :interval "1m" "./modules/wifi.sh")

(defwidget wifi []
  (tooltip
    (label :class "tooltip" :text "${wifi == 0 ? 'Disconnected' : wifi}")
    (box :class "wifi" :space-evenly false
      (label :class "icon ${wifi == 0 ? 'disconnected' : 'connected'}"
             :text "${wifi == 0 ? '󰖪' : '󱚽' }"))))
```

Sentinel value `"0"` doubles as the disconnected state — avoids needing a JSON struct for a simple on/off + name widget. Polling at `1m` is appropriate for network info (it changes rarely).

---

**rxyhn — argument-dispatched, separate icon and name, reads kernel sysfs**

```sh
# Source: rxyhn — github.com/rxyhn/yoru
#!/bin/sh

symbol() {
  [ $(cat /sys/class/net/w*/operstate) = down ] && echo  && exit
  echo
}

name() {
  nmcli | grep "^wlp" | sed 's/\ connected\ to\ /Connected to /g' | cut -d ':' -f2
}

[ "$1" = "icon" ] && symbol && exit
[ "$1" = "name" ] && name   && exit
```

```yuck
; Source: rxyhn — github.com/rxyhn/yoru
(defpoll wifi-icon :interval "1s" "scripts/wifi icon")
(defpoll wifi-name :interval "1s" "scripts/wifi name")

(defwidget wifi []
  (box :orientation "v" :tooltip wifi-name
    (button :onclick "scripts/popup wifi"
            :class "wifi-icon" wifi-icon)))
```

`symbol()` checks `/sys/class/net/w*/operstate` (kernel net state via glob over wireless interfaces) instead of nmcli — faster and works without NetworkManager running. `name()` still uses nmcli for the human-readable SSID since sysfs doesn't expose it.

---

**owenrumney — defpoll with iwgetid**

```yuck
; Source: owenrumney — github.com/owenrumney/dotfiles
(defpoll wirelessId   :interval "60s" "iwgetid -r")
(defpoll interfaceId  :interval "60s" "route | grep default | head -n1 | awk '{print $8}'")
```

Minimal — `iwgetid -r` outputs just the SSID. No script file at all, just an inline command. Polling at 60s is appropriate since network changes are rare.

---

### Popup Toggle

**rxyhn — lockfile-based toggle, single dispatcher script**

```sh
# Source: rxyhn — github.com/rxyhn/yoru
#!/bin/sh

calendar() {
  LOCK_FILE="$HOME/.cache/eww-calendar.lock"
  EWW_BIN="$HOME/.local/bin/eww"

  run() {
    ${EWW_BIN} -c $HOME/.config/eww/bar open calendar
  }

  # Run eww daemon if not running
  if [[ ! `pidof eww` ]]; then
    ${EWW_BIN} daemon
    sleep 1
  fi

  # Open widgets
  if [[ ! -f "$LOCK_FILE" ]]; then
    touch "$LOCK_FILE"
    run
  else
    ${EWW_BIN} -c $HOME/.config/eww/bar close calendar
    rm "$LOCK_FILE"
  fi
}

if   [ "$1" = "launcher"  ]; then $HOME/.local/bin/launcher
elif [ "$1" = "wifi"      ]; then kitty -e nmtui
elif [ "$1" = "audio"     ]; then pavucontrol
elif [ "$1" = "calendar"  ]; then calendar
fi
```

A single `popup` script handles all button actions via argument dispatch. The lockfile pattern (`~/.cache/eww-*.lock`) is the idiomatic eww toggle: create the file on open, delete it on close, check existence to decide which to do.

```yuck
; Source: rxyhn — github.com/rxyhn/yoru
(button :onclick "scripts/popup calendar" :class "time-hour" hour)
(button :onclick "scripts/popup wifi"     :class "wifi-icon" wifi-icon)
(button :onclick "scripts/popup audio"    :class "volume-icon" "")
(button :onclick "scripts/popup launcher" :class "launcher_icon" "")
```

One script file, four buttons. The `wifi` and `audio` cases just launch external apps (nmtui, pavucontrol) instead of toggling eww windows — the popup script is the central dispatcher for all bar button actions.

rxyhn also has a separate calendar month helper with an important gotcha:

```sh
# Source: rxyhn — github.com/rxyhn/yoru
#!/bin/sh
month=$(date +%m)
month=$((month-1))  # eww's calendar widget takes month as 0-based integer
echo $month
```

The eww `(calendar)` widget's `:month` property is zero-indexed (0 = January), but `date +%m` outputs 1-indexed. Subtract 1.

---

### Camera Presence Detection

**druskus20 — lsmod-based kernel module check**

```bash
# Source: druskus20 — github.com/druskus20/simpler-bar
#!/bin/bash

usage_count=$(lsmod | grep '^uvcvideo' | awk '{print $3}')
if [[ "$usage_count" -gt 0 ]]; then
    echo 1
else
    echo 0
fi
```

```yuck
; Source: druskus20 — github.com/druskus20/simpler-bar
(defpoll camera :initial "0" :interval "10s" "./modules/camera.sh")

(defwidget camera []
  (box :class "camera" :space-evenly false
    (label :class "icon ${camera == 0 ? 'disconnected' : 'connected'}"
           :text "${camera == 0 ? '󱜷' : '󰖠'}")))
```

`uvcvideo` is the Linux kernel module for USB video class devices (webcams). The third column of `lsmod` is the use count — non-zero means a process is actively using the camera. No external tools needed beyond lsmod.

---

### owenrumney's RAM Script

```sh
# Source: owenrumney — github.com/owenrumney/dotfiles
#!/bin/sh
printf "%.0f\n" $(free -m | grep Mem | awk '{print ($3/$2)*100}')
```

Note that eww provides `EWW_RAM.used_mem_perc` as a built-in — no script needed for basic RAM percentage. Use a custom script only if you need additional fields (swap, available, etc.) or a specific format.

```yuck
; Using the built-in magic variable instead of a script
(label :text "${round(EWW_RAM.used_mem_perc, 0)}%")
```
