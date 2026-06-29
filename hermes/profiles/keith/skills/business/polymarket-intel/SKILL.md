---
name: polymarket-intel
description: Work on the Polymarket Intel paper-trading dashboard, crypto engine, scanner, and research agent. Covers project structure, dashboard design, deployment, common pitfalls, and P/L display conventions.
---

# Polymarket Intel

Paper-trading research platform at `/root/polymarket-intel`. Flask dashboard + crypto 5-min engine + market scanner + smart-wallet watcher + news research agent. The midpoint scalp strategy can go live (real USDC on Polymarket CLOB) when `LIVE_TRADING_ENABLED=true`; all other strategies are paper-only. Paper and live data are separated in the DB via `is_live` flag and displayed in separate UI sections — never mix paper dollar amounts into live displays.

## GitHub repository

- **Remote**: `git@github.com-polymarket:keithit110/Polymarket-hermes-project.git`
- **Deploy key**: `/root/.ssh/polymarket_ed25519` (write access enabled on GitHub)
- **`.gitignore`**: Excludes `.env*`, `data/*.sqlite`, `logs/`, `__pycache__/`, `*.pyc`, private keys
- **Full setup & secrets audit**: see `references/github-repo-setup.md`
- **Before every push**: Run secrets audit — staged files must contain zero API keys, private keys, or passwords. `.env.vpn` must NOT be tracked.

### Commit messages — revert-friendly

Keith wants commit messages that **explain the logic**, not just state the change. The commit must be self-contained enough that reverting to it restores the EXACT behavior, and reading it in the log reveals WHY the change was made. Format:

```
<action>: <what changed> — <why, including old vs new values and impact>

Example (good):
recalibrate crypto engine: proportional model_up replaces fixed 62%; share tiers recut (0.40/0.30 replaces 0.65/0.55).
Rationale: fixed 62% gave equal conviction to 0.04% and 0.12% moves. Old share thresholds (0.55/0.65) never triggered
— all trades got 2 shares. New model: 0.04%→54%, 0.08%→58%, 0.12%→62%. Revert with previous commit if results degrade.

Example (good):
fix: sync_final_results slug match was reversed — m_slug in slug should be slug in m_slug; add sub-slug match from details_json

Example (bad):
fix shares
update model
```

### Explanation style — concrete before abstract

When explaining a proposed change (model, algorithm, strategy), lead with **concrete examples using real data** before giving the abstract formula or rule. Keith needs to see what the change DOES to actual trades before understanding the math. Pattern: (1) show current behavior with a real trade example, (2) show what the proposal would produce for that same example, (3) compare. Then give the formula. Do not lead with the formula.

## Project structure

```
/root/polymarket-intel/
├── docker-compose.yml     # services: web, crypto, gluetun (VPN), scanner (one-shot)
├── app/
│   ├── web.py             # Flask dashboard (HTML template inline as PAGE)
│   ├── crypto_engine.py   # BTC 5-min lane-engine (Binance WS, Polymarket WS)
│   ├── main.py            # Scanner: market discovery, arbitrage, wallet scoring
│   └── db.py              # Shared DB helpers (used by crypto_engine)
├── scripts/               # Research candidates, cron scripts
├── data/                  # SQLite DB (polymarket_intel.sqlite)
└── logs/                  # latest_report.md, research_notes.md
```

## Services

| Service | Port | Network | Purpose |
|---------|------|---------|---------|
| `web` | 8095 (host) → 8080 (container) | default bridge | Flask dashboard (outbound API calls via gluetun HTTP proxy) |
| `crypto` | none exposed | `service:gluetun` | BTC 5-min engine (Chainlink + Binance + Polymarket WS) |
| `gluetun` | internal + proxy `:8888` | VPN + HTTP proxy | Surfshark WireGuard (Polymarket-allowed country: Canada, Netherlands, etc. — NOT UK/US/FR) |
| `scanner` | one-shot | default bridge | Market scan, arbitrage detection, wallet scoring (outbound via gluetun HTTP proxy) |

## VPN routing — Polymarket-allowed country REQUIRED (2026-06-28)

Polymarket geo-blocks: **US, UK, France, OFAC-sanctioned countries**. The VPN must exit a non-restricted country (Canada, Netherlands, Switzerland, Germany, etc.). The UK VPN that worked for read-only API calls blocks ALL order placement with `403 Trading restricted in your region`.

### Two routing methods

| Container | Method | Why |
|-----------|--------|-----|
| **crypto** | `network_mode: service:gluetun` | Long-running WebSocket connections need direct tunnel |
| **scanner** | `HTTP_PROXY=http://gluetun:8888` | One-shot batch job, uses `requests` library which auto-reads env vars |
| **web** | `HTTP_PROXY=http://gluetun:8888` | Future-proofing — dashboard API doesn't call Polymarket today but may later |

### Gluetun HTTP proxy setup

In `.env.vpn`, enable gluetun's built-in HTTP proxy:

```
HTTPPROXY=on
HTTPPROXY_LISTENING_ADDRESS=:8888
```

In `docker-compose.yml`, add proxy env vars to scanner, web, and crypto:

```yaml
environment:
  <<: *polymarket-env
  HTTP_PROXY: "http://gluetun:8888"
  HTTPS_PROXY: "http://gluetun:8888"
  NO_PROXY: "localhost,127.0.0.1,.local"
```

Python's `requests` library automatically respects `HTTP_PROXY`/`HTTPS_PROXY` — no code changes needed.

### Verification

```bash
# Crypto engine (network_mode tunnel)
docker exec polymarket-intel-crypto python3 -c "import urllib.request; print(urllib.request.urlopen('https://ipinfo.io/json', timeout=5).read().decode())"

# Scanner/web (HTTP proxy) — run a test container on same network
docker run --rm --network polymarket-intel_default \
  -e HTTP_PROXY=http://gluetun:8888 -e HTTPS_PROXY=http://gluetun:8888 \
  python:3.11-slim sh -c 'pip install -q requests; python3 -c "
import requests; r=requests.get(\"https://ipinfo.io/json\", timeout=10);
print(r.json().get(\"city\"), r.json().get(\"country\"), r.json().get(\"ip\"))"'
```

Both must show a Polymarket-allowed country (Canada, Netherlands, etc. — NOT UK/US/FR).

### Pitfall: forgetting to enable HTTP proxy in gluetun

Without `HTTPPROXY=on` in `.env.vpn`, gluetun's HTTP proxy port 8888 returns network errors (exit code 4). The containers have `HTTP_PROXY` set but the proxy isn't listening — all outbound Polymarket API calls fail or fall through to direct (US IP).

## Dashboard (web.py)

The entire HTML/CSS/JS is inline in the `PAGE` template string in `web.py`. No separate template files. Charts use Chart.js 4.x loaded from CDN.

### Tab layout

Only 3 tabs: **Home**, **Paper Bets**, **Wallets**. All other tabs (Markets, Arbitrage, News, Research, Logs) are REMOVED from the UI — they exist as internal data for the AI agents, not for Keith to browse. The JavaScript must not reference removed tab DOM elements (no `logsBox`, `researchBox`, `arbTable`, `marketsTable`, `infoTable`).

### Auto-refresh toggle (added 2026-06-26)

A toggle button in the filters bar controls 30-second auto-refresh:

- **Button**: `Auto-Refresh: OFF` (gray) / `Auto-Refresh: ON` (green) — in the filters bar next to Refresh
- **Interval**: 30 seconds
- **Page Visibility API gate**: `document.visibilityState !== 'visible'` — skips refresh when tab is backgrounded so it doesn't waste requests when nobody's watching
- **State persistence**: `localStorage.setItem('pm_autorefresh', ...)` — survives page reload and browser restart
- **Init on load**: reads `localStorage` and starts interval if it was ON

Implementation pattern (all inline in `web.py` PAGE template):

```javascript
let autoRefreshInterval = null, autoRefreshOn = false;

function toggleAutoRefresh() {
  autoRefreshOn = !autoRefreshOn;
  try { localStorage.setItem('pm_autorefresh', autoRefreshOn ? '1' : '0'); } catch(e) {}
  updateAutoRefreshBtn();
  if (autoRefreshOn) startAutoRefresh(); else stopAutoRefresh();
}

function startAutoRefresh() {
  if (autoRefreshInterval) clearInterval(autoRefreshInterval);
  autoRefreshInterval = setInterval(() => {
    if (!autoRefreshOn) return;
    if (document.visibilityState !== 'visible') return; // tab in background → skip
    loadAll().catch(() => {});
  }, 30000);
}

// Init on page load:
(function initAutoRefresh() {
  try { autoRefreshOn = localStorage.getItem('pm_autorefresh') === '1'; } catch(e) {}
  updateAutoRefreshBtn();
  if (autoRefreshOn) startAutoRefresh();
})();
```

CSS for toggle states:
```css
.btn-toggle { transition: all .15s; border: 1px solid var(--border); background: var(--surface2); color: var(--text2); }
.btn-toggle.on { border-color: #05966955; background: #064e3b33; color: var(--green); }
```

### Design conventions

