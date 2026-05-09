# Native First — Don't Reinvent What eww Already Draws

eww ships native widgets that draw their own visuals via Cairo: `progress`,
`circular-progress`, `graph`, `scale`, plus the magic system variables
(`EWW_CPU`, `EWW_RAM`, `EWW_BATTERY`, `EWW_DISK`, `EWW_NET`, `EWW_TEMPS`)
that update reactively without any user script. Before reaching for an
external renderer (Python+Cairo, ImageMagick, conky, polybar-msg…), check
what eww already provides — the native path is almost always smaller,
faster, and more maintainable.

This file is a pattern document, not a rant. The case study at the bottom
is the cautionary tale that motivated writing it.

---

## The principle

**For visual primitives that eww supports natively (rings, bars, graphs,
sliders), drive them directly from a magic variable or a tiny `defpoll`.
Reach for an external renderer only when the visual genuinely cannot be
expressed in eww — and weigh the cost first.**

Native widgets are drawn by the eww daemon itself using its bundled Cairo
backend. Their state is reactive: when the underlying variable changes,
eww redraws automatically. There is no subprocess, no temp file, no
cache-busting trick.

External rendering means: every refresh tick, eww forks a script, the
script forks more processes (Python, Cairo, ImageMagick), reads sensors
or files, writes a PNG to `/tmp`, and updates a path variable so eww
loads the new image. Each step has a cost, and they add up to seconds
of CPU per minute just to keep an idle dashboard alive.

---

## How to choose

When a user says "I want to show CPU usage / RAM / GPU temp / battery /
network speed", first ask: which eww widget naturally represents this?

| Visual goal | Native widget | Driver |
|---|---|---|
| Ring meter (0–100%) | `circular-progress` | magic var or `defpoll` integer |
| Linear bar | `progress` | magic var or `defpoll` integer |
| Time-series sparkline | `graph` | magic var or `defpoll` JSON |
| Slider with input | `scale` | `defvar` + `:onchange` |
| Live numeric label | `label` | magic var or `defpoll` |
| Color tier (ok/warn/crit) | dynamic CSS class on the widget | ternary on the value |

Only fall back to image rendering when:

- The shape isn't supported (gauges with custom needles, multi-axis charts)
- You need pixel-perfect typography that GTK CSS can't reach
- You're displaying actual artwork (album art, screenshots, icons)

In those cases the image approach is appropriate. For everything else,
native is the answer.

---

## Why external rendering is expensive

Each refresh tick of an external-renderer gauge incurs the following
fixed costs, regardless of how clever the script is:

1. `defpoll` forks a shell to run the command (≈10ms).
2. The shell forks a renderer (Python, awk, jq, cairo, …) — each is its
   own fork+exec (≈20–50ms for Python with cairo because of import time).
3. The renderer reads a system source (`sensors`, `/proc/meminfo`,
   `nvidia-smi`, etc.).
4. The renderer writes a PNG/SVG to disk (even on tmpfs there's I/O,
   metadata updates, and the write barrier).
5. **Path alternation hack**: GTK caches images by path. If your script
   writes to the same path every tick, eww shows the *cached* image and
   never refreshes. The standard workaround is to alternate between two
   paths (`gauge-0.png`, `gauge-1.png`) and update a `defpoll` variable
   to the new path. This works but it's a code smell — it means you're
   fighting the framework.
6. eww invalidates its image cache, loads the new PNG, copies it into
   the GTK pixbuf cache, and triggers a redraw.

Three gauges × every 3 seconds ≈ one fork per second of pure overhead
for a screen that doesn't even change pixel-for-pixel between most
refreshes.

A `circular-progress` driven by a magic variable does none of this. The
daemon already knows the value (it polls `EWW_CPU` itself, internally,
once per 2s). When the value changes, the next frame redraws the arc.
Zero subprocesses. Zero I/O. Zero path tricks.

---

