---
name: software-development-workflows
description: "Class-level umbrella for planning, spikes, TDD, debugging, code review, and language-specific debugger workflows across software projects."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [software-development, planning, debugging, testing, code-review, tdd]
    related_skills: [github-workflows, hermes-agent]
---

# Software Development Workflows

## Overview

Use this umbrella when a task is about deciding what to build, validating a risky idea, implementing safely, debugging failures, or reviewing code before handoff. The goal is not one narrow ritual; it is the class-level loop that keeps software work grounded:

1. clarify the desired behavior and constraints,
2. create a small executable plan or spike when needed,
3. write or preserve tests before changing behavior,
4. debug from evidence rather than guessing,
5. review the diff for correctness/security/regressions,
6. verify with real commands before reporting completion.

## When to Use

- User asks for an implementation plan, plan mode, or a saved markdown plan.
- A build idea is uncertain enough to need a throwaway spike before committing.
- User asks for tests first, TDD, regression coverage, or safe refactoring.
- Existing code fails and root cause is unknown.
- A code change is ready for pre-commit/pre-PR verification.
- Python or Node.js behavior requires an interactive debugger rather than print-and-guess edits.

## Workflow Selection

| Situation | Use this subsection | Core output |
|---|---|---|
| User explicitly asks to plan only | Planning | `.hermes/plans/<topic>.md` with bite-sized tasks, no implementation |
| Requirements are uncertain or risky | Spike | Disposable experiment + verdict: VALIDATED / PARTIAL / INVALIDATED |
| Behavior should be protected | TDD | RED failing test, GREEN minimal fix, REFACTOR cleanup |
| Failure already exists | Systematic debugging | Repro + root cause + minimal fix + regression test |
| Diff is ready | Code review | Security scan, quality gates, test/lint results, risk notes |
| Need runtime inspection | Debuggers | `pdb`/`debugpy` or `node --inspect`/CDP workflow |

## Planning

Plan mode is for producing an actionable markdown plan without executing it. Include exact files, commands, expected tests, rollback notes, and bite-sized tasks that can each be implemented independently. Do not hide vague mega-tasks under labels like “build auth”; decompose to model, migration, route, test, UI, and verification units.

## Spikes

A spike is a throwaway proof that resolves uncertainty. Keep it small, isolate it from production code, and finish with a verdict:

- **VALIDATED** — evidence supports building the real solution.
- **PARTIAL** — approach works with named caveats.
- **INVALIDATED** — evidence shows it should not be built this way.

Archive or delete throwaway files unless the user asks to keep them.

## Test-Driven Development

Follow RED → GREEN → REFACTOR:

1. Write the smallest failing test that captures the required behavior.
2. Run that specific test and confirm it fails for the expected reason.
3. Implement the minimum production change.
4. Run the specific test, then relevant broader tests.
5. Refactor only after green tests, then run tests again.

Never claim TDD if the RED failure was not observed.

## Systematic Debugging

Debugging starts with evidence:

1. Reproduce consistently.
2. Read exact error messages and logs.
3. Identify the smallest failing boundary.
4. Inspect recent changes and relevant code paths.
5. Fix the root cause, not a symptom.
6. Add or update a regression test.
7. Verify the original failure path is fixed.

## Code Review and Pre-Commit Verification

Before telling the user a change is ready, inspect the diff and run appropriate checks. Look for hardcoded secrets, shell injection, unsafe deserialization, SQL/string interpolation risks, missing tests, backwards-incompatible API changes, and unhandled failure paths. Prefer specific evidence: command, exit code, and result.

## Signal-Driven / Paper-Trading Bots

When changing bots that open simulated trades from signals, preserve existing strategy lanes unless the user explicitly asks to replace them. If adding a safer/stronger strategy such as consensus, keep the original lane separately identified so results can be compared. Use clear status names that describe bot state rather than implying user intervention. See `references/paper-trading-strategy-bots.md` for candidate-filter, strategy-lane, status-label, and verification patterns.

## Debuggers

Use runtime debuggers when static inspection stalls:

- Python: `python -m pdb`, `pytest --pdb`, or `debugpy` for DAP/remote attach.
- Node: `node inspect`, `node --inspect`, `SIGUSR1` attach, or Chrome DevTools Protocol scripts.

Prefer targeted breakpoints around the suspected boundary. Remove temporary breakpoints/imports before finalizing.

## Absorbed Detailed Packages

Detailed original skill packages, including their support files, are stored under `references/absorbed/<old-skill-name>/` when present:

- `references/absorbed/plan/SOURCE_SKILL.md`
- `references/absorbed/spike/SOURCE_SKILL.md`
- `references/absorbed/test-driven-development/SOURCE_SKILL.md`
- `references/absorbed/systematic-debugging/SOURCE_SKILL.md`
- `references/absorbed/requesting-code-review/SOURCE_SKILL.md`
- `references/absorbed/python-debugpy/SOURCE_SKILL.md`
- `references/absorbed/node-inspect-debugger/SOURCE_SKILL.md`

Use those when the umbrella overview is not enough.

## Domain References

- `references/polymarket-paper-trading-safety.md` — use when building or reassessing Polymarket scanners, paper-trading dashboards, or copy-trade logic; includes API surfaces, read-only verification, copy-trade pitfalls, safer paper-copy filters, 5-min BTC crypto engine architecture, and resolution P/L accounting.
- `references/docker-compose-vpn-glutun.md` — route a Docker Compose service through a VPN (gluetun + Surfshark WireGuard) when a downstream API geoblocks the host IP.
- `references/crypto-5m-btc-strategy.md` — use when building or modifying the Polymarket BTC 5-minute paper-trading engine; includes slug-based market discovery, VPN setup with gluetun/Surfshark, WebSocket dynamic subscription, Lane 2 (late directional) and Lane 3 (profit-lock hedge) strategies, and resolution checking with outcomePrices parsing.
- `references/btc-5min-crypto-engine.md` — BTC 5-min paper-trading engine: slug-predicted market discovery (`btc-updown-5m-{epoch}`), dual-WebSocket architecture (Coinbase/Binance + Polymarket Market Channel), Lane 2 late-directional and Lane 3 profit-lock hedge strategies, US geoblock workarounds, and Docker deployment pattern.
- `references/dashboard-ui-pitfalls.md` — use when building or debugging web dashboards with Chart.js, P/L aggregations, status pills, or strategy summary cards; covers canvas height constraints, sum-vs-average P/L bugs, double-multiplication, status formatting, and tab simplification.

## Verification Checklist

- [ ] Picked the correct workflow branch instead of mixing all of them.
- [ ] Used real tool output for tests/debugging/review claims.
- [ ] Preserved user constraints such as “plan only” or “no code changes.”
- [ ] Left the repository cleaner than found, or explicitly explained remaining risks.
