# Deposit Wallet Proxy Deployment — Critical Prerequisite for Live Trading

## Problem

New Polymarket users (post-2026-04-28) with wallet-login (MetaMask/EOA) have a
**counterfactual deposit wallet proxy**. The proxy address exists (deterministic CREATE2)
but has **zero bytecode** until the first on-chain transaction.

Without deployed bytecode:
- `eth_getCode(funder)` returns `0x`
- EIP-1271 (`isValidSignature()`) fails — no contract to call
- Server returns `400 the order signer address has to be the address of the API KEY`
  (misleading — the real issue is undeployed proxy, not API key binding)

## The 400 error chain we hit

```
1. py-clob-client v1: "invalid order version"       → upgraded to v2
2. py-clob-client v2 EOA: "maker address not allowed" → need POLY_1271
3. POLY_1271: "order signer address has to be address of API KEY" → proxy undeployed
```

Step 3 is NOT a code bug — it's an on-chain prerequisite.

## Fix: One UI trade

1. Open https://polymarket.com in browser with MetaMask
2. Place one small order (any market, any size — even $1)
3. This deploys the proxy (~250 bytes) AND sets unlimited USDC allowances for:
   - CTF Exchange V2
   - Neg-Risk Adapter
   - Neg-Risk CTF Exchange
4. After this, SDK orders work with `SignatureTypeV2.POLY_1271`

## Verification

```python
from web3 import Web3
w3 = Web3(Web3.HTTPProvider('https://1rpc.io/matic'))
code = w3.eth.get_code(Web3.to_checksum_address(FUNDER_ADDRESS))
print(f'Proxy deployed: {code not in (b"", b"0x")}  |  bytes: {len(code)}')
# Must be > 0 (should be ~250 bytes)
```

## GitHub issue reference

https://github.com/Polymarket/py-clob-client-v2/issues/64

> "for me the issue was that my deposit wallet was not deployed on chain. Placing 1 small first order from UI solved it"

> "i solved the issue just by placing one trade of $1 — it asked some sign in and migration and got new api key address and it worked"

## API key creation (after proxy deployed)

After the proxy is deployed, create CLOB API keys at https://polymarket.com/settings/api-keys
(NOT Relayer API keys — Polymarket has two separate key systems).

Copy all three values (shown once):
- apiKey
- secret
- passphrase

Set in `.env.live`:
```bash
POLYMARKET_API_KEY=...
POLYMARKET_API_SECRET=...
POLYMARKET_API_PASSPHRASE=...
```

## Signature type cheat sheet

| Value | Enum | Use case |
|-------|------|----------|
| 0 | EOA | Legacy — blocked for new users post-2026-04-28 |
| 1 | POLY_PROXY | Magic-wallet users only |
| 2 | POLY_GNOSIS_SAFE | Safe multisig wallets |
| 3 | POLY_1271 | Deposit wallets (MetaMask/EOA) — use this |
