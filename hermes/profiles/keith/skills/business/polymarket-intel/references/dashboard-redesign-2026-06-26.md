# Dashboard Redesign — Session 2026-06-26

## Before (problems)

- Monospaced system font used for all UI text (not a single trading dashboard uses this)
- Strategy P/L cards showed **SUM** of all trade P/Ls, not average → COPY_HIGH_WIN_RATE_WALLET showed +1005.30% while aggregate card showed 4.95%
- -200.00% displayed for 2 trades at -100% each (mathematically impossible as an average)
- Status column showed raw DB values: `closed_resolved_loss`, `closed_window_expired`
- Side indicators were bare text arrows (▲ UP, ▼ DOWN) with no visual pill styling
- "What this dashboard means" was a permanent large card consuming prime real estate
- Reset button was red (destructive color for a non-destructive action)
- No hover feedback on cards
- Chart legends cluttered, grid inconsistent

## After

- **Font**: Inter (Google Fonts) for UI, JetBrains Mono for log blocks
- **P/L cards**: Show average per trade (`totalPnl / count`)
- **Status pills**: Won (green), Lost (red), Open, Expired, TP Hit
- **Side pills**: Color-coded pill-up (green bg) and pill-down (red bg)
- **Help section**: Collapsible toggle, hidden by default
- **Reset**: Neutral "Clear Filters" button
- **Cards**: Hover border transition
- **Charts**: No legends, clean grid, better color palette (#10b981 green, #3b82f6 blue, #f59e0b amber)

## Specific code fix: P/L average

File: `app/web.py`, function `computePnLSummary()`

```javascript
// BEFORE (line 188):
`${v.totalPnl.toFixed(2)}%`

// AFTER:
let avgPnl = v.count > 0 ? v.totalPnl / v.count : 0;
`${avgPnl.toFixed(2)}%`
```

## Specific code fix: Status formatter

File: `app/web.py`, function `fmtStatus()`

```javascript
function fmtStatus(s){
  s = String(s||'').replace(/_/g,' ');        // closed_resolved_loss → closed resolved loss
  if (s.startsWith('closed resolved win'))    return pill('Won', 'won');     // NOTE: spaces, not underscores
  if (s.startsWith('closed resolved loss'))   return pill('Lost', 'lost');
  if (s.startsWith('closed take profit'))     return pill('TP Hit', 'won');
  if (s.startsWith('closed window expired'))  return pill('Expired', 'lost');
  if (s.startsWith('open'))                   return pill('Open', '');
  return pill(s, '');
}
```

Key gotcha: `startsWith` strings must use **spaces** because `replace(/_/g,' ')` runs first. Using underscores in the startsWith strings silently fails — the status falls through to the default raw-text pill.

## Specific code fix: Chart data consistency

File: `app/web.py`, `/api/summary` endpoint (~line 391)

```python
# BEFORE — chart included ALL trades (open + closed), metric card used closed-only:
pnl_series = rows("select substr(opened_at,1,10) t, avg(pnl_pct) v
                   from paper_trades where opened_at >= ?
                   group by substr(opened_at,1,10) order by t", (cutoff,))

# AFTER — both chart and metric card filter to closed-only:
pnl_series = rows("select substr(opened_at,1,10) t, avg(pnl_pct) v
                   from paper_trades where opened_at >= ?
                   and status like 'closed%'
                   group by substr(opened_at,1,10) order by t", (cutoff,))
```

Symptom: June 26 showed -13.8% on the chart (14 trades including open ones at 0% P/L) but the metric card showed +4.45% (183 closed trades). After fix they match.

## Specific code fix: Chart canvas height containment

File: `app/web.py`, CSS and HTML sections

```css
/* CSS — constrain chart containers */
.chart-card .card-body { position: relative; height: 240px; }
.chart-card canvas { max-height: 240px; }
```

```html
<!-- HTML — wrap canvas in card-body div, remove height attribute -->
<div class="card chart-card">
  <div class="label">Paper P/L Over Time</div>
  <div class="card-body"><canvas id="pnlChart"></canvas></div>
</div>
```

Symptom: Chart.js with `responsive: true + maintainAspectRatio: false` caused the chart to stretch infinitely downward. The canvas `height="220"` attribute has no effect — Chart.js overrides it. Explicit CSS height on the parent container is required.
