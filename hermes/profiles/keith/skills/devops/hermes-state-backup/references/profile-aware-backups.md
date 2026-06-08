# Profile-aware Hermes backup notes

Use this reference when adapting a Hermes backup script for multiple people/profiles on one VPS.

## Profile layout

Common paths:

- Root/default Hermes home: `/root/.hermes`
- Named profiles: `/root/.hermes/profiles/<profile-name>/`

A Git backup repo should preserve profile boundaries:

```text
hermes/default/
hermes/profiles/keith/
hermes/profiles/wife/
```

## Copy policy

For each profile/home, copy only durable state:

- `SOUL.md`
- `config.yaml`
- `skills/`
- `memories/`
- `cron/`
- `sessions/` excluding `request_dump_*`
- `state.db`

Exclude:

- `.env`
- `auth.json`
- `nous_auth.json`
- SSH keys
- token/secret-looking filenames
- logs, caches, browser state, temp files, locks, pid files

## Privacy/security notes

- A private repo is still required: sessions/memory may contain personal conversation history.
- `.gitignore` is not enough by itself; use whitelist copying and inspect `git ls-files` before first push.
- If a secret is embedded inside an otherwise-allowed file like `config.yaml`, `state.db`, memories, sessions, or skills, filename filters will not catch it.
- Do not ask the user to paste GitHub credentials in chat. Prefer deploy keys configured manually on the VPS/GitHub UI.

## Verification

Before claiming success:

```bash
cd /root/hermes-backup
git remote -v
git ls-files | grep -Ei '(^|/)(\.env|auth\.json|nous_auth\.json|.*token.*|.*secret.*|request_dump_|logs/|cache/|bin/|node/|\.lock$|\.pid$)' || true
git push -u origin main
git status --short
```

For daily 00:00 UTC cron, verify:

```bash
cat /etc/cron.d/hermes-github-backup
systemctl is-active cron
```
