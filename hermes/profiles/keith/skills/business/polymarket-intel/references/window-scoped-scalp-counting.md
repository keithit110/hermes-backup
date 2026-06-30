# Window-scoped scalp counting — why global counting broke entries

## Problem (2026-06-29)

`evaluate_scalp()` used `count_open_scalps(conn)` which counted ALL open scalp positions across ALL windows via:
```sql
SELECT COUNT(*) FROM paper_trades WHERE source='midpoint_scalp' AND status='open'
```

When a DOWN position from the previous window was still open (held as loser, waiting for resolution), it consumed one of the `SCALP_MAX_OPEN=3` slots. When the new window started and UP was opened (now 2 total), the DOWN check saw `open_count=2` but the old DOWN was still open (now 3 total actual). If the old position hadn't closed yet, the new DOWN was blocked.

Additionally, `open_count` was computed ONCE before both UP and DOWN checks. If UP opened, the stale count could incorrectly suggest there was room for DOWN (or vice versa when close to the limit).

## Fix

### 1. Per-window counting

```python
def count_open_scalps(conn, event_slug=None):
    if event_slug:
        row = conn.execute(
            "SELECT COUNT(*) FROM paper_trades WHERE source=? AND status='open' AND event_slug=?",
            (SCALP_SOURCE, event_slug),
        ).fetchone()
    else:
        row = conn.execute(
            "SELECT COUNT(*) FROM paper_trades WHERE source=? AND status='open'",
            (SCALP_SOURCE,),
        ).fetchone()
    return row[0] if row else 0
```

### 2. Recompute between sides

```python
# UP check: uses window_open
if ... not has_scalp_position(conn, event_slug, "UP"):
    if window_open >= SCALP_MAX_OPEN:  # uses current-window count
        SKIP
    else:
        OPEN_UP
        window_open += 1

# Recompute for DOWN check
window_open = count_open_scalps(conn, event_slug=event_slug)

# DOWN check: fresh count reflects UP if it was just opened
if ... not has_scalp_position(conn, event_slug, "DOWN"):
    if window_open >= SCALP_MAX_OPEN:
        SKIP
    else:
        OPEN_DOWN
```

### 3. Safety logging

`count_open_scalps_other_windows()` tracks stale positions from old windows:
```python
def count_open_scalps_other_windows(conn, event_slug):
    row = conn.execute(
        "SELECT COUNT(*) FROM paper_trades WHERE source=? AND status='open' AND event_slug != ?",
        (SCALP_SOURCE, event_slug),
    ).fetchone()
    return row[0] if row else 0
```

Log output shows both: `window=N/M global=N` so you can see if old windows are accumulating positions that need manual cleanup.

## Verification

After the fix, each new window opens BOTH UP and DOWN independently:
```
Window 1782693600 → NEW:
  #1234 UP  @ 0.53 (live, 5 shares)
  #1235 DOWN @ 0.48 (live, 5 shares)
```

The previous window's DOWN (closed as loss) didn't block the new window's DOWN entry.
