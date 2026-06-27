# Polymarket paper-trading / copy-trading safety notes

Use this reference when building or refining Polymarket scanners, paper-trading dashboards, or copy-trade logic.

## Verified API landscape

Polymarket has three public API surfaces useful for read-only/paper systems:

- Gamma API: `https://gamma-api.polymarket.com` — events, markets, discovery, tags, public profiles. **API gotchas**: `outcomes` and `outcomePrices` fields are JSON strings, not parsed lists — always call `json.loads()` if `isinstance(x, str)`. `outcomePrices: ["0","1"]` means the second listed outcome resolved as YES. 5-min BTC markets follow predictable slug pattern `btc-updown-5m-{window_start_epoch}` — use `GET /events/slug/{slug}` instead of `public-search` for reliable lookup.
- CLOB API: `https://clob.polymarket.com` — public orderbook/price endpoints; authenticated order placement/cancel endpoints require L1/L2 auth.
- Data API: `https://data-api.polymarket.com` — positions, closed positions, activity, values, holders, trades.

A read-only/paper system should have no private key, no `POLY_*` auth credentials, and no order-posting calls. Verify by searching code/config for private keys/auth variables and by checking endpoint usage.

## Copy-trade pitfalls found

### Wallet copy strategy split (single vs consensus)

The scanner uses two separate wallet-copy strategies, but the DB may contain legacy trades from old code:

- **`copy_single_high_win_rate_wallet`** (source: `smart_wallet_copy`): copies a single high-win wallet's BUY. Current code uses this. Requires wallet closed ≥50, win rate ≥70%.
- **`copy_wallet_consensus`** (source: `smart_wallet_consensus`): copies when multiple distinct wallets take the same side on the same market. Requires ≥2 wallets. May rarely trigger — don't expect this to fire every cycle.
- **`copy_high_win_rate_wallet`**: legacy kind from old pre-split code. If present in the DB, it predates the single/consensus split. New trades use the split kinds above.

When the dashboard shows `copy_high_win_rate_wallet` with many trades and `copy_single_high_win_rate_wallet` with few, the former is legacy — don't try to reconcile them as part of the same strategy.

### Write-only data collection pitfall

The scanner populates `markets`, `info_items`, and `research_notes` tables, but neither the scanner nor the research agent **reads** these tables to make decisions. Both agents query Polymarket's API directly each cycle. This means:

- Markets tab data is collected but never used for trade filtering
- News/Info items are stored but never influence trade recommendations
- Research notes are written but never referenced in subsequent cycles

If you want the agents to actually use this intelligence, you need to wire the DB reads into the decision pipeline — otherwise these tabs are pure dashboards for manual browsing, not decision inputs.

Naive copy logic such as `closed_count >= 20` and `win_rate >= 70%` is not sufficient. It can overfit recent wallet history and copy unrelated/tail-risk trades after the wallet's edge is already gone.

Failure modes observed:

- Stale/expired markets copied because only `end <= now + 2 days` was checked; must also require `end > now + safety buffer`.
- Wallet win rate from recent closed positions can be misleading; require higher sample size and stronger thresholds before even paper-copying.
- Exact score, halftime/1st-half, both-teams-to-score, and similar prop markets create near-total-loss tail risk.
- Cheap contracts can show attractive percentage gains but behave like lottery tickets.
- Duplicate recopying of the same event/token after a paper trade closes pollutes results.
- Mark-to-market close at expiry is not official settlement; label it clearly until true resolved-result accounting is implemented.

## Safer default paper-copy filters

Before considering live trading, keep copy mode paper-only and start stricter than seems necessary:

- BUY side only.
- Market end date must be in the future and within the user's max window.
- Require a minimum time-to-end buffer, e.g. 15+ minutes.
- Require stronger wallet stats, e.g. closed positions >= 50 and win rate >= 70%.
- Block brittle prop categories: exact score, halftime, 1st half, both teams to score, leading at halftime.
- Add both take-profit and stop-loss paper exits, e.g. +10% / -20%.
- Do not recopy the same event/token simply because the prior paper trade closed.

### Sport-category-dependent price/size thresholds (critical pitfall)

**One-size-fits-all filter bands are wrong.** Different sports/markets trade at different price profiles. A filter tuned for FIFA favorites (0.55–0.80, sizes 50+) will silently reject all tennis, MLB, and WNBA copy signals — even from 100%-win-rate wallets with $10M+ realized PnL.

**Discovered failure mode:** We tracked 18 wallets with 112 non-FIFA signals (tennis, MLB, WNBA, KBO) from wallets scoring 100/100 with $9.87M and $11.9M PnL. **Every single one was rejected** by `COPY_MIN_PRICE=0.67` and `COPY_MIN_SIZE=25` because:

| Category | Typical Price Range | Typical Size Range |
|----------|--------------------|--------------------|
| FIFA favorites | 0.55–0.80 | 50–5,000 |
| Tennis underdogs | 0.20–0.60 | 10–500 |
| MLB/WNBA | 0.30–0.70 | 10–200 |

