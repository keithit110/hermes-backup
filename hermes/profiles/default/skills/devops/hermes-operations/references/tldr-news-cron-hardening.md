# TLDR news cron hardening pattern

Use this when a recurring news briefing cron job is failing, hanging, or silently skipping runs.

## Durable failure modes observed

- A cron run can become effectively stuck; later ticks log `already running — skipping` instead of producing a failed output. That skip may not notify the user.
- LLM-driven jobs that do many live `web_search` / `web_extract` calls are fragile: source blocking, tool-provider config differences, rate limits, large context, and extraction timeouts can turn a simple briefing into a long-running job.
- Generic news homepages/section pages can be blocked or paywalled. Prefer deterministic accessible feeds or source-specific APIs where available.
- Prompt-injection scanner can block assembled prompts if external feed text contains invisible/control Unicode. Sanitize external source text before injecting it into cron prompts.

## Reliable pattern

1. **Diagnose scheduler state first**
   - `hermes --profile <profile> cron status`
   - `hermes --profile <profile> cron list --all`
   - Inspect `~/.hermes/profiles/<profile>/cron/output/<job_id>/` for the newest output file.
   - Grep gateway/agent logs for the job ID and `already running`, `failed`, `delivered`, `web_search`, `web_extract`, `429`, and `Connection error`.

2. **Move source collection into a pre-run script**
   - Write a deterministic script under the profile, e.g. `~/.hermes/profiles/<profile>/scripts/news_brief_sources.py`.
   - Script fetches known-accessible RSS/source feeds with normal HTTP requests.
   - Script prints compact source snippets plus source-health lines.
   - Script exits nonzero only if every source section is empty.

3. **Make the LLM summarize only the source packet**
   - Attach the script to the cron job via `script`.
   - Remove web-heavy tool access from the job where possible; the LLM should not need live web tools for the briefing.
   - Prompt should say: use only script output, do not call tools, do not invent facts, summarize source health.

4. **Remove inaccessible sources from primary source list**
   - If a source URL is blocked/paywalled/403/404 from the cron environment, remove it from the deterministic source script or mark it as excluded.
   - Keep excluded-source notes in script output so the user sees why a source is absent.

5. **Sanitize external text**
   - Strip HTML, normalize whitespace, and remove Unicode category `C` control/invisible characters except normal newline/tab before injecting into the prompt.

6. **Add a no-agent watchdog for silent skips**
   - Script-only cron (`no_agent=True`) checks TLDR job status every 30 minutes.
   - It prints nothing when healthy.
   - It prints an alert only when a watched job is overdue, skipped/stuck, disabled, failed, or has a delivery error.
   - Store a tiny state file so the same failure is not re-alerted every tick.

## Verification checklist

- Source script runs locally and produces non-empty USA/geopolitics/AI sections.
- Source script output contains no invisible/control Unicode.
- Manual `cron run <job_id>` completes with `last_status: ok`.
- Output file contains an actual briefing, not just a tooling apology.
- Logs show `completed successfully` and `delivered to telegram` or the relevant target.
- Watchdog manual run is silent when healthy and its cron record shows `last_status: ok`.

## User-facing expectation

When fixing recurring briefings for Keith, do not stop at “scheduled” or “updated.” Rerun the job, verify the output file, verify delivery log/status, and add/verify watchdog coverage so future failures are not silent.
