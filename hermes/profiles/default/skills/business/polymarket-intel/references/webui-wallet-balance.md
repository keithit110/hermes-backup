# Web UI wallet balance — real Polymarket balance in dashboard (updated 2026-06-29)

The dashboard shows actual Polymarket wallet balance pulled from the engine's health log, alongside DB-computed P/L. The DB-based P/L can be inaccurate because `is_live` flags are set at order submission time, and some "live" trades may have failed to fill. The real wallet balance is the ground truth.

## How it works

### Backend (`/api/summary` in `web.py`)

1. Check `crypto_engine_state` for persisted `wallet_start_balance` (survives log rotation)
2. Read `logs/live_health.jsonl` (written by `live_trading.log_health_heartbeat()` every ~60s)
3. Parse the latest entry for current balance
4. Fallback: if no persisted start balance, use first health log entry
5. Compute actual P/L = current - starting **server-side**
6. Return `wallet_balance`, `wallet_balance_ts`, `wallet_start_balance`, `actual_pnl` in the summary JSON

**`live_pnl_pct` REMOVED from the API response entirely** — percentages have no place in the live section.

### Starting balance persistence

Stored in `crypto_engine_state` table (`key='wallet_start_balance'`) so it survives health log rotation:

```sql
INSERT OR REPLACE INTO crypto_engine_state(key, value) VALUES('wallet_start_balance', '46.71');
```

Server-side computation:

```python
# /api/summary — actual_pnl computed server-side, not client-side
actual_pnl = round(wallet_balance - wallet_start_balance, 2) if (
    wallet_balance is not None and wallet_start_balance is not None
) else None
```

### Parsing pitfall: `balance_usdc` is Python dict repr, NOT JSON

The health log stores balance as `str()` of a Python dict, not `json.dumps()`:

```python
# What's in the JSONL file:
"balance_usdc": "{'balance': '40382603', 'allowances': {...}}"
# NOT valid JSON because single quotes
```

The parsing workaround in `web.py`:

```python
bal = entry.get("balance_usdc")
if bal and isinstance(bal, str):
    bal = bal.replace("'", '"')  # convert single quotes to double
    try:
        bal = json.loads(bal)
    except Exception:
        pass
if isinstance(bal, dict):
    raw = bal.get("balance", "0")
    wallet_balance = float(raw) / 1_000_000.0  # 6 decimal pUSD
```

### Frontend (`PAGE` template in `web.py`)

The live metrics section uses `s.actual_pnl` from the API (server-computed, NOT client-computed):

```javascript
let actualPnL = s.actual_pnl;  // server-computed, no client-side math
document.getElementById('liveMetrics').innerHTML = [
    ['Wallet Balance', balDisplay, 'real Polymarket balance'],
    ['Actual P/L', actualPnLDisplay, 'vs starting balance'],
    ['Live Deployed', '$' + liveDeployed.toFixed(2), 'capital in open live positions'],
    ['Live Open', s.open_live || 0, 'active live positions'],
    ['DB Est P/L', '$' + livePnL.toFixed(2), 'from trade records (not actual)'],
];
```

## Why DB P/L can be wrong

1. `is_live=1` is set when order is SUBMITTED, not when it fills
2. A buy may succeed on the CLOB but never match (the price moved away)
3. The DB records the entry price but the shares were never acquired
4. FOK market sells can fail silently while DB marks position as closed (fixed 2026-06-29)
5. When the window resolves, the DB thinks it was a live trade with full shares

The computed P/L from health log (start → current balance) is the ONLY reliable source.

## Midpoint_scalp filtered from paper UI (2026-06-29)

When midpoint scalp went fully live, all paper-only records (`is_live=0`) were deleted. The UI hard-filters midpoint_scalp out of the paper trades table, strategy dropdown, open counts, and P/L summaries:

- `/api/paper`: `WHERE kind != 'midpoint_scalp'`
- `applyPaperFilters()`: `data.filter(r => r.kind !== 'midpoint_scalp')` BEFORE user filters
- `/api/summary`: `kind != 'midpoint_scalp'` on all queries (open_paper, closed_trades)
- Strategy dropdown: naturally excluded (no rows → no dropdown entry)

Live records (`is_live=1`) remain in the DB for the engine's TP checker and open-position tracking.

## SQLite WAL mode for web.py

The web container's `conn()` helper enables WAL mode on every connection:

```python
def conn() -> sqlite3.Connection:
    c = sqlite3.connect(DB, timeout=10)
    c.execute("PRAGMA journal_mode=WAL")
    c.execute("PRAGMA busy_timeout=5000")
    c.row_factory = sqlite3.Row
    return c
```

## Volume mount

The web container needs `./logs:/logs` volume mount to read the health log.

## Verification

```bash
curl -s http://localhost:8095/api/summary | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('Wallet Balance:', d.get('wallet_balance'))
print('Start Balance:', d.get('wallet_start_balance'))
print('Actual P/L:', d.get('actual_pnl'))
print('DB Est P/L:', d.get('live_pnl_dollars'))
"
```
