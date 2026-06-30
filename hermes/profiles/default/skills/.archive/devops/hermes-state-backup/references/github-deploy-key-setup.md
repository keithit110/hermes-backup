# GitHub deploy-key setup for Hermes backups

Use this when the user will configure GitHub credentials manually and should not paste secrets into chat.

## Create private GitHub repo

Recommended settings:

- Visibility: Private
- License: None / No license
- README: Do not initialize if a local repo already exists
- `.gitignore`: Do not initialize if a local repo already exists

Licenses are about legal reuse, not cost. For a private backup repo, no license is usually best.

## Generate a dedicated deploy key on the VPS

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

`Repo → Settings → Deploy keys → Add deploy key`

- Title: `Hermes VPS Backup`
- Key: paste `.pub` content
- Allow write access: checked

## SSH alias

If the key path above was used, this block can be used as-is:

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

Only modify `IdentityFile` if a different key path was used.

Test:

```bash
ssh -T git@github.com-hermes-backup
```

A GitHub message saying shell access is not provided is expected.

## Add or fix remote

If no origin exists:

```bash
cd /root/hermes-backup
git remote add origin git@github.com-hermes-backup:GITHUB_OWNER/hermes-backup.git
```

If `remote origin already exists`:

```bash
git remote set-url origin git@github.com-hermes-backup:GITHUB_OWNER/hermes-backup.git
```

Then:

```bash
git remote -v
git push -u origin main
```

Replace `GITHUB_OWNER` and repo name. Do not leave placeholders literal.

## Troubleshooting

- `remote origin already exists`: use `git remote set-url origin ...`.
- `Repository not found`: wrong owner/repo name, repo missing/private without key access, deploy key not added, or write access not checked.
- `Permission denied (publickey)`: SSH alias/key path is wrong or deploy key was not added.
- Push denied: deploy key exists but write access was not checked.
