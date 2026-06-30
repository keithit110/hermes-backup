# Live Trading Engine Verification Patterns

## Polymarket Minimum Order Size

**Hard-learned**: Polymarket CLOB requires **minimum 5 tokens** for BTC Up/Down markets. Orders with fewer shares are rejected:

```
Size (3) lower than the minimum: 5
```

This means momentum follower can't use 3 shares — minimum is 5. At ~$0.70/share, that's ~$3.50 per trade minimum.

Always test proposed share counts against live order placement. A successful API call to place the order does NOT mean Polymarket accepted it — check for error responses.

## Max Entry Price Guard

Momentum follower entering at extreme prices creates terrible risk/reward. Example: buying DOWN at $0.94 means risking $4.70 to win $0.30 (16:1 against).

Add a configurable cap:

```python
MOMENTUM_MAX_ENTRY = float(os.getenv("MOMENTUM_MAX_ENTRY", "0.75"))

# In evaluate_momentum():
if entry_cost > MOMENTUM_MAX_ENTRY:
    return  # upside too small — skip this signal
```

At $0.75 max, risk/reward is ~3:1:
- Win: +$0.25 × 5 = +$1.25
- Loss: -$0.75 × 5 = -$3.75

## Stale BTC Price in Window Resolution

**Critical bug**: `_close_old_window_positions()` reads `state.window_start_price` to determine if UP or DOWN won. But by the time it runs (new window start), `state.window_start_price` has already been overwritten with the NEW window's BTC price.

```python
# ❌ BROKEN — reads new window's price, not old
if new_start != state.window_start:
    state.window_start_price = mid  # ← OVERWRITTEN with new price

_close_old_window_positions(old_slug)  # reads stale window_start_price
```

This causes `btc_start` and `btc_end` to be nearly identical (both from new window), making every resolution look like a tie.

**Fix**: capture prices BEFORE overwriting:

```python
_old_btc_start = state.window_start_price
_old_btc_end = state.btc_chainlink

if new_start != state.window_start:
    state.window_start_price = mid  # overwrite AFTER capture

_close_old_window_positions(old_slug, _old_btc_start, _old_btc_end)
```

Pass the saved prices as parameters to `_close_old_window_positions()` — don't read from state inside the function.

## Tie-Breaker: Flat BTC → DOWN Wins

On Polymarket, "Will BTC be above $X?" resolves to **No** when BTC is exactly at $X. So flat BTC means DOWN wins, not UP:

```python
# ❌ WRONG — tie defaults to "Up"
winner = "Down" if btc_end < btc_start else "Up"

# ✅ CORRECT — tie goes to Down (UP resolves to No)
winner = "Down" if btc_end <= btc_start else "Up"
```

**Hard-learned**: `OrderType.IOC` does NOT exist in py-clob-client-v2.

Available order types:
- `FOK` (Fill-or-Kill) — all-or-nothing. Fails if full size can't fill. DO NOT USE for sells.
- `FAK` (Fill-and-Kill) — fills whatever shares are available, cancels rest. USE THIS for market sells.
- `GTC` (Good-Til-Cancelled) — limit orders only.
- `GTD` (Good-Til-Date) — limit orders with expiry.

Verify available types at runtime:
```python
from py_clob_client_v2 import OrderType
print([x for x in dir(OrderType) if not x.startswith('_')])
# ['FAK', 'FOK', 'GTC', 'GTD']
```

For true market sells: use `MarketOrderArgs` with `OrderType.FAK`.

## Fill Verification — Polymarket is Source of Truth

**Never** assume an order succeeded. The engine must NOT close a DB trade record until Polymarket confirms the fill.

Pattern:
```python
# Tick N: place sell order, track as pending
sell_id = live_trading.place_market_sell(token, shares, trade_id)
if sell_id is None:
    continue  # retry next tick

# Tick N+1, N+2, ...: poll fill status
if not live_trading.check_sell_fill(trade_id):
    continue  # keep waiting

# Polymarket confirmed filled → now close DB
conn.execute("update paper_trades set status='closed_take_profit' ...")
```

