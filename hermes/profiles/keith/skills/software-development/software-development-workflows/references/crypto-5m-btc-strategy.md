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

### Lane 2 — Late directional (current implementation)

⚠️ **This section reflects the actual running engine as of 2026-06-28.** The old fixed-bucket model (0.75/0.62/0.38/0.25) was replaced with a proportional model.

- **Window**: 60-180 seconds remaining (`CRYPTO_MIN_SECONDS_REMAINING` / `CRYPTO_MAX_SECONDS_REMAINING`)
- **Model**: **proportional** — `model_up = 0.50 + pct_change * 100`. A +0.05% BTC move → model_up = 55%. A +0.10% move → model_up = 60%. Note: this treats every 0.01% BTC move as +1% probability, making tiny drift (0.04%) produce model_up=54% — barely above noise.
- **Entry trigger**: `model_direction_prob - market_direction_prob ≥ CRYPTO_MIN_EDGE` (default 5%). Trade when the market is pricing the opposite direction heavily (e.g., mkt_up=17% but model_up=54% → edge=37% → BUY UP).
- **Entry cost**: ask price of the chosen side
- **Max spread**: 3% (`CRYPTO_MAX_SPREAD`)
- **Max entry**: 0.85 (`CRYPTO_MAX_ENTRY_PRICE`)
- **Kind**: `crypto_5m_late_directional`

#### Share scaling (FLAT as of 2026-06-28)

**Current: flat 2 shares for all trades.** Variable sizing removed after analysis showed it amplified losses without improving win rate.

| Shares | Trades | Win Rate | Total PnL |
|--------|--------|----------|-----------|
| 1 (was min) | 55 | 80% | +$0.36 |
| 2 (was min) | 64 | 77% | -$2.06 |
| 3 (was edge≥0.30) | 8 | 88% | +$1.02 |
| 4 (was edge≥0.40) | 1 | 0% | -$3.32 |

The single 4-share loss (-$3.32) exceeded all 3-share wins combined (+$1.02). Flat 2 shares removes this blowup risk.

**Previous (removed):** `raw_edge ≥ 0.40` → 4 shares, `≥ 0.30` → 3 shares, else → 2 shares. This put MORE capital on higher-edge trades, but high-edge trades happen when the market is at extremes (e.g., mkt_up=17%, model_up=54%). You're putting 4 shares against an 83% market consensus. When they lose, single-trade losses of -$3.32 erase days of wins.

#### Market consensus filter (added 2026-06-28)

**Only enter when market gives our direction <25% chance** (`market_prob < MIN_MARKET_PROB`, default 0.25). Rationale from historical backtest:

| Market odds for our dir | Trades | Win Rate | PnL |
|---|---|---|---|
| < 20% (extreme fade) | 78 | 85.9% | +$0.78 |
| 20-24% | 19 | 73.7% | +$0.13 |
| **25-39% (danger zone)** | **30** | **60-67%** | **-$4.91** |

The 30 "danger zone" trades (where market gives us ≥25% chance) lost -$4.91. Dropping them flips the engine from -$4.00 to +$3.36 across the same historical data. The filter is: `market_up_prob < 0.25` (for UP trades) or `market_down_prob < 0.25` (for DOWN trades), where `market_prob = 1.0 - ask_price`.

#### Acceleration multiplier (accel✓)
- When BTC price is accelerating in the trade's direction during evaluation, multiplier `t` increases (1.0x → 1.5x-1.9x)
- This allows entries further from the window boundary or with tighter edges
- **Pitfall**: acceleration is detected late in the 5-min window — the price already moved. "accel✓" trades frequently lose because the move is already priced in. Multiple large losses had accel✓ tags.

#### Binary option expectancy math (CRITICAL)

This is the single most important concept for understanding strategy P/L:

| Entry | Avg win return | Breakeven win rate | Current win rate | Profitable? |
|-------|---------------|-------------------|-----------------|-------------|
| 0.65 | +54% | 65% | ~71% | Marginal |
| 0.75 | +33% | 75% | ~73% | **No** |
| 0.80 | +25% | 80% | ~86% | **No** |
| 0.85 | +18% | 85% | ~84% | **No** |

Formula: `breakeven_win_rate = entry / (entry + avg_win_return)`

The strategy has 78% win rate overall but is net negative (-$4.45 across 127 trades) because:
- Wins are capped at `(1.0 - entry) / entry` — a 25% max at 0.80 entry
- Losses are always -100%
- Share scaling amplifies losses more than wins (4-share loss = 20 winning trades to recover)

**This is not fixable with better directional accuracy alone — the payoff structure requires entry price discipline.**

### Lane 3 — Profit-lock hedge

- **Trigger**: engine already holds one side, opposite ask is cheap enough
- **Condition**: existing_entry_cost + opposite_ask ≤ `CRYPTO_HEDGE_MAX_COST` (default 0.98)
- **Kind**: `crypto_5m_profit_lock_hedge`

### Lane 4 — Midpoint scalp (added 2026-06-28)

Independent from Lane 2 directional. Based on reverse-engineering of wallet **nj23adsknml3** — buys near midpoint and scalps small bounces.

