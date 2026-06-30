# BTC 5-Minute Crypto Paper-Trading Engine

## Slug-based market discovery

Polymarket 5-min BTC Up/Down markets follow a predictable slug pattern:

```text
btc-updown-5m-{window_start_unix_epoch}
```

Window start is the Unix epoch (seconds) rounded down to the nearest multiple of 300.

```python
window_start = int(time.time()) - (int(time.time()) % 300)
slug = f"btc-updown-5m-{window_start}"
```

Fetch via Gamma API:

```text
GET https://gamma-api.polymarket.com/events/slug/{slug}
```

Returns a single event with one market, two outcomes (Up/Down), two token IDs, one condition ID.

The `public-search` endpoint does NOT reliably return these markets. Always use the slug endpoint for 5-min BTC windows.

## Architecture

Two persistent WebSocket connections, one evaluation loop:

```
Coinbase/Binance WebSocket → BTC price stream → EngineState.btc_bid/btc_ask
Polymarket Market Channel   → orderbook stream → EngineState.up_ask/down_ask
Evaluation timer (every 5s) → lane logic        → paper_trades inserts
```

All three threads are daemon-threaded. The main thread blocks on `time.sleep(10)` with signal handlers for graceful shutdown.

**Use Coinbase** (`wss://ws-feed.exchange.coinbase.com`, subscribe to BTC-USD ticker) on US-based VPS where Binance is geoblocked.

**Or attach a gluetun VPN container** (Surfshark WireGuard) and route the crypto engine through it to use Binance and bypass Polymarket geoblocks.

## Lane 2 — Late Directional

Buys UP or DOWN when:
- 60–180 seconds remain in the current 5-min window
- BTC price has moved at least ±0.1% from window-start price
- Model probability diverges from market price by ≥5% edge
- Market price ≤ 0.85 (avoid low-prob lottery tickets)
- Spread ≤ 3%
- Liquidity ≥ 500 (min ask size)

Model maps pct_change from window-start to probability:

| pct_change | model_up_prob |
|---|---|
| > +0.3% | 0.75 |
| +0.1% to +0.3% | 0.62 |
| -0.1% to +0.1% | skip (too close) |
| -0.3% to -0.1% | 0.38 |
| < -0.3% | 0.25 |

Entry cost = current ask price for the chosen side. Paper trade source = `crypto_5m_engine`, kind = `crypto_5m_late_directional`.

## Lane 3 — Profit-Lock Hedge

If the engine already has an open Lane 2 paper position on one side, buys the opposite side only when:

```text
existing_entry_cost + opposite_side_ask <= 0.98
```

This locks in a small profit regardless of outcome. Paper trade source = `crypto_5m_engine`, kind = `crypto_5m_profit_lock_hedge`.

Hedge is checked before Lane 2 evaluation each tick — lower risk, higher priority.

## Window lifecycle

- New window detected → capture window-start BTC price from current feed
- Old positions closed when slug changes: status = `closed_window_expired`
- Market subscription updated each evaluation tick via slug-based discovery

## Docker deployment

Separate container from the batch scanner:

```yaml
crypto:
  build: .
  container_name: polymarket-intel-crypto
  restart: unless-stopped
  command: ["python", "-m", "app.crypto_engine"]
  volumes:
    - ./data:/data
    - ./logs:/logs
```

Shares the same SQLite database and Docker image as the scanner/web services. Paper trades appear in the web UI automatically (no source filtering on the `paper_trades` table).

## Env vars

| Var | Default | Purpose |
|---|---|---|
| CRYPTO_MIN_SECONDS_REMAINING | 60 | Min seconds left for Lane 2 |
| CRYPTO_MAX_SECONDS_REMAINING | 180 | Max seconds left for Lane 2 |
| CRYPTO_MIN_EDGE | 0.05 | Min model-vs-market edge |
| CRYPTO_MAX_SPREAD | 0.03 | Max order book spread |
| CRYPTO_MIN_LIQUIDITY | 500 | Min ask size for Lane 2 |
| CRYPTO_MAX_ENTRY_PRICE | 0.85 | Max entry price for Lane 2 |
| CRYPTO_HEDGE_MAX_COST | 0.98 | Max total cost for Lane 3 hedge |

