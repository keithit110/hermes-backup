---
name: ai-coding-agents
description: "Delegate coding, reviews, and long-running implementation work to local AI coding CLIs such as Claude Code, Codex, and OpenCode."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [agents, coding, claude-code, codex, opencode, delegation, worktrees]
    related_skills: [github-pr-workflow, requesting-code-review]
---

# AI Coding Agents

## Overview

Use this umbrella for spawning external AI coding agents as workers. The class-level pattern is the same across Claude Code, Codex, and OpenCode: isolate workspace state, provide a self-contained task prompt, run non-interactively when possible, supervise output, and verify with tests/diffs before reporting success.

## When to Use

- Large implementation or refactor tasks that benefit from an isolated worker.
- Parallel PR reviews or issue fixes in separate worktrees.
- Long-running coding tasks where the parent agent should orchestrate and verify.
- Interactive agent exploration only when one-shot mode is insufficient.

## Universal Workflow

1. **Prepare workspace:** clean git status or create a dedicated worktree/branch.
2. **Write the worker prompt:** include repo path, goal, constraints, tests to run, and expected deliverables.
3. **Prefer one-shot/print mode:** avoid interactive TUI unless the tool needs a multi-turn session.
4. **Run in background for long tasks:** capture logs and use completion notifications.
5. **Verify independently:** inspect diff, run tests, and check for security/quality issues.
6. **Integrate intentionally:** commit, PR, or discard only after verification.

## Tool Selection

- **Claude Code:** strong general coding worker; print mode is preferred. Interactive PTY requires robust dialog handling for trust and permission prompts.
- **Codex CLI:** useful for one-shot implementation, PR review, and batch worktree tasks; background mode fits long jobs.
- **OpenCode:** useful for feature work and PR reviews; resolve the binary path explicitly and know the TUI keybindings before interactive use.

## Parallel Worktree Pattern

- Create one worktree per issue/PR/task.
- Give each worker a bounded task and test command.
- Never let multiple agents mutate the same branch concurrently.
- After completion, review each diff yourself and run tests from that worktree.

## Common Pitfalls

- Treating a worker's self-report as verified fact.
- Starting interactive agents without PTY/tmux readiness.
- Forgetting first-run trust or permission dialogs.
- Sharing credentials or broad secrets in prompts.
- Letting workers create huge unreviewed changes without a diff gate.

## Verification Checklist

- [ ] Worker prompt was self-contained.
- [ ] Workspace isolation was explicit.
- [ ] Agent output was captured.
- [ ] Diff was reviewed by the parent agent.
- [ ] Tests or relevant checks were run after the worker finished.
