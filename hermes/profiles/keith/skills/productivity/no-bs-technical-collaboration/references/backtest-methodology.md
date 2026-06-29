# Backtest Methodology for Trading Engine Changes

## When Keith asks "what would have happened?"

Keith expects a data-driven answer BEFORE implementing. Never theorize — compute.

## Scenario A: historical data available

When engine logs or DB contain the needed raw inputs (pct_change, accel, time_factor):

1. Query all historical trades with their raw features
2. For each trade, run the NEW model/logic to compute what would have happened
3. Compare: old shares vs new shares, old entry yes/no vs new entry yes/no
4. Present: total P/L change, trade count change, blocked trades

## Scenario B: historical data missing (pruned logs)

When Docker container logs were pruned and pct_change is unavailable:

1. Query all closed trades from DB (entry_cost, pnl_pct, status)
2. Estimate missing features from what IS available:
   - `pct_change`: infer from entry_price (market_prob = 1 - entry_cost) and trade direction
   - For sensitivity analysis, test multiple pct_change levels (0.04%, 0.06%, 0.08%, 0.10%)
3. Compute new model behavior for each trade under each assumption
4. Present a sensitivity table

## Python backtest template

```python
import sqlite3
db = sqlite3.connect("/root/polymarket-intel/data/polymarket_intel.sqlite")
db.row_factory = sqlite3.Row

trades = db.execute("""
    SELECT id, entry_cost, pnl_pct, kind
    FROM paper_trades
    WHERE kind LIKE 'crypto_5m_%' AND status LIKE 'closed%'
    ORDER BY opened_at
""").fetchall()

# Old model: fixed 62% model_up, always 2 shares
# New model: proportional model_up = 0.50 + pct_change * 100, variable shares

total_old = sum(t['pnl_pct'] * 2 * t['entry_cost'] for t in trades)

for pct_assumption in [0.04, 0.06, 0.08, 0.10]:
    total_new = 0
    blocked = 0
    for t in trades:
        new_model = 0.50 + pct_assumption
        market_prob = 1.0 - t['entry_cost']
        edge = new_model - market_prob
        if edge < 0.05 or t['entry_cost'] > 0.85:
            blocked += 1
            continue
        shares = 4 if edge >= 0.40 else (3 if edge >= 0.30 else 2)
        total_new += t['pnl_pct'] * shares * t['entry_cost']
    print(f"pct={pct_assumption:.2f}%: return=${total_new:+.2f}  blocked={blocked}")

print(f"Old (2 shares): ${total_old:+.2f}")
```

## When blocked trades matter

If the backtest shows entries blocked, present:
- How many wins blocked (lost profit)
- How many losses blocked (saved capital)
- Net effect on P/L

If blocked trades are ALL losses, the filter is good. If blocked trades include profitable trades, show the net P/L impact.

## Sensitivity analysis output format

```
=== SENSITIVITY: varying assumed pct_change ===
  0.04%: return=$+5.54  blocked=0
  0.06%: return=$+6.76  blocked=0
  0.08%: return=$+7.71  blocked=0
  0.10%: return=$+6.55  blocked=0
Old (2 shares): $+2.96
```

If returns are positive across all assumptions → change is robust. If some assumptions show negative → flag to Keith.

## Presenting results

Always present:
1. Old P/L vs new P/L (absolute dollar amounts)
2. Trade count change
3. Blocked trades count
4. Whether the improvement is robust across assumptions
5. A clear "go/no-go" recommendation
