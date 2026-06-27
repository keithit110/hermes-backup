# Dashboard UI pitfalls (Chart.js + paper-trading dashboards)

Use when building or debugging web dashboards with charts, P/L displays, status pills, and strategy summary cards.

## Chart.js canvas height constraint

`responsive: true` + `maintainAspectRatio: false` without a height constraint causes the canvas to stretch to its full data height — often thousands of pixels, creating infinite scroll.

**Fix:** wrap `<canvas>` in a container with fixed height:

```html
<div class="card chart-card">
  <div class="label">Paper P/L Over Time</div>
  <div class="card-body"><canvas id="pnlChart"></canvas></div>
</div>
```

```css
.chart-card .card-body { position: relative; height: 240px; }
.chart-card canvas { max-height: 240px; }
```

Remove `height="220"` from the `<canvas>` element — Chart.js ignores it with responsive mode.

## P/L display: sum vs average

When aggregating per-strategy P/L in dashboard cards, **show average P/L per trade, not sum**. Summing causes misleading numbers:

- 2 trades at +69% and +22% → sum = +91% (wrong), average = +45.5% (correct)
- 176 trades each averaging +5.7% → sum = +1005% (alarming), average = +5.7% (accurate)
- 2 trades at -100% each → sum = -200% (mathematically impossible for per-trade P/L), average = -100% (correct)

**Fix:** compute `totalPnl / count` not `totalPnl`:

```javascript
let avgPnl = v.count > 0 ? v.totalPnl / v.count : 0;
html += `...${avgPnl.toFixed(2)}%...`;
```

## Double-multiplication between backend and chart

Common pattern: backend scales DB fractions to percentages, chart scales again:

```
DB: avg(pnl_pct) = 0.0445
Backend: * 100 → 4.45
Chart JS: * 100 → 445  ← BUG
```

**Fix:** ensure exactly one scaling point. Either:
- Backend returns raw fraction, chart does `* 100`, OR
- Backend returns percentage, chart uses value directly without `* 100`

Pick one and verify both metric cards and chart share the same source query.

## Chart and metric card must share the same data filter

If the metric card shows "Closed Paper P/L: 4.45%" from `avg(pnl_pct) WHERE status LIKE 'closed%'`, the chart must use the **same filter** — otherwise it averages in open trades (pnl_pct=0) and shows a lower or misleading line.

## Status pill formatting

When DB stores statuses with underscores (`closed_resolved_win`), display them as clean labels:

```javascript
function fmtStatus(s) {
  s = String(s || '').replace(/_/g, ' ');  // underscores → spaces FIRST
  if (s.startsWith('closed resolved win')) return pill('Won', 'won');
  if (s.startsWith('closed resolved loss')) return pill('Lost', 'lost');
  if (s.startsWith('closed take profit')) return pill('TP Hit', 'won');
  if (s.startsWith('closed window expired')) return pill('Expired', 'lost');
  if (s.startsWith('open')) return pill('Open', '');
  return pill(s, '');  // fallback
}
```

**Pitfall:** the `startsWith` strings must use spaces (post-replace), not underscores. `s.startsWith('closed_resolved_win')` will never match after `.replace(/_/g, ' ')`.

## Combined P/L for paired strategies (BTC directional + hedge)

Strategies that pair directional and hedge trades per window must show **combined net P/L**, not individual averages. Averaging individual pnl_pct values is misleading because the hedge always shows -100% when the directional wins — the user sees disconnected numbers.

**Fix — backend combined logic:**

```python
# Bucket BTC trades by event_slug, compute per-pair net P/L
btc_slugs = {}
for r in all_closed:
    if r["kind"].startswith("crypto_"):
        btc_slugs.setdefault(r["event_slug"], []).append(r)

combined_pnls = []
for slug, trades in btc_slugs.items():
    deployed = sum(t["entry_cost"] or 0 for t in trades)
    returned = sum(t["current_value"] or 0 for t in trades)  # NO status filter — current_value is already 0 for losers, 1 for winners
    if deployed > 0:
        combined_pnls.append((returned - deployed) / deployed)
# Other strategies: individual pnl_pct as-is
for r in other_rows:
    if r["pnl_pct"] is not None:
        combined_pnls.append(float(r["pnl_pct"]))

closed_avg = sum(combined_pnls) / len(combined_pnls) if combined_pnls else 0.0
```

**Fix — frontend combined card:** group by event_slug, sum deployed/returned across matched pairs, show `netPnl = ((totalReturned - totalDeployed) / totalDeployed) * 100`. Hide individual cryptos from the regular strategy cards.

**Must agree across all three:** metric card (backend `closed_avg`), strategy cards (frontend `computePnLSummary`), and P/L chart (backend `pnl_series`). All three must use the same combined logic — verify with `curl /api/summary` after any change.

**Verification command — after EVERY dashboard change:**

```bash
curl -s http://localhost:8095/api/summary?days=7 | python3 -c "
import json,sys
d=json.load(sys.stdin)
card = d['closed_pnl_pct']
chart = d['pnl_series'][-1]['v']*100 if d['pnl_series'] else 0
print(f'Card: {card:.2f}%  |  Chart: {chart:.2f}%  |  Match: {\"YES\" if abs(card-chart)<0.01 else \"NO\"}')
"
```

If `NO`, stop everything and fix before presenting to Keith. He considers any mismatch a hard bug.

## Table column ordering for betting dashboards

When a user complains about table clunkiness or excessive horizontal scroll, the fix is rarely just narrowing columns. **Reorder columns so the scanning priority matches what eyes actually look for.** The old default (timestamps first, P/L last) forces users to scroll right past noise to find the signal.

**Correct order for paper-trading / betting tables:**

| # | Column | Rationale |
|---|--------|----------|
| 1 | Market | Anchor — first thing you identify ("what did I bet on?") |
| 2 | Side | Visual pill — UP/DOWN, instantly tells direction |
| 3 | P/L | The money — eyes go here third for green/red signal |
| 4 | Status | Open/Resolved/Won/Lost pill — is it done? |
| 5 | Strategy | Short label — groups trades by strategy lane |
| 6 | Entry | Cost basis (narrow number) |
| 7 | Current | Current value (narrow number) |
| 8 | Reason | Why this trade was opened (truncated) |
| 9 | Opened | When placed (compact timestamp) |
| 10 | Closed | When resolved (compact timestamp) |
| 11 | Resolves | Deadline (compact timestamp) |

**Anti-pattern:** Full ISO timestamps (`2026-06-26T15:52:30.277407+00:00` — 32 chars) as columns 1 and 2. They consume 200+ horizontal pixels of noise before the user sees what market they bet on.

## Value compacting helpers

Long display values create wide columns. Use compacting helpers with lookup maps and regex:

### Timestamp compacting (ISO → human)

```javascript
function fmtTime(v) {
  if (!v) return '';
  // "2026-06-26T15:52:30.277407+00:00" → "06-26 15:52"
  let m = String(v).match(/^\d{4}-(\d{2})-(\d{2})T(\d{2}):(\d{2})/);
  if (m) return m[1] + '-' + m[2] + ' ' + m[3] + ':' + m[4];
  return String(v).substring(0, 16);
}
```

Saves ~21 characters per timestamp (32 → 11). Three timestamp columns = ~63 chars saved horizontally.

### Strategy name compacting (DB value → human label)

```javascript
function fmtStrategy(v) {
  let map = {
    'crypto_5m_late_directional': 'BTC Dir',
    'crypto_5m_profit_lock_hedge': 'BTC Hedge',
    'copy_single_high_win_rate_wallet': 'Copy Wallet',
    'copy_wallet_consensus': 'Copy Consensus',
    'research_paper_buy': 'Research Buy',
  };
  return map[v] || String(v || '');
}
```

Saves ~20 characters per strategy cell (31 → 8). Use a lookup map rather than regex replacement — strategy names may change and the map is explicit about what labels exist.

### tbl() function must apply column CSS classes

When column definitions include a 4th element (CSS class like `'hide-mobile'`), the table builder must actually apply it to `<th>` and `<td>`:

```javascript
// Before (broken — ignores class):
`<th onclick="sortPaper(${i})">${esc(c[0])}...</th>`
`<td>${...}</td>`

// After (fixed):
`<th onclick="sortPaper(${i})" class="${esc(c[3]||'')}">${esc(c[0])}...</th>`
`<td class="${esc(c[3]||'')}">${...}</td>`
```

