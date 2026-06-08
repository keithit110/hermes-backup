# Keith VPS Hermes backup notes

Session-specific details from Keith's setup. Use as a reference when maintaining or verifying his backup, but re-check live state before acting.

## Known layout

- Active profile became `keith`.
- Profile home shape: `/root/.hermes/profiles/keith/`.
- Local backup repo: `/root/hermes-backup`.
- Backup script: `/usr/local/bin/hermes-github-backup`.
- System cron file: `/etc/cron.d/hermes-github-backup`.
- Requested schedule: daily at `00:00 UTC` (`0 0 * * *`).
- GitHub remote shape used: `git@github.com-hermes-backup:keithit110/hermes-backup.git`.

## Workflow lessons

1. Build the local backup repo first; do not assume GitHub credentials exist.
2. Tell Keith not to paste credentials in chat. Use a GitHub deploy key created on the VPS.
3. If the user accidentally used placeholder `YOUR_GITHUB_USERNAME`, fix with `git remote set-url origin ...`.
4. Verify the remote and push before installing/enabling cron.
5. Keep the backup script profile-aware so future profiles such as `wife` are included automatically under `hermes/profiles/<name>/`.

## Sensitive data posture

Keep excluding `.env`, `auth.json`, `nous_auth.json`, SSH keys, token/secret-named files, logs, caches, locks, pids, and request dumps. Remind the user that `.gitignore` does not detect secrets embedded inside allowed files such as sessions, memory, config, or SQLite state; the repo must remain private.
