---
name: polymarket-intel
description: Work on the Polymarket Intel paper-trading dashboard, crypto engine, scanner, and research agent. Covers project structure, dashboard design, deployment, common pitfalls, and P/L display conventions.
---

# Polymarket Intel

Paper-trading research platform at `/root/polymarket-intel`. Flask dashboard + crypto 5-min engine + market scanner + smart-wallet watcher + news research agent. All paper only — no real money, no private keys.

## GitHub repository

- **Remote**: `git@github.com-polymarket:keithit110/Polymarket-hermes-project.git`
- **Deploy key**: `/root/.ssh/polymarket_ed25519` (write access enabled on GitHub)
- **`.gitignore`**: Excludes `.env*`, `data/*.sqlite`, `logs/`, `__pycache__/`, `*.pyc`, private keys
- **Full setup & secrets audit**: see `references/github-repo-setup.md`
- **Before every push**: Run secrets audit — staged files must contain zero API keys, private keys, or passwords. `.env.vpn` must NOT be tracked.

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
| `gluetun` | internal + proxy `:8888` | VPN + HTTP proxy | Surfshark WireGuard UK + built-in HTTP proxy for other containers |
| `scanner` | one-shot | default bridge | Market scan, arbitrage detection, wallet scoring (outbound via gluetun HTTP proxy) |

## VPN routing — ALL containers exit UK (2026-06-26)

When Keith switches to real trading, Polymarket must see a non-US IP. **Every container that makes Polymarket API calls** must route through the UK VPN, not just the crypto engine.

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

Both must show UK (London, GB), not US.

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

### Table columns (order matters)

The Paper Bets table column order is: Market, Side, P/L, Status, Strategy, Shares, Entry, Current, Reason, Opened, Closed, Resolves.

**Shares column**: Always present. Bold (`<b>`) when >1 so high-conviction positions stand out visually. Populated from `paper_trades.shares` (defaults to 1). The engine stores a single row per position — never duplicate rows per share.

Open/unrealized trades must show "—" in the table P/L column, not a percentage. The strategy card for a strategy with only open trades must show "N open" count, not a fake -100% or +X% mark-to-market. Only closed/realized trades get percentages. This applies to ALL strategy types.

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

## Common pitfalls