- **Font**: Inter for UI, JetBrains Mono for log blocks
- **Theme**: Dark (OLED-deep navy), surfaced cards, blue accent
- **"What is this?" section**: Collapsible toggle, hidden by default
- **Status pills**: Clean labels (Won/Lost/Open/Expired/TP Hit), not raw DB values
- **Side pills**: Color-coded UP (green) / DOWN (red) badges
- **Strategy cards**: Show **capital-weighted return on deployed capital** — same formula as the overall metric card. Each position's entry_cost × shares counts toward the deployment base.

### CRITICAL: P/L display — capital-weighted portfolio return

ALL P/L values (metric card, strategy cards, chart) must use the SAME formula: **capital-weighted return = (total_returned − total_deployed) / total_deployed**. This mimics a real wallet — if you deployed $X and got back $Y, the return is (Y−X)/X.

The formula replaces the old "simple average of per-trade P/L%" approach (which overweighted small losing trades) and the old "sum entry_cost without shares" approach (which underweighted multi-share positions).

```javascript
// CORRECT: capital-weighted with shares
let totalCost = trades.reduce((s,r) => s + Number(r.entry_cost||0) * Number(r.shares||1), 0);
let totalReturn = trades.reduce((s,r) => s + Number(r.current_value||0) * Number(r.shares||1), 0);
let netPnl = totalCost > 0 ? ((totalReturn - totalCost) / totalCost) * 100 : 0;
```

```python
# CORRECT: capital-weighted with shares (backend /api/summary)
total_deployed += entry * shares
total_returned += ret * shares
closed_pnl_pct = ((total_returned - total_deployed) / total_deployed * 100) if total_deployed > 0 else 0.0
```

The old "average P/L per trade" approach is WRONG and has been removed. A 3-share -100% loss ($2.43 gone) must weigh 3x more than a 1-share +20% win ($0.17 gain) in the portfolio return.

### BTC Directional card (HEDGING REMOVED 2026-06-26)


BTC strategy is now **directional-only** — no more Lane 3 hedge. The engine buys UP or DOWN based on edge but never buys the opposite side. Each trade is stored as a single row with a `shares` column (1-3). The old `crypto_5m_profit_lock_hedge` trades are historical — do NOT display them in strategy cards, filters, or backend metrics.

Directional card: show wins/losses count and net P/L on deployed capital. Entry cost is per-share; the net return calculation works the same regardless of share count.

```javascript
let btcTrades = data.filter(r => r.kind === 'crypto_5m_late_directional');
let rest = data.filter(r => r.kind !== 'crypto_5m_late_directional');
let closedBtc = btcTrades.filter(r => r.status !== 'open');
let totalCost = closedBtc.reduce((s,r) => s + Number(r.entry_cost||0) * Number(r.shares||1), 0);
let totalReturn = closedBtc.reduce((s,r) => s + Number(r.current_value||0) * Number(r.shares||1), 0);
let netPnl = totalCost > 0 ? ((totalReturn - totalCost) / totalCost) * 100 : 0;
let wins = closedBtc.filter(r => Number(r.pnl_pct||0) > 0).length;
let losses = closedBtc.filter(r => Number(r.pnl_pct||0) <= 0).length;
```

### CRITICAL: Paper vs Live trading — never mix dollar amounts

Keith wants paper and live trading stats **completely separate** in the UI. Never show dollar amounts derived from paper trades — paper trades get percentages only. Live trades get their own dedicated stats section with exact dollar amounts. When Keith asks for dollar amounts, he means **live trading dollars only**, not paper-derived figures.

**Implementation (2026-06-28):**

- **`is_live` column** on `paper_trades` (INTEGER DEFAULT 0): set to 1 ONLY after a successful CLOB order placement. `insert_paper_trade()` always inserts with `is_live=False` initially. After `place_buy_limit()` returns a valid order ID, the engine runs `UPDATE paper_trades SET is_live=1 WHERE id=?` — the flag reflects REAL order execution, not intent. Engine logs show `LIVE` vs `PAPER` based on whether the buy succeeded (`did_live` flag), not whether live trading is enabled. **User-explicit correction (2026-06-28):** Keith caught the web UI displaying fake live dollar amounts because the old code set `is_live=True` at insert time BEFORE placing the order — when orders silently failed (geoblock, signer mismatch, version error), the DB falsely claimed live trades. Reset bad data with `UPDATE paper_trades SET is_live=0 WHERE is_live=1 AND kind='midpoint_scalp'` (stop engine first to release DB lock).

- **Stats strip** split into two labeled groups: "Paper Trading" (open bets, closed P/L%, arb, last scan) and "Live Trading" (P/L $, P/L %, deployed, returned, open). Live stats are dollar-focused; paper stats are percentage-focused.

- **P/L $ table column**: shows `—` unless `is_live=1`, then shows `(current_value - entry_cost) × shares` as a dollar amount. Uses the `is_live` column as the data source (not `pnl_pct`).

- **Live PnL summary card**: appears below paper PnL cards only when live trades exist. Shows realized dollar P/L, percentage, and open count.

- **Trade Type filter** in the drawer: All / Live Only / Paper Only. Filters by `is_live` value.

- **`/api/summary`** returns both paper totals (`total_deployed`, `total_returned`, `closed_pnl_pct`) and live totals (`live_deployed`, `live_returned`, `live_pnl_dollars`, `live_pnl_pct`, `open_live`).

- **`computePnLSummary()`** (paper cards) — percentages only, no dollar amounts in sub-text.

- **`renderLivePnL()`** — separate function for the live card. Returns empty string when no live trades exist.

**Pitfall — never compute dollar amounts from paper data**: When Keith asks "show me the dollar amounts," he means live trading dollars. Paper-derived figures like the $7.98 I computed from paper closed trades are NOT what he wants. If live trading hasn't produced any trades yet, the answer is "$0.00" — not a paper-derived number.

### Paper Bets table columns (order matters)

The Paper Bets table column order is: Market, Side, P/L, Status, Strategy, Shares, Entry, Current, P/L $, Reason, Opened, Closed, Resolves.

The **P/L $** column (added 2026-06-28) shows dollar P/L per trade — `—` for paper or open trades, `+$X.XX` / `-$X.XX` for closed live trades. Uses the `is_live` DB column as the data key (not `pnl_pct`).

**Shares column**: Always present. Bold (`<b>`) when >1 so high-conviction positions stand out visually. Populated from `paper_trades.shares`. All strategies have minimum 2 shares — crypto engine sets 2-4 via `_shares_from_edge()`, scanner INSERTs hardcode 2 for copy wallet, consensus, and arbitrage.

14. **Pagination**: 20 trades per page with Prev/Next buttons and "Page X of Y (N trades)" indicator. Prev disabled on page 1, Next disabled on last page. Changing filters (strategy, status, side), keyword, or day range resets to page 1. `paperPage` variable and `PAPER_PAGE_SIZE=20` control the slice. `renderPagination()` builds the control bar in a SEPARATE `#paperPagination` div OUTSIDE the scrollable `.table-wrap` — pagination buttons must stay stationary when the table scrolls horizontally. Appending pagination HTML inside the table-wrap causes buttons to scroll with the table.

15. **sync_final_results endpoint 403**: The original code queried `/markets?slug=X` — this endpoint returns 403 from non-browser user agents. The fix: use `/events/slug/{slug}` (same endpoint crypto engine uses). The event object contains all child markets with their `outcomePrices`. Match trades to markets by slug substring or question text. Do NOT check `m.get("closed")` — some events are resolved but the `closed` flag is stale. Check for any outcome with price ≥ 0.99.\n    **Immediate resolution in close loop**: Before marking a trade as `closed_pending_final_result_sync`, try to resolve it immediately by querying Gamma `/events/slug/{slug}`. If the event has a settled market (price ≥ 0.99), mark as `closed_won`/`closed_lost` directly. Only fall back to `closed_pending_final_result_sync` if Gamma doesn't have the result yet. This avoids the \"stuck pending\" state entirely.

16. **Immediate resolution in close loop**: Before marking a trade as `closed_pending_final_result_sync`, try to resolve it immediately by querying Gamma `/events/slug/{slug}`. If the event has a settled market (price ≥ 0.99), mark as `closed_won`/`closed_lost` directly. Only fall back to `closed_pending_final_result_sync` if Gamma doesn't have the result yet. This avoids the "stuck pending" state entirely.

17. **Staleness guard for copy trades** (`COPY_MAX_AGE_MINUTES = 15`): Compare `activity.timestamp` vs `now()` in `smart_copy_candidate()`. Skip signals older than 15 minutes. Prevents copying stale trades after scanner downtime. Current signals are all under 7 minutes — the guard is forward protection only.

18. **TAKE_PROFIT is 0.40 (40%)**: Set 2026-06-28 (updated from 0.20). All copy wallet, copy consensus, and research paper trades close at +40% mark-to-market gain. Stop-loss stays at -20%. Config: `TAKE_PROFIT: "0.40"` in docker-compose.yml (YAML anchor `&polymarket-env`, inherited by all services). After changing, rebuild scanner: `docker compose build scanner && docker compose up -d scanner --force-recreate`. History: 0.10 → 0.20 → 0.40.

