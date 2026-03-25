# Community eww Configs — Code Reference

Real code patterns from named community configs, attributed by author, so Claude can match style requests to actual implementation details.

---

## druskus20

Repo: github.com/druskus20/eugh — Catppuccin-themed multi-monitor horizontal bar with deflisten-driven workspaces, per-monitor JSON routing, hover revealer components, and a vertical sidebar workspace-indicator concept.

### Window definition (multi-monitor)

```lisp
; Source: druskus20 — github.com/druskus20/eugh (simpler-bar)
; Three separate defwindow instances, one per monitor.
; Width "100.01%" is a workaround for a 1px gap on secondary monitors.

(defwindow win0 :monitor 0
               :geometry (geometry :x 0 :y 0 :height "15px" :width "100%" :anchor "center top")
               :stacking "fg"
               :exclusive true
               :focusable false
  (bar0 :monitor "eDP-1"))

(defwindow win1 :monitor 1
               :geometry (geometry :x 0 :y 0 :height "15px" :width "100.01%" :anchor "center top")
               :stacking "fg"
               :exclusive true
               :focusable false
  (bar0 :monitor "HDMI-A-1"))
```

### Bar widget layout

```lisp
; Source: druskus20 — github.com/druskus20/eugh (simpler-bar)
; Bar receives monitor name as a param so workspaces can filter by output.

(defwidget bar0 [monitor]
  (box :orientation "h"
       :space-evenly true
       :class "bar"
    (box :class "right" :halign "start" :spacing 20
      (workspaces :monitor monitor))
    (box :class "right" :halign "end" :spacing 20 :space-evenly false
      (tray)
      (wifi)
      (camera)
      (mic)
      (speakers)
      (battery)
      (time)
      (date))))
```

### Workspace tracking (per-monitor JSON)

```lisp
; Source: druskus20 — github.com/druskus20/eugh (simpler-bar)
; workspaces is a JSON object keyed by output name.
; Each entry has: name, monitor, focused, visible, class, icon.

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

### Workspace script (sway-workspaces.py)

```python
# Source: druskus20 — github.com/druskus20/eugh (simpler-bar)
# Forked from elkowar. Adds icon field: "" focused, "" visible, "" hidden.
# Emits JSON once on start, then on every workspace event.

#!/usr/bin/env python3
import subprocess, json

def get_workspaces():
    output = subprocess.check_output(["swaymsg", "-t", "get_workspaces"])
    return json.loads(output.decode("utf-8"))

def generate_workspace_data() -> dict:
    data = {}
    for wsp in get_workspaces():
        if wsp["output"] not in data:
            data[wsp["output"]] = []
        i = {"name": wsp["name"], "monitor": wsp["output"],
             "focused": wsp["focused"], "visible": wsp["visible"]}
        if wsp["focused"]:
            i["class"] = "focused"; i["icon"] = ""
        elif wsp["visible"]:
            i["class"] = "visible"; i["icon"] = ""
        else:
            i["class"] = "hidden";  i["icon"] = ""
        data[wsp["output"]].append(i)
    return data

if __name__ == "__main__":
    process = subprocess.Popen(
        ["swaymsg", "-t", "subscribe", "-m", '["workspace"]', "--raw"],
        stdout=subprocess.PIPE)
    while True:
        print(json.dumps(generate_workspace_data()), flush=True)
        line = process.stdout.readline().decode("utf-8")
        if line == "": break
```

### Volume (deflisten with pactl subscribe)

```bash
# Source: druskus20 — github.com/druskus20/eugh (simpler-bar/modules/listen-volume.sh)
# Emits JSON: {"volume": 72, "muted": false}
# Uses pactl subscribe to react to real sink events. Deduplicates double-fire.

get_sink_volume_and_mute_status() {
    sink_id=$1
    volume=$(pactl list sinks | grep -A 10 "Sink #$sink_id" | grep 'Volume:' | awk '{print $5}' | cut -d'%' -f1)
    mute_status=$(pactl list sinks | grep -A 15 "Sink #$sink_id" | grep 'Mute:' | awk '{print $2}')
    echo "{\"volume\": $volume, \"muted\": $( [[ "$mute_status" == "yes" ]] && echo "true" || echo "false" ) }"
}

DEFAULT_SINK_NAME="$(pactl get-default-sink)"
DEFAULT_SINK_ID="$(pactl list sinks short | grep "$DEFAULT_SINK_NAME" | awk '{print $1}')"
CURR_STATUS="$(get_sink_volume_and_mute_status $DEFAULT_SINK_ID)"
echo "$CURR_STATUS"

pactl subscribe | while read -r line; do
    if echo "$line" | grep -q "Event 'change' on sink #$DEFAULT_SINK_ID"; then
        NEW_STATUS="$(get_sink_volume_and_mute_status $DEFAULT_SINK_ID)"
        if [[ "$CURR_STATUS" != "$NEW_STATUS" ]]; then
            CURR_STATUS=$NEW_STATUS
            echo "$CURR_STATUS"
        fi
        DEFAULT_SINK_NAME="$(pactl get-default-sink)"
        DEFAULT_SINK_ID=$(pactl list sinks short | grep "$DEFAULT_SINK_NAME" | awk '{print $1}')
    fi
done
```

### Volume widget (with inline scale)

```lisp
; Source: druskus20 — github.com/druskus20/eugh (simpler-bar)
; speakers is a JSON object with .volume and .muted fields.
; Scale is always visible; icon changes with mute/volume level.

(deflisten speakers :initial '{ "volume": 0, "muted": "false" }' "./modules/listen-volume.sh")

