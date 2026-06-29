# Live Trading Architecture

`app/live_trading.py` — Polymarket CLOB v2 order placement for midpoint scalp only.

**SDK**: `py-clob-client-v2>=1.0` (v1 deprecated — returns `400 invalid order version`).

## Security: credential flow

```
.env.live (gitignored, local only)
  ├── POLYMARKET_PRIVATE_KEY   → signs EIP-712 orders locally
  ├── POLYMARKET_FUNDER         → public wallet address (0x...)
  └── LIVE_TRADING_ENABLED      → "true" to activate

docker-compose.yml (in repo)
  └── env_file: .env.live       ← references the file, no secrets inline
```

## Startup flow (v2 two-step auth)

1. `live_trading.init()` called in `main()`
2. Checks `LIVE_TRADING_ENABLED` — if not "true", stays in paper mode
3. **L1 auth**: Creates `ClobClient(host=…, chain_id=137, key=PRIVATE_KEY)` (L1 only)
4. Calls `create_or_derive_api_key()` — derives API credentials from wallet signature
   - NOTE: may log `400 "Could not create api key"` internally but still returns valid `ApiCreds`
5. **L2 auth**: Creates fully-authenticated `ClobClient(host=…, chain_id=137, key=PRIVATE_KEY, creds=creds)`
6. Now ready for order placement

## Env vars needed

```bash
LIVE_TRADING_ENABLED="true"              # ON/OFF switch
POLYMARKET_PRIVATE_KEY="0x…"            # Polygon wallet private key
POLYMARKET_FUNDER="0x…"                 # Wallet public address
# POLYMARKET_HOST defaults to https://clob.polymarket.com
# POLYMARKET_CHAIN_ID defaults to 137 (Polygon mainnet)
```

**No API_KEY, API_SECRET, or API_PASSPHRASE needed.** The v2 SDK's `create_or_derive_api_key()` generates these from the wallet signature.

## Order types

| When | Order | Type | Stays on book? |
|------|-------|------|---------------|
| Entry | BUY limit @ ask | GTC | Fills immediately (market) |
| Immediately after | SELL limit @ ask+$0.04 | GTC | Yes — waits for bid |
| TP fills | — | — | DB update + re-assess entry |
| Window expires | CANCEL unfilled TP | Cancel | Removes from book |

## v2 API surface (vs deprecated v1)

| Operation | v1 (deprecated) | v2 (current) |
|-----------|-----------------|--------------|
| Import | `from py_clob_client...` | `from py_clob_client_v2...` |
| Client init | `ClobClient(host, key, funder)` | `ClobClient(host, chain_id, key)` |
| Auth | `derive_api_key()` auto-called | Two-step: L1→`create_or_derive_api_key()`→L2 |
| Place order | `create_order()` + `post_order()` | `create_and_post_order(args, options, type)` |
| Side | `"BUY"`, `"SELL"` (strings) | `Side.BUY`, `Side.SELL` (enums) |
| Order type | `"GTC"` (string) | `OrderType.GTC` (enum) |
| Options | None or optional | `PartialCreateOrderOptions(tick_size="0.01")` REQUIRED |
| Cancel | `cancel(order_id)` | `cancel_order(order_id)` |
| Balance | `get_balance()` | `get_balance_allowance(asset_type=AssetType.USDC)` |

## Fill detection — polling (WebSocket TODO)

Current: polls `get_order()` every ~5 seconds via `_check_live_scalp_fills()`.
Future: Polymarket User Channel WebSocket for real-time fill events.

## Minimum capital

SCALP_SHARES=1, tokens at ~$0.50 → $0.50 per entry. SCALP_MAX_OPEN=3 → max $1.50 deployed.
Recommended starting balance: $20-$50.

## Diagnosing failed orders

```bash
# Check the live orders log for errors
docker exec polymarket-intel-crypto python3 -c "
import json
with open('/logs/live_orders.jsonl') as f:
    for line in f:
        e = json.loads(line)
        if e.get('status') == 'error':
            print(f'{e[\"type\"]} #{e[\"trade_id\"]} | {e.get(\"error\",\"\")[:120]}')
"
```

Common errors:
- `403 Trading restricted in your region` → VPN country is blocked. Change `SERVER_COUNTRIES` in `.env.vpn`.
- `400 invalid order version` → Using deprecated v1 SDK. Upgrade to `py-clob-client-v2>=1.0`.
- `400 Could not create api key` → Cosmetic during L1 auth; derive fallback usually succeeds.

## Enabling live trading

1. Edit `/root/polymarket-intel/.env.live`
2. Fill in `POLYMARKET_PRIVATE_KEY`, `POLYMARKET_FUNDER`, `LIVE_TRADING_ENABLED=true`
3. Ensure VPN country is Polymarket-allowed (NOT US/UK/France) — check `.env.vpn` `SERVER_COUNTRIES`
4. `docker compose up -d crypto`
5. Verify: `docker compose logs crypto | grep "\[LIVE\]"` — should show `ClobClient v2 initialised`
6. Check for errors: `docker compose logs crypto | grep "FAILED"`
