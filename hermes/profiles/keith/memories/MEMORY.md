Hermes backup: local /root/hermes-backup, GitHub remote keithit110/hermes-backup. Script /usr/local/bin/hermes-github-backup, cron daily 00:00 UTC. USER.md ≠ MEMORY.md — when asked about backup state, verify against repo, not memory.
§
For Cebu Direct Stays, Keith prefers avoiding full host accounts/dashboard early; for features like availability calendars, favor admin-managed/manual updates, private magic-link editing, or iCal import over requiring hosts to create profiles.
§
Keith's scheduled TLDR news briefings should include latest AI updates and use only accessible, working sources; inaccessible/paywalled/problematic sources should be skipped or removed rather than retried.
§
Polymarket /root/polymarket-intel: web UI :8095. Crypto engine: directional-only, variable shares (≥55%→3, ≥48%→2, else 1), Chainlink BTC/USD via Poly RTDS (free, resolution source), Binance fallback. All 3 containers route through Surfshark UK VPN (gluetun proxy :8888). Paper-only. 65-day data retention. Capital-weighted P/L with shares. Open trades never show P/L. Don't change UI without Keith asking. GitHub: keithit110/Polymarket-hermes-project, zero secrets policy.
§
Keith expects AI agents to actually USE collected data (Markets, News, Research) when making trading decisions — not just collect it for display. He was frustrated to learn the agents write to these tables but never read from them.
§
Keith requires that I PROPOSE changes first and wait for his explicit approval before making any modifications — especially config changes, parameter adjustments, or strategy logic. Never change anything without first presenting the proposal and getting a "go ahead" or similar explicit approval. This applies to: docker-compose.yml, crypto_engine.py, main.py, web.py, .env, .env.vpn, cron jobs, and any other project files.
§
Live Polymarket trading needs only: wallet address + private key. No RPC endpoint, no proxy contract approval — engine signs EIP-712 orders like the website. Keith's wallet 0xa5Df... has $46.94 USDC on Polygon.