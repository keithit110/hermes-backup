# GTC Orders vs FAK for Guaranteed Fills

## Critical Lesson: GTC Orders Can Sit Unfilled Even at Ask Price

A GTC (Good-Til-Cancelled) limit buy at the current ask price `place_buy_limit(token, ask, size, trade_id)` **may not fill immediately**. The order is placed on the book but if liquidity is thin or the ask moves away, it sits unmatched indefinitely. The engine logs "LIVE" because the order placement succeeded — but no money moved.

**Result**: DB records the trade as `is_live=1`, P/L is computed from it, but wallet balance never changed. This creates a growing gap between DB-estimated P/L and actual wallet P/L.

## Solution: FAK (Fill-And-Kill) for Momentum Entries

```python
def place_buy_fak(token_id, price, size, trade_id):
    """FAK buy — fills what's available right now, cancels remainder."""
    result = _client.create_and_post_order(
        order_args=OrderArgs(token_id=token_id, price=price, size=size, side=Side.BUY),
        options=PartialCreateOrderOptions(tick_size="0.01"),
        order_type=OrderType.FAK,  # ← key difference
    )
```

FAK guarantees either:
- **Immediate fill** (money moves, position is real)
- **Immediate cancel** (no phantom trade)

If FAK fails, delete the DB row — no paper fallback, no ghost records.

## When to Use Each

| Order Type | Use Case | Fill Guarantee |
|---|---|---|
| GTC | Midpoint scalp (high-liquidity $0.45-$0.55 zone) | Usually fills at ask, but not guaranteed |
| GTC | Lane 2 late directional (paper only) | N/A — paper trades |
| FAK | Momentum follower (requires confirmed fill) | Fills or cancels — no ambiguity |
| FOK | Do NOT use — fails when can't fill all | All-or-nothing, unreliable |

## P/L Discrepancy from Phantom GTC Trades

When `actual_pnl` (wallet) ≠ `closed_pnl_dollars` (DB):
1. Check for `is_live=1` trades placed with GTC
2. Compute expected wallet balance: `start_balance + sum(winning trades) - sum(losing trades)`
3. If wallet balance doesn't match → phantom fills
4. Fix: clean up phantom trades in DB, reset `wallet_start_balance` to current wallet

Going forward with FAK, DB P/L should stay in sync with wallet.