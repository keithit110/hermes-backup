# DB Locking — Zombie Process Kill + Prevention

## Symptom

Engine produces zero scalp entries, zero TP closes, zero fills for hours. Log shows:

```
sqlite3.OperationalError: database is locked
[EVAL ERROR] database is locked
```

Every `evaluate_and_act()` cycle crashes. All positions ride to resolution as -100% losses.

## Root Cause

A zombie Python process from a previous `docker compose up` holds a write lock on SQLite. Docker containers restart creates a new process, but the old one may survive if the container wasn't fully stopped.

## Fix

```bash
# 1. Find what's holding the lock
fuser /root/polymarket-intel/data/polymarket_intel.sqlite

# 2. Kill the zombie
kill <pid>

# 3. Restart the engine
docker restart polymarket-intel-crypto
```

## Prevention — get_db() helper

All DB connections in crypto_engine.py now use:

```python
def get_db() -> sqlite3.Connection:
    c = sqlite3.connect(DB, timeout=10)
    c.execute("PRAGMA journal_mode=WAL")
    c.execute("PRAGMA busy_timeout=5000")
    return c
```

Never raw `sqlite3.connect(DB)`. Always `get_db()`.

## Verification

```bash
# Check no lock errors in recent logs
docker logs polymarket-intel-crypto --tail 50 | grep -c "database is locked"
# Must return 0

# Check engine is producing entries
docker logs polymarket-intel-crypto --tail 20 | grep "SCALP OPEN"