## Pattern: native ring driven by a magic var

```yuck
(defwidget ram-gauge []
  (overlay
    (circular-progress :class "gauge ${EWW_RAM.used_mem_perc <= 60 ? 'safe'
                                       : EWW_RAM.used_mem_perc <= 85 ? 'warning'
                                       : 'danger'}"
                       :value {EWW_RAM.used_mem_perc}
                       :thickness 14
                       :start-at 75 :clockwise true
                       :width 180 :height 180)
    (box :orientation "v" :halign "center" :valign "center"
      (label :class "gauge-icon"  :text "")
      (label :class "gauge-value" :text "${round(EWW_RAM.used_mem_perc, 0)}%")
      (label :class "gauge-name"  :text "RAM"))))
```

Color tiers are a CSS class toggled by a ternary on the value. No script
maps thresholds to colors — eww does it inline.

---

## Pattern: native ring with a tiny defpoll fallback

When the metric isn't in a magic var (e.g. NVIDIA GPU temperature), use
a one-line `defpoll` that emits the integer and feed that into the ring:

```yuck
(defpoll gpu-temp :interval "5s" :initial "0"
  "nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits 2>/dev/null || echo 0")

(circular-progress :value gpu-temp :thickness 14 :start-at 75 :clockwise true ...)
```

Notice what's missing: no Python, no Cairo, no `/tmp` files, no path
alternation. The poll fork is the only cost, and it's once per 5s.

---

## Anti-pattern (case study): PNG-rendered gauges in `/tmp`

This is what the temps widget in this dotfile config looked like before
it was migrated. Documenting it here because the failure mode is so
educational.

### What the code was doing

```yuck
; widgets/temps.yuck (BEFORE)
(defpoll cpu-gauge-path :interval "3s" :initial ""
  "~/.config/eww/scripts/cpu-gauge.sh")

(defpoll gpu-gauge-path :interval "3s" :initial ""
  "~/.config/eww/scripts/gpu-gauge.sh")

(defpoll ram-gauge-path :interval "3s" :initial ""
  "~/.config/eww/scripts/ram-gauge.sh")

(defwidget temps []
  (box :orientation "v"
    (image :path cpu-gauge-path :image-width 200 :image-height 210)
    (image :path gpu-gauge-path :image-width 200 :image-height 210)
    (image :path ram-gauge-path :image-width 200 :image-height 210)))
```

```bash
# scripts/cpu-gauge.sh (BEFORE)
TEMP=$(sensors -j | python3 -c "import json,sys; print(int(json.load(sys.stdin)['coretemp-isa-0000']['Package id 0']['temp1_input']))")

# Alternate between two files so eww detects the path change and reloads.
N=$(( ($(cat /tmp/eww-cpu-gauge-n 2>/dev/null || echo 0) + 1) % 2 ))
echo "$N" > /tmp/eww-cpu-gauge-n

OUT="/tmp/eww-cpu-gauge-${N}.png"
python3 ~/.config/eww/scripts/gauge-render.py "$TEMP" 0 100 "CPU" "$OUT" "°C" ""
echo "$OUT"
```

`gauge-render.py` was a 100+ line Python+Cairo program that drew a
speedometer-style 300° arc with a label, an icon, and tick coloring,
saving it as a 200×210 RGBA PNG.

### Why the author probably did it

A speedometer with a 300° arc and a needle is achievable in
`circular-progress` with `:start-at` but not pixel-identical to a
hand-drawn Cairo speedometer. The author wanted a specific look that
felt easier to draw in Python than in eww+CSS, and reached for the
familiar tool.

That's a defensible choice in isolation. The hidden cost is what makes
it an anti-pattern.

### The hidden cost

- **Three scripts every three seconds** — bash fork, python fork, cairo
  import, sensors call, PNG render, file write.
- **Six PNG files in `/tmp`** held in the GTK pixbuf cache because of
  the path-alternation trick.
