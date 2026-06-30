# Directional Stop-Loss — implementation and risk/reward

## Problem

The BTC directional strategy (Lane 2) held positions to resolution with no mid-window exit. Risk/reward was 3.55:1 against:

| Metric | Value |
|--------|-------|
| Avg entry cost | $0.78 |
| Avg win (2 shares) | +$0.44 (+28.3%) |
| Avg loss (2 shares) | -$1.56 (-100%) |
| Win rate | 88% (15/17) |
| Required win rate to break even | 78% |

## Solution: -40% mark-to-market stop-loss with WS fast-path

`DIRECTIONAL_STOP_LOSS = 0.40` (-40%). Two trigger paths:

### Trigger path 1 — WS fast-path (primary, tick-level)

`_check_directional_sl_fastpath()` runs inside the Polymarket WebSocket callback — fires on EVERY orderbook tick (dozens/sec). When the OPPOSITE token's ask crosses `sl_opposite = 1.0 - (entry × 0.60)`, it adds the trade ID to `_sl_triggered`. The cycle checker's `_process_sl_close()` processes flagged trades immediately.

### Trigger path 2 — cycle checker (fallback, ~1s)

`check_directional_stop_loss()` checks every ~1s for `opposite_ask >= sl_opposite` OR `mark_price <= sl_price`.

### Why monitor the OPPOSITE token

In binary markets, the losing token crashes $0.50→$0.01 in one tick. The winning token rises gradually: $0.50→$0.55→$0.60→...→$1.00. Example: entry DOWN at $0.80, -40% SL → sl_opposite = 1.0 - 0.48 = $0.52. When UP ask crosses $0.52, SL fires and DOWN exits at ~$0.48 (-40% vs -98% without opposite monitoring).

### Status persistence (bug fix)

`resolve_pending_trades()` queries `status like 'closed%'` for Gamma API correction. Without a guard, it overwrites `closed_stop_loss` with `closed_resolved_win/loss`. Fix: query excludes `closed_stop_loss` — `status like 'closed%' and status != 'closed_stop_loss'`.

### Re-entry prevention

`freeze_window(conn, slug)` after SL. `evaluate_and_act()` checks `is_window_frozen()` before entry.

### New risk/reward

| | Before | After (-40% SL) |
|---|---|---|
| Win | +$0.44 | +$0.44 (unchanged) |
| Loss | -$1.56 (-100%) | -$0.62 (-40%) |
| Risk/reward | 3.55:1 against | 0.71:1 against |
| Required win rate | 78% | 42% |
| Margin | 10pts | 46pts |

## Config

```python
# crypto_engine.py
DIRECTIONAL_STOP_LOSS = float(os.getenv("DIRECTIONAL_STOP_LOSS", "0.30"))
```

```yaml
# docker-compose.yml
DIRECTIONAL_STOP_LOSS: "0.40"
```

## Key functions

- `_cache_sl_threshold(slug, trade_id, sl_opposite, side)` — cache on entry
- `_clear_sl_cache(slug)` — clear on close
- `_check_directional_sl_fastpath()` — WS callback check
- `_sl_triggered: set[int]` — flagged trade IDs for immediate close
- `_process_sl_close(conn, trade_id, now, sl_threshold_mult)` — close flagged trade
- `check_directional_stop_loss()` — cycle checker
- `freeze_window(conn, slug)` — prevent re-entry

History: 0.30 → 0.40 (2026-07-01).
