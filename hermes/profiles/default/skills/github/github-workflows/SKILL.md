---
name: github-workflows
description: "Operate GitHub repositories end-to-end: authentication, repo management, issues, PR lifecycle, CI, code review, and codebase inspection."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [github, gh, git, pull-requests, issues, review, repositories, ci]
    related_skills: [requesting-code-review, test-driven-development]
---

# GitHub Workflows

## Overview

Use this umbrella for GitHub-backed development operations. It combines auth setup, repository management, issue triage, PR lifecycle, review, CI troubleshooting, releases, and codebase inspection. Prefer `gh` when available, fall back to git/REST only when needed, and verify every remote side effect.

## When to Use

- Configure GitHub auth for git or `gh`.
- Clone, create, fork, sync, or inspect repositories.
- Create/triage/label/assign issues.
- Branch, commit, push, open, update, review, merge PRs.
- Diagnose CI failures and review diffs.
- Count or characterize codebases before planning work.

## Auth First

1. Detect current state: `gh auth status`, git remotes, SSH keys, credential helpers.
2. Choose the least invasive path: existing `gh`, HTTPS token, or SSH key.
3. Verify with both `gh` and git operations when relevant.
4. Never print tokens; use stdin/env-safe methods.

## Repository Operations

- Extract owner/repo from `git remote -v` before GitHub commands.
- For forks, track upstream and sync intentionally.
- For releases and settings, read existing state before mutation.

## Issues

- Search existing issues before creating duplicates.
- Use structured bodies for bug reports and feature requests.
- Apply labels/assignees/milestones only when the repo convention is clear.

## Pull Request Lifecycle

1. Check clean working tree and branch.
2. Make focused commits with conventional messages when applicable.
3. Push branch and create PR with a body that explains tests and risk.
4. Monitor CI and address failures.
5. Merge only when policy and user intent allow it.

## Code Review

- Review local diffs before push and PR diffs before remote comments.
- Separate critical correctness/security findings from style suggestions.
- Use inline comments only when they add value; otherwise provide a concise review summary.

## Codebase Inspection

- Use LOC/language tools such as `pygount` for quick sizing.
- Exclude generated/vendor/build directories.
- Interpret counts as planning inputs, not quality signals.

## Verification Checklist

- [ ] Auth and repo target verified.
- [ ] Remote side effects confirmed with `gh`/git readback.
- [ ] PR/issue URLs captured when created.
- [ ] CI status checked when relevant.
- [ ] Diff and tests reviewed before claiming success.

## Secrets-Safe Repo Setup

When initializing a new repo from an existing project directory that contains secrets (`.env` files, API keys, databases, private keys):

### 1. Create a deploy key (not a personal SSH key)

```bash
ssh-keygen -t ed25519 -C "project-name-hermes" -f /root/.ssh/project_ed25519 -N ""
cat /root/.ssh/project_ed25519.pub
```

Add to GitHub repo Settings → Deploy keys → **Allow write access** if pushing is needed.

### 2. SSH config alias

```
Host github.com-project
  HostName github.com
  User git
  IdentityFile /root/.ssh/project_ed25519
  IdentitiesOnly yes
```

### 3. .gitignore MUST exclude all secrets

Minimal secrets exclusions:

```
.env
.env.*
*.pem
*_ed25519
*.sqlite
*.db
logs/
__pycache__/
*.pyc
```

### 4. Secrets audit before first push

```bash
# See what's staged
git diff --cached --name-only

# Scan for secrets in staged content
git diff --cached | grep -inE '(sk-|pk-|private.key|WIREGUARD|password|token|0x[a-fA-F0-9]{64})' || echo "CLEAN"

# Verify excluded files aren't tracked
git ls-files --cached | grep -i 'env' || echo "No env files tracked"
```

### 5. Init and push

```bash
git init && git branch -M main
git remote add origin git@github.com-project:OWNER/REPO.git
git add -A && git status
# Run audit above
git commit -m "Initial commit"
git push origin main
```