- **No reactivity**. The widget visually lags 3s + render time behind
  the actual metric.
- **The path alternation itself is fighting eww**. It's there because
  the natural code (write to the same path) doesn't work with GTK's
  image cache. If you're alternating paths to defeat a cache, you've
  already left the happy path.

In aggregate this is roughly 1 fork per second of CPU activity dedicated
to drawing three rings on an idle desktop. On any machine — but
especially on laptops or low-power machines — that's wasted budget.

### The migration

```yuck
; widgets/temps.yuck (AFTER)
(defpoll cpu-temp :interval "3s" :initial "0"
  "sensors coretemp-isa-0000 2>/dev/null | awk '/Package id 0:/{gsub(\"[+°C]\",\"\",$4); print int($4); exit}'")
(defpoll ram-pct  :interval "3s" :initial "0"
  "awk '/MemTotal/{t=$2}/MemAvailable/{a=$2}END{printf \"%d\", (t-a)/t*100}' /proc/meminfo")
(defpoll gpu-temp :interval "5s" :initial "0"
  "nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits 2>/dev/null || echo 0")

(defwidget temps-gauge [val unit name icon thresh1 thresh2]
  (overlay
    (circular-progress :class "temps-gauge ${val <= thresh1 ? 'safe' : val <= thresh2 ? 'warning' : 'danger'}"
                       :value val :thickness 14 :start-at 75 :clockwise true
                       :width 180 :height 180)
    (box :orientation "v" :halign "center" :valign "center"
      (label :class "gauge-icon"  :text icon)
      (label :class "gauge-value" :text "${val}${unit}")
      (label :class "gauge-name"  :text name))))

(defwidget temps []
  (box :class "temps" :orientation "v" :spacing 12
    (temps-gauge :val cpu-temp :unit "°C" :name "CPU" :icon "" :thresh1 60 :thresh2 80)
    (temps-gauge :val ram-pct  :unit "%"  :name "RAM" :icon "" :thresh1 60 :thresh2 85)
    (temps-gauge :val gpu-temp :unit "°C" :name "GPU" :icon "󰺷" :thresh1 60 :thresh2 80)))
```

What changed:

- **3 scripts → 3 one-line shell pollers**. No Python. No Cairo. No PNG.
- **6 PNG files in `/tmp` → 0**.
- **Path alternation hack → gone** (not needed when eww draws the visual).
- **Color tiers → CSS classes toggled by a ternary on the value.**
- **Net diff: −87 lines of code.**

The look is slightly different (a clean ring instead of a speedometer
with a needle) — and that's the only thing the migration loses. For
this user, that tradeoff was a clear win. For another user it might
not be. The point isn't that PNGs are forbidden — it's that they are
expensive, and the cost should be paid consciously.

---

## Decision checklist

Before adding a new "gauge that calls a script that renders an image":

1. Does eww have a native widget for this shape?
   `circular-progress`, `progress`, `graph`, `scale` cover most cases.
2. Is the value reachable from a magic var (`EWW_CPU`, `EWW_RAM`,
   `EWW_BATTERY`, `EWW_DISK`, `EWW_NET`, `EWW_TEMPS`)? If yes, no script.
3. If the value isn't in a magic var, can a one-line `defpoll` get it?
4. Are you alternating output paths to defeat a cache? That's the smoke
   alarm — back up and use a native widget.

If the answer to (1) is "no" or to "is the visual genuinely
unrepresentable in CSS+native widgets" is "yes", then the image approach
is correct. Otherwise it's overhead.

---

## Integration with other skills

- `eww-widgets` — full reference for `circular-progress`, `progress`,
  `graph`, `scale`, and the magic variables.
- `eww-patterns/POSITIONING.md` — for laying out multiple rings.
- `eww-patterns/CLICK_HANDLERS.md` — sister doc on the cost of `eww`
  CLI roundtrips, with the same "minimize the work per tick" theme.