This applies to all three exit paths: TP hit, spike, near-close.

Implementation in `live_trading.py`:
- `_pending_sell_orders: dict[int, str]` — trade_id → order_id
- `is_sell_pending(trade_id)` — already placed?
- `check_sell_fill(trade_id)` — poll `_client.get_order(order_id)`, check status MATCHED/FILLED
- Remove from pending on confirm

Same pattern for buys: `check_buy_fill()` before allowing any sell to proceed.

## FAK Fill Price Extraction — Actual Fill, Not Assumed Ask

**Never** record the assumed ask price as `entry_cost` for live orders. The CLOB may fill at a different price than the quoted ask. Extract the actual fill price from the FAK order response:

```python
def _extract_fill_price(result: dict, order_id: str, fallback: float) -> float | None:
    """Extract actual fill price from FAK order response or query."""
    if isinstance(result, dict):
        # Try keys: avg_price, price_matched, matched_price
        for key in ("avg_price", "price_matched", "matched_price"):
            val = result.get(key)
            if val is not None:
                return float(val)
        # Try fills array
        fills = result.get("fills") or result.get("matches") or []
        if fills:
            total = sum(float(f["price"]) * float(f["size"]) for f in fills)
            size = sum(float(f["size"]) for f in fills)
            if size > 0:
                return total / size
    # Fallback: query the order via get_order()
    try:
        order = _client.get_order(order_id)
        return float(order.get("avg_price") or order.get("price") or fallback)
    except Exception:
        return None
```

After FAK placement, update DB with actual fill price:
```python
buy_result = place_buy_fak(token, price, size, trade_id)  # returns {"order_id": "...", "fill_price": 0.675}
actual_fill = buy_result.get("fill_price")
if actual_fill and abs(actual_fill - assumed_price) > 0.001:
    conn.execute("update paper_trades set entry_cost=? where id=?", (actual_fill, trade_id))
    # Also update details_json to preserve assumed price for audit
```

## Resolution Bug — Gamma API Correction Must Query Closed Trades

The crypto engine resolves trades in TWO phases:
1. **Immediate**: `_close_old_window_positions()` uses Chainlink BTC data (best-guess) → marks as `closed_resolved_win/loss`
2. **Delayed**: `resolve_pending_trades()` queries Gamma API (official outcome) → corrects if wrong

**Bug**: `resolve_pending_trades()` only queried `status='open'` for momentum/scalp trades — but those trades were already marked `closed_*` by phase 1. The correction NEVER fired.

**Fix**: Query must include already-resolved trades:
```python
# ❌ BROKEN — misses trades already marked closed_*
"select ... from paper_trades where source in (...) and event_slug=? and status='open'"

# ✅ CORRECT — re-resolves all trades for that window
"select ... from paper_trades where source in (...) and event_slug=? and (status='open' or status like 'closed%')"
```

This applies to ALL strategies that hold to resolution (momentum, scalp, directional).

## Stop-Loss Bugs — Status Overwrite and Vanishing Bid

Two bugs when implementing mark-to-market stop-loss for directional paper trades:

**Bug 1 — Status overwrite**: `resolve_pending_trades()` was re-resolving `closed_stop_loss` trades and overwriting them with `closed_resolved_win/loss`. The query `status like 'closed%'` matched both.

**Fix**: Exclude stop-loss status:
```python
"... and status like 'closed%' and status != 'closed_stop_loss'"
```

Also, `_close_old_window_positions()` must NOT overwrite SL'd trades — only process `status='open'`:
```python
"... and event_slug=? and status='open'"  # already correct
```

**Bug 2 — Bid vanishes to 0**: When the market crashes, buyers disappear and the bid drops to 0. The SL check used `current_bid` and skipped when `bid <= 0`:

```python
# ❌ BROKEN — bid=0 causes continue, SL never fires
current_bid = state.up_bid if side == "UP" else state.down_bid
if current_bid <= 0:
    continue     # <-- skips the SL check entirely!
```

