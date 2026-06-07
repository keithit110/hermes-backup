# Hermes Agent Backup

This private repository is an automated backup of durable Hermes Agent state.

Included for the default profile:
- `hermes/default/SOUL.md`
- `hermes/default/config.yaml`
- `hermes/default/skills/`
- `hermes/default/memories/`
- `hermes/default/cron/`
- `hermes/default/sessions/` excluding request dumps
- `hermes/default/state.db`

Included for named profiles, if they exist:
- `hermes/profiles/<profile-name>/SOUL.md`
- `hermes/profiles/<profile-name>/config.yaml`
- `hermes/profiles/<profile-name>/skills/`
- `hermes/profiles/<profile-name>/memories/`
- `hermes/profiles/<profile-name>/cron/`
- `hermes/profiles/<profile-name>/sessions/` excluding request dumps
- `hermes/profiles/<profile-name>/state.db`

Intentionally excluded:
- `.env`, `auth.json`, `shared/nous_auth.json`, OAuth/token files
- files/paths containing `token` or `secret`
- logs, caches, binaries, runtime locks, request dumps, temp files

Treat this repository as private. Session and memory files can contain personal conversation history.
