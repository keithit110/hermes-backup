# Polymarket API Landscape

Reference for when the user asks about Polymarket's API surface, automated trading capability, or platform mechanics. Keep this concise — it's a knowledge bank, not a tutorial.

## What Polymarket is

A prediction market on Polygon (Ethereum L2). Users buy/sell binary outcome tokens (Yes/No) on events. Prices range from $0.00–$1.00 and represent the market's implied probability. Payout is $1.00 for correct, $0.00 for incorrect. All settlement is on-chain via the Conditional Token Framework (CTF).

## API surfaces (from most to least relevant for automation)

### 1. Gamma Markets API (data + market discovery)
- **Base**: `https://gamma-api.polymarket.com`
- **Docs**: https://docs.polymarket.com/developers/gamma-markets-api/overview
- **What it does**: Market discovery, event listing, orderbook snapshots, historical prices, resolution status
- **Key endpoints**:
  - `/events` — list events with filters (tag, liquidity, volume, start/end dates)
  - `/markets?event_id=...` — get markets for a specific event
  - `/markets/{condition_id}` — single market info including `outcomePrices` and `closed` status
- **Rate limits**: Generous. Used by the scanner for bulk market discovery.
- **Auth**: None needed for read-only queries.
- **Cannot place trades**. Read-only data.

### 2. CLOB API (order placement — live trading)
- **Base**: `https://clob.polymarket.com`
- **Docs**: https://docs.polymarket.com/api-reference/introduction
- **What it does**: Order creation, cancellation, book depth, account state
- **Key endpoints**:
  - `POST /order` — place a limit/market order (requires signed EIP-712 message)
  - `GET /book?token_id=...` — orderbook depth
  - `GET /orders` — list open orders by wallet address
- **Auth**: Two-step L1+L2 via wallet signature. Use `py-clob-client-v2>=1.0` (v1 deprecated — returns `400 invalid order version`). L1 derives API credentials via `create_or_derive_api_key()`, L2 uses HMAC for order placement. **Deposit wallets** require pre-created API keys from https://polymarket.com/settings/api-keys — `create_or_derive_api_key()` binds the API key to the EOA, causing `400 the order signer address has to be the address of the API KEY`. Initialize with `signature_type=SignatureTypeV2.POLY_1271` and `funder=funder_address`.
- **Geoblocking**: Blocks US, UK, France, OFAC-sanctioned countries. Returns `403 Trading restricted in your region`. VPN must exit an allowed country (Canada, Netherlands, Switzerland, etc.)
- **This is the ONLY way to place trades programmatically**. The Gamma API is read-only.

### 3. Polymarket WebSocket feeds
- **Market Channel**: `wss://ws-subscriptions-clob.polymarket.com/ws/market`
  - Subscribe to specific asset IDs for real-time book changes and last-trade prices
  - Used by the crypto engine for live BTC 5-min market prices
- **User Channel**: `wss://ws-subscriptions-clob.polymarket.com/ws/user`
  - Subscribe to wallet address for your own order fills/cancellations
  - Used for live trading bots monitoring their own positions

### 4. Polymarket Data API (analytics/volume)
- **Base**: `https://data-api.polymarket.com`
- Used for volume, liquidity, and historical aggregations. Secondary to Gamma.

## Automated trading: what's possible vs what the project does

| Capability | API Required | This project does? |
|---|---|---|
| Discover markets | Gamma | ✓ Scanner queries Gamma |
| Get live prices | CLOB WS | ✓ Crypto engine subscribes |
| Read orderbooks | Gamma or CLOB | ✓ Via Gamma snapshot |
| **Place real orders** | **CLOB + private key** | **Midpoint scalp only (LIVE)** |
| Cancel orders | CLOB + private key | ✗ |
| Check resolution | Gamma | ✓ Engine polls Gamma |
| Track smart wallets | Gamma (events by address) + Polymarket Data | ✓ Scanner |

## Key architectural constraint for live trading

To go live, the project would need:
1. A Polygon wallet with USDC deposited
2. Private key stored securely (never in chat, never in git)
3. CLOB client library integrated into `crypto_engine.py`
4. Replace `simulate_trade()` calls with real `POST /order` calls
5. Fund management: position sizing, collateral checks, USDC balance monitoring

**Current state (2026-06-28):** The midpoint scalp strategy is live-trading on Polymarket CLOB v2 when `LIVE_TRADING_ENABLED=true`. All other strategies (directional, copy wallet, research, arbitrage) remain paper-only. Real orders are placed via `py-clob-client-v2` with POLY_1271 deposit wallet flow + pre-created API keys. Live trading dollars are tracked in `paper_trades.is_live` and displayed separately from paper stats in the dashboard.

## Client libraries

- **Python**: `py-clob-client-v2` (official v2, requires chain_id + two-step auth), `polymarket-python` (Gamma wrapper)
- **TypeScript**: `@polymarket/clob-client` (official)
- **Gamma API**: Plain HTTP — no SDK needed, just `requests`/`curl`

## Geoblocking and API access constraints (updated 2026-06-28)

Polymarket's API surface has become increasingly restricted. Many endpoints that previously worked now fail even through VPN. Here's the current access reality:

### Endpoints that WORK (as of 2026-06-28)

| Endpoint | Access method | Notes |
|----------|--------------|-------|
| `GET /events/slug/{slug}` | Crypto container (VPN) | Returns full event + markets + token IDs. Engine's primary discovery method |
| `GET /events?tag=...&limit=N` | Crypto container (VPN) | Works but `tag=btc5m` returns unrelated events — title-based filtering is more reliable |
| `GET /markets?condition_id=...` | Crypto container (VPN) | Resolution status + outcome prices |
| Binance WebSocket `btcusdt@bookTicker` | Any | No auth required. Engine's primary BTC price source |
| Polymarket Channel WS `wss://ws-subscriptions-clob...` | Crypto container (VPN) | Live orderbook for BTC 5-min tokens |

### Endpoints that FAIL (as of 2026-06-28)

| Endpoint | Error | Impact |
|----------|-------|--------|
| `GET /profile/{username}` (website) | Generic error page + "Go back" | Cannot view user profiles through browser |
| `GET /profile/{username}/activity` (website) | Same generic error | Cannot browse user trade history |
| Gamma `/profile/{username}` API | Empty response | No programmatic profile access |
| Gamma `/users/search` API | 405 Method Not Allowed | No user search |
| Gamma `/users/{username}` API | 404 Not Found | No user lookup |
| CLOB `/data/trades?token_id=...` | 401 Unauthorized | Cannot query per-token trade history programmatically |
| CLOB `/data/trades?market=...` | 401 Unauthorized | Same — auth required for trade data |
| Data API `/activity?user=...` | 400 Bad Request | Query param rejected (likely needs different param format) |
| Data API `/positions?user=...` | "required query param 'user' not provided" | Position history inaccessible |
| Polymarket subgraph (The Graph) | 301 Moved Permanently | Subgraph migrated off hosted service |
| Telonex API | Requires API key | Cannot query on-chain fills without paid key |
| `curl` from VPS host (no proxy) | Works but tagged queries return wrong events | Gamma `/events?tag=btc5m` returns Kraken IPO, Macron, etc. — tag filtering is broken |

### What this means for wallet research

**You cannot currently pull nj23adsknml3's or any wallet's raw trade-by-trade history through Polymarket's APIs.** The CLOB data endpoints require authentication, the subgraph has migrated, profile pages are geoblocked, and Telonex requires a paid API key. The earlier wallet research data came from a working window that has since closed.

**Workaround options (not yet implemented):**
- **Polygonscan API**: Query the CTF contract events for a wallet address — but nj23adsknml3 is a username, not an EVM address. Need to resolve username → address first.
- **Dune Analytics**: Polymarket has Dune dashboards with wallet data — may be queryable through Dune API.
- **Telonex paid tier**: The article we extracted had 15.3M fills from 46,945 wallets. Their API requires a key but may have the data.
- **Polymarket leaderboard**: May expose wallet addresses with stats — but the `/leaderboard` endpoint returned 404.

### What still works for engine operation

The engine's core loop is UNIMPACTED by these restrictions:
- Market discovery via `/events/slug/{slug}` — works
- Binance BTC price via WebSocket — works (no auth)
- Polymarket orderbook via Channel WS — works (no auth)
- Resolution checking via Gamma `/markets` — works
- Scanner market discovery via Gamma `/events` — works (with correct query params)

### Wallet analysis tools — available alternatives (2026-06-28)

Despite the API restrictions above, wallet strategy verification is possible through third-party tools:

- **predicts.guru** (`https://www.predicts.guru/checker/<address>`) — Free wallet checker. Shows live stats: net PnL, volume, fees, buy/sell ratio, avg hold time, win/loss fill counts, position counts, market diversity, PnL timeline. Requires the EVM wallet address (not Polymarket username). This is the fastest way to verify a wallet's strategy. Successfully used to correct nj23adsknml3's strategy from "scalper" to "market maker" (2026-06-28).
- **Polygonscan** — Raw on-chain data when predicts.guru stats are ambiguous. Free tier works without API key for basic lookups.

See `references/polymarket-wallet-analysis.md` for full methodology and verification checklist.
