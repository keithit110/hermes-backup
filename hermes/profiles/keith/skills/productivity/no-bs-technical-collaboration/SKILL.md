---
name: no-bs-technical-collaboration
description: "Work with Keith in a direct, no-BS technical-partner style: concise, grounded, willing to challenge assumptions, and practical before clever."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [collaboration, communication, user-preference, technical-advice]
    created_by: agent
---

# No-BS Technical Collaboration

## When to use

Load this whenever working with Keith on:

- technical troubleshooting, VPS/Docker/Hermes setup, AWS/Aurora/Postgres, DBA work
- business/project strategy, Airbnb/direct-booking ideas, web apps, monetization
- any topic where the user asks for judgment, recommendations, or feasibility

## Core style

- Be straightforward and useful. No fluff, no politeness theater.
- Prefer concise answers, but include enough detail to be operational. Keith explicitly likes this no-BS style; keep it as the default unless he asks for a deeper explanation.
- Witty is fine when natural; clarity wins over jokes.
- Give recommendations, not just options, when there is a clear best path.
- Use labels like **No-BS answer**, **Bottom line**, **Caveat**, or **Recommendation** when helpful.
- When the conversation jumps topics, briefly re-anchor the active topic and continue; do not make session-title confusion worse.

## Pushback rule

Do not blindly agree with Keith. If his assumption, wording, or plan is wrong/risky, say so directly and explain the practical consequence.

## Change-approval rule (HARD — do not skip)

**Propose. Get approval. Then implement.** Never change a parameter, threshold, strategy setting, or any config/env var without Keith's explicit "go ahead" or "let's do it." This includes but is not limited to:

- Docker Compose environment variables (COPY_MIN_PRICE, CRYPTO_MIN_EDGE, etc.)
- Strategy thresholds, filters, or entry/exit rules
- Docker service definitions, network mode, volumes
- Cron job schedules, prompts, or model overrides
- Any `.env`, `.env.vpn`, or credentials-adjacent file

**Correct pattern:**

```
1. Do analysis → present findings
2. State: "Here's my proposal: change X from A → B because [data-backed reason]"
3. Wait for Keith's response
4. Only implement after Keith says yes
```

**Wrong pattern (triggered this rule):**

```
1. See a problem
2. Change the config/docker-compose.yml immediately
3. Tell Keith what you changed after the fact
```

This rule was added after Keith explicitly said: "I don't like the fact that you are changing these parameters without even asking for my approval. Propose to me these changes, and then I will make a decision afterwards."

Examples:

- Correct terminology: Google **AdSense** is for monetizing ads; Google **Ads Keyword Planner** / Trends are for search-volume research.
- Clarify legal/business risk: Airbnb not requiring business registration does not necessarily mean rental income has no tax/legal obligations.
- Correct implementation framing: a host-paid listing site is not “Airbnb without commission”; it is a lead-generation directory / marketplace.

## Uncertainty handling

- If unsure, say what is known, what is unverified, and how to verify.
- Do not invent exact metrics, traffic numbers, command outputs, or credentials state.
- If a tool result is partial or a source is blocked, state that plainly and suggest the next clean verification path.

## Workflow preference

- Keith prefers practical, actionable guidance over theory.
- For setup work, separate what has already been done from what still needs manual action.
- Do not ask for secrets in chat. Prefer deploy keys, local setup instructions, or credentials entered directly on the VPS/provider UI.
- For Telegram/Hermes session management, explain commands in simple operational terms: `/new` starts a session, `/title` names it, `/sessions` lists recent/history, `/resume <id>` attempts to resume.

## Pitfalls

- Do not overstate capability. Say "configured but not yet tested" when true.
- Do not treat a stale session title as current topic truth; sessions can contain multiple topics after the title was set.
- Do not store temporary task progress in memory; save durable preferences and reusable procedures only.
- **Docker --no-cache on source changes.** After modifying any `.py` file in `app/`, `docker compose build` reuses cached COPY layers and deploys stale code. Always use `docker compose build --no-cache web` (or crypto) when the source changed. If the build log shows "CACHED" on the COPY app step, the old code is being served.
- **Crypto engine stuck-trade recovery.** If open trades accumulate without closing, the engine likely crashed on an AttributeError. Check `docker logs polymarket-intel-crypto`. The EngineState class has `btc_bid`/`btc_ask` — there is NO `btc_mid` field. Add a `_close_all_expired_windows()` safety-net that runs BEFORE every evaluation cycle, closing ALL open trades for expired windows. Then back-close stuck trades by querying Gamma API per slug. See `references/polymarket-debugging-patterns.md`.
- **Rebuild crypto container needs VPN.** The crypto service uses `network_mode: "service:gluetun"`. Docker Compose handles the dependency ordering when using `docker compose up -d crypto`.
- **NEVER claim a fix is complete without browser QA.** Keith tests on mobile. Browse the live site, click the relevant tab, verify tables render, verify numbers match. Before saying "fixed," produce actual browser console evidence that the change is live. Docker cache + leftover CSS are recurring failure modes — the build might succeed and still serve broken code.

