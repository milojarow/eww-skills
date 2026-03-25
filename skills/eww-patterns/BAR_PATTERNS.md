# BAR_PATTERNS.md — Status Bar Patterns

Four complete bar patterns with full yuck and scss, plus a "Community Bar Implementations" section with real code from public dotfiles.

---

## Pattern 1: Minimal Horizontal Bar

The canonical eww bar from the official example, extended with Wayland `exclusive` support. Three-zone layout via `centerbox`: workspaces left, clock center, system info right.

```
~/.config/eww/
├── eww.yuck
├── eww.scss
└── scripts/
    └── getvol
```

**eww.yuck**
```yuck
; Top-level bar widget using centerbox for 3 zones
(defwidget bar []
  (centerbox :orientation "h"
    (workspaces)
    (clock)
    (sidestuff)))

; Left zone: workspace buttons
(defwidget workspaces []
  (box :class "workspaces"
       :orientation "h"
       :space-evenly true
       :halign "start"
       :spacing 6
    (button :onclick "swaymsg workspace 1" "1")
    (button :onclick "swaymsg workspace 2" "2")
    (button :onclick "swaymsg workspace 3" "3")
    (button :onclick "swaymsg workspace 4" "4")
    (button :onclick "swaymsg workspace 5" "5")))

; Center zone: time
(defwidget clock []
  (label :class "clock"
         :text {formattime(EWW_TIME, "%H:%M")}))

; Right zone: metrics + systray + time label
(defwidget sidestuff []
  (box :class "sidestuff"
       :orientation "h"
       :space-evenly false
       :halign "end"
       :spacing 12
    (metric :label ""
            :value volume
            :onchange "amixer -D pulse sset Master {}%")
    (metric :label ""
            :value {round(EWW_RAM.used_mem_perc, 0)}
            :onchange "")
    (metric :label ""
            :value {round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}
            :onchange "")
    (systray :spacing 6 :icon-size 16)
    time))

; Reusable metric: icon + scale slider
(defwidget metric [label value onchange]
  (box :orientation "h"
       :class "metric"
       :space-evenly false
    (box :class "metric-label" label)
    (scale :min 0
           :max 101
           :active {onchange != ""}
           :value value
           :onchange onchange)))

; Data sources
(defpoll volume :interval "1s"
  "scripts/getvol")

(defpoll time :interval "10s"
  "date '+%H:%M %b %d, %Y'")

; Window definition — Wayland
(defwindow bar
  :monitor 0
  :geometry (geometry :x "0%"
                      :y "0%"
                      :width "100%"
                      :height "30px"
                      :anchor "top center")
  :stacking "fg"
  :exclusive true        ; Wayland: reserve space automatically
  (bar))

; Window definition — X11 alternative
; (defwindow bar
;   :monitor 0
;   :windowtype "dock"
;   :geometry (geometry :x "0%" :y "0%"
;                       :width "90%" :height "30px"
;                       :anchor "top center")
;   :reserve (struts :side "top" :distance "4%")
;   (bar))
```

**scripts/getvol**
```bash
#!/bin/bash
# Output current master volume as an integer 0-100
amixer -D pulse get Master | grep -oP '\d+(?=%)' | head -1
```

**eww.scss**
```scss
* { all: unset; }

.bar {
  background-color: #1e1e2e;
  color: #cdd6f4;
  padding: 0 12px;
  font-family: "JetBrainsMono Nerd Font", monospace;
  font-size: 13px;
}

.workspaces button {
  padding: 4px 8px;
  border-radius: 4px;
  color: #6c7086;

  &:hover { color: #cba6f7; }
}

.clock {
  font-weight: bold;
  color: #cba6f7;
}

.metric {
  padding: 0 4px;

  scale trough {
    all: unset;
    background-color: #313244;
    border-radius: 10px;
    min-height: 4px;
    min-width: 50px;
    margin: 0 8px;
  }

  scale trough highlight {
    all: unset;
    background-color: #cba6f7;
    border-radius: 10px;
  }
}
```

---

## Pattern 2: Vertical Sidebar Bar

A 40px-wide vertical bar anchored to the left edge. Uses `centerbox :orientation "v"` for top/middle/bottom zones. Based on elkowar/dots-of-war — see Community section below for the real source.

