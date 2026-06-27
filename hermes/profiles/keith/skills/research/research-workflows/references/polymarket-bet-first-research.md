# Polymarket bet-first research workflow

Session-derived pattern for Keith's Polymarket research/paper-trading system.

## Core correction

Do **not** broadly research news across sports/crypto/politics/etc. before identifying tradable markets. That wastes web/LLM tokens and produces irrelevant context. Use a bet-first pipeline:

1. Discover active Polymarket markets that are tradable and resolve soon (Keith currently prefers <=3 days).
2. Filter to a small candidate set with exact token IDs, end dates, spread/liquidity sanity checks, and non-garbage market types.
3. Generate targeted research queries from those candidate bets.
4. Research only those candidate markets. If there are no candidates, write `IGNORE` and do no web browsing.
5. Only open paper trades when the candidate has exact token/market mapping, source URLs, and a clear explanation of why the information creates edge not already obvious in price.

## Token-conserving candidate generator pattern

Use a deterministic script before any agentic/web research. The script should read the local DB first and print JSON candidates like:

```json
{
  "event_slug": "...",
  "market": "Will France win on 2026-06-26?",
  "outcome": "Yes",
  "token_id": "...",
  "condition_id": "...",
  "end_date": "...",
  "best_bid": 0.62,
  "best_ask": 0.63,
  "spread": 0.01,
  "liquidity": 920564.77,
  "volume_24h": 691546.85,
  "category": "sports",
  "research_query": "official latest Will France win on 2026-06-26? injury lineup score news",
  "polymarket_url": "https://polymarket.com/event/..."
}
```

If the script prints no candidates, the research cron should not call web tools.

## Useful implementation details

- Direct Gamma markets endpoint (`https://gamma-api.polymarket.com/markets`) can be more reliable for fast/live near-resolution market discovery than the events endpoint alone.
- Merge market-shaped pseudo-events into the scanner pipeline so the same orderbook/DB insertion path is reused.
- Reject markets with missing end dates for near-resolution research candidates.
- Keep the candidate count small (e.g. max 5) and prefer tight spreads/liquid markets.
- Do **not** let sports dominate the candidate list. Keith wants news/research-driven opportunities too: politics/geopolitics, macro/regulatory, tech/AI/business, weather/events, and general news-driven markets should surface when tradable near-resolution markets exist.
- Crypto is intentionally excluded from Keith's current research bucket list. Remove crypto discovery queries/hints and reject obvious crypto market titles (`bitcoin`, `btc`, `ethereum`, `eth`, `crypto`) unless he explicitly asks to restore that category.
- Keep the workflow bet-first even for non-sports categories: discover a specific market/token first, then research that candidate only. Do not broadly research politics/news/AI unless a tradable candidate exists.
- If non-sports candidates are too sparse under a universal near-term window, consider category-specific windows rather than broad news scanning: e.g. sports <=3 days, politics/geopolitics or macro/regulatory up to ~7 days when Keith explicitly wants more event-driven research coverage.
- Skip noisy market types unless explicitly requested: exact score, halftime, 1st half, both teams to score, leading at halftime.
- Research outputs should use `IGNORE`, `WATCH`, or `PAPER_BUY`; prefer `WATCH` when uncertain.

## Active paper-trade strategies

Keep the paper-trade paths explicit when recapping or modifying the system:

1. `arbitrage_detector` / `binary_yes_no`: mechanical binary YES/NO ask-sum opportunity. Only paper-trade when buying both sides costs less than $1 by at least the configured `MIN_ARBITRAGE_EDGE`. Multi-outcome baskets stay disabled unless exhaustive resolution rules are proven.
2. `smart_wallet_copy` / `copy_single_high_win_rate_wallet`: one qualified wallet makes a near-term BUY that passes closed-position count, win-rate, price, size, blocked-term, and time-to-resolution filters.
3. `smart_wallet_consensus` / `copy_wallet_consensus`: stronger wallet signal where at least the configured number of qualified wallets buy the same event/token/outcome and meet total-size threshold.
4. Research-agent `PAPER_BUY`: only when a specific candidate market has exact token ID, high-confidence source-backed edge, URLs, and clear reasoning that the information is not already obvious in price. Prefer `WATCH` when uncertain.

## Paper-trading/result labels

Avoid labels that imply user intervention. Use `closed_pending_final_result_sync` instead of labels like `closed_expired_needs_manual_resolution`. The expected behavior is that the bot eventually syncs official outcome data and changes the trade to `closed_won`, `closed_lost`, or `closed_void_or_unknown` when possible.

## Smart-wallet copy strategy note

Keep single-wallet copy and consensus copy as separate strategies for comparison:

- `smart_wallet_copy` / `copy_single_high_win_rate_wallet`: one qualified wallet passes filters.
- `smart_wallet_consensus` / `copy_wallet_consensus`: multiple qualified wallets buy the same event/token/outcome.

Do not remove one strategy when adding the other unless Keith explicitly asks; he wants both available as separate paper-trade sources.
