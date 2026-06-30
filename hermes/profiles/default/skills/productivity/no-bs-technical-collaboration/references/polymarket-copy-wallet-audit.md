# Polymarket Copy Wallet Audit & Filtering

## Core finding: Polymarket wallet scores ≠ copy profitability

A wallet with 100% win rate on Polymarket can be a **net loser** when blindly copied at fixed position sizes (2 shares per trade).

**Why**: Polymarket's `/closed-positions` API returns at most 100 positions. Wallets that make hundreds of trades will have many missing from the API response. If the API returns 100 winners and 0 losers, the computed score is 100% — but the wallet may have 300 losing trades that weren't returned.

Additionally, wallet sizing dynamics: a profitable wallet might bet $10 on high-conviction trades and $1 on speculative bets. When we copy at fixed 2-share sizing, we overweight the speculative bets and underweight the conviction bets, turning a profitable strategy into a losing one.

## Audit methodology

When Keith asks "are these wallets profitable for us?", run this query:

```sql
SELECT
  LOWER(json_extract(details_json, '$.copied_wallet')) as wallet,
  COUNT(*) as trades,
  SUM(CASE WHEN status LIKE 'closed_wo%' OR status LIKE '%take_profit%' THEN 1 ELSE 0 END) as wins,
  SUM(CASE WHEN status LIKE 'closed_lo%' OR status LIKE '%stop_loss%' THEN 1 ELSE 0 END) as losses,
  ROUND(AVG(CASE WHEN pnl_pct > 0 THEN pnl_pct END) * 100, 1) as avg_win_pct,
  ROUND(AVG(CASE WHEN pnl_pct < 0 THEN pnl_pct END) * 100, 1) as avg_loss_pct,
  ROUND(AVG(pnl_pct) * 100, 1) as avg_pnl_pct
FROM paper_trades
WHERE kind = 'copy_single_high_win_rate_wallet'
  AND status LIKE 'closed%'
  AND json_extract(details_json, '$.copied_wallet') IS NOT NULL
GROUP BY wallet
ORDER BY avg_pnl_pct DESC;
```

Key columns: `avg_pnl_pct` (our actual copy P/L) vs Polymarket's score. A wallet with 100% score and -20% avg_pnl_pct is a proven loser — blacklist it.

## Blacklist implementation

Three-layer defense:

1. **Hard blacklist**: `COPY_BLOCKED_WALLETS` env var (comma-separated addresses). Checked at top of `smart_copy_candidate()`.

2. **Performance-based blocking**: `get_blocked_wallets(store)` queries our own `paper_trades` for wallets losing more than `COPY_MAX_DAILY_LOSS_PER_WALLET` (default -$5.00) in the last 24 hours with ≥3 trades.

3. **Tiered thresholds**: Wallets scoring ≥90 get looser filters (`COPY_HIGH_SCORE_MIN_PRICE`/`COPY_HIGH_SCORE_MIN_SIZE`), but these should be **tight enough to filter bad bets** (0.30/15 minimum).

## Entry price vs win rate trade-off

Heavy favorite bettors (entry 0.80+) get high win rates but low returns per win (+10-25%). At -100% losses, they need win rates >80% to be profitable. Most 90+ score wallets on Polymarket don't clear this bar at fixed position sizing.

The profitable wallets are those that consistently bet at 0.50-0.70 range — where wins return +40-100%. Check entry price distribution before trusting a wallet: `SELECT AVG(entry_cost), COUNT(*) FROM paper_trades WHERE kind=... GROUP BY wallet`.

## Trade volume vs quality

After blacklisting proven losers, expect trade volume to drop 90-95%. That's expected and healthy. The trade-off:

| Scenario | Trades/day | Net P/L |
|----------|:---:|:---:|
| All wallets (including losers) | 60+ | Negative |
| Profitable wallets only | 1-5 | Positive |

The solution to low volume is **wallet discovery** (find more wallets like BreakTheBank), not loosening filters on proven losers.