20. **Share minimums (all strategies)**: Minimum 2 shares for every strategy. Crypto engine tiers: raw edge ≥65% → 4, ≥55% → 3, <55% → 2. Scanner INSERTs hardcode `shares=2` for copy wallet, consensus, and arbitrage. The raw (pre-time-decay) edge is stored in `details_json.raw_edge` and displayed in the dashboard Reason column as `[edge 45.1%]`.

**Tiered thresholds (updated 2026-06-27)**: 90+ wallets now use min_price=0.10 and min_size=5 (loosened from 0.30/15 on 2026-06-27 to capture tennis underdogs at 0.11-0.29 and micro-bets at 5-14 USDC). The 70-89 tier stays strict at 0.67/25. Copy Consensus shares the same tiered filter: a candidate must pass `smart_copy_candidate()` first, then gets checked for multi-wallet agreement.

21. **4-hour health check cron**: Job `fa2705b56898` runs every 4 hours, delivers to Keith's Telegram. Checks all containers, engine status, strategy P/L, VPN IP. Read-only — no trades or config changes.

Open/unrealized trades must show "—" in the table P/L column, not a percentage. The strategy card for a strategy with only open trades must show "N open" count, not a fake -100% or +X% mark-to-market. Only closed/realized trades get percentages. This applies to ALL strategy types.

**Reason column — raw edge display**: When crypto engine trades include `raw_edge` in `details_json`, the dashboard Reason column prepends `[edge 45.1%]` before the trade reason text. This shows the pre-time-decay edge used for share sizing, separate from the boosted edge shown in the reason string. Non-crypto trades (copy wallet, research) don't have `raw_edge` and show the reason unchanged.

**Health check cron**: `fa2705b56898` runs every 4 hours, delivers to this chat. Checks containers, engine status, strategy P/L, VPN IP. Status-only — no action required.

Table formatter: `(v,r) => r.status === 'open' ? '<span class="muted">—</span>' : pct(v)`

Card: when `v.closed.length === 0`, show `"${v.open.length} open"` as the value, not a P/L percentage.

### Realized vs unrealized split on strategy cards

Strategy cards must distinguish closed (realized) from open (unrealized) P/L. Build the card sub-text as:

```
Realized: +X% (N closed) · Unrealized: +Y% (M open) · total
```

If no closed trades exist, show only unrealized. If no open trades, show only realized. The dominant display value (the big number) comes from realized if available, otherwise unrealized.

### Closed Paper P/L metric card: CLOSED TRADES ONLY

The top-level metric card shows ONLY realized P/L from closed trades. Open trades do NOT contribute. This must match the strategy cards' realized portion. The chart must also use the same closed-only data subset.

### Status formatting

DB stores statuses with underscores. The `fmtStatus()` function must handle ALL these statuses:

```javascript
function fmtStatus(s){
  s = String(s||'').replace(/_/g,' ');
  if (s.startsWith('closed resolved win'))    return pill('Won', 'won');
  if (s.startsWith('closed resolved loss'))   return pill('Lost', 'lost');
  if (s.startsWith('closed resolution assumed')) return pill('Resolved', 'won');
  if (s.startsWith('closed stop loss'))       return pill('Stop Loss', 'lost');
  if (s.startsWith('closed take profit'))     return pill('TP Hit', 'won');
  if (s.startsWith('closed window expired'))  return pill('Expired', 'lost');
  if (s.startsWith('open'))                   return pill('Open', '');
  return pill(s, '');
}
```

Note: `closed_resolution_assumed` is a SCANNER status (used when scanner closes a trade before resolution data arrives). The engine sets `closed_resolved_win`/`closed_resolved_loss`. Never let the scanner set `closed_resolution_assumed` on crypto engine trades — the scanner must skip `source='crypto_5m_engine'` entirely.

## Engine deployment

```bash
cd /root/polymarket-intel

# Dashboard changes only:
docker compose stop web && docker compose build web && docker compose up -d web

# Crypto engine changes:
docker compose stop crypto && docker compose build crypto && docker compose up -d crypto

# Both:
docker compose stop web crypto && docker compose build web crypto && docker compose up -d crypto web
```

Verify web: `curl -sI http://localhost:8095 | head -1` → 200.
Verify engine: `docker logs polymarket-intel-crypto 2>&1 | tail -5` — no AttributeError or crash.

### Rollback tags

