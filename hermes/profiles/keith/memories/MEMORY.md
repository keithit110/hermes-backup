Hermes backup: local /root/hermes-backup, remote keithit110/hermes-backup. Script /usr/local/bin/hermes-github-backup, cron daily 00:00 UTC.
§
For Cebu Direct Stays, Keith prefers avoiding full host accounts/dashboard early; for features like availability calendars, favor admin-managed/manual updates, private magic-link editing, or iCal import over requiring hosts to create profiles.
§
Keith's scheduled TLDR news briefings: include latest AI updates, use only accessible/working sources; skip problematic ones. Keith prefers data-driven decisions: "formulate an answer based on data. Not assumptions." Always commit before implementing risky changes so rollback is easy.
§
Polymarket /root/polymarket-intel: web :8095, crypto WebSocket engine (5 strategies). VPN via gluetun — MUST use non-UK country (Netherlands/CH/Canada); UK geoblocks Polymarket CLOB orders with 403. Diagnose via /logs/live_orders.jsonl. DB has is_live col to separate paper (0) from live (1) trades. UI has split Paper/Live stats strips. Paper-only, 65-day retention, cap-weighted P/L. Binance WS = BTC price, Chainlink via Poly RTDS. Scanner cron 15m. Keith requires approval for param changes.
§
Keith's #1 rule: never claim UI fix verified without browser proof (Playwright 390×844). Verify: JS valid, data loaded, table has rows, clicks work (check pointer-events:none on hidden panels), pagination gap ≤10px. Grep/curl are NOT evidence.
§
nj23adsknml3 wallet: 0x674887d1ac838099a48b629dff53f25b7b87ee08 (confirmed via predicts.guru/checker). Noise farmer/market maker, NOT a scalper: ~$12M volume on ~$34K-$190K PnL, avg bet $5-8, 1,500+ open positions, buys near 0.55, makes spread, holds losers. Strategy is NOT replicable at small scale. Predicts.guru is useful for Polymarket wallet analysis when APIs are geoblocked.
§
SQLite LIKE trap in Polymarket engine: JSON fields in details_json use {\"side\": \"UP\"} (space after colon). LIKE patterns must include the space: '%\"side\": \"UP\"%' works, '%\"side\":\"UP\"%' silently matches nothing. Guards using the wrong pattern appear coded but never fire. Check all LIKE patterns against actual JSON format when debugging silent guard failures.