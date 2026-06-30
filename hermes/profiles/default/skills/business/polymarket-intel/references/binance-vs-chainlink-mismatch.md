# Binance vs Chainlink Latency Comparison

## Result (2026-06-28): Chainlink RTDS WebSocket is DEAD

The Polymarket Chainlink RTDS WebSocket (`crypto_prices_chainlink` topic on `wss://ws-subscriptions-clob.polymarket.com/ws/market`) stopped sending data.

**Test methodology:**
1. Connected simultaneously to Binance WS (`btcusdt@bookTicker`) and Polymarket Chainlink WS
2. Logged nanosecond timestamps for every price update from both sources
3. Ran for 60 seconds inside the crypto container (VPN-required)
4. Result: Binance updates every ~100ms; Chainlink **0 messages received**

The test was repeated multiple times with the same result. The subscription format is correct — `[POLY WS] Subscribed to Chainlink BTC/USD` appears in logs — but no data ever arrives.

## What this means

- **Binance is the only working real-time BTC price source**
- All `state.btc_chainlink` values stay at 0.0
- The latency test script is at `scripts/compare_ws_latency.py`

## Resolution still uses Chainlink (correctly)

Polymarket resolves BTC 5-min markets on Chainlink data. The engine's `_check_market_resolution()` queries Gamma API after window close — Gamma returns the official Chainlink-based outcome. This is correct and independent of the WS feed.

## Latency test script

`/root/polymarket-intel/scripts/compare_ws_latency.py` — standalone script that connects to both WS feeds, logs nanosecond timestamps, and reports which source is faster. Run inside the crypto container.

## History

Before 2026-06-26, the engine used Binance exclusively for entry AND best-guess close. On 2026-06-26, a 3-share UP bet lost -100% because Binance showed +0.14% UP but Chainlink resolved Down. The switch to Chainlink-primary was intended to eliminate this mismatch. Since Chainlink RTDS is no longer available, we're back to Binance for entry signals and rely on Gamma API for official resolution — which is the correct split.
