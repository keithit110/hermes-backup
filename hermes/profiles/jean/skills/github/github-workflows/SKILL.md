---
name: github-workflows
description: "Use when working with GitHub repositories end-to-end: authentication, repo management, codebase inspection, issues, branches, PRs, reviews, CI, releases, and merge workflows."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [github, gh-cli, git, pull-requests, issues, code-review, ci, repositories]
    related_skills: [autonomous-coding-agents, requesting-code-review]
---

# GitHub Workflows

## Overview

This umbrella covers GitHub repository operations as a class: setting up authentication, inspecting a codebase, managing repositories, filing/triaging issues, creating PRs, reviewing code, tracking CI, and merging/releasing. Prefer `gh` CLI plus `git` for authenticated workflows; use REST API calls only when `gh` lacks the needed capability.

## Routing Decision

1. **Cannot access GitHub / push / clone** → Authentication subsection.
2. **Need repository metrics or language counts** → Codebase inspection subsection.
3. **Need clone/create/fork/remotes/releases** → Repository management subsection.
4. **Need issue create/triage/labels/assignment** → Issues subsection.
5. **Need branch/commit/push/open PR/CI/merge** → PR lifecycle subsection.
6. **Need review local changes or a PR** → Code review subsection.

## Authentication

### Detection flow

```bash
gh auth status || true
git remote -v
git config --get user.name || true
git config --get user.email || true
ssh -T git@github.com || true
```

Prefer the least invasive fix: existing `gh` auth, existing SSH key, HTTPS token, then new auth setup. Avoid `sudo` for per-user credential fixes.

If a workflow needs a reusable `gh` environment helper, see `scripts/gh-env.sh`.

## Codebase Inspection

Use `pygount` or equivalent line-count tooling when the user asks for LOC, language split, generated-code ratios, or repository size summaries.

Common pattern:

```bash
pygount --format=summary --folders-to-skip .git,node_modules,dist,build,vendor .
```

Interpret counts carefully: generated files, vendored dependencies, lockfiles, and minified assets can dominate totals.

## Repository Management

Common operations:

```bash
gh repo clone OWNER/REPO
gh repo create OWNER/REPO --private --source=. --remote=origin --push
gh repo fork OWNER/REPO --clone
git remote -v
gh release list
```

For API details that are easier via REST, see `references/github-api-cheatsheet.md`.

## Issues

Common operations:

```bash
gh issue list --state open
gh issue view 123 --comments
gh issue create --title "Title" --body-file issue.md --label bug
gh issue edit 123 --add-label triage --assign @me
```

Use templates for repeatable issue bodies:

- `templates/bug-report.md`
- `templates/feature-request.md`

## Pull Request Lifecycle

1. Inspect branch and uncommitted work.
2. Create a focused branch.
3. Make commits with clear messages.
4. Push and create a PR.
5. Monitor CI.
6. Address review comments.
7. Merge using the repository’s preferred strategy.

Common commands:

```bash
git status --short
git checkout -b feature/my-change
git add -A && git commit -m "feat: describe change"
git push -u origin HEAD
gh pr create --fill
gh pr checks --watch
gh pr merge --squash --delete-branch
```

Use `templates/pr-body-feature.md` or `templates/pr-body-bugfix.md` for PR descriptions. See `references/ci-troubleshooting.md` and `references/conventional-commits.md` for recurring details.

## Code Review

For local review, inspect the diff before running tools:

```bash
git diff --stat
git diff --check
git diff --cached
```

For PR review:

```bash
gh pr view <number> --json title,body,author,headRefName,baseRefName,files
 gh pr diff <number>
```

Review for correctness, regressions, security/privacy, tests, backward compatibility, and maintainability. Prefer concrete file/line findings over generic advice. Use `references/review-output-template.md` for structured output.

## Support Files

This umbrella preserves templates/scripts/references from the absorbed GitHub skills:

- `scripts/gh-env.sh`
- `templates/bug-report.md`
- `templates/feature-request.md`
- `templates/pr-body-feature.md`
- `templates/pr-body-bugfix.md`
- `references/review-output-template.md`
- `references/ci-troubleshooting.md`
- `references/conventional-commits.md`
- `references/github-api-cheatsheet.md`

## Common Pitfalls

1. **Skipping auth discovery.** Check `gh auth status`, remotes, and SSH before changing credentials.
2. **Opening huge unfocused PRs.** Keep branches scoped and PR bodies explicit.
3. **Trusting CI from memory.** Fetch live `gh pr checks` or Actions status.
4. **Reviewing only summaries.** Read diffs and changed files directly.
5. **Counting vendored/generated code as product LOC.** Exclude common generated directories.

## Verification Checklist

- [ ] Live GitHub state checked with `gh`/`git` before acting.
- [ ] Authentication method is understood and minimally changed.
- [ ] For PRs/issues, final URL or number captured.
- [ ] For review, findings are tied to concrete files/lines where possible.
- [ ] CI/release/merge state verified after side effects.
