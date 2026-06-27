# Crypto Engine Silence — Diagnostic Pattern

When Keith asks "why aren't we getting any crypto trades?", the answer is always in the container logs.
Never speculate — `grep` the logs, categorize, and present a summary table.

## Diagnostic command

```bash
docker logs polymarket-intel-crypto --tail 200 2>&1 | grep "IGNORE\|Lane\|ERROR\|no live"
```

## Rejection categories (engine is healthy)

| Log pattern | Meaning | Action |
|-------------|---------|--------|
| `pct_change 0.0X% too close to 0` | BTC flat — no directional signal | Normal; wait for volatility |
| `edge_up=X% edge_down=Y% below min 0.05` | Edge exists but below 5% threshold | Normal; model not confident enough |
| `Xs outside 60-180s window` | Window too early or too late | Normal; entry zone hasn't started or passed |
| `IGNORE: ask >= 0.85` | Market odds too high — price already moved past entry zone | Normal; BTC clearly moving, too late to enter |
| `failed to map slug` | Gamma API has no live market for this 5-min window | Normal; off-hours or API gap |
| `no live trades to resolve` | Nothing open to close | Normal |

## When to investigate further

| Pattern | Concern |
|---------|---------|
| `ERROR` or `Traceback` in logs | Engine crash — check `docker inspect` health |
| All `IGNORE` for >4 hours with significant BTC movement | Edge thresholds might need calibration |
| `failed to map slug` on every single window | Gamma API connection issue |
| Chainlink price stale (>30s since last update) | WebSocket disconnected |

## Edge misleading in logs

The log line `edge_up=-80.8% edge_down=80.9% below min 0.05` is a **catch-all message** for both edge AND price checks. Edge can be high (80.9%) but still rejected because `down_ask >= MAX_ENTRY_PRICE` (0.85). Always check BOTH: edge value AND the secondary price filter at line 282 of `crypto_engine.py`.

## Quick health check

```bash
# Engine running?
docker ps --filter name=polymarket-intel-crypto --format '{{.Status}}'

# Last trade?
sqlite3 data/polymarket_intel.sqlite "SELECT MAX(opened_at) FROM paper_trades WHERE kind='crypto_5m_late_directional'"

# Open positions?
sqlite3 data/polymarket_intel.sqlite "SELECT COUNT(*) FROM paper_trades WHERE kind='crypto_5m_late_directional' AND status='open'"

# Chainlink price feed alive?
docker logs polymarket-intel-crypto --tail 5 2>&1 | grep -c "IGNORE\|entered\|no live"
```