```yuck
; Vertical bar — top zone has systray, middle has clock, bottom has metrics
(defwidget bar [screen]
  (centerbox :orientation "v"
    (box :class "segment-top" :valign "start"
      (top :screen screen))
    (box :valign "center" :class "middle"
      (middle :screen screen))
    (box :valign "end" :class "segment-bottom"
      (bottom :screen screen))))

; Top section: system tray (vertical orientation)
(defwidget top [screen]
  (box :orientation "v" :space-evenly false :spacing 8
    (systray :orientation "v" :icon-size 15 :spacing 10)))

; Middle section: time stacked vertically
(defwidget middle []
  (box :orientation "v" :class "time" :space-evenly false
    hour min sec))

; Bottom section: volume + disk + ram + cpu + date
(defwidget bottom [screen]
  (box :orientation "v"
       :valign "end"
       :space-evenly true
       :spacing 5
    (volume)
    (metric :icon "" :font-size 0.8
      "${round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}%")
    (metric :icon "" "${round(EWW_RAM.used_mem_perc, 0)}%")
    (metric :icon "" "${round(EWW_CPU.avg, 0)}%")
    (box :class "metric" (date))))

; Reusable vertical metric: icon above value
(defwidget metric [icon ?font-size]
  (box :class "metric" :orientation "v"
    (label :class "metric-icon"
           :style {font-size != "" ? "font-size: ${font-size}rem;" : ""}
           :text icon)
    (children)))

; Volume with scroll-to-adjust
(defwidget volume []
  (box :class "volume-metric"
       :orientation "v"
       :space-evenly false
       :valign "fill"
       :vexpand false
    (scale :orientation "h"
           :min 0 :max 100
           :onchange "pamixer --set-volume $(echo {} | sed 's/\\..*//g')"
           :value volume)
    (eventbox :onscroll "if [ '{}' == 'up' ]; then pamixer -i 5; else pamixer -d 5; fi"
              :vexpand true :valign "fill"
      (button :onclick "pavucontrol &"
        (label :text audio_sink)))))

; Date stacked vertically
(defwidget date []
  (box :orientation "v" :halign "center"
    day_word day month year))

; Data sources
(deflisten volume :initial "0" "./audio.sh volume")

(defpoll hour     :interval "1s"  "date +%H")
(defpoll min      :interval "1s"  "date +%M")
(defpoll sec      :interval "1s"  "date +%S")
(defpoll day_word :interval "10m" "date +%a | tr [:upper:] [:lower:]")
(defpoll day      :interval "10m" "date +%d")
(defpoll month    :interval "1h"  "date +%m")
(defpoll year     :interval "1h"  "date +%y")

(defvar audio_sink "")

; Window — 40px wide, full height, anchored left (Wayland)
(defwindow bar_1
  :monitor '["<primary>", "DisplayPort-0", "PHL 345B1C"]'
  :stacking "fg"
  :geometry (geometry :x 0 :y 0 :width "40px" :height "100%" :anchor "center left")
  :reserve (struts :distance "40px" :side "left")
  :exclusive true
  (bar :screen 1))
```

**eww.scss**
```scss
* { all: unset; }

window {
  background: #282828;
  color: #ebdbb2;
  font-size: 14px;
  font-family: "Terminus (TTF)";

  & > * { margin: 3px; }
}

.segment-top  { margin-top: 10px; }
.segment-bottom { margin-bottom: 10px; }

.metric {
  background-color: #1d2021;
  padding: 5px 2px;
}
.metric-icon {
  font-family: "Font Awesome 6 Free";
  font-size: 0.7em;
}

.time {
  padding-top: 7px;
  padding-bottom: 7px;
  color: #a89984;
}

.volume-metric {
  background-color: #1d2021;

  scale trough highlight {
    all: unset;
    background-color: #665c54;
    border-bottom-right-radius: 5px;
  }
  scale trough {
    all: unset;
    background-color: #1d2021;
    min-width: 34px;
    min-height: 2px;
  }
  slider { all: unset; }
}
```

---

## Pattern 3: Multi-Monitor Bar

Two `defwindow` definitions targeting different monitors. Both call the same `bar` widget but pass a `monitor` parameter for workspace filtering. The workspace variable is a JSON object keyed by output name — each monitor's bar accesses only its own slice.

```yuck
; Three monitors, one bar widget, monitor name passed as parameter
; Source: druskus20 — github.com/druskus20/simpler-bar
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

(defwindow win2 :monitor 2
               :geometry (geometry :x 0 :y 0 :height "15px" :width "100.01%" :anchor "center top")
               :stacking "fg"
               :exclusive true
               :focusable false
  (bar0 :monitor "DP-2"))

; Shared bar widget — monitor param slices the workspace JSON object
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

; Workspace data: JSON object keyed by output name
; {"DP-2": [...], "HDMI-A-1": [...], "eDP-1": [...]}
; Each bar accesses only its monitor's slice via workspaces[monitor]
(deflisten workspaces :initial '{"DP-2": [], "HDMI-A-1": [], "eDP-1": []}'
  "./modules/sway-workspaces.py")

(defwidget workspaces [monitor]
  (box :orientation "h" :class "workspaces"
    (for wsp in {workspaces[monitor]}
      (button :class "workspace ${wsp.class}"
              :onclick "swaymsg workspace ${wsp.name}"
        (box
          (label :class "icon" :text "${wsp.icon}")
          (label :class "name" :text "${wsp.name}"))))))

; Open all bars at once:
; eww open-many win0 win1 win2
```

