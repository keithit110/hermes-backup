# Polymarket Wallet Research — BTC 5-Minute Markets

Use this when researching profitable Polymarket wallets, especially for short-duration crypto markets.

## Data Sources (ranked by usefulness for BTC 5m)

### 1. Telonex 15-min Crypto Research (best available)
https://telonex.io/research/top-crypto-traders-polymarket-15m

Analyzed 46,945 wallets, 15.3M on-chain fills, $153M volume across BTC/ETH/SOL/XRP 15-min markets. Same binary mechanics as 5-min. Contains specific wallet addresses, PnL, edge (PnL/volume), markets traded, and maker ratio. Key finding: makers lose more often (47%) but profit (+$728K); takers win more often (53%) but lose (-$728K). Top 5 wallets: 0x63ce ($270K/wk), 0x7780 ($120K, 137% edge), 0x5e2b ($68K, 155% edge), 0xf800 ($59K), 0x66c5 ($61K).

### 2. Polymarket Analytics Leaderboard
https://polymarketanalytics.com/traders

Overall leaderboard (not BTC 5m specific). swisstony #1 ($14M PnL all-time, sports/politics — not BTC). BreakTheBank, RN1, 0x2c33 are hybrid traders. The "Hot Traders" list shows wallet addresses with PnL percentages. Category filter "crypto" exists but may not return BTC 5m-specific results.

### 3. Predicts.guru Leaderboard + Wallet Checker
https://www.predicts.guru/leaderboard

Crypto-category weekly leaderboard with wallet addresses, volume, and PnL. Filter by category=crypto&period=week. Individual wallet checker at `/checker/{address}` shows per-wallet stats including BTC 5m trade history, strategy analysis, and open positions. Wallet nj23adsknml3 (0x674887) shows: $183K PnL, 3,459 trades, noise-farming strategy — tiny bets ($250-300) on both sides of nearly every 5-min window.

### 4. Polymarket Data API
https://data-api.polymarket.com

Public, no auth. Key endpoints:
- `GET /holders?market={conditionId}&limit=N` — top holders per specific market. Returns wallet addresses, amounts, pseudonyms.
- `GET /closed-positions?user={address}&limit=N` — closed positions for a wallet (requires user param, can't query by market alone). Shows realizedPnl, avgPrice, totalBought.
- `GET /activity?user={address}&limit=N` — onchain activity.

**Limitation:** Cannot aggregate PnL across markets by condition ID alone — must query per-wallet. No "top traders by market type" endpoint.

### 5. Gamma API
https://gamma-api.polymarket.com

Market discovery. BTC 5m markets found via slug pattern: `btc-updown-5m-{unix_timestamp}` (timestamps divisible by 300). Each market has a conditionId and clobTokenIds. Use `GET /events?slug={slug}` to get market details.

### 6. Dune Analytics
https://dune.com/lujanodera/polymarket-analysis

Public dashboard with Polymarket analysis including 5-min crypto markets. Requires Dune account to query/export. Cookie-walled for direct scraping.

### 7. Bitquery GraphQL (API key required)
https://docs.bitquery.io/docs/examples/polymarket-api/bitcoin-polymarket-api/

Most powerful — can query on-chain Polygon data with GraphQL, filter by Question.Title containing "Bitcoin Up or Down", aggregate by wallet. Requires API key (unauthorized without it). Can stream real-time trades via WebSocket.

## What Doesn't Exist (as of June 2026)

- **No public BTC 5m-specific wallet leaderboard.** Markets are ~2 months old. Public dashboards categorize all crypto together.
- **No "top traders by market type" endpoint.** Polymarket Data API requires per-wallet queries.
- **No official Polymarket wallet PnL API for 5-min markets.** The closed-positions and activity endpoints exist but require wallet addresses.

## Verifying if a Wallet Trades BTC 5m

Query closed positions and check if titles contain "Bitcoin Up or Down" with timestamps:

```bash
curl -s "https://data-api.polymarket.com/closed-positions?user={ADDRESS}&limit=20" | \
  python3 -c "import sys,json; [print(p['title'][:80]) for p in json.load(sys.stdin) if 'Bitcoin' in p.get('title','')]"
```

## Wallet Strategy Archetypes (from Telonex + predicts.guru)

1. **Sniper** (0x7780, 0x5e2b, 0xf800, 0x66c5): 6-11 markets/week. Huge per-trade returns. Partially makers (25-45%). Use external price signals (Binance spot). Wait for extreme dislocations.
2. **Noise farmer** (nj23adsknml3): Trade every 5-min window. Tiny bets ($250-300) on BOTH sides. Exit winners fast. Collect 1-2% edge at massive scale. $11.6M volume → $183K profit.
3. **Hybrid maker/taker** (0x63ce): ~50% maker ratio. High volume. Collects spread + directional edge.
4. **Pure maker** (0x7163, 0xf49a): 100% maker, 2,000+ markets. No directional prediction. Collect spread across thousands of trades.
