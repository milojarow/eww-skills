# Alignment & Expand Behavior — eww-widgets

The four properties `halign`, `valign`, `hexpand`, `vexpand` need a mental model to use correctly.
All other universal properties (`:class`, `:visible`, `:active`, etc.) are self-explanatory.

> For practical techniques to achieve specific layout goals (centering, push-to-edge, spacing),
> see [eww-patterns/POSITIONING.md](../eww-patterns/POSITIONING.md).

---

## Overview

eww widget layout is powered by GTK's allocation system. When a parent widget (e.g., a `box`)
lays out its children, it allocates space to each child. The child then decides how to use that
allocated space. These four properties control both sides of that interaction:

- **hexpand / vexpand** — does this widget request extra allocated space from its parent?
- **halign / valign** — how does this widget use the space its parent allocates to it?

---

## halign and valign — Alignment Values

Both properties accept the same five values:

| Value | Effect |
|---|---|
| `"fill"` | Stretch to fill all allocated space **(default)** |
| `"start"` | Align to the left edge (h) or top edge (v) |
| `"end"` | Align to the right edge (h) or bottom edge (v) |
| `"center"` | Center within allocated space |
| `"baseline"` | Align along the text baseline — for mixed-height inline elements |

**Mental model:** `halign`/`valign` control where the widget sits *within the space its parent
gives it*, not within the screen or window. If the parent gives the widget no extra space beyond
what it needs, then `halign` has no visible effect — the widget is already as small as it can be.

```yuck
; halign: "start" only has visible effect if the widget has room to move
(label :text "hello"
       :halign "start"   ; no visible effect — label is already minimal size
       :hexpand false)   ; (default) — label only takes what it needs

; halign: "start" IS visible here — hexpand gives it room, then start pulls it left
(label :text "hello"
       :halign "start"
       :hexpand true)
```

---

## hexpand / vexpand — Expand Behavior

### hexpand: false (default)

The widget takes only as much horizontal space as its content needs.
If content grows, the widget grows with it. The widget never exceeds its content size.

```yuck
; ✅ CORRECT — label stays exactly as wide as "CPU: 42%"
(label :text "CPU: 42%" :hexpand false)

; ✅ CORRECT — as text grows, so does the label
(label :text {some-long-dynamic-string} :hexpand false)
```

```yuck
; ❌ WRONG expectation — :width cannot shrink the widget below content size
(label :text "A very long label that overflows" :width 50)
; Result: label ignores :width and expands to fit text
```

---

### hexpand: true — One Expanding Child

The widget requests all remaining horizontal space in the parent box, after non-expanding
siblings have been placed at their natural sizes.

```yuck
; ✅ CORRECT — spacer box claims all leftover space, pushing labels to edges
(box :orientation "h" :space-evenly false
  (label :text "left")
  (box :hexpand true)      ; spacer — eats all remaining space
  (label :text "right"))

; ✅ CORRECT — expanding label fills remaining space
(box :orientation "h" :space-evenly false
  (label :text "Title")
  (label :text {dynamic-value} :hexpand true))
```

---

### hexpand: true on Multiple Siblings — Tug of War

When more than one sibling has `:hexpand true`, they compete for available space.
**The widget with the larger content wins more space. Equal content = equal split (stalemate).**

```yuck
; ❌ PROBLEM — both expand: shorter one loses
(box :orientation "h"
  (label :text "short"             :hexpand true)   ; gets less space
  (label :text "a much longer text" :hexpand true))  ; wins more space

; ✅ CORRECT — use a single spacer instead; deterministic result
(box :orientation "h" :space-evenly false
  (label :text "short")
  (box :hexpand true)               ; one spacer does the job
  (label :text "a much longer text"))
```

> **Why this matters:** If you expect a stalemate (both get equal space) but one label
> grows dynamically, the layout will shift unpredictably at runtime.

---

## halign + hexpand Together

This is the most powerful combination: give the widget all available space with `hexpand`,
then use `halign` to position the widget within that space.

```yuck
; Push element to the RIGHT end of its parent
(label :text "status"
       :hexpand true
       :halign "end")

; Push element to the LEFT (start)
(label :text "title"
       :hexpand true
       :halign "start")

; Center an element within all available space
(label :text "centered"
       :hexpand true
       :halign "center")
```

```yuck
; ❌ WRONG — halign alone does nothing if the widget hasn't expanded
(box :orientation "h"
  (label :text "I want to be on the right"
         :halign "end"))         ; no effect — label is at its natural size
; Result: label stays at the left, same as always

; ✅ CORRECT — hexpand gives room, halign places it
(box :orientation "h"
  (label :text "I want to be on the right"
         :hexpand true
         :halign "end"))
; Result: label text appears at the right edge of the box
```

---

## Common Mistakes

### 1. Using halign without hexpand
`halign` has no visible effect unless the widget has extra allocated space.
Without `hexpand: true`, the widget is already at its minimum size — nothing to align.

### 2. Expecting equal split from two hexpand siblings
Two siblings both with `:hexpand true` do NOT reliably split space 50/50.
The one with more content wins. Use a single `(box :hexpand true)` spacer instead.

### 3. Using :width or :height to shrink a widget
```yuck
; ❌ WRONG — :width cannot force a widget smaller than its content
(label :text "long text that won't shrink" :width 20)
```
`:width` and `:height` set a *minimum*, not a maximum. Content always wins.

### 4. Forgetting valign for vertical alignment
The same rules apply vertically. A label inside a tall box won't center itself vertically
without `:valign "center"`. And it won't have room to move unless `:vexpand true`.

```yuck
; ✅ CORRECT — vertically centered label in a fixed-height box
(box :height 48
  (label :text "centered"
         :vexpand true
         :valign "center"))
```

---

## See Also

- **Practical layout techniques** (push to edge, centering, spacing): [POSITIONING.md](../eww-patterns/POSITIONING.md)
- **box / centerbox layout behavior** in depth: [CONTAINERS.md](./CONTAINERS.md)
- **CSS-based spacing** (margin, padding, gap): `eww-styling` skill
