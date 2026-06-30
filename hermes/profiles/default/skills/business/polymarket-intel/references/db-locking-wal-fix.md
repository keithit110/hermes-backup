# SQLite WAL mode for concurrent access

The crypto engine and web dashboard both write to `polymarket_intel.sqlite`. Without WAL mode, concurrent writes cause `sqlite3.OperationalError: database is locked`.

## Fix (applied 2026-06-28)

Enable WAL mode on every connection:

```python
# In web.py conn() helper:
c = sqlite3.connect(DB, timeout=10)
c.execute("PRAGMA journal_mode=WAL")
c.execute("PRAGMA busy_timeout=5000")

# In crypto_engine.py, after each sqlite3.connect():
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA busy_timeout=5000")
```

WAL mode allows concurrent readers + one writer without blocking reads. `busy_timeout=5000` makes the writer wait up to 5 seconds instead of failing immediately.

## Verification

```bash
sqlite3 data/polymarket_intel.sqlite "PRAGMA journal_mode"
# should return: wal
```

## Symptoms when missing

```
[SAFETY ERROR] database is locked
[EVAL ERROR] database is locked
```

These errors cascade — when the engine can't write to the DB, trades don't get recorded, positions don't close, and the dashboard shows stale data.
