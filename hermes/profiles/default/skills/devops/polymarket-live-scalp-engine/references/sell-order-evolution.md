# Sell Order Evolution (2026-06-29)

History of sell order approaches and why we landed on GTC limit at bid.

## FOK (Fill-or-Kill) — FAILED
```python
MarketOrderArgs(token_id, amount=size, side=Side.SELL, order_type=OrderType.FOK)
```
Error: "order couldn't be fully filled. FOK orders are fully filled or killed."
Reason: orderbook depth < 5 shares for thin BTC Up/Down markets.
Verdict: Unusable for 5-share scalp.

## FAK (Fill-and-Kill) — FAILED
```python
MarketOrderArgs(token_id, amount=size, side=Side.SELL)
create_and_post_market_order(args, order_type=OrderType.FAK)
```
Error: "not enough balance / allowance: balance: 4994443, order amount: 5000000"
Reason: MarketOrderArgs `amount` parameter has precision issues — Polymarket's
internal representation (1,000,000x multiplier) causes rounding that makes
5000000 (5.0) not match the actual share balance (4994443 = 4.99 shares).
Verdict: Unreliable — caused infinite retry spam at ~1s intervals.

## GTC Limit at Bid — CURRENT
```python
OrderArgs(token_id, price=current_bid, size=size, side=Side.SELL)
create_and_post_order(args, options=PartialCreateOrderOptions(tick_size="0.01"), order_type=OrderType.GTC)
```
Behavior: Posts a limit sell at the current bid price. Fills against existing
bids. No MarketOrderArgs precision issues. No balance errors.
Verdict: Reliable — this is what we use now.

## Sell Retry Cap

Added `MAX_SELL_RETRIES = 3` in _sell_fail_count dict:
```python
def place_market_sell(..., limit_price):
    fails = _sell_fail_count.get(trade_id, 0)
    if fails >= MAX_SELL_RETRIES:
        return None  # abandoned
    try:
        # ... place GTC limit order ...
        _sell_fail_count[trade_id] = 0  # reset on success
    except:
        _sell_fail_count[trade_id] = fails + 1
        return None
```

On abandon: engine marks position `closed_sell_failed` with current mark price.
No more retries for that trade_id.
