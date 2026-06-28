# Crypto Engine Debugging — When Trades Stop

Systematic diagnostic workflow when the crypto engine produces zero entries for extended periods (1+ hours).

## Step 1: Check when the last trade occurred

```bash
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT opened_at, kind, entry_cost, status FROM paper_trades
   WHERE kind LIKE 'crypto_5m_%'
   ORDER BY opened_at DESC LIMIT 5;"
```

If last trade is hours old but status shows recent wins → the engine is alive but not entering.

## Step 2: Identify the dominant rejection reason

```bash
docker logs polymarket-intel-crypto --tail 200 2>&1 | grep IGNORE | sort | uniq -c | sort -rn
```

Three possible dominant rejections:

### A. `pct_change X% too close to 0` (most common)

The pct_change floor (`CRYPTO_MIN_PCT_CHANGE`, default 0.0008 = 0.08%) is blocking entries. Get exact pct_change distribution:

```bash
docker logs polymarket-intel-crypto --tail 500 2>&1 | grep -oP 'pct_change [\d.]+%' | sort -t' ' -k2 -n | uniq -c | sort -rn
```

At $60K BTC: 0.08% = ~$48 move needed. If max observed is 0.064%, floor is too high.

**Fix**: Lower `CRYPTO_MIN_PCT_CHANGE` in docker-compose.yml (env block). Only with Keith's approval. Rebuild: `docker compose up -d --build crypto`.

History of floor adjustments:
- Original: 0.1% → too tight
- 2026-06-26: lowered to 0.08% → still blocked during consolidation
- TBD: may need 0.04% for ultra-low-volatility periods

### B. `Xs outside 60-180s window`

Normal — the engine only enters between 60-180s into the 5-min window. Brief gap at window start and end.

### C. `edge_up=X% edge_down=Y% below min 0.05`

Check if both edge AND max_ask are failing. Specifically: `edge_down` TECHNICALLY passes `>= 0.05` check but `down_ask` may be > 0.85. The log message is misleading — real rejection is `MAX_ENTRY_PRICE`. See pitfall #12 in main skill.

## Step 3: Verify the engine is running

```bash
docker logs polymarket-intel-crypto --tail 5 2>&1
```

Should show `[WINDOW] New window start=...` and Chainlink/Binance price updates. If nothing for minutes, engine may be crashed — check `docker ps | grep crypto`.

## Step 4: Check if entries are even being attempted

```bash
docker logs polymarket-intel-crypto --tail 500 2>&1 | grep -c "EVAL\|LANE2"
```

0 evaluations = engine connection issues (VPN, WebSocket). Many evaluations but all IGNORE = filter too tight.

## Step 5: Verify BTC price and volatility

```bash
# Current Chainlink price
docker logs polymarket-intel-crypto --tail 20 2>&1 | grep -oP 'chainlink [\d.]+' | tail -1

# Recent pct_change values (volatility proxy)
docker logs polymarket-intel-crypto --tail 300 2>&1 | grep -oP 'pct_change [\d.]+%' | tail -20
```

If pct_change is persistently <0.08% → BTC is in dead-flat consolidation. The engine is working correctly (protecting against noise entries). The floor may need lowering.

## Decision tree

```
No trades for 1+ hours?
├─ Last trade was win → engine WAS working
├─ Logs show "pct_change too close to 0" → floor too tight for current volatility
│  ├─ Max pct_change ≥ 0.06% → lower floor to 0.04%
│  └─ Max pct_change < 0.03% → BTC dead flat, floor won't help (no edge to capture)
├─ Logs show "outside 60-180s window" only → rare edge case at window boundaries, wait
├─ Logs show "below min 0.05" → likely max_ask > 0.85, market repricing too fast (correct behavior)
└─ No log output → engine crashed, restart with docker compose
```
