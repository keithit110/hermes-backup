# Strategy Not Trading — Diagnostic Workflow

Step-by-step guide for diagnosing "the strategy hasn't made a trade in hours."

## Real session example (2026-07-01)

**User report:** "diagnose the momentum follower strategy? It hasn't made a trade for several hours now."

**What we found:**
- Engine was alive and evaluating (HEARTBEAT lines, `[MOMENTUM] checking…` output)
- Strategy was detecting signals and attempting FAK buys
- BUT: every buy failed with `not enough balance / allowance: the balance is not enough -> balance: 0`
- 75 failed buys, 6 successful total (lifetime counter)
- Balance went from $75.99 to $0.00 between 23:59 and 00:04 UTC
- Last successful trade (#1754) cost only $3.75 — $72 unaccounted for
- Net trading P&L was -$0.05 across 92 resolved trades (essentially breakeven)

**Root cause:** Wallet USDC balance dropped to zero. The engine was working correctly but had no capital.

**Unresolved:** ~$72 missing — balance dropped far more than the last trade cost. Possible withdrawal, conditional token redemption, or API reporting glitch. User needs to check Polymarket transaction history directly.

## Step-by-step diagnostic

### Step 1: Check engine is alive

```bash
docker logs polymarket-intel-crypto --tail 50 | grep -E 'MOMENTUM|HEARTBEAT|LIVE HEALTH'
```

Normal output:
- `[HEARTBEAT] iter=N btc=... window=... up_ask=... down_ask=...` — engine alive
- `[MOMENTUM] checking… s_left=N pct=... thresh=... | no signal` — strategy evaluating but no signal yet
- `[MOMENTUM DBG] s_left=N btc_mid=... start=... pct=... thresh=... in_window=False` — outside trigger window
- `[LIVE HEALTH] balance={...} open_tps=... fills=... | ok: BUY=N | err: BUY=N` — health snapshot

### Step 2: Check for order errors

```bash
docker logs polymarket-intel-crypto --tail 100 2>&1 | grep -E 'FAILED|balance|error|PolyApiException'
```

Key error patterns:

| Pattern | Meaning |
|---|---|
| `not enough balance / allowance: the balance is not enough -> balance: 0` | Wallet has $0 USDC — can't trade |
| `PolyApiException[status_code=400]` | API rejected order (check error message for reason) |
| `PolyApiException[status_code=403]` | Geo-blocked — VPN country not allowed |
| `database is locked` | SQLite contention (see DB Locking section in parent skill) |

### Step 3: Trace wallet balance over time (THE KEY DIAGNOSTIC)

```bash
tail -50 /root/polymarket-intel/logs/live_health.jsonl
```

The `balance_usdc` field is a Python dict repr (single quotes), not JSON. The balance value is in smallest units (6 decimal places for USDC). To convert to dollars:

```python
import ast
balance_str = "{'balance': '75998814', ...}"
balance_dict = ast.literal_eval(balance_str)
usdc = int(balance_dict['balance']) / 1_000_000  # 75.998814
```

Look for:
1. The exact timestamp when balance changed significantly
2. Cross-reference with `live_orders.jsonl` to see what trades executed at that time
3. If the drop exceeds the cost of trades, investigate for withdrawals or API glitches

### Step 4: Check successful vs failed orders

```bash
# All successful BUY_FAK orders (with fill prices)
grep -v 'error' /root/polymarket-intel/logs/live_orders.jsonl | grep 'BUY'

# Last 20 failed orders
grep 'error' /root/polymarket-intel/logs/live_orders.jsonl | tail -20

# Count successes
grep '"status": "success"' /root/polymarket-intel/logs/live_orders.jsonl | wc -l

# Count failures  
grep '"status": "error"' /root/polymarket-intel/logs/live_orders.jsonl | wc -l
```

The `ok: BUY=N | err: BUY=N` in live health is a LIFETIME counter — it includes all trades since the container started. It's the same data as counting the JSONL file, but faster to check.

### Step 5: Check recent trades and net P&L

```bash
# Recent momentum trades
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT id, opened_at, substr(event_slug, -10) as window, entry_cost, pnl_pct, status
   FROM paper_trades WHERE source='momentum_follower' ORDER BY id DESC LIMIT 15;"

# Net P&L breakdown (5 shares per trade)
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT status, COUNT(*) cnt, ROUND(SUM(entry_cost*5),2) total_cost,
          ROUND(SUM(CASE WHEN status='closed_resolved_win' THEN 5.0 ELSE 0 END),2) total_return
   FROM paper_trades WHERE source='momentum_follower'
   AND status IN ('closed_resolved_win','closed_resolved_loss') GROUP BY status;"

# Net P&L in dollars (closed trades only, accounting for 5 shares)
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT ROUND(SUM(CASE WHEN status='closed_resolved_win' THEN (1.0-entry_cost)*5
                          WHEN status='closed_resolved_loss' THEN -entry_cost*5 ELSE 0 END), 2) as net_pnl
   FROM paper_trades WHERE source='momentum_follower'
   AND status IN ('closed_resolved_win','closed_resolved_loss');"

# Check for open/stranded trades
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT id, opened_at, status, entry_cost FROM paper_trades
   WHERE source='momentum_follower' AND status NOT IN ('closed_resolved_win','closed_resolved_loss')
   ORDER BY id DESC LIMIT 10;"
```

### Step 6: Check engine state

```bash
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT * FROM crypto_engine_state WHERE key IN ('wallet_start_balance','cumulative_deposits');"
```

The `wallet_start_balance` is the user's deposited capital baseline. `cumulative_deposits` tracks additional deposits detected (negative = withdrawal/deduction in some conventions, positive = deposit).

### Step 7: Verify with Polymarket directly

The engine's built-in health check calls `get_balance_allowance()` every cycle (throttled to 60s). The results in `logs/live_health.jsonl` are authoritative. If needed, a manual API call can be made:

```bash
docker exec polymarket-intel-crypto python3 -c "
from py_clob_client_v2.client import ClobClient
from py_clob_client_v2.clob_types import ApiCreds, BalanceAllowanceParams, AssetType
import os
client = ClobClient(os.environ['POLYMARKET_HOST'], key=os.environ['POLYMARKET_PRIVATE_KEY'],
                    chain_id=int(os.environ.get('POLYMARKET_CHAIN_ID',137)),
                    creds=ApiCreds(api_key='', api_secret='', api_passphrase=''))
client.set_api_creds(client.create_or_derive_api_key())
balance = client.get_balance_allowance(params=BalanceAllowanceParams(asset_type=AssetType.COLLATERAL, signature_type=3))
print('Balance:', balance.get('balance','N/A'))
"
```

Note: the import is `py_clob_client_v2` (not `py_clob_client`). The method is `create_or_derive_api_key()` (not `create_or_derive_api_creds()`).

## Common causes summary

| Symptom | Likely Cause | Fix |
|---|---|---|
| `not enough balance: balance: 0` in logs | Wallet USDC depleted | Deposit funds on Polymarket |
| Only `[MOMENTUM DBG]` lines, all `in_window=False` | BTC not moving enough, or outside entry window | Normal quiet period — strategy is alive, just waiting |
| Zero MOMENTUM output of any kind | Engine crashed or MOMENTUM_ENABLED=false | Check container logs for crash/startup; check env vars |
| Balance dropped $76 but last trade was only $3.75 | Possible withdrawal, conditional token redemption, or API glitch | Check Polymarket UI for recent transactions |
| `database is locked` errors | SQLite contention | Check WAL/busy_timeout settings across all containers |
| Successful buy count stuck at 6 while errors at 75 | Wallet ran out of funds after 6th buy; engine keeps trying and failing | Normal for $0 balance — deposit to resume |

## PITFALL: wallet_activity table ≠ our wallet

The `wallet_activity` table is populated by the smart-wallet SCANNER. It tracks OTHER traders' proxy wallets (BreakTheBank, swisstony, etc.), not our own trading wallet. When diagnosing where our funds went:

- ❌ `SELECT * FROM wallet_activity WHERE wallet LIKE '%0x0A47%'` — returns nothing
- ✅ Check `logs/live_health.jsonl` for our wallet's balance history
- ✅ Check `logs/live_orders.jsonl` for our wallet's order events
- ✅ Check Polymarket.com directly for withdrawal/transfer history
