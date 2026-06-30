---
name: polymarket-live-scalp-engine
description: Operate, debug, and tune Keith's Polymarket BTC 5-min live trading engine (momentum follower + midpoint scalp).
triggers:
  - "scalp engine not selling"
  - "market sell failing"
  - "position stuck open"
  - "re-entry not working"
  - "dashboard shows wrong P/L"
  - "adjust scalp parameters"
  - "Polymarket order not filling"
  - "live trading not placing orders"
  - FOK / FAK / IOC order type issues
  - "momentum follower"
  - "momentum not firing"
  - "poly market trading"
  - "polymarket api"
---

# Polymarket Live Midpoint Scalp Engine

Keith's live automated trading bot for Polymarket BTC 5-min Up/Down markets.
Runs in Docker at `/root/polymarket-intel` (crypto container, VPN-routed via gluetun).

## Architecture

```
docker compose services:
  vpn     → gluetun Surfshark Canada (required — Polymarket geo-blocks)
  crypto  → live midpoint scalp engine (WebSocket + CLOB v2)
  web     → dashboard on port 8095
  scanner → copy-trade wallet scanner (cron every 15min)

Data: SQLite at data/polymarket_intel.sqlite (WAL mode)
Logs: logs/live_health.jsonl, logs/live_orders.jsonl, logs/scalp_decisions.jsonl
```

## Strategy Parameters

Two strategies run in the same engine, independently gated:

**Midpoint scalp** — CURRENTLY HALTED (`SCALP_SHARES=0`, `SCALP_SHARES_NEUTRAL=0`). Halted 2026-06-29 by user due to risk/reward concerns. Code remains, can be re-enabled by setting shares > 0.

**Momentum follower** — ACTIVE LIVE STRATEGY. Only strategy currently placing real orders on Polymarket. See dedicated section below for parameters and pitfalls.

**BTC Directional (Lane 2)** — ACTIVE PAPER STRATEGY. Paper-only directional bets with -40% mark-to-market stop-loss. See dedicated section below.

### Midpoint Scalp Parameters (halted)

All configurable via env vars in `docker-compose.yml`:

| Env Var | Default | Meaning |
|---|---|---|
| `SCALP_MIN_ENTRY` | 0.45 | Lowest ask price to buy |
| `SCALP_MAX_ENTRY` | 0.55 | Highest ask price to buy |
| `SCALP_TAKE_PROFIT` | 0.05 | +$0.05 from entry = sell |
| `SCALP_SPIKE_THRESHOLD` | 0.07 | +$0.07 from entry = immediate sell |
| `SCALP_DEADLINE_SECONDS` | 120 | T-N seconds before close: freeze new entries |
| `SCALP_HOLD_SECONDS` | 120 | T-N seconds before close: stop ALL exits, hold to resolution |
| `SCALP_SHARES` | 5 | Shares per position (min 5 = $1 Polymarket minimum) |
| `SCALP_MAX_OPEN` | 3 | Max open scalp positions per window |

To change: edit `docker-compose.yml` → `docker compose up -d --build crypto`.

## Market Sell Order Types — CRITICAL

**Use GTC limit sell at current bid, never FAK or FOK.**

Available in `py-clob-client-v2.OrderType`: `GTC | GTD | FAK | FOK`

- **FOK** (Fill-or-Kill): fails entirely if orderbook can't fill ALL shares at once. Causes "order couldn't be fully filled" errors.
- **FAK** (Fill-and-Kill): SELL: causes "not enough balance / allowance: balance: 4994443, order amount: 5000000" errors due to precision/balance quirks in MarketOrderArgs. BUY: works reliably — momentum follower uses `place_buy_fak()` with FAK for instant fills. Different behavior for buys vs sells due to balance/precision handling.
- **GTC limit sell at bid** (RECOMMENDED): place a regular limit sell order at the current bid price. Fills against existing orders without precision issues. Reliable, no balance errors.
- `OrderType.IOC` DOES NOT EXIST in py-clob-client-v2. It will error.

Implementation in `app/live_trading.py`:
```python
from py_clob_client_v2 import OrderArgs, OrderType, PartialCreateOrderOptions, Side
order_args = OrderArgs(token_id=token_id, price=current_bid, size=size, side=Side.SELL)
options = PartialCreateOrderOptions(tick_size="0.01")
result = _client.create_and_post_order(
    order_args=order_args, options=options, order_type=OrderType.GTC)
```

See `references/sell-order-evolution.md` for the full history of why FOK and FAK failed.

## Sell Retry Cap + Abandoned Sells

After 3 consecutive sell failures (`MAX_SELL_RETRIES = 3`), the engine ABANDONS the sell and marks the position `closed_sell_failed`. This prevents infinite API spam when:
- Position already resolved on-chain (shares converted to USDC, no shares to sell)
- Balance/precision issues between buy fill amount and sell amount