1. **P/L formula: capital-weighted return, not average**: The overall metric, chart, and strategy cards all use `(total_returned − total_deployed) / total_deployed`. Do NOT revert to simple average of per-trade percentages — that overweighted small losses and was removed 2026-06-26.
2. **Status pill underscores**: After `replace(/_/g,' ')`, the `startsWith` strings must use spaces, not underscores.
3. **Chart and metric card MUST use the same formula AND data subset**: Both use capital-weighted cumulative return: `(cum_returned − cum_deployed) / cum_deployed`. Both exclude open trades. Both multiply entry_cost and current_value by `shares`. A mismatch on any of these three axes (formula, subset, shares) produces the "numbers don't add up" complaint. The chart builds cumulative deployed/returned by day; the metric does it across all closed trades at once — same formula, same data.
4. **Chart.js canvas height**: Use `height: 240px` on container div with `canvas { max-height: 240px }`. Chart.js overrides the canvas `height` attribute when `responsive: true`.
5. **Chart P/L double-multiplication**: Backend returns capital-weighted return as fraction (e.g., -0.0103 for -1.03%). Chart multiplies by 100 once: `data: s.pnl_series.map(x => x.v * 100)`. If backend also multiplies, value explodes 100x. Verify ONE side does the *100.
6. **Never show P/L for open trades**: Table must show "—", cards must show "N open" count. Only closed trades get percentages. This applies to ALL strategy types.
7. **Hedge trades are historical**: `crypto_5m_profit_lock_hedge` is no longer produced. Filter them OUT of UI display. Do not group BTC trades by slug — each directional trade is standalone.
8. **Engine `btc_mid` computation — Chainlink first, Binance fallback**: `EngineState` has `btc_chainlink`, `btc_bid`, and `btc_ask` but NOT a combined `btc_mid` field. All price lookups must compute: `state.btc_chainlink if state.btc_chainlink > 0 else (btc_bid + btc_ask) / 2`. This pattern is applied consistently in `evaluate_and_act()` (btc_mid), `refresh_token_ids()` (window_start_price), `_close_old_window_positions()`, and `_close_all_expired_windows()`. The old bug where code accessed the nonexistent `state.btc_mid` attribute (silent crash) was fixed 2026-06-26 by replacing ALL such accesses with the Chainlink-first fallback pattern.
9. **Scanner must skip crypto engine trades**: Scanner's main loop needs `if source == 'crypto_5m_engine': continue` at the TOP. Otherwise scanner updates crypto trade `current_value` and produces skewed mark-to-market numbers.
10. **Polymarket BTC 5-min settlement delay**: Gamma API returns `closed: false` for 10-15+ minutes after window ends. Compute best-guess P/L from Binance BTC direction at window close rather than waiting for Polymarket.
11. **BTC 5-min resolution = Chainlink, NOT Binance**: Polymarket resolves these markets on Chainlink BTC/USD data stream. The engine now uses Chainlink as the PRIMARY price source (via Polymarket RTDS, free/sponsored). Binance is a fallback only. Before 2026-06-26 the engine used Binance exclusively — this caused a confirmed -100% loss because Binance showed +0.14% UP while Chainlink resolved Down. The switch eliminates the data-source mismatch entirely for live trading.
12. **Misleading "below min 0.05" log message**: When the engine logs `edge_up=-80.8% edge_down=80.9% below min 0.05`, the edge_down of 80.9% TECHNICALLY passes the `>= MIN_EDGE` check. The real rejection is `MAX_ENTRY_PRICE` — `down_ask` has already surged above 0.85 because the market repriced before the engine could enter. The message comes from the else-branch at line 288 which fires when NEITHER direction satisfies BOTH conditions (`edge >= MIN_EDGE AND ask <= MAX_ENTRY_PRICE`). BTC moving clearly → market reprices instantly → engine detects the edge but can't enter at a price above 0.85. This is correct behavior — the engine is protecting against late entries, not missing opportunities.
13. **No crypto entries during low volatility is normal**: Three filters correctly block entries when BTC is flat: (a) `pct_change` too close to 0 — no directional signal, (b) outside 60-180s window — too early or too late, (c) ask > 0.85 — market already repriced the move. An hour with zero entries during range-bound BTC is the engine working as designed, not a bug. Check `docker logs polymarket-intel-crypto 2>&1 | grep IGNORE` before investigating.

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

- **Directional-only**: Lane 3 (hedging) removed. Engine only buys UP or DOWN based on edge.
- **Variable shares**: 3 shares at edge ≥55%, 2 shares at edge ≥48%, 1 share otherwise. Tiers calibrated to the engine's actual edge range (43-70%). The old thresholds (≥20% → 3, ≥10% → 2) were too low — every trade triggered 3 shares because the model never enters below ~43% edge. Changed 2026-06-26 after user identified that `t=1.0x` always mapped to 3 shares.
- **Shares column**: `paper_trades.shares INTEGER DEFAULT 1`. Bold in table when >1. Backfilled from legacy duplicate-row trades by keeping the MIN(id) row, deleting the rest, and setting `shares = COUNT(*)`.
- **3-second evaluation**: `evaluate_and_act()` runs every ~3s (fast, math-only). Slow API calls (`refresh_token_ids()`, `resolve_pending_trades()`) only every 3rd cycle (~9s).
- **BTC acceleration tracking**: Records `btc_halfway_price` at ~150s mark. Compares first-half vs second-half movement. Same direction in both → +5% confidence boost. Reversal → -8% penalty.
- **Time-decay boost**: Later in window → more confidence in the move. `time_factor = 1.0` at 180s → `2.0` at 60s.
- **Entry max price**: Skip any window where entry > $0.85 (market says ≤15% chance).
- **65-day data retention**: `purge_old_data()` deletes rows older than 65 days from `paper_trades` (by `opened_at`), `runs` (by `ts`), `arbitrage_opportunities` (by `seen_at`), and `smart_wallet_signals` (by `seen_at`). Runs on engine startup and every ~3 hours (~3600 evaluation cycles). Keeps historical analysis window meaningful and queries fast.

