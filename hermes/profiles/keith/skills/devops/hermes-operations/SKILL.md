---
name: hermes-operations
description: "Operate Hermes profiles, gateways, durable state backups, scheduled jobs, and personal VPS deployments safely and reproducibly."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, profiles, gateways, backups, cron, systemd, vps, operations]
    related_skills: [hermes-agent, dockerized-static-webapps]
---

# Hermes Operations

## Overview

Use this umbrella for operational work around Hermes itself and the user's small-server environment: profile creation, gateway processes, durable state backup, scheduled-job audits, systemd/cron checks, VPS hygiene, Dockerized side projects, and secure credential handling.

## When to Use

- Create or troubleshoot separate Hermes profiles and profile-specific gateways.
- Back up Hermes durable state to private Git remotes.
- Audit scheduled jobs across Hermes cron, OS cron, systemd timers, and app schedulers.
- Operate the user's VPS for Hermes, static webapps, revenue sites, or backups.
- Diagnose provider auth, gateway service, or reboot-readiness issues.

## Operational Principles

- Treat profiles as separate state roots; never mix default and named profile files casually.
- Read current config/state before changing services, cron, or secrets.
- Backups must filter secrets and preserve enough state to restore useful operation.
- Scheduled-job counts require checking every scheduler class, not just Hermes cron.
- Prefer reversible changes and keep exact commands in the final report.

## Profile and Gateway Workflow

1. Identify active profile and intended profile.
2. Inspect config, env, gateway service/process, and logs.
3. Apply scoped changes only under the target profile.
4. Restart or reload the relevant service.
5. Verify with `hermes status`, logs, and a real message/tool test when applicable.

## Third-Party Skill Installation Workflow

When installing a non-Hermes GitHub skill into Hermes, port it into the active profile as a Hermes-compatible class-level skill instead of blindly running an upstream assistant-specific installer. Inspect manifests/templates, copy functional scripts/data, adapt paths in `SKILL.md`, preserve provenance under `references/`, then verify with `hermes --profile <profile> skills list`, `skill_view`, and a representative script or `hermes chat -s <skill>` run. See `references/third-party-skill-porting.md`.

## Backup Workflow

- Decide backup contents: skills, memories, cron, plugins/config minus secrets.
- Use private Git remotes and deploy keys when appropriate.
- Validate ignore rules before first push.
- Schedule pushes with a quiet-success/noisy-failure pattern.
- Test restore assumptions periodically.

## Scheduled Job Audit Workflow

Check all relevant layers before reporting: Hermes cron jobs, OS crontabs, systemd timers/services, app-specific schedulers, and external CI/webhooks when applicable. Include enabled/disabled state and last/next run if available.

When a Hermes cron job is failing, hanging, or not messaging the user, inspect both job state and logs. A skipped tick such as `already running — skipping` may not deliver a failure message. For recurring news/TLDR briefings, prefer a deterministic pre-run source script plus a no-agent watchdog over a web-heavy LLM-only cron job. See `references/tldr-news-cron-hardening.md` for the reliable pattern, source sanitization step, and verification checklist.

## VPS Workflow

- Inspect services, disk, ports, Docker state, and firewall before mutation.
- Keep credentials out of shell history/logs.
- Verify deployed sites with HTTP and, when relevant, browser checks.

## Verification Checklist

- [ ] Target profile/server/service unambiguous.
- [ ] Secrets were not printed or committed.
- [ ] Backup/scheduler/service changes verified by readback.
- [ ] Final report includes exact evidence and remaining risks.
