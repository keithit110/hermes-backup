# Polymarket Live Scalp — Order Execution Patterns (2026-06-29)

## Market Sell Order Types — FAK, not FOK

**py-clob-client-v2 available OrderTypes**: `GTC | GTD | FAK | FOK`

- **FOK** (Fill-or-Kill): fails if orderbook can't fill ALL shares at once. Returns 400: "order couldn't be fully filled. FOK orders are fully filled or killed."
- **FAK** (Fill-and-Kill): fills whatever shares available, cancels rest. USE THIS for market sells.
- `OrderType.IOC` DOES NOT EXIST. Trying it produces `AttributeError: type object 'OrderType' has no attribute 'IOC'` — spammed 100+ times per minute.

## Fill Verification — Never Trust the Engine

Three fatal bugs converged to create false TP entries:

1. **FOK sells failed silently** — engine didn't check `place_market_sell()` return value
2. **DB marked closed_take_profit regardless** — even when `sell_id was None`
3. **User saw "tp_hit" in logs** but shares never sold on Polymarket

Fix: three-step verification before closing any DB row:
```python
# 1. Place sell (FAK), track in _pending_sell_orders
sell_id = live_trading.place_market_sell(token, size, trade_id)
if sell_id is None:
    continue  # Retry next tick

# 2. Poll Polymarket: is the sell filled?
if not live_trading.check_sell_fill(trade_id):
    continue  # Still pending

# 3. Only NOW close the DB
conn.execute("update paper_trades set status='closed_take_profit' ...")
```

## Dashboard P/L — Ground in Wallet, Not DB

The DB `live_pnl_dollars` is an ESTIMATE from trade records. Marked "(not actual)" in UI.

Real P/L = `wallet_balance - wallet_start_balance`

Starting balance persisted in `crypto_engine_state` table (`key='wallet_start_balance'`) to survive health log rotation.

## Re-Entry Blocking — Check Sell Status

When a sell is pending (placed but not yet confirmed filled), the DB row still shows `status='open'`. This blocks `has_scalp_position()` from allowing re-entry.

Fix: `is_sell_pending(trade_id)` check — if a sell is already pending, skip placing another. Once `check_sell_fill()` confirms filled and the row closes, re-entry works.

## Error Logs First, Market Theory Second

When Keith says "it should have sold but didn't":
1. CHECK LOGS FIRST: `docker logs polymarket-intel-crypto --since 20m | grep -E "FAILED|Error"`
2. If errors found → CODE BUG, not design limitation
3. Do NOT explain with "the market trended too hard" until logs are clean
