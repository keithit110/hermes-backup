# Market Sell Exits — Midpoint Scalp (2026-06-28, updated 2026-06-29)

User requirement: maximize profits with market sells, not fixed limit sells.

## Before (old limit-based)

After a live buy filled:
1. `place_buy_limit()` returns order ID
2. `place_sell_tp(token, entry+0.04, shares, trade_id)` places a GTC SELL LIMIT at fixed TP
3. Limit order sits on the book until filled or cancelled
4. If bid spikes past TP, the limit fills at the exact TP price — missed windfall

Problem: limit order caps profit at entry+$0.04 even if market blows past.

## After (GTC limit sells at current bid — 2026-06-29)

After a live buy fills:
1. `place_buy_limit()` returns order ID
2. **Nothing else** — no limit TP placed

`check_scalp_take_profits()` runs every second and handles exits via `place_market_sell(token, size, trade_id, limit_price=current_bid)`.

### Exit 1: TP hit (bid ≥ entry + $0.04)
```python
if current_bid >= tp_target:
    sell_ok = True  # paper: always ok
    if is_live and live_trading.check_buy_fill(trade_id):
        token = state.up_token_id if side == "UP" else state.down_token_id
        sell_ok = live_trading.place_market_sell(
            token, SCALP_SHARES, trade_id, limit_price=current_bid
        ) is not None
    if not sell_ok:
        continue  # retry next tick — don't close DB yet
    # mark closed_take_profit in DB
```

### Exit 2: Spike (bid ≥ entry + $0.06)
Same pattern — limit sell at current_bid, check return, retry on failure.

### Exit 3: Near-close (T-30s, bid > entry)
Same pattern — limit sell at current_bid, check return, retry on failure.

## GTC limit sell implementation (2026-06-29 — replaces FOK market orders)
```python
# live_trading.py
def place_market_sell(token_id, size, trade_id, limit_price=None):
    """GTC limit sell at limit_price (current bid). More reliable than FOK/IOC
    market orders which get killed when liquidity is insufficient for all shares."""
    order_args = OrderArgs(
        token_id=token_id,
        price=limit_price or 0.01,
        size=size,
        side=Side.SELL,
    )
    result = _client.create_and_post_order(
        order_args=order_args,
        options=PartialCreateOrderOptions(tick_size="0.01"),
        order_type=OrderType.GTC,
    )
    # Returns order_id on success, None on failure
```

## Critical pitfall — FOK market sells killed silently (2026-06-29)

The original `place_market_sell()` used `OrderType.FOK` (Fill or Kill):
```
[LIVE] MKT SELL #1239 FAILED:
  "order couldn't be fully filled. FOK orders are fully filled or killed."
```

When the orderbook lacked enough bids to fill all 5 shares, Polymarket killed the entire order. BUT the engine didn't check the return value and marked the position as `closed_take_profit` in the DB — the shares never actually sold.

**Fix:** (1) Switch to GTC limit sell at current bid — fills against existing bids without all-or-nothing requirement. (2) Check `place_market_sell()` return value — `continue` (retry) on failure instead of marking closed.

## Losers: unchanged
If bid ≤ entry at T-30s, hold to resolution. No stop-loss.
