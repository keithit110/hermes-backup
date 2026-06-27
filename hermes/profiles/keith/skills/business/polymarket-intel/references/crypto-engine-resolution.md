# Crypto Engine Resolution — Best-Guess P/L Pattern

## Problem

Polymarket's Gamma API takes 10-15+ minutes to mark BTC 5-min markets as `closed: true` after the window ends. During that gap, the engine had no P/L to display — trades showed `closed_window_expired` with `pnl_pct=0.0`.

Root cause: `_check_market_resolution()` returns `None` when `ev.get("closed")` is `False`. The engine retries every 5 seconds via `resolve_pending_trades()`, but Polymarket is just slow.

## Solution

Compute best-guess P/L **immediately** from BTC direction at window close, using the Binance price feed the engine already has. Polymarket resolution still runs as a safety-net verification afterward.

### Implementation in `_close_old_window_positions()` (crypto_engine.py)

```python
def _close_old_window_positions(old_slug: str):
    conn = sqlite3.connect(DB)
    now = now_utc().isoformat()
    
    # Determine immediate outcome from BTC direction
    with state.lock:
        btc_end = state.btc_mid
        btc_start = state.window_start_price
    winner = "Down" if btc_end < btc_start else "Up"
    
    # Update each open trade with best-guess P/L
    trades = conn.execute(
        "select id, entry_cost, details_json from paper_trades "
        "where source='crypto_5m_engine' and event_slug=? and status='open'",
        (old_slug,),
    ).fetchall()
    
    for t in trades:
        trade_id, entry_cost, details_json = t
        details = json.loads(details_json)
        side = details.get("side", "")
        resolved_price = 1.0 if side.upper() == winner.upper() else 0.0
        pnl_pct = (resolved_price - entry_cost) / entry_cost if entry_cost > 0 else 0.0
        new_status = "closed_resolved_win" if pnl_pct > 0 else "closed_resolved_loss"
        conn.execute(
            "update paper_trades set status=?, closed_at=?, current_value=?, pnl_pct=? where id=?",
            (new_status, now, resolved_price, pnl_pct, trade_id),
        )
    
    # Still queue for official resolution (safety net)
    conn.execute(
        "insert or ignore into crypto_engine_state(key,value) values(?,?)",
        (f"pending_resolve:{old_slug}", now),
    )
    conn.commit()
    conn.close()
```

### Key detail: `resolve_pending_trades()` must match all closed statuses

Since trades now start as `closed_resolved_win` or `closed_resolved_loss` (best-guess) rather than `closed_window_expired`, the resolution query must use `status like 'closed%'`:

```python
trades = conn.execute(
    "select id, kind, entry_cost from paper_trades "
    "where source='crypto_5m_engine' and event_slug=? and status like 'closed%'",
    (slug,),
).fetchall()
```

## Manual fix for stuck trades

When trades are already stuck in `closed_window_expired` with P/L=0:

```python
# Get the slug's window timestamp, determine direction from trade reason field
# (e.g., "BTC -0.13%" means DOWN won)
# Then update directly:
conn.execute('update paper_trades set status=?, current_value=?, pnl_pct=? where id=?',
             (new_status, resolved_price, pnl, trade_id))
```

## Verification

After fix, trades should show non-zero P/L immediately on the dashboard when a window closes. Example output:
```
[CLOSE] #215 side=DOWN btc_start=60379.92 btc_end=60299.42 winner=Down pnl=+23.46%
[CLOSE] #216 side=UP   btc_start=60379.92 btc_end=60299.42 winner=Down pnl=-100.00%
```
