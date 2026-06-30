# Dockerized Django Directory MVP Notes

Session-derived build notes for Cebu Direct Stays and similar local accommodation directories.

## User requirements pattern

Keith's MVP requirements in this class of task:

- Build locally on the VPS first; GitHub can come later.
- Run the product inside Docker containers, not directly on the host.
- Expose an MVP on a VPS port first, commonly `8080`; domain/HTTPS comes after the MVP is visible.
- Start with external Facebook/Messenger/personal-account contact links if that is simpler than embedded Meta chat.
- Provide a listing-upload/editing experience with Airbnb-style amenity checkboxes.
- Seed from the user's owned listings when possible.

## Practical stack

Recommended first build:

```text
/root/cebu-direct-stays/
  docker-compose.yml
  app/
    Dockerfile
    manage.py
    requirements.txt
    entrypoint.sh
    cebu_direct_stays/
    stays/
```

Containers:

- `web`: Django + Gunicorn
- `db`: PostgreSQL

Expose:

```yaml
ports:
  - "8080:8000"
```

Use restart policies:

```yaml
restart: unless-stopped
```

## MVP pages

Public routes:

```text
/
/properties/
/properties/<slug>/
/list-your-property/
/mactan-airport-accommodation/
/cebu-city-condo-daily-rental/
/cebu-weekly-monthly-stays/
/cebu-staycation-condos/
/robots.txt
/sitemap.xml
```

Admin:

```text
/admin/
```

## Django app features

Core admin-managed models:

- Host
- Amenity
- Listing
- ListingPhoto
- Inquiry
- HostApplication

Use Django admin for early listing creation/editing. This gives the user upload forms, checkbox amenities, publish/featured flags, and inquiry review without building a host dashboard too early.

## Contact strategy

For MVP, use both:

1. On-page inquiry form saved to the database.
2. External host contact links:
   - Facebook/Messenger URL
   - WhatsApp deep link
   - phone/SMS if available

Do not require every host to create a Facebook Page or configure the Messenger Customer Chat plugin. That is too much friction for a local multi-host directory.

## Airbnb seeding notes

Airbnb short links may resolve but expose only partial public data. In the Cebu Direct Stays session:

- `ctyscpe13` exposed title and some image URLs publicly.
- Other provided Airbnb URLs mostly returned shell/search UI text or resolved room IDs without listing details.

Good behavior:

- Seed partial data where available.
- Add reasonable placeholders only when explicitly marked as needing verification.
- Keep the original Airbnb URL in the listing admin for reference.
- Ask the user to edit exact guests/beds/baths/rates/photos later in admin.
- Rewrite copy instead of copying Airbnb descriptions verbatim.

## Verification checklist

Before saying the MVP is done, run real checks:

```bash
cd /root/cebu-direct-stays
# Prefer background/daemonized compose startup in Hermes terminal contexts;
# some tool guards classify foreground `docker compose up -d --build` as server startup.
docker compose build
docker compose up -d
docker compose ps
curl -fsS http://127.0.0.1:8080/health/
curl -fsS http://127.0.0.1:8080/
curl -fsS http://127.0.0.1:8080/properties/
curl -fsS http://127.0.0.1:8080/robots.txt
curl -fsS http://127.0.0.1:8080/sitemap.xml
```

Then verify at least one listing page, the admin redirect/login response, database seed counts, and one form flow. If browser tools are available, visually inspect mobile-ish layout and admin availability.

## Generation/validation pattern

When generating a Django app quickly from tool code, add a tight validation loop before rebuilding containers:

1. Run Python syntax checks for generated `.py` files (`python -m py_compile` or Django `manage.py check` inside the container).
2. Watch for newline escaping mistakes in generated Python string literals, especially `robots.txt`, sitemap strings, and seed data with `\n` landmarks.
3. Check template block balance: pages extending `base.html` need `{% endblock %}` for every `{% block %}`; `base.html` itself should not get an extra trailing `{% endblock %}`.
4. Rebuild/recreate the web container after file fixes; do not keep testing an old image.
5. Only mark the job complete after live HTTP checks pass.

## Pitfalls

- If a file creation/build command is blocked by approval, do not claim files were written. Stop and ask for approval/scope clearly.
- Do not put real secrets in committed files. Use local env files or Docker secrets later.
- For a public VPS port, remind the user this is an MVP without domain/HTTPS yet.
- Do not run Django directly with `runserver` on the host when the requirement is Docker-only.
- Do not say "done" after containers merely start; a container can be running while the app returns HTTP 500. Verify pages, seed data, and admin response.
