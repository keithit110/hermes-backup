# Hermes GitHub Backup Pattern

Use this reference when setting up a private GitHub backup for durable Hermes state on a personal VPS.

## Goal

Back up durable Hermes Agent state without committing credentials:

- `SOUL.md`
- `config.yaml`
- `skills/`
- `memories/`
- `cron/`
- `sessions/` excluding request dumps
- `state.db`
- named profiles under `~/.hermes/profiles/<name>/`

Never back up:

- `.env`
- `auth.json`
- `nous_auth.json`
- OAuth/cache/token files
- SSH keys
- logs
- request dumps
- lock/pid/runtime files

## Safe GitHub auth

Prefer a repository deploy key over a pasted token.

Create key on VPS:

```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
ssh-keygen -t ed25519 \
  -f /root/.ssh/hermes_backup_ed25519 \
  -C "hermes-backup@$(hostname)" \
  -N ""
cat /root/.ssh/hermes_backup_ed25519.pub
```

Add the public key in GitHub:

```text
Repo → Settings → Deploy keys → Add deploy key → Allow write access
```

SSH config:

```bash
cat >> /root/.ssh/config <<'EOF'

Host github.com-hermes-backup
  HostName github.com
  User git
  IdentityFile /root/.ssh/hermes_backup_ed25519
  IdentitiesOnly yes
EOF
chmod 600 /root/.ssh/config
```

Remote URL format:

```bash
git remote add origin git@github.com-hermes-backup:OWNER/hermes-backup.git
```

If `origin` already exists, use:

```bash
git remote set-url origin git@github.com-hermes-backup:OWNER/hermes-backup.git
```

Common pitfall: do not leave placeholder text such as `YOUR_GITHUB_USERNAME` in the remote URL.

## Backup script shape

Use a whitelist rebuild model: rebuild the repo working tree from approved paths, then commit and push if changed.

Important behavior:

- Initialize repo if missing.
- Copy default/current profile into `hermes/default/` or equivalent.
- Copy named profiles into `hermes/profiles/<profile>/`.
- Use `rsync` excludes for auth/log/cache/runtime files.
- Commit only when `git diff --cached` is non-empty.
- Push only when `origin` is configured.

## Cron

For daily 00:00 UTC backup:

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HERMES_HOME=/root/.hermes
HERMES_BACKUP_REPO=/root/hermes-backup

0 0 * * * root flock -n /tmp/hermes-github-backup.lock /usr/local/bin/hermes-github-backup >> /var/log/hermes-github-backup.log 2>&1
```

Verify:

```bash
systemctl enable --now cron
systemctl is-active cron
/usr/local/bin/hermes-github-backup
cd /root/hermes-backup
git log --oneline -3
git remote -v
```

## Safety checks before first push

```bash
cd /root/hermes-backup
git ls-files | grep -Ei '(^|/)(\.env|auth\.json|nous_auth\.json|.*token.*|.*secret.*|request_dump_|logs/|cache/|bin/|node/|\.lock$|\.pid$)' || true
git grep -Ei 'api[_-]?key|secret|token|password|private[_-]?key|BEGIN OPENSSH|BEGIN RSA|BEGIN PRIVATE' || true
```

Treat positive matches as a stop sign until reviewed.

## Restore caveat

This backup is for durable state recovery and auditability, not a full bare-metal restore. It intentionally excludes credentials and runtime caches, so a restored instance still needs fresh auth/setup for providers, Telegram bots, GitHub, and external services.