---

## Pattern 4: Hover-Reveal Module

Shows a compact icon normally; reveals a slider or extra content on hover. Uses `eventbox` + `defvar` + `revealer`. See Community section for owenrumney's real source.

```yuck
; Simple volume module: icon + slide-in scale on hover
(defvar show-vol false)

(defwidget volume-hover []
  (eventbox :onhover     "eww update show-vol=true"
            :onhoverlost "eww update show-vol=false"
    (box :orientation "h" :space-evenly false :class "volume-hover"
      (label :class "vol-icon"
             :text {volume > 70 ? "" : volume > 30 ? "" : ""})
      (revealer :transition "slideleft"
                :reveal show-vol
                :duration "300ms"
        (scale :min 0 :max 100
               :value volume
               :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"
               :orientation "h")))))

(deflisten volume :initial "50"
  `pactl subscribe | grep --line-buffered "sink" | while read _; do
     pactl get-sink-volume @DEFAULT_SINK@ | grep -oP '\d+(?=%)' | head -1
   done`)
```

**Hover-reveal SCSS**
```scss
.revealer-on-hover {
  padding: 0 4px;
}

.volume-hover {
  .vol-icon {
    font-size: 14px;
    padding-right: 4px;
  }

  scale {
    min-width: 80px;
  }

  scale trough {
    all: unset;
    background-color: #313244;
    border-radius: 6px;
    min-height: 4px;
    margin: 0 4px;
  }

  scale trough highlight {
    all: unset;
    background-color: #a6e3a1;
    border-radius: 6px;
  }
}
```

---

## Community Bar Implementations

Real defwindow + main widget structure from public dotfiles, verbatim with attribution.

---

### elkowar/dots-of-war — Vertical bar, multi-monitor

Source: `elkowar` — github.com/elkowar/dots-of-war

The canonical example of a 40px vertical sidebar bar with multi-monitor support. Workspace data is a JSON object keyed by output name; each bar instance passes its own screen index to slice it. Also includes a `niri-scroller` — a thin strip anchored to the bottom that emits niri focus commands on scroll, illustrating how an eww window can be an input surface with no visible content.

