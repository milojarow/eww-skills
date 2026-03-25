# POPUP_PATTERNS.md — Popup and Overlay Patterns

Four complete popup patterns with full yuck and scss, plus named community patterns for power menus, notification revealers, and reusable hover/click toggle widgets.

---

## Pattern 1: Power Menu

A full-screen overlay with power action buttons. Based on the dharmx powermenu tutorial and the iSparsh/gross powermenu structure.

**How it works:** The window fills 100% of the screen with `:wm-ignore true` so the WM doesn't tile it. A close button or each action button dismisses it via `eww close powermenu`. A bar button opens it with `eww open powermenu --toggle`.

```yuck
; Full-screen overlay window
(defwindow powermenu
  :stacking "fg"
  :wm-ignore true
  :windowtype "normal"     ; required for X11; Wayland can omit
  :geometry (geometry :width "100%" :height "100%")
  (powermenu-layout))

; Layout: greeting + button row
(defwidget powermenu-layout []
  (box :class "powermenu"
       :orientation "v"
       :space-evenly true
    (box :orientation "v" :valign "center" :halign "center" :spacing 8
      (label :class "pm-greeting" :text "See you later.")
      (label :class "pm-time"
             :text {formattime(EWW_TIME, "%H:%M — %A, %B %d")}))
    (box :class "pm-buttons"
         :orientation "h"
         :halign "center"
         :valign "end"
         :spacing 20
      (power-btn :icon "" :label "Lock"
                 :cmd "loginctl lock-session")
      (power-btn :icon "" :label "Suspend"
                 :cmd "systemctl suspend")
      (power-btn :icon "󰍃" :label "Logout"
                 :cmd "loginctl kill-session self")
      (power-btn :icon "󰜉" :label "Reboot"
                 :cmd "systemctl reboot")
      (power-btn :icon "⏻"  :label "Shutdown"
                 :cmd "systemctl poweroff"))
    ; Close button in top-right corner
    (button :class "pm-close"
            :halign "end" :valign "start"
            :onclick "eww close powermenu"
      "")))

; Reusable power button: closes the menu then runs the command
(defwidget power-btn [icon label cmd]
  (button :class "power-btn"
          :onclick "eww close powermenu && ${cmd}"
          :tooltip label
    (box :orientation "v" :space-evenly false :spacing 6
      (label :class "power-btn-icon" :text icon)
      (label :class "power-btn-label" :text label))))

; Toggle from a bar button:
; (button :onclick "eww open powermenu --toggle" "⏻")
```

**eww.scss**
```scss
.powermenu {
  background-color: rgba(30, 30, 46, 0.92);
  padding: 40px;
  font-family: "JetBrainsMono Nerd Font", sans-serif;
}

.pm-greeting {
  font-size: 28px;
  font-weight: bold;
  color: #cdd6f4;
}

.pm-time {
  font-size: 14px;
  color: #6c7086;
}

.pm-close {
  font-size: 24px;
  color: #6c7086;
  padding: 8px;
  border-radius: 6px;

  &:hover { color: #f38ba8; }
}

.power-btn {
  background-color: #313244;
  border-radius: 16px;
  padding: 24px 20px;
  min-width: 90px;
  min-height: 90px;

  &:hover {
    background-color: #45475a;
    transition: 200ms linear background-color;
  }

  &:first-child { color: #a6e3a1; }  /* Lock — green */
  &:last-child  { color: #f38ba8; }  /* Shutdown — red */
}

.power-btn-icon {
  font-size: 32px;
  halign: center;
}

.power-btn-label {
  font-size: 11px;
  color: #6c7086;
}
```

**Toggle script** (optional, for complex multi-window toggle):
```bash
#!/bin/bash
# scripts/powermenu — toggle power menu
eww open powermenu --toggle
```

---

## Pattern 2: Calendar Popup

A floating calendar window that opens on clock click. The `calendar` widget is built-in to eww.

```yuck
; Floating calendar window — positioned relative to anchor
(defwindow calendar-popup
  :geometry (geometry :x "-20px"
                      :y "35px"
                      :width "280px"
                      :height "auto"
                      :anchor "top right")
  :stacking "fg"
  (cal-widget))

; Calendar widget with today highlighted
(defwidget cal-widget []
  (box :class "cal-box" :orientation "v"
    (calendar :class "calendar"
              :day   {formattime(EWW_TIME, "%d")}
              :month {formattime(EWW_TIME, "%m")}
              :year  {formattime(EWW_TIME, "%Y")})))

; In the bar — clicking the clock toggles the calendar
(defwidget clock-btn []
  (button :onclick "eww open calendar-popup --toggle"
          :class "clock-btn"
    {formattime(EWW_TIME, "%H:%M")}))
```

