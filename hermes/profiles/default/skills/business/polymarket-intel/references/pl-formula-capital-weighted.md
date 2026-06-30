# P/L Formula: Capital-Weighted Portfolio Return

**Decision date**: 2026-06-26

## Problem

Three different P/L formulas coexisted across the dashboard:

| Formula | Where used | Why wrong |
|---------|-----------|-----------|
| Simple average of per-trade P/L% | Backend `/api/summary` | Equal-weights all trades. A -100% loss and a +20% win each count as 1. Does NOT reflect actual wallet return. |
| Capital-weighted without shares | Frontend strategy cards | Sums `entry_cost` dollars but ignores the `shares` column. A 3-share trade at $0.84 counts as $0.84 deployed instead of $2.52. |
| Capital-weighted × shares | **Correct** | `(total_returned − total_deployed) / total_deployed` where `total = Σ(entry_cost × shares)`. Mirrors a real wallet. |

The symptom: overall metric showed -29.53% while every strategy card was positive. The backend's simple average got dragged down by 42 hedge trades while the frontend filtered them out. After deleting hedges, the metric showed +4.07% while the BTC card showed +3.33% — still a mismatch because the frontend wasn't multiplying by shares.

## Solution

**Single formula everywhere**: capital-weighted return with shares multiplier.

```
return = (Σ(current_value × shares) − Σ(entry_cost × shares)) / Σ(entry_cost × shares)
```

This applies to:
- Backend `/api/summary` `closed_pnl_pct`
- Chart's cumulative P/L series
- Frontend `computePnLSummary()` — BTC Directional card AND non-BTC strategy cards
- Open trade "deployed" subtitles (for consistency, even though no P/L is shown)

## Verification checklist

After any dashboard or engine change, verify:

```bash
# 1. Backend metric
curl -s 'http://localhost:8095/api/summary?days=7' | python3 -c "
import json, sys
s = json.load(sys.stdin)
print(f'Metric: {s[\"closed_pnl_pct\"]:.2f}%')
print(f'Chart:  {s[\"pnl_series\"][-1][\"v\"]*100:.2f}%')
"

# 2. DB ground truth (shares-weighted)
sqlite3 data/polymarket_intel.sqlite "
SELECT ROUND((SUM(current_value * COALESCE(shares,1)) - SUM(entry_cost * COALESCE(shares,1))) / SUM(entry_cost * COALESCE(shares,1)) * 100, 2) 
FROM paper_trades WHERE status != 'open' AND entry_cost > 0;
"

# 3. Browser: Home tab metric == Paper Bets tab strategy card sum
```

All three must match within 0.01%.

## Common regression pattern

If you see the metric card showing a number that doesn't match what the strategy cards imply, check these three axes in order:

1. **Formula**: Is it capital-weighted (return/deployed) or simple average?
2. **Shares**: Is entry_cost and current_value multiplied by `shares`?
3. **Subset**: Are both including/excluding the same set of trades (same `kind` filter, same date range)?
