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
- **Auth**: Requires a Polygon wallet private key. All orders are signed EIP-712 messages. The CLOB client library (`@polymarket/clob-client` or Python `py-clob-client`) handles signing.
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
| **Place real orders** | **CLOB + private key** | **✗ PAPER ONLY** |
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

**Current state**: Paper-only by design. `simulate_trade()` writes to SQLite with entry_cost, trade_price, no blockchain interaction. The user has explicitly stated no live trading / no private keys.

## Client libraries

- **Python**: `py-clob-client` (community), `polymarket-python` (Gamma wrapper)
- **TypeScript**: `@polymarket/clob-client` (official)
- **Gamma API**: Plain HTTP — no SDK needed, just `requests`/`curl`

## Geoblocking

Polymarket blocks US IPs. The crypto engine routes through Surfshark VPN (gluetun container) to access WebSocket feeds. The scanner and web dashboard use host network (non-US VPS IP, typically allowed). If Polymarket tightens geoblocking, more services may need VPN routing.
