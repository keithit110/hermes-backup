Hermes backup: local /root/hermes-backup, remote keithit110/hermes-backup. Script /usr/local/bin/hermes-github-backup, cron daily 00:00 UTC.
§
For Cebu Direct Stays, Keith prefers avoiding full host accounts/dashboard early; for features like availability calendars, favor admin-managed/manual updates, private magic-link editing, or iCal import over requiring hosts to create profiles.
§
Keith's scheduled TLDR news briefings: include latest AI updates, use only accessible/working sources; skip problematic ones. Keith prefers data-driven decisions: "formulate an answer based on data. Not assumptions." Always commit before implementing risky changes so rollback is easy.
§
Polymarket /root/polymarket-intel: web :8095, crypto+scanner Docker via gluetun VPN Canada. Midpoint scalp halted. Momentum live-only: 5-share FAK, $0.58–0.75 entry, T+45–85s, 0.05% BTC threshold, hold to resolution. Directional paper-only w/ -40% SL. Polymarket is SOURCE OF TRUTH for all data: use actual fill prices for entry_cost (never assume ask), Gamma API outcomePrices for resolution (trump Chainlink). Funder 0x0A47...c7d8. Deposits excluded from P/L. SQLite LIKE: match '%\"side\": \"UP\"%' (space after colon).
§
Keith's #1 rule: never claim UI fix verified without browser proof (Playwright 390×844). Verify: JS valid, data loaded, table has rows, clicks work (check pointer-events:none on hidden panels), pagination gap ≤10px. Grep/curl are NOT evidence.
§
Keith rejects complex entry-band/price-band filtering for momentum strategy ("these prices always fluctuate"). Prefers simple min/max entry price guards over multi-band analysis. Only use entry price filtering when it directly gates risk/reward (e.g., max entry $0.75 prevents upside-too-small trades), not for statistical "sweet spot" analysis.