## Numbers must match — no exceptions

Keith treats divergence between any two numbers that claim to represent the same thing as a hard bug. When building dashboards:

- **Metric card, strategy cards, and chart must use identical formulas.** If the metric card says 3.09%, the chart endpoint must also say 3.09%.
- **Never compute the same aggregate two different ways.** Use one shared function/query, call it everywhere.
- **After any dashboard change, verify all views agree before reporting completion.**
- **BTC directional+hedge pairs are ONE combined bet**, not two independent trades. Return = (total_returned - total_deployed) / total_deployed per event_slug.
- **Closed P/L metric card = closed trades only.** Open/unrealized positions do not contribute.
- **Strategy cards must distinguish realized from unrealized P/L** when a strategy has both open and closed trades.
- **P/L over time chart = same formula as the P/L metric card.** Every single time. Keith will spot any mismatch instantly and considers the entire dashboard broken.

Frustration = first-class signal. If Keith says "I thought we fixed this already" or "what are you not getting," stop, re-read the exact formula used by the source-of-truth (usually the metric card), and make everything else match it exactly. Do not iterate on approximations.

## Multi-process trade corruption (polymarket-specific)

The scanner and crypto engine are separate processes that both write to `paper_trades`. When one process corrupts the other's data, all dashboard numbers break. **The scanner MUST have a blanket `continue` skip at the TOP of its trade-processing loop** for any source it does not own:

```python
for row in rows:
    if row["source"] == "crypto_5m_engine":
        continue  # Engine manages its own trades — scanner must not touch
```

A conditional skip buried inside a nested if-else chain is NOT sufficient — it will be missed when new code paths are added above it. The skip must be the FIRST check in the loop, before any mark-to-market updates, take-profit checks, or expiry closures.

**Symptoms of corruption**: all trades for a source showing `current_value=1.0` with status `closed_resolution_assumed` (a scanner-only status) — impossible outcomes like both sides of a binary market "winning." Fix: query Gamma API for each affected slug to recover the true winner, then back-update all corrupted rows.

See `references/polymarket-debugging-patterns.md` for the full data recovery and strategy analysis workflows. See `references/crypto-engine-architecture.md` for the WebSocket feeds, window math, market discovery, and evaluation loop architecture. See `references/crypto-engine-diagnostics.md` for the diagnostic approach when the engine goes silent (no trades).

## Capital-weighted portfolio return (correct P/L formula)

When tracking portfolio P/L across strategies with variable shares, the correct formula is:

```
return = (total_returned - total_deployed) / total_deployed
total_deployed = sum(entry_cost × shares) for all closed trades
total_returned = sum(current_value × shares) for all closed trades
```

**Never** use a simple average of per-trade P/L percentages. A -100% loss and a +20% win averaging to -40% is mathematically misleading when the loss was $1 and the win was $100.

Three formulas that give three different (wrong) answers:
- Simple average of P/L% → equal-weights every trade, ignores capital sizing
- Capital-weighted without shares → undercounts multi-share positions
- Any formula that doesn't multiply by `shares` → a 3-share $0.84 trade counts as $0.84 instead of $2.52

The metric card, chart, and strategy cards must ALL use the same share-weighted formula. Discrepancy between any two = hard bug.

## Variable shares must be calibrated to actual edge range

The model's edge values are NOT uniformly distributed. If the engine only enters when edge clears a minimum threshold, the edge range of actual trades will be compressed. Thresholds must span that range:

Before calibrating shares, query the actual edge distribution from closed trades. Then set tiers that split the observed range:
- Top third → 3 shares
- Middle third → 2 shares  
- Bottom third → 1 share

Thresholds like `≥20% → 3` are useless when every trade has 43-68% edge. The engine will always max shares and the user loses conviction-gated sizing.

## Chainlink data source (BTC resolution)

Polymarket BTC 5-min markets resolve on **Chainlink BTC/USD** data stream, NOT Binance BTC/USDT. Polymarket provides Chainlink prices free via their RTDS WebSocket:

```
wss://ws-live-data.polymarket.com
Topic: crypto_prices_chainlink
Symbol: btc/usd
Update rate: ~1/sec
```

Binance vs Chainlink divergence caused a real trade loss — Binance showed +0.14% UP, Chainlink resolved DOWN. Switch the engine's BTC price source to Chainlink via Polymarket RTDS (no separate API key needed). Binance should remain as fallback only.

Subscription format on the same Poly WS connection:
```python
{"action": "subscribe", "subscriptions": [
    {"topic": "crypto_prices_chainlink", "type": "*", "filters": '{"symbol":"btc/usd"}'}
]}
```

## API payload size kills dashboard speed

`SELECT *` from large tables with text/blob columns (like `raw_json` with full API responses) produces multi-MB JSON payloads to the browser. The fix:

