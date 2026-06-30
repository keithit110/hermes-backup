# Polymarket Deposit Wallet Auth — Complete Debugging Session (2026-06-28)

## The problem

Live orders to Polymarket CLOB API always returned `400 the order signer address has to be the address of the API KEY`, even after:
- Upgrading from py-clob-client v1 to v2 (1.0.1)
- Switching VPN from UK (geoblocked) to Canada
- Changing signature types between EOA, POLY_1271
- Multiple API key deletions and recreations

## What we tried (all failed)

1. **EOA signature type**: `400 maker address not allowed, please use the deposit wallet flow`
2. **POLY_1271 with derived keys**: `400 the order signer address has to be the address of the API KEY`
3. **Monkeypatch L1 auth to use funder as POLY_ADDRESS**: `401 Invalid L1 Request headers` — wrong approach
4. **Deposit wallet "proxy deployment" via UI trade**: User placed a trade on Polymarket.com (confirmed in portfolio history as +$3.51 win). Did NOT fix the signer error.
5. **API key deletion + recreation**: Keys could be created successfully but orders still failed.

## Root cause

**The funder address was WRONG.** The EOA (derived from private key) is `0xa5Df31bB4cDD4c94E789C6D7ac302662EE7934B9`. The deposit wallet is `0x0A47689Ab9025E1D6036856dFD52Edd588eDc7d8`. They are DIFFERENT addresses.

The `.env.live` had `POLYMARKET_FUNDER=0xa5Df...` (the EOA). The order builder correctly set `maker=funder` and `signer=funder` for POLY_1271 orders. But since funder was the EOA, not the deposit wallet, the backend rejected every order.

## How to find the deposit wallet address

1. Go to https://polymarket.com
2. Click the avatar icon (top-right corner)
3. Dropdown shows username + truncated address
4. Click the **copy icon** next to the address
5. The copied value is the deposit wallet — NOT the EOA

## Verified working configuration

```bash
# .env.live
LIVE_TRADING_ENABLED=true
POLYMARKET_PRIVATE_KEY=0xc519b3f8db43f9302b15d6de2073025c0b64fd52f7f7947e420e55482d12cfa2
POLYMARKET_FUNDER=0x0A47689Ab9025E1D6036856dFD52Edd588eDc7d8  # ← deposit wallet from UI
```

```yaml
# docker-compose.yml crypto service
environment:
  SCALP_SHARES: "3"  # minimum order is $1.00; 3 × $0.47 = $1.41
```

## SDK configuration (no monkeypatch needed)

```python
sig_type = SignatureTypeV2.POLY_1271  # deposit wallet
funder = "0x0A47689Ab9025E1D6036856dFD52Edd588eDc7d8"  # deposit wallet, NOT EOA

# L1 auth uses EOA (correct)
l1_client = ClobClient(host=host, chain_id=chain_id, key=private_key,
                       signature_type=sig_type, funder=funder)
creds = l1_client.create_or_derive_api_key()

# L2 client for orders
_client = ClobClient(host=host, chain_id=chain_id, key=private_key,
                     creds=creds, signature_type=sig_type, funder=funder)
```

## Balance query fix

```python
from py_clob_client_v2.clob_types import BalanceAllowanceParams, AssetType
params = BalanceAllowanceParams(asset_type=AssetType.COLLATERAL, signature_type=3)
balance = _client.get_balance_allowance(params=params)
```

## Minimum order size

Polymarket requires ≥$1.00 per order. With midpoint scalp at $0.47-$0.53:
- 1 share: $0.47-$0.53 → REJECTED
- 2 shares: $0.94-$1.06 → REJECTED at low end
- 3 shares: $1.41-$1.59 → ALWAYS passes

## Result

After fixing the funder address, bumping shares to 3, and fixing the balance query:
```
[LIVE] BUY #1211 token=7928674678… @ 0.52 × 3 → order 0x8a34ec50...  ← SUCCESS
[SCALP] OPEN #1211 UP @ 0.52 | open=2/3 LIVE
[LIVE HEALTH] balance={'balance': '39976944', ...}  ← $39.98 USDC
```

No more signer errors. Live orders flowing.
