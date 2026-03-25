# eww Positioning & Spacing — Proven Techniques

Findings from controlled layout experiments comparing four techniques across
alignment, gap control, centering, and size behavior. Use this as the decision
guide any time a user asks to move, space, center, or align elements.

> For the underlying behavioral model of `halign`, `valign`, `hexpand`, and `vexpand`
> (what they are, how they interact, tug-of-war rules), see
> [eww-widgets/ALIGNMENT.md](../eww-widgets/ALIGNMENT.md).

---

## Quick Decision Guide

| Goal | Use | Avoid |
|------|-----|-------|
| Fixed gap between two elements | CSS `margin-left` on the second element | `:spacing` for large gaps |
| Push element to far right/bottom | T2: `(box :hexpand true)` spacer | `:spacing` with large values |
| Center an element in its row | T3: `:hexpand true` + `:halign "center"` on the element | `:space-evenly true` (expands, doesn't center) |
| Align element's edge to a sibling's edge | T3: wrap in a `(box :width N)` + `:hexpand true :halign "end"` | `:style "margin-left: Npx"` (fragile math) |
| Container padding (d1/d2) | CSS `padding` on the container | Inline `:style "padding: …"` |
| Colored accent bar at top of widget | Outer box + `.header` child + `.body` child with its own padding | Top padding + background tricks |

---

## The Four Techniques — Behavior Summary

### T1 — `:spacing N` on the row box

```lisp
(box :orientation "h" :spacing 6 :space-evenly false
  icon
  gauge)
```

**What it does:** Adds N pixels between every pair of adjacent children.

**Self-contained?** Only for small values. For large values (e.g., 200px) the box
requests more space than the window declares, causing the **window to grow and
potentially overflow the screen** — bidirectionally from its anchor point.

**Use when:** The gap is small and known (e.g., 6px icon-to-bar gap). Avoid
for positioning where the separation is a significant fraction of the window width.

**Critical failure mode:** `:space-evenly true` with a single child does NOT
center it — it expands the child to fill the full row width, making the
element appear as wide as the container.

---

### T2 — Empty `(box :hexpand true)` spacer

```lisp
; Push element to far right
(box :orientation "h" :space-evenly false
  icon
  (box :hexpand true)   ; absorbs all remaining space
  gauge)

; Center an element
(box :orientation "h" :space-evenly false
  (box :hexpand true)
  element
  (box :hexpand true))
```

**What it does:** The empty hexpand box absorbs all available space in the row,
pushing its siblings to the opposite edges. Two equal hexpand spacers produce
true centering.

**Self-contained?** Yes. The spacer works within the already-allocated window
width. The window never grows.

**Limitation:** Cannot achieve pixel-exact offsets (e.g., "align icon2's right
edge with gauge's right edge"). For that, use a `(box :width N)` fixed-width
spacer instead of hexpand — but note this introduces a hardcoded pixel value.

---

### T3 — `:hexpand true` + `:halign` on the element ✓ RECOMMENDED

```lisp
; Element takes all remaining space, content positioned within it
(box :hexpand true :orientation "h" :space-evenly false
  icon
  (box :class "bar" :hexpand true :halign "fill"))   ; bar fills rest

; Center
(box :hexpand true :orientation "h" :space-evenly false
  (box :class "element" :width 90 :height 28 :hexpand true :halign "center"))

; Align right edge of icon to gauge's right edge (gauge = 90px, icon = 28px)
(box :hexpand true :orientation "h" :space-evenly false
  (box :width 90 :space-evenly false          ; 90px container = gauge width
    (box :class "icon" :width 28 :height 28
         :hexpand true :halign "end")))        ; icon's right edge = 90px mark
```

**What it does:** `:hexpand true` gives the element all remaining allocated
space. `:halign` then positions the element's rendered content within that
allocation — without changing the element's visual size.

**Self-contained?** Yes. Always. The window never grows.

**Best for:** Progress bars that should fill remaining space, centering, pushing
to an edge, or aligning to a relative position via a sized wrapper box.

**Gap between icon and bar:** Use CSS `margin-left` on the expanding element
rather than `:spacing` on the parent. This keeps the gap in CSS and doesn't
affect the window's natural size:

```scss
.bar-track {
  margin-left: 18px;   /* gap between icon and bar */
}
```

---

### T4 — Inline `:style "margin-left: Npx"`

```lisp
(box :hexpand true :orientation "h" :space-evenly false
  (box :class "element" :width 90 :height 28
       :style "margin-left: 53px;"))
```

**What it does:** Applies a GTK CSS left margin directly via inline style.

**Self-contained?** Only if the parent row has `:hexpand true`. Without it,
the margin forces the window to grow (right-side overflow).

**Centering is fragile:** Centering requires `margin-left = (available_width - element_width) / 2`.
This math breaks if the window width, padding, or GTK internal spacing
differs from your assumptions. In practice this is unreliable.

**Use when:** You need a fixed, known offset that doesn't need to be
semantically derived from other elements. For everything else, prefer T3.

---

## Container Padding (d1 / d2)

```
|<-- d1 -->icon<-- gap -->bar<-- d2 -->|
```

- **d1 and d2** are controlled exclusively by the container's CSS `padding`.
- They are independent of the layout technique used for children.
- Changing padding does not break T2 or T3 layouts — both adapt automatically.
- T1 and T4 may require recalculating pixel values if padding changes.

```scss
.my-widget {
  padding: 12px;        /* d1 = d2 = 12px, uniform */
  padding: 12px 20px;   /* 12px top/bottom, 20px left/right */
}
```

---

## Colored Header Bar Pattern

Proven pattern for widgets with a colored accent bar at the top:

```lisp
(defwidget my-widget []
  (box :class "widget-outer" :orientation "v" :space-evenly false
    (box :class "widget-header")          ; accent bar, no padding
    (box :class "widget-body"             ; content area, has padding
         :orientation "v"
         :spacing 12
         :space-evenly false
      (content ...))))
```

```scss
.widget-outer {
  background: #2E3440;
  border-radius: 14px;
  padding: 0;                             /* no padding on outer */

  .widget-header {
    background: #D08770;                  /* accent color */
    min-height: 36px;
    border-radius: 14px 14px 0 0;        /* match outer top corners */
  }

  .widget-body {
    padding: 24px;                        /* all padding lives here */
  }
}
```

**Key:** The outer container has `padding: 0` so the header is flush with
the border. The body has the full padding. The header's top border-radius
must match the outer container's border-radius.

---

## Window Geometry — Sign Convention for Right-Anchored Windows

**Critical:** For `anchor "top right"` and `anchor "bottom right"`, the `:x`
coordinate is in screen space (positive = rightward). This means:

```lisp
; CORRECT — window's right edge is 20px INSIDE the screen
:geometry (geometry :x "20px" :y "20px" :width "220px" :anchor "top right")

; WRONG — window's right edge is 20px OUTSIDE the screen (20px clipped)
:geometry (geometry :x "-20px" :y "20px" :width "220px" :anchor "top right")
```

The same applies to `anchor "bottom right"`.

For left-anchored windows (`"top left"`, `"bottom left"`), positive `:x` is
also inward — consistent.

**Symmetric layout (widgets in all 4 corners, 20px from edges):**

```lisp
; Top-left
:geometry (geometry :x "20px"  :y "20px"  :width "220px" :anchor "top left")
; Top-right
:geometry (geometry :x "20px"  :y "20px"  :width "220px" :anchor "top right")
; Bottom-left
:geometry (geometry :x "20px"  :y "20px"  :width "220px" :anchor "bottom left")
; Bottom-right
:geometry (geometry :x "20px"  :y "20px"  :width "220px" :anchor "bottom right")
```

All four use positive `"20px"` — the direction of the offset is determined
by the anchor, not by the sign of the value.

---

## Size-Forcing vs Self-Contained — Critical Distinction

| Technique | Window grows? | Survives `eww reload`? | Restores with reload? |
|-----------|:-------------:|:----------------------:|:---------------------:|
| T1 `:spacing` large | YES — bidirectional | Yes | NO — must close+reopen |
| T2 hexpand spacer | Never | Yes | N/A |
| T3 `:hexpand` + `:halign` | Never | Yes | N/A |
| T4 `:style margin` | YES — one direction | Yes | NO — must close+reopen |

When T1 or T4 expand a window beyond its declared `:width`, reverting the
config and running `eww reload` does NOT shrink the window back. The window
must be explicitly closed and reopened: `eww close name && eww open name`.

T2 and T3 never change the window size regardless of values used.

---

## Widget Overlap Detection — Coordinate System

### Why This Matters

Desktop widgets have invisible bounding boxes that can block clicks on other widgets. Before debugging "broken click areas" or "visual artifacts," map all active widget positions to detect collisions.

### Anchor-Relative Coordinate System

Each `defwindow` has an `:anchor` that defines its 0,0 reference point. X and Y offsets move the widget **inward** from that corner:

| Anchor | 0,0 corner | X+ direction | Y+ direction |
|--------|-----------|-------------|-------------|
| `"top left"` | Top-left | Right → | Down ↓ |
| `"top right"` | Top-right | Left ← | Down ↓ |
| `"bottom left"` | Bottom-left | Right → | Up ↑ |
| `"bottom right"` | Bottom-right | Left ← | Up ↑ |

### Example: Detecting Overlap

Widget A (`activate-linux`): anchor `bottom right`, x=-122px, y=71px, width=787px
Widget B (`mongo-tunnel`): anchor `bottom right`, x=68px, y=200px

Both anchored bottom-right, so X increases leftward. Widget A's 787px width extends far into the screen, potentially covering Widget B even though they have different x/y offsets.

### Quick Check Procedure

```bash
# 1. List all open windows
eww active-windows

# 2. Extract geometry for each
for w in $(eww active-windows 2>/dev/null | cut -d: -f1); do
  echo "=== $w ==="
  grep -A6 "defwindow $w" ~/.config/eww/widgets/*.yuck 2>/dev/null | \
    grep -E "anchor|:x |:y |:width|:height|stacking"
done

# 3. Check stacking layers — higher layers block lower ones
# bottom < bg < fg < overlay
```