(defwidget speakers []
  (tooltip
    (label :class "tooltip" :text "${ speakers.volume }%")
    (box :class "speakers" :space-evenly false
      (button :class "icon ${ speakers.volume > 100 ? 'over' : '' } ${ speakers.muted ? 'muted' : '' }"
              :onclick "pactl set-sink-mute @DEFAULT_SINK@ toggle"
        (label :text "${ speakers.muted ? '󰖁' :
                        speakers.volume > 70 ? '󰕾' :
                        speakers.volume > 35 ? '󰖀' : '󰕿' }"))
      (scale :class "${ speakers.volume > 100 ? 'over' : '' }"
             :min 0 :max 100
             :value "${speakers.volume}"
             :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"))))
```

### Revealer on hover (generic component)

```lisp
; Source: druskus20 — github.com/druskus20/eugh (revealer-hover-module)
; Generic hover-reveal pattern. Children: [0]=icon when collapsed,
; [1]=content revealed on hover, [2]=trailing content.
; hovered-sign swaps between two child slots based on var state.

(defwidget hovered-sign [var]
  (box :space-evenly false
    (revealer :reveal {!var} :duration "100ms" :transition "slideleft"
      (children :nth 0))
    (revealer :reveal {var} :duration "100ms" :transition "slideleft"
      (children :nth 1))))

(defwidget revealer-on-hover [var varname ?class ?duration ?transition]
  (box :class "${class} revealer-on-hover" :orientation "h" :space-evenly false
    (eventbox :class "eventbox"
              :onhover "eww update ${varname}=true"
              :onhoverlost "eww update ${varname}=false"
      (box :space-evenly false
        (children :nth 0)
        (revealer :reveal var
                  :transition {transition ?: "slideright"}
                  :duration {duration ?: "500ms"}
          (children :nth 1))
        (children :nth 2)))))
```

### Vertical sidebar workspace-indicator

```lisp
; Source: druskus20 — github.com/druskus20/eugh (workspace-indicator)
; Left sidebar: small dots per workspace, large focused number.
; Uses bspwm queries. Two ws groups on separate monitors.

(defwindow bar
  :monitor 0
  :reserve (struts :distance "50px" :side "left")
  :geometry (geometry :anchor "top left" :x 0 :y 0 :width "50px" :height "100%")
  (box :orientation "vertical" :space-evenly false :valign "start"
    (ws)))

(defwidget ws []
  (box :orientation "vertical" :space-evenly false
    (small-wss :wss "${WS.ws1}")
    (small-wss :wss "${WS.ws2}")
    (big-ws :focused-ws "${WS.focused}")))

(defwidget small-ws [ws]
  (eventbox :cursor "hand"
    (button :onclick "bspc desktop -f ${ws.name}"
      (label :class "small-ws status-${ws.status}" :text "${ws.icon}"))))

(defwidget big-ws [focused-ws]
  (box :height "50" :width "50"
    (label :text "${focused-ws}")))
```

```scss
/* Source: druskus20 — github.com/druskus20/eugh (workspace-indicator/eww.scss) */
/* Status classes control icon color opacity */
.ws .small-ws {
    &.status-empty    { color: rgba(237, 112, 112, 0.3); }
    &.status-occupied { color: rgba(237, 112, 112, 0.5); }
    &.status-focused  { color: rgba(237, 112, 112, 1.0); }
    &.status-urgent   { color: rgba(255, 255, 255, 1.0); }
}
.ws .big-ws {
    font-size: 20px;
    font-weight: bold;
    background-image: url("res/rhombus.png");
    background-size: 100% 100%;
    background-position: center;
}
```

### SCSS theming (Catppuccin Mocha)

```scss
/* Source: druskus20 — github.com/druskus20/eugh (simpler-bar/eww.scss) */
$rosewater: #f5e0dc; $flamingo: #f2cdcd; $pink: #f5c2e7;
$mauve: #cba6f7; $red: #f38ba8; $maroon: #eba0ac;
$peach: #fab387; $yellow: #f9e2af; $green: #a6e3a1;
$teal: #94e2d5; $sky: #89dceb; $sapphire: #74c7ec;
$blue: #89b4fa; $lavender: #b4befe; $text: #cdd6f4;
$subtext1: #bac2de; $subtext0: #a6adc8; $overlay2: #9399b2;
$overlay1: #7f849c; $overlay0: #6c7086; $surface2: #585b70;
$surface1: #45475a; $surface0: #313244; $base: #1e1e2e;
$mantle: #181825; $crust: #11111b;

* { all: unset; }
window { font-family: "NotoSans Nerd Font", sans-serif; background-color: $base; }

/* Scale trough: highlight goes orange when volume > 100% */
scale trough { background-color: $crust; border-radius: 2px; min-height: 5px; min-width: 50px; }
scale trough highlight { background-color: $text; border-radius: 2px; }
scale.over trough highlight { background-color: $peach; }

/* Tray icon scaling via GTK transform */
.tray image { -gtk-icon-transform: scale(0.5); }
```

---

## elkowar (dots-of-war)

Repo: github.com/elkowar/dots-of-war — The canonical reference for Sway workspace integration. Vertical left sidebar using `centerbox`, per-monitor workspace routing, `pamixer`-based volume, and the original swayspaces.py that every other config forks.

### Window definition (vertical sidebar)

```lisp
; Source: elkowar — github.com/elkowar/dots-of-war (eww-bar)
; 40px wide left sidebar, anchored center-left. Uses struts to reserve space.
; Monitor matched by array: primary first, then by name or index as fallback.

(defwindow bar_1
  :monitor '["<primary>", "DisplayPort-0", "PHL 345B1C"]'
  :stacking "fg"
  :geometry (geometry :x 0 :y 0 :width "40px" :height "100%" :anchor "center left")
  :reserve (struts :distance "40px" :side "left")
  :exclusive true
  (bar :screen 1))

(defwindow bar_2
  :monitor '[2, "HDMI-A-1"]'
  :geometry (geometry :x 0 :y 0 :width "40px" :height "100%" :anchor "top left")
  :reserve (struts :distance "40px" :side "left")
  (bar :screen 2))
```

### Bar layout (centerbox vertical)

```lisp
; Source: elkowar — github.com/elkowar/dots-of-war (eww-bar)
; centerbox :orientation "v" gives top/center/bottom sections naturally.
; screen param routes workspace data to the correct monitor.

(defwidget bar [screen]
  (centerbox :orientation "v"
    (box :class "segment-top" :valign "start"
      (top :screen screen))
    (box :valign "center" :class "middle"
      (middle :screen screen))
    (box :valign "end" :class "segment-bottom"
      (bottom :screen screen))))

(defwidget bottom [screen]
  (box :orientation "v" :valign "end" :space-evenly true :spacing "5"
    (volume)
    (metric :icon "" :font-size 0.8
      "${round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}%")
    (metric :icon "" "${round(EWW_RAM.used_mem_perc, 0)}%")
    (metric :icon "" "${round(EWW_CPU.avg, 0)}%")
    (box :class "metric" (date))))
```

### Workspace tracking (original swayspaces.py)

```python
# Source: elkowar — github.com/elkowar/dots-of-war (eww-bar/swayspaces.py)
# The original. Emits a dict keyed by output name.
# Each value is a list of {name, monitor, focused, visible}.

#!/usr/bin/env python3
import subprocess, json

def get_workspaces():
    output = subprocess.check_output(["swaymsg", "-t", "get_workspaces"])
    return json.loads(output.decode("utf-8"))

def generate_workspace_data() -> dict:
    data = {}
    for wsp in get_workspaces():
        if wsp["output"] not in data:
            data[wsp["output"]] = []
        data[wsp["output"]].append({
            "name": wsp["name"], "monitor": wsp["output"],
            "focused": wsp["focused"], "visible": wsp["visible"],
        })
    return data

if __name__ == "__main__":
    process = subprocess.Popen(
        ["swaymsg", "-t", "subscribe", "-m", '["workspace"]', "--raw"],
        stdout=subprocess.PIPE)
    while True:
        print(json.dumps(generate_workspace_data()), flush=True)
        line = process.stdout.readline().decode("utf-8")
        if line == "": break
```

### Audio script (sink switching + volume stream)

```bash
# Source: elkowar — github.com/elkowar/dots-of-war (eww-bar/audio.sh)
# Multi-mode script: "volume" emits a stream for deflisten,
# "symbol" streams sink icons, "toggle" switches between two sinks.

case "$1" in
  "volume")
    pamixer --get-volume
    pactl subscribe \
      | grep --line-buffered "Event 'change' on sink " \
      | while read -r evt; do pamixer --get-volume | cut -d " " -f1; done
    ;;
  "symbol")
    pactl subscribe | grep --line-buffered "Event 'change' on client" | while read -r; do
      case "$(pactl get-default-sink)" in
        *Arctis_9*) echo "";;
        *)          echo "";;
      esac
    done
    ;;
  "toggle")
    speaker_sink_id=$(pamixer --list-sinks | grep "Komplete_Audio_6" | awk '{print $1}')
    game_sink_id=$(pamixer --list-sinks | grep "stereo-game" | awk '{print $1}')
    case "$(pactl get-default-sink)" in
      *Arctis_9*)
        eww -c ~/.config/eww-bar update audio_sink=""
        pactl set-default-sink $speaker_sink_id ;;
      *)
        eww -c ~/.config/eww-bar update audio_sink=""
        pactl set-default-sink $game_sink_id ;;
    esac
    ;;
