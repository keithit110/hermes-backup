---
name: hermes-state-backup
description: "Back up durable Hermes Agent state to a private Git repository with secret-safe filtering and scheduled cron pushes."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, backup, github, cron, deploy-key, secrets, vps]
---

# Hermes State Backup

Use this skill when the user wants Hermes Agent settings, skills, memory, sessions, cron jobs, or identity files backed up to GitHub/GitLab/a private Git remote on a VPS.

## Principles

- Prefer a **private repository**.
- Prefer **SSH deploy keys** over pasting tokens into chat.
- Do **not** ask the user to send credentials in chat.
- Stage a **local-only backup repo first**, then add the remote after the user configures credentials manually.
- Back up a **whitelist** of durable Hermes state, not all of `~/.hermes`.
- Verify with real `git remote`, `git status`, `git log`, cron status, and a manual push.
- Be terse with Keith during operational setup unless he asks for detail.

## Recommended backup contents

Include:

- `SOUL.md`
- `config.yaml`
- `skills/`
- `memories/`
- `cron/`
- `sessions/` excluding request dumps
- `state.db`

Exclude:

- `.env`
- `auth.json`
- `nous_auth.json`
- SSH keys
- tokens/secrets by filename
- logs, caches, browser data, temp files
- `request_dump_*`
- locks and pid files

## Workflow

1. Confirm the Hermes home and backup repo path. Defaults commonly used:
   - `HERMES_HOME=/root/.hermes`
   - `BACKUP_REPO=/root/hermes-backup`
2. Create/update a local backup script that copies only whitelisted state into the backup repo.
3. Initialize the local Git repo and commit once locally.
4. Verify no obvious secret filenames are tracked:
   - `git ls-files | grep -Ei '(^|/)(\.env|auth\.json|nous_auth\.json|.*token.*|.*secret.*|request_dump_|logs/|cache/|bin/|node/|\.lock$|\.pid$)' || true`
5. Tell the user to create a private remote repo and set up credentials manually.
6. For GitHub, recommend a write-enabled deploy key; see `references/github-deploy-key-setup.md`.
7. After the user adds the remote, verify:
   - `git remote -v`
   - `git push -u origin main`
8. Install cron only after the remote push works.
9. Run the backup script manually once after installing cron.
10. Report exact paths, schedule, latest commit, and push result.

## Cron pattern

For daily 00:00 UTC root cron:

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HERMES_HOME=/root/.hermes
HERMES_BACKUP_REPO=/root/hermes-backup
HERMES_BACKUP_BRANCH=main

0 0 * * * root flock -n /tmp/hermes-github-backup.lock /usr/local/bin/hermes-github-backup >> /var/log/hermes-github-backup.log 2>&1
```

## Pitfalls

- Do not claim GitHub is configured before `git remote -v` and `git push` prove it.
- If `remote origin already exists`, use `git remote set-url origin <url>` instead of `git remote add origin <url>`.
- If GitHub says `Repository not found`, check placeholder values, repo owner/name, deploy key write access, and whether the remote URL uses the SSH host alias.
- Do not commit broad `~/.hermes` snapshots; use whitelist copying.
- `.gitignore` is not sufficient by itself: it does not remove already-tracked files and does not detect secrets embedded inside allowed files.
- Existing bundled/hub-installed skills may be large; backing up them is okay if the user wants full recoverability, but call out repo size/privacy tradeoffs.

## Verification checklist

- `git remote -v` shows the intended private remote.
- Manual script run prints either `No Hermes backup changes to commit.` or pushes a backup commit.
- `git status --short` is clean after manual run.
- Cron file exists and has the requested UTC schedule.
- Cron service is active.
- GitHub shows the latest commit.
