---
name: kanban-workflows
description: "Use when operating Hermes Kanban work: decomposing initiatives, coordinating orchestrator/worker roles, avoiding over-granular tasks, and handling worker edge cases."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [kanban, orchestration, workers, project-management, decomposition]
    related_skills: []
---

# Kanban Workflows

## Overview

This umbrella combines the Kanban orchestrator and worker playbooks. Use it for Hermes Kanban boards, multi-step decomposition, worker assignment, status hygiene, and edge cases encountered while implementing board tasks.

## Role Routing

- **Orchestrator mode**: decompose an initiative, create/sequence cards, assign workers, and keep the board coherent.
- **Worker mode**: execute a single assigned card, report blockers precisely, and avoid expanding scope without updating the board.

## Orchestrator Playbook

1. Clarify the outcome and acceptance criteria from the user’s request or existing board.
2. Decompose into bite-sized cards with explicit deliverables and verification.
3. Sequence cards by dependency; avoid parallelizing tasks that edit the same files/state.
4. Assign workers with self-contained context.
5. Review worker outputs as evidence, not truth; verify side effects.
6. Close or revise cards promptly so the board reflects reality.

## Worker Playbook

1. Read the assigned card and linked context before acting.
2. Identify prerequisites and blockers early.
3. Execute only the card’s scope unless the orchestrator updates the task.
4. Produce verifiable artifacts: paths, diffs, command outputs, URLs, screenshots, or IDs.
5. Mark the card complete only after verification; otherwise report a blocker and the smallest next action.

## Anti-Temptation Rules

- Do not turn every tiny shell command into a separate card.
- Do not keep vague “investigate” cards open after the investigation yields concrete work.
- Do not let workers silently choose incompatible implementation directions.
- Do not mark a card done based only on an agent’s self-report.
- Do not create dependency cycles or parallel cards that fight over the same resource.

## Common Edge Cases

- **Ambiguous ownership**: assign one owner and list reviewers separately.
- **Blocked worker**: convert the blocker into a new card only if it is independently actionable.
- **Scope explosion**: split follow-up work and complete the original card if its acceptance criteria are met.
- **Failed verification**: reopen or revise the card with the exact failing command/output.

## Verification Checklist

- [ ] Every active card has an owner, deliverable, and verification method.
- [ ] Dependencies are explicit.
- [ ] Worker results were independently checked when side effects matter.
- [ ] Completed cards map to actual artifacts or verified outcomes.