1. Select only columns the frontend actually renders (check `col` arrays in the JS table builder)
2. Drop `raw_json`, `details_json`, and other blob columns from API responses
3. Cap row limits conservatively (500 not 2000)

A 1.73MB → 137KB payload (92% reduction) turns a sluggish page into an instant load. The browser renders ~50 rows; sending 2,000 with full JSON blobs is wasted bandwidth.

## Data retention cleanup

65-day rolling window via periodic purge in the always-running engine process. Different tables use different timestamp columns:

| Table | Timestamp column |
|-------|-----------------|
| `paper_trades` | `opened_at` |
| `runs` | `ts` |
| `arbitrage_opportunities` | `seen_at` |
| `smart_wallet_signals` | `seen_at` (try/except — table may not exist) |

Run on startup + every ~3600 evaluation cycles (~3 hours). Use separate SQL per table — no generic loop since column names differ.

When Keith asks whether a strategy is working or whether approach X is better than Y, **never theorize — always query the actual trade data from the DB.** Compute both scenarios side by side:

1. Query all relevant closed trades grouped by window/slug
2. Compute deployed, returned, net $, and net % for scenario A
3. Compute the same for scenario B
4. Present a comparison table with exact numbers
5. Let the data drive the recommendation

For crypto pairs: a directional bet and its hedge are ONE combined position per slug. Net P/L = (total_returned - total_deployed) / total_deployed per slug. Individual leg percentages (+614% on a $0.14 hedge) are mathematically true but meaningless in isolation — the pair-level net is what matters.

## "Make sense as a user" test

After fixing dashboard math, verify by asking: would these numbers make sense if I were a user scanning the page cold? Specifically:
- Can I see what I bet on first? (Market column first)
- Do the P/L percentages reconcile? (Individual leg P/L vs combined pair P/L)
- Would a +614% hedge next to a -9.02% overall confuse me? If yes, the display needs context (pair-level net, not per-leg)

## Unrealized trades — ZERO P/L display (hard rule)

Keith has stated this explicitly and will spot violations instantly: **open/unrealized trades must NEVER show a P/L percentage.** Not in the table, not in cards, not anywhere.

- Table P/L column: open rows show `<span class="muted">—</span>`, never a percentage
- Strategy cards with zero closed trades: show "N open" count, never a mark-to-market percentage
- This applies to ALL strategy types: crypto, wallet copy, research, consensus, arbitrage

The formatter for the P/L column must check row status:

```javascript
['P/L','pnl_pct',(v,r)=>r.status==='open'?'<span class="muted">—</span>':pct(v)]
```

Strategy cards must branch on whether closed trades exist:

```javascript
if(v.closed.length > 0){
  // Show realized P/L from closed trades only
} else {
  // Show "N open" — no percentage
}
```

A research_paper_buy card showing "-100.00%" when the trade is still open is a hard bug. A +5% on an open wallet copy trade in the table column is a hard bug. Keith considers these misleading and they undermine trust in every other number on the dashboard.

## Mobile / UI preferences

**CRITICAL: Never change the layout, interaction model, or component type without being explicitly asked.** Keith's strongest frustration signal is unrequested UI changes. If he says a table is clunky, fix the column widths, font sizes, or spacing — do NOT replace it with cards. If he hasn't complained about the layout at all, do NOT touch it.

**Before claiming any fix is complete, QA it in the browser:**
1. Browse the live site at http://localhost:8095/
2. Click the relevant tab(s)
3. Verify with browser console that tables render, data is present, numbers match
4. On mobile: check that no CSS rule accidentally removes essential elements (recurring bug: `@media(max-width:768px)` blocks that strip table borders)
5. Produce console evidence (`JSON.stringify({...})`) before telling Keith it's fixed

**Proven failure patterns (DO NOT repeat):**
- Adding mobile card layouts when the user only asked for cleaner column widths
- Removing mobile CSS blocks and accidentally deleting the table border/display rules
- Docker cached COPY layers serving old code after a source fix (use `--no-cache`)
- Claiming "fixed" based on the code diff alone without actually browsing the live site

Keith tests dashboards on mobile. Key preferences:

- **Tables over card layouts.** Cards that stack vertically force too much scrolling. Prefer compact tables even on mobile — hide low-priority columns via CSS media queries rather than switching to a card format.
- **Sorting and filtering must work on mobile.** No loss of functionality in the mobile view.
- **Collapse secondary content** (explanations, strategy cards) to save vertical space on mobile.
- **Stack filters vertically** on narrow screens.
- When Keith says a UI element is ugly or clunky, he means the design — not the interaction model. Improve the CSS/fonts/colors/spacing; don't replace tables with cards or vice versa.
- **Never change the layout without being asked.** If Keith hasn't complained about the layout, don't touch it. Only fix what he explicitly requested. Unrequested layout changes cause frustration.
