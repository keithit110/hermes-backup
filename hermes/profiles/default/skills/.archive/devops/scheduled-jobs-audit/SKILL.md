---
name: scheduled-jobs-audit
description: "Audit scheduled work across Hermes cron, OS cron, systemd timers, and app-specific schedulers before reporting counts or status."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [cron, scheduler, devops, verification, hermes]
    related_skills: [hermes-agent, hermes-operations]
archived_note: "Reconstructed from prior skill_view transcript after consolidation into hermes-operations."
---

# Scheduled Jobs Audit

Archived in favor of `hermes-operations`.

## When to Use

Use this when the user asks about how many cron jobs / scheduled jobs / reminders they have; whether a backup or recurring task exists; why a job is missing from a scheduler list; or scheduled alerts, backups, timers, watchdogs, and recurring automations.

## Core Rule

Do **not** equate “Hermes cron jobs” with “all scheduled jobs.” Hermes jobs, Linux cron, `/etc/cron.d`, user crontabs, systemd timers, Docker/app schedulers, and CI schedules can all coexist.

When the user asks “how many scheduled/cron jobs do I have,” clarify the scope in the answer or audit all likely scheduler layers before giving a total.

## Audit Procedure

1. **Hermes cron jobs** — use the Hermes cron job manager/listing tool; record job ID, name, schedule, next run, enabled state, and delivery target.
2. **System cron** — check user crontabs, `/etc/crontab`, `/etc/cron.d/*`, and periodic cron directories when relevant.
3. **systemd timers** — check `systemctl list-timers --all` and inspect related units.
4. **App-specific schedulers** — check Docker/app configs and GitHub/CI workflows if suspected.
5. **Verify existence and operation** — inspect scripts, logs, last run evidence, target repo/remote/artifacts, and next scheduled time.

## Reporting Format

Separate scheduler classes clearly:

```text
Total scheduled jobs found: N

Hermes cron jobs: X
- name — schedule — next run — status

System cron jobs: Y
- file/user — schedule — command — last evidence

Systemd timers: Z
- timer — next/last — service
```

If the user expected a job and it is not in Hermes cron, explicitly say where it actually lives instead of implying it is missing.

## Pitfalls

- Counting only Hermes cron.
- Reporting existence without evidence.
- Treating transient scheduler storage as canonical.
- Ambiguous wording: user may say “ground jobs” or “cron jobs” and mean all recurring jobs, not only Hermes-managed jobs.

## Example: Hermes Backup Job Pattern

A daily Hermes backup may be implemented as system cron rather than Hermes cron:

```text
/etc/cron.d/hermes-github-backup
0 0 * * * root flock -n /tmp/hermes-github-backup.lock /usr/local/bin/hermes-github-backup >> /var/log/hermes-github-backup.log 2>&1
```

Verification should include cron file, script, logs, recent backup commit/push, and expected remote URL.
