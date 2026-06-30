# Deployed Site Verification Checklist

Use this when a website/app change passed local tests but the user will judge the running site.

## Failure mode captured

A UI change was implemented and verified against a local Django development server on port 8000. The actual user-facing site was a Docker Compose web container on port 8080. Because the container was not rebuilt/restarted, the live site still served old templates and assets: the old contact panel remained, the sticky button still linked out, and the new map was missing.

## Checklist

1. Identify the real target:
   - `docker compose ps`, process list, reverse proxy config, exposed ports, or the public URL.
   - Distinguish local dev server, container port, staging, and production.
2. Deploy/restart the real target:
   - For Docker Compose source changes: rebuild/recreate the web container, e.g. `docker compose up -d --build web`.
   - Run migrations/static collection as appropriate for the app’s entrypoint/deploy flow.
3. Verify with HTTP from the target URL:
   - Fetch a real page from the served port/domain.
   - Check for removal/addition markers in served HTML.
   - Check static asset cache-buster/version and that JS/CSS URLs return non-404 content.
4. Verify in a browser against the target URL:
   - Navigate to the same URL the user will open.
   - Click the changed UI path.
   - Confirm the visible behavior, DOM state, and console errors.
   - For mobile-impacting changes, use mobile viewport/screenshot or responsive inspection.
5. Only then report done:
   - State exactly which URL/port was verified.
   - Include the behavioral checks, not just test output.

## Good completion language

"I verified the running Docker site on `http://127.0.0.1:8080/...`: the old panel is absent, the map region is present, and clicking the sticky Message host button opens the inquiry modal."
