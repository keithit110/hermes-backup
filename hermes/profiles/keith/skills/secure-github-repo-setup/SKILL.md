---
name: secure-github-repo-setup
description: Initialize GitHub repos with deploy keys, .gitignore, and zero-secrets verification
category: github
---

# Secure GitHub Repository Setup

Use when: initializing a new GitHub repo for a project where secrets must never be committed.

## Steps

1. Read `skills_list` and find the `github-workflows` skill for gh CLI auth setup if not already done

2. **Generate a dedicated SSH key** (one per repo, not reused):
   ```bash
   ssh-keygen -t ed25519 -C "project-name-hermes" -f ~/.ssh/project_ed25519 -N ""
   ```

3. **Add SSH config entry**:
   ```bash
   cat >> ~/.ssh/config << 'EOF'

   Host github.com-projectname
     HostName github.com
     User git
     IdentityFile ~/.ssh/project_ed25519
     IdentitiesOnly yes
   EOF
   ```

4. **Give the user the public key** to add as a GitHub deploy key:
   ```bash
   cat ~/.ssh/project_ed25519.pub
   ```
   Instruct them to go to `https://github.com/OWNER/REPO/settings/keys` → Add deploy key → Title: "Hermes VPS" → Paste key → ✅ Allow write access

5. **Create `.gitignore` BEFORE `git init`** — critical to prevent accidental staging:
   ```
   # Secrets
   .env
   .env.*
   *.pem

   # Database
   data/*.sqlite
   data/*.db

   # Python
   __pycache__/
   *.pyc

   # Logs
   logs/
   ```

6. **Verify staged files** with `git add -A && git status` — confirm zero secrets before committing
1. `git add -A && git status` — verify staged files
2. **Secrets audit**: confirm NO `.env*`, `*.pem`, `*_ed25519`, `data/*.sqlite`, or `logs/` files in staging
3. Commit and push:
   ```bash
   git remote add origin git@github.com-projectname:OWNER/REPO.git
   git push -u origin main
   ```
4. **Post-push verification**: `git ls-tree --name-only HEAD` — confirm only intended files are on remote. If any secret file appears, it's in git history and must be removed urgently.

## Pitfalls
- `.gitignore` must be created BEFORE `git add` — once a secret is committed, it stays in history even if removed later
- Always verify with `git status` before committing
- Use `git ls-tree --name-only HEAD` after push to confirm only intended files are on remote
- **Never commit `.env.vpn`**: contains WireGuard private key. Blocked by `.env.*` in `.gitignore`