**eww.scss**
```scss
.cal-box {
  background-color: #1e1e2e;
  border-radius: 12px;
  padding: 12px;
  border: 1px solid #313244;
}

/* GTK calendar widget selectors */
.calendar {
  color: #cdd6f4;
  font-family: "JetBrainsMono Nerd Font", monospace;
}

.calendar:selected {
  background-color: #cba6f7;
  color: #1e1e2e;
  border-radius: 4px;
}

.calendar.highlight {
  color: #cba6f7;
  font-weight: bold;
}

/* Navigation arrows */
.calendar button {
  color: #89b4fa;
  &:hover { color: #cba6f7; }
}
```

---

## Pattern 3: Volume OSD Popup

A short-lived volume indicator that appears when volume changes and auto-hides. Uses `deflisten` to watch for volume events, plus a shell script that manages the popup lifetime.

```yuck
; Small OSD window — centered, auto-sized
(defwindow volume-osd
  :geometry (geometry :x "0%"
                      :y "-15%"
                      :width "200px"
                      :height "60px"
                      :anchor "bottom center")
  :stacking "overlay"
  :focusable false
  (volume-osd-widget))

; OSD content: icon + percentage + progress bar
(defwidget volume-osd-widget []
  (box :class "osd-box" :orientation "h" :space-evenly false :spacing 10
    (label :class "osd-icon"
           :text {volume == 0 ? "󰝟" : volume < 30 ? "󰕿" : volume < 70 ? "󰖀" : "󰕾"})
    (box :orientation "v" :space-evenly false :spacing 4
      (label :class "osd-value" :text "${volume}%")
      (scale :min 0 :max 100
             :active false
             :value volume
             :class "osd-scale"))))

; Listen for volume changes via pactl
(deflisten volume :initial "50"
  `pactl subscribe | grep --line-buffered 'sink' | while read _; do
     pactl get-sink-volume @DEFAULT_SINK@ | grep -oP '\d+(?=%)' | head -1
   done`)
```

**Shell script to show OSD with timeout** (`scripts/show-osd`):
```bash
#!/bin/bash
# Show the volume OSD for 2 seconds, then hide it.
# Call this whenever volume changes (e.g. from keybind).
eww open volume-osd

# Use a lockfile approach to debounce rapid changes
LOCK="/tmp/eww-osd-timer.lock"
rm -f "$LOCK"
touch "$LOCK"

{
  sleep 2
  # Only close if this is still the latest invocation
  if [ -f "$LOCK" ]; then
    eww close volume-osd
    rm -f "$LOCK"
  fi
} &
```

**Usage in Sway config:**
```
bindsym XF86AudioRaiseVolume exec pactl set-sink-volume @DEFAULT_SINK@ +5% && ~/.config/eww/scripts/show-osd
bindsym XF86AudioLowerVolume exec pactl set-sink-volume @DEFAULT_SINK@ -5% && ~/.config/eww/scripts/show-osd
bindsym XF86AudioMute        exec pactl set-sink-mute @DEFAULT_SINK@ toggle && ~/.config/eww/scripts/show-osd
```

**eww.scss**
```scss
.osd-box {
  background-color: rgba(30, 30, 46, 0.90);
  border-radius: 10px;
  padding: 10px 16px;
  border: 1px solid #313244;
}

.osd-icon {
  font-size: 22px;
  color: #a6e3a1;
}

.osd-value {
  font-size: 13px;
  font-weight: bold;
  color: #cdd6f4;
}

.osd-scale scale trough {
  all: unset;
  background-color: #313244;
  border-radius: 4px;
  min-height: 4px;
  min-width: 120px;
}

.osd-scale scale trough highlight {
  all: unset;
  background-color: #a6e3a1;
  border-radius: 4px;
}
```

---

## Pattern 4: Notification Revealer Panel

A slide-in notification panel from the right edge. Toggled by a button in the bar. Uses a `defvar` for visibility state and a `revealer` for the slide animation. Inspired by druskus20/eugh notification-revealer example.

