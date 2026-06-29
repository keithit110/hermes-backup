# Paper vs Live Trading Separation

Use when building a trading dashboard that tracks BOTH paper-trading simulations AND live real-money orders from the same engine process. The user requires complete separation between paper and live stats — no mixing, no paper data masquerading as dollars.

## Core principle

Paper data and live data must be **visually and computationally separated** at every level:

- **DB**: Shared `paper_trades` table with `is_live` boolean flag
- **Backend**: Separate paper and live aggregations in `/api/summary`
- **Frontend**: Separate stats strips, filter drawer option, and conditional P/L $ rendering

## DB schema — `is_live` flag

```sql
ALTER TABLE paper_trades ADD COLUMN is_live INTEGER DEFAULT 0;
```

Set `is_live=1` when the engine actually places a real order. The engine must resolve this BEFORE `insert_paper_trade()`:

```python
token = state.up_token_id
is_live = bool(token and live_trading.is_enabled())
trade_id = insert_paper_trade(conn, ..., is_live=is_live)
if trade_id:
    print(f"[SCALP] OPEN #{trade_id} {'LIVE' if is_live else 'PAPER'}")
    if is_live:
        buy_ok = live_trading.place_buy_limit(token, price, shares, trade_id)
```

**Pitfall**: computing `is_live` after `insert_paper_trade()` means the row is already committed with `is_live=0`. Must resolve BEFORE the insert. Also must resolve `token` (e.g. `state.up_token_id`) before computing `is_live`.

## Backend — separate aggregations

In `/api/summary`, compute paper and live P/L separately:

```python
live_deployed = 0.0
live_returned = 0.0
for r in closed_trades:
    is_live = r.get("is_live") or 0
    if is_live:
        live_deployed += entry * shares
        live_returned += ret * shares

live_pnl_dollars = round(live_returned - live_deployed, 2)
live_pnl_pct = ((live_returned - live_deployed) / live_deployed * 100) if live_deployed > 0 else 0.0
open_live = one("select count(*) from paper_trades where status='open' and is_live=1").get("c", 0)
```

Return both paper and live fields in the JSON response — the frontend picks which to display where.

## Frontend — split stats strips

Two separate stats sections with labeled headers, NOT intermixed in one strip:

```html
<div class="stats-section-label">Paper Trading</div>
<div class="stats-strip" id="paperMetrics"></div>
<div class="stats-section-label">Live Trading</div>
<div class="stats-strip" id="liveMetrics"></div>
```

CSS for section labels:
```css
.stats-section-label {
  font-size: 9px;
  text-transform: uppercase;
  letter-spacing: .08em;
  font-weight: 700;
  color: var(--text3);
  padding: 0 2px;
  margin-bottom: 2px;
}
```

**Paper stats** — percentages, no dollar amounts:
```
Open Bets | Closed P/L % | Arb | Last Scan
```

**Live stats** — dollars first, then percentages:
```
Live P/L $ | Live P/L % | Live Deployed | Live Returned | Live Open
```

## Table — P/L $ column only for live trades

```javascript
['P/L $','is_live',(v,r)=>{
  if(!v || r.status==='open') return '<span class="muted">—</span>';
  let entry=Number(r.entry_cost||0), cur=Number(r.current_value||0), shares=Number(r.shares||1);
  let dollars=(cur-entry)*shares;
  return dollars>=0
    ? `<span class="pos">+$${dollars.toFixed(2)}</span>`
    : `<span class="neg">-$${Math.abs(dollars).toFixed(2)}</span>`;
}],
```

All paper trades (is_live=0) show `—` in the P/L $ column. Only live trades show actual dollar amounts.

## Filter drawer — "Trade Type" option

```html
<label>Trade Type</label>
<select id="paperLive">
  <option value="">All</option>
  <option value="1">Live Only</option>
  <option value="0">Paper Only</option>
</select>
```

Filter function:
```javascript
function applyPaperFilters(data){
  let f = getPaperFilters();  // now includes f.live
  // ... other filters ...
  if(f.live !== '') data = data.filter(r => String(r.is_live || 0) === f.live);
  return data;
}
```

## Live PnL summary card — only when live trades exist

Add a dedicated `<div id="livePnLSummary">` after the paper PnL cards. The render function returns empty string when no live trades exist, so the card appears only when there are actual live positions:

```javascript
function renderLivePnL(data){
  let live = data.filter(r => r.is_live == 1);
  if(!live.length) return '';
  // ... compute deployed, returned, net dollars ...
  return `<div class="pnl-card" style="border-color:#05966955;border-width:2px">
    <div class="kind">Live Trading</div>
    <div class="val">$${cDollars.toFixed(2)}</div>
    <div class="sub">Realized: ${cPnl}% (${closed.length} closed) · ${open.length} open</div>
  </div>`;
}
```

## Verification checklist

After implementing paper/live separation:
- [ ] `/api/summary` returns `live_deployed`, `live_returned`, `live_pnl_dollars`, `live_pnl_pct`, `open_live`
- [ ] Paper stats strip shows paper-only data (percentages, no dollars)
- [ ] Live stats strip shows $0.00 across all fields when no live trades exist
- [ ] P/L $ table column shows `—` for ALL rows when no live trades exist
- [ ] "Trade Type" filter in drawer: All / Live Only / Paper Only
- [ ] Selecting "Live Only" shows zero rows (when no live trades)
- [ ] Live PnL summary card is absent when no live trades
- [ ] Paper PnL cards still show paper-only percentages (not affected by new code)
