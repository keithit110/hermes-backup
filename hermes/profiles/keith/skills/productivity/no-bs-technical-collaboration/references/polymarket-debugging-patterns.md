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

Individual leg percentages (+614% on a $0.14 hedge) are mathematically true but MEANINGLESS in isolation. The pair-level net is the only number that matters.

### Strategy comparison pattern (with-hedge vs without-hedge)

When asked "should I do X or Y?", compute both scenarios from actual data, present in a simple side-by-side table with Deployed/Returned/Net $/Net %.

### Why solo DIR windows lose (and how to detect them)

Solo DIR windows = no hedge could be placed because `dir_entry + min_possible_hedge > 0.98`. The market is heavily skewed against the engine's direction. BTC typically reverses before window close. No structural edge — gambling on momentum continuation.

## Scanner-engine isolation rule

The scanner and crypto engine are separate processes sharing `paper_trades`. The scanner MUST have a blanket skip at the TOP of its loop:

```python
for row in rows:
    if row["source"] == "crypto_5m_engine":
        continue  # Engine manages its own trades
```

The skip must be the FIRST check in the loop, before any mark-to-market updates, take-profit checks, or expiry closures.

## `closed_pending_final_result_sync` — resolution staleness fix

**Symptom**: Copy trades stuck as `closed_pending_final_result_sync` for 17+ hours after event ended.

**Root cause**: `sync_final_results()` queries `GET /markets?slug=X` which returns 403. Correct endpoint is `GET /events/slug/{slug}`.

**Fix**: Use Gamma endpoint `GET /events/slug/{slug}`, parse `outcomePrices` for winner (price == 1.0). Try resolution BEFORE marking pending.

## Engine sell failures — check logs first, not design

**When Keith says "the engine didn't take profit" or "it should have sold but didn't":**

The FIRST diagnostic step is checking engine logs for errors:
```bash
docker logs polymarket-intel-crypto --since 20m 2>&1 | grep -E "FAILED|Error|error|sell_"
```

If you see repeated failures (e.g., `OrderType has no attribute 'IOC'`), that's a CODE BUG, not a design limitation. Do not explain away the behavior as "the market trended too hard" until you've confirmed zero errors. The user should never have to ask three times for you to find an error that prints 100+ times per minute.

## Hold zone datetime bug — `now_utc()` aware-vs-naive silent failure

**Symptom**: Hold zone configured, code in place, ZERO `[SCALP] HOLD` messages. Positions still sold at TP inside hold zone.

**Root cause**: `now_utc()` returns timezone-aware datetime. `end_date` from SQLite parsed via `fromisoformat()` also timezone-aware. Original code: `end_dt.replace(tzinfo=None) - now` (naive minus aware) → TypeError, silently swallowed. `seconds_left` stayed at 300 default.

**Fix**: `seconds_left = max(0, (end_dt - now).total_seconds())` — both timezone-aware. No stripping needed.

**Stale pycache**: After source fix, `.pyc` in `__pycache__/` can persist old bytecode. Dockerfile must include `RUN rm -rf /app/app/__pycache__` after COPY.

## Trend detector self-reinforcing cycle (verified working)

The trend detector compares UP vs DOWN dollar P/L across last 6 windows. When trend=UP, only UP trades are placed → DOWN always has $0 P/L → UP always "wins" → trend stays locked. **This self-corrects naturally** — when BTC goes DOWN, UP positions lose money at resolution ($0 value), and after 4 losing UP windows the lock flips to DOWN. No probe trades needed — UP losses ARE the probe signal.

## Momentum follower diagnostic

When user says "momentum follower stopped making trades," FIRST check whether BTC moved enough during trigger window:
```bash
docker logs polymarket-intel-crypto 2>&1 | grep "MOMENTUM DBG" | tail -20
```
Trigger window is `MOMENTUM_TRIGGER_START=215` to `MOMENTUM_TRIGGER_END=255`. Only fires when BTC moves > `MOMENTUM_THRESHOLD` during those seconds. Quiet BTC = correct skip.

## Stop-loss overwrite bug — resolution correction was overwriting closed_stop_loss (2026-06-30)