```yuck
; Notification panel window — tall, anchored right
(defwindow notif-panel
  :geometry (geometry :x "0%"
                      :y "0%"
                      :width "300px"
                      :height "100%"
                      :anchor "top right")
  :stacking "fg"
  (notif-panel-widget))

; State variable — controls panel open/closed
(defvar notif-open false)

; Panel content with slide-in from right
(defwidget notif-panel-widget []
  (box :class "notif-panel-outer" :orientation "h" :halign "end"
    (revealer :transition "slideright"
              :reveal notif-open
              :duration "300ms"
      (box :class "notif-panel"
           :orientation "v"
           :space-evenly false
           :spacing 8
        ; Header row
        (box :orientation "h" :halign "fill"
          (label :class "notif-title" :text "Notifications" :hexpand true)
          (button :class "notif-close"
                  :onclick "eww update notif-open=false"
            ""))
        ; Notification list (populated by a script)
        (box :class "notif-list" :orientation "v" :spacing 4
          (for notif in notifications
            (notif-item :summary {notif.summary}
                        :body    {notif.body}
                        :icon    {notif.icon})))
        ; Empty state
        {arraylength(notifications) == 0 ?
          "(label :class \"notif-empty\" :text \"No notifications\")" : ""}))))

; Individual notification card
(defwidget notif-item [summary body icon]
  (box :class "notif-item" :orientation "h" :space-evenly false :spacing 8
    (label :class "notif-icon" :text icon)
    (box :orientation "v" :space-evenly false
      (label :class "notif-summary" :text summary :truncate true :limit-width 25)
      (label :class "notif-body"    :text body    :truncate true :limit-width 30))))

; Bar button that opens the panel
(defwidget notif-btn []
  (button :class "notif-btn"
          :onclick "eww update notif-open=${notif-open ? false : true}"
    ""))

; Notifications variable — populated by a notification daemon script
(deflisten notifications :initial "[]"
  "scripts/notifications")
```

**eww.scss**
```scss
.notif-panel {
  background-color: #1e1e2e;
  border-left: 1px solid #313244;
  padding: 16px 12px;
  min-height: 100%;
}

.notif-title {
  font-size: 14px;
  font-weight: bold;
  color: #cdd6f4;
}

.notif-close {
  font-size: 16px;
  color: #6c7086;
  padding: 4px;
  &:hover { color: #f38ba8; }
}

.notif-item {
  background-color: #313244;
  border-radius: 8px;
  padding: 8px 10px;

  &:hover { background-color: #45475a; }
}

.notif-icon {
  font-size: 20px;
  color: #89b4fa;
}

.notif-summary {
  font-size: 12px;
  font-weight: bold;
  color: #cdd6f4;
}

.notif-body {
  font-size: 11px;
  color: #a6adc8;
}

.notif-empty {
  color: #6c7086;
  font-style: italic;
}
```

---

## Community Popup Patterns

Named, attributed patterns from public dotfiles.

---

### iSparsh/gross — bigpowermenu: centered overlay, onrightclick actions

Source: `iSparsh` — github.com/iSparsh/gross

A compact centered overlay power menu. Notable differences from the full-screen pattern: small fixed size (`20% x 10%`) centered on screen via `anchor "center center"`; uses `:onrightclick` instead of `:onclick` for the action bindings; each button carries an inline `:style` for its accent color. The window is opened from a quicksettings bar button.

```yuck
;  Source: iSparsh — github.com/iSparsh/gross
(defwindow bigpowermenu
  :wm-ignore true
  :monitor 0
  :windowtype "dock"
  :geometry (geometry :x "0px"
                      :y "0%"
                      :width "20%"
                      :height "10%"
                      :anchor "center center")
  (bigpowermenu))

(defwidget bigpowermenu []
  (box :orientation "h" :space-evenly false :class "bigpowermenu"
       :halign "center" :valign "center" :spacing 20
    ; Actions bound to onrightclick — left click is intentionally unused
    (button :style "color: #d8dee9;" :class "shutdown"
            :onrightclick "systemctl poweroff" "")
    (button :style "color: #e5e9f0;" :class "reboot"
            :onrightclick "systemctl reboot" "")
    (button :style "color: #eceff4;" :class "lock"
            :onrightclick "bsplock" "")
    (button :style "color: #e8e8e8;" :class "suspend"
            :onrightclick "mpc -q pause & amixer set Master mute & systemctl suspend" "")
    (button :style "color: #ffffff;" :class "logout"
            :onrightclick "bspc quit" "")))
```

**Key differences from Pattern 1:**
- Small centered window rather than full-screen overlay — less intrusive
- `:onrightclick` binding — left click does nothing, right click acts; unusual but intentional
- Per-button inline `:style` color rather than CSS class theming
- No close button — clicking outside or using the WM closes it (`:wm-ignore true` still applies)

iSparsh also has a full multi-window desktop layout (fetch, sys, quotes, quicksettings, appbar, calendar, favorites, notes, search) where every element is its own `defwindow` with `windowtype "dock"` positioned by absolute coordinates. See `eww_windows.yuck` for the complete set.

---

### druskus20/eugh — notification-revealer: deflisten-driven inline revealer

Source: `druskus20` — github.com/druskus20/eugh