esac
```

### Volume widget (vertical slider + scroll + click)

```lisp
; Source: elkowar — github.com/elkowar/dots-of-war (eww-bar)
; Vertical scale at top, eventbox below for scroll-to-adjust.
; Button opens rofi sink chooser; right-click calls audio.sh toggle.

(deflisten volume :initial "0" "./audio.sh volume")

(defwidget volume []
  (box :class "volume-metric" :orientation "v" :space-evenly false
    (scale :orientation "h" :min 0 :max 100
           :onchange "pamixer --set-volume $(echo {} | sed 's/\\..*//g')"
           :value volume)
    (eventbox :onscroll "if [ '{}' == 'up' ]; then pamixer -i 5; else pamixer -d 5; fi"
              :vexpand true :valign "fill"
      (box :orientation "v" :valign "fill" :vexpand true
        (button :onclick "rofi -show rofi-sound -modi 'rofi-sound:rofi-sound-output-chooser' &"
                :onrightclick "./audio.sh toggle"
          (label :text audio_sink))
        (button :onclick "pavucontrol &" "${volume}%")))))
```

### Vertical metric widget

```lisp
; Source: elkowar — github.com/elkowar/dots-of-war (eww-bar)
; Reusable metric: icon above, child content below.
; Optional font-size param applies inline style to the icon.

(defwidget metric [icon ?font-size]
  (box :class "metric" :orientation "v"
    (label :class "metric-icon"
           :style {font-size != "" ? "font-size: ${font-size}rem;" : ""}
           :text icon)
    (children)))
