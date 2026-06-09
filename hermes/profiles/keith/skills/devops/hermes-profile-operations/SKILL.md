---
name: hermes-profile-operations
description: Create, configure, troubleshoot, and operate separate Hermes profiles and profile-specific gateways for users/family members.
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [hermes, profiles, gateway, telegram, troubleshooting, systemd]
    created_by: agent
---

# Hermes Profile Operations

Use this when creating or troubleshooting separate Hermes Agent profiles for different users, especially profile-specific Telegram gateways on Keith's VPS.

## Quick workflow

1. **Inspect profiles first**
   ```bash
   hermes profile list
   hermes profile show <name>
   ```

2. **Prefer lowercase profile names**
   - Use names like `jean`, not `Jean`.
   - Profile paths normalize under `~/.hermes/profiles/<name>/`, but aliases and commands are cleaner with lowercase.

3. **Use the actual global profile flag supported by the installed CLI**
   - Check `hermes --help` if unsure.
   - On Keith's current install, use:
     ```bash
     hermes -p <profile> status
     hermes -p <profile> chat -q 'Reply with exactly: OK' --quiet
     ```
   - Do not assume `--profile` works in every installed version.
   - The profile flag is global and belongs before the subcommand.

4. **Create a profile from a working profile when appropriate**
   ```bash
   hermes profile create <profile> --clone-from keith
   ```
   If the user needs clean memories/sessions but a working setup, inspect what clone mode actually copied before promising isolation.

5. **Configure gateway only after the model/provider works**
   ```bash
   hermes -p <profile> status
   hermes -p <profile> chat -q 'Reply with exactly: OK' --quiet
   ```
   If the chat smoke test fails, fix model/provider/auth first.

## Provider auth failure triage

When the gateway says provider authentication failed, do not stop at auth. Check whether the profile has a model configured:

```bash
hermes -p <profile> status
hermes -p <profile> config
```

A common bad state is:

```yaml
model: ''
```

That can surface in gateway logs as:

```text
Primary provider auth failed: No inference provider configured.
```

Fix by setting a known-good model/provider for that profile. Example matching Keith's working OpenAI Codex setup:

```bash
hermes -p <profile> config set model.default gpt-5.5
hermes -p <profile> config set model.provider openai-codex
hermes -p <profile> config set model.base_url https://chatgpt.com/backend-api/codex
hermes -p <profile> chat -q 'Reply with exactly: OK' --quiet
```

Only after that passes, restart the profile gateway.

## Gateway service handling

From inside a running gateway session, `hermes gateway restart` may be refused to prevent restart loops. On Keith's VPS, profile gateways may be systemd services named like:

```bash
systemctl list-units --type=service --all | grep -i hermes
systemctl restart hermes-gateway-<profile>.service
systemctl status hermes-gateway-<profile>.service --no-pager -l
```

Verify logs after restart:

```bash
tail -80 ~/.hermes/profiles/<profile>/logs/gateway.log
```

Look for:

```text
Active profile: <profile>
Connected to Telegram (polling mode)
Gateway running with 1 platform(s)
```

## Reboot readiness for multiple Telegram profiles

When Keith asks whether Telegram bots/profiles will survive a VPS reboot, verify **systemd enablement**, not just that gateway processes are currently running.

Use:

```bash
hermes profile list
systemctl list-unit-files | grep -i hermes || true
systemctl list-units --type=service --all | grep -i hermes || true
for s in hermes-gateway-keith.service hermes-gateway-jean.service; do
  printf '%s: active=%s enabled=%s\n' "$s" "$(systemctl is-active "$s" 2>/dev/null || true)" "$(systemctl is-enabled "$s" 2>/dev/null || true)"
done
ls -l /etc/systemd/system/multi-user.target.wants/hermes-gateway-*.service 2>/dev/null || true
```

A profile gateway is reboot-safe when its profile-specific service is both:

```text
active=active
enabled=enabled
```

and it has a symlink under:

```text
/etc/systemd/system/multi-user.target.wants/
```

It is normal for the legacy/default `hermes-gateway.service` to be disabled/inactive when named profile services are used. If it only has a stale failed state, `systemctl reset-failed hermes-gateway.service` is a cleanup step, not a functional fix.

## Session/title management

Telegram `/sessions` may show only a limited recent list and session titles can be stale/misleading after the user changes topics inside the same session. Do not treat a title like `wifey new profile` as proof of the current topic.

Useful commands/patterns:

```text
/title <clear topic name>
/new
/resume <session_id-or-title>
```

From the VPS CLI, use a high limit for older sessions:

```bash
hermes -p keith sessions list --limit 1000
hermes -p keith sessions browse
```

If a long session ID appears wrapped in terminal output, reconstruct the full ID before claiming it is missing.

## Verification checklist

Before telling Keith it is fixed:

- `hermes -p <profile> status` shows a model and provider.
- `hermes -p <profile> chat -q 'Reply with exactly: OK' --quiet` succeeds.
- The profile gateway process or service is running.
- For reboot survival, profile-specific systemd service is `active` and `enabled`.
- Gateway logs show the correct active profile and Telegram connected.

## Pitfalls

- Screenshots or copied commands may contain an em dash `—` instead of two hyphens `--`; correct it before diagnosing deeper.
- Do not confuse provider auth failure with missing model configuration. The gateway may report it under the auth-failure path even when the real issue is `model: ''`.
- Do not restart all Hermes gateways if only one profile is broken; restart the profile-specific service.
- Do not over-trust prompt-time environment hints if live command output contradicts them; verify with real process/config/status checks and correct the record explicitly.
