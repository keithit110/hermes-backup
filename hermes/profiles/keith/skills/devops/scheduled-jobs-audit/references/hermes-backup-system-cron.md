# Session Note: Hermes Backup Was System Cron, Not Hermes Cron

## Context

The user asked to list cron jobs and expected three recurring jobs:

1. BPI loan reminder
2. Sterling Bank Cityscape loan reminder
3. Daily Hermes backup to GitHub at 00:00 UTC

The Hermes cron manager listed only the two reminder jobs. The daily backup existed but was implemented outside Hermes cron as a Linux system cron entry.

## Durable Lesson

When auditing scheduled work, list across scheduler layers before reporting a total. A Hermes `cronjob list` result is only the Hermes scheduler's state, not all recurring jobs on the host.

## Concrete Pattern Observed

Backup job location:

```text
/etc/cron.d/hermes-github-backup
```

Schedule and command:

```cron
0 0 * * * root flock -n /tmp/hermes-github-backup.lock /usr/local/bin/hermes-github-backup >> /var/log/hermes-github-backup.log 2>&1
```

Verification evidence used:

```text
/usr/local/bin/hermes-github-backup
/var/log/hermes-github-backup.log
/root/hermes-backup/.git/logs/HEAD
```

The backup repo remote was an SSH GitHub remote under `/root/hermes-backup`.

## How to Answer Next Time

Say something like:

```text
You have 3 scheduled jobs total, but only 2 are Hermes cron jobs. The backup is a system cron job in /etc/cron.d, so Hermes cron listing does not show it.
```

Then provide grouped counts:

- Hermes cron jobs
- System cron jobs
- systemd timers/app schedulers if relevant