**Fix**: Use the ask as mark-to-market when bid is unavailable:
```python
# ✅ Use ask as mark when bid vanishes
bid = state.up_bid if side == "UP" else state.down_bid
ask = state.up_ask if side == "UP" else state.down_ask
mark_price = bid if bid > 0 else ask  # fallback to ask
```

Also **freeze the window** after SL fires to prevent re-entry:
```python
if mark_price <= sl_price:
    # ... close the trade ...
    freeze_window(conn, slug)  # no more entries this window
```

## Eval Gate — Chainlink Fallback When Binance WS Is Slow

The evaluation loop checked `btc_bid > 0 AND btc_ask > 0` before running strategies. But Binance WebSocket can have a startup race, while the Polymarket Chainlink BTC/USD feed is already active. The eval loop silently returned with no log output.

**Fix**: Accept Chainlink as a valid BTC data source:
```python
# Before: require Binance
if btc_bid <= 0 or btc_ask <= 0:
    return

# After: accept Chainlink as fallback
have_btc = (btc_bid > 0 and btc_ask > 0) or state.btc_chainlink > 0
if not have_btc:
    return
```

Also compute `btc_mid` from Chainlink when available:
```python
btc_mid = state.btc_chainlink if state.btc_chainlink > 0 else ((btc_bid + btc_ask) / 2)
```

## FOK Market Sell Failure Pattern

When FOK sells fail, the error is:
```
"order couldn't be fully filled. FOK orders are fully filled or killed."
```

This happens when the orderbook lacks enough bids for all 5 shares at once. Solution: switch to FAK.

## Trend Detection from Own P/L Data

Instead of price-based indicators (lag), analyze resolved window P/L:

```python
def detect_trend(conn, current_slug):
    # Get last 6 resolved btc-updown windows
    # For each: compute UP vs DOWN dollar P/L
    # Count which side "won" each window
    # Require >= 67% majority to declare trend
    # Return "UP", "DOWN", or "NEUTRAL"
```

Use in entry logic:
- UP trend → only open UP positions (DOWN blocked)
- DOWN trend → only open DOWN positions (UP blocked)
- NEUTRAL → open both with reduced sizes

Self-corrects within 2-3 windows after trend reversal.

## Dynamic Position Sizing

Pair trend detection with risk-adjusted sizing:

| Trend | Favored side | Disfavored side |
|-------|-------------|-----------------|
| UP | SCALP_SHARES (5) | 0 (blocked) |
| DOWN | 0 (blocked) | SCALP_SHARES (5) |
| NEUTRAL | SCALP_SHARES_NEUTRAL (2) | SCALP_SHARES_NEUTRAL (2) |

Store per-trade shares in `details_json` so the sell logic uses the correct count.

## Scanner Skip List — Protect Engine-Managed Trades

The scanner auto-closes paper trades at +10% TP or -20% SL. It must NOT touch trades managed by the crypto engine. When adding a new strategy, add its source to the skip list:

```python
# In app/main.py, scanner's mark-to-market loop:
if row["source"] in ("crypto_5m_engine", "midpoint_scalp", "momentum_follower"):
    continue  # crypto engine manages these — scanner hands off
```

Without this, the scanner will auto-close momentum follower trades at +10%, destroying the hold-to-resolution strategy.

## Live-Only Strategy — No Paper Fallback

When a strategy goes live, it should NOT fall back to paper on failure. Pattern:

```python
# ❌ WRONG — paper fallback masks failures
trade_id = insert_paper_trade(..., is_live=False)
if want_live:
    buy_ok = place_buy_limit(...)
    if buy_ok:
        conn.execute("update paper_trades set is_live=1 where id=?", (trade_id,))

# ✅ CORRECT — live or nothing
if not want_live:
    return  # can't go live, skip entirely
trade_id = insert_paper_trade(..., is_live=True)
buy_ok = place_buy_fak(...)
if not buy_ok:
    conn.execute("delete from paper_trades where id=?", (trade_id,))  # clean up
    conn.commit()
```