**Symptom**: Directional trade hits stop-loss, shows `closed_stop_loss` status, then minutes later mysteriously changes to `closed_resolved_win` or `closed_resolved_loss`. The P/L jumps from -40% to +28% or -100%.

**Root cause**: `resolve_pending_trades()` queries `status like 'closed%'` to re-resolve trades with Gamma API data. This INCLUDES `closed_stop_loss` trades. When Gamma confirms the outcome, it overwrites the SL status.

**Fix**: Exclude SL'd trades from re-resolution:
```python
trades = conn.execute(
    "select id, kind, entry_cost from paper_trades "
    "where source='crypto_5m_engine' and event_slug=? "
    "and status like 'closed%' and status != 'closed_stop_loss'",
    (slug,),
).fetchall()
```

## Resolution correction NEVER fired for momentum trades (2026-06-30)

**Symptom**: Momentum follower trade shows as a win in DB, but Polymarket Gamma API says the opposite outcome. User's wallet lost money but DB shows profit.

**Root cause — TWO bugs**:

1. `_close_old_window_positions()` resolves ALL open trades (including momentum) using Chainlink BTC data as a "best guess." It marks them as `closed_resolved_win` or `closed_resolved_loss` immediately when the window changes. This is fast but can be wrong.

2. `resolve_pending_trades()` is supposed to correct with Gamma API data (the truth), but only queried `status='open'` for momentum/scalp trades. By the time it runs, `_close_old_window_positions()` already changed them to `closed_resolved_*`. It NEVER found them.

**Fix**: Query both open and closed trades for momentum/scalp:
```python
# BEFORE (bug — missed trades already resolved by window-close):
scalp_trades = conn.execute(
    "select id, kind, entry_cost from paper_trades "
    "where source in ('midpoint_scalp','momentum_follower') "
    "and event_slug=? and status='open'",
    (slug,),
).fetchall()

# AFTER (fix — catches all trades):  
scalp_trades = conn.execute(
    "select id, kind, entry_cost from paper_trades "
    "where source in ('midpoint_scalp','momentum_follower') "
    "and event_slug=? and (status='open' or status like 'closed%')",
    (slug,),
).fetchall()
```

**Detection — manual cross-check**:
```bash
# Query Gamma API for a slug:
curl -s "https://gamma-api.polymarket.com/events/slug/{slug}" | python3 -c \
  "import json,sys;ev=json.load(sys.stdin);...print winner"

# Compare to DB status:
sqlite3 data/polymarket_intel.sqlite \
  "SELECT id,status,details_json FROM paper_trades WHERE event_slug='{slug}'"
```

## Entry price — actual Polymarket fill price, not assumed ask (2026-06-30)

**Symptom**: DB entry_price for momentum trades disagrees with what Polymarket actually filled at. Small discrepancies ($0.66 vs $0.67) compound into misleading P/L over many trades.

