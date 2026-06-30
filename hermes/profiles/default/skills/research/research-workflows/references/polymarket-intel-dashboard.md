# Polymarket Intelligence Dashboard Lessons

Session-derived guidance for building read-only Polymarket / prediction-market intelligence tools.

## Product shape

Prefer a **read-only scanner + paper-trading dashboard** before any live execution:

1. Market scanner: active/near-resolution markets, bid/ask/spread, liquidity, volume, end date.
2. Arbitrage detector: start with conservative binary YES/NO ask-sum checks only.
3. Smart-wallet monitor: tracked wallet activity as intelligence, not blind copy-trading.
4. Information monitor: official/news/RSS items that may explain market movement.
5. Paper-trading ledger: simulated positions, P/L, close-at-profit rules, invalid-signal states.

## UI/UX lesson

Do not dump cron reports into chat every 30 minutes. For recurring market monitoring, route noisy output into a dashboard with date slicing and local logs. Telegram should be reserved for exceptional alerts or summaries.

The UI must answer these questions immediately:

- What fake/paper positions are currently open?
- What bet/market is each position about?
- Is it real money or simulated? State this plainly.
- What is the current P/L and status?
- What do the tabs/buttons mean?
- What are logs vs signals vs raw market data?

Use human labels:

- `Paper Bets`, not `Paper P/L` only.
- `Possible Arbitrage`, not just `Arbitrage`.
- `Market Watch`, not just `Markets`.
- `System Logs`, not `Cron Logs`.

Add a persistent plain-English explainer at the top of the dashboard.

## Arbitrage pitfall

Do **not** assume a cheap basket of YES outcomes across related markets is true arbitrage. Some events contain multiple binary markets where all YES legs can resolve NO. Only call a multi-outcome basket arbitrage after validating that the outcomes are mutually exclusive **and collectively exhaustive** under the resolution rules.

Safe first implementation:

- Enable binary YES/NO ask-sum checks.
- Disable multi-outcome paper-trades until resolution-rule parsing proves the basket is exhaustive.
- If a signal is later found invalid, mark paper trades as `closed_invalid_signal` rather than pretending it is a loss/win.

## Data retention

For monitoring dashboards, enforce retention explicitly. For Keith's Polymarket dashboard, use 60 days unless told otherwise. Preserve open paper trades until closed; purge stale runs, market rows, wallet activity, info items, and closed paper trades beyond retention.

## Compliance boundary

Keep the tool read-only/paper-only unless the user confirms a compliant venue and account path. Do not help bypass geographic restrictions or VPN-blocked access for international Polymarket.
