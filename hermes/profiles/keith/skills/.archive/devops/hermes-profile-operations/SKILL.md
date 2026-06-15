---
name: hermes-profile-operations
description: "Create, configure, troubleshoot, and operate separate Hermes profiles and profile-specific gateways for users/family members."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, profiles, gateways, operations]
    related_skills: [hermes-agent, hermes-operations]
archived_note: "Reconstructed archive stub after consolidation into hermes-operations; original active package was absorbed."
---

# Hermes Profile Operations

Archived in favor of `hermes-operations`.

## Quick workflow

- Identify the active profile and intended target profile before editing files or services.
- Inspect config, environment, gateway process/service, logs, and delivery target.
- Scope changes to the target profile's Hermes home.
- Restart/reload only the relevant gateway or service.
- Verify with status/log readback and a real message/tool test when applicable.

## Provider auth failure triage

Check profile-local env/config first, then provider credentials, then gateway process environment. Do not assume the default profile's credentials apply to a named profile.

## Gateway service handling

Treat profile-specific gateways as separate services/processes. Confirm service name, profile env, logs, and restart behavior before reporting success.

## Reboot readiness for multiple Telegram profiles

Verify each profile has durable config, service units or startup commands, and non-conflicting delivery targets.

## Session/title management

Profile/session state is durable across `/new`; use session search rather than memory for previous task transcripts.

## Verification checklist

- [ ] Profile root identified.
- [ ] Config/env inspected without leaking secrets.
- [ ] Gateway/service logs checked.
- [ ] Restart or change verified by readback.

## Pitfalls

- Mixing default-profile files with named-profile files.
- Restarting the wrong gateway.
- Treating provider auth errors as model failures without checking profile-local credentials.