When a user says they don't use certain tabs, remove them entirely — don't just hide them. Remove:
- Tab button HTML
- Section content HTML  
- JS fetch/render calls for that data
- Unused API endpoints can stay (no harm)

This reduces page load time (fewer API calls) and removes clutter.

## Docker rebuild pattern for web changes

Single-file web.py changes need full rebuild:

```bash
cd /root/polymarket-intel
docker compose stop web
docker compose build web        # Use --no-cache if COPY step shows CACHED
docker compose up -d web
curl -s -o /dev/null -w "HTTP %{http_code}" http://localhost:8095/
```

If the API still returns old values after deploy, the Docker COPY layer was cached — rebuild with `--no-cache`.

## Docker cached COPY layers (--no-cache)

When `docker compose build` shows `#10 CACHED` for the `COPY app ./app` step, the running container is serving STALE code. The Docker build cache didn't detect the file changes.

**Fix:** force rebuild without layer cache:
```bash
docker compose build --no-cache web
docker compose up -d web
```

**Symptom:** `curl` returns correct values but browser shows old behavior, OR backend API returns `-85%` when manual calculation shows `6.88%`.

**Always verify after deploy:**
```bash
curl -s http://localhost:8095/api/summary?days=7 | python3 -c "..."
```

## Auto-refresh with tab-awareness

When adding auto-refresh to a trading dashboard, use the Page Visibility API to skip refreshes when the tab is backgrounded — otherwise the page burns API calls and CPU while the user isn't watching.

### Pattern (single <script> block, no dependencies):

```javascript
let refreshTimer = null;
let autoRefreshOn = false;

function toggleAutoRefresh() {
  autoRefreshOn = !autoRefreshOn;
  try { localStorage.setItem('pm_autorefresh', autoRefreshOn ? '1' : '0'); } catch(e) {}
  updateAutoRefreshBtn();
  if (autoRefreshOn) startAutoRefresh(); else stopAutoRefresh();
}

function startAutoRefresh() {
  stopAutoRefresh();
  refreshTimer = setInterval(() => {
    if (document.visibilityState !== 'visible') return;  // tab is hidden — skip
    loadAll();
  }, 30000);  // 30 seconds
}

function stopAutoRefresh() {
  if (refreshTimer) { clearInterval(refreshTimer); refreshTimer = null; }
}

function updateAutoRefreshBtn() {
  let btn = document.getElementById('autoRefreshBtn');
  btn.textContent = 'Auto-Refresh: ' + (autoRefreshOn ? 'ON' : 'OFF');
  btn.className = autoRefreshOn ? 'btn btn-sm auto-on' : 'btn btn-sm';
}

// Restore state on page load
(function initAutoRefresh() {
  try { autoRefreshOn = localStorage.getItem('pm_autorefresh') === '1'; } catch(e) {}
  updateAutoRefreshBtn();
  if (autoRefreshOn) startAutoRefresh();
})();
```

### Key design decisions:
- **localStorage** key (not sessionStorage) — survives browser restart so the user doesn't need to re-enable every session
- **visibilityState gate** — `document.visibilityState !== 'visible'` returns early when tab is not the active tab, skipping the expensive `loadAll()` call
- **toggle button** in the filters bar next to the manual refresh button — always visible, clearly labeled ON/OFF with visual color change (gray vs green)
- **singleton timer** — `startAutoRefresh()` calls `stopAutoRefresh()` first to prevent timer leaks if toggled rapidly
- **graceful failure** — wraps localStorage in try/catch for environments where storage is disabled

### Integration checklist:
1. Add `<button id="autoRefreshBtn" class="btn btn-sm" onclick="toggleAutoRefresh()">Auto-Refresh: OFF</button>` next to the existing refresh button
2. Add CSS: `.auto-on { background: var(--green); color: #fff; }`
3. Add the full JS block above (timer vars, toggle, start, stop, update, init)
4. Verify: toggle ON → navigate to another tab → wait 35s → come back → data hasn't changed (no wasteful fetches)
5. Verify: toggle ON → stay on tab → wait 35s → data refreshed

