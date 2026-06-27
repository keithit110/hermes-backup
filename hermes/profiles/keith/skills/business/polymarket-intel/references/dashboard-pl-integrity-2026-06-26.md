# Dashboard P/L Integrity Fixes — 2026-06-26 Session

## Bugs fixed and root causes

### 1. Strategy cards showed SUM not AVERAGE (recurring)
- Symptom: COPY_HIGH_WIN_RATE_WALLET showed +1005.30% (sum of 176 trades × 5.71%)
- Root: `v.totalPnl.toFixed(2)` directly displayed the accumulated sum
- Fix: `(v.totalPnl / v.count).toFixed(2)` — display average per trade
- Reality check: 1005% / 176 = 5.71% which matched the aggregate Closed Paper P/L of ~5%

### 2. Chart double-multiplication
- Symptom: Chart Y-axis showed 586, tooltip said "Avg P/L: 445.021"
- Root: Backend `pnl_series` returned `v` already scaled to 100 (cumulative avg * 100), then chart JS did `v * 100` again
- Fix: Backend returns raw fraction (`v / 100`), chart does `v * 100` once

### 3. Metric card vs chart: different data subsets
- Symptom: Closed Paper P/L card = 4.45%, chart showed -19%
- Root: Metric card queried `status like 'closed%'`; chart queried ALL trades including open (0% P/L)
- Fix: Both use `status like 'closed%'` filter

### 4. BTC combined P/L: hedge losses double-counting
- Symptom: Hedge showed -200% (sum of two -100% trades)
- Root: Lane 3 hedge is insurance, not standalone — must be paired with Lane 2
- Fix: BTC Combined card computes net P/L = (total_returned - total_deployed) / total_deployed across both legs
- Individual crypto cards hidden from display

### 5. Open trades counted in closed P/L
- Symptom: User filtered to copy_single strategy, saw 1.92% but expected higher
- Reality: 1.92% IS correct — $0.06 gain on $3.12 deployed
- Fix: Strategy cards now show Realized vs Unrealized split explicitly
- Closed Paper P/L metric uses closed trades ONLY

## Design decisions made

- **Only 3 tabs**: Home, Paper Bets, Wallets. All others removed from UI.
- **Collapsible "What is this?"** section — collapsed by default
- **Inter font** for UI, JetBrains Mono for logs
- **Status pills**: clean labels (Won/Lost/Open/Expired/TP Hit) not raw DB values
- **Side pills**: color-coded UP (green) / DOWN (red)

## Verification checklist after any dashboard change

1. Closed Paper P/L metric card matches BTC Combined strategy card
2. Chart endpoint matches metric card (same data subset)
3. Strategy cards show average, not sum
4. BTC individual cards are hidden, only Combined shows
5. Status pills render clean labels, not raw DB values
6. Open trades don't contribute to Closed Paper P/L
