---
name: personal-vps-operations
description: "Operate a personal VPS for Hermes, Dockerized side projects, backups, and small revenue websites with secure credential handling."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [vps, devops, hermes, docker, backups, security]
    related_skills: [hermes-operations, dockerized-static-webapps]
archived_note: "Reconstructed archive stub after consolidation into hermes-operations; original active package was absorbed."
---

# Personal VPS Operations

Archived in favor of `hermes-operations`.

## User interaction standard

Be direct and operational. Verify before claiming success. Report exact commands, files, URLs, logs, and residual risks.

## Security defaults

- Do not expose tokens, env vars, or deploy keys in output.
- Prefer least-privilege service users and private repos for backups.
- Inspect before mutating firewall, systemd, cron, Docker, or public-facing services.

## Hermes profile operations

Treat Hermes profiles and gateways as profile-scoped state. Confirm the active profile/root before editing config, cron, skills, plugins, or memories.

## GitHub backup workflow for Hermes state

Back up durable Hermes state with secret-safe filtering, private Git remotes, deploy keys, cron/systemd scheduling, and readback verification.

## Dockerized static web apps

For side projects and revenue sites: build reproducibly, expose only intended ports, verify container health, and test the public URL after deploy.

## Airbnb/direct-booking side-project guidance

For direct-booking sites and related lead-gen/marketplace apps, verify real browser/UI behavior and preserve revenue-critical flows such as listing cards, calls to action, inquiry forms, and booking/contact paths.

## Verification before final reply

- [ ] Server/service target identified.
- [ ] State inspected before mutation.
- [ ] Secrets protected.
- [ ] Health check/log/browser verification completed where relevant.
- [ ] Final report includes evidence, not just intended actions.
