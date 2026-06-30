# Crypto Engine Architecture (BTC 5-Min Paper Trading)

Reference for understanding the engine's data flow, decision logic, and WebSocket architecture.

## Data feeds (two persistent WebSocket connections)

### Feed 1: Binance — live BTC price
```
wss://stream.binance.com:9443/ws/btcusdt@bookTicker
```
Pushes best bid/ask continuously. No auth required. Stored as `state.btc_bid` / `state.btc_ask`.

### Feed 2: Polymarket — live order book
```
wss://ws-subscriptions-clob.polymarket.com/ws/market
```
Subscribes to UP and DOWN token IDs for the current 5-min window. Pushes best bid/ask on every change. Stored as `state.up_ask`, `state.down_ask`, etc.

## 5-minute window math

Polymarket runs BTC contracts in 5-min windows. Engine computes current window:

```python
window_start = int(time.time()) - (int(time.time()) % 300)  # round down to nearest 5 min
slug = f"btc-updown-5m-{window_start}"  # deterministic slug pattern
```

Market discovery: queries `GET gamma-api.polymarket.com/events/slug/{slug}` once per window to find UP/DOWN token IDs.

## Three threads running forever

| Thread | Function | Purpose |
|--------|----------|---------|
| Binance WS | `on_binance_message()` | Writes `btc_bid`, `btc_ask` to state |
| Polymarket WS | `on_polymarket_message()` | Writes `up_ask`, `down_ask`, etc. to state |
| Evaluation timer | `evaluation_timer()` | Reads state, makes decisions every ~5s |

All share one `state` object with a `threading.Lock`.

## Evaluation cycle (runs every ~3s for decisions, ~9s for API calls)

The evaluation timer was split into fast/slow tiers to improve reaction time without hammering APIs:

```
Every 3s:  evaluate_and_act()        ← fast (<0.1s, just math)
Every ~9s: resolve_pending_trades()  ← slow (API call, runs every 3rd cycle)
On window: refresh_token_ids()       ← slow (API call, only 1x per 5 min)
Every cycle: _close_all_expired_windows() ← safety net
```

**CPU note:** Full 2s loop was rejected — would push CPU from ~37% to 70%+. The 3s fast/slow split keeps CPU manageable while giving faster entry reaction.

1. `_close_all_expired_windows()` — Safety net: close ANY open trades for expired windows (runs FIRST, before anything else)
2. `refresh_token_ids()` — Detect new 5-min window, discover new UP/DOWN token IDs via Gamma API, close trades from old window
3. `evaluate_and_act()` — Main brain (runs every cycle)

### Decision flow

```
Do we have valid BTC and Polymarket data? → NO → skip
Already have an open position this window?
  → YES → skip (hedging removed — directional-only now)
  → NO → try directional entry
```

### Directional entry (hedging removed)

Hedging was removed after data analysis showed it cost more than it saved: $3.40 in dead insurance across 36 winning windows vs only $0.06 saved across 3 rescue windows.

Only enters if 60-180s remaining AND spread ≤ 3% AND no open position.

Model probability is mechanical — based on BTC % change from window start:

```
pct_change = (current_btc - window_start_btc) / window_start_btc

> +0.3%  → model says 75% UP
> +0.1%  → model says 62% UP
between  → skip (no opinion)
< -0.1%  → model says 62% DOWN
< -0.3%  → model says 75% DOWN
```

Then compares model to market:
```
market_up_prob = 1.0 - up_ask_price
edge = model_up_prob - market_up_prob
```

**New entry rules:**

1. **Edge ≥ 5%** AND entry price ≤ $0.85 (market not too confident against us)
2. **Variable shares** based on edge strength:
   - Edge ≥ 30% → buy **2 shares** (higher conviction)
   - Edge 15-30% → buy **1 share** (standard)
   - Edge 5-15% → buy **1 share** (threshold)
3. **Skip extreme market odds**: if `up_ask > 0.85` or `down_ask > 0.85` → skip (market too confident, don't fight it)
4. **Acceleration tracking**: engine tracks whether BTC move is accelerating or fading:
   - If BTC was moving up but rate is DECELERATING → skip (momentum dying)
   - If BTC was moving up and rate is ACCELERATING → stronger conviction
5. **Time-decay confidence**: closer to window end → stronger signal:
   ```
   time_factor = 1.0 + (180 - seconds_remaining) / 120  # 1.0 at 180s, 2.0 at 60s
   adjusted_edge = edge * time_factor
   ```

### Resolution

- `_close_old_window_positions()`: Immediate close using BTC direction. **Uses `(state.btc_bid + state.btc_ask) / 2` — there is NO `state.btc_mid` field.** This crashed the engine silently for hours until fixed.
- `resolve_pending_trades()`: Queries Gamma API for official Polymarket resolution, overwrites if available
- Safety net `_close_all_expired_windows()`: Catches any stragglers — runs BEFORE every cycle

## Key constants

```
MIN_SECONDS_REMAINING = 60    # Don't enter too late
MAX_SECONDS_REMAINING = 180   # Don't enter too early
MIN_EDGE = 0.05               # Model must beat market by 5%
MAX_SPREAD = 0.03             # Order book spread must be ≤ 3%
MAX_ENTRY_PRICE = 0.85        # Won't pay more than $0.85 for directional
EVAL_INTERVAL_FAST = 3        # seconds for decision math
EVAL_INTERVAL_SLOW = 9        # seconds for API calls
```

## Known weakness

The model has ZERO predictive intelligence. It's pure momentum: "BTC moved X%, bet it keeps moving." It doesn't look at:
- Order book depth or volume
- Whether the move is accelerating or fading (now partially addressed with acceleration tracking)
- Time-of-day patterns
- Previous window outcomes
- Any external data (news, macro, etc.)

Win rate (~77%) comes entirely from BTC's tendency to continue moving same direction over 5-min windows. When BTC reverses, the directional bet loses. The acceleration tracking and extreme-odds skip rules should reduce some of these losses, but the model itself remains purely mechanical.