**eww.yuck (core structure)**
```yuck
;  Source: elkowar — github.com/elkowar/dots-of-war
(defwidget bar [screen]
  (centerbox :orientation "v"
    (box :class "segment-top" :valign "start"
      (top :screen screen))
    (box :valign "center" :class "middle"
      (middle :screen screen))
    (box :valign "end" :class "segment-bottom"
      (bottom :screen screen))))

(defwidget top [screen]
  (box :orientation "v"
    ; workspaces line commented out in source — systray used instead
    (systray :orientation "v" :icon-size 15 :spacing 10)))

(defwidget middle [] (time))
(defwidget time []
  (box :orientation "v" :class "time"
    hour min sec))

(defwidget bottom [screen]
  (box :orientation "v" :valign "end" :space-evenly true :spacing "5"
    (volume)
    (metric :icon "" :font-size 0.8
      "${round((1 - (EWW_DISK["/"].free / EWW_DISK["/"].total)) * 100, 0)}%")
    (metric :icon "" "${round(EWW_RAM.used_mem_perc, 0)}%")
    (metric :icon "" "${round(EWW_CPU.avg, 0)}%")
    (box :class "metric" (date))))

(defwidget metric [icon ?font-size]
  (box :class "metric" :orientation "v"
    (label :class "metric-icon"
           :style {font-size != "" ? "font-size: ${font-size}rem;" : ""}
           :text icon)
    (children)))

(defwidget volume []
  (box :class "volume-metric" :orientation "v" :space-evenly false :valign "fill" :vexpand false
    (scale :orientation "h" :min 0 :max 100
           :onchange "pamixer --set-volume $(echo {} | sed 's/\\..*//g')"
           :value volume)
    (eventbox :onscroll "if [ '{}' == 'up' ]; then pamixer -i 5; else pamixer -d 5; fi"
              :vexpand true :valign "fill"
      (box :orientation "v" :valign "fill" :vexpand true
        (button :onclick "rofi -show rofi-sound-output-chooser &"
                :onrightclick "./audio.sh toggle"
          (label :text audio_sink))
        (button :onclick "pavucontrol &"
          "${volume}%")))))

(defwidget date []
  (box :orientation "v" :halign "center"
    day_word day month year))

; niri-scroller: a thin invisible strip — scroll on it to move niri focus
(defwidget niri-scroller []
  (eventbox :onscroll "if [ '{}' == 'down' ]; then niri msg action focus-column-right; else niri msg action focus-column-left; fi"
            :vexpand true :valign "fill"
            :style "background-color: #8ec07c; border-radius: 10px;"))

(deflisten volume :initial "0" "./audio.sh volume")
(deflisten workspaces :initial '{"DP-2": [], "HDMI-A-1": []}' "./swayspaces.py")

(defpoll hour     :interval "1s"  "date +%H")
(defpoll min      :interval "1s"  "date +%M")
(defpoll sec      :interval "1s"  "date +%S")
(defpoll day_word :interval "10m" "date +%a | tr [:upper:] [:lower:]")
(defpoll day      :interval "10m" "date +%d")
(defpoll month    :interval "1h"  "date +%m")
(defpoll year     :interval "1h"  "date +%y")

(defvar audio_sink "")

; Primary monitor — monitor array tries entries in order, first match wins
(defwindow bar_1
  :monitor '["<primary>", "DisplayPort-0", "PHL 345B1C"]'
  :stacking "fg"
  :geometry (geometry :x 0 :y 0 :width "40px" :height "100%" :anchor "center left")
  :reserve (struts :distance "40px" :side "left")
  :exclusive true
  (bar :screen 1))

; Secondary monitor
(defwindow bar_2
  :monitor '[2, "HDMI-A-1"]'
  :geometry (geometry :x 0 :y 0 :width "40px" :height "100%" :anchor "top left")
  :reserve (struts :distance "40px" :side "left")
  (bar :screen 2))

; Thin bottom strip — input surface only, no visible content
(defwindow niri_scroller
  :monitor '["<primary>", "DisplayPort-0", "PHL 345B1C"]'
  :stacking "fg"
  :geometry (geometry :x 0 :y 0 :width "800px" :height "5px" :anchor "bottom center")
  :reserve (struts :distance "10px" :side "bottom")
  :exclusive false
  (niri-scroller))
```

**Key pattern — monitor array fallback:** `:monitor '["<primary>", "DisplayPort-0", "PHL 345B1C"]'` tries each entry in order; first match wins. Handles connector name variations across machines.

**eww.scss (excerpt)**
```scss
;  Source: elkowar — github.com/elkowar/dots-of-war
* { all: unset; }

window {
  background: #282828;
  color: #ebdbb2;
  font-size: 14px;
  font-family: "Terminus (TTF)";
  & > * { margin: 3px; }
}

.workspaces button {
  background: none;
  margin: 3px;
  &.inactive { color: #888974; }
  &.active   { color: #8ec07c; }
  &.occupied { font-size: 1.01rem; }
  &.empty    { font-size: 0.8rem; }
}

.segment-top   { margin-top: 10px; }
.segment-bottom { margin-bottom: 10px; }

.volume-metric {
  background-color: #1d2021;
  scale trough highlight {
    all: unset;
    background-color: #665c54;
    border-bottom-right-radius: 5px;
  }
  scale trough {
    all: unset;
    background-color: #1d2021;
    min-width: 34px;
    min-height: 2px;
  }
  slider { all: unset; }
}

.metric {
  background-color: #1d2021;
  padding: 5px 2px;
}
.metric-icon {
  font-family: "Font Awesome 6 Free";
  font-size: 0.7em;
}

.time {
  padding-top: 7px;
  padding-bottom: 7px;
  color: #a89984;
}
```

---

### rxyhn — Vertical bar, BSPWM, hover-reveal controls

Source: `rxyhn` — github.com/rxyhn/yuki

A 47px vertical bar for BSPWM. Notable patterns: workspace widget uses `literal :content` (eww renders a yuck string emitted by the script), and all hover-reveal controls (volume, brightness, power) use the same `eventbox + revealer :transition "slideup"` pattern. The revealer slides upward to reveal a vertical scale above the trigger button.

**eww.yuck (core structure)**
```yuck
;  Source: rxyhn — github.com/rxyhn/yuki
(defvar eww "$HOME/.local/bin/eww -c $HOME/.config/eww/bar")

; Workspaces via literal — script emits a yuck string directly
(defwidget workspaces []
  (literal :content workspace))
(deflisten workspace "scripts/workspace")

; Battery: two separate polls — icon and capacity
(defwidget bat []
  (box :orientation "v" :space-evenly "false"
    (label :class "bat" :halign "end" :text battery
           :tooltip "Battery: ${battery-cappacity}%")))
(defpoll battery           :interval "1s" "scripts/battery icon")
(defpoll battery-cappacity :interval "1s" "scripts/battery percent")

; Brightness: hover to reveal vertical scale (slideup)
(defwidget bright []
  (eventbox :onhover "${eww} update bright=true"
            :onhoverlost "${eww} update bright=false"
    (box :orientation "v" :space-evenly "false" :spacing 2
      (revealer :transition "slideup" :reveal bright :duration "550ms"
        (scale :class "bribar" :value current-brightness :orientation "v"
               :flipped true :tooltip "Brightness: ${current-brightness}%"
               :onchange "brightnessctl set {}%"
               :max 101 :min 0))
      (label :class "brightness-icon" :text ""))))
(defpoll current-brightness :interval "1s"
  "brightnessctl -m -d amdgpu_bl0 | awk -F, '{print substr($4, 0, length($4)-1)}' | tr -d '%'")
(defvar bright false)

; Volume: hover to reveal vertical scale (slideup)
(defwidget volum []
  (eventbox :onhover "${eww} update volum=true"
            :onhoverlost "${eww} update volum=false"
    (box :orientation "v" :space-evenly "false" :spacing "2"
      (revealer :transition "slideup" :reveal volum :duration "550ms"
        (scale :class "volbar" :value current-volume :orientation "v"
               :flipped true :tooltip "Volume: ${current-volume}%"
               :max 101 :min 0
               :onchange "amixer -D pulse sset Master {}%"))
      (button :onclick "scripts/popup audio" :class "volume-icon" ""))))
(defpoll current-volume :interval "1s"
  "amixer -D pulse sget Master | grep 'Left:' | awk -F'[][]' '{ print $2 }' | tr -d '%'")
(defvar volum false)

; Power menu: hover to reveal action buttons (slideup), trigger is shutdown icon
(defwidget power []
  (eventbox :onhover "${eww} update power=true"
            :onhoverlost "${eww} update power=false"
    (box :orientation "v" :space-evenly "false" :class "powermenu"
      (revealer :transition "slideup" :reveal power :duration "550ms"
        (box :orientation "v" :space-evenly "false"
          (button :class "button-bspres" :tooltip "BSPWM Restart"
                  :onclick "bspc wm -r" "")
          (button :class "button-reb" :tooltip "Reboot"
                  :onclick "reboot" "")
          (button :class "button-quit" :tooltip "Logout"
                  :onclick "killall bspwm" "")
          (button :class "button-lock" :tooltip "Lock Screen"
                  :onclick "betterlockscreen -l" "")))
      (button :class "button-off" :tooltip "Shutdown"
              :onclick "shutdown now" ""))))
(defvar power false)

; Top / End composition
(defwidget top []
  (box :orientation "v" :space-evenly "false" :valign "start"
    (launcher)
    (workspaces)))

(defwidget end []
  (box :orientation "v" :space-evenly "false" :valign "end" :spacing 5
    (control)
    (bottom)))

(defwidget bar []
  (box :class "eww_bar" :orientation "v" :vexpand "false" :hexpand "false"
    (top)
    (end)))

(defwindow bar
  :geometry (geometry :x "0" :y "0" :height "100%" :width "47px")
  :monitor 0
  :reserve (struts :distance "35px" :side "left")
  :wm-ignore false
  :hexpand "false" :vexpand "false"
  (bar))

; Calendar popup — positioned to the right of the bar
(defwindow calendar
  :geometry (geometry :x "70px" :y "65%" :width "270px" :height "60px")
  (cal))
```

**eww.scss (excerpt)**
```scss
;  Source: rxyhn — github.com/rxyhn/yuki
* { all: unset; }

$background: #1A1B26;
$foreground: #A9B1D6;
$black: #24283B;
$gray: #565F89;
$blue: #7AA2F7;
$magenta: #BB9AF7;
$green: #73DACA;
$yellow: #E0AF68;
$red: #F7768E;

.eww_bar { background-color: $background; padding: .3rem; }

.control {
  padding: .5rem;
  font-family: Material Icons;
  font-size: 1.6em;
  background-color: $black;
  border-radius: 5px;
}

.volume-icon { margin: .2rem 0; color: $green; }
.bat { font-family: JetBrainsMono Nerd Font; font-size: 1.2em; color: $blue; }

scale trough {
  all: unset;
  background-color: $background;
  border-radius: 5px;
  min-height: 80px;
  min-width: 10px;
  margin: .3rem 0;
}
.volbar trough highlight { background-color: $green; border-radius: 5px; }

.powermenu { font-family: feather; font-size: 1.4em; font-weight: bold; }
.button-off  { color: $red;     margin-bottom: .5rem; }
.button-reb  { color: $yellow; }
.button-quit { color: $magenta; }
.button-lock { color: $blue; }

.time {
  font-family: Comic Mono;
  font-weight: bold;
  font-size: 1.2em;
  background-color: $black;
  border-radius: 5px;
  padding: .7rem 0 .5rem;
  margin: .5rem 0;
}
```

