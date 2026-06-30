# On-Chain Wallet Forensics (Polygon)

Use when tracing where funds went after a suspected wallet drain, especially for Polymarket-related wallets.

## Key facts about Polymarket's balance model

- **Polymarket does NOT hold USDC at the user's wallet address.** Deposits go to Polymarket's smart contracts. The `get_balance_allowance()` API reports an internal ledger balance.
- **On-chain balance at the user's address will always be $0** after depositing to Polymarket. This is normal.
- **Withdrawals appear on-chain as:** Polymarket_contract → recipient_address, NOT user_address → recipient_address.
- Scanning the user's wallet for outgoing USDC transfers will NOT find Polymarket withdrawals.

## Scanning for USDC transfers on Polygon

USDC (native) contract: `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359`
USDC.e (bridged): `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`

Transfer event signature: `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`

### Finding outgoing transfers from a wallet
```python
from_topic = '0x000000000000000000000000' + address[2:].lower()
logs = w3.eth.get_logs({
    'address': usdc_contract,
    'topics': [transfer_sig, from_topic]  # indexed 'from' = first topic after sig
})
```

### Finding incoming transfers to a wallet
```python
to_topic = '0x000000000000000000000000' + address[2:].lower()
logs = w3.eth.get_logs({
    'address': usdc_contract,
    'topics': [transfer_sig, None, to_topic]  # indexed 'to' = second topic after sig
})
```

### Decoding transfer values
USDC has 6 decimals. Value = `int(log['data'].hex(), 16) / 1_000_000`.

## RPC rate limits (critical)

Free/public Polygon RPCs have strict rate limits:
- `polygon-rpc.com` — often times out
- `1rpc.io/matic` — works but has per-plan limits
- `llamarpc.com` — sometimes works
- `maticvigil.com` — usually blocked

**Pattern for batch scanning with rate limits:**
```python
for start in range(start_block, end_block, 50):
    try:
        logs = w3.eth.get_logs({'fromBlock': start, 'toBlock': start + 49, ...})
        time.sleep(0.5)  # rate limit guard
    except Exception as e:
        if 'limit' in str(e).lower():
            time.sleep(10)  # back off on rate limit
```

Block ranges are limited to ~50 blocks per call on free RPCs. With Polygon's ~2s block time, 50 blocks = ~100 seconds. Scanning hours requires many iterations and is slow.

## Checking current balances (no scanning needed)

```python
# Native MATIC
matic = w3.eth.get_balance(address)
# USDC (call contract directly)
usdc = w3.eth.contract(address=usdc_addr, abi=erc20_abi)
balance = usdc.functions.balanceOf(address).call()
```

Single RPC call per token — no rate limit concern. Use this first.

## Known scammer wallet patterns

- Funds appear as USDC, not conditional tokens
- Clean round amounts (not partial fills)
- Same destination wallet receives from multiple victims
- Wallet is a regular EOA: `w3.eth.get_code(address)` returns `0x`

## Practical forensic flow

1. Check on-chain balances first (MATIC, USDC, USDC.e) — instant
2. If $0, check Polymarket health logs for time-of-drain
3. Scan blocks around that timestamp for USDC transfers FROM known Polymarket contract addresses TO suspect wallet
4. If RPC rate-limits prevent scanning, use polygonscan.com manually with suspect address
5. Cross-reference amount with missing Polymarket balance — close match = confirmed drain path
