# Django/Docker marketplace UI modernization pass

Use this reference when modernizing an existing Django marketplace/listing site, especially one served from a Docker image with static assets baked at build time.

## Durable workflow

0. **Always commit a safe revert point first.** Before a major UI overhaul, commit the current working state:
   ```bash
   git add -A && git commit -m "safe revert point before UI overhaul"
   ```
   Then make the overhaul as a separate commit. This way:
   - `git revert <overhaul-commit>` undoes just the overhaul
   - `git revert <revert-point-commit>` goes back to pre-revert-point state
   - The user doesn't need to remember which commits span the overhaul

1. Load the project/domain skill first, then apply UI/UX Pro Max as a visual layer — do not restart the product strategy.\n2. For mobile data table patterns, consult `references/mobile-data-table-cards.md` — the pattern of rendering both a `<table>` (desktop) and stacked cards (mobile) from shared data, with CSS media queries hiding/showing the appropriate layout.\n3. Inspect the actual templates/CSS/JS before editing. For Django sites, expect the key files to be under `templates/`, `static/css/`, and `static/js/`.
3. Make the visual pass concrete, not theoretical:
   - improve type scale and font pairing
   - add glass/gradient/aurora treatment where appropriate
   - add obvious high-intent search/filter chips
   - improve cards, hover states, shadows, focus-visible states, and CTA panels
   - preserve existing information architecture unless the user asked for a restructure
4. Cache-bust static assets whenever editing CSS/JS referenced from templates, e.g. increment `site.css?v=N` and `site.js?v=N`.
5. Run the app tests after visual edits. Existing UI tests may assert details like “only unit-type chips animate” or forbid specific transform names.
6. If the app is Dockerized and the Dockerfile copies `app/` into the image, rebuild and **recreate** the web service; editing host files alone will not update the running container:
   ```bash
   docker compose build web
   docker compose up -d web    # NOT restart — restart reuses old image
   ```
   (When in doubt: `docker compose stop web && docker compose up -d web` forces a fresh container.)
7. Verify with real browser screenshots, not just curl:
   - desktop homepage
   - mobile homepage
   - browse/listing page
   - at least one property-detail page if the visual layer touches shared CSS
8. Check for console JS errors and layout metrics such as horizontal overflow on mobile.

## Pitfall: invisible scroll-reveal content in screenshots

IntersectionObserver reveal patterns can leave elements at `opacity:.001` in full-page screenshots or when the element starts just below the viewport. If a critical navigation/filter strip uses scroll reveal, do not let the non-visible state hide it indefinitely.

Safer pattern:

```css
body.scroll-motion-ready .unit-type-filter.scroll-pop-item:not(.is-visible){
  opacity:1!important;
  transform:none!important;
}
.unit-type-filter.scroll-pop-item.is-visible{
  opacity:1!important;
  animation:categorySweepIn .72s var(--marketplace-ease) both!important;
}
```

This preserves the animation when the observer marks it visible, but keeps the filter UI usable if observer timing or screenshot stitching misses it.

## Pitfall: test suites may encode visual scope

If tests say a motion pass should affect only certain components, do not broaden the animation globally. In one Django listing site, adding `translate3d` anywhere in `site.css` failed a test that asserted property cards did not get the unit-type animation treatment. Fix by using an allowed transform form such as `translate(...)` or by narrowing selectors.

## Verification commands/patterns

```bash
# Django tests
python3 manage.py test

# Dockerized static rebuild/restart
cd /path/to/project
docker compose build web
docker compose up -d web

docker compose ps
docker logs --tail=80 <web-container>

# Basic HTTP checks
curl -fsS -o /dev/null -w 'home:%{http_code}\n' http://127.0.0.1:8080/
curl -fsS -o /dev/null -w 'properties:%{http_code}\n' http://127.0.0.1:8080/properties/
```

Use Playwright/browser screenshots for final validation when available; screenshots are the source of truth for Keith's website UI work.

## Flask raw-string template overhauls

When the entire HTML lives inside a Python raw string (`PAGE = r"""..."""` in `web.py`), use assembly instead of direct editing to avoid corrupting the Python code:

```bash
# 1. Write new HTML to temp file (use write_file)
# 2. Find PAGE boundaries in web.py:
#    Start:  PAGE = r"""  (line ~38)
#    End:    """          (line ~488, before Flask routes)
# 3. Assemble:
cd project && head -37 app/web.py > /tmp/new.py
echo 'PAGE = r"""' >> /tmp/new.py
cat /tmp/new-template.html >> /tmp/new.py
echo '"""' >> /tmp/new.py && echo "" >> /tmp/new.py
tail -n +490 app/web.py >> /tmp/new.py  # line after closing """
# 4. Syntax-check:
python3 -c "compile(open('/tmp/new.py').read(),'web.py','exec')"
# 5. Backup + swap:
cp app/web.py app/web.py.bak && cp /tmp/new.py app/web.py
# 6. Rebuild + recreate:
docker compose build web && docker compose up -d web
```

Pitfall: verify the `tail` start line with `grep -n '^"""$' app/web.py | tail -1` — the closing `"""` must be on its own line.