A minimal inline notification strip anchored top-left. The revealer state is driven directly by a field in the `deflisten` JSON output (`notifications_listen.show`) — no separate `defvar` needed. The content is rendered via `literal :content` so the script controls the full widget structure of each notification.

```yuck
;  Source: druskus20 — github.com/druskus20/eugh
(defwindow test
  :monitor 0
  :hexpand false
  :vexpand false
  :geometry (geometry :anchor "top left" :x 350 :y 30 :width "900px" :height "50px")
  (box :space-evenly false
    (notification-revealer)))

; Script emits {"show": true/false, "content": "<yuck string>"}
; .show drives the revealer; .content is rendered via literal
(deflisten notifications_listen :initial '{"show": false, "content": ""}' "./notifications.sh")

(defwidget notification-revealer []
  (box :class "notification-revealer"
       :orientation "h"
       :space-evenly false
    "NOTIFICATIONS: "
    (revealer :reveal {notifications_listen.show}
              :transition "slideright"
      (box
        (literal :valign "center" :content {notifications_listen.content})))))
```

**Key pattern:** The script is the single source of truth for both visibility and content. `notifications_listen.show` is a boolean field in the JSON — no `eww update` call is ever needed from the bar side. The notification daemon sets show to true when a notification arrives and false when it expires.

---

### owenrumney — revealer-on-hover and clickbox: reusable generic toggle widgets

Source: `owenrumney` — github.com/owenrumney/squeaky-theme

Two generic parameterized widgets that encapsulate the entire hover-reveal and click-toggle patterns. Both accept `var` (the boolean state variable), `varname` (its string name for `eww update`), and optional `?class`, `?duration`, `?transition`. Children are addressed by index with `children :nth`.

**revealer-on-hover** — hover the first child to reveal the second:

```yuck
;  Source: owenrumney — github.com/owenrumney/squeaky-theme

; Generic hover revealer — wraps any two children
; Usage:
;   (revealer-on-hover :var show-vol :varname "show-vol"
;     (label :text "")          ; child 0 — always visible, triggers hover
;     (scale ...))              ; child 1 — revealed on hover
;   child 2 (optional) — extra content after the revealer
(defwidget revealer-on-hover [var varname ?class ?duration ?transition]
  (box :class "${class} revealer-on-hover"
       :orientation "h"
       :space-evenly false
    (eventbox :class "eventbox"
              :onhover "eww update ${varname}=true"
              :onhoverlost "eww update ${varname}=false"
      (box :space-evenly false
        (children :nth 0)
        (revealer :reveal var
                  :transition {transition ?: "slideright"}
                  :duration {duration ?: "500ms"}
          (children :nth 1))
        (box :class "${class}" (children :nth 2))))))
```

**hovered-sign** — swap between two children based on a boolean (no eventbox; caller manages hover):

```yuck
;  Source: owenrumney — github.com/owenrumney/squeaky-theme

; Shows child 0 when var is false, child 1 when var is true.
; Both are always mounted; only one is revealed at a time.
; Useful for toggling between two icons or labels.
(defwidget hovered-sign [var]
  (box :space-evenly false
    (revealer :reveal {!var}
              :duration "100ms"
              :transition "slideleft"
      (children :nth 0))
    (revealer :reveal {var}
              :duration "100ms"
              :transition "slideleft"
      (children :nth 1))))
```

**clickbox** — click the first child to reveal the second; includes a built-in Close button:

```yuck
;  Source: owenrumney — github.com/owenrumney/squeaky-theme

; Generic click-toggle revealer with built-in close button
; Usage:
;   (clickbox :var show-cal :varname "show-cal"
;     (button :class "clock" "12:00")   ; child 0 — the trigger button
;     (cal-widget))                     ; child 1 — revealed content
(defwidget clickbox [var varname ?class ?duration ?transition]
  (box :class "${class} clickbox" :orientation "h" :space-evenly false
    (button :onclick "eww update ${varname}=${var ? false : true}"
      (children :nth 0))
    (revealer :reveal var
              :transition {transition ?: "slideleft"}
              :duration {duration ?: "500ms"}
      (box :class "${class}" :space-evenly false
        (children :nth 1)
        ; Built-in close button — always dismisses the revealer
        (button :onclick "eww update ${varname}=false" :class "close"
          (label :text "Close"))))))
```

**Usage notes:**
- `revealer-on-hover` and `clickbox` are drop-in wrappers — define the var/varname pair once and pass any content as children.
- `?transition` defaults to `"slideright"` for `revealer-on-hover` and `"slideleft"` for `clickbox` if not provided.
- `hovered-sign` is for icon/label swapping only — it does not manage the hover event itself; pair it inside an `eventbox`.
- The `children :nth` approach means call-site ordering matters: child 0 is always the trigger/icon, child 1 is always the revealed content.
