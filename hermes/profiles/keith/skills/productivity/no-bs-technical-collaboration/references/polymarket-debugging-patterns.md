# Polymarket Debugging & Strategy Analysis Patterns

## Data corruption recovery

When all trades for a source show `current_value=1.0` with status `closed_resolution_assumed` (scanner-only status), the scanner corrupted the engine's data.

### Recovery procedure

1. Query affected slugs from DB:
```sql
SELECT DISTINCT event_slug FROM paper_trades 
WHERE status='closed_resolution_assumed' AND kind LIKE 'crypto_5m%'
```

2. Query Gamma API for each slug to get the true winner:
```python
url = f"https://gamma-api.polymarket.com/events/slug/{slug}"
ev = json.loads(response)
# Parse outcomePrices: ["1", "0"] means first outcome won, ["0", "1"] means second
for m in ev["markets"]:
    outcomes = json.loads(m["outcomes"]) if isinstance(m["outcomes"], str) else m["outcomes"]
    prices = json.loads(m["outcomePrices"]) if isinstance(m["outcomePrices"], str) else m["outcomePrices"]
    for outcome, price in zip(outcomes, prices):
        if str(price) == "1":
            winner = outcome
```

3. Update each trade:
```python
side = json.loads(details_json)["side"]
resolved_price = 1.0 if side.upper() == winner.upper() else 0.0
pnl = (resolved_price - entry_cost) / entry_cost
new_status = "closed_resolved_win" if pnl > 0 else "closed_resolved_loss"
```

## Engine crash: AttributeError on state field (silent stuck-trades)

**Symptom**: Open crypto trades never close, even though windows expired hours ago. New trades keep opening but old ones accumulate.

**Root cause**: `_close_old_window_positions()` references `state.btc_mid` but `EngineState` only has `btc_bid` and `btc_ask`. The AttributeError crashes every evaluation cycle inside the `except` block, preventing both `evaluate_and_act()` and `resolve_pending_trades()` from running.

**Detection**:
```bash
docker logs polymarket-intel-crypto 2>&1 | grep "AttributeError"
# Or check for open trades past end_date:
sqlite3 data/polymarket_intel.sqlite \
  "SELECT COUNT(*) FROM paper_trades WHERE kind LIKE 'crypto_5m%' AND status='open' AND end_date < datetime('now')"
```

**Fix — two parts**:

1. Fix the attribute reference:
```python
# WRONG:
btc_end = state.btc_mid

# RIGHT:
with state.lock:
    btc_bid = state.btc_bid
    btc_ask = state.btc_ask
    btc_start = state.window_start_price
btc_end = (btc_bid + btc_ask) / 2 if btc_bid > 0 and btc_ask > 0 else btc_start
```

2. Add a safety-net function that runs BEFORE any new evaluation, closing ALL open trades for expired windows:
```python
def _close_all_expired_windows():
    """Close any open crypto trades whose window has expired (safety net)."""
    conn = sqlite3.connect(DB)
    now = now_utc().isoformat()
    rows = conn.execute(
        "select id, event_slug, entry_cost, details_json from paper_trades "
        "where source='crypto_5m_engine' and status='open' and end_date < ?",
        (now,),
    ).fetchall()
    for t in rows:
        # Use BTC direction as best guess, queue for official Gamma resolution
        ...
        conn.execute("update paper_trades set status=?, closed_at=?, current_value=?, pnl_pct=? where id=?", ...)
```

Call `_close_all_expired_windows()` at the TOP of `evaluation_timer()`, before `refresh_token_ids()`. This ensures even if some other part crashes, the safety net catches stragglers on the next cycle.

## Strategy P/L analysis

When Keith asks whether a strategy is working, always compute from actual trade data, not memory or intuition.

### Pair-level P/L for hedged strategies

BTC directional + hedge pairs are ONE combined bet per slug. Compute:

```sql
SELECT event_slug,
  SUM(entry_cost) as deployed,
  SUM(current_value) as returned,
  (SUM(current_value) - SUM(entry_cost)) / SUM(entry_cost) * 100 as net_pnl_pct
FROM paper_trades
WHERE kind LIKE 'crypto_5m%' AND status LIKE 'closed_resolved%'
GROUP BY event_slug
ORDER BY net_pnl_pct;
```

Individual leg percentages (+614% on a $0.14 hedge) are mathematically true but MEANINGLESS in isolation. The pair-level net is the only number that matters. A hedge leg that shows +614% next to a DIR leg at -100% means the pair net is ~+2% — the big percentage is an artifact of a tiny denominator.

### Strategy comparison pattern (with-hedge vs without-hedge)

When asked "should I do X or Y?", compute both scenarios from actual data:

1. Query all closed trades grouped by slug
2. Compute scenario A (with hedge): `SUM(dir_entry + hedge_entry)`, `SUM(dir_return + hedge_return)`
3. Compute scenario B (directional only): `SUM(dir_entry)`, `SUM(dir_return)` — ignore hedge legs
4. Present side-by-side in a simple table:

```
| Strategy | Deployed | Returned | Net $ | Net % |
|----------|----------|----------|-------|-------|
| With hedge | $36.27 | $33.00 | -$3.27 | -9.0% |
| Dir only   | $30.62 | $30.00 | -$0.62 | -2.0% |
```

Key insight: the hedge costs money in every window where the DIR wins (hedge premium is dead money), but saves the pair in windows where the DIR loses. The net effect = `(hedge_premiums_paid) - (dir_losses_saved)`. If this is negative, the hedge is a net drain.

### Why solo DIR windows lose (and how to detect them)

Solo DIR windows = no hedge could be placed because `dir_entry + min_possible_hedge > 0.98`. The market is heavily skewed against the engine's direction (market says 15-25% chance).

In these windows, BTC typically moves in the engine's direction during entry (momentum) but REVERSES before window close. The engine has no structural edge — it's gambling on momentum continuation.

To check: `SELECT event_slug FROM paper_trades WHERE kind LIKE 'crypto_5m%' GROUP BY event_slug HAVING COUNT(*) = 1 AND SUM(CASE WHEN pnl_pct > 0 THEN 1 ELSE 0 END) = 0`

## Scanner-engine isolation rule

The scanner and crypto engine are separate processes sharing `paper_trades`. The scanner MUST have a blanket skip at the TOP of its loop:

```python
for row in rows:
    if row["source"] == "crypto_5m_engine":
        continue  # Engine manages its own trades
```

A conditional skip buried inside a nested if-else is NOT sufficient — it will be missed when new code paths are added above it. The skip must be the FIRST check in the loop, before any mark-to-market updates, take-profit checks, or expiry closures.
