# Hold Zone Datetime Bug — Full Investigation

## Symptom
Hold zone (`SCALP_HOLD_SECONDS=120`) configured correctly, env var set, code in place, but ZERO `[SCALP] HOLD` messages in logs. All positions still selling at TP inside the hold zone window. Positions from 11:10-11:15 PM ET sold at `closed_take_profit` when they should have been held.

## Investigation path

1. **Verified code in container**: Source file has correct hold zone guard (`if seconds_left <= SCALP_HOLD_SECONDS: continue`). Env var present (`SCALP_HOLD_SECONDS=120`).

2. **Added debug logging**: Printed `seconds_left` value inside the datetime computation block. Found: `seconds_left PARSE ERROR: can't subtract offset-naive and offset-aware datetimes`.

3. **Root cause identified**: The error was silently swallowed by `except Exception: pass`. `seconds_left` defaulted to 300, making the hold zone condition `300 <= 120` always false.

4. **Why the subtraction failed**: `now_utc()` = `datetime.now(timezone.utc)` returns **timezone-aware** datetime. `end_date` from SQLite is `"2026-06-29T03:35:00+00:00"` — parsed by `datetime.fromisoformat()` also becomes timezone-aware. The original code tried `end_dt.replace(tzinfo=None) - now` (naive minus aware) which throws TypeError.

5. **Fix**: `seconds_left = max(0, (end_dt - now).total_seconds())` — both timezone-aware after `fromisoformat()`. Tested in container: works.

6. **Stale pycache**: Even after source fix, old `.pyc` bytecode persisted in `__pycache__/`, causing OLD logic to run. Added `RUN rm -rf /app/app/__pycache__` to Dockerfile and `ENV PYTHONDONTWRITEBYTECODE=1`.

## Verified working
After fix: `[SCALP] HOLD #1365 UP — 117s left, holding to resolution (entry=0.550 bid=0.240)` — hold zone fires correctly.

## Key functions involved
- `now_utc()` at line 64 of crypto_engine.py: `return datetime.now(timezone.utc)` 
- `check_scalp_take_profits()` at line 1121: hold zone guard at line ~1168
- `end_date` stored in paper_trades as text: `"2026-06-29T03:35:00+00:00"`

## Verification command
```bash
docker exec polymarket-intel-crypto python3 -c "
from datetime import datetime, timezone
end = datetime.fromisoformat('2026-06-29T03:35:00+00:00')
now = datetime.now(timezone.utc)
print('seconds_left:', (end - now).total_seconds())
"
```
Must return a number (not an error).
