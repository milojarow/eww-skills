# Click Handlers — Latency Patterns

When users click eww buttons rapidly and feel like clicks are "dropped",
the cause is almost always **cumulative `eww` CLI cost in the click handler
script**, not GTK input drops. This file documents the cost model and the
patterns that keep clicks responsive.

---

## The Cost Model

Every `eww` CLI invocation is:

1. fork() + exec() the binary (~10-30ms)
2. Connect to the daemon's UNIX socket
3. Send the command, wait for response
4. Daemon serializes the request against other in-flight calls
5. Return value, exit

Empirically each call costs **~50-150ms** on a modern desktop, more under load.
A click handler that runs 4 `eww` calls takes ~300-500ms before it can accept
the next click. Rapid clicks queue up and feel dropped.

---

## Anti-Pattern: Refetching State Every Click

```bash
#!/usr/bin/env bash
# next.sh — increments a wrapping index. THIS IS SLOW.

idx=$(eww get my-index 2>/dev/null)            # eww call #1
list=$(eww get my-list 2>/dev/null)            # eww call #2
len=$(echo "$list" | python3 -c "import sys,json; print(len(json.load(sys.stdin)))")
new=$(( (idx + 1) % len ))
eww update my-index="$new"                     # eww call #3
```

Three `eww` calls + one python invocation per click. ~300-500ms latency.
Five rapid clicks → ~2 seconds of queued work. User experiences "dropped" clicks.

---

## Pattern 1: Cache Read-Only State in Tempfiles

When a value rarely changes (list length, fixed metadata), cache it to a
tempfile when it's produced and read it directly in the click handler — zero
IPC cost.

**Producer side** (the `deflisten` or initialization script):

```bash
emit_list() {
    local json
    json=$(build_json)
    echo "$json"   # to deflisten stdout

    # Cache derived values for click handlers.
    echo "$json" | jq 'length' > /tmp/eww-my-count
}
```

**Consumer side** (the click handler):

```bash
#!/usr/bin/env bash
# next.sh — same logic, ~50ms instead of ~400ms.

[ -f /tmp/eww-my-count ] || exit 0
count=$(</tmp/eww-my-count)        # instant (no fork)

idx=0
[ -f /tmp/eww-my-index ] && idx=$(</tmp/eww-my-index)   # instant

new=$(( (idx + 1) % count ))
echo "$new" > /tmp/eww-my-index    # instant
eww update my-index="$new"         # only eww call
```

One `eww` call instead of three. Bash `$(<file)` is in-process — no fork.

---

## Pattern 2: Batch Multiple Updates Into One Call

`eww update` accepts multiple `var=value` pairs in a single invocation. Use it
whenever you'd otherwise call `eww update` twice in a row.

```bash
# ❌ Two roundtrips
eww update foo=1
eww update bar=2

# ✅ One roundtrip
eww update foo=1 bar=2
```

This matters most in handlers that toggle several state variables at once
(close + reset preview + record applied, etc.).

---

## Pattern 3: Fire-and-Forget Side Effects

If the handler does blocking work (network, disk, swaymsg) that the user
doesn't need to wait for, background it so the script returns immediately
and the next click is unblocked.

```bash
# ❌ User waits for swaymsg roundtrip before next click can register
swaymsg "output * bg \"$path\" fill"
eww update applied-index="$idx"

# ✅ Background the slow side effect
( swaymsg "output * bg \"$path\" fill" >/dev/null 2>&1 ) &
eww update applied-index="$idx"
```

Don't background the `eww update` itself — it's fast and the UI needs the
state change to feel instant.

---

## Pattern 4: Skip Validation in Click Path

Validation belongs in the producer (the deflisten that builds the data) or in
an init step — not the click handler. By the time a button is clickable, the
data should be valid.

```bash
# ❌ Validates on every click
list=$(eww get my-list)
echo "$list" | python3 -c "import sys,json; assert json.load(sys.stdin), 'empty'"
# ... then act

# ✅ Trust the cache, no-op if missing
[ -f /tmp/eww-my-count ] || exit 0
count=$(</tmp/eww-my-count)
[ "$count" -gt 0 ] || exit 0
# ... then act
```

---

## Diagnostic — Is The Handler Slow?

Time a single click-handler invocation in isolation:

```bash
time ~/.config/eww/scripts/my-handler.sh next
```

| Result | What it means |
|--------|---------------|
| < 100ms | Fine. User-perceived clicks won't drop. |
| 100-300ms | Borderline. Rapid clicks (3+/sec) start queuing. |
| > 300ms | Will feel laggy. Apply the patterns above. |

If a single handler is < 100ms but rapid clicks still feel dropped, the daemon
itself is overloaded. Check `pgrep -af 'eww (open|close|update)'` for hung
CLI processes piling up.

---

## Why Not Use a Single Long-Running Script?

A long-lived companion process (instead of one-shot scripts per click) could
avoid fork+exec entirely — just send commands over a FIFO. eww supports this
via `deflisten`, but the response path back from script to UI still requires
`eww update` calls, which are still IPC.

In practice the tempfile-cache pattern is simpler and gets within ~10ms of an
in-process implementation. Use it first; reach for a daemon-style helper only
if profiling shows the single `eww update` is the bottleneck.

---

## Integration with Other Skills

- **eww-yuck** — `defvar`, `deflisten`, `eww update` syntax
- **eww-patterns/DATA_PATTERNS.md** — emitting JSON from deflisten scripts
- **eww-troubleshooting/COMMON_ERRORS.md** — daemon-overload symptoms and recovery