## Pitfalls encountered

- **window_start_price stuck at 0 (silent eval skip)**: Binance/Coinbase WebSocket data arrives asynchronously — often after the first window transition is detected. When `refresh_token_ids()` sets `window_start` on the new window but `btc_bid`/`btc_ask` are still 0, `window_start_price` stays at 0 permanently. Fix: in the evaluation loop, when `window_start_price <= 0` but BTC data IS available, fall back to `(btc_bid + btc_ask) / 2` and write it back to state. Without this, Lane 2 and Lane 3 silently skip every evaluation forever.
- **Polymarket WebSocket needs dynamic re-subscription on window change**: The initial subscription (`on_poly_open`) sends token IDs for the window at connection time. When the 5-min window rolls over, the token IDs change but the WebSocket subscription does not — no book/best_bid_ask events arrive for the new tokens. Fix: store a global reference to the WebSocket (`poly_ws_app`) and send unsubscribe + subscribe messages in `refresh_token_ids()` when token IDs change, using `{"assets_ids": [...], "operation": "subscribe/unsubscribe"}`.
- **best_bid_ask events omit size**: The `best_bid_ask` event type gives bid/ask/spread but NOT size. The `book` event gives full orderbook including sizes. If the engine only uses `best_bid_ask` for price discovery, `up_ask_size`/`down_ask_size` remain 0 and the liquidity check silently blocks all trades. Fix: process both event types; `book` for sizes, `best_bid_ask` for tight spread updates. Set `CRYPTO_MIN_LIQUIDITY=0` during initial bring-up to avoid silent skips.
- **gluetun Surfshark country name format**: gluetun expects the full country name ("United Kingdom", "Netherlands", "Japan") from its hardcoded server list, NOT the two-letter country code ("uk", "nl"). Using a country code causes an immediate startup failure with "value is not one of the possible choices".
- **SILENT CRASH: `state.btc_mid` AttributeError prevents window closure**: The `EngineState` class has `btc_bid` and `btc_ask` attributes but NOT `btc_mid`. When `_close_old_window_positions()` accesses `state.btc_mid`, Python raises `AttributeError`. The `evaluation_timer` loop catches exceptions at the top level (after `refresh_token_ids` fails), so `evaluate_and_act()` and `resolve_pending_trades()` never run — new trades are still opened in subsequent iterations (the crash is caught), but old positions are NEVER closed. **Symptom**: 8+ open trades for expired 5-min windows, engine logs showing `AttributeError: 'EngineState' object has no attribute 'btc_mid'` but otherwise appearing healthy.
  **Fix**: Replace `state.btc_mid` with `(state.btc_bid + state.btc_ask) / 2` (with fallback to `window_start_price` if bid/ask are 0).
- **Safety net required — close ALL expired windows every cycle**: The `_close_old_window_positions()` function only closes trades for ONE window (the one whose slug just changed). If any window was missed (crash, restart, timing), its trades stay open forever. Add `_close_all_expired_windows()` that queries `WHERE status='open' AND end_date < now` and closes ALL matching trades — call it at the TOP of every evaluation_timer iteration, BEFORE any new evaluation. This catches every straggler regardless of crash history.

  **CRITICAL**: when adding a new strategy (e.g. midpoint scalp), `_close_all_expired_windows()` MUST be updated to also query the new `source`/`kind` value. The original implementation only checked `source='crypto_5m_engine'`, so scalp trades (`source='midpoint_scalp'`) were never closed by the safety net and drifted to resolution at -100%. Every strategy gets its own safety-net query block. Check ALL strategy sources in `_close_all_expired_windows()` before deploying a new strategy.
