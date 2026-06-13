---
name: marketplace-leadgen-webapps
description: Build Dockerized lead-generation marketplace/directory web apps with SEO pages, admin review workflows, host submissions, and verified deployment.
version: 1.0.0
author: Hermes Agent
license: MIT
tags: [django, docker, marketplace, lead-generation, seo, admin, web-apps]
---

# Marketplace Lead-Gen Web Apps

Use this skill when building an MVP directory/marketplace where suppliers list inventory and buyers contact them directly: rental directories, direct-booking sites, local service directories, host/property lead-gen portals, or similar projects.

This is not for a one-page static landing page. Use it when the app needs listings, admin review, forms, photos, SEO pages, Docker deployment, or future host accounts.

## User/style preference for this class of task

For Keith, be direct and no-BS:

- Challenge unrealistic assumptions, especially claims like “SEO ranking is guaranteed” or “no business registration means no tax/legal obligations.”
- Prefer practical MVP sequencing over polished fantasy architecture.
- Explain tradeoffs briefly, then build/verify.
- If the user dislikes a design, iterate with concrete visual improvements and verify with browser screenshots.

## Recommended MVP architecture

For a small marketplace/directory MVP, prefer:

- **Django** for server-rendered SEO pages, forms, built-in admin, media uploads, and fast CRUD.
- **PostgreSQL** for production-like persistence.
- **Docker Compose** for app + DB isolation.
- **Django admin** for first-stage admin-only publishing.
- **Public host submission form** for suppliers/hosts to submit listings for review.
- **External contact links** for the earliest version (Messenger/Facebook/WhatsApp/phone), unless a real internal messaging product is explicitly required.

Avoid building a full host dashboard, payment processing, or real-time chat before traffic and inventory are proven.

## Core data model

At minimum:

1. `Host`
   - name/display name
   - email/phone
   - Facebook URL
   - Messenger URL (`https://m.me/pageusername` when it is a Page)
   - WhatsApp number
   - verified flag

2. `Listing`
   - title/slug
   - host
   - area/neighborhood
   - property/category type
   - capacity/facts
   - prices or price ranges
   - description/short description
   - nearby landmarks
   - house rules
   - amenities many-to-many
   - featured/published flags

3. `Amenity`
   - standardized checkbox values; avoid free-text amenity chaos.

4. `ListingPhoto`
   - uploaded image and/or external URL
   - caption/sort order

5. `Inquiry`
   - listing
   - name/contact
   - dates/guest count
   - message
   - handled flag
   - **Set `verbose_name_plural = "Inquiries"`**; Django's default would display `Inquirys`.

6. `HostListingSubmission`
   - mirrors listing fields plus host contact info
   - status: new/reviewing/approved/rejected
   - admin notes
   - amenities checkboxes
   - optional photo links

## Host access policy

Start with this safer MVP flow:

```text
host submits property form
→ admin reviews in Django admin
→ admin creates/publishes listing
```

Do not give random hosts broad Django admin access. If host self-service is needed later, build a limited dashboard where a logged-in host can only manage their own listings and where publishing still requires admin approval.

## SEO strategy for local directories

Do not promise rankings. Build pages that can rank over time:

- homepage with clear local positioning
- listing index
- listing detail pages with unique descriptions
- intent landing pages (e.g. airport stays, monthly stays, staycations)
- sitemap.xml
- robots.txt
- semantic titles/meta descriptions
- fast mobile-first server-rendered pages
- internal links from guides/landing pages to listings

Target long-tail high-intent pages before broad competitive terms.

## Design expectations

First-pass admin-style UIs often look dated. For public marketplace pages:

