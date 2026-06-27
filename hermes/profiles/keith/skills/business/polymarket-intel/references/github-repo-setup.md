# GitHub Repository Setup for Polymarket Intel

## Repo info

- **Remote**: `git@github.com-polymarket:keithit110/Polymarket-hermes-project.git`
- **SSH host alias**: `github.com-polymarket` → `github.com` using `/root/.ssh/polymarket_ed25519`
- **Key label**: `polymarket-intel-hermes`

## Deploy key setup (on VPS)

```bash
ssh-keygen -t ed25519 \
  -C "polymarket-intel-hermes" \
  -f /root/.ssh/polymarket_ed25519 \
  -N ""
cat /root/.ssh/polymarket_ed25519.pub
```

Then on GitHub: repo → Settings → Deploy keys → Add deploy key. Paste public key, **enable write access**. Title: `Hermes VPS`.

SSH config entry (`/root/.ssh/config`):

```
Host github.com-polymarket
  HostName github.com
  User git
  IdentityFile /root/.ssh/polymarket_ed25519
  IdentitiesOnly yes
```

## Git init

```bash
cd /root/polymarket-intel
git init
git branch -M main
git remote add origin git@github.com-polymarket:keithit110/Polymarket-hermes-project.git
```

## .gitignore (must exclude all secrets)

```
# Secrets — never commit
.env
.env.*
*.pem
*_ed25519

# Database files
data/*.sqlite
data/*.db
logs/*.db

# Python
__pycache__/
*.pyc
*.pyo

# Logs
logs/

# IDE
.vscode/
.idea/
```

## Secrets audit before every push

Before committing, verify no secrets leaked:

```bash
git diff --cached --name-only  # see staged files
git diff --cached | grep -inE '(sk-|pk-|private.key|WIREGUARD|password|token|0x[a-fA-F0-9]{64})' || echo "CLEAN"
```

Also verify excluded files aren't tracked:

```bash
git ls-files --cached | grep -i 'env' || echo "No env files tracked"
git ls-files --cached | grep -i 'data/' || echo "No data files tracked"
```

## What gets committed

Only source code and configs — no secrets, no DBs, no logs:

```
.dockerignore
.gitignore
Dockerfile
README.md
app/crypto_engine.py
app/main.py
app/web.py
docker-compose.yml
requirements.txt
run_once.sh
scripts/research_candidates.py
```

## Push workflow

```bash
cd /root/polymarket-intel
git add -A
git status
# Run secrets audit above
git commit -m "descriptive message"
git push origin main
```
