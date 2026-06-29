# Scanner Debugging — sync_final_results Resolves Nothing

Diagnostic workflow when `sync_final_results` returns 0 despite Gamma API having resolved market data.

## Step 1: Check if resolved data exists

Query Gamma for each stuck event slug:

```bash
for slug in "fifwc-col-prt-2026-06-27" "atp-bergs-humbert-2026-06-27"; do
  curl -s --max-time 10 "https://gamma-api.polymarket.com/events/slug/$slug" | python3 -c "
import sys,json
data=json.load(sys.stdin)
markets=data.get('markets',[])
for m in markets:
    outcomes=m.get('outcomes','[]')
    prices=m.get('outcomePrices','[]')
    try: outcomes=json.loads(outcomes) if isinstance(outcomes,str) else outcomes
    except: pass
    try: prices=json.loads(prices) if isinstance(prices,str) else prices
    except: pass
    for o,p in zip(outcomes,prices):
        if float(p) >= 0.99: print(f'{m.get(\"slug\")}: {o}={p} WINNER')
"
done
```

If some events show winners but `sync_final_results` returns 0, the matching logic is broken.

## Step 2: Replicate the exact logic

Use raw Python (can run from Hermes terminal) to trace through `sync_final_results` for a single stuck trade:

```python
import json, urllib.request
from decimal import Decimal

slug = "fifwc-col-prt-2026-06-27"
# Simulate wanted from trade #404: details["outcome"] = "Yes"
wanted = "yes"

url = f"https://gamma-api.polymarket.com/events/slug/{slug}"
req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
with urllib.request.urlopen(req, timeout=15) as resp:
    data = json.loads(resp.read())

# Replicate parse_jsonish and d() helpers
def parse_jsonish(s):
    if s is None: return []
    if isinstance(s, list): return s
    if isinstance(s, str):
        try: return json.loads(s)
        except: return [s]
    return []
d = Decimal

markets = data.get("markets", [])
for m in markets:
    m_slug = str(m.get("slug") or "").lower()
    prices = [d(x) for x in parse_jsonish(m.get("outcomePrices"))]
    outcomes = [str(x).strip().lower() for x in parse_jsonish(m.get("outcomes"))]
    
    # Check for winner
    winner = next((o for o, p in zip(outcomes, prices) if p >= d("0.99")), None)
    if not winner: continue
    
    # Test slug match
    print(f"m_slug={m_slug}")
    print(f"  'slug in m_slug' = {'fifwc-col-prt-2026-06-27' in m_slug}")
    print(f"  'm_slug in slug' = {m_slug in 'fifwc-col-prt-2026-06-27'}")
    print(f"  winner={winner}, wanted={wanted}, match={winner == wanted}")
```

## Common failure modes

### 1. Reversed slug check (`m_slug in slug` vs `slug in m_slug`)

**Bug**: `m_slug in slug` checks if the market slug (e.g. `fifwc-col-prt-2026-06-27-draw`) is a substring of the event slug (`fifwc-col-prt-2026-06-27`). This is always FALSE — the market slug is longer.

**Fix**: `slug in m_slug` — the event slug is always a prefix of the market slug.

### 2. Wrong-market matching

**Bug**: The `slug in m_slug` prefix check matches ALL sub-markets of an event. If Gamma returns markets in the order [draw, col, prt] and the draw market is resolved first, a trade on "Colombia win / No" gets matched to the draw market (wrong winner → wrong result).

**Fix**: Three-tier match priority:
1. **Exact sub-slug match** — `details_json["slug"]` == `market["slug"]` (most precise, catches the right sub-market)
2. **Event slug prefix** — `event_slug in market_slug` (fallback for old trades without stored sub-slug)
3. **Question text match** — `wanted in question.lower()` (last resort)

### 3. Unresolved event (no 0.99/1.0 prices)

Some events (e.g., postponed tennis matches) never get 0.99/1.0 resolution prices because the match hasn't happened yet. Gamma shows `closed=False`, `active=True`, and the `endDate` may be days after the slug date. Check `event.closed` and `event.endDate` before assuming unresolved = broken. The trade correctly stays in `closed_pending_final_result_sync` until Gamma has resolved data.

### 4. Missing sub-slug in trade details (consensus trades)

**Bug**: Consensus trades store `details_json` with `"event_slug"` but NOT `"slug"` (the sub-market slug). When `sync_final_results` runs, `details.get("slug")` returns `None` → exact match skipped → falls to prefix match → may match wrong market.

**Fix**: Two changes required:
1. `smart_copy_candidate()` must include `"slug": str(a.get("slug") or "")` in the returned candidate dict
2. Consensus trade details must include `"slug": best.get("slug") or best["activity"].get("slug", "")`

Without this, consensus trades are vulnerable to the same wrong-market bug even after the three-tier matching fix.

## Verification after fix

```bash
# Count pending trades
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT status, count(*) FROM paper_trades WHERE status LIKE '%pending%' GROUP BY status;"

# Wait for next scanner run, recheck
# Expected: previously stuck trades move to closed_won/closed_lost
```

## Post-fix audit — check incorrect resolutions (2026-06-28)

After fixing `sync_final_results`, audit ALL trades that were resolved by the buggy code. Query for `closed_won`/`closed_lost` trades where the matching may have been wrong:

```bash
# Check closed_lost trades from recently-resolved events
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT id, opened_at, kind, event_slug, event_title, entry_cost, status, pnl_pct
   FROM paper_trades WHERE status IN ('closed_won','closed_lost')
   AND event_slug LIKE '%fifwc-col-prt%' ORDER BY id;"
```

On 2026-06-28, this audit revealed three incorrect resolutions from the prefix-match bug:

| Trade | Correct | Buggy Code Said | Why |
|-------|---------|----------------|-----|
| #370 (Colombia win/No, 0.71) | **won** (+40.8%) | lost (-100%) | Draw market (slug matches prefix) resolved "yes" — wrong market |
| #406 (Consensus Colombia/No, 0.72) | **won** (+39.3%) | lost (-100%) | Same prefix-match bug |
| #423 (Colombia win/Yes, 0.23) | **lost** (-100%) | won (+334%) | Draw market "yes" ≠ Colombia market "yes" |

**Correction SQL pattern**:
```sql
-- Won but marked lost: set closed_won, current_value=1.0, pnl_pct=(1-entry)/entry
-- Lost but marked won: set closed_lost, current_value=0.0, pnl_pct=-1.0
UPDATE paper_trades SET status='closed_won', current_value=1.0,
  pnl_pct=(1.0-entry_cost)/entry_cost WHERE id=370;
```

Always verify by checking the Gamma resolution data: "Did the specific sub-market this trade bet on actually win?" — not "what won in this event?"