- Use a real design system reference when available (Airbnb-like for property marketplaces; dark premium styles when requested). For Keith's accommodation marketplace work, load `popular-web-designs` and the Airbnb template before redesigning.
- Add a dark-mode toggle when requested and persist it in `localStorage`.
- Use modern typography, big hero copy, rounded cards, subtle shadows, hover effects, and lightweight page-load animations.
- If the user says the site feels outdated, clunky, or lacks "pop," treat that as a design-quality failure: redesign the visual hierarchy, CTA structure, hero, card system, and content sections — do not only tweak colors.
- For mobile marketplace homepages, do not leave core navigation categories only as cards below the hero. Put primary stay/search categories in the top nav via a compact dropdown or hamburger menu. If the hamburger/dropdown already contains those options, do not duplicate the same category cards on the homepage — it creates clutter and confuses the primary guest action.
- Mobile nav must behave like an app menu: close when the user taps outside, presses Escape, or taps a link. Verify the close behavior after implementing the hamburger menu.
- If implementing a hero text carousel inside a full-bleed image, it must be centered on mobile, have visible pagination dots, support touch/pointer swipe left/right in addition to auto-advance, and keep the primary CTA obvious. Do not substitute a separate below-hero card carousel when the reference shows text inside the hero image.
- If the user references a travel screenshot with circled text inside the hero image, implement the text carousel *inside the full-bleed hero image* with centered slide text and dot indicators. Do not misinterpret this as a separate below-hero card carousel.
- Do not style non-clickable labels as button-like pills. If it looks like a button, it should be clickable; otherwise turn it into plain supporting text or remove it.
- Remove prototype/internal wording before presenting as "production-ready". Avoid labels like "MVP", "directory MVP", or test-ish copy in public hero sections; use customer-facing value propositions instead.

- Use scroll and card animations as progressive enhancement: sections can rise/fade in, cards can lift/scale and images can subtly zoom on hover/tap-capable devices, but the content must remain visible without JS and respect `prefers-reduced-motion`.
- If Keith says a property/listing card animation merely appears/disappears or is not special enough, treat subtle opacity-only reveal as insufficient. Preserve components he explicitly approved (for example unit-type/category cards) and make the targeted property card motion visibly distinct: directional entry, slight scale/settle, masked reveal/clip-path, image zoom settle, and tasteful staggering. Verify by scrolling to the card region and capturing the animation in a mobile viewport.
- If Keith pivots from property-card animation to “like Airbnb” sideways browsing, stop improving vertical scroll animation. Remove per-card listing scroll-reveal hooks and turn the homepage featured properties into a native horizontal swipe carousel with `overflow-x:auto`, CSS grid column flow, and `scroll-snap-type:x mandatory`; keep approved unit-type/category animations unchanged. Verify `scrollWidth > clientWidth`, `scrollLeft` changes after manual swipe/scroll, and card animation is `none` in a mobile viewport.
- Remove prototype/internal wording before presenting as "production-ready". Avoid labels like "MVP", "directory MVP", or test-ish copy in public hero sections; use customer-facing value propositions instead.
- Dark-mode toggles should use recognizable icons (prefer inline SVG moon/sun icons) and persist preference in `localStorage`; verify both visual states.
- Avoid hiding essential content behind JavaScript-only scroll reveal. Progressive enhancement is safer: content visible by default, animation layered on top.
- Verify visually with browser screenshots, not just `curl`. Do not tell Keith a frontend change is complete until you have inspected the rendered page and judged whether the result is appealing, coherent, mobile-friendly, and aligned with the requested direction.
- For mobile-first sites, validate at a real mobile viewport before claiming completion. A good default is Playwright Chromium at `393x852` with `is_mobile=True` and `has_touch=True`, plus an actual screenshot review.
- For mobile carousels, prefer native horizontal scrolling/scroll-snap over transform-only fake swipes. Measure active slide/text with `getBoundingClientRect()` and confirm center offset is near 0; also verify `scrollLeft` changes and active dots update.
- Use relevant design skills (`popular-web-designs`, `sketch`, or project-specific references) before major visual changes. Treat design validation as part of the deliverable, not optional polish.
- Check section spacing carefully: page hero/header panels must not touch listing-card grids; keep clear vertical rhythm between boxed sections and cards.
- Avoid overly tight negative letter-spacing on marketplace headings/cards; if text looks compressed or nearly overlapping, switch to a cleaner font stack and relax tracking.
- Do not blindly copy Airbnb's coral/pink palette for accommodation sites; if the user dislikes it, pivot to a distinct brand palette (for Cebu Direct Stays: teal/navy/gold feels more ownable and less derivative).
- Scroll down during verification; do not only inspect the hero section. Sticky nav can obscure content while checking screenshots.
- For Keith's mobile-first web projects, visual/mobile validation is part of the deliverable, not optional polish. Before saying “done,” use a rendered mobile/touch viewport, inspect screenshots, test interactions, and measure DOM geometry for alignment/edge cases.
- Treat “dead space” feedback on mobile hero sections as a conversion/layout bug, not a subjective nit. Tighten the mobile-only vertical rhythm: reduce oversized `min-height`/padding, move headline/lead/CTA groups upward, bring dots/buttons closer to copy, and preserve readability over the image overlay. Verify with a screenshot at the same approximate mobile height the user showed, plus DOM measurements for nav/content/dots/button gaps.
- For hero text carousels inside full-bleed images, prefer native horizontal scrolling + `scroll-snap` on mobile. Validate that the carousel track and active slide span the viewport edge-to-edge; avoid transform-only swipes inside centered wrappers that create clipped left/right edges.
- If Keith dislikes a landing-page hero/background image as low-quality, pixelated, generic, or not emotionally compelling, generate a numbered option set (20 worked well), present a contact sheet, let him choose by number, then apply that exact selected image as a local static asset. Before applying, verify the specific emotional subject he named remains visible after `background-size: cover` cropping on desktop and mobile.
- For property detail pages, use an Airbnb-style compact photo mosaic before listing details, plus a manual swipe lightbox for more photos. Do not vertically dump all photos above the description.
- See `references/cebu-direct-stays-design-iteration.md` for session-derived design/verification notes from the Cebu Direct Stays MVP.
- See `references/cebu-direct-stays-mobile-ui-validation.md` for the mobile validation workflow, carousel fix pattern, and listing-detail photo layout lessons from Keith's feedback.
- See `references/mobile-hero-dead-space-validation.md` for tightening full-bleed mobile hero spacing when Keith flags wasted above-the-fold real estate.
- See `references/property-card-scroll-animation-validation.md` for scoped, visually noticeable property/listing-card scroll animation improvements while preserving approved category/unit-type animations.
- See `references/property-card-horizontal-carousel-validation.md` for replacing vertical featured-property browsing with an Airbnb-style native horizontal swipe carousel and verifying no property-card scroll animation remains.
- See `references/hero-background-image-selection.md` for generating numbered hero-background options, applying the selected image as a local static asset, and verifying subject visibility/readability across desktop and mobile.