**Root cause**: `place_buy_fak()` sends FAK order at the current ask, records THAT price as `entry_cost`. But Polymarket might fill at a different price (FAK takes whatever's available). We never queried the actual fill.

**Fix — three parts**:

1. Extract fill price from FAK order response in `live_trading.py`:
```python
def _extract_fill_price(result, order_id, fallback):
    # Try response keys: avg_price, price_matched, matched_price
    # Try fills array: sum(price * size) / sum(size)
    # Fallback: client.get_order(order_id) for avg_price
    return fill_price or None
```

2. `place_buy_fak()` now returns `{"order_id": ..., "fill_price": ...}` dict instead of just order_id string.

3. Crypto engine updates DB when fill differs from assumed ask:
```python
buy_result = live_trading.place_buy_fak(token, entry_cost, shares, trade_id)
actual_fill = buy_result.get("fill_price")
if actual_fill and abs(actual_fill - entry_cost) > 0.001:
    conn.execute("update paper_trades set entry_cost=? where id=?", (actual_fill, trade_id))
```

**Keith's rule**: "Polymarket is the source of truth for ALL data — not the DB. When there is ANY discrepancy, Polymarket wins. Always."

## Stop-loss doesn't work in binary markets when monitoring own token's bid (2026-06-30)

**Symptom**: Directional stop-loss at -40% produces -98% losses. The `closed_stop_loss` status fires, but exit price is $0.01.

**Root cause**: In 5-minute BTC binary markets, the losing token's price crashes INSTANTLY:
```
DOWN bid: $0.50 → $0.01  (no $0.48, $0.40, $0.30 in between)
```
The SL check runs every ~1 second, but by the time it sees the crash, the bid is already at $0.01. The SL fires correctly but exits at a garbage price.

**Fix**: Monitor the OPPOSITE (winning) token's ask instead. The winning side rises gradually:
```
UP ask: $0.50 → $0.52 → $0.55 → $0.60 → $0.70 → $0.85 → $1.00
```
When the opposite token's ask exceeds `1.0 - sl_price`, trigger the SL:
```python
# Entry DOWN @ $0.80, -40% SL → sl_price = $0.48, sl_opposite = $0.52
opposite_ask = state.up_ask if side == "DOWN" else state.down_ask
sl_opposite = 1.0 - sl_price
if opposite_ask >= sl_opposite:
    # UP ask crossed $0.52 → DOWN bid ~$0.48 → close at ~-40%
```

The opposite token's ask is the RELIABLE signal for when a position is underwater. Never monitor the losing token's bid alone — it vanishes in one tick.

## Window re-entry after stop-loss (2026-06-30)

**Symptom**: Directional trade hits stop-loss, then the engine re-enters the SAME window with a new trade. The SL position was closed but `has_open_position()` sees no open trade, so `evaluate_and_act()` enters again.

**Fix**: Freeze the window when SL fires, same as the scalp near-close exit:
```python
# In check_directional_stop_loss(), after closing SL trade:
freeze_window(conn, slug)  # prevent re-entry into same window

# In evaluate_and_act(), before entering directional:
if is_window_frozen(conn, event_slug):
    return  # skip — already had an SL this window
```

The `freeze_window()` / `is_window_frozen()` pair uses `crypto_engine_state` table with key `frozen:{slug}`.

## Eval gate requires Binance WS data — alternate with Chainlink (2026-06-30)

**Symptom**: EVAL TIMER starts but produces zero evaluation output. No [LANE2], no [MOMENTUM], no trades. Engine is alive but silent.

**Root cause**: `evaluate_and_act()` has a gate: `if btc_bid <= 0 or btc_ask <= 0: return`. If Binance WebSocket has a connection race (VPN startup, reconnect, etc.), BTC data is 0 and the eval silently returns.

**Fix**: Accept Chainlink BTC/USD from Polymarket WS as a valid BTC source:
```python
# Use Chainlink as BTC price source when Binance WS is slow to connect
have_btc = (btc_bid > 0 and btc_ask > 0) or state.btc_chainlink > 0
if not have_btc:
    return
```

The Polymarket WS already provides Chainlink BTC/USD via RTDS subscription. Use it as fallback for the eval gate.

## Momentum follower performance analysis (2026-06-30)

When analyzing momentum follower profitability:

**Key metrics**: 57 closed live trades, 68.4% win rate, avg win +45.4%, avg loss -100%, net +$0.73 on $194.27 deployed (0.4%).

**Essential breakdown — pre-cap vs post-cap**:
- Pre-cap (IDs ≤1400, 5 trades): 1 win / 4 losses, max entry $0.94, net -$11.32 (-69%)
- Post-cap (IDs >1400, 52 trades): 38 wins / 14 losses, max entry $0.75, net +$12.05 (+6.8%)

**The $0.75 cap saved it.** Without it, the strategy was a money furnace. With it, it's breakeven.

**Entry price bands (NOT monotonic)** — don't try skip bands:
- <$0.60: 5 trades, 2/3 W/L, net -$3.45
- $0.60-0.64: 8 trades, 7/1 W/L, net +$10.00
- $0.65-0.67: 10 trades, 4/6 W/L, net -$11.17
- $0.68-0.70: 10 trades, 9/1 W/L, net +$10.30
- >$0.70: 26 trades, 19/7 W/L, net -$1.50

Entry price is NOT a clean predictor. The only reliable lever is a stop-loss.
