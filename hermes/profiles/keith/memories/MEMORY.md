Keith's Hermes backup setup on VPS ffvdgzle.colocrossing.cloud uses local repo /root/hermes-backup with GitHub remote git@github.com-hermes-backup:keithit110/hermes-backup.git. Backup script: /usr/local/bin/hermes-github-backup (normalizes profile-scoped HERMES_HOME back to /root/.hermes and backs up all named profiles). Cron file: /etc/cron.d/hermes-github-backup scheduled daily at 00:00 UTC as root.
§
When Keith asks about persisted memory or backups, distinguish USER.md (user preferences/profile) from MEMORY.md (environment/procedural notes) and verify against /root/hermes-backup plus the GitHub remote before claiming it is backed up.
§
For Cebu Direct Stays, Keith prefers avoiding full host accounts/dashboard early; for features like availability calendars, favor admin-managed/manual updates, private magic-link editing, or iCal import over requiring hosts to create profiles.
§
Keith's scheduled TLDR news briefings should include latest AI updates and use only accessible, working sources; inaccessible/paywalled/problematic sources should be skipped or removed rather than retried.