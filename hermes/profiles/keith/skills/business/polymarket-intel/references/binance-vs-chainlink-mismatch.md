# Binance vs Chainlink: BTC Price Data Source Mismatch

**Discovery date:** 2026-06-26

## The Problem

The crypto engine uses **Binance WebSocket** (`wss://stream.binance.com:9443/ws/btcusdt@ticker`) for real-time BTC price data and direction determination. However, Polymarket's BTC 5-minute Up/Down markets resolve based on the **Chainlink BTC/USD data stream** (`https://data.chain.link/streams/btc-usd`), NOT Binance.

These two data feeds can diverge at the sub-second level. In a 5-minute window with small price moves (0.01-0.15%), a minor discrepancy between Binance and Chainlink can flip the resolution.

## Reproduced Incident

**2026-06-26 2:10-2:15 PM ET window** (slug `btc-updown-5m-1782497400`):

```
Engine (Binance):
  BTC start: $60,067.34
  BTC end:   $60,152.00
  Direction: +0.14% → UP
  Action: bought 3 shares UP at $0.81 each ($2.43 deployed)
  Best-guess at expiry: win

Polymarket official resolution (Chainlink):
  outcomePrices: ["0", "1"]
  Up: $0.00, Down: $1.00
  Winner: Down
  Final: closed_resolved_loss, -100%
```

The engine's best-guess close initially set the trade to "won" (based on Binance direction), then the official Gamma API resolution corrected it to "lost" (based on Chainlink). The dashboard showed "Won" briefly after expiry, then flipped to "Lost -100.00%" after the resolve cycle.

## Impact

- **Directional accuracy is computed on wrong data.** The engine's "edge" calculation (model_up vs market_up) uses Binance prices, but Polymarket settles on Chainlink. The edge numbers are meaningful for Binance direction but don't predict Polymarket resolution.
- **For paper trading:** Acceptable — the dashboard correctly reflects Polymarket's official resolution. The best-guess flip is an annoyance but not a bug.
- **For live trading with real money:** CRITICAL. Trading on Binance data when settlement uses Chainlink data is gambling, not edge-based trading.

## Resolution Options

1. **Switch to Chainlink data stream:** Replace Binance WebSocket with Chainlink's BTC/USD data stream. This aligns the engine's price feed with Polymarket's resolution source.
2. **Use both:** Keep Binance for liquidity/spread signals, use Chainlink for direction determination.
3. **Accept the mismatch:** Only viable if we can quantify the divergence and adjust edge thresholds accordingly.

## RESOLVED (2026-06-26): Option 2 implemented

The engine now uses **Chainlink BTC/USD as primary** via Polymarket's own RTDS WebSocket (`wss://ws-live-data.polymarket.com`, topic `crypto_prices_chainlink`). This is free — Polymarket sponsors the Chainlink API key. Binance bid/ask is retained as fallback.

**Implementation details:**
- Added `btc_chainlink` field to `EngineState`
- Added Chainlink RTDS subscription in `on_poly_open()` alongside the Channel WS orderbook subscription
- Added RTDS topic handler in `on_polymarket_message()` — checks `topic == "crypto_prices_chainlink"` before checking `event_type`
- All BTC price lookups use: `state.btc_chainlink if state.btc_chainlink > 0 else binance_fallback`
- Chainlink used for: btc_mid, window_start_price, halfway acceleration price, best-guess close, safety-net close
- Binance WebSocket connection retained but only used as fallback when Chainlink price is unavailable

## Market Description (from Polymarket)

> "This market will resolve to 'Up' if the Bitcoin price at the end of the time range specified in the title is greater than or equal to the price at the beginning of that range. Otherwise, it will resolve to 'Down'. The resolution source for this market is information from Chainlink, specifically the BTC/USD data stream available at https://data.chain.link/streams/btc-usd. Please note that this market is about the price according to Chainlink data stream BTC/USD, not according to other sources or spot markets."

## Gamma API Resolution Check

After a window expires, the Gamma API returns `outcomePrices` as a JSON string array:
```json
{
  "closed": true,
  "outcomes": ["Up", "Down"],
  "outcomePrices": "[\"0\", \"1\"]"
}
```

Index 0 = Up price, Index 1 = Down price. Price "1" = winner, "0" = loser.

## Latency Comparison (2026-06-26)

Tested live: Binance WebSocket (BTC/USDT) vs Chainlink via Polymarket RTDS (BTC/USD).

### Tick rates
- **Binance**: ~154 ticks/sec (6ms interval) — market-level bid/ask updates
- **Chainlink**: ~1 tick/sec (1000ms interval) — aggregated price

### Price divergence
- Binance consistently ~$92 higher than Chainlink (0.15%) — a fixed USDT/USD premium, NOT a latency lead
- When BTC moves, both feeds move together. The $92 gap stays stable.
- For 5-minute windows, direction is the same on both feeds except at extreme boundary cases (the 2:10 PM loss was one such case)

### Latency
- Median latency: Chainlink ~58ms behind Binance (statistical noise at these tick rates)
- No exploitable millisecond edge either way
- Binance's higher tick rate gives finer granularity but doesn't predict Chainlink direction any better

### Conclusion
The REAL edge is using Chainlink data because that's what resolves the markets. Speed is irrelevant — both feeds deliver within 1 second, and 5-minute windows don't care about sub-second differences. Switching to Chainlink eliminates resolution-mismatch losses, which is the actual performance lever.