---

### saimoomedits — Horizontal bar + multi-window left sidebar

Source: `saimoomedits` — github.com/saimoomedits/dotfiles

Two separate eww configs: a horizontal bar and a multi-window desktop sidebar. The bar uses `circular-progress` for battery and RAM, and `literal :content` for workspaces (script emits yuck directly). The sidebar splits each UI element into its own `defwindow` positioned by absolute pixel coordinates — this gives full control over layering and independent open/close per element.

**eww/bar/eww.yuck (horizontal bar)**
```yuck
;  Source: saimoomedits — github.com/saimoomedits/dotfiles
(defwidget bar_1 []
  (box :class "bar_class" :orientation "h"
    (right)
    (center)
    (left)))

(defwidget right []
  (box :orientation "h" :space-evenly false :halign "start" :class "right_modules"
    (workspaces)))

(defwidget center []
  (box :orientation "h" :space-evenly false :halign "center" :class "center_modules"
    (music)))

(defwidget left []
  (box :orientation "h" :space-evenly false :halign "end" :class "left_modules"
    (bright)
    (volume)
    (wifi)
    (sep)
    (bat)
    (mem)
    (sep)
    (clock_module)))

; Workspaces via literal — scripts/workspace emits yuck string directly
(defwidget workspaces []
  (literal :content workspace))
(deflisten workspace "scripts/workspace")

; Battery with circular-progress
(defwidget bat []
  (box :class "bat_module" :vexpand "false" :hexpand "false"
    (circular-progress :value battery :class "batbar" :thickness 4
      (button :class "iconbat" :limit-width 2
              :tooltip "battery on ${battery}%"
              :onclick "$HOME/.config/eww/bar/scripts/pop system"
        ""))))
(defpoll battery :interval "15s" "./scripts/battery --bat")

; Volume with hover-reveal (slideleft)
(defwidget volume []
  (eventbox :onhover "${eww} update vol_reveal=true"
            :onhoverlost "${eww} update vol_reveal=false"
    (box :class "module-2" :space-evenly "false" :orientation "h" :spacing "3"
      (button :onclick "scripts/pop audio" :class "volume_icon" "")
      (revealer :transition "slideleft" :reveal vol_reveal :duration "350ms"
        (scale :class "volbar" :value volume_percent :orientation "h"
               :tooltip "${volume_percent}%"
               :max 100 :min 0
               :onchange "amixer -D pulse sset Master {}%")))))
(defpoll volume_percent :interval "3s"
  "amixer -D pulse sget Master | grep 'Left:' | awk -F'[][]' '{ print $2 }' | tr -d '%'")
(defvar vol_reveal false)

; WiFi with hover-reveal ESSID (slideright)
(defwidget wifi []
  (eventbox :onhover "${eww} update wifi_rev=true"
            :onhoverlost "${eww} update wifi_rev=false"
    (box :vexpand "false" :hexpand "false" :space-evenly "false"
      (button :class "module-wif" :onclick "networkmanager_dmenu"
              :style "color: ${COL_WLAN};" WLAN_ICON)
      (revealer :transition "slideright" :reveal wifi_rev :duration "350ms"
        (label :class "module_essid" :text ESSID_WLAN)))))

; Clock with hover-reveal date (slideleft)
(defwidget clock_module []
  (eventbox :onhover "${eww} update time_rev=true"
            :onhoverlost "${eww} update time_rev=false"
    (box :class "module" :space-evenly "false" :orientation "h" :spacing "3"
      (label :text clock_time :class "clock_time_class")
      (label :text "" :class "clock_time_sep")
      (label :text clock_minute :class "clock_minute_class")
      (revealer :transition "slideleft" :reveal time_rev :duration "350ms"
        (button :class "clock_date_class"
                :onclick "$HOME/.config/eww/bar/scripts/pop calendar" clock_date)))))

; Music with album art + hover-reveal playback controls (slideright)
(defwidget music []
  (eventbox :onhover "${eww} update music_reveal=true"
            :onhoverlost "${eww} update music_reveal=false"
    (box :class "module-2" :orientation "h" :space-evenly "false"
      (box :class "song_cover_art"
           :style "background-image: url('${cover_art}');")
      (button :class "song" :wrap "true"
              :onclick "~/.config/eww/bar/scripts/pop music" song)
      (revealer :transition "slideright" :reveal music_reveal :duration "350ms"
        (box
          (button :class "song_btn_prev"
                  :onclick "~/.config/eww/bar/scripts/music_info --prev" "")
          (button :class "song_btn_play"
                  :onclick "~/.config/eww/bar/scripts/music_info --toggle" song_status)
          (button :class "song_btn_next"
                  :onclick "~/.config/eww/bar/scripts/music_info --next" ""))))))

(defwindow bar
  :geometry (geometry :x "0%" :y "9px" :width "98%" :height "30px" :anchor "top center")
  :stacking "fg"
  :windowtype "dock"
  (bar_1))
```

