# Momentum Follower — Live-Only Strategy

Created: 2026-06-29  
Status: **LIVE** (primary active strategy)

## Strategy Logic

1. Every 1 second, check if `seconds_remaining` is in trigger window (215-255, i.e. T+45s to T+85s)
2. If BTC has moved ≥ threshold (0.05%) from window start price → enter that direction
3. If entry cost > MOMENTUM_MAX_ENTRY ($0.75) → SKIP (upside too small)
4. Place **FAK (Fill-And-Kill)** buy at current ask price — fills immediately or cancels
5. Hold to resolution — NO take profit, NO stop-loss, NO spike sell
6. One entry max per window (checked via DB)

## Production Config (docker-compose.yml)

```
MOMENTUM_ENABLED: "true"
MOMENTUM_LIVE: "true"
MOMENTUM_THRESHOLD: "0.0005"     # 0.05% BTC move
MOMENTUM_SHARES: "5"             # Polymarket CLOB minimum for BTC markets
MOMENTUM_TRIGGER_START: "215"    # ~85s into 5-min window
MOMENTUM_TRIGGER_END: "255"      # ~45s into 5-min window
MOMENTUM_MAX_ENTRY: "0.75"       # skip if ask > this — protects against bad risk/reward
```

**Why max entry $0.75:** At $0.75, max upside is $0.25/share (3:1 risk/reward). At $0.94, max upside is $0.06/share (16:1 risk/reward — terrible). The cap prevents the engine from taking huge risks for tiny profits when the market is already heavily leaning one direction.

At ~$0.50-$0.75 per share with 5 shares, each trade deploys $2.50-$3.75.

## Order Type: FAK (Fill-And-Kill) — CRITICAL

Momentum follower uses FAK buy orders, NOT GTC. Why:

- **Midpoint scalp** used GTC at ask and filled fine — because entries were near $0.50 where liquidity is deep.
- **Momentum follower** entries can be at $0.65-$0.94 where liquidity is thinner. GTC orders at ask in these ranges sometimes sit unfilled, creating "ghost trades" in the DB but no actual positions on-chain.
- **FAK** guarantees: either fill immediately at the best available price, or cancel entirely. No ghost trades.

Implementation: `place_buy_fak()` in `app/live_trading.py`:

```python
def place_buy_fak(token_id, price, size, trade_id):
    result = _client.create_and_post_order(
        order_args=OrderArgs(token_id=token_id, price=price, size=size, side=Side.BUY),
        options=PartialCreateOrderOptions(tick_size="0.01"),
        order_type=OrderType.FAK,  # <— fills immediately or cancels
    )
```

**How to verify fills actually happened:** Check wallet balance after a trade fires. The `[LIVE HEALTH] balance=…` in engine logs should decrease by ~entry_cost × shares. If it doesn't move, the order didn't fill.

## Live-Only Enforcement

The engine has ZERO paper fallback for momentum. The code path in `evaluate_momentum()`:

```python
# Skip entirely if can't go live
if not want_live:
    return
# Insert with is_live=True
trade_id = insert_paper_trade(conn, ..., is_live=True)
# Place live order
buy_ok = live_trading.place_buy_fak(token, entry_cost, shares, trade_id)
if not buy_ok:
    # DELETE the row — no paper entry
    conn.execute("delete from paper_trades where id=?", (trade_id,))
    conn.commit()
```

If `MOMENTUM_LIVE=false` or live trading is disabled, the function returns before inserting anything — not even a paper row.

## Debug Visibility

The engine prints different messages at different stages:

| Message | Meaning |
|---|---|
| `[MOMENTUM DBG] s_left=N … in_window=False` | Outside trigger window (near 200-270s). Engine alive, waiting. |
| `[MOMENTUM] checking… s_left=N pct=X%` | Inside trigger window, BTC below threshold. Engine alive, no signal. |
| `[MOMENTUM] #ID DOWN/UP @ $0.XX ×5` | Signal fired, live order placed. |
| `[MOMENTUM] #ID FAILED (order rejected)` | Live order rejected by Polymarket (e.g. below minimum). Row deleted. |

When BTC is flat (<0.05% moves), you'll only see `DBG` and `checking…` messages. This is normal — the strategy requires meaningful momentum.

## Common Issues

### "Not firing" / "Is it working?"

Check: `docker logs polymarket-intel-crypto --tail 50 | grep MOMENTUM`
- If you see `checking…` or `DBG` lines → engine IS running, BTC just isn't moving enough
- If you see nothing → engine may not have hit a new window yet (5-min windows)
- If you see `FAILED` → order rejected, check shares ≥ 5

### Polymarket rejects 3-share orders

Error: `Size (3) lower than the minimum: 5`

Polymarket CLOB enforces 5-token minimum for BTC Up/Down markets. Orders below 5 are rejected silently if paper fallback exists. Always use ≥5 shares for momentum.

### Scanner auto-closes momentum positions

The scanner's mark-to-market loop auto-closes paper trades at +10% TP / -20% SL. The skip list in `app/main.py` line ~473 must include `"momentum_follower"`:

```python
if row["source"] in ("crypto_5m_engine", "midpoint_scalp", "momentum_follower"):
    continue
```

### No momentum trades in web UI

After a live-only restart, old paper-only momentum rows were deleted. Only `is_live=1` trades remain. If a live order fails, the row is deleted immediately — so failed attempts never appear in the UI. This is intentional: the UI only shows actual Polymarket positions.

## P/L Reset Procedure

When switching to a new strategy or after cleanup:

```bash
# 1. Get current wallet balance from engine logs
docker logs polymarket-intel-crypto --tail 20 | grep "LIVE HEALTH"
# Extract balance (USDC wei / 1_000_000)

# 2. Update DB baseline
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "UPDATE crypto_engine_state SET value='XX.XX' WHERE key='wallet_start_balance'"

# 3. Delete old strategy trades if needed
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "DELETE FROM paper_trades WHERE kind='momentum_follower' AND is_live=0"
```

The web UI reads `wallet_start_balance` from DB first (line ~727 of `web.py`), then falls back to the health log. DB value takes priority. The engine does NOT write this on startup — only manual SQL or the web UI updates it.
