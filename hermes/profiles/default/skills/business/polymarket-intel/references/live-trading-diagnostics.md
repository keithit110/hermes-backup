# Live Trading — Diagnostic Logging

*Updated 2026-06-28 after adding comprehensive health/heartbeat logging.*

## Log files

All logs are in `/root/polymarket-intel/logs/` (mounted as `/logs/` inside the crypto container).

| File | Format | Content | Interval |
|------|--------|---------|----------|
| `live_orders.jsonl` | JSONL | Every order event: BUY, SELL_TP, CANCEL_TP, TP_FILLED | Per event |
| `live_health.jsonl` | JSONL | Balance, open TP count, fill count, error/success counters | Every 60s |
| `scalp_decisions.jsonl` | JSONL | Strategy decisions: ENTRY, TP_FILLED, RESOLVED, SKIP | Per event |

## Health heartbeat

The `log_health_heartbeat()` function in `live_trading.py` runs every cycle (every ~1 second) but is internally throttled to once per 60s. It writes:

```json
{
    "balance_usdc": null,
    "open_tp_orders": 0,
    "total_fills": 5,
    "success": {"BUY": 5, "SELL_TP": 5},
    "errors": {},
    "live_enabled": true,
    "_ts": "2026-06-28T17:40:19.904428Z"
}
```

Errors by type (BUY, SELL_TP, CANCEL_TP) are tracked via `_bump_error()` / `_bump_success()` counters.

## Fill detection

`get_fill_status(trade_id)` polls the CLOB API via `_client.get_order(order_id)` every ~5 seconds. On first detection of a FILLED status, it logs a `TP_FILLED` event with the fill price and size. A `_logged_fills` set prevents duplicate fill notifications.

Follow-up in `crypto_engine.py`: `_check_live_scalp_fills()` runs every 5th evaluation cycle and calls `get_fill_status()` for each open scalp trade. When a fill is detected, it updates the DB trade row to `closed_take_profit`.

## Console output

`docker compose logs crypto` shows real-time summaries:
- `[LIVE] BUY #123 token=50148506... @ 0.49 × 1 → order abc123`
- `[LIVE] TP SELL #123 @ 0.53 → order def456`
- `[LIVE FILL] #123 TP filled | entry=0.490 exit=0.530 pnl=+8.16%`
- `[LIVE HEALTH] balance=$None open_tps=0 fills=5 | ok: BUY=5 SELL_TP=5 | err: none`

## Balance tracking

`get_balance()` calls `_client.get_balance_allowance()`. May return None even when order placement works — this is cosmetic. The health heartbeat tolerates null balances.

## Path resolution pitfall

The Dockerfile copies `app/` to `/app/app/`. Relative log paths from `os.path.dirname(__file__)` resolve to `/app/logs/` but the Docker volume mounts `./logs` to `/logs/`. Always use the env-var-based path `os.environ.get("LIVE_LOG_DIR", "/logs")` to write to the host-visible directory.

## Post-deployment verification

```bash
# Check engine is live-capable
docker compose logs crypto | grep "ClobClient initialised"

# Check health heartbeat is writing
tail -1 /root/polymarket-intel/logs/live_health.jsonl | python3 -m json.tool

# Check order log for entries
wc -l /root/polymarket-intel/logs/live_orders.jsonl

# Real-time monitoring
docker compose logs crypto -f | grep "\[LIVE"
```
