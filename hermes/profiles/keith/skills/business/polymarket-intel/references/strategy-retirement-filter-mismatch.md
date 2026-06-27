# Strategy Retirement: Backend/Frontend Filter Mismatch Pattern

## The bug (hit 2026-06-26 when removing hedging)

### Setup
1. Engine stops producing `crypto_5m_profit_lock_hedge` trades
2. Frontend JS filters these trades out of strategy cards and table
3. All visible strategy cards show positive P/L

### The mismatch
4. `/api/summary` SQL query still includes hedge trades in `closed_pnl_pct`
5. 42 hedge trades (39 losses, 3 wins, total -$29.50, avg -0.70%) drag the overall metric down
6. **Metric card shows -29.53% while every strategy card is positive**

### Root cause
Frontend filters retired strategies but backend doesn't. The user sees:
- Strategy cards: +2.55%, +8.63%, +22.10% (all positive)
- Overall metric: -29.53% (contradicts the cards)

They understandably ask "why doesn't the math add up?"

### Full fix checklist
1. Add `AND kind != 'retired_kind'` to backend SQL in `/api/summary`
2. `DELETE FROM paper_trades WHERE kind = 'retired_kind'` (clean up DB)
3. Remove from `fmtStrategy()` name mapping in frontend
4. Add defensive filter in `updatePaperFilters()` so dropdown doesn't list it
5. Remove defensive JS filters in table/card rendering (no-ops without DB rows)
6. Rebuild web container

### Verification
```bash
# After fix, metric should match strategy cards
curl -s 'http://localhost:8095/api/summary?days=7' | python3 -c \
  "import json,sys; s=json.load(sys.stdin); print(f'closed_pnl: {s[\"closed_pnl_pct\"]:.2f}%')"

# Confirm 0 retired trades in DB
sqlite3 data/polymarket_intel.sqlite \
  "SELECT COUNT(*) FROM paper_trades WHERE kind = 'retired_kind'"
```

### Key insight
When retiring a strategy: **delete from DB first**, then fix both backend SQL AND frontend JS. The DB is the source of truth — filtering in only one layer guarantees mismatch.