**Fix — use broader thresholds that cover multiple sports:**
```yaml
COPY_MIN_PRICE: "0.20"   # Not 0.67 — tennis underdogs trade at 0.25–0.60
COPY_MAX_PRICE: "0.93"   # Still blocks 0.96+ late entries
COPY_MIN_SIZE: "10"      # Not 25 — tennis signals often 10–20 USDC
```

**Validation:** after lowering to 0.20/10, run the scanner and verify non-FIFA signals appear in `wallet_activity` with non-null `copied_paper_trade_id`. If they still don't, check:
1. `COPY_MIN_MINUTES_TO_END` — some tennis/MLB markets resolve within minutes
2. `MAX_DAYS_TO_RESOLUTION` — some political/long-term markets exceed 3 days
3. Whether the top 5 activities per wallet happen to be all FIFA at scan time (the scanner only processes 5 per wallet per run)

## Crypto 5-minute engine (BTC Up/Down)

The crypto engine is a long-running WebSocket-driven paper trader for Polymarket's 5-min BTC Up/Down markets. Key architectural decisions and pitfalls:

### Market discovery

5-min BTC markets follow a predictable slug pattern: `btc-updown-5m-{window_start_unix_epoch}`. Use `GET /gamma-api.polymarket.com/events/slug/{slug}` instead of `public-search` — the search endpoint does not reliably return these markets.

### API field parsing (critical pitfall)

The Gamma API returns `outcomes` and `outcomePrices` as **JSON strings**, not parsed lists:

```python
outcomes = m.get("outcomes") or []
if isinstance(outcomes, str):
    outcomes = json.loads(outcomes)
outcome_prices = m.get("outcomePrices") or []
if isinstance(outcome_prices, str):
    outcome_prices = json.loads(outcome_prices)
```

Failing to parse these results in iterating over individual characters. `["0","1"]` means the second outcome won.

### Latency architecture

Two persistent WebSocket connections, event-driven evaluation:

- **Binance bookTicker** (`wss://stream.binance.com:9443/ws/btcusdt@bookTicker`) → live BTC bid/ask (no auth)
- **Polymarket Market Channel** (`wss://ws-subscriptions-clob.polymarket.com/ws/market`) → subscribe to UP/DOWN token IDs, receive `book`, `price_change`, `best_bid_ask` events (no auth)

Polling REST is too slow for 5-min markets. WebSocket push cuts latency from 500ms-2s to ~100-300ms.

### Dynamic WebSocket subscription

When the 5-min window rolls, token IDs change. The Poly WS must unsubscribe old tokens and subscribe new ones without reconnecting:

```python
ws.send(json.dumps({"assets_ids": [old_up, old_down], "operation": "unsubscribe"}))
ws.send(json.dumps({"assets_ids": new_ids, "operation": "subscribe", "custom_feature_enabled": True}))
```

### Resolution P/L accounting

When a window ends, mark positions `closed_window_expired` and queue the slug for resolution. A separate `resolve_pending_trades()` function checks `Gamma /events/slug/{slug}` for `"closed": True` with `outcomePrices` showing `"1"` on the winning outcome. Update P/L:

```python
resolved_price = 1.0 if side == winner else 0.0
pnl_pct = (resolved_price - entry_cost) / entry_cost
```

Wait 10 seconds after expiry before checking — Polymarket needs time to settle the oracle.

### VPN for geoblocking

Binance WebSocket is geoblocked from US IPs (error 451). Solution: route the crypto engine container through **gluetun** with Surfshark WireGuard:

```yaml
gluetun:
  image: qmcgaw/gluetun:latest
  cap_add: [NET_ADMIN]
  environment:
    VPN_SERVICE_PROVIDER: surfshark
    VPN_TYPE: wireguard
    WIREGUARD_PRIVATE_KEY: ...
    WIREGUARD_ADDRESSES: "10.14.0.2/16"
    SERVER_COUNTRIES: United Kingdom  # full name, NOT country code

crypto:
  network_mode: "service:gluetun"  # inherits VPN IP
```

**Pitfall:** gluetun requires full country names (e.g., "United Kingdom"), not country codes ("uk"). Wrong country name causes immediate exit.

### Strategy lanes

- **Lane 2 (late directional):** fires at 60-180s remaining, requires ≥5% edge between model probability and market price, max entry $0.85, max spread 3%. Uses simplified distance model: if BTC is >0.3% away from window-start price, model probability is 75%/25%.
- **Lane 3 (profit-lock hedge):** if already holding one side, buys opposite only when total cost ≤ $0.98 ($HEDGE_MAX_COST).

### Common bugs

- `window_start_price` stays 0 when Binance data arrives after window transition → use current BTC mid as fallback
- Patch churn can silently drop module-level constants (`SESSION`, `CLOB`) → `_check_market_resolution` returns `None` silently because `except Exception` catches the `NameError`

## Review workflow

When asked to reassess a strategy:

1. Verify live code endpoints and absence of live-trading credentials.
2. Query current paper trades and group performance by status, price band, and market type.
3. Inspect worst losses and best wins separately; jackpot wins can hide bad tail risk.
4. Identify whether results are official resolutions or just mark-to-market values.
5. Patch strategy rules, run syntax/tests, rebuild/restart if applicable, and run one scanner pass.
6. Report current open trades and exactly what changed.
