# Wallet Drain Forensics

When the Polymarket wallet balance drops to $0 and the engine's order log doesn't explain it.

## Symptoms

- Health log shows balance was $X → next check shows $0
- Order log shows only small trades (~$3-5) in the drain window
- Balance drop ($72+) far exceeds total trading activity
- All subsequent BUY_FAK orders fail with "not enough balance: balance: 0"

## Diagnostic workflow

### Step 1: Cross-reference health log with order log

```bash
# Find the exact timestamp of the drop
tail -100 /root/polymarket-intel/logs/live_health.jsonl | python3 -c "
import sys, json, ast
lines = [json.loads(l) for l in sys.stdin]
for l in lines:
    bal_str = l.get('balance_usdc','')
    try: bal = ast.literal_eval(bal_str).get('balance','N/A')
    except: bal = 'N/A'
    print(f\"{l['_ts']} | balance={bal}\")
"

# Check orders around that time
grep '2026-06-30T00:' /root/polymarket-intel/logs/live_orders.jsonl | python3 -c "
import sys, json
for l in sys.stdin:
    d = json.loads(l)
    if d.get('status') == 'success':
        print(f\"{d['_ts']} | {d['type']} | tid={d['trade_id']} | price={d.get('price','')} size={d.get('size','')}\")
"
```

### Step 2: Verify on-chain balance

Check USDC, MATIC, and USDC.e balances directly on Polygon:

```python
from web3 import Web3

w3 = Web3(Web3.HTTPProvider('https://1rpc.io/matic'))
addr = w3.to_checksum_address('0x...')

# MATIC
matic = w3.eth.get_balance(addr)
print(f"MATIC: {w3.from_wei(matic, 'ether')}")

# USDC (native)
usdc = w3.eth.contract(
    address=w3.to_checksum_address('0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359'),
    abi=[{"constant":True,"inputs":[{"name":"_owner","type":"address"}],"name":"balanceOf","outputs":[{"name":"balance","type":"uint256"}],"type":"function"}]
)
bal = usdc.functions.balanceOf(addr).call()
print(f"USDC: {bal/1e6}")

# USDC.e (bridged)
usdc_e = w3.eth.contract(
    address=w3.to_checksum_address('0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174'),
    abi=[...]
)
print(f"USDC.e: {bal_e/1e6}")
```

If on-chain balance is 0 but Polymarket health showed $X, the funds were transferred out.

### Step 3: Determine if engine caused it

- Sum all live trade P&L from the order log's successful entries
- If the sum is a small fraction of the missing amount, the drain is EXTERNAL to the engine
- Example: $76 disappeared but total engine live P&L in the last 6 hours was -$1.80 → external drain

### Step 4: Check for seed phrase compromise

If multiple chains drain simultaneously (ETH, BSC, Polygon), the seed phrase is compromised. Every address derived from that seed is exposed — including the Polymarket wallet. The hacker has the private key and can sign any transaction.

**Check scammer addresses:**
```python
# From the user: the scammer wallets they identified
scammer1 = '0x311975ef0e589A695694e0fA406918B0D1E8b74C'  # ETH/VAI/Cake drain
scammer2 = '0x3D7140370aA9Aa142a922Bf1f308565C7CABE603'  # MATIC drain

# Check if they're contracts (EOA = hacker wallet)
code = w3.eth.get_code(w3.to_checksum_address(scammer1))
print(f"Is contract: {len(code) > 0}")

# Check current balances (funds may still be sitting there)
usdc_bal = usdc.functions.balanceOf(w3.to_checksum_address(scammer1)).call()
print(f"Scammer USDC: ${usdc_bal/1e6:.2f}")
```

### Step 5: Code audit

Verify the `.env.live` file is gitignored and never exposed:

```bash
cd /root/polymarket-intel
# Check permissions
ls -la .env.live  # must be -rw------- (600)
# Check git history
git log --all --oneline -- .env.live .env.vpn  # should be EMPTY
# Check .gitignore
grep '\.env' .gitignore  # should show .env* pattern
```