## Scanner corrupting crypto engine paper trades

The Polymarket scanner (`main.py`, `update_paper_trades`) has an expiry handler at line ~508 that runs AFTER the crypto engine resolves its own trades. For ANY non-wallet trade with `source != "smart_wallet_copy"` whose `end_date` has passed, it sets `current_value=1.0` and `pnl_pct = (1.0 - entry) / entry` — assumed FULL WIN.

This overrides the crypto engine's correct resolution where hedges are properly marked as -100%. Result: crypto hedges show +500% to +900% instead of -100%.

**Fix — BLANKET skip at TOP of trade loop (NOT nested elif):**

```python
rows = store.conn.execute("select * from paper_trades where status='open'").fetchall()
for row in rows:
    # Crypto engine manages its own trades — scanner must not touch
    if row["source"] == "crypto_5m_engine":
        continue  # Skip BEFORE any mark-to-market, take-profit, or expiry logic
    
    details = json.loads(row["details_json"] or "[]")
    # ... rest of trade processing ...
```

A conditional skip buried inside a nested if-else chain (`elif row["source"] == "crypto_5m_engine": pass`) is NOT sufficient — the mark-to-market update at line ~481 runs BEFORE the expiry check and corrupts `current_value`. The skip must be the FIRST check in the loop.

**Corruption recovery:** Query Gamma API for each corrupted slug to recover the true winner, then back-update all rows. The Gamma API `outcomePrices` array shows resolution: `["1","0"]` means first outcome (Up) won, `["0","1"]` means second outcome (Down) won.

```python
# Recovery script pattern
for slug in corrupted_slugs:
    ev = requests.get(f"{GAMMA}/events/slug/{slug}").json()
    for m in ev["markets"]:
        outcomes = json.loads(m["outcomes"])  # ["Up", "Down"]
        prices = json.loads(m["outcomePrices"])  # ["0", "1"]
        for outcome, price in zip(outcomes, prices):
            if str(price) == "1":
                winner = outcome  # "Up" or "Down"
    # Update each trade: resolved_price = 1.0 if side==winner else 0.0
```

## Open trades in strategy cards — realized vs unrealized split

Strategy cards must distinguish open (unrealized) from closed (realized) trades:

- Open trades with status='open' → count separately, do NOT include in win/loss tally
- Win rate = wins / (wins + losses), NOT wins / total (which would include open)
- Display format: "4 trades · 3 closed (67% win) · 1 open"

```javascript
if (r.status === 'open') byKind[k].open++;
else if (Number(r.pnl_pct) > 0) byKind[k].wins++;
else if (Number(r.pnl_pct) < 0) byKind[k].losses++;
// Compute win rate from closed only
let closed = v.wins + v.losses;
let wr = closed > 0 ? ((v.wins / closed) * 100).toFixed(0) : null;
```

## UNREALIZED TRADES MUST NEVER SHOW P/L PERCENTAGES

This is Keith's strongest dashboard rule. Open/unrealized trades do NOT have a final P/L — showing a percentage based on mark-to-market is misleading.

**Table column fix — P/L column must hide for open rows:**

```javascript
// Before (broken — shows fake +5% or -100% for open trades):
['P/L','pnl_pct',v=>pct(v)],

// After (correct):
['P/L','pnl_pct',(v,r)=>r.status==='open'?'<span class="muted">—</span>':pct(v)],
```

**Strategy card fix — open-only strategies must show count, not percentage:**

```javascript
// If a strategy has zero closed trades, show "N open" instead of a fake P/L%
if(v.closed.length > 0){
  // Show realized P/L from closed trades only
  html += `<div class="pnl-card">...${cPnl.toFixed(2)}%...</div>`;
} else {
  // No closed trades — show count only, no percentage
  html += `<div class="pnl-card">...${v.open.length} open...</div>`;
}
```

**Example of what this prevents:**
- Research paper buy: 1 open trade with mark-to-market showing -100% → card would falsely show "-100.00%" as if it's a realized loss. Fix: shows "1 open" instead.
- Wallet copy trades showing +5% unrealized → misleading. Fix: shows "—" in P/L column.
