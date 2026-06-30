# Paper-trading strategy bot patterns

Use this reference when building or changing read-only/paper-trading bots for prediction markets, copy trading, or similar signal-driven systems.

## Preserve strategy lanes

When the user asks to add a stronger or safer strategy, do not silently replace an existing strategy unless explicitly asked. Implement separate lanes with distinct identifiers so performance can be compared:

- `source` identifies broad origin, e.g. `smart_wallet_copy`, `smart_wallet_consensus`, `research_signal`, `arbitrage_detector`.
- `kind` identifies strategy variant, e.g. `copy_single_high_win_rate_wallet`, `copy_wallet_consensus`.
- Keep separate de-duplication keys per lane when the user wants both to coexist.
- Report clearly which lane opened a paper trade.

## Candidate filters vs strategy filters

Factor shared eligibility checks into a candidate function, then let each strategy decide how to use candidates.

Shared candidate checks often include:

- wallet history quality: closed-position count and win rate
- side/action, usually BUY-only for simple copy strategies
- resolution window
- minimum time before event end
- token/asset exists
- price range
- trade size
- blocked market terms/categories

Strategy-specific checks then apply on top:

- Single copy: one qualifying wallet activity can open a trade.
- Consensus copy: multiple distinct qualifying wallets must buy the same event/token/outcome and meet combined-size rules.
- Research/news: candidate market must match a sourced news item and pass confidence/relevance scoring.

## Status labels should describe system state, not ask for user work

Avoid labels like `closed_expired_needs_manual_resolution` unless a human truly must intervene. Prefer names that describe the bot state:

- `closed_pending_final_result_sync` — event ended, but official settlement/result sync has not confirmed outcome.
- `closed_take_profit` — closed by mark-to-market take-profit rule.
- `closed_stop_loss` — closed by mark-to-market stop-loss rule.
- `closed_won`, `closed_lost`, `closed_void_or_unknown` — use after final settlement sync exists.

Tell the user whether action is required. For paper trading, the expectation should usually be no manual intervention; the bot should sync official results later.

## Verification checklist

After changing strategy logic:

1. Unit-test boundary filters directly (e.g. price just below/at/inside/above range).
2. Test each strategy lane separately with synthetic candidates.
3. Compile/lint the code.
4. Rebuild/restart the service if containerized.
5. Run one scanner/job execution.
6. Query the DB/API to verify status counts, open trades, and config values.
7. Summarize actual results, including when no trade opened and why likely filters blocked it.
