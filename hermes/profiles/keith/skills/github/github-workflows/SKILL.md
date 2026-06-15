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
