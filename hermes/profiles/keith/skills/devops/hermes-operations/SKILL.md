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

## Backup Workflow

- Decide backup contents: skills, memories, cron, plugins/config minus secrets.
- Use private Git remotes and deploy keys when appropriate.
- Validate ignore rules before first push.
- Schedule pushes with a quiet-success/noisy-failure pattern.
- Test restore assumptions periodically.

## Scheduled Job Audit Workflow

Check all relevant layers before reporting: Hermes cron jobs, OS crontabs, systemd timers/services, app-specific schedulers, and external CI/webhooks when applicable. Include enabled/disabled state and last/next run if available.

## VPS Workflow

- Inspect services, disk, ports, Docker state, and firewall before mutation.
- Keep credentials out of shell history/logs.
- Verify deployed sites with HTTP and, when relevant, browser checks.

## Verification Checklist

- [ ] Target profile/server/service unambiguous.
- [ ] Secrets were not printed or committed.
- [ ] Backup/scheduler/service changes verified by readback.
- [ ] Final report includes exact evidence and remaining risks.
