# BTC 5-Minute Crypto Paper-Trading Strategy

Use this reference when building or modifying the Polymarket BTC 5-minute paper-trading engine.

## Architecture

The engine uses **two persistent WebSocket connections** for minimal latency:

- **Binance** `wss://stream.binance.com:9443/ws/btcusdt@bookTicker` → live BTC price (best bid/ask). No auth. Blocked from US IPs — use VPN.
- **Polymarket Market Channel** `wss://ws-subscriptions-clob.polymarket.com/ws/market` → live orderbook for current 5-min UP/DOWN token IDs. No auth.

## Market discovery: predictable slug pattern

Polymarket 5-min BTC markets follow a fixed naming convention:

```
btc-updown-5m-{window_start_unix_epoch}
```

Example for window starting at epoch 1782457500:

```
btc-updown-5m-1782457500
```

**Always use** `GET /gamma-api/events/slug/{slug}` instead of `public-search` — the public-search endpoint does not reliably index these markets.

Each market has exactly 2 outcomes: `["Up", "Down"]` with 2 corresponding `clobTokenIds`.

## WebSocket dynamic subscription

When the 5-min window changes, token IDs change. Resubscribe without reconnecting:

```python
# Unsubscribe old
poly_ws.send(json.dumps({"assets_ids": [old_up, old_down], "operation": "unsubscribe"}))
# Subscribe new
poly_ws.send(json.dumps({"assets_ids": [new_up, new_down], "operation": "subscribe", "custom_feature_enabled": True}))
```

## Strategy lanes

### Lane 2 — Late directional

- **Window**: 60-180 seconds remaining (configurable: `CRYPTO_MIN_SECONDS_REMAINING`, `CRYPTO_MAX_SECONDS_REMAINING`)
- **Model**: simplified distance model — if BTC is up > 0.3% from window start, model UP probability = 75%. If down > 0.3%, model DOWN probability = 75%.
- **Edge threshold**: model_prob - market_prob ≥ `CRYPTO_MIN_EDGE` (default 5%)
- **Max spread**: 3% (`CRYPTO_MAX_SPREAD`)
- **Max entry**: 0.85 (`CRYPTO_MAX_ENTRY_PRICE`)
- **Kind**: `crypto_5m_late_directional`

### Lane 3 — Profit-lock hedge

- **Trigger**: engine already holds one side, opposite ask is cheap enough
- **Condition**: existing_entry_cost + opposite_ask ≤ `CRYPTO_HEDGE_MAX_COST` (default 0.98)
- **Kind**: `crypto_5m_profit_lock_hedge`

### Both lanes

- Paper-only. No live orders, no private keys.
- `source`: `crypto_5m_engine`
- Trades shown in dashboard Paper Bets tab with full filtering/sorting

## Resolution checking

When a 5-min window expires, the engine resolves **immediately** using BTC direction — no waiting for Polymarket settlement:

1. Compute winner from `btc_end` vs `btc_start`: DOWN if BTC ended lower, UP if higher
2. Update each trade's `status`, `current_value`, and `pnl_pct` immediately
3. Queue as `pending_resolve:{slug}` as a safety net
4. `resolve_pending_trades()` polls Gamma API every 5s; if official settlement disagrees (extremely rare), corrects P/L

```python
def _close_old_window_positions(old_slug):
    with state.lock:
        btc_end = state.btc_mid
        btc_start = state.window_start_price
    winner = "Down" if btc_end < btc_start else "Up"
    
    for trade in open_trades:
        side = json.loads(trade["details_json"]).get("side", "")
        resolved = 1.0 if side.upper() == winner.upper() else 0.0
        pnl_pct = (resolved - entry_cost) / entry_cost if entry_cost > 0 else 0.0
        # Update immediately — no more 0% P/L display while waiting
```

**Pitfall — Polymarket settlement delay:** BTC 5-min markets can take 10-15+ minutes for Polymarket to officially mark as `closed: true`. Without instant resolution, the dashboard shows 0% P/L the entire time, confusing users. Always resolve from BTC direction immediately.

**Important gotcha**: `outcomes` and `outcomePrices` are JSON strings in the Gamma API response, not lists. Always parse with `json.loads()` if `isinstance(x, str)`. This is still used by the safety-net poll.

## VPN setup

Binance WebSocket blocks US-based IPs. Route the crypto container through a VPN:

### gluetun with Surfshark WireGuard

```yaml
gluetun:
  image: qmcgaw/gluetun:latest
  cap_add: [NET_ADMIN]
  env_file: .env.vpn

crypto:
  network_mode: "service:gluetun"
  # ... inherits gluetun's network stack
```

`.env.vpn` requires:

```
VPN_SERVICE_PROVIDER=surfshark
VPN_TYPE=wireguard
WIREGUARD_PRIVATE_KEY=...
WIREGUARD_ADDRESSES=10.14.0.2/16
SERVER_COUNTRIES=United Kingdom  # full name, not "uk"
```

**Pitfall**: gluetun validates country names against a strict list. Use "United Kingdom" not "uk", "Netherlands" not "nl".

### Coinbase fallback

If VPN is not available, Coinbase WebSocket works from US IPs without geoblocking:

```
wss://ws-feed.exchange.coinbase.com
Subscribe: {"type": "subscribe", "product_ids": ["BTC-USD"], "channels": ["ticker"]}
```

Coinbase ticker provides `best_bid`, `best_ask`, and `price` fields. Higher latency than Binance but no geoblocking issues.