## Docker deployment checklist

1. Write project under a clear path (e.g. `/root/<project-name>`).
2. Include:
   - `Dockerfile`
   - `docker-compose.yml`
   - app entrypoint
   - persistent DB volume
   - persistent media volume
   - restart policy
3. Expose a non-conflicting port for MVP preview (e.g. `8080:8000`).
4. Add `/health/` endpoint.
5. Run/rebuild containers.
6. Verify:
   - `docker compose ps`
   - `/health/` returns `ok`
   - homepage loads
   - listing page loads
   - admin redirects to login
   - host submission form loads
   - form POST creates a row
   - admin model labels are spelled correctly
   - dark-mode toggle works if implemented

## Pitfalls

- Airbnb or marketplace pages may block scraping. Seed with partial/placeholder data and make content editable in admin; do not fabricate precise details.
- If using `makemigrations` at container startup with persisted DB and no committed migration history, schema drift can happen. For durable projects, commit migrations or explicitly verify new tables exist after rebuild.
- Do not copy third-party listing copy verbatim for SEO. Rewrite descriptions for uniqueness.
- External Messenger links are easiest; embedded Messenger plugins are painful for multi-host marketplaces because each host may need a Facebook Page/domain setup.

## Reference files

- `references/cebu-direct-stays-mvp.md` — session-derived notes from the Cebu direct-stay directory MVP build, including decisions, verification checks, and follow-up recommendations.
