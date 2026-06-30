# Trend Detection + Dynamic Sizing + Momentum Follower (Live)

## Trend Detection

P/L-based detector using own trade data (not price action):

```python
def detect_trend(conn, event_slug):
    # Query last 6 resolved scalp windows
    # Per window: compare UP vs DOWN dollar P/L
    # Tally wins per side
    # One side ≥ 67% wins → lock trend
    # Mixed or <2 windows → NEUTRAL
```

**Zero lag:** No moving averages, no RSI, no MACD. Pure dollar-outcome lookback.

**Self-correcting:** If trend flips, old profitable windows fall out of the 6-window lookback and the detector shifts.

## Dynamic Shares

| Trend | UP | DOWN | Risk profile |
|---|---|---|---|
| UP | 5 shares | Blocked | One direction, full risk |
| DOWN | Blocked | 5 shares | One direction, full risk |
| NEUTRAL | 2 shares | 2 shares | Both sides, capped risk |

Config: `SCALP_SHARES` (5) and `SCALP_SHARES_NEUTRAL` (2).

Blocked sides: `evaluate_scalp()` skips the check entirely — no DB insert, no live order, no logging.

## Momentum Follower (LIVE as of 2026-06-29)

**Status: LIVE trading — no paper fallback.** Real orders on Polymarket CLOB.

**Strategy:** During trigger window (T+45s to T+85s), if BTC moves ≥0.05%, enter that direction and hold to resolution. No TP, no stop-loss, no spike exit. One entry per window max.

**Live-only enforcement:** If the Polymarket order fails, the DB row is deleted and the window is skipped. Zero paper entries are created.

**Config:**

| Var | Value | Meaning |
|---|---|---|
| `MOMENTUM_ENABLED` | true | Enable |
| `MOMENTUM_LIVE` | **true** | Live trading |
| `MOMENTUM_THRESHOLD` | 0.0005 | 0.05% BTC move |
| `MOMENTUM_SHARES` | **5** | Shares (**Polymarket minimum**) |
| `MOMENTUM_TRIGGER_START` | 215 | ~85s into the 5-min window |
| `MOMENTUM_TRIGGER_END` | 255 | ~45s into the 5-min window |

**Polymarket minimum:** BTC Up/Down markets require min 5 tokens per order. Orders with <5 shares are rejected. First trade at 2 shares worked as a fluke (token-specific), but all subsequent 2- and 3-share orders were rejected silently.

**Entry signal:**
```
[MOMENTUM] #NNNN DOWN @ 0.66 ×5 | BTC -0.050% in 45s LIVE
```

**Failed entry:**
```
[MOMENTUM] #NNNN FAILED (order rejected) — skipped
```

**Resolution:** Handled by same `resolve_pending_trades()` and `_close_all_expired_windows()` as midpoint_scalp — both sources share the resolution pipeline.

**Risk:** Higher entry price (BTC already moved) means larger loss if trend reverses. But profits are 6-8x larger per win (full resolution to 1.0 vs scalp's +0.05 TP). At 5 shares, each trade deploys $3.25-$4.25.
