---
name: autonomous-coding-agents
description: "Use when delegating coding, PR review, or repository work to external autonomous coding CLIs such as Claude Code, Codex, or OpenCode. Chooses the right agent mode and supervises results."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [coding-agents, delegation, claude-code, codex, opencode, worktrees]
    related_skills: [github-workflows]
---

# Autonomous Coding Agents

## Overview

This umbrella covers spawning and supervising external autonomous coding CLIs. Use it when the task benefits from an isolated coding agent that can inspect a repo, make edits, run tests, review a PR, or work in a separate worktree. The agent summary is a self-report: always verify file changes, tests, PR URLs, and side effects yourself before telling the user the work succeeded.

## When to Use

- Large coding tasks where an external CLI can work independently.
- Parallel workstreams across issues/PRs/worktrees.
- PR review or implementation where isolated context is useful.
- Tasks where the parent agent should retain a concise orchestration context rather than absorb long logs.

Do not use an external coding agent for a single mechanical shell command, simple file read, or a task that needs live user clarification.

## Agent Selection

| Agent | Best for | Notes |
| --- | --- | --- |
| Claude Code | Feature work, repo-scale edits, PR implementation | Often interactive; handle PTY prompts carefully. |
| Codex | One-shot coding, PR review, batch issue fixing | Strong CLI flags for automation and worktree flows. |
| OpenCode | Feature work and PR review with OpenCode CLI | Resolve binary path first; support interactive and one-shot modes. |

If the user explicitly asks for a specific agent, honor that. Otherwise choose the one installed and configured in the environment; if several are available, prefer the one with the clearest non-interactive mode for the task.

## Generic Workflow

1. **Discover prerequisites**: repo path, branch state, uncommitted changes, test command, and which agent binaries are installed.
2. **Choose isolation**: use a temporary worktree/branch for risky or parallel edits.
3. **Write a self-contained prompt**: include goal, constraints, exact files/commands, expected output, and verification requirements.
4. **Run the agent**: foreground for short bounded jobs; background with notification for long bounded jobs; PTY only for interactive prompts.
5. **Verify independently**: inspect diffs, run tests/lints, read generated files, and fetch PR/CI status if relevant.
6. **Summarize**: report what changed, what passed, what failed, and any follow-up needed.

## Claude Code Notes

- Use when Claude Code is installed and the task is repo-scale implementation or PR work.
- Interactive mode may require PTY handling; do not run an interactive session without `pty=true`.
- If it asks to approve tool use or file edits, respond only after reading the prompt and confirming it matches the user’s scope.

## Codex Notes

- Good for one-shot tasks, PR reviews, batch issue fixing, and worktree-based parallelism.
- Prefer explicit non-interactive flags when available.
- For batch PR reviews, give each run the PR number/URL, repository path, and exact review rubric.

## OpenCode Notes

- Resolve the binary path before invoking; installations may expose `opencode` in different locations.
- Use one-shot mode for bounded edits and background PTY for interactive sessions.
- Capture the final diff and tests rather than trusting the CLI’s success message.

## Worktree Pattern for Parallel Fixes

```bash
git worktree add ../repo-issue-123 -b fix/issue-123
# run selected agent inside ../repo-issue-123
# verify, commit, push, and open PR if requested
```

Keep each agent in its own worktree to avoid cross-contamination. Clean up worktrees only after the user no longer needs the branch.

## Common Pitfalls

1. **Trusting self-reports.** Always verify side effects and tests in the parent session.
2. **Running interactive CLIs without PTY.** This can hang the process.
3. **Letting multiple agents edit the same checkout.** Use separate worktrees.
4. **Prompting too vaguely.** External agents need exact constraints because they lack the parent conversation unless you pass it.
5. **Forgetting user language/tone.** Include output-language requirements in the delegated prompt.

## Verification Checklist

- [ ] Confirmed selected CLI is installed/authenticated or reported the blocker.
- [ ] Passed a self-contained prompt with paths and constraints.
- [ ] Used worktree isolation for risky/parallel edits.
- [ ] Inspected diffs and ran the relevant tests/lints.
- [ ] Verified any external side effects (PR URL, CI status, pushed branch).