Key rules:
1. Skip if can't go live — don't create any DB record
2. Use FAK for guaranteed fills (see GTC vs FAK reference)
3. Delete the DB row on failure — no ghost trades
4. Zero paper trades for live strategies per Keith's preference

- Wallet/on-chain is the **source of truth**. DB must mirror wallet reality — if there's a discrepancy, fix the DB, not the wallet.
- Read real wallet balance from `live_health.jsonl` (logged by engine health heartbeat)
- Persist `wallet_start_balance` in `crypto_engine_state` table (survives log rotation)
- Display `actual_pnl = wallet_balance - wallet_start_balance` as the authoritative number
- Label DB-computed P/L from trade records as "DB est" or "(not actual)"
- **Show midpoint_scalp live trades in the web UI** — do NOT hide or exclude them. When all paper data is deleted, midpoint_scalp IS the live trading data the user wants to see. Remove `kind != 'midpoint_scalp'` from all API queries and JavaScript filters when going live.
- After deleting paper data, reset `wallet_start_balance` to current wallet balance so P/L reflects performance from that point forward.

## Deposit/Withdrawal Detection — Don't Count Transfers as Profit

When money is deposited TO or withdrawn FROM the Polymarket wallet, `actual_pnl = wallet_balance - wallet_start_balance` will show a false profit/loss. The UI must detect external transfers and auto-reset the baseline.

**Pattern (in web API, every request):**

```python
# After computing actual_pnl and live_pnl_dollars:
if actual_pnl is not None and abs(actual_pnl - live_pnl_dollars) > 0.50:
    # Wallet moved more than trades can explain → deposit or withdrawal
    new_start = wallet_balance - live_pnl_dollars
    db = conn()
    db.execute(
        "insert or replace into crypto_engine_state(key,value) "
        "values('wallet_start_balance',?)",
        (str(round(new_start, 2)),),
    )
    db.commit()
    wallet_start_balance = new_start
    actual_pnl = round(wallet_balance - wallet_start_balance, 2)
```

**Threshold**: $0.50 — enough to ignore rounding/gas noise, tight enough to catch most deposits. Adjust if gas fees on the chain are higher.

**Key insight**: this runs on every dashboard load (API call), not on a timer. It self-corrects within one page refresh after a deposit or withdrawal.

**User preference**: Keith explicitly does NOT want deposits counted as profit. The DB Est P/L panel was removed from the UI entirely because it showed phantom profits from unfilled GTC orders. Only the wallet-grounded Actual P/L remains.

## Midpoint Scalp Hold Zone

When the 5-minute window is near resolution, hold positions instead of scalping:

- **`SCALP_HOLD_SECONDS`** env var (default 60, Keith prefers 120)
- At `seconds_left <= SCALP_HOLD_SECONDS`, the engine skips ALL three exits (TP, spike, near-close)
- Positions ride to resolution at 1.0 or 0.0 instead of selling early
- Only log `[SCALP] HOLD` once per position on entry to the hold zone (gate with `seconds_left + 5 > SCALP_HOLD_SECONDS`)
- New entries are already blocked by `SCALP_DEADLINE_SECONDS` (freeze) which fires earlier

Guard placement in `check_scalp_take_profits()`:
```python
# After computing seconds_left, before any exit:
if seconds_left <= SCALP_HOLD_SECONDS:
    if (seconds_left + 5) > SCALP_HOLD_SECONDS:
        print(f"[SCALP] HOLD #{trade_id} {side} — holding to resolution")
    continue  # skip TP, spike, and near-close exits
```

This avoids leaving money on the table when price has already converged near the outcome. Example: UP at 0.97 with 100s left — TP would sell at +0.05 but holding gives +0.45 per share.

## Python Datetime Naive vs Aware Pitfall

**Critical**: `now_utc()` returns `datetime.now(timezone.utc)` — this is **timezone-aware**. The DB stores end_dates with `+00:00` suffix, and `datetime.fromisoformat(...)` also produces timezone-aware datetimes. **Subtract directly** — do NOT strip tzinfo from either side:

```python
# ✅ CORRECT — both are aware, subtract directly
now = datetime.now(timezone.utc)                           # aware
end_dt = datetime.fromisoformat("2026-06-29T03:40:00+00:00")  # aware
seconds_left = (end_dt - now).total_seconds()

# ❌ WRONG — naive vs aware mismatch
end_naive = end_dt.replace(tzinfo=None)  # naive
seconds_left = (end_naive - now).total_seconds()  # TypeError!

# ❌ ALSO WRONG — now was already aware, replace does nothing
seconds_left = (end_dt - now.replace(tzinfo=None)).total_seconds()  # TypeError!
```

**Symptom**: `can't subtract offset-naive and offset-aware datetimes` in logs, and `seconds_left` silently stays at 300 (default) because the `except: pass` swallowed the error. This causes the hold zone to NEVER activate.

**Fix verification**: test directly inside the container:
```python
docker exec polymarket-intel-crypto python3 -c "
from datetime import datetime, timezone
end_dt = datetime.fromisoformat('2026-06-29T03:40:00+00:00')
now = datetime.now(timezone.utc)
print((end_dt - now).total_seconds())  # must succeed
"
```

## Docker Stale .pyc Cache Pitfall

Even with `ENV PYTHONDONTWRITEBYTECODE=1` in the Dockerfile, old `.pyc` files from previous build layers can persist and shadow updated source. Result: code changes appear deployed but old bytecode executes.

**Fix in Dockerfile**:
```dockerfile
COPY app ./app
RUN rm -rf /app/app/__pycache__
CMD ["python", "-m", "app.main"]
```

**Verification**: after rebuild, confirm no `.pyc` files exist:
```bash
docker exec polymarket-intel-crypto find /app -name '*.pyc'
# should return nothing
```

## Never Silently Swallow Exceptions in Critical Paths

The `seconds_left` computation had `except Exception: pass` — when the datetime subtraction failed, `seconds_left` stayed at the default 300. This silently disabled the hold zone with zero indication in logs.

**Always log exceptions** in critical decision paths. Use a concise `except Exception as e: print(...)` so failures are visible in logs.

## Wallet-First Debugging (User Pref)

When Keith says a position exists but the DB disagrees, **query the wallet/chain first**. Do not argue from DB data. The Polymarket wallet is the source of truth — if there's a mismatch, the DB is wrong and must be reconciled.

Priority order when debugging live trading discrepancies:
1. Query wallet balance and positions (on-chain, or engine health log `live_health.jsonl`)
2. Compare with DB `paper_trades` for that window
3. If mismatch → fix DB to match wallet, not vice versa
4. Only then investigate why the DB got out of sync

## DB Lock Recovery (SQLite)

When containers are stopped but zombie processes still hold SQLite locks:

```bash
# Find PIDs holding the lock
fuser /path/to/polymarket_intel.sqlite

# Kill them
kill -9 <pid1> <pid2>

# Verify lock released
sqlite3 /path/to/polymarket_intel.sqlite "SELECT 1;"
```

Symptom: `Error: stepping, database is locked (5)` on any write operation, even after the container is stopped.

## Python Docstring Code Pitfall

Code placed inside a docstring (between the opening `"""` and closing `"""`) is treated as documentation text and **never executed**. This is easy to miss when inserting guards via scripted line insertions.

```python
# ❌ WRONG — guard is inside docstring, never runs
def evaluate_scalp(...):
    """Midpoint scalping: buy tokens near $0.50...
    
    if SCALP_SHARES <= 0:   # <-- INSIDE docstring!
        return              # <-- never executed!
    
    Each new window gets its own budget..."""

# ✅ CORRECT — guard after docstring
def evaluate_scalp(...):
    """Midpoint scalping: buy tokens near $0.50..."""
    if SCALP_SHARES <= 0:
        return  # scalp disabled
```

Verify by checking the line position relative to the closing `"""`.
