# Private Key Security Audit Checklist

Use when a user's wallet was drained and the agent's codebase stores a private key.

## Audit steps (run in order)

### 1. Git history check
```bash
cd <project> && git log --all --oneline -- .env.live .env.prod
git log --all -p -- .env.live | head -20
```
Confirm `.env` files are in `.gitignore` and never committed.

### 2. Log file scan
```bash
grep -r "PRIVATE_KEY\|0x[a-f0-9]\{64\}" logs/ --include="*.log" --include="*.jsonl"
```
Zero matches = clean.

### 3. Docker log scan
```bash
docker logs <container> 2>&1 | grep -i "private\|0x[a-f0-9]\{64\}"
```
Zero key matches = clean. Order IDs and tx hashes are fine.

### 4. File permissions
```bash
stat <project>/.env.live
```
Must be `0600` (owner read/write only).

### 5. File search
```bash
find <project> -name "*.env*" -o -name "*.log" | xargs grep -l "PRIVATE_KEY"
```
Should only return the `.env.live` file itself.

### 6. SSH access
```bash
last -20
grep "Accepted\|Failed\|session opened" /var/log/auth.log | tail -15
```
Verify only known user IPs. Look for brute-force attempts (failed passwords from unknown IPs).

### 7. Key transmission paths
In the code, trace every usage of `os.getenv("PRIVATE_KEY")` or equivalent:
- Only used to initialize crypto clients (PyClobClient, web3, etc.)
- Never sent to external APIs except the intended service (e.g., `clob.polymarket.com`)
- Never logged, never included in error messages

## Interpretation

- **Single private key leak**: Only affects one blockchain address. Explains drain on that chain, but NOT drains on other chains.
- **Seed phrase compromise**: Affects ALL addresses derived from that seed. Explains multi-chain simultaneous drains (ETH + BSC + Polygon). A single private key CANNOT recover the seed phrase, so a single-key leak from one env file cannot cause multi-chain drains unless that same seed was exposed elsewhere.

## Polymarket-specific

Polymarket's `get_balance_allowance()` reports internal exchange balance, not on-chain balance. The wallet's USDC may be held in Polymarket's smart contracts, not at the wallet address. On-chain scanning at the wallet address will show $0 even if Polymarket shows a balance. This is normal.

When funds are withdrawn via Polymarket's API, the on-chain USDC transfer goes from Polymarket's contract → recipient, NOT from the user's wallet → recipient. Scanning the user's wallet for outgoing USDC transfers will find nothing.
