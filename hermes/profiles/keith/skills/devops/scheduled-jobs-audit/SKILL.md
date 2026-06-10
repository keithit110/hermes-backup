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
    related_skills: [hermes-agent]
---

# Scheduled Jobs Audit

## When to Use

Use this when the user asks about:

- how many cron jobs / scheduled jobs / reminders they have
- whether a backup or recurring task exists
- why a job is missing from a scheduler list
- scheduled alerts, backups, timers, watchdogs, or recurring automations

## Core Rule

Do **not** equate “Hermes cron jobs” with “all scheduled jobs.” Hermes jobs, Linux cron, `/etc/cron.d`, user crontabs, systemd timers, Docker/app schedulers, and CI schedules can all coexist.

When the user asks “how many scheduled/cron jobs do I have,” clarify the scope in the answer or audit all likely scheduler layers before giving a total.

## Audit Procedure

1. **Hermes cron jobs**
   - Use the Hermes cron job manager/listing tool.
   - Record job ID, name, schedule, next run, enabled state, and delivery target.

2. **System cron**
   - Check user crontabs, usually root and relevant service users.
   - Check `/etc/crontab` and `/etc/cron.d/*`.
   - Check `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly`, `/etc/cron.monthly` when relevant.

3. **systemd timers**
   - Check `systemctl list-timers --all` for timer-backed jobs.
   - Inspect related `.timer`/`.service` units if a relevant name appears.

4. **App-specific schedulers**
   - For Dockerized apps, check Compose services and app config if the job may live inside a container.
   - For GitHub/CI scheduled jobs, check workflow files if repo automation is suspected.

5. **Verify the job exists and works**
   - Inspect the command/script path.
   - Check logs, last run timestamp, last successful artifact/commit/output, and next scheduled time.
   - For backups, verify the target repo/remote and recent commit/push, not only that cron exists.

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

If the user previously expected a job and it is not in Hermes cron, explicitly say where it actually lives instead of implying it is missing.

## Pitfalls

- **Pitfall: Counting only Hermes cron.** If a backup was installed in `/etc/cron.d`, Hermes `cronjob list` will not show it.
- **Pitfall: Reporting existence without evidence.** For backups, include recent log lines, commit hashes, timestamps, or artifact paths.
- **Pitfall: Treating transient scheduler storage as canonical.** Check the actual file/service/remote that performs the work.
- **Pitfall: Ambiguous wording.** User may say “ground jobs” or “cron jobs” and mean all recurring jobs, not only Hermes-managed jobs.

## Example: Hermes Backup Job Pattern

A daily Hermes backup may be implemented as a system cron rather than Hermes cron:

```text
/etc/cron.d/hermes-github-backup
0 0 * * * root flock -n /tmp/hermes-github-backup.lock /usr/local/bin/hermes-github-backup >> /var/log/hermes-github-backup.log 2>&1
```

See `references/hermes-backup-system-cron.md` for the session-specific pattern where a missing-looking Hermes backup job was actually present as Linux system cron.

Verification should include:

- cron file exists
- backup script exists
- log shows recent success
- backup repo has recent commit/push
- remote URL is the expected repository
