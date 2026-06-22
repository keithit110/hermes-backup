# Deployed web UI verification checklist

Use this when a web/UI change is made in a project that has a separate running target (Docker, staging, production, reverse proxy, staticfiles, CDN) in addition to the local source tree.

## Why

A local test server or Django `TestCase` can pass while the user-facing site still serves the old container image, old staticfiles, or an un-restarted process. Do not report UI work as done until the same target the user will inspect shows the new behavior.

## Checklist

1. Identify the actual user-facing target:
   - Docker published port (`docker compose ps`, `docker ps`)
   - reverse proxy URL / staging URL / production URL
   - whether static assets are collected/bundled separately
2. Run automated checks first:
   - narrow failing-then-passing tests
   - full suite
   - framework checks (`python manage.py check`, lint/build as applicable)
3. Deploy/restart the actual target:
   - Django Docker example: `docker compose up -d --build web`
   - wait for health endpoint or container readiness
   - apply migrations in the same runtime/database that serves the site
4. Verify served HTML/assets from the actual target:
   - `curl` the exact route the user will inspect
   - assert old markers are gone and new markers are present
   - confirm cache-busted CSS/JS version or built asset hash changed when JS/CSS changed
5. Browser-verify the actual target:
   - navigate to the live/running URL, not a local dev substitute
   - click changed buttons/links/modals
   - inspect map/iframe/embedded widgets visually or via DOM
   - check console errors
6. Only then summarize with the exact checks run and what they proved.

## Common pitfalls

- Testing `runserver :8000` while the real site is a Docker container on `:8080`.
- Running `collectstatic` locally but not rebuilding/restarting the container that serves `/static/`.
- Verifying HTML contains a modal but not clicking the sticky button that opens it.
- Verifying a link exists but not checking device-specific URL switching logic (e.g. Google Maps vs Apple Maps).

## Good completion evidence

Report concrete evidence, for example:

```text
Running target: http://127.0.0.1:8080/list-your-property/
Tests: Ran 5 tests — OK
Django check: no issues
Container: rebuilt/restarted web
Live HTML: exact_property_address=True, old_contact_panel=False
Browser: clicked Message host; modal opened; console errors=[]
```