Key functions:
- `place_market_sell(token_id, size, trade_id, limit_price)` — GTC limit sell at bid, tracks failures in `_sell_fail_count[trade_id]`
- `is_sell_abandoned(trade_id)` — returns True if `_sell_fail_count[trade_id] >= MAX_SELL_RETRIES`
- On abandon: `conn.execute("update paper_trades set status='closed_sell_failed' where id=?")`

All three exit paths (TP, spike, near-close) handle abandoned sells by marking `closed_sell_failed` with the current mark price.

## Fill Verification — Polymarket Is Source of Truth

Never assume a sell succeeded. Never trust the DB alone. Flow:

```
1. Place GTC limit sell at current bid → store order_id in _pending_sell_orders[trade_id]
2. Poll: check_sell_fill(trade_id) → calls _client.get_order(order_id)
3. If status == "MATCHED" → sell confirmed → close DB row
4. If still pending → keep polling next tick
5. If sell fails repeatedly → after 3 attempts, ABANDON → mark closed_sell_failed
6. NEVER close the DB row until Polymarket confirms fill
```

Key functions in `live_trading.py`:
- `place_market_sell(token_id, size, trade_id, limit_price)` → GTC limit sell at bid, tracks failures
- `is_sell_pending(trade_id)` → checks if sell already placed (don't duplicate)
- `is_sell_abandoned(trade_id)` → checks if retry cap hit (stop retrying)
- `check_sell_fill(trade_id)` → polls Polymarket for MATCHED status
- `check_buy_fill(trade_id)` → polls buy fill before allowing sell

**Critical: wallet/on-chain is always source of truth.** When the engine and Polymarket disagree, Polymarket wins. Verify positions with `get_open_orders()` and `get_balance_allowance()` directly, not just the DB.

## Engine Exit Logic

**Hold zone gate (applied BEFORE all three exits below):**
When `seconds_left <= SCALP_HOLD_SECONDS`, the engine skips ALL sell logic and holds the position to resolution. No TP sells, no spike sells, no near-close sells. The position rides to the 1.0 or 0.0 outcome.

The hold zone check is in `check_scalp_take_profits()`:

```python
# ── Hold zone: near resolution, let position ride to outcome ──
if seconds_left <= SCALP_HOLD_SECONDS:
    continue  # skip all three exits below
```

Messages: `[SCALP] HOLD #NNN UP — NNs left, holding to resolution`

**CRITICAL PITFALL: hold zone must be wide enough** — see Pitfalls section.

**CRITICAL PITFALL: datetime timezone mismatch** — see Pitfalls section for the now_utc() aware-vs-naive bug.

Three exit paths in `check_scalp_take_profits()`, all using GTC limit sell at current bid (gated by hold zone above):

1. **TP hit**: `bid >= entry + SCALP_TAKE_PROFIT` → GTC limit sell at bid → verify fill → close
2. **Spike**: `bid >= entry + SCALP_SPIKE_THRESHOLD` → GTC limit sell at bid → verify fill → close
3. **Near-close**: `seconds_left <= SCALP_DEADLINE_SECONDS AND bid > entry` → GTC limit sell at bid → verify fill → close

If any sell fails to place: retry next tick (up to 3 attempts). On 3rd failure: mark `closed_sell_failed`.
If sell placed but not yet filled: keep polling, don't mark closed.

Losers (bid <= entry at window close): held to resolution. No stop-loss.

## Re-entry After Take Profit

- `has_scalp_position(conn, event_slug, side)` checks for OPEN positions only
- Once a position is `closed_take_profit`, the slot is free
- Next evaluation cycle: if price back in `[SCALP_MIN_ENTRY, SCALP_MAX_ENTRY]` and `window_open < SCALP_MAX_OPEN` → re-enter
- Window count is per-event_slug — old-window positions don't block new entries

## Dashboard — Live Section Rules

Live trading section shows wallet-grounded numbers ONLY:

| Metric | Source |
|---|---|
| Wallet Balance | Polymarket API (health log) |
| Actual P/L | balance - starting_balance (persisted in DB) |
| Live Deployed | capital in open live positions |
| Live Open | count of is_live=1 AND status='open' |
| DB Est P/L | from trade records — labeled "(not actual)" |

Starting balance persisted in `crypto_engine_state` table with key `wallet_start_balance`.

NOTE: The web UI originally hard-excluded `midpoint_scalp` from all views (API, JS filter, summary counts, P/L) since it was the only live strategy. After deleting paper data (`is_live=0` rows), all remaining midpoint_scalp rows are live — remove all exclusions so they appear in the bets table. See Pitfalls section for the 4 exclusion points.

## Debugging Commands

### Full Reconciliation: Wallet vs DB

When the user suspects the DB is out of sync with their wallet, run this end-to-end check:

```bash
# 1. Get wallet balance from engine health (on-chain source of truth)
docker logs polymarket-intel-crypto --tail 30 | grep "LIVE HEALTH"
# Extract: balance, open_tps, buys/sells/errors

# 2. Check DB for open live positions
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT id, status, entry_cost, current_value, shares, 
          substr(event_slug, -10) as window
   FROM paper_trades 
   WHERE kind='midpoint_scalp' AND is_live=1 AND status LIKE 'open%'
   ORDER BY id;"

# 3. Check for stuck/failed sells
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT id, status, event_slug FROM paper_trades 
   WHERE status IN ('pending_sell','sell_failed','closed_sell_failed')
   ORDER BY id DESC LIMIT 10;"

# 4. Verify all positions in the target window are closed
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT id, status, entry_cost, current_value FROM paper_trades 
   WHERE event_slug='<slug>' ORDER BY id;"

# 5. Sum live P/L for the session
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT COUNT(*) trades, 
          SUM((current_value - entry_cost) * shares) net_pnl
   FROM paper_trades 
   WHERE kind='midpoint_scalp' AND is_live=1 
     AND id >= <first_id> AND status NOT LIKE 'open%';"
```

**Reconciliation rules:**
- `open_tps=0` in health check means NO open positions on-chain — DB should also show zero open
- If health says `open_tps=0` but DB shows `open`, check timing: position may have been filled/sold after the health snapshot
- Wallet balance wins every dispute — if DB says open but wallet says closed, DB is wrong
- Check `MATCHED` status in engine logs to confirm sells actually executed on-chain

```bash
# Check engine logs
docker logs polymarket-intel-crypto --tail 50

# Check live health (wallet balance, error counts)
tail -5 /root/polymarket-intel/logs/live_health.jsonl

# Check live order events
tail -20 /root/polymarket-intel/logs/live_orders.jsonl

# Check scalp decisions
tail -20 /root/polymarket-intel/logs/scalp_decisions.jsonl

# Check open live positions
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT id, event_slug, json_extract(details_json,'$.side') as side, entry_cost, shares, status FROM paper_trades WHERE source='midpoint_scalp' AND is_live=1 ORDER BY id DESC LIMIT 10;"

# Check wallet balance
curl -s http://localhost:8095/api/summary | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'Wallet: {d[\"wallet_balance\"]}  Start: {d[\"wallet_start_balance\"]}  Actual P/L: {d[\"actual_pnl\"]}')"

# Check env vars in running container
docker exec polymarket-intel-crypto env | grep SCALP

# Rebuild and restart (after code/config changes)
cd /root/polymarket-intel && docker compose up -d --build crypto
```

## Trend Detection + Dynamic Sizing (2026-06-29)

P/L-based trend detector analyzes last 6 resolved scalp windows:

```python
def detect_trend(conn, event_slug):
    # Query last 6 resolved windows
    # Per window: compute UP vs DOWN dollar P/L
    # Tally which side won more money per window
    # If one side wins ≥67% of windows → trend locked
    # Mixed or <2 windows → NEUTRAL
```

**Dynamic shares based on trend:**
| Trend | UP shares | DOWN shares |
|---|---|---|
| UP | `SCALP_SHARES` (5) | 0 (blocked) |
| DOWN | 0 (blocked) | `SCALP_SHARES` (5) |
| NEUTRAL | `SCALP_SHARES_NEUTRAL` (2) | `SCALP_SHARES_NEUTRAL` (2) |

Config: `SCALP_SHARES_NEUTRAL` env var (default 2).

When trend is UP or DOWN, `evaluate_scalp()` blocks the disfavored side entirely. Drops `SCALP OPEN` log entries show `×5` (favored) or `×2` (neutral) for the shares per trade.

## Momentum Follower Strategy (2026-06-29, LIVE as of 2026-06-29)

**Status: LIVE trading — NO PAPER FALLBACK.** Live-or-nothing: if the Polymarket order fails, the DB row is deleted and the window is skipped. No paper entries are created for momentum.

Full references:
- `references/momentum-follower-live.md` — strategy logic, production config, live-only enforcement pattern, debug visibility guide, common issues, and P/L reset procedure
- `references/momentum-follower-analysis.md` — full 57-trade audit (2026-07-01): 68.4% win rate, breakeven 0.4% return, risk/reward 2.2:1 against, proposed -30% SL would make it 22% profitable

Separate from midpoint scalp — isolated `source='momentum_follower'`, `kind='momentum_follower'`, no shared position gating. Holds to resolution (no TP, no spike sell, no stop-loss). One entry max per window.

### BTC Directional Strategy (paper-only, active)

Paper-only directional bets on BTC 5-min markets. Wins capped at resolution return (+25-31% typical), losses capped at -40% via mark-to-market stop-loss.

**Config (docker-compose.yml):**
| Var | Value | Meaning |
|---|---|---|
| `CRYPTO_MIN_EDGE` | 0.05 | Minimum edge (5%) to enter |
| `CRYPTO_MAX_ENTRY_PRICE` | 0.85 | Skip if ask > this |
| `CRYPTO_MIN_MARKET_PROB` | 0.25 | Only fade when market 75%+ sure opposite |
| `CRYPTO_MIN_PCT_CHANGE` | 0.0004 | Minimum BTC movement (0.04%) |
| `DIRECTIONAL_STOP_LOSS` | **0.40** | -40% mark-to-market stop-loss |

**Stop-loss implementation — CRITICAL details (2026-07-01):**

The SL has TWO trigger paths:

1. **WS fast-path (primary):** `_check_directional_sl_fastpath()` runs inside the Polymarket WebSocket callback — fires on EVERY orderbook tick (dozens/sec). When the OPPOSITE token's ask crosses `sl_opposite = 1.0 - (entry × 0.60)`, it adds the trade ID to `_sl_triggered`. The cycle checker processes flagged trades immediately.

2. **Cycle checker (fallback):** `check_directional_stop_loss()` also checks every ~1s for `opposite_ask >= sl_opposite` or `mark_price <= sl_price`.

**Why monitor the OPPOSITE token, not the position's token:**
In binary markets, the losing token crashes $0.50→$0.01 in one tick. The winning token rises gradually: $0.50→$0.55→$0.60→...→$1.00. Monitoring the OPPOSITE token's ask catches the crossing before the crash. Example: entry DOWN at $0.80, -40% SL → sl_opposite = 1.0 - 0.48 = $0.52. When UP ask crosses $0.52, the SL fires and DOWN exits at ~$0.48.

**Window freeze after SL:** `freeze_window()` called immediately. `evaluate_and_act()` checks `is_window_frozen()` before entering — prevents re-entry into the same window after SL.

**Status persistence pitfall (fixed 2026-07-01):** `resolve_pending_trades()` queries `status like 'closed%'` for correction, but this would overwrite `closed_stop_loss` with `closed_resolved_win/loss`. Fix: query now excludes `closed_stop_loss` — `status like 'closed%' and status != 'closed_stop_loss'`.

### Entry Price Source of Truth — Polymarket Fill, Not Assumed Ask

**Logic:** During the trigger window (T+45s to T+85s, i.e. `seconds_remaining` between 215 and 255), check if BTC has moved ≥ threshold. If so, enter that direction at market (GTC limit buy at ask) and hold to resolution.

**Production config (docker-compose.yml):**
| Var | Value | Meaning |
|---|---|---|
| `MOMENTUM_ENABLED` | true | Enable |
| `MOMENTUM_LIVE` | true | Live trading on Polymarket |
| `MOMENTUM_THRESHOLD` | 0.0005 | 0.05% BTC move |
| `MOMENTUM_SHARES` | **5** | Shares per trade (**Polymarket minimum for BTC markets**) |
| `MOMENTUM_TRIGGER_START` | 215 | ~85s into the 5-min window |
| `MOMENTUM_TRIGGER_END` | 255 | ~45s into the 5-min window |
| `MOMENTUM_MIN_ENTRY` | 0.58 | Skip if ask < this — too low, no conviction |
| `MOMENTUM_MAX_ENTRY` | 0.75 | Skip if ask > this — protects against bad risk/reward (e.g. $0.94 entry = $0.06 upside) |

**CRITICAL: Momentum uses FAK (Fill-And-Kill) buys, not GTC.** Midpoint scalp at $0.50 had deep liquidity so GTC filled. Momentum entries at $0.65-$0.94 have thinner books — GTC sits unfilled, creating "ghost trades" in the DB but no positions on-chain. FAK guarantees instant fill or cancel. Implementation: `place_buy_fak()` in `app/live_trading.py` uses `OrderType.FAK`.

**CRITICAL: Polymarket CLOB minimum is 5 tokens.** Orders with fewer shares are rejected with `Size (N) lower than the minimum: 5`. 2- and 3-share orders were silently failing, falling back to paper. The engine now requires 5 shares minimum for live momentum entries. At ~$0.65-$0.85 per share, that's $3.25-$4.25 deployed per trade.

**Live-only enforcement:** The `evaluate_momentum()` function has NO paper fallback path:
```python
# Skip entirely if can't go live
if not want_live:
    return
# Insert DB row as is_live=True
trade_id = insert_paper_trade(conn, ..., is_live=True)
# Place live order
buy_ok = live_trading.place_buy_fak(token, entry_cost, MOMENTUM_SHARES, trade_id)
if not buy_ok:
    # Delete the paper row — no paper fallback
    conn.execute("delete from paper_trades where id=?", (trade_id,))
    conn.commit()
    print(f"[MOMENTUM] #{trade_id} FAILED (order rejected) — skipped")
```

This ensures the web UI never shows momentum trades that didn't actually execute on Polymarket.

## DB Locking — Cross-Container WAL Contention + get_db() Helper (2026-07-01)

**Critical pitfall #1 — scanner connects without WAL mode:** The scanner (`app/main.py`) was the primary source of `database is locked` errors (177 logged). It opened a bare `sqlite3.connect(path)` with NO WAL mode and NO busy_timeout. When it bulk-inserted wallet signals, it held an exclusive write lock that blocked the web container's reads. Symptom: web UI returns 500, logs show `sqlite3.OperationalError: database is locked` on every other request.

**Critical pitfall #2 — WAL file pile-up:** With `wal_autocheckpoint=1000` (SQLite default), the WAL file grew to 7MB of pending writes. Checkpointing that much data at once blocks all readers.

**Fix — applied to ALL containers (web, crypto, scanner, purge):**
- `sqlite3.connect(DB, timeout=30)` — 30s timeout (was 10s or 0s)
- `PRAGMA busy_timeout=30000` — 30s wait through checkpoints (was 5s or 0s)
- `PRAGMA wal_autocheckpoint=100` — checkpoint every 100 pages (was 1000 default)

**Containers that needed this:**
| Container | File | Was |
|---|---|---|
| web | `app/web.py` `conn()` | timeout=10, busy_timeout=5000 |
| crypto | `app/crypto_engine.py` `get_db()` | timeout=10, busy_timeout=5000 |
| crypto purge | `app/crypto_engine.py` `purge_old_data()` | timeout=10, busy_timeout=10000 |
| **scanner** | `app/main.py` `__init__()` | **bare `connect(path)` — worst offender** |

**Verification:** After deploy, WAL file should stay under 500KB. Web logs should show zero `database is locked` errors. Check with:
```bash
ls -lh /root/polymarket-intel/data/polymarket_intel.sqlite-wal
docker logs polymarket-intel-web 2>&1 | grep -c "database is locked"
```

**Legacy zombie-process fix (2026-06-29):** A zombie Python process from an old container can also hold locks. Find with `fuser /root/polymarket-intel/data/polymarket_intel.sqlite` → `kill <pid>`. But this is rare — cross-container WAL contention is the common cause.

**Rule:** Every container that touches the SQLite DB must use the same WAL+busy_timeout+autocheckpoint pragmas. Never use raw `sqlite3.connect()`.

## Near-Close Window Freeze (2026-06-29)

When a near-close exit fires (within `SCALP_DEADLINE_SECONDS`), the engine adds the `event_slug` to `_frozen_windows` set. `evaluate_scalp()` checks this set first and returns immediately for frozen windows — no more entries, re-entries, or TP checks. Losers still held to resolution. Output: `[SCALP] WINDOW FROZEN slug — near-close exit fired, no more entries`.

The freeze is in-memory only — survives engine runtime, resets on restart.

## Pitfalls


### Deploy-then-verify discipline (2026-07-01)

**Never deploy a crypto engine fix and call it done. Verify in three stages:**

1. **Code in container:** `docker exec polymarket-intel-crypto grep "<key function>" /app/app/crypto_engine.py`
2. **Runtime behavior:** Check engine logs after at least one full trade cycle — look for the new log output (e.g. `[DIR-SL]`, `[SCALP RESOLVE]`, `[MOMENTUM] actual fill`). If no output after a cycle, the fix didn't take effect or the code path isn't being hit.
3. **DB state:** Query the DB to confirm trades reflect the fix — correct status, correct entry_cost, correct P/L. If the DB shows old behavior, the fix is broken.

This discipline was added after Keith had to tell the agent three times that the -40% SL was still producing -98% losses. Each deploy cycle: agent changed code → rebuilt → said "fixed" → Keith checked UI → still broken. The root cause: the agent never verified step 2 (runtime) or step 3 (DB state). The code was deployed correctly but the runtime behavior never matched because the SL couldn't catch the losing token's price crash.

- **IOC doesn't exist**: py-clob-client-v2 has FAK, FOK, GTC, GTD. IOC errors silently.
- **FAK balance/precision bugs**: FAK market sell causes "not enough balance / allowance: balance: 4994443, order amount: 5000000" errors. MarketOrderArgs `amount` field has precision quirks with Polymarket's internal representation. Use GTC limit sell at bid instead.
- **FOK kills orders**: if orderbook depth < full size, FOK fails entirely. Don't use.
- **GTC limit sell at bid**: place a regular limit order at `current_bid` price. Fills against existing orders, no balance issues. This is the current recommended approach.
- **Sell retry spam**: without a retry cap, failed sells retry every ~1s indefinitely. Added `MAX_SELL_RETRIES = 3` — after 3 failures, mark `closed_sell_failed` and stop.
- **Buy fill check**: never sell before `check_buy_fill()` confirms the buy order MATCHED. Selling shares you don't own fails.
- **DB assumes success**: old code marked `closed_take_profit` before verifying fill. Now gated on `check_sell_fill()`. On abandoned sells, mark `closed_sell_failed` with current mark price.
- **Wallet is source of truth**: DB can be stale. Verify against Polymarket directly (`get_open_orders()`, `get_balance_allowance()`). If they disagree, on-chain wins.
- **Window-scoped counts**: old engine counted open scalps globally — a stuck DOWN from previous window blocked new entries. Fixed: `count_open_scalps(conn, event_slug=slug)`.
- **Geo-block**: VPN must be Polymarket-allowed country (Canada works, UK/US/France blocked). Set `SERVER_COUNTRIES=Canada` in `.env.vpn`.
- **Database locked — zombie process**: a zombie Python process from old container holds write lock. Find with `fuser`, kill, restart engine. All connections now use `get_db()` with WAL+busy_timeout.
- **Transient DB lock at window transitions**: `_close_old_window_positions()` writes during evaluation-loop reads, causing brief `database is locked` errors. These are transient (WAL allows concurrent reads) and the engine retries the next cycle. Verify no data loss by checking all positions from the affected window show proper `closed_*` status — if any remain `open`, a manual retry may be needed. Symptom: `[EVAL ERROR] database is locked` / `[SAFETY ERROR] database is locked` at window boundaries, then engine continues normally.
- **Polymarket UI coalesces rapid buy/sell pairs**: five buy→sell→buy→sell cycles in a single 5-minute window appear as one continuous position in the Polymarket web UI. Users will see a position at 0.97¢ and think the engine failed to sell, when in reality all positions already closed at TP +0.05–0.12. Verify with engine logs (`MKT SELL ... FILLED (status=MATCHED)`), not the Polymarket UI. Engine logs show every discrete trade — trust logs over visual UI.
- **`balance_usdc` is Python dict repr, not JSON**: parsing must do `bal.replace("'", '"')` before `json.loads()`.
- **Web UI hard-excludes midpoint_scalp in 4 places**: when midpoint_scalp trades don't appear in the dashboard bets table, check ALL exclusion points in `app/web.py`: (1) `/api/paper` endpoint — `WHERE kind != 'midpoint_scalp'` (line ~813), (2) frontend JS `applyPaperFilters()` — `data.filter(r => r.kind !== 'midpoint_scalp')` (line ~426), (3) summary `open_paper` count — `kind != 'midpoint_scalp'` in SQL (line ~662), (4) summary `closed_trades` query — `kind != 'midpoint_scalp'` in SQL (line ~666). Removing just one won't fix the display — all four must be removed. This exclusion was originally added to hide paper midpoint_scalp from the paper-only dashboard, but it also hides live midpoint_scalp. After deleting paper data, remove ALL exclusions since remaining midpoint_scalp rows are live.
- **Hold zone never activates — positions sold before hold zone threshold**: The hold zone (`SCALP_HOLD_SECONDS`) only protects positions that STILL EXIST when `seconds_left ≤ SCALP_HOLD_SECONDS`. But TP exits typically fire at `seconds_left=200-250` (early in the window), closing all positions before the hold zone ever activates. Symptom: zero `[SCALP] HOLD` messages in logs, all positions show `closed_take_profit`. Fix: widen `SCALP_HOLD_SECONDS` to 180 or more so it gates before TP typically hits. At 180s, the hold zone fires while positions still exist, blocking TP sells and forcing positions to ride to resolution. Verify with `docker exec polymarket-intel-crypto env | grep SCALP_HOLD` and check for `HOLD` messages in logs.

- **Hold zone datetime bug — `now_utc()` is timezone-aware, subtraction silently fails**: `now_utc()` returns `datetime.now(timezone.utc)` — a timezone-aware datetime. `end_date` parsed via `datetime.fromisoformat()` with `+00:00` also becomes timezone-aware. The ORIGINAL code tried to make one naive via `.replace(tzinfo=None)` but the resulting subtraction `naive - aware` still threw `TypeError: can't subtract offset-naive and offset-aware datetimes`. The `except: pass` silently ate it, `seconds_left` stayed at 300 default, and `300 <= SCALP_HOLD_SECONDS` was always false — hold zone NEVER fired. **Fix**: compute `seconds_left = max(0, (end_dt - now).total_seconds())` — both timezone-aware, no stripping. Verify: no `seconds_left FAIL` errors in engine output, `[SCALP] HOLD` messages appear. Test: `docker exec polymarket-intel-crypto python3 -c "from datetime import datetime,timezone; end=datetime.fromisoformat('2026-06-29T03:35:00+00:00'); now=datetime.now(timezone.utc); print((end-now).total_seconds())"` must return a number.

- **Stale `.pyc` bytecode after rebuild**: Docker COPY updates source but cached `__pycache__/*.pyc` bytecode can persist, causing OLD logic to run despite new source files. Symptom: file content verified correct via `docker exec cat`, but runtime behavior matches old code. Fix: (1) `RUN rm -rf /app/app/__pycache__` in Dockerfile after COPY, (2) `ENV PYTHONDONTWRITEBYTECODE=1` already set, (3) full cycle: `docker compose stop crypto && docker compose rm -f crypto && docker compose build crypto && docker compose up -d crypto`. Verify: `docker exec polymarket-intel-crypto ls /app/app/__pycache__/` must error "No such file." Full investigation: `references/hold-zone-datetime-bug.md`.

- **Trend detector self-reinforcing cycle — stuck on UP**: The trend detector compares UP vs DOWN dollar P/L per window. When trend is locked to UP, the engine only opens UP positions — so DOWN always shows $0 P/L and UP always "wins." The detector stays locked to UP permanently until UP positions start losing at resolution. This self-corrects naturally: after ~4 losing UP windows, DOWN wins enough windows (0 > negative P/L) to flip the trend. Takes ~20 minutes (4 windows × 5 min). Diagnosis: `SELECT json_extract(details_json,'$.side'), COUNT(*) FROM paper_trades WHERE source='midpoint_scalp' GROUP BY 1` — if DOWN count is 0 or very low, the trend is locked. NOT a bug — self-corrects when BTC reverses.

- **Momentum follower not firing — BTC didn't move enough**: The engine only prints `[MOMENTUM DBG]` when it's OUTSIDE the trigger window (s_remaining > 255 or < 215, near the 200-270 range). Inside the actual trigger window (215-255), if BTC is below threshold, there is ZERO output — it looks like the engine is dead. The engine added a `[MOMENTUM] checking…` periodic log (every 10th tick inside the window) to prove it's alive even when no signal fires. Verify: `docker logs polymarket-intel-crypto --tail 50 | grep MOMENTUM` — should show either `checking…` or actual trades. If only `DBG` lines appear with `in_window=False`, the engine is running but BTC hasn't moved enough yet. Lowering `MOMENTUM_THRESHOLD` increases frequency (with more noise).

- **Polymarket CLOB 5-token minimum — silent live-to-paper fallback**: BTC Up/Down markets on Polymarket CLOB enforce a minimum of 5 tokens per order. When `MOMENTUM_SHARES < 5`, every live buy gets rejected with `Size (N) lower than the minimum: 5`. If the code has a paper fallback path, trades silently become paper-only with no indication the live order failed. **Fix**: (1) set `MOMENTUM_SHARES ≥ 5` (currently 5), (2) remove ALL paper fallback — if live fails, delete the DB row and log `[MOMENTUM] FAILED`, (3) verify by checking engine logs for `PolyApiException` errors or `FAILED` messages. The first successful trade can go through at 2 shares if the token has a different minimum, but this is NOT reliable — all subsequent trades will fail. Always use ≥5 shares for momentum.

- **Scanner auto-closes crypto-engine trades — skip list must cover all sources**: The scanner's mark-to-market loop in `app/main.py` iterates ALL open paper trades and auto-closes them at +10% TP or -20% SL. There's a skip list for crypto-engine-managed trades: `if row["source"] in ("crypto_5m_engine", "midpoint_scalp", "momentum_follower")`. **EVERY new crypto engine source MUST be added to this list** or the scanner will auto-close live trades prematurely. The momentum follower was initially missing from this list — if the scanner had run during an active momentum position, it would have closed the position at +10% (contradicting the hold-to-resolution strategy). Fix: add the new `source` string to the tuple in `app/main.py` line ~473.

- **Disabling a strategy lane — must guard evaluation call AND debug printing**: Setting `SCALP_SHARES=0` does NOT stop the evaluation loop or debug prints from running. `evaluate_scalp()` still fires every tick, and the `[SCALP DBG]` spam prints every 10 ticks filling up logs. Fix: (1) wrap the `evaluate_scalp()` call with `if SCALP_SHARES > 0:`, (2) remove the periodic `[SCALP DBG]` debug print entirely — it serves no purpose when scalp is disabled. The eval timer, WebSocket subscriptions, and all other engine functionality continue normally.

- **Momentum follower bad entry prices — $0.94 entries risk $4.70 to win $0.30**: The momentum follower blindly bought at whatever the current ask was. When the market was heavily leaning UP (DOWN ask = $0.94), it entered DOWN at a catastrophic 16:1 risk/reward. Fix: `MOMENTUM_MAX_ENTRY` env var (default 0.75). `MOMENTUM_MIN_ENTRY` (default 0.58) also added — entries below $0.58 show no conviction and are mostly noise. The `evaluate_momentum()` function now skips any entry outside `[MOMENTUM_MIN_ENTRY, MOMENTUM_MAX_ENTRY]`.

- **Resolution bug — momentum trades never corrected by Gamma API (fixed 2026-07-01):** `resolve_pending_trades()` queried momentum/scalp trades with `status='open'` only. But `_close_old_window_positions()` already marked them `closed_resolved_win/loss` using Chainlink BTC data. Result: if Chainlink says UP but Polymarket resolves DOWN, the correction never fires. Example: momentum trade at 11:10-11:15 AM showed +$1.55 win based on BTC, but Polymarket settled as -$3.45 loss. Fix: changed query to `status='open' or status like 'closed%'` — re-resolves ALL trades for the slug against Gamma API. See `references/momentum-follower-analysis.md`.

- **DB entry_cost assumed = ask at decision time (fixed 2026-07-01):** The momentum follower recorded `entry_cost = up_ask or down_ask` — the price seen at decision time. But FAK orders may fill at different prices. Fix: `place_buy_fak()` now extracts actual fill price from Polymarket's order response via `_extract_fill_price()` (tries `avg_price`, `matched_price`, `price_matched`, `filled_avg_price`, and nested `fills` array). Returns `{"order_id": ..., "fill_price": actual_fill}`. `evaluate_momentum()` updates DB `entry_cost` immediately when actual_fill differs from assumed ask. Logged as `[MOMENTUM] actual fill @ $0.xxx (assumed $0.yyy)`. **Keith's rule: Polymarket is source of truth for all entry prices. Never assume.**

- **Paper data cleanup + P/L reset**: to wipe all paper trading data while keeping live: `DELETE FROM paper_trades WHERE is_live = 0`. After deletion, ALL remaining midpoint_scalp rows are live — remove the web UI exclusions (see above). Then reset P/L baseline to current wallet: `UPDATE crypto_engine_state SET value = '<current_balance>' WHERE key = 'wallet_start_balance'`. This resets Actual P/L to $0.00 from the current balance. The web UI reads `wallet_start_balance` from DB first (line ~727), falling back to health log only if DB is null — so DB value takes priority. The engine does NOT write this value on startup; only the web UI or manual SQL updates it.\n\n- **Stale BTC price at resolution — window_start_price overwritten before resolution runs**: The `_close_old_window_positions()` function resolves positions from the PREVIOUS window, but it runs AFTER `state.window_start_price` has been overwritten with the NEW window's BTC price. Result: `btc_start` and `btc_end` are both captured from the new window → they're always nearly identical → ties defaulted to UP (see next pitfall). **Fix**: (1) capture `_old_btc_start = state.window_start_price` and `_old_btc_end = state.btc_chainlink` BEFORE `state.window_start_price` is overwritten (lines ~481-486), (2) pass both as parameters to `_close_old_window_positions(old_slug, btc_start, btc_end)` instead of reading stale state, (3) remove the `with state.lock:` block from inside `_close_old_window_positions()` since prices are now pre-captured. The old window's start price and end price are now accurate because they're snapshot at the moment the window transitions.\n\n- **Tie-breaker bug — flat BTC resolves to wrong winner**: Resolution code: `winner = \"Down\" if btc_end < btc_start else \"Up\"`. When `btc_end == btc_start` (flat BTC), the condition is False → defaults to `\"Up\"`. But on Polymarket, flat BTC means DOWN wins (UP token resolves to No — BTC was not above the start price). Fixed both occurrences (lines ~515 and ~1415) to: `winner = \"Down\" if btc_end <= btc_start else \"Up\"`. Ties now correctly resolve to DOWN. This combined with the stale price bug (above) caused UP trades to incorrectly show as wins when BTC was flat or moved down.\n\n- **Deposit/withdrawal detection — wallet transfers inflated Actual P/L**: When the user deposits funds to their Polymarket wallet, `wallet_balance - wallet_start_balance` increases without any corresponding trades. The UI would show this as profit. **Fix**: The web API (`app/web.py`) now compares `actual_pnl` (wallet change) with `live_pnl_dollars` (cumulative closed trade P/L). If the gap exceeds $0.50, it assumes an external transfer and adjusts by tracking `cumulative_deposits` in `crypto_engine_state`. The `actual_pnl` returned is: `wallet_balance - wallet_start_balance - cumulative_deposits`. This is computed per-request (no persisted side effects except the cumulative_deposits counter). The `wallet_start_balance` is NEVER auto-reset — it stays fixed for accurate multi-session tracking.\n\n- **DB Est P/L panel removed from web UI**: The \"DB Est P/L\" stat and the `renderLivePnL()` card were removed because they showed phantom trades (GTC orders recorded as live but never filled on-chain). The DB-calculated P/L was misleading — sometimes showing -$12.25 when the wallet was only down -$3.13. Removed: (1) `renderLivePnL(data)` call — replaced with empty string, (2) the \"DB Est P/L\" row from `liveMetrics`. The Live Trading section now shows only wallet-grounded metrics: Wallet Balance, Actual P/L, Live Deployed, Live Open. No more confusing phantom numbers.
