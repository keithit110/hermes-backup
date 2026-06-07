# Hermes Agent Backup

This private repository is an automated backup of durable Hermes Agent state.

Included:
- `hermes/SOUL.md`
- `hermes/config.yaml`
- `hermes/skills/`
- `hermes/memories/`
- `hermes/cron/`
- `hermes/sessions/` excluding request dumps
- `hermes/state.db`

Intentionally excluded:
- `.env`, `auth.json`, `shared/nous_auth.json`, OAuth/token files
- logs, caches, binaries, runtime locks, request dumps, temp files

Treat this repository as private. Session and memory files can contain personal conversation history.