- **Entry range**: `SCALP_MIN_ENTRY`–`SCALP_MAX_ENTRY` (default 0.47–0.53) — token ask price must be near 50/50
- **Position size**: `SCALP_SHARES` (default 2) per entry, flat
- **Take profit**: exit when bid reaches `entry_cost + SCALP_TAKE_PROFIT` (default +0.04)
- **Max concurrent**: `SCALP_MAX_OPEN` (default 3) open positions across all windows
- **Deadline bail-out**: close position at last known bid when ≤ `SCALP_DEADLINE_SECONDS` (default 30) remaining; no holding to resolution
- **Source**: `midpoint_scalp`, **Kind**: `midpoint_scalp` — completely separate from directional in DB
- **JSONL logging**: every decision logged to `logs/scalp_decisions.jsonl`

#### Pitfalls

**Bailout fails when `current_bid = 0`**: near deadline, the orderbook may thin out and `best_bid` drops to 0. The bailout check `current_bid > 0` blocks, and the position drifts to resolution at -100%. Fix: the `_close_all_expired_windows()` safety net must cover `midpoint_scalp` — use last-known bid from engine state, fallback to entry price if stale.

**Re-entry loop**: after TP close or deadline bail-out, `has_scalp_position()` only checks OPEN positions. A closed position clears the guard, so the next eval cycle re-enters the same side in the same window. Fix: at the top of `evaluate_scalp()`, check for any `closed_deadline` trades in the current slug — if found, skip all further entries in that window.

**nj23adsknml3 holds losers**: the wallet we reverse-engineered holds losing positions through resolution instead of bailing at deadline. They have 46+ positions simultaneously (across many windows, ~3–4 per window), tiny bets (~$1–2 each), and don't optimize for gas. Their 1.6% edge comes from scale, not from avoiding losses. Holding losers means no re-entry loop, but occasional full -100% losses.

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

## Strategy post-mortem / loss analysis methodology

When the strategy is losing money despite a high win rate, use this systematic debugging approach. The binary option payoff structure means high win rate ≠ profitable.

### Step 1: Overall P/L breakdown

```sql
SELECT 
  CASE WHEN pnl_pct > 0 THEN 'WIN' ELSE 'LOSS' END as outcome,
  COUNT(*), ROUND(AVG(pnl_pct)*100,1) as avg_pnl_pct,
  ROUND(SUM(pnl_pct * entry_cost * shares),2) as total_pnl
FROM paper_trades 
WHERE source='crypto_5m_engine' AND status IN ('closed_resolved_win','closed_resolved_loss')
GROUP BY outcome;
```

**What to look for**: high win rate + negative total PnL = structural expectancy problem. Not a bad-luck run.

### Step 2: Entry cost bucket analysis

```sql
SELECT 
  CASE WHEN entry_cost < 0.65 THEN '<0.65' WHEN entry_cost < 0.75 THEN '0.65-0.74'
       WHEN entry_cost < 0.85 THEN '0.75-0.84' ELSE '0.85+' END as bucket,
  COUNT(*), ROUND(1.0*SUM(CASE WHEN pnl_pct>0 THEN 1 ELSE 0 END)/COUNT(*),2) as win_rate,
  ROUND(SUM(pnl_pct * entry_cost * shares),2) as pnl
FROM paper_trades WHERE source='crypto_5m_engine' 
  AND status IN ('closed_resolved_win','closed_resolved_loss')
GROUP BY bucket ORDER BY MIN(entry_cost);
```

**What to look for**: which entry price ranges are profitable vs unprofitable. High-entry buckets may have high win rates but still lose money.

### Step 3: Breakeven math

For each entry bucket, compute: `breakeven_win_rate = avg_entry / (avg_entry + avg_win_return)`

If actual win rate < breakeven, that bucket is structurally unprofitable. No amount of tuning short of changing entry prices can fix it.

### Step 4: Direction bias

```sql
SELECT json_extract(details_json, '$.side') as side,
  CASE WHEN pnl_pct>0 THEN 'WIN' ELSE 'LOSS' END, COUNT(*),
  ROUND(SUM(pnl_pct * entry_cost * shares),2) as pnl
FROM paper_trades WHERE source='crypto_5m_engine' 
  AND status IN ('closed_resolved_win','closed_resolved_loss')
GROUP BY side, outcome;
```

**What to look for**: one direction systematically worse? The model may have directional bias.

### Step 5: Individual loss inspection

```sql
SELECT id, json_extract(details_json, '$.side') as side, entry_cost,
  json_extract(details_json, '$.edge') as edge, shares,
  json_extract(details_json, '$.reason') as reason
FROM paper_trades WHERE source='crypto_5m_engine' AND status='closed_resolved_loss'
ORDER BY id DESC LIMIT 20;
```

**What to look for**: patterns — "accel✓" tag on losing trades, high share counts on high-entry losses, fixed model_up=62% (old code), edge values that should have been strong but lost anyway.

### Step 6: Simulate fixes against historical data

Before implementing any change, test against the full trade history:

```sql
-- Example: skip accel trades, cap entry at 0.82, flat 1 share
SELECT COUNT(*), ROUND(SUM(pnl_pct * entry_cost),2)
FROM paper_trades WHERE source='crypto_5m_engine'
  AND status IN ('closed_resolved_win','closed_resolved_loss')
  AND entry_cost <= 0.82
  AND details_json NOT LIKE '%accel✓%';
```

### Common failure modes

1. **Share scaling amplifies losses**: More shares on higher edge → 4-share loss at 0.83 = -$3.32. Takes ~20 winning trades to recover.
2. **"accel✓" entries are late**: Price already moved, market already priced it in. These lose more than they win.
3. **Betting against market consensus**: When mkt_up=17% and model says buy UP, you're betting against 83% market confidence.
4. **No positive-expectancy zone**: The math may simply not work at any entry level with this model.