```

### SCSS (Gruvbox dark)

```scss
/* Source: elkowar — github.com/elkowar/dots-of-war (eww-bar/eww.scss) */
window {
  background: #282828; color: #ebdbb2;
  font-size: 14px; font-family: "Terminus (TTF)";
}
.workspaces button {
  &.inactive { color: #888974; }
  &.active   { color: #8ec07c; }
}
.volume-metric scale trough {
  all: unset; background-color: #1d2021;
  min-width: 34px; min-height: 2px;
}
.volume-metric scale trough highlight {
  all: unset; background-color: #665c54; border-bottom-right-radius: 5px;
}
.metric { background-color: #1d2021; padding: 5px 2px; }
.metric-icon { font-family: "Font Awesome 6 Free"; font-size: 0.7em; }
```

---

## rxyhn

Repo: github.com/rxyhn/yoru — Vertical left sidebar for bspwm. Signature: Tokyo Night colors, `literal`+shell-script workspace rendering (dynamic yuck generation), hover-reveal sliders for volume and brightness, Comic Mono clock font, and a hover powermenu.

### Window definition (vertical bar)

```lisp
; Source: rxyhn — github.com/rxyhn/yoru (eww/bar)
; 47px wide left bar. defvar stores eww path so widgets can call "eww update".

(defvar eww "$HOME/.local/bin/eww -c $HOME/.config/eww/bar")

(defwindow bar
  :geometry (geometry :x "0" :y "0" :height "100%" :width "47px")
  :monitor 0
  :reserve (struts :distance "35px" :side "left")
  :wm-ignore false
  :hexpand "false" :vexpand "false"
  (bar))
```

### Workspace tracking (bspwm + literal yuck)

```bash
# Source: rxyhn — github.com/rxyhn/yoru (eww/bar/scripts/workspace)
# Generates raw yuck markup as a string; consumed by (literal :content workspace).
# Classes like "0", "01", "011" encode: unoccupied / occupied / focused.

workspaces() {
  ws1=1; ws2=2; ws3=3; ws4=4; ws5=5; ws6=6; un="0"
  o1=$(bspc query -D -d .occupied --names | grep "$ws1")
  f1=$(bspc query -D -d focused   --names | grep "$ws1")
  # ... repeat for ws2-ws6 ...
  echo "(box :class \"works\" :orientation \"v\" :halign \"center\" \
    :space-evenly \"false\" :spacing \"-5\" \
    (button :onclick \"bspc desktop -f $ws1\" :class \"$un$o1$f1\" \"\") \
    (button :onclick \"bspc desktop -f $ws2\" :class \"$un$o2$f2\" \"\") \
    ...)"
}
workspaces
bspc subscribe desktop node_transfer | while read -r _; do workspaces; done
```

```lisp
; Source: rxyhn — github.com/rxyhn/yoru (eww/bar)
; (literal) renders the shell-generated yuck string directly.
(defwidget workspaces []
  (literal :content workspace))
(deflisten workspace "scripts/workspace")
```

### Hover-reveal volume slider

```lisp
; Source: rxyhn — github.com/rxyhn/yoru (eww/bar)
; onhover/onhoverlost toggle the reveal var.
; scale is vertical, flipped=true puts 100 at the top.

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

(defpoll current-volume :interval "1s"
  "amixer -D pulse sget Master | grep 'Left:' | awk -F'[][]' '{ print $2 }' | tr -d '%'")
(defvar volum false)
```

### Hover-reveal brightness slider

```lisp
; Source: rxyhn — github.com/rxyhn/yoru (eww/bar)
; Same pattern as volume. brightnessctl reads amdgpu_bl0 backlight.

(defwidget bright []
  (eventbox :onhover "${eww} update bright=true"
            :onhoverlost "${eww} update bright=false"
    (box :orientation "v" :space-evenly "false" :spacing 2
      (revealer :transition "slideup" :reveal bright :duration "550ms"
        (scale :class "bribar"
               :value current-brightness
               :tooltip "Brightness: ${current-brightness}%"
               :onchange "brightnessctl set {}%"
               :orientation "v" :flipped true :max 101 :min 0))
      (label :class "brightness-icon" :text ""))))

(defpoll current-brightness :interval "1s"
  "brightnessctl -m -d amdgpu_bl0 | awk -F, '{print substr($4, 0, length($4)-1)}' | tr -d '%'")
(defvar bright false)
```

### Hover powermenu

```lisp
; Source: rxyhn — github.com/rxyhn/yoru (eww/bar)
; Power buttons slide up on hover. Icon font: "feather".

(defwidget power []
  (eventbox :onhover "${eww} update power=true"
            :onhoverlost "${eww} update power=false"
    (box :orientation "v" :space-evenly "false" :class "powermenu"
      (revealer :transition "slideup" :reveal power :duration "550ms"
        (box :orientation "v" :space-evenly "false"
          (button :class "button-reb"  :tooltip "Reboot"       :onclick "reboot"        "")
          (button :class "button-quit" :tooltip "Logout"       :onclick "killall bspwm" "")
          (button :class "button-lock" :tooltip "Lock Screen"  :onclick "betterlockscreen -l" "")
          (button :class "button-off"  :tooltip "Shutdown"     :onclick "shutdown now"  "")))
      (button :class "button-off" :tooltip "Shutdown" :onclick "shutdown now" ""))))
(defvar power false)
```

### Battery script

```bash
# Source: rxyhn — github.com/rxyhn/yoru (eww/bar/scripts/battery)
# Returns icon string for given arg: "icon" or "percent".
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
  elif [ "$per" -gt "10" ]; then icon=""; notify-send -u critical "Battery Low" "Connect Charger"
  else icon=""; notify-send -u critical "Battery Low" "Connect Charger"; fi
  echo "$icon"
}
[ "$1" = "icon" ]    && icon    && exit
[ "$1" = "percent" ] && echo $per && exit
```

### Wifi script

```bash
# Source: rxyhn — github.com/rxyhn/yoru (eww/bar/scripts/wifi)
#!/bin/sh
symbol() {
  [ $(cat /sys/class/net/w*/operstate) = down ] && echo  && exit
  echo
}
name() {
  nmcli | grep "^wlp" | sed 's/\ connected\ to\ /Connected to /g' | cut -d ':' -f2
}
[ "$1" = "icon" ] && symbol && exit
[ "$1" = "name" ]  && name   && exit
```

### SCSS (Tokyo Night)

```scss
/* Source: rxyhn — github.com/rxyhn/yoru (eww/bar/eww.scss) */
$background: #1A1B26; $foreground: #A9B1D6;
$black: #24283B; $gray: #565F89; $red: #F7768E;
$green: #73DACA; $yellow: #E0AF68; $blue: #7AA2F7;
$magenta: #BB9AF7; $cyan: #7DCFFF;

.eww_bar { background-color: $background; padding: .3rem; }
.launcher_icon { color: $cyan; font-size: 1.7em; padding: 1rem 0; }

/* Workspace classes encode state: "0"=unoccupied, "01"=occupied, "011"=focused */
.works { font-family: "Font Awesome 6 Pro"; padding: .2rem .7rem; background-color: $black; border-radius: 5px; }
.0  { color: $gray; }   /* unoccupied */
.01 { color: $gray; }   /* occupied */
.011 { color: $foreground; } /* focused */

/* Clock font: Comic Mono */
.time { font-family: Comic Mono; font-weight: bold; font-size: 1.2em;
        background-color: $black; border-radius: 5px; padding: .7rem 0 .5rem; }

/* Vertical scale (volume/brightness bars) */
scale trough { all: unset; background-color: $background; border-radius: 5px; min-height: 80px; min-width: 10px; }
.volbar  trough highlight { background-color: $green;  border-radius: 5px; }
.bribar  trough highlight { background-color: $yellow; border-radius: 5px; }
```

---

## owenrumney

Repo: github.com/owenrumney/eww-bar — Horizontally split dual-bar (top + bottom), modular file structure, hover-reveal sidebar modules, clickbox toggle pattern, network throughput coloring, and external tool integrations (Docker, GitHub).

### Window definition (dual horizontal bars)

```lisp
; Source: owenrumney — github.com/owenrumney/eww-bar
; 90% width, centered. Top and bottom bars via two defwindow.

(defwindow bar
  :monitor 0 :windowtype "dock"
  :geometry (geometry :x "0%" :y "0%" :width "90%" :height "10px" :anchor "top center")
  :reserve (struts :side "top" :distance "4%")
  (bar))

(defwindow bottombar
  :monitor 0 :windowtype "dock"
  :geometry (geometry :x "0%" :y "0%" :width "90%" :height "10px" :anchor "bottom center")
  :reserve (struts :side "bottom" :distance "4%")
  (bottombar))
```

### Modular include structure

```lisp
; Source: owenrumney — github.com/owenrumney/eww-bar (eww.yuck)
; Each concern in its own file. Revealer pattern isolated to revealer.yuck.
(include "variables.yuck")
(include "controls.yuck")
(include "listeners.yuck")
(include "metrics.yuck")
(include "pollers.yuck")
(include "revealer.yuck")
```

### Revealer on hover + clickbox toggle

```lisp
; Source: owenrumney — github.com/owenrumney/eww-bar (revealer.yuck)
; Two patterns: hover-reveal and click-toggle.
; hovered-sign shows child[0] when collapsed, child[1] when expanded.

(defwidget hovered-sign [var]
  (box :space-evenly false
    (revealer :reveal {!var} :duration "100ms" :transition "slideleft" (children :nth 0))
    (revealer :reveal {var}  :duration "100ms" :transition "slideleft" (children :nth 1))))

(defwidget revealer-on-hover [var varname ?class ?duration ?transition]
  (box :class "${class} revealer-on-hover" :orientation "h" :space-evenly false
    (eventbox :onhover "eww update ${varname}=true"
              :onhoverlost "eww update ${varname}=false"
      (box :space-evenly false
        (children :nth 0)
        (revealer :reveal var :transition {transition ?: "slideright"} :duration {duration ?: "500ms"}
          (children :nth 1))
        (box :class "${class}" (children :nth 2))))))

; Click-toggle: button switches var; content has a "Close" button to dismiss.
(defwidget clickbox [var varname ?class ?duration ?transition]
  (box :class "${class} clickbox" :orientation "h" :space-evenly false
    (button :onclick "eww update ${varname}=${ var ? false : true }"
      (children :nth 0))
    (revealer :reveal var :transition {transition ?: "slideleft"} :duration {duration ?: "500ms"}
      (box :class "${class}" :space-evenly false
        (children :nth 1)
        (button :onclick "eww update ${varname}=false" :class "close"
          (label :text "Close"))))))
```

### Volume as hover-reveal module

```lisp
; Source: owenrumney — github.com/owenrumney/eww-bar (eww.yuck)
; Volume widget uses revealer-on-hover with hovered-sign for the expand icon.

(defvar revealVolume false)

(defwidget volume [?class]
  (box :space-evenly false :class "hover-module ${class}"
    (revealer-on-hover :class "hl-on-hover"
                       :var revealVolume :varname "revealVolume"
                       :transition "slideleft"
      (hovered-sign :var revealVolume
        (label :text "")
        (label :text ""))
      (metric :icon "" :class "volume" :value volume
              :onchange "amixer -D pulse sset Master {}%")
      "    ")))
```

### Metric widget (with threshold auto-coloring)

```lisp
; Source: owenrumney — github.com/owenrumney/eww-bar (metrics.yuck)
; Scale class switches to "warning" at >50% and "error" at >75% automatically.

(defwidget metric [icon value ?onchange ?onclick ?class]
  (box :orientation "h" :class "metric" :space-evenly false
    (termbutton :command "${onclick}" :height "1000" :width "1000" :text "${icon}")
    (scale :class {class != "" ? class : value > 50 ? value > 75 ? "error" : "warning" : "normal"}
           :min 0 :max 101
           :active {onchange != ""}
           :value value
           :onchange onchange)))
```

### Network throughput with color classes

```lisp
; Source: owenrumney — github.com/owenrumney/eww-bar (eww.yuck)
; NET_UP/NET_DOWN in bytes/s from EWW_NET. Class changes color at thresholds.

(defwidget network []
  (box :orientation "h" :space-evenly false
    (label :text "${interfaceId}: ${round(EWW_NET[interfaceId].NET_UP / 1000000, 2)}")
    (label :class {round(EWW_NET[interfaceId].NET_UP / 1000000, 2) > 0.1 ?
                   round(EWW_NET[interfaceId].NET_UP / 1000000, 2) > 5 ?
                   "veryuplink" : "uplink" : "noactive"} :text "  ")
    (label :text "${round(EWW_NET[interfaceId].NET_DOWN / 1000000, 2)}")
    (label :class {round(EWW_NET[interfaceId].NET_DOWN / 1000000, 2) > 0.1 ?
                   round(EWW_NET[interfaceId].NET_DOWN / 1000000, 2) > 10 ?
                   "verydownlink" : "downlink" : "noactive"} :text "  ")))

(defpoll interfaceId :interval "60s" "route | grep default | head -n1 | awk '{print $8}'")
```

### Scripts

```bash
# Source: owenrumney — github.com/owenrumney/eww-bar (scripts/getvol)
#!/bin/sh
amixer -D pulse sget Master | grep 'Left:' | awk -F'[][]' '{ print $2 }' | tr -d '%' | head -1
```

```bash
# Source: owenrumney — github.com/owenrumney/eww-bar (scripts/getram)
#!/bin/sh
printf "%.0f\n" $(free -m | grep Mem | awk '{print ($3/$2)*100}')
```

### SCSS variables (Gruvbox dark variant)

```scss
/* Source: owenrumney — github.com/owenrumney/eww-bar (gruvbox.scss) */
$background: #0d1117; $foreground: #b0b4bc; $background-alt: #232320;
$green: #98971a; $lightgreen: #b8bb26; $yellow: #d79921; $lightyellow: #fabd2d;
$red: #cc241d; $lightred: #fb4934; $blue: #0db7ed; $lightblue: #83a598;
$magenta: #b16286; $lightmagenta: #d3869b; $cyan: #689d6a; $lightcyan: #8ec07c;
$gray: #928374; $lightgray: #a89984;
$warning: $yellow; $error: $red; $urgent: $red;
```

---

## isparsh

Repo: github.com/isparsh/gross — Desktop overlay setup: multiple small floating windows composited together to form a sidebar dashboard. Separated into windows/widgets/variables files. Nord color scheme, inline SVG app icons via `:style "background-image: url(...);"`.

### Window definition (floating overlay panels)

```lisp
; Source: isparsh — github.com/isparsh/gross (eww_windows.yuck)
; Multiple small :windowtype "dock" windows positioned to form a dashboard.
; :wm-ignore true keeps them off the taskbar and outside WM layout.
; A "bg" window provides a unified background container.

(defwindow bg
  :class "bg" :wm-ignore true :monitor 0 :windowtype "dock"
  :geometry (geometry :x "5px" :y "40px" :width "380px" :height "800px" :anchor "top left")
  (bg))

(defwindow fetch
  :wm-ignore true :monitor 0 :windowtype "dock"
  :geometry (geometry :x "20px" :y "65px" :width "170px" :height "200px" :anchor "top left")
  (uinfo))

(defwindow sys
  :class "cpu-win" :wm-ignore true :monitor 0 :windowtype "dock"
  :geometry (geometry :x "200px" :y "275px" :width "170px" :height "48px" :anchor "top left")
  (sys))

(defwindow calendar
  :wm-ignore true :monitor 0 :windowtype "dock"
  :geometry (geometry :x "20px" :y "415px" :width "240px" :height "160px" :anchor "top left")
  (cal))

(defwindow notes
  :wm-ignore true :monitor 0 :windowtype "dock"
  :geometry (geometry :x "20px" :y "630px" :width "350px" :height "100px" :anchor "top left")
  (notes))
```

### Inline fetch / user info widget

```lisp
; Source: isparsh — github.com/isparsh/gross (eww_widgets.yuck)
; Fetch widget uses two vertical boxes side-by-side: icons on left, values on right.
; Inline :style for per-label color avoids needing unique CSS classes.

(defwidget uinfo []
  (box :class "uinfo" :orientation "v" :space-evenly false :halign "center" :valign "center"
    (label :style "color: #5e81ac;" :text "sparsh@asus-sea" :halign "center" :limit-width 25)
    (box :orientation "h" :space-evenly "false" :spacing 10
      (box :orientation "v" :class "fetch" :spacing 2  ; icon column
        (label :style "color: #b48ead;" :halign "end" :text "")
        (label :style "color: #ebcb8b;" :halign "end" :text "缾")
        (label :style "color: #80a0c0;" :halign "end" :text "")
        (label :style "color: #b48ead;" :halign "end" :text ""))
      (box :orientation "v" :class "fetch"             ; value column
        (label :style "color: #b48ead;" :halign "start" :text ": ${distro}")
        (label :style "color: #ebcb8b;" :halign "start" :text ": ${wm}")
        (label :style "color: #80a0c0;" :halign "start" :text ": ${shell}")
        (label :style "color: #b48ead;" :halign "start" :text ": ${uptime}")))))
```

### Metric widget (reusable bar)

```lisp
; Source: isparsh — github.com/isparsh/gross (eww_widgets.yuck)
; Simple metric: label on left, scale on right. onchange="" disables interaction.

(defwidget metric [label value onchange]
  (box :orientation "h" :class "metric" :space-evenly false
    (box :class "label" label)
    (scale :min 0 :max 101 :active {onchange != ""} :value value :onchange onchange)))

; System widget using EWW magic variables
(defwidget sys []
  (box :class "cpu" :orientation "v" :space-evenly false
    (metric :label "﬙" :value {EWW_CPU.avg} :onchange "")
    (metric :label "" :value {EWW_RAM.used_mem_perc} :onchange "")
    (metric :label "龍" :value {(EWW_NET.wlan0.NET_UP)/100} :onchange "")
    (metric :label "" :value {(EWW_DISK["/"].free / EWW_DISK["/"].total) * 100} :onchange "")))
```

### Eww-powered search input

```lisp
; Source: isparsh — github.com/isparsh/gross (eww_widgets.yuck)
; :input widget feeds keystrokes to a search script.
; eventbox :onhoverlost closes the window automatically.

(defwidget searchapps []
  (eventbox :onhoverlost "eww close searchapps"
    (box :orientation "v" :space-evenly false :class "search-win"
      (box :orientation "h" :space-evenly false :class "searchapps-bar"
        (label :class "search-label" :text "")
        (input :class "search-bar" :onchange "~/.config/eww/scripts/search.sh {}"))
      (literal :class "app-container" :content search_listen))))
```

### Variables file (defpoll patterns)

```lisp
; Source: isparsh — github.com/isparsh/gross (eww_variables.yuck)
; fortune for random quotes. Distro/WM/shell via shell commands.
; NOTES watches a file live for a TODO widget.

(defpoll quote_text :interval "3600s" `fortune -n 90 -s`)
(defpoll TODAY      :interval "1s"    `date +%m/%d/%y`)
(defpoll distro     :interval "12h"   "awk '/^ID=/' /etc/*-release | awk -F'=' '{ print tolower($2) }'")
(defpoll wm         :interval "12h"   "wmctrl -m | grep \"Name:\" | awk '{print $2}'")
(defpoll shell      :interval "5m"    "echo $SHELL | awk -F'/' '{print $NF}'")
(defpoll uptime     :interval "30s"   "uptime -p | sed -e 's/up //;s/ hours,/h/;s/ minutes/m/'")
(defpoll packages   :interval "5m"    "pacman -Q | wc -l")
(defpoll NOTES      :interval "1s"    "cat -s ~/Documents/notes.txt")
```

### SCSS (Nord, gradient metric bars)

```scss
/* Source: isparsh — github.com/isparsh/gross (eww.scss) */
.bg { background-color: #1E222A; border: 2px solid #80A0C0; }
window { background: #2e3440; border: 3px solid #80A0C0; }

/* Gradient scale bar — blue gradient fill */
.metric scale trough highlight {
  all: unset;
  background: linear-gradient(90deg, #88c0d0 0%, #81a1c1 50%, #5E81ac 100%);
}
.metric scale trough {
  all: unset; background-color: #545454;
  min-height: 10px; min-width: 100px; margin-left: 10px;
}
.metric .label { color: #B48EAD; font-size: 1.5rem; }
.quote-text { font-style: italic; font-weight: 600; color: #BF616A; }
```

---

## nycta

Repo: github.com/Nycta-b424b3c7/eww_activate-linux — Single-purpose: an "Activate Linux" parody watermark widget. Minimal, transparent, background-stacked.

### Window definition and widget

```lisp
; Source: nycta — github.com/Nycta-b424b3c7/eww_activate-linux
; :stacking "fg" keeps it visible but non-interactive.
; :anchor "bottom right" with negative offsets for screen-edge positioning.

(defwidget activate-linux []
  (box :orientation "v" :halign "start" :valign "start"
    (label :xalign 0 :markup "<span font_size=\"large\">Activate Linux</span>")
    (label :xalign 0 :text "Go to Settings to activate Linux")))

(defwindow activate-linux
  :monitor 0
  :stacking "fg"
  :geometry (geometry :x "8px" :y "32px" :width "250px" :anchor "bottom right")
  (activate-linux))
```

### SCSS

```scss
/* Source: nycta — github.com/Nycta-b424b3c7/eww_activate-linux */
.activate-linux { color: rgba(250, 250, 250, 0.5); }
```

---

## saimoomedits

Repo: github.com/saimoomedits/eww-widgets — Floating top bar with circular-progress battery/memory indicators, hover-reveal sliders, inline album art via CSS background-image, and a multi-window left sidebar dashboard variant.

### Window definition (floating top bar)

```lisp
; Source: saimoomedits — github.com/saimoomedits/eww-widgets (bar)
; Bar floats 9px from top, 98% wide, 30px tall. Not exclusive (no reserved space).

(defwindow bar
  :geometry (geometry :x "0%" :y "9px" :width "98%" :height "30px" :anchor "top center")
  :stacking "fg" :windowtype "dock"
  (bar_1))

(defwidget bar_1 []
  (box :class "bar_class" :orientation "h"
    (right)    ; workspaces
    (center)   ; music
    (left)))   ; controls (brightness, volume, wifi, battery, memory, clock)
```

### Circular-progress battery and memory

```lisp
; Source: saimoomedits — github.com/saimoomedits/eww-widgets (bar)
; circular-progress is eww's built-in donut widget. :thickness controls ring width.
; Button inside acts as the icon/label in the donut center.

(defwidget bat []
  (box :class "bat_module"
    (circular-progress :value battery :class "batbar" :thickness 4
      (button :class "iconbat" :tooltip "battery on ${battery}%"
              :onclick "$HOME/.config/eww/bar/scripts/pop system"
              ""))))

(defwidget mem []
  (box :class "mem_module"
    (circular-progress :value memory :class "membar" :thickness 4
      (button :class "iconmem" :tooltip "using ${memory}% ram"
              :onclick "$HOME/.config/eww/bar/scripts/pop system"
              ""))))
```

### Music widget with inline album art

```lisp
; Source: saimoomedits — github.com/saimoomedits/eww-widgets (bar)
; Album art via CSS background-image set from a polled file path.
; Controls revealed on hover via slideright revealer.

(defwidget music []
  (eventbox :onhover "${eww} update music_reveal=true"
            :onhoverlost "${eww} update music_reveal=false"
    (box :class "module-2" :orientation "h" :space-evenly "false"
      (box :class "song_cover_art" :style "background-image: url('${cover_art}');")
      (button :class "song" :onclick "~/.config/eww/bar/scripts/pop music" song)
      (revealer :transition "slideright" :reveal music_reveal :duration "350ms"
        (box :orientation "h"
          (button :class "song_btn_prev" :onclick "...music_info --prev" "")
          (button :class "song_btn_play" :onclick "...music_info --toggle" song_status)
          (button :class "song_btn_next" :onclick "...music_info --next" ""))))))
```

### Hover-reveal volume and brightness

```lisp
; Source: saimoomedits — github.com/saimoomedits/eww-widgets (bar)
; eventbox triggers eww update; revealer slides the scale in from left.

(defwidget volume []
  (eventbox :onhover "${eww} update vol_reveal=true"
            :onhoverlost "${eww} update vol_reveal=false"
    (box :class "module-2" :space-evenly "false" :orientation "h" :spacing "3"
      (button :onclick "scripts/pop audio" :class "volume_icon" "")
      (revealer :transition "slideleft" :reveal vol_reveal :duration "350ms"
        (scale :class "volbar" :value volume_percent :orientation "h"
               :tooltip "${volume_percent}%" :max 100 :min 0
               :onchange "amixer -D pulse sset Master {}%")))))
```

### Leftbar: multi-window sidebar

```lisp
; Source: saimoomedits — github.com/saimoomedits/eww-widgets (leftbar)
; Sidebar uses many independent windows: time, profile pic, music, sys stats, audio.
; Each is positioned independently by absolute coordinates.

(defwindow pfp
  :geometry (geometry :anchor "left top" :width "190px" :height "23" :x "0px" :y "65px")
  (profile))

(defwindow time
  :geometry (geometry :anchor "left top" :width "260px" :height "100" :x "220px" :y "100px")
  (time))

(defwindow song :stacking "fg" :focusable "false" :screen 1
  :geometry (geometry :width "260px" :height "140px" :x "0px" :y "300px")
  (music))

(defwindow sys_usg
  :geometry (geometry :width "420px" :height "160px" :x "10px" :y "510px")
  (system))
```

### SCSS (bar — rounded dark theme)

```scss
/* Source: saimoomedits — github.com/saimoomedits/eww-widgets (bar/eww.scss) */
.bar_class { background-color: #0f0f17; border-radius: 16px; }

/* Circular progress colors */
.membar { color: #e0b089; background-color: #38384d; border-radius: 10px; }
.batbar { color: #afbea2; background-color: #38384d; border-radius: 10px; }

/* Gradient scale bars */
.volbar trough highlight {
  background-image: linear-gradient(to right, #afcee0 30%, #a1bdce 50%, #77a5bf 100%);
  border-radius: 10px;
}
.brightbar trough highlight {
  background-image: linear-gradient(to right, #e4c9af 30%, #f2cdcd 50%, #e0b089 100%);
  border-radius: 10px;
}

/* Album art circle */
.song_cover_art {
  background-size: cover; background-position: center;
  min-height: 24px; min-width: 24px; margin: 10px; border-radius: 100px;
}

/* Workspace dot classes (bspwm style, used by rxyhn and saimoomedits) */
.0   { color: #3e424f; }  /* unoccupied */
.01  { color: #bfc9db; }  /* occupied   */
.011 { color: #a1bdce; }  /* focused    */
```

---

## axarva

Repo: github.com/Axarva/dotfiles-2.0 — Full desktop overlay sidebar for XMonad. Signature: many independent floating windows composited over the desktop, Spotify/playerctl integration with live album art, weather widget reading from /tmp files, home-dir quicklaunch, VPN indicator with color-coded hex, do-not-disturb toggle, and `Museo Sans` font.

### Window definition (multi-window overlay sidebar)

```lisp
; Source: axarva — github.com/Axarva/dotfiles-2.0 (eww-1920)
; Each widget is its own window. No single "bar" — instead a composition.
; Absolute pixel positions tuned for 1920x1080.

(defwindow time_side
  :geometry (geometry :x "10px" :y "130px" :width "300px" :height "135px")
  (time_side))

(defwindow player_side
  :geometry (geometry :x "10px" :y "270px" :width "300px" :height "122px")
  (player_side))

(defwindow sliders_side
  :geometry (geometry :x "10px" :y "397px" :width "300px" :height "205px")
  (sliders_side))

(defwindow sys_side
  :geometry (geometry :x "10px" :y "607px" :width "300px" :height "153px")
  (sys_side))

(defwindow weather
  :geometry (geometry :x "740px" :y "220px" :width "410px" :height "400px")
  (weather))

(defwindow profile
  :geometry (geometry :x "390px" :y "220px" :width "340px" :height "520px")
  (profile))
```

### Music player widget

```lisp
; Source: axarva — github.com/Axarva/dotfiles-2.0 (eww-1920)
; cover is a path polled from a script. art and title also from scripts.
; musicstat polls ~/bin/spotifystatus for play/pause icon.

(defwidget player []
  (box :orientation "v" :space-evenly "false"
    (box :class "musicart" :style "background-image: url('${cover}');" {art})
    (box :class "musictitle" "${music3}${title}")
    (box :class "musicartist" "${music2}${artist}")
    (box :orientation "h" :halign "center" :class "musicbtn" :space-evenly "false"
      (button :onclick "playerctl previous" "")
      (button :onclick "playerctl play-pause" {musicstat})
      (button :onclick "playerctl next" ""))
    (box :orientation "h" :class "music-slider" :halign "center"
      (scale :min 0 :max 101 :value {musicpos} :active "false"))))

(defpoll cover    :interval "2s"   "~/.config/eww/scripts/echoart")
(defpoll title    :interval "1s"   "~/.config/eww/scripts/echotitle")
(defpoll artist   :interval "1s"   "cat /tmp/xmonad/spotify/artist")
(defpoll musicpos :interval "16ms" "~/.config/eww/scripts/getposition")
(defpoll musicstat :interval "1s"  "~/bin/spotifystatus")
```

### Sliders sidebar (volume + brightness + ram + battery)

```lisp
; Source: axarva — github.com/Axarva/dotfiles-2.0 (eww-1920)
; All four sliders in one vertical box. RAM and battery are read-only (:active "false").

(defwidget sliders_side []
  (box :orientation "v" :space-evenly "false" :class "sliders-side"
    (box :orientation "h" :class "slider-vol-side" :space-evenly "false"
      (box :class "label-vol-side" "")
      (scale :min 0 :max 101 :value {volume} :onchange "amixer -D pulse sset Master {}%"))
    (box :orientation "h" :class "slider-bright-side" :space-evenly "false"
      (box :class "label-bright-side" "")
      (scale :min 0 :max 101 :value {bright} :onchange "brightnessctl s {}%"))
    (box :orientation "h" :class "slider-ram-side" :space-evenly "false"
      (box :class "label-ram-side" "")
      (scale :min 0 :active "false" :max 101 :value {ram-used}))
    (box :orientation "h" :class "slider-battery-side" :space-evenly "false"
      (box :class "label-battery-side" {bat-icon})
      (scale :min 0 :active "false" :max 101 :value {battery-remaining}))))

(defpoll volume            :interval "16ms" "~/.config/eww/scripts/getvol")
(defpoll bright            :interval "16ms" "~/.config/eww/scripts/getbri")
(defpoll ram-used          :interval "1s"   "~/.config/eww/scripts/getram")
(defpoll battery-remaining :interval "5s"   "cat /sys/class/power_supply/BAT0/capacity")
```

### Weather widget (reads from /tmp files)

```lisp
; Source: axarva — github.com/Axarva/dotfiles-2.0 (eww-1920)
; Weather data lives in /tmp/xmonad/weather/ — written by an XMonad hook.
; weather-hex is a hex color string that drives the icon color via inline style.

(defwidget weather []
  (box :orientation "v" :space-evenly "false"
    (box :orientation "h" :space-evenly "false"
      (box :class "weather-icon" :style "color: ${weather-hex}" {weather-icon})
      (box :class "temperature" "${temperature}  "))
    (box :orientation "v" :space-evenly "false"
      (box :class "weather-stat"       {weather-stat})
      (box :class "weather-quote-head" {weather-quote})
      (box :class "weather-quote-tail" {weather-quote2}))))

(defpoll weather-icon :interval "20m" "cat /tmp/xmonad/weather/weather-icon")
(defpoll temperature  :interval "20m" "cat /tmp/xmonad/weather/weather-degree")
(defpoll weather-hex  :interval "20m" "cat /tmp/xmonad/weather/weather-hex")
```

### VPN + do-not-disturb toggles

```lisp
; Source: axarva — github.com/Axarva/dotfiles-2.0 (eww-1920)
; Color-coded VPN state: hex read from file, icon includes status text.
; Do-not-disturb calls a shell script; color read from file.

(defwidget vpn []
  (box :orientation "v" :space-evenly "true"
    (button :class "vpn-icon"
            :onclick "~/.config/eww/scripts/vpntoggle"
            :style "color: ${vpn-hex}"
            "嬨${getvpnstat}")))

(defwidget donotdisturb []
  (box :orientation "v" :halign "center"
    (button :class "disturb-icon"
            :onclick "~/bin/do_not_disturb.sh"
            :style "color: ${disturb-hex}"
            "⏾")))

(defpoll vpn-hex     :interval "1s"  "cat /tmp/xmonad/vpnstat-hex")
(defpoll getvpnstat  :interval "10s" "~/.config/eww/scripts/getvpnstat")
(defpoll disturb-hex :interval "1s"  "cat /tmp/xmonad/donotdisturb/color")
```

### SCSS (dark, Museo Sans font)

```scss
/* Source: axarva — github.com/Axarva/dotfiles-2.0 (eww-1920/eww.scss) */
* { all: unset; }
window { background-color: #121212; color: #ffd5cd; font-family: Museo Sans; }
button { all: unset; background-color: #121212; padding: 10px; }

.musicart { background-size: 260px; min-height: 260px; min-width: 260px; margin: 20px; border-radius: 10px; }
.musictitle { font-size: 20px; font-weight: bold; }
.musicbtn button:hover { color: #D35D6E; }
.music-slider scale trough { all: unset; background-color: #4e4e4e; border-radius: 50px; min-height: 5px; min-width: 240px; }
.music-slider scale trough highlight { all: unset; background-color: #ffd5cd; border-radius: 10px; }

/* Time: minute in accent color, hour in base */
.time-side { margin: 20px 65px 0px 65px; font-size: 60px; }
.minute-side { color: #D35D6E; }
.day-side { font-size: 16px; font-weight: bold; color: #90c861; }

/* Per-slider trough highlight colors */
.slider-vol-side    scale trough highlight { all: unset; background-color: #c47eb7; border-radius: 10px; }
.slider-bright-side scale trough highlight { all: unset; background-color: #84afdb; border-radius: 10px; }
.slider-ram-side    scale trough highlight { all: unset; background-color: #D35D6E; border-radius: 10px; }
.slider-battery-side scale trough highlight { all: unset; background-color: #90c861; border-radius: 10px; }

/* Profile pic: circular, large margin for centering */
.pfp { background-size: 200px; min-height: 200px; min-width: 200px; border-radius: 500px; }
.pfptxt { color: #D35D6E; font-size: 36px; }
```

---

## adi1090x

Repo: github.com/adi1090x/widgets — Two styles: a full-screen dashboard (many windows over a wallpaper) and a minimal top dockbar (arin). Nordic/Nord-inspired palette, `genwin` CSS class as a reusable card background, SVG app icons via inline style, and a consistent multi-window absolute-position layout.

### Window definition (full-screen dashboard)

```lisp
; Source: adi1090x — github.com/adi1090x/widgets (eww/dashboard)
; Each widget is a separate :stacking "fg" window with absolute geometry.
; :focusable "false" :screen 1 is the standard combination.

(defwindow background :stacking "fg" :focusable "false" :screen 1
  :geometry (geometry :x 0 :y 0 :width "1920px" :height "1080px")
  (bg))

(defwindow profile :stacking "fg" :focusable "false" :screen 1
  :geometry (geometry :x 150 :y 150 :width 350 :height 440)
  (user))

(defwindow system :stacking "fg" :focusable "false" :screen 1
  :geometry (geometry :x 150 :y 605 :width 350 :height 325)
  (system))

(defwindow music :stacking "fg" :focusable "false" :screen 1
  :geometry (geometry :x 515 :y 490 :width 610 :height 280)
  (music))

(defwindow weather :stacking "fg" :focusable "false" :screen 1
  :geometry (geometry :x 880 :y 150 :width 550 :height 325)
  (weather))
```

### genwin card pattern

```lisp
; Source: adi1090x — github.com/adi1090x/widgets (eww/dashboard)
; All widget boxes use :class "genwin" for a consistent card background.
; Avoids repeating background/radius CSS on every widget.

(defwidget user []
  (box :class "genwin" :orientation "v" :spacing 35 :space-evenly "false"
    (box :style "background-image: url('${IMAGE}');" :class "face" :halign "center")
    (label :class "fullname" :halign "center" :text NAME)
    (label :class "username" :halign "center" :text UNAME)))

(defwidget clock []
  (box :class "genwin" :orientation "h" :spacing 50 :space-evenly false
    (box :orientation "h"
      (label :class "time_hour" :valign "start" :text HOUR)
      (label :class "time_min"  :valign "end"   :text MIN))
    (box :orientation "v"
      (label :class "time_mer" :valign "start" :halign "end" :text MER)
      (label :class "time_day" :valign "end"   :halign "end" :text DAY))))
```

### System bars (horizontal scale per metric)

```lisp
; Source: adi1090x — github.com/adi1090x/widgets (eww/dashboard)
; Four icon+scale rows, each with a different highlight color in SCSS.

(defwidget system []
  (box :class "genwin" :vexpand "false" :hexpand "false"
    (box :orientation "v" :spacing 35 :halign "center" :valign "center" :space-evenly "false"
      (box :class "cpu_bar" :orientation "h" :spacing 20 :space-evenly "false"
        (label :class "iconcpu" :text "")
        (scale :min 0 :max 100 :value CPU_USAGE :active "false"))
      (box :class "mem_bar" :orientation "h" :spacing 20 :space-evenly "false"
        (label :class "iconmem" :text "")
        (scale :min 0 :max 100 :value MEM_USAGE :active "false"))
      (box :class "bright_bar" :orientation "h" :spacing 20 :space-evenly "false"
        (label :class "iconbright" :text "")
        (scale :min 0 :max 100 :value BLIGHT :active "false"))
      (box :class "bat_bar" :orientation "h" :spacing 20 :space-evenly "false"
        (label :class "iconbat" :text "")
        (scale :min 0 :max 100 :value BATTERY :active "false")))))
```

### SVG icon app launcher

```lisp
; Source: adi1090x — github.com/adi1090x/widgets (eww/dashboard)
; App buttons use SVG files via :style "background-image: url(...);"
; App class controls size via min-height/min-width in SCSS.

(defwidget apps []
  (box :class "genwin" :orientation "v" :space-evenly "false"
    (box :class "appbox" :orientation "h" :space-evenly "false"
      (button :style "background-image: url('images/icons/firefox.svg');"
              :class "app_fox" :onclick "scripts/open_apps --ff")
      (button :style "background-image: url('images/icons/discord.svg');"
              :class "app_discord" :onclick "scripts/open_apps --dc")
      (button :style "background-image: url('images/icons/terminal.svg');"
              :class "app_terminal" :onclick "scripts/open_apps --tr"))))
```

### Arin: minimal dockbar (multi-widget top bar)

```lisp
; Source: adi1090x — github.com/adi1090x/widgets (eww/arin)
; Instead of one bar window, arin uses separate windows side by side at the top.
; Each module is independently reserving space with struts.

(defwindow search
  :monitor 0
  :geometry (geometry :x "20px" :y "20px" :width "180px" :height "60px" :anchor "top left")
  :stacking "fg" :reserve (struts :distance "80px" :side "top") :windowtype "dock"
  (search))

(defwindow apps
  :monitor 0
  :geometry (geometry :x "230px" :y "20px" :width "400px" :height "60px" :anchor "top left")
  :stacking "fg" :reserve (struts :distance "80px" :side "top") :windowtype "dock"
  (apps))

(defwindow music
  :monitor 0
  :geometry (geometry :x "990px" :y "20px" :width "400px" :height "60px" :anchor "top left")
  :stacking "fg" :reserve (struts :distance "80px" :side "top") :windowtype "dock"
  (music))

(defwindow system
  :monitor 0
  :geometry (geometry :x "1420px" :y "20px" :width "480px" :height "60px" :anchor "top left")
  :stacking "fg" :reserve (struts :distance "80px" :side "top") :windowtype "dock"
  (system))
```

### SCSS (dashboard — Nord palette, genwin card)

```scss
/* Source: adi1090x — github.com/adi1090x/widgets (eww/dashboard/eww.scss) */
* { all: unset; font-family: feather; font-family: Iosevka; }

.genwin { background-color: #2E3440; border-radius: 16px; }

/* Profile */
.face { background-size: 200px; min-height: 200px; min-width: 200px; border-radius: 100%; }
.fullname { color: #D46389; font-size: 30px; font-weight: bold; }
.username { color: #8FBCBB; font-size: 22px; font-weight: bold; }

/* System bars — each metric icon and trough highlight has a unique color */
.iconcpu    { color: #BF616A; } .iconcpu    { font-size: 32px; }
.iconmem    { color: #A3BE8C; }
.iconbright { color: #EBCB8B; }
.iconbat    { color: #88C0D0; }
.cpu_bar    scale trough highlight { background-color: #BF616A; border-radius: 16px; }
.mem_bar    scale trough highlight { background-color: #A3BE8C; border-radius: 16px; }
.bright_bar scale trough highlight { background-color: #EBCB8B; border-radius: 16px; }
.bat_bar    scale trough highlight { background-color: #88C0D0; border-radius: 16px; }
.cpu_bar, .mem_bar, .bright_bar, .bat_bar, scale trough {
  all: unset; background-color: #3A404C; border-radius: 16px;
  min-height: 28px; min-width: 240px;
}

/* Clock */
.time_hour, .time_min { color: #81A1C1; font-size: 70px; font-weight: bold; }
.time_mer  { color: #A3BE8C; font-size: 40px; font-weight: bold; }
.time_day  { color: #EBCB8B; font-size: 30px; }

/* Music */
.album_art { background-size: 240px; min-height: 240px; min-width: 240px; border-radius: 14px; }
.song   { color: #8FBCBB; font-size: 24px; font-weight: bold; }
.artist { color: #EBCB8B; font-size: 16px; }
.btn_play { color: #A3BE8C; font-size: 48px; font-weight: bold; }
.btn_prev, .btn_next { color: #EBCB8B; font-size: 32px; }
.music_bar scale trough highlight { background-color: #B48EAD; border-radius: 8px; }
.music_bar scale trough { background-color: #3A404C; border-radius: 8px; min-height: 20px; min-width: 310px; }

/* Power buttons */
.btn_logout { color: #BF616A; } .btn_sleep  { color: #A3BE8C; }
.btn_reboot { color: #EBCB8B; } .btn_poweroff { color: #88C0D0; }

/* App icons via SVG background-image */
.app_fox, .app_telegram, .app_discord, .app_terminal,
.app_files, .app_geany, .app_code, .app_gimp, .app_vbox {
  background-repeat: no-repeat; background-size: 64px;
  min-height: 64px; min-width: 64px; margin: 8px;
}

/* Social link buttons */
.github  { background-color: #24292E; border-radius: 16px; }
.reddit  { background-color: #E46231; border-radius: 16px; }
.twitter { background-color: #61AAD6; border-radius: 16px; }
.youtube { background-color: #DF584E; border-radius: 16px; }
.iconweb { color: #FFFFFF; font-size: 70px; }
```