### Chainlink BTC/USD as primary price source (2026-06-26)

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

## Copy trading — tiered thresholds + wallet scoring (2026-06-26)

The scanner copies high-performing Polymarket wallets into paper trades. **Tiered filtering**: wallets scoring 90+ (≥90% win rate, ≥20 closed) get looser thresholds (min price 0.30, min size 15) because their track record justifies trust at lower prices/sizes. Lower-score wallets keep strict FIFA-tuned filters (0.67/25).

**Scoring**: `wallet_stats()` queries `/closed-positions?user=X&limit=100` — this API shows BOTH wins (realizedPnl > 0) and losses (realizedPnl < 0). Score = win_rate × 100, capped at 50 for <20 closed. Realized PnL ($) is NOT in the score. Runs every **15 min** via cron `1979b309d4db` (changed from 30 min 2026-06-26 for faster detection of new hot wallets).

**Scoring accuracy — losses ARE detected**: The `/closed-positions` endpoint is Polymarket's settlement ledger. Every closed position has a `realizedPnl` field — positive for wins, negative for losses. This is DIFFERENT from the activity feed (which can't show losses because losing positions expire with no on-chain claim). Losses correctly lower a wallet's win rate and score.

**Staleness guard**: `COPY_MAX_AGE_MINUTES = 15`. Scanner compares `activity.timestamp` vs `now()`. Skip signals older than 15 minutes — prevents copying a 2-hour-old trade after scanner downtime. Current signals all under 7 minutes; the guard is forward protection.

**Consensus shares the same tiered filter**: a candidate must pass `smart_copy_candidate()` first, then gets checked for multi-wallet agreement. The 90+ tier relaxation benefits both single and consensus strategies.

Full details: `references/copy-trading-architecture.md`.

## References

- `references/polymarket-api-landscape.md` — Polymarket API surfaces (Gamma, CLOB, WebSocket), automated trading capability, geoblocking, and client libraries. Load when the user asks about Polymarket APIs, trading automation, or platform mechanics.
- `references/dashboard-redesign-2026-06-26.md` — Session-specific redesign details, P/L bug reproduction, and status formatter gotcha.
- `references/crypto-engine-resolution.md` — Best-guess P/L pattern for slow Polymarket settlement on BTC 5-min markets.
- `references/strategy-retirement-filter-mismatch.md` — The 2026-06-26 hedge-removal bug: backend SQL includes retired strategy trades while frontend filters them out, producing a -29.53% overall metric when all strategy cards are positive. Full fix checklist and verification steps.
- `references/binance-vs-chainlink-mismatch.md` — Binance vs Chainlink BTC price data source mismatch: engine uses Binance, Polymarket resolves on Chainlink. Reproduced incident from 2026-06-26 where 3-share UP bet lost -100% because Binance showed +0.14% but Chainlink resolved Down. Critical for going live — trading on wrong data source is gambling.
- `references/vpn-proxy-setup.md` — Gluetun HTTP proxy configuration so ALL containers (scanner, web, crypto) route through UK VPN. Required for Polymarket International (non-US) real trading. Step-by-step setup, verification commands, and caveats.
- `references/github-repo-setup.md` — GitHub repo setup: deploy key creation, SSH config, .gitignore (excluding .env*, data/*.sqlite, logs/), secrets audit checklist, and push workflow. Load when initializing the repo or before pushing to verify no secrets leaked.
