---
name: kanban-workflows
description: "Use Hermes Kanban as an orchestration substrate: decompose work, create linked cards, route workers, monitor progress, and summarize outcomes."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [kanban, orchestration, workers, task-management, hermes]
    related_skills: []
---

# Kanban Workflows

## Overview

Use this umbrella when Hermes work should be coordinated through a Kanban board rather than completed inline. It covers both orchestrator and worker behaviors: decomposition, card creation, dependency routing, worker lifecycle, progress heartbeats, blocking, and completion summaries.

## When to Use

- A task splits into multiple independent or dependent work items.
- Different profiles/workers need clear ownership boundaries.
- Progress must survive across sessions or be visible to collaborators.
- The parent agent must coordinate rather than directly perform all work.

## Orchestrator Playbook

1. Understand the goal and acceptance criteria.
2. Sketch the task graph: independent tasks, dependencies, and verification gates.
3. Create cards with specific deliverables, context, and success checks.
4. Assign or route cards to suitable workers.
5. Monitor status, answer blockers, and keep ownership boundaries clean.
6. Close with a synthesized summary and evidence.

## Worker Playbook

1. Claim only cards you can actually work.
2. Read full card context and linked dependencies before acting.
3. Work in the specified workspace/tenant/profile.
4. Send useful heartbeats when progress is non-obvious.
5. Block with a concrete question and what you already tried.
6. Complete with outputs, changed files/URLs/IDs, tests, and residual risks.

## Pitfalls

- Orchestrator doing the work instead of routing it.
- Workers claiming cards they created or cannot access.
- Missing tenant/profile isolation.
- Vague blockers that cannot be answered quickly.
- Completion summaries without verifiable artifacts.

## Verification Checklist

- [ ] Task graph and ownership boundaries are clear.
- [ ] Cards contain enough context for stateless workers.
- [ ] Dependencies and blockers are represented.
- [ ] Completion summaries include evidence and next steps.
