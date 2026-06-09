# Cebu Direct Stays MVP Notes

Session context: Keith wanted a Dockerized MVP for a Cebu accommodation/direct-stay directory seeded by his Airbnb listings. The business model was a local lead-gen directory: guests contact hosts directly, no platform payment cut, with future monetization via featured listings or listing fees.

## Product decisions

- Site name: **Cebu Direct Stays**.
- Deploy locally on the VPS but run the app only inside Docker.
- Expose MVP on port `8080` until a domain is chosen.
- Use Django + PostgreSQL + Docker Compose instead of a static site because the app needs listing CRUD, photos, forms, amenities checkboxes, and admin review.
- Keep host payments/contact outside the platform for MVP using Facebook/Messenger/WhatsApp links.
- Do not require every host to set up embedded Facebook Messenger; for a multi-host site it creates Meta setup friction and routing problems.
- Use a host submission form first; admin reviews and publishes.

## Airbnb extraction lesson

Airbnb public extraction was inconsistent. Some `/h/<slug>` pages expose titles/photo URLs/details; others return generic Airbnb shell content or limited room pages.

Pattern:

1. Try public extraction/browser access.
2. Capture what is visible.
3. Seed placeholders for blocked details.
4. Make every listing field editable in admin.
5. Rewrite copy for uniqueness rather than copying Airbnb text verbatim.

## MVP entities used

- `Amenity`
- `Host`
- `Listing`
- `ListingPhoto`
- `Inquiry`
- `HostListingSubmission`

Important Django polish: set `Inquiry.Meta.verbose_name_plural = "Inquiries"`, because Django otherwise displays the ugly/misspelled-looking default `Inquirys` in admin.

## Design iteration lesson

The first MVP layout felt outdated. The improved pass used:

- Airbnb-inspired warm visual system
- DM Sans font
- large gradient hero
- rounded cards
- glass panel
- hover lift effects
- scroll reveal animations
- pulsing contact bubble
- dark mode toggle using `data-theme` + `localStorage`

Verification required browser screenshots, including scrolling past the hero. The initial screenshot only showed the hero/blank space; after scrolling, category cards and listing cards were visible.

## Verification checklist from the session

Run checks after rebuild:

```bash
cd /root/cebu-direct-stays
docker compose ps
curl -fsS http://127.0.0.1:8080/health/
curl -fsS http://127.0.0.1:8080/ | grep -Eo '<title>[^<]+'
curl -fsS http://127.0.0.1:8080/static/js/site.js | grep -q 'data-theme-toggle'
curl -fsS http://127.0.0.1:8080/static/css/site.css | grep -q 'data-theme=dark'
curl -fsS http://127.0.0.1:8080/list-your-property/ | grep -q 'Submit property for review'
docker compose exec -T web python manage.py shell -c "from listings.models import Inquiry, HostListingSubmission; print(Inquiry._meta.verbose_name_plural); print(HostListingSubmission._meta.verbose_name_plural)"
```

Also smoke-test the host submission form and clean the test row afterward.

## Schema drift pitfall encountered

A new `HostListingSubmission` model was added after the first migration had already been applied in a persisted DB. Startup showed "No migrations to apply" even though the table did not exist, because migration state and generated migration file contents were out of sync.

Durable fix for future projects: commit stable migration files and do not rely only on runtime `makemigrations`. For quick MVP recovery, create the missing table with Django's schema editor and then verify table existence/counts.

## Recommended next product steps

1. Replace placeholder listing data/photos with accurate host-owned content.
2. Add a limited host dashboard only after manual-review flow proves useful.
3. Add analytics/Search Console once a domain exists.
4. Put Cloudflare + HTTPS in front when moving from IP/port preview to public launch.
5. Add image upload UX for host submissions if non-admin hosts will submit listings regularly.