## Common root causes

| Cause | Indicators | Priority |
|---|---|---|
| Seed phrase compromise OR single-key leak | Multiple chains drain simultaneously | **URGENT** — create new wallet immediately |
| Polymarket API balance glitch | On-chain balance still positive | Low — wait and recheck |
| Unredeemed conditional tokens | Balance fluctuates with market resolution | Medium — tokens auto-redeem eventually |
| Engine order log gap | Balance drop matches trade P&L | Fix engine logging/retention |
| VPS host access | No breach evidence on server, but host can read disk | Medium — mitigate with dedicated bot wallet |

### CRITICAL: Private key alone is sufficient

One EVM private key controls the SAME address on ALL EVM chains:
```
Address: 0x0A47... → same key works on Ethereum, BSC, Polygon, Arbitrum, etc.
```

A leak of the POLYMARKET_PRIVATE_KEY alone explains ALL chain drains — no seed phrase needed. Every token on every chain at that address is accessible. This was initially misdiagnosed as "seed phrase compromise" — but a single private key exposure is sufficient for total loss on all chains sharing that address.

### VPS host trust model

Any VPS host (ColoCrossing, AWS, GCP, etc.) has hypervisor-level access and CAN technically:
- Read disk contents (including `.env.live`)
- Read process memory
- Sniff network traffic (if not end-to-end encrypted)

**Mitigation: dedicated bot wallet.** Create a separate wallet used ONLY for the trading bot with a small balance ($50-100). Main funds stay in a wallet whose keys never touch any server. If the VPS is compromised, losses are capped.

### SSH hardening (required setup)

Budget VPS servers attract constant brute-force attacks. Verify SSH security:
```bash
# Check for password auth on root (DANGEROUS)
grep 'PermitRootLogin' /etc/ssh/sshd_config
# Check brute-force volume
journalctl -u ssh --since today --no-pager | grep "Failed password" | wc -l
```

If `PermitRootLogin yes` with password auth, lock it down immediately:
```bash
sed -i 's/PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
echo 'PasswordAuthentication no' >> /etc/ssh/sshd_config.d/50-cloudimg-settings.conf
systemctl restart sshd
```

**Before doing this:** ensure an SSH key is installed in `/root/.ssh/authorized_keys` or you'll lock yourself out.

### Why Polymarket withdrawals are hard to trace on-chain

Polymarket holds USDC in their smart contracts, NOT in your wallet address. The on-chain withdrawal flow is:
```
Polymarket_contract → recipient_wallet
```
NOT:
```
your_wallet → recipient_wallet
```

When scanning for USDC transfers FROM your wallet address, you'll find NOTHING — the transfer came from Polymarket's address. To trace, scan for incoming transfers TO the suspected scammer address. Even then, Polymarket may use different proxy contracts for withdrawals, making attribution difficult.

Also: free Polygon RPCs (1rpc.io, polygon-rpc.com) have aggressive rate limits (50-block query windows, cooldowns). Scanning for specific transfers can take hours due to rate limiting. Polygonscan API v2 requires an Etherscan API key. The old v1 API is fully deprecated.

## Recovery steps (seed compromise)

1. Create NEW MetaMask wallet on a clean device
2. Write seed phrase on PAPER only — never digital
3. Never use old seed for anything ever again
4. Update `.env.live` with new wallet's private key and funder address
5. Deposit fresh USDC to new Polymarket wallet
6. Rebuild crypto container: `docker compose up -d --build crypto`

## PITFALL: wallet_activity table

The `wallet_activity` table contains smart-wallet scanner data — OTHER traders' proxy wallets. It does NOT contain our wallet's transactions. When diagnosing our wallet's activity, use `logs/live_health.jsonl` and `logs/live_orders.jsonl` ONLY. Do not query `wallet_activity` expecting to find our wallet.