**eww/leftbar/eww.yuck (multi-window sidebar)**
```yuck
;  Source: saimoomedits — github.com/saimoomedits/dotfiles
; Each UI element is its own defwindow, positioned by absolute pixel coordinates.
; This gives full control over layering and independent open/close per element.

(defwidget profile []
  (box :orientation "v" :spacing 20 :space-evenly "false"
    (box :class "profile_picture" :halign "center"
         :style "background-image: url('${IMAGE}');")))

(defwidget time []
  (box :orientation "v" :space-evenly "false"
    (box :orientation "h" :space-evenly "false" :halign "start"
      (label :class "hour_class"   :text time_hour)
      (label :class "minute_class" :text time_min)
      (label :class "mer_class"    :text time_mer))
    (box :orientation "h" :space-evenly "false" :halign "start"
      (label :class "day_class"   :text time_day)
      (label :class "month_class" :text time_month)
      (label :class "year_class"  :text time_year))
    (box :orientation "h" :space-evenly "false" :halign "start"
      (label :class "day_class_n"   :text time_day_name)
      (label :class "month_class_n" :text time_month_name))))

; System stats: battery, CPU, memory as full-ring circular-progress widgets
(defwidget system []
  (box :class "sys_win" :orientation "h" :space-evenly "false" :spacing 13
    (box :class "sys_bat_box" :orientation "v"
      (circular-progress :value battery :class "sys_bat" :thickness 100
        (label :text " " :class "cc_cc"))
      (label :text "BAT" :class "sys_icon_bat"))
    (box :class "sys_cpu_box" :orientation "v"
      (circular-progress :value cpu :class "sys_cpu" :thickness 100
        (label :text " " :class "cc_cc"))
      (label :text "CPU" :class "sys_icon_cpu"))
    (box :class "sys_mem_box" :orientation "v"
      (circular-progress :value memory :class "sys_mem" :thickness 100
        (label :text " " :class "cc_cc"))
      (label :text "MEM" :class "sys_icon_mem"))))

; Each element is an independent window, positioned absolutely on screen
(defwindow pfp
  :geometry (geometry :anchor "left top" :width "190px" :height "23"
                      :x "0px" :y "65px")
  (profile))

(defwindow time
  :geometry (geometry :anchor "left top" :width "260px" :height "100"
                      :x "220px" :y "100px")
  (time))

(defwindow sys_usg
  :geometry (geometry :width "420px" :height "160px" :x "10px" :y "510px")
  (system))

(defwindow audio
  :geometry (geometry :width "0px" :height "160px" :x "370px" :y "307px")
  (audio))
```

---

### druskus20/simpler-bar — Minimal horizontal, 3-monitor, clean module structure

Source: `druskus20` — github.com/druskus20/simpler-bar

A clean minimal bar. Notable: one `bar0` widget called by three windows; `deflisten` for volume and mic (not `defpoll`); camera module reads lsmod uvcvideo usage count; per-icon class system for connected/disconnected states; `tooltip` wrapper widget for hover text.

