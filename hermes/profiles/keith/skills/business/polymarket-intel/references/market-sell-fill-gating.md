# Market sell fill-gating — don't sell before the buy fills

When the engine places a live buy order, it takes 1-2 seconds for Polymarket to match it. `check_scalp_take_profits()` runs every 1 second — it can see the bid at TP level and try to market sell BEFORE the buy fills. This fails with `not enough balance: balance: 0`.

## Solution (2026-06-28)

### 1. Track pending buy orders

```python
# live_trading.py — global dict
_pending_buy_orders: dict[int, str] = {}  # trade_id → order_id

# In place_buy_limit(), after successful order:
with _lock:
    _pending_buy_orders[trade_id] = order_id
```

### 2. Poll for fill before allowing sell

```python
def check_buy_fill(trade_id: int) -> bool:
    with _lock:
        order_id = _pending_buy_orders.get(trade_id)
    if not order_id:  # no pending buy — paper trade or already resolved
        return True
    status = _client.get_order(order_id)
    if status.get("status", "").upper() in ("MATCHED", "FILLED", "CLOSED"):
        with _lock:
            _pending_buy_orders.pop(trade_id, None)
        return True
    return False  # still pending — skip sell, retry next tick
```

### 3. Gate all three exits on fill

```python
# Exit 1 (TP hit): only sell if buy filled
if is_live and live_trading.check_buy_fill(trade_id):
    live_trading.place_market_sell(token, SCALP_SHARES, trade_id)
else:
    continue  # buy hasn't filled — retry next tick

# Exit 2 (spike): same pattern
if is_live and live_trading.check_buy_fill(trade_id):
    ...

# Exit 3 (near-close): same pattern
if is_live and live_trading.check_buy_fill(trade_id):
    ...
```

### 4. Startup grace period — don't sell orphaned positions immediately

After engine restart, `_pending_buy_orders` is empty (in-memory). Without a guard, `check_buy_fill()` would return True for ALL open positions, allowing sells on positions where the buy may have never filled.

```python
_startup_ts: float = 0.0  # set in init() via global

def check_buy_fill(trade_id):
    with _lock:
        order_id = _pending_buy_orders.get(trade_id)
    if not order_id:
        if _startup_ts > 0 and (time.time() - _startup_ts) < 60:
            return False  # grace period: don't sell stale positions
        return True  # assumed filled after 60s
```

Set `_startup_ts` in `init()`:
```python
def init():
    global _live, _client, _startup_ts
    ...
    _startup_ts = time.time()
```

### 5. Fix: `_client` is NOT set globally without explicit declaration

Python's scoping means `_client = ClobClient(...)` inside `init()` creates a LOCAL variable unless `global _client` is declared. Without this, `place_buy_limit()` sees `_client is None` and all orders silently return None. Confirmed: `init()` line 123 already has `global _live, _client` — this works.

## Symptoms when missing

```
[LIVE] MKT SELL #1217 FAILED: not enough balance / allowance: balance: 0, order amount: 5000000
```

The sell was attempted before the buy matched. With fill-gating, this error goes away.

## Verification

After a live scalp entry, check logs for:
```
[LIVE] BUY #N FILLED → sell enabled
```

This confirms `check_buy_fill()` detected the MATCHED status and removed the trade from pending.
