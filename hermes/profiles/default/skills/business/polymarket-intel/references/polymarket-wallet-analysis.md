# Polymarket Wallet Analysis — Tools & Methodology

Verification-first approach for analyzing Polymarket wallet strategies. Keith's directive: don't assume from summaries — verify from actual data.

## Primary tools (free, no API key)

### 0. Polymarket profile pages — live activity inspection

**URLs:**
- Profile: `https://polymarket.com/@<username>`
- Positions: `https://polymarket.com/@<username>?tab=positions` (Active/Closed tabs)
- Activity: `https://polymarket.com/@<username>?tab=activity` ← most useful for strategy verification

**Key finding (2026-06-28):** Polymarket profile pages are NOT geoblocked for the Hermes browser tool — unlike the main market page which shows a generic "Go back" blocker. The profile pages load full SSR content including the active positions list and recent activity feed. The activity tab shows individual trades with side, price, shares, and timestamp — this reveals LIVE execution patterns, not just aggregate stats.

**What each tab shows:**
- **Positions → Active**: Current open positions with market, side, avg entry, shares, value, PnL. Critical insight: if all positions show 100% profit, the trader likely sells losers before resolution (only winners remain visible).
- **Positions → Closed**: Actually resolved positions. nj23adsknml3 only has 7 — confirms he rarely lets positions reach resolution.
- **Activity**: Individual buy/sell events with side, price, shares, and relative timestamp. This is the fastest way to classify a strategy: maker (variable prices, both sides, high frequency) vs taker (consistent prices, one side, spaced out).

**Limitations:** Polymarket pages are SSR with JavaScript tabs. Tab clicks via `browser_click` may not trigger content swaps (SPA hydration issue). Use URL-based navigation instead: `?tab=activity` or `?tab=positions&positionType=closed`. The activity feed may be truncated to recent events only. **Data consistency warning:** Polymarket profile pages can return different aggregate numbers between loads (e.g., PnL showing $33K vs $189K). This likely reflects partial/cached API responses under heavy load. When Polymarket stats conflict with predicts.guru, trust predicts.guru — it's purpose-built for wallet analytics.

### 1. predicts.guru — fastest first stop

**URL:** `https://www.predicts.guru/checker/<wallet_address>`

Shows live wallet stats scraped from Polymarket's API plus on-chain data:

| Data point | What it tells you |
|------------|-------------------|
| Net PnL | Real profit (not extrapolated) |
| Volume + fees | Whether they're a maker or taker (high fees = taker) |
| Buy/Sell ratio | <1.0 = net seller (market maker); >1.0 = net buyer (speculator) |
| Avg hold time | Same-window (<5m) vs multi-window (>10m) vs long-term |
| Win/Loss fill count | Individual fills vs trade count — distinguishes noise farming from selective trading |
| Position count (open/total) | If almost all open → strategy is long-horizon or unresolved |
| Unique markets | Focused (10-50) vs diversified (1,000+) |
| PnL timeline chart | When they made/lost money — strategy refinement over time |

**Tabs that sometimes work:** Overview, Strategy, Trades, Positions, Risk, Similar Wallets. JavaScript-heavy — browser clicks may not trigger tab switches. Overview tab alone often has enough data.

### 2. Polygonscan — on-chain verification

**URL:** `https://polygonscan.com/address/<wallet_address>`

Polymarket trades are Polygon token transfers involving the Polymarket CTF contract (`0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`). Every buy/sell is a blockchain transaction.

**API endpoint:** `https://api.polygonscan.com/api?module=account&action=tokentx&address=<wallet>&startblock=0&endblock=99999999&page=1&offset=100&sort=desc`

Limitations: free tier rate-limited, may return empty without API key, results are raw token transfers (need decoding to map to Polymarket markets).

### 3. Telonex — full research dataset

**URL:** `https://api.telonex.io/v1/...` (API key required)

The Telonex research article analyzed 46,945 wallets and 15.3M on-chain fills. Their dataset has per-wallet, per-market trade data. This was used in our earlier nj23adsknml3 research but requires an API key for direct access.

## Verification checklist

When analyzing ANY Polymarket wallet:

1. **Get the wallet address** — Polymarket usernames (e.g., "nj23adsknml3") are NOT wallet addresses. Find the mapping via predicts.guru or Polymarket profile.
2. **Check predicts.guru** — Net PnL, buy/sell ratio, avg hold time, position counts. These instantly classify: maker vs taker, scalper vs holder, focused vs diversified.
3. **Don't trust win rate alone** — If total positions >> closed positions, the win rate is meaningless.
4. **Compare fees to profit** — If fees > profit, they're a volume/maker player, not an edge player.
5. **Check market count** — 1-50 = focused strategy. 500+ = noise farming/market making.
6. **Verify on-chain if possible** — Polygonscan for transaction-level data if stats are ambiguous.

## Pitfall: extrapolating from conversation summaries

Compacted conversation summaries can contain stale or wrong data from earlier sessions. When we first described nj23adsknml3 as a "BTC 5-min scalper with 46 positions, 1.6% edge, $183K PnL," we were working from earlier research that was either incomplete or based on a different time window. The real data (predicts.guru, 2026-06-28) showed: $33.9K PnL, 18-min avg hold, 1,574 markets, 1,571/1,578 positions still open, market maker role.

**Rule:** Never present a wallet's strategy as fact without running the verification checklist above. If the data sources are unavailable (API geoblocked, site down), say so explicitly and label what you're presenting as "based on earlier research — not verified this session."