**eww.yuck**
```yuck
;  Source: druskus20 — github.com/druskus20/simpler-bar

(defwindow win0 :monitor 0
  :geometry (geometry :x 0 :y 0 :height "15px" :width "100%" :anchor "center top")
  :stacking "fg" :exclusive true :focusable false
  (bar0 :monitor "eDP-1"))
(defwindow win1 :monitor 1
  :geometry (geometry :x 0 :y 0 :height "15px" :width "100.01%" :anchor "center top")
  :stacking "fg" :exclusive true :focusable false
  (bar0 :monitor "HDMI-A-1"))
(defwindow win2 :monitor 2
  :geometry (geometry :x 0 :y 0 :height "15px" :width "100.01%" :anchor "center top")
  :stacking "fg" :exclusive true :focusable false
  (bar0 :monitor "DP-2"))

(defwidget bar0 [monitor]
  (box :orientation "h" :space-evenly true :class "bar"
    (box :class "right" :halign "start" :spacing 20
      (workspaces :monitor monitor))
    (box :class "right" :halign "end" :spacing 20 :space-evenly false
      (tray) (wifi) (camera) (mic) (speakers) (battery) (time) (date))))

; Workspaces: JSON keyed by monitor name, each entry has .class and .icon set by script
(deflisten workspaces :initial '{"DP-2": [], "HDMI-A-1": [], "eDP-1": []}'
  "./modules/sway-workspaces.py")

(defwidget workspaces [monitor]
  (box :orientation "h" :class "workspaces"
    (for wsp in {workspaces[monitor]}
      (button :class "workspace ${wsp.class}"
              :onclick "swaymsg workspace ${wsp.name}"
        (box
          (label :class "icon" :text "${wsp.icon}")
          (label :class "name" :text "${wsp.name}"))))))

; Camera: reads lsmod uvcvideo usage count — 0 means disconnected
(defpoll camera :initial "0" :interval "10s" "./modules/camera.sh")
(defwidget camera []
  (box :class "camera" :space-evenly false
    (label :class "icon ${camera == 0 ? 'disconnected' : 'connected'}"
           :text "${camera == 0 ? '󱜷' : '󰖠'}")))

; WiFi: returns SSID string or "0" when disconnected
(defpoll wifi :initial "0" :interval "1m" "./modules/wifi.sh")
(defwidget wifi []
  (tooltip
    (label :class "tooltip" :text "${wifi == 0 ? 'Disconnected' : wifi}")
    (box :class "wifi" :space-evenly false
      (label :class "icon ${wifi == 0 ? 'disconnected' : 'connected'}"
             :text "${wifi == 0 ? '󰖪' : '󱚽'}"))))

; Mic: deflisten — 0=muted, 1=idle, 2=running; click to toggle mute
(deflisten mic :initial 0 "./modules/listen-mic.sh")
(defwidget mic []
  (box :class "mic" :space-evenly false
    (button :class "mute-toggle"
            :onclick "pactl set-source-mute @DEFAULT_SOURCE@ toggle"
      (label :class "icon single-icon ${mic == 2 ? 'running' : mic == 0 ? 'muted' : ''}"
             :text "${mic == 0 ? '󰍭' : '󰍬'}"))))

; Volume: deflisten — emits {"volume": N, "muted": bool}; inline scale in bar
(deflisten speakers :initial '{ "volume": 0, "muted": "false" }'
  "./modules/listen-volume.sh")
(defwidget speakers []
  (tooltip
    (label :class "tooltip" :text "${speakers.volume}%")
    (box :class "speakers" :space-evenly false
      (button :class "icon ${speakers.volume > 100 ? 'over' : ''} ${speakers.muted ? 'muted' : ''}"
              :onclick "pactl set-sink-mute @DEFAULT_SINK@ toggle"
        (label :text "${speakers.muted ? '󰖁' :
                       speakers.volume > 70 ? '󰕾' :
                       speakers.volume > 35 ? '󰖀' : '󰕿'}"))
      (scale :class "${speakers.volume > 100 ? 'over' : ''}"
             :min 0 :max 100 :value "${speakers.volume}"
             :onchange "pactl set-sink-volume @DEFAULT_SINK@ {}%"))))

; Battery: uses EWW_BATTERY magic variable with 10-level icon expression
(defwidget battery []
  (box :class "battery" :space-evenly false
    (label :class "icon ${EWW_BATTERY["BAT0"].capacity <= 10 ? 'critical' : ''}"
           :text "${EWW_BATTERY["BAT0"].capacity > 95 ? '󰁹'
                 : EWW_BATTERY["BAT0"].capacity > 90 ? '󰂂'
                 : EWW_BATTERY["BAT0"].capacity > 80 ? '󰂁'
                 : EWW_BATTERY["BAT0"].capacity > 70 ? '󰂀'
                 : EWW_BATTERY["BAT0"].capacity > 60 ? '󰁿'
                 : EWW_BATTERY["BAT0"].capacity > 50 ? '󰁾'
                 : EWW_BATTERY["BAT0"].capacity > 40 ? '󰁽'
                 : EWW_BATTERY["BAT0"].capacity > 30 ? '󰁼'
                 : EWW_BATTERY["BAT0"].capacity > 20 ? '󰁻'
                 : EWW_BATTERY["BAT0"].capacity > 10 ? '󰁺'
                 : '󰂃'}")
    (label :text "${EWW_BATTERY["BAT0"].capacity}" :halign "center" :xalign 0.5)))
```