- `pre-one-entry-per-side` — state before `already_traded_side_in_window()` guard (commit `0777327`). **Note**: this guard was later removed in `25657be` because it killed scalp volume. The LIKE fix (pitfall #23) was the real solution.
- `25657be` (HEAD) — current: re-entry ENABLED, uses `has_scalp_position()` (open-only). One UP or DOWN at a time, but re-enters after TP close.

## Common pitfalls

1. **P/L formula: capital-weighted return, not average**: The overall metric, chart, and strategy cards all use `(total_returned − total_deployed) / total_deployed`. Do NOT revert to simple average of per-trade percentages — that overweighted small losses and was removed 2026-06-26.
2. **Status pill underscores**: After `replace(/_/g,' ')`, the `startsWith` strings must use spaces, not underscores.
3. **Chart and metric card MUST use the same formula AND data subset**: Both use capital-weighted cumulative return: `(cum_returned − cum_deployed) / cum_deployed`. Both exclude open trades. Both multiply entry_cost and current_value by `shares`. A mismatch on any of these three axes (formula, subset, shares) produces the "numbers don't add up" complaint. The chart builds cumulative deployed/returned by day; the metric does it across all closed trades at once — same formula, same data.
4. **Chart.js canvas height**: Use `height: 240px` on container div with `canvas { max-height: 240px }`. Chart.js overrides the canvas `height` attribute when `responsive: true`.
5. **Chart P/L double-multiplication**: Backend returns capital-weighted return as fraction (e.g., -0.0103 for -1.03%). Chart multiplies by 100 once: `data: s.pnl_series.map(x => x.v * 100)`. If backend also multiplies, value explodes 100x. Verify ONE side does the *100.
6. **Never show P/L for open trades**: Table must show "—", cards must show "N open" count. Only closed trades get percentages. This applies to ALL strategy types.
7. **Hedge trades are historical**: `crypto_5m_profit_lock_hedge` is no longer produced. Filter them OUT of UI display. Do not group BTC trades by slug — each directional trade is standalone.
8. **Engine `btc_mid` computation — Binance only (Chainlink RTDS dead as of 2026-06-28)**: `EngineState` has `btc_chainlink`, `btc_bid`, and `btc_ask`. The price fallback pattern `state.btc_chainlink if state.btc_chainlink > 0 else (btc_bid + btc_ask) / 2` still exists but Chainlink's RTDS WebSocket stopped sending data — `btc_chainlink` stays at 0.0. Binance is the effective sole BTC price source. All price lookups resolve to the Binance mid-price. The Chainlink subscription code is retained but dormant. See `references/binance-vs-chainlink-mismatch.md` for the latency comparison test results.

9. **Safety net `_close_all_expired_windows()` only checked `crypto_5m_engine` (fixed 2026-06-28)**: The original function only queried `source='crypto_5m_engine'` — scalp positions (`source='midpoint_scalp'`) expired silently and went to resolution as -100% losses. Fix: added a scalp-specific block that inserts `pending_resolve` entries for expired scalp windows.

10. **`resolve_pending_trades()` only handled `crypto_5m_engine` (fixed 2026-06-28)**: The function queried `source='crypto_5m_engine' and status like 'closed%'` — open scalp trades were invisible. Fix: added a second query for `source='midpoint_scalp' and status='open'` trades on resolved slugs.

11. **Re-entry loop: LIKE pattern was the real bug, not the guard (2026-06-28)**: The duplicate entry chaos had two phases. Phase 1: `already_traded_side_in_window()` blocked ALL re-entry per side per window — killed scalp volume. Phase 2: the real root cause was the LIKE pattern bug (pitfall #12 below). Original `has_scalp_position()` guard actually works — prevents opening a second same-side position while one is open, allows re-entry after TP close. This IS the intended midpoint scalp behavior.

12. **LIKE pattern bug: JSON matching requires space after colon (fixed 2026-06-28)**: Both `has_scalp_position()` and `already_traded_side_in_window()` used LIKE pattern `%"side":"UP"%` but `json.dumps()` produces `{"side": "UP"}` (SPACE after colon). The LIKE pattern matched ZERO rows, so both guards silently returned `False` for every call. Windows routinely had 6 UP + 5 DOWN entries. Fix: `f'%"side": "{side}"%'` — note the space.

13. **MAX_SPREAD caps entries before SCALP_MAX_OPEN**: In trending markets, spread widens immediately after first 1-2 entries, blocking the 3rd. Correct — don't want entries when order book is 8% wide.

14. **Live trading import path**: Engine runs as `python -m app.crypto_engine` in Docker (workdir `/app`, code at `/app/app/`). Import MUST be `from app import live_trading`, NOT `import live_trading`. Causes `ModuleNotFoundError` otherwise.

15. **Live trading credentials — auto-derive (2026-06-28)**: `py-clob-client` provides `derive_api_key()` which generates API credentials from wallet signature. The live_trading module auto-derives on startup. User only needs `POLYMARKET_PRIVATE_KEY` and `POLYMARKET_FUNDER` — no manual API key creation on polymarket.com needed.

16. **Live trading secrets in `.env.live` (2026-06-28)**: Credentials live in `/root/polymarket-intel/.env.live`, NOT in `docker-compose.yml`. The `.env.live` file is gitignored (`.env.*` pattern in `.gitignore`). `docker-compose.yml` references it via `env_file: - path: .env.live`. Only three vars needed: `LIVE_TRADING_ENABLED`, `POLYMARKET_PRIVATE_KEY`, `POLYMARKET_FUNDER`.

17. **Live trading diagnostic logging (2026-06-28)**: `live_trading.py` writes two JSONL log files:
   - `logs/live_orders.jsonl` — every BUY, SELL_TP, CANCEL_TP, and TP_FILLED event with timestamps
   - `logs/live_health.jsonl` — periodic (60s) heartbeat with balance, open TP count, fill count, error/success counters
   The `log_health_heartbeat()` function is wired into `evaluation_timer()` and called every cycle (internally throttled to 60s). Error counters track failures per order type (BUY, SELL_TP, CANCEL_TP).

18. **Docker log path: use `/logs/` not relative paths (2026-06-28)**: Inside the crypto container, code lives at `/app/app/` (Dockerfile `COPY app ./app`). Relative path `os.path.join(os.path.dirname(__file__), "..", "logs")` resolves to `/app/logs/` — but the Docker volume mounts `./logs:/logs`, not `/app/logs`. Fix: use `os.environ.get("LIVE_LOG_DIR", "/logs")` to write to the volume-mounted host directory. Always verify with `ls -la logs/live_*.jsonl` on the host after deployment.

19. **Balance check returns None — cosmetic (2026-06-28)**: `live_trading.get_balance()` may return None even when the CLOB client is initialized and order placement works. This is because `get_balance_allowance()` requires Level 2 auth that `derive_api_key()` may not provide. Order placement (Level 1) works fine. The health heartbeat will show `balance=$None` — this is normal.

20. **purge_old_data() crashes on DB lock (fixed 2026-06-28)**: The engine's startup purge can collide with a concurrent scanner run. Without retry logic, `sqlite3.OperationalError: database is locked` crashes the engine thread on startup. Fix: wrap in retry loop (5 attempts, 2s backoff) with `PRAGMA busy_timeout = 10000` before each attempt. If all attempts fail, log and skip the purge — don't crash the engine.

21. **Polymarket geo-blocks UK — VPN country must be allowed (2026-06-28)**: The VPN was set to `SERVER_COUNTRIES=United Kingdom`. Polymarket's CLOB API returns `403 Trading restricted in your region` for UK IPs. Change to a Polymarket-allowed country: Canada, Netherlands, Switzerland, Germany. Edit `.env.vpn`, then `docker compose down gluetun && docker compose up -d gluetun`. Verify with `docker exec polymarket-intel-crypto python3 -c "import urllib.request,json; ip = urllib.request.urlopen('https://api.ipify.org', timeout=10).read().decode(); info = json.loads(urllib.request.urlopen(f'http://ip-api.com/json/{ip}', timeout=10).read()); print(info['country'], info['countryCode'])"`.

22. **Live orders fail silently with 403/400 — check live_orders.jsonl (2026-06-28)**: When the engine prints `[SCALP] OPEN #N ... LIVE` but no fills appear, the orders were placed but rejected. Check `docker exec polymarket-intel-crypto cat /logs/live_orders.jsonl | tail -5 | python3 -m json.tool` for the error field. Common causes: (a) geoblock (403 — wrong VPN country), (b) invalid order version (400 — upgrade to v2 SDK), (c) signer address mismatch (400 — need pre-created API keys for deposit wallets), (d) insufficient balance.

23. **py-clob-client v1 deprecated (2026-06-28)**: Polymarket now requires `py-clob-client-v2>=1.0`. The v1 package returns `400 invalid order version, please use the latest clob-client` on every order. Migrate to v2: see Live Trading section for full API surface differences.

24. **is_live flag false positives — reset DB when orders silently fail (2026-06-28)**: The engine's `is_live` flag is now set ONLY after successful order placement. Previously it was set at insert time — when orders failed (geoblock, signer mismatch), the DB falsely claimed live trades. If the web UI shows live dollar amounts for trades you know never executed: (1) stop the engine, (2) run `sqlite3 data/polymarket_intel.sqlite "UPDATE paper_trades SET is_live=0 WHERE is_live=1 AND kind='midpoint_scalp'"`, (3) verify with `SELECT COUNT(*) FROM paper_trades WHERE is_live=1`, (4) restart. Verify with `curl -s http://localhost:8095/api/summary | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['live_pnl_dollars'])"` → must be 0.0.
11. **BTC 5-min resolution = Chainlink, but our engine uses Binance**: Polymarket resolves BTC 5-min markets on Chainlink BTC/USD data. Our engine uses Binance as the real-time price source (Chainlink RTDS WebSocket is dead — confirmed 2026-06-28). The resolution check `_check_market_resolution()` queries Gamma API which returns the official Chainlink-based outcome — so resolution is correct. The Binance-vs-Chainlink mismatch only affects our best-guess close in `_close_old_window_positions()`, which is always overwritten by official resolution within minutes. Before 2026-06-26 the engine used Binance for both entry AND resolution — this caused a -100% loss when Binance showed +0.14% UP but Chainlink resolved Down. The fix: use Binance for entries (only working source), Gamma API for resolution (official Chainlink outcome).
12. **Misleading "below min 0.05" log message**: When the engine logs `edge_up=-80.8% edge_down=80.9% below min 0.05`, the edge_down of 80.9% TECHNICALLY passes the `>= MIN_EDGE` check. The real rejection is `MAX_ENTRY_PRICE` — `down_ask` has already surged above 0.85 because the market repriced before the engine could enter. The message comes from the else-branch at line 288 which fires when NEITHER direction satisfies BOTH conditions (`edge >= MIN_EDGE AND ask <= MAX_ENTRY_PRICE`). BTC moving clearly → market reprices instantly → engine detects the edge but can't enter at a price above 0.85. This is correct behavior — the engine is protecting against late entries, not missing opportunities.
13. **No crypto entries during low volatility is normal**: Three filters correctly block entries when BTC is flat: (a) `pct_change` too close to 0 — no directional signal, (b) outside 60-180s window — too early or too late, (c) ask > 0.85 — market already repriced the move. An hour with zero entries during range-bound BTC is the engine working as designed, not a bug. Check `docker logs polymarket-intel-crypto 2>&1 | grep IGNORE` before investigating.

    **pct_change floor**: Set via `CRYPTO_MIN_PCT_CHANGE` env var (current: 0.0004 = 0.04%). This is the minimum BTC price movement in a 5-min window before the engine computes edge. Below this, the engine skips (`IGNORE: pct_change too close to 0`). At $60K BTC, 0.04% = ~$24 move. The floor prevents 100s of useless evaluations per window when BTC is dead flat. **Only adjust with Keith's approval.** See `references/crypto-engine-debugging.md` for the full diagnostic workflow when the engine stops producing trades.

    History: 0.1% → 0.08% (2026-06-26) → 0.04% (2026-06-28, catches moves during dead-flat BTC consolidation).

14. **Crypto engine code changes not deploying**: `docker compose build crypto` may use a cached layer if the Dockerfile hasn't changed. After multiple code-only changes, use `docker compose build crypto --no-cache` to force a full rebuild. Verify with `docker exec polymarket-intel-crypto grep '<unique_string>' /app/app/crypto_engine.py` that the expected code is in the container.

15. **Strategy gating pitfall — Lane 2 `has_open_position` blocked Lane 3**: The original `evaluate_and_act()` had `if has_open_position(): return` BEFORE the scalp evaluation. This meant if Lane 2 had a position, the ENTIRE function returned and Lane 3 (scalp) never ran. The fix (2026-06-28): move the `has_open_position` check inside the Lane 2 block only. Lane 3 runs independently with its own `has_scalp_position()` check. The original code had three bugs: (a) line 888 checked `m_slug in slug` (reversed — market slug is always longer than event slug, e.g. "fifwc-col-prt-2026-06-27-draw" in "fifwc-col-prt-2026-06-27" = False), (b) the fix `slug in m_slug` matched the FIRST resolved sub-market in Gamma response order, not the specific market the trade bet on, (c) consensus and copy trades didn't store sub-market slug in `details_json["slug"]`. Fixes applied across three commits (`1d938dd`, `64407d4`, `851ed58`):

    - **Matching priority (highest → lowest)**: (1) exact sub_slug from `details["slug"]` == `market["slug"]` — most precise, (2) event slug prefix `slug in m_slug` — fallback for trades without sub_slug in details, (3) question text match `wanted in m_question` — last resort.
    - **New trades**: `smart_copy_candidate()` now stores `"slug"` from `a.get("slug")` (Polymarket sub-market slug) in the candidate dict. Consensus trades store `best["slug"]` in `details_json`. This ensures future trades match their specific sub-market on resolution.
    - **Retroactive fix**: Trades #370, #406, #423 were incorrectly resolved by the prefix-match bug. #370 (Colombia win/No at 0.71) was closed_lost → corrected to closed_won (+40.8%). #406 (consensus Colombia win/No) corrected from closed_lost to closed_won (+39.3%). #423 (Colombia win/Yes at 0.23) corrected from closed_won to closed_lost (-100%).
    - **Unresolvable edge case**: Trade #365 (ATP Bergs vs Humbert) — Gamma shows `active=True, closed=False, endDate=2026-07-04`. Match was postponed from June 27. No market has price ≥0.99. This is correct — nothing to resolve until July 4.
    - See `references/scanner-debugging.md` for full diagnostic workflow.

### CRITICAL: Backend and frontend must apply the SAME filters

When a strategy is retired (e.g., hedging removed), BOTH the frontend and backend must exclude its trades. A mismatch causes the overall metric card and the strategy cards to show different numbers — the single most common cause of "the numbers don't add up" complaints.

**Concrete example (2026-06-26 hedge removal):** The frontend JS filtered out hedge trades from the table and strategy cards, so all strategy cards showed positive P/L. But `/api/summary`'s SQL query included all 42 hedge trades (total -$29.50, average -0.70%) in the overall `closed_pnl_pct` calculation. Result: metric card showed -29.53% while every strategy card was positive. Fix: add `and kind != 'crypto_5m_profit_lock_hedge'` to the backend SQL AND delete the historical rows from the DB.

**The strategy filter dropdown is populated from live DB data** in `updatePaperFilters()` — it scans `allPaperData` for unique `kind` values. Frontend JS filters won't help because the dropdown populates before the table filter runs. To remove a retired strategy from the dropdown, either:
- Delete its trades from the DB (cleanest), OR
- Add a filter in `updatePaperFilters()`: `if(r.kind!='old_strategy_kind') strategies.add(r.kind)`

**Pattern when retiring a strategy completely:**
1. Stop producing new trades of that kind (engine/scanner change)
2. Delete historical trades from DB: `DELETE FROM paper_trades WHERE kind = 'old_kind'`
3. Add SQL exclusion in backend `/api/summary` (belt-and-suspenders)
4. Remove from `fmtStrategy()` name mapping
5. Remove defensive JS filters (they're no-ops without DB rows)
6. Consider keeping a defensive filter in `updatePaperFilters()` in case old trades reappear

### Architecture

- **Two independent strategies**: (1) BTC 5M Directional — fades extreme market odds, (2) Midpoint Scalp — buys near $0.50, sells on small bounces. Separate sources, separate DB rows, no shared position gating.
- **Directional (source=`crypto_5m_engine`, kind=`crypto_5m_late_directional`)**: Buys UP or DOWN based on edge when market gives our direction <25% chance. Flat 2 shares. Proportional model_up (0.50 + pct_change×100).
- **Midpoint Scalp (source=`midpoint_scalp`, kind=`midpoint_scalp`)**: Buys tokens at $0.47-0.53 when market is uncertain. Take-profit at entry + $0.04 bid move. Holds losers to resolution (matches nj23adsknml3 pattern — no deadline bail-out). Max 3 open positions. Flat 1 share (changed from 2 on 2026-06-28 for live trading). JSONL decision log at `logs/scalp_decisions.jsonl`. Re-entry: `has_scalp_position()` prevents opening a second position while one exists, but allows re-entry after a TP close — this IS the intended scalp behavior.
- **1-second evaluation** (changed from 3s on 2026-06-28): `evaluate_and_act()` + `check_scalp_take_profits()` run every ~1s. Slow API calls every 5th cycle (~5s). The 1s cycle catches micro-oscillations where price crosses the $0.47/$0.53 boundary in under 3 seconds — critical for live midpoint scalping. CPU usage increase is minimal since the evaluation is fast math-only.
- **BTC acceleration tracking**: Records `btc_halfway_price` at ~150s mark. Same direction in both halves → +5% confidence boost. Reversal → -8% penalty.
- **Time-decay boost**: Later in window → more confidence in move. `time_factor = 1.0` at 180s → `2.0` at 60s.
- **Entry max price (directional)**: Skip any window where entry > $0.85.
- **65-day data retention**: `purge_old_data()` runs on startup and every ~3 hours.
- **Window close handles both sources**: `_close_old_window_positions()` closes `crypto_5m_engine` AND `midpoint_scalp` open trades on window expiry.

### Live trading — midpoint scalp only (2026-06-28, v2 SDK)

Module `app/live_trading.py` provides Polymarket CLOB v2 order placement via `py-clob-client-v2>=1.0`. Only the midpoint scalp strategy goes live — directional (Lane 2) stays paper-only.

**⚠️ v1→v2 migration (June 2026):** The deprecated `py-clob-client==0.34.6` returned `400 invalid order version` on all orders. Polymarket now requires the v2 SDK. Key differences:
- Import from `py_clob_client_v2`, not `py_clob_client`
- `ClobClient(host=host, chain_id=137, key=private_key)` — requires `chain_id`
- Two-step auth: L1 `create_or_derive_api_key()` → yields `ApiCreds`, then L2 `ClobClient(..., creds=creds)` for order placement
- `create_and_post_order(order_args, options, order_type)` replaces `create_order()` + `post_order()`
- `Side.BUY` / `Side.SELL` enums, not strings
- `OrderType.GTC` enum, not string `"GTC"`
- `PartialCreateOrderOptions(tick_size="0.01")` must be passed
- `cancel_order(order_id)` not `cancel(order_id)`
- `get_balance_allowance(asset_type=AssetType.COLLATERAL)` not `get_balance()` — and the asset type is `COLLATERAL`, not `USDC`
- Market orders use `MarketOrderArgs(token_id, amount, side, order_type)` + `create_and_post_market_order()` — amount is USDC, not token count

**Relayer API keys vs CLOB API keys — Polymarket UI confusion (2026-06-28):** Polymarket's Settings page labels API keys as "Relayer API Keys" in the UI, leading users to believe only relayer keys exist. These ARE the CLOB API keys — but key creation in the UI (https://polymarket.com/settings/api-keys) requires the proxy to be fully deployed first. Until the proxy is deployed, the Settings page only shows Relayer API key options. After proxy deployment, CLOB API key creation becomes available.

**API key derivation — SDK bug (L1 auth always binds to EOA):** `create_or_derive_api_key()` logs a `400 "Could not create api key"` error internally but falls back to derivation successfully. The returned `ApiCreds` object works for L2 auth despite the logged error — this is cosmetic for EOA wallets but **causes order rejection for deposit wallets** (`400 maker address not allowed` / `400 the order signer address has to be the address of the API KEY`).

**CRITICAL — py-clob-client-v2 SDK monkeypatch for POLY_1271 L1 auth (2026-06-28):** The SDK's `create_level_1_headers()` and `sign_clob_auth_message()` always use `signer.address()` (the EOA) as `POLY_ADDRESS` and in the ClobAuth message, even when `signature_type=POLY_1271` and `funder` is set. This means the API key is ALWAYS EOA-bound regardless of signature type. Fix by monkeypatching in `live_trading.py` BEFORE client init:

```python
# In live_trading.py init(), after imports, before ClobClient init:
from py_clob_client_v2.headers import headers as _hdr_mod
from py_clob_client_v2.signing import eip712 as _eip712_mod

def _patched_create_l1(signer, nonce=None, timestamp=None,
                       signature_type=None, funder=None):
    from datetime import datetime
    ts = timestamp if timestamp is not None else int(datetime.now().timestamp())
    n = nonce if nonce is not None else 0
    use_funder = (signature_type is not None
                  and int(signature_type) == 3  # POLY_1271
                  and funder)
    funder_addr = funder if use_funder else None
    # Sign ClobAuth with funder address when POLY_1271
    actual_addr = funder_addr if funder_addr else signer.address()
    # ... build ClobAuth(addr=actual_addr, ...) and sign ...
    poly_addr = funder if use_funder else signer.address()
    return {POLY_ADDRESS: poly_addr, ...}

_hdr_mod.create_level_1_headers = _patched_create_l1

# Also patch ClobClient._l1_headers to pass sig_type + funder:
ClobClient._l1_headers = lambda self, nonce=None: _patched_create_l1(
    self.signer, nonce=nonce, timestamp=self._get_timestamp(),
    signature_type=getattr(self.builder, 'signature_type', None),
    funder=getattr(self.builder, 'funder', None),
)
```

This fix alone is NOT sufficient for order placement — the proxy must ALSO be deployed (see Deposit wallet section below). The monkeypatch ensures `POLY_ADDRESS` is the funder/deposit wallet, but the backend still rejects orders if the proxy has insufficient bytecode for EIP-1271 verification.

**Deposit wallet workaround (2026-06-28, UPDATED 2026-06-28):** Polymarket's deposit wallets use `SignatureTypeV2.POLY_1271`. The `create_or_derive_api_key()` method binds the API key to the EOA (private key owner), but the CLOB requires the deposit wallet address as the order signer. This causes `400 the order signer address has to be the address of the API KEY` on every order. **Root cause: the deposit wallet proxy contract is counterfactual** — it has an address but minimal bytecode (~23 bytes) until the first on-chain transaction. Without full bytecode (~250+ bytes), EIP-1271 signature verification fails.

**Verified on-chain state (2026-06-28):** Proxy at `0xa5Df31bB4cDD4c94E789C6D7ac302662EE7934B9` (EOA = funder = same address for Polymarket deposit wallets) has 23 bytes of bytecode. This is insufficient — orders will fail regardless of SDK patches or API key recreation. After a successful UI trade, the proxy gains ~250 bytes and orders work.

**CRITICAL — verified fix workflow (do in order):**

1. **Deploy the proxy**: Place ONE small order through https://polymarket.com in your browser (with MetaMask). This deploys the proxy contract (~250 bytes) AND sets unlimited USDC allowances for the exchange contracts. Verify with:
   ```python
   from web3 import Web3
   w3 = Web3(Web3.HTTPProvider('https://polygon-mainnet.g.alchemy.com/v2/demo'))  # or any Polygon RPC
   code = w3.eth.get_code(Web3.to_checksum_address(FUNDER))
   print(f'Proxy deployed: {len(code)} bytes')  # must be ~250+, not 23
   ```
2. **Delete old EOA-bound API keys**: Init ClobClient with derivation → `get_api_keys()` → `delete_api_key()` for each. Old keys bound to EOA will reject orders even after proxy deployment.
3. **Create fresh API keys**: With the monkeypatch applied (L1 headers now use funder address), call `create_api_key()`. After proxy deployment, this succeeds and the new key is bound to the deposit wallet.
4. **Set env vars for production** — the `init()` function checks for these first; if present, skips `create_or_derive_api_key()` entirely and builds `ApiCreds` directly:
   ```bash
   POLYMARKET_API_KEY=...
   POLYMARKET_API_SECRET=...
   POLYMARKET_API_PASSPHRASE=...
   ```

```bash
# .env.live additions for deposit wallets
POLYMARKET_API_KEY=...
POLYMARKET_API_SECRET=...
POLYMARKET_API_PASSPHRASE=...
```

The `init()` function checks for these env vars first; if present, skips `create_or_derive_api_key()` entirely and builds `ApiCreds` directly. Both ClobClient instances (L1 auth + L2 order) use `signature_type=SignatureTypeV2.POLY_1271` and `funder=funder_address`.

**Module capabilities:**
- `init()` — Initialize `ClobClient` from env vars. Idempotent. Uses pre-created API keys if available; falls back to `create_or_derive_api_key()`. Both ClobClient instances use `SignatureTypeV2.POLY_1271` and `funder` address.
- `place_buy_limit(token_id, price, size, trade_id)` — Place a GTC BUY limit order. Returns order ID or None on failure.
- `place_sell_tp(token_id, price, size, trade_id)` — Place a GTC SELL limit order for TP. Tracks order ID for cancellation.
- `place_market_sell(token_id, size, trade_id)` — Place a FOK market SELL order. Uses `MarketOrderArgs` with `amount=size` (USDC), `side=Side.SELL`, `order_type=OrderType.FOK`. Used for spike exit and near-close opportunistic exit.
- `cancel_tp(trade_id)` — Cancel an unfilled TP order via `cancel_order()`.
- `get_fill_status(trade_id)` — Poll exchange for TP fill. Returns `{"filled": True/False}`.
- `log_health_heartbeat()` — Log health snapshot every 5 minutes (balance, open TPs, fill count, error/success counters).

**Env vars needed for live (in `/root/polymarket-intel/.env.live`, NOT docker-compose.yml):**

`.env.live` is gitignored (`.env.*` pattern in `.gitignore`). Docker Compose loads it via `env_file: - path: .env.live` on the crypto service.

```bash
LIVE_TRADING_ENABLED="true"
POLYMARKET_PRIVATE_KEY="a1b2c3…"  # Polygon wallet private key — 0x prefix optional, code handles both
POLYMARKET_FUNDER="0x…"           # Wallet public address
```

Only three vars. API credentials (key/secret/passphrase) are **auto-derived** from the wallet signature via `py-clob-client`'s `derive_api_key()` on startup — no manual API key creation on polymarket.com needed. The `0x` prefix on the private key is optional; the code strips/re-adds it as needed via a normalise function.

**Live trading flow per entry:**

```
evaluate_scalp() decides ENTER
  → insert_paper_trade(is_live=False)     ← always starts as PAPER
  → live_trading.place_buy_limit()         ← real BUY order (if enabled)
  → if buy succeeds: UPDATE is_live=1      ← only mark live AFTER confirmation
  → if buy succeeds: place_sell_tp()       ← real SELL TP order

Every ~5s:
  → _check_live_scalp_fills()            ← polls exchange for TP fills
  → If filled → update DB

Window expires:
  → _close_all_expired_windows()         ← queue for resolution
  → live_trading.cancel_tp()             ← cancel unfilled TP orders (TODO: wire this)
```

**WebSocket fill detection:** The current implementation POLLS for fills via `get_order()` every 5 seconds. A WebSocket-based fill listener (Polymarket's User Channel) would be more efficient but requires authentication — the public Market Channel doesn't send user-specific fill events. This is a future optimization, not a blocker.

**Directional stays paper-only:** Lane 2 (`evaluate_lane_2()`) only calls `insert_paper_trade()` directly — no `live_trading` calls. Source=`crypto_5m_engine` never triggers real orders regardless of `LIVE_TRADING_ENABLED`.

**Hybrid scalp exits (2026-06-28):** `check_scalp_take_profits()` has three exit paths:

1. **TP limit fill** (unchanged): Bid reaches `entry + SCALP_TAKE_PROFIT` → close as `closed_take_profit`
2. **Spike exit**: Bid reaches `entry + 0.06` (above the $0.04 TP) → cancel TP limit, close as `closed_spike_market_sell` at windfall profit. Pure upside — never worse than the limit.
3. **Near-close opportunistic exit**: T-30s before window close, bid > entry but below TP → cancel TP limit, close as `closed_opportunistic_exit` at whatever profit is available. Locks in a sure profit instead of riding to binary resolution.

**Losers still hold to resolution**: If bid ≤ entry at T-30s, do nothing — same as before. No bail-out on losers. The near-close exit only triggers when there's profit to capture.

Live orders: the TP limit cancellation (`cancel_tp`) on spike/near-close exits only fires when `is_live=1`. Paper trades still get the same exit logic but without CLOB order cancellation.

**Minimum capital:** With SCALP_SHARES=1 and tokens at ~$0.50, each entry deploys $0.50. With SCALP_MAX_OPEN=3, max deployed is $1.50. Recommended starting balance: $20-50.

**Import path pitfall:** The engine runs as `python -m app.crypto_engine` in the Docker container (workdir `/app`). The import MUST be `from app import live_trading`, not `import live_trading`. The Dockerfile copies `app/` to `/app/app/`, so `live_trading.py` lives at `/app/app/live_trading.py`.

**Engine startup log:**
```
[ENGINE] BTC 5-min crypto paper-trading engine starting
[LIVE] LIVE_TRADING_ENABLED is not 'true' — staying paper-only   ← paper mode
[LIVE] ClobClient initialised — host=… funder=0x…                ← live mode
[EVAL TIMER] Started (1s decision, API every ~5s)
```

### BTC price source: Binance only (Chainlink RTDS dead as of 2026-06-28)

The Polymarket Chainlink RTDS WebSocket (`crypto_prices_chainlink` topic) stopped sending data. Latency comparison test (2026-06-28) confirmed zero Chainlink messages over 60+ seconds across multiple tests. The engine's `state.btc_chainlink` stays at 0.0 and all price lookups fall through to Binance. **Binance is the only working real-time BTC price source.** The Chainlink subscription code is retained but dormant — if Polymarket restores the feed, the engine will automatically pick it up via the existing fallback pattern.

**For resolution**: Polymarket resolves BTC 5-min markets on Chainlink, NOT Binance. The engine's `_check_market_resolution()` queries Gamma API after window close — Gamma returns the official Chainlink-based outcome. This is correct. The Binance-vs-Chainlink mismatch only affects our best-guess close in `_close_old_window_positions()`, which is always overwritten by official resolution within minutes.

**Latency test script**: `scripts/compare_ws_latency.py` — connects to both Binance and Polymarket WS simultaneously, logs nanosecond timestamps, computes lag. Run inside the crypto container (VPN-required for Polymarket WS). Result: Binance updates every ~100ms; Chainlink never arrives.

### Directional engine — flat 2 shares + market consensus filter (2026-06-28)

Two changes validated against 128 historical paper trades:

1. **Flat 2 shares**: Removed `_shares_from_edge()` variable sizing. 3-4 share trades accounted for -$5.75 in losses vs +$3.19 in wins (-$2.56 net). Flat sizing stops loss amplification.

2. **Market consensus filter** (`MIN_MARKET_PROB=0.25`): Only enter when market gives our direction <25% chance (entry > $0.75). The 31 trades where mkt ≥ 25% lost -$4.91 at 61% win rate. Dropping them flips the engine from negative to positive expectancy. Historical backtest: -$4.00 → +$3.36 across same 97 qualifying trades.

```python
if edge_up >= MIN_EDGE and up_ask <= MAX_ENTRY_PRICE and market_up_prob < MIN_MARKET_PROB:
    shares = 2  # flat sizing
```

### Directional engine model — proportional conviction (2026-06-28)

The engine's model_up probability used to be fixed 62% for ANY pct_change above the floor (0.04%). This treated a $24 wiggle and a $72 surge identically. Replaced with proportional formula:

```python
pct_pct = pct_change * 100              # convert decimal ratio (0.0004) to percentage points (0.04)
model_up_prob = 0.50 + pct_pct           # 0.04% → 54%, 0.08% → 58%, 0.12% → 62%
model_up_prob = min(0.75, model_up_prob)  # clamp at 75% for large moves
model_up_prob = max(0.25, model_up_prob)  # floor at 25% for large down moves
model_down_prob = 1.0 - model_up_prob
```

This shifts the raw_edge distribution from 0.27-0.52 to 0.19-0.45, requiring recalibrated share thresholds (0.40/0.30, see Architecture section above). Commit `36b748a`; revert to `64407d4` to restore fixed 62% model.

The diagnosis preceding this change is documented in `references/crypto-engine-debugging.md` — the workflow: check last trade time, grep `IGNORE` logs for dominant rejection filter, compute pct_change distribution, compare winning vs losing trades at similar edge levels. When every trade produces 2 shares despite variable-sizing logic, the tier thresholds don't match the actual edge distribution.

The engine now uses **Chainlink BTC/USD** as the primary price source for all BTC direction determination, window-start capture, halfway price, and best-guess close. Chainlink is the EXACT data source that Polymarket uses to resolve BTC 5-min markets. Binance is retained as a fallback only.

**Why**: On 2026-06-26, a 3-share UP bet lost -100% ($2.43) because Binance showed BTC +0.14% UP but Chainlink (the resolution source) showed a slight decline. The engine was trading on the wrong data feed. See `references/binance-vs-chainlink-mismatch.md` for the reproduced incident.

**How**: Chainlink BTC/USD prices arrive via Polymarket's own RTDS WebSocket (`wss://ws-live-data.polymarket.com`), topic `crypto_prices_chainlink`, symbol `btc/usd`. This is **free** — Polymarket sponsors the Chainlink API key. The subscription is sent in `on_poly_open()` alongside the Channel WS orderbook subscription:

```python
chainlink_sub = json.dumps({
    "action": "subscribe",
    "subscriptions": [
        {"topic": "crypto_prices_chainlink", "type": "*", "filters": '{"symbol":"btc/usd"}'}
    ]
})
ws.send(chainlink_sub)
```

The `on_polymarket_message()` handler checks for `topic == "crypto_prices_chainlink"` before checking `event_type` — RTDS messages have a `topic` field, Channel WS messages have `event_type`.

**EngineState**: Added `btc_chainlink: float = 0.0` field. All price lookups use: `state.btc_chainlink if state.btc_chainlink > 0 else binance_fallback`.

**Fallback pattern** (applied consistently across btc_mid, window_start_price, _close_old_window_positions, _close_all_expired_windows):
```python
btc_mid = state.btc_chainlink if state.btc_chainlink > 0 else (btc_bid + btc_ask) / 2
```

**Bug: `state.btc_mid` AttributeError**: The `EngineState` class has `btc_bid` and `btc_ask` but NOT `btc_mid`. Any code that accesses `state.btc_mid` will crash silently (caught by try/except). Fix: compute `btc_end = (btc_bid + btc_ask) / 2`. This bug caused ALL open trades to never close for hours.

**Bug: Scanner overwriting crypto trades**: The scanner must have a blanket `continue` skip for `source='crypto_5m_engine'` at the TOP of the trade processing loop. The old partial check at the "close expired" block was insufficient — the scanner's mark-to-market update (line 481) was updating crypto trade `current_value` before reaching the skip.

**Bug: Open trades accumulating across windows**: If `_close_old_window_positions()` crashes, old windows stay open forever. The safety net `_close_all_expired_windows()` runs at the top of every evaluation cycle and catches any stragglers.

### Local dev: engine CPU, VPN, and code reload

- **Engine CPU**: 3-second eval loop keeps usage manageable. Do NOT reduce below 2.5s without measuring.
- **VPN health**: `docker logs polymarket-intel-gluetun 2>&1 | grep -i wireguard`. Engine cannot reach Binance or Polymarket WS without VPN.
- **Code changes take effect on rebuild**: `docker compose up -d --build crypto` (or `web`). Restart-only won't pick up new code.

## Architectural notes

**Copy strategy naming**: The scanner uses three copy strategies:
- `copy_single_high_win_rate_wallet` — copies a single high-win-rate wallet's trade (current active strategy)
- `copy_wallet_consensus` — copies only when multiple wallets independently take the same side (rarely triggers)
- `copy_high_win_rate_wallet` — LEGACY, no longer produced by current code. Old trades with this kind are historical.

**Tabs are data-rich but agents don't read them back**: The scanner populates `markets`, `info_items`, and `research_notes` tables on every run. The research agent populates `research_notes`. But neither agent queries these tables when deciding what to trade — they query Polymarket APIs directly. The data in Markets/News/Research tabs is **write-only intelligence**. This is a known gap: if the research agent or scanner should leverage stored market/news/research data for better trade decisions, wire them to query these tables before making external API calls.

## Research candidate generation (multi-topic, bet-first)

Keith wants non-sports markets (politics, crypto, macro, tech/AI, weather) to surface alongside sports. The research agent must NOT research broad topics blindly — only specific candidate markets that pass liquidity/spread/resolution-window filters.

### Design: bet-first candidate generator

1. Query Gamma `/events` across multiple tag/topic buckets: sports, politics, crypto, macro, tech/AI, weather
2. Filter each event's markets to ≤3-day resolution, minimum liquidity, tight spread
3. **Cap sports** entries (e.g., max 3 of top-N results) so non-sports can surface
4. Output a small JSON candidate list with event_slug, market, outcome, token_id, condition_id, end_date, best_bid, best_ask, spread, topic tag
5. Research agent only researches candidates in this list — never broad topics

### Topic tag classification

Classify markets by keyword in title/question:
- `crypto`: bitcoin, btc, ethereum, eth, crypto
- `politics/geopolitics`: trump, iran, ukraine, election, president, senate, congress, minister, supreme court
- `macro/regulatory`: fed, rate, cpi, sec, cftc, tariff, inflation
- `tech/ai/business`: openai, anthropic, nvidia, tesla, ipo (NOT standalone "ai" — too many false matches on player names like "Jai")
- `weather/events`: hurricane, earthquake, weather, temperature
- `general`: anything that doesn't match above but has news-driven characteristics (headline-driven, not sports)

### Cron job

Job ID: `87c151428514` — runs every 120 minutes, generates candidates then researches them. Prompt: `polymarket_research_candidates.py` → filter → research only valid candidates.

### Pitfall: standalone "ai" keyword

The word "ai" appears in many sports player names (e.g., "Jai", "Kai"). Do NOT use "ai" alone as a tech classifier keyword — it floods the tech bucket with sports. Use "openai", "anthropic" (proper nouns) instead.

## Copy trading — probation system + tiered thresholds + own P/L tracking (2026-06-28)

The scanner copies high-performing Polymarket wallets into paper trades. Three-tier wallet classification:

| Tier | Criteria | Filters | How to change |
|------|----------|---------|---------------|
| **Probation** | <5 settled copy trades in our DB OR net P/L ≤ $0 | Strict: min_price=0.67, min_size=25 | Settle 5+ trades with positive P/L |
| **Graduated (90+ score)** | ≥5 settled, positive P/L, Polymarket score ≥90 | Loose: min_price=0.30, min_size=15 | `COPY_HIGH_SCORE_MIN_PRICE/SIZE` env vars |
| **Graduated (70-89 score)** | ≥5 settled, positive P/L, score 70-89 | Strict: min_price=0.67, min_size=25 | `COPY_MIN_PRICE/SIZE` env vars |
| **Blocked** | Env-var blacklist OR negative P/L > $5/day | Skipped entirely | `COPY_BLOCKED_WALLETS` env var |

**Configurable env vars**: `PROBATION_MIN_SETTLED_TRADES` (default 5), `COPY_BLOCKED_WALLETS` (comma-separated), `COPY_MAX_DAILY_LOSS_PER_WALLET` (default -5.00), `COPY_HIGH_SCORE_MIN_PRICE` (default 0.30), `COPY_HIGH_SCORE_MIN_SIZE` (default 15), `COPY_MIN_PRICE` (0.67), `COPY_MIN_SIZE` (25).

**Graduation is automatic**: every scanner run (15 min), `get_probation_wallets()` checks all scored wallets (≥70) against our paper_trades. A wallet graduates when it has ≥5 settled copy trades AND net P/L > $0. No manual approval needed. For live trading, switch the graduation trigger from paper trades to real-money trades.

**Blocking is also automatic**: `get_blocked_wallets()` checks two sources — (1) `COPY_BLOCKED_WALLETS` env var (hard blacklist), (2) our own paper_trades for wallets with net P/L < -$5 in the last 24 hours across 3+ trades. A blocked wallet is skipped entirely — no API calls, no activity logging.

**Two critical probation edge cases fixed 2026-06-28**:

1. **Zero-settled-trades wallets**: The original `get_probation_wallets` GROUP BY query only returned wallets with ≥1 closed trade in paper_trades. Wallets with zero settled trades (new, unseen) never appeared in the GROUP BY result and slipped through probation entirely — they got full graduated-tier filters despite having no track record. Fix: added Case 2 query that checks `wallet_scores` for scored (≥70) wallets with zero closed copy trades.

2. **Slow-bleed negative P/L**: Original HAVING only checked `n < ?` (settled count). A wallet with 10 settled trades and -$3 overall would graduate because 10 ≥ 5. Fix: added `OR total_pnl <= 0` to the HAVING clause. A wallet stays in probation until BOTH conditions hold: settled ≥ N AND total_pnl > 0.

**Scoring**: `wallet_stats()` queries `/closed-positions?user=X&limit=100` — this API shows BOTH wins (realizedPnl > 0) and losses (realizedPnl < 0). Score = win_rate × 100, capped at 50 for <20 closed. Realized PnL ($) is NOT in the score. Runs every **15 min** via cron.

**Cron pitfall — `"once"` vs `"every"`**: `"once in 15m"` is a ONE-SHOT delay. After it completes, `state: completed, enabled: false`. Always use `"every"` prefix for recurring: `"every 15m"`, `"every 4h"`, `"0 9 * * *"`.

**Staleness guard**: `COPY_MAX_AGE_MINUTES = 15`. Skip signals older than 15 minutes.

**Consensus shares the same tiered filter + probation**: a candidate must pass `smart_copy_candidate()` with its wallet's probation/graduated tier first, then gets checked for multi-wallet agreement.

**History of threshold values**: 
- 90+ tier: 0.10/5 (2026-06-27, too loose — caught tennis underdogs but let in swisstony's losing favorites) → 0.30/15 (2026-06-28, tightened after copy audit proved net negative P/L)
- 70-89 tier: unchanged at 0.67/25 since inception

Full details: `references/copy-trading-architecture.md`.

### Copy wallet performance audit (2026-06-28)

Polymarket's wallet "win rate" scores are **misleading at copy level**. A wallet can have 100% score on 50+ closed positions but still be a net loser when copied at fixed 2-share sizing. The wallet likely uses dynamic position sizing (Kelly criterion) — large on conviction, small on speculative. Our blind 2-share copy destroys that sizing edge.

**Audit methodology**: Query our own `paper_trades` for each wallet's closed trades. Compute net P/L using the same capital-weighted formula as the dashboard. Compare against Polymarket's reported score. A wallet is **copy-profitable** only if our OWN net P/L > 0 across ≥10 closed trades.

**2026-06-28 findings** (yesterday's closed trades):

| Wallet | Polymarket Score | Our Trades | Wins | Losses | Our Net P/L | Verdict |
|--------|:---:|:---:|:---:|:---:|:---:|:---:|
| BreakTheBank (0xf031...) | 88% | 3 | 3 | 0 | +$2.64 | ✅ Copy |
| swisstony (0x204f...) | 100% | 44 | 17 | 27 | -$11.13 | ❌ Block |
| Substantial-Svc (0x2c33...) | 100% | 16 | 5 | 11 | -$6.78 | ❌ Block |

**Key insight**: swisstony bets on heavy favorites (avg entry 0.56). When they win, returns average +40.8%. When they lose, it's -100%. At 42% win rate with those numbers, net is deeply negative. The wallet IS profitable on Polymarket because they size trades intelligently. Our 2-share blind copy destroys the math.

**Blocking approach**: Add wallet addresses to an env-var blacklist (`COPY_BLOCKED_WALLETS`) or — better — track our own per-wallet P/L across the last N trades and auto-block wallets with net negative returns. Polymarket scores are supplemental, not primary.

**Trade-off**: Blacklisting swisstony+Substantial-Service reduces yesterday's copy volume from 63 trades (-$15.28) to 3 trades (+$2.64) — a 95% volume reduction but a swing from -$15.28 to +$2.64 net P/L. The question isn't trade count — it's P/L.

**Unknown wallet**: 0x076daa87 has 6 open trades, 0 closed. No verdict yet. Let it settle before judging.

**Next step**: Wallet discovery — scan the Polymarket leaderboard for wallets like BreakTheBank (high score AND positive copy results in our OWN data). The bottleneck isn't filters — it's finding enough genuinely copy-profitable wallets.

See `references/copy-wallet-audit.md` for the full raw data and step-by-step audit methodology.

### Wallet strategy verification — never assume from summaries (2026-06-28)

Keith's explicit directive: **"Don't assume. Reverse engineer his wallet so we know."** When discussing another wallet's strategy, do NOT extrapolate from compacted conversation summaries or incomplete earlier research. Verify from actual data sources:

1. **predicts.guru** — Free wallet checker at `https://www.predicts.guru/checker/<address>`. Shows live stats: net PnL, volume, fees, buy/sell ratio, avg hold time, win/loss fill counts, position counts, market diversity. This is the fastest first stop.
2. **Polygonscan** — On-chain transaction history for the wallet address on Polygon. Polymarket CTF contract interactions reveal every trade. Requires the wallet address (EVM format).
3. **Telonex** — Full on-chain dataset (46,945 wallets, 15.3M fills). API requires key.

**What we got wrong about nj23adsknml3 by extrapolating from summaries**: assumed he was a same-window BTC 5-min scalper with 46 positions and 1.6% edge. predicts.guru showed he's a market maker with 18-min avg hold across 1,574 markets, only 7 closed positions, and 0.28% net margin. The earlier data was stale/misleading. Always verify against live sources.

See `references/polymarket-wallet-analysis.md` for the full wallet analysis methodology and `references/midpoint-scalp-strategy.md` for the corrected nj23 data.

## References

- `references/midpoint-scalp-strategy.md` — Midpoint scalping strategy: how it works, nj23adsknml3 actual data (corrected 2026-06-28 via predicts.guru), entry/exit logic, gas/fee analysis for real trading, and why we cannot replicate nj23's market-making approach. **2026-06-28 update**: SCALP_SHARES → 1, eval cycle → 1s, live trading module wired in.
- `references/live-trading-architecture.md` — Live trading module (`app/live_trading.py`): ClobClient setup, order placement flow, TP fill detection, credential management, WebSocket vs polling tradeoff, minimum capital requirements, and deployment checklist.
- `references/live-trading-diagnostics.md` — Live trading JSONL logging: order events, health heartbeat, error counters, fill detection, log file paths, and post-deployment verification.
- `references/polymarket-wallet-analysis.md` — Wallet analysis tools and methodology: predicts.guru checker, Polygonscan on-chain verification, Telonex dataset. How to verify wallet strategies from actual blockchain data rather than extrapolating from summaries.
- `references/polymarket-api-landscape.md` — Polymarket API surfaces (Gamma, CLOB, WebSocket), geoblocking, API access constraints (updated 2026-06-28 with working/failing endpoint matrix), automated trading capability, and client libraries. Load when the user asks about Polymarket APIs, trading automation, or platform mechanics.
- `references/dashboard-redesign-2026-06-26.md` — Session-specific redesign details, P/L bug reproduction, and status formatter gotcha.
- `references/crypto-engine-resolution.md` — Best-guess P/L pattern for slow Polymarket settlement on BTC 5-min markets.
- `references/strategy-retirement-filter-mismatch.md` — The 2026-06-26 hedge-removal bug: backend SQL includes retired strategy trades while frontend filters them out.
- `references/binance-vs-chainlink-mismatch.md` — Binance vs Chainlink mismatch. Chainlink RTDS WS is DEAD as of 2026-06-28.
- `references/vpn-proxy-setup.md` — Gluetun HTTP proxy configuration.
- `references/py-clob-v2-monkeypatch.md` — Full monkeypatch code for the py-clob-client-v2 L1 auth bug (POLY_ADDRESS always uses EOA instead of deposit wallet). Apply in live_trading.py before client init. Necessary but not sufficient — proxy must also be deployed on-chain.
- `references/deposit-wallet-proxy.md` — Deposit wallet proxy deployment prerequisite: why the 400 signer error happens, the one-UI-trade fix, verification, and signature type cheat sheet. **2026-06-28 user-explicit correction** — this blocked live trading for the entire session.\n- `references/github-repo-setup.md` — GitHub repo setup
- `references/crypto-engine-debugging.md` — Diagnostic workflow when crypto engine stops producing trades.
- `references/copy-trading-architecture.md` — Copy trading probation/graduation system.
- `references/copy-wallet-audit.md` — Copy wallet performance audit (2026-06-28).
- `references/copy-wallet-probation-system.md` — Full probation system design.
- `references/scanner-debugging.md` — sync_final_results diagnostic workflow.
- `references/midpoint-scalp-strategy.md` — Midpoint scalping strategy: how it works, nj23adsknml3 reverse-engineering, entry/exit logic, gas/fee analysis for real trading, and why $100 capital isn't enough.