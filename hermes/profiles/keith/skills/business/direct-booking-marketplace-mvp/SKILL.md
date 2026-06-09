---
name: direct-booking-marketplace-mvp
description: "Plan and build direct-booking accommodation sites or local stay directories: start with owned listings, capture leads, avoid overbuilding marketplace/payment/chat too early."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [direct-booking, marketplace, accommodation, airbnb, lead-generation, website, mvp]
    created_by: agent
---

# Direct-Booking Marketplace MVP

## When to use

Use this when Keith asks about monetizing accommodation properties, Cebu/Airbnb alternatives, direct-booking websites, host listing directories, payment flows, lead generation, or chat/contact features for stay/rental sites.

## Core framing

A host-paid directory is **not** “Airbnb without commission.” It is a **local accommodation lead-generation directory / marketplace**.

The hard part is not building the website; the hard part is getting guest traffic and proving leads to hosts.

## Recommended strategy

1. Start with owned/controlled inventory first.
   - Keith has Cebu condo units that can seed the directory.
   - Build it like a directory from day one, but launch with his own listings.
2. Add a `List your property` application page.
3. Manually approve early hosts; do not build a full host dashboard too soon.
4. Add 10–30 vetted free listings before charging.
5. Drive traffic through SEO landing pages + Facebook groups + repeat guests.
6. Monetize only after showing traffic/leads.

## Good positioning

Example positioning:

> Find Cebu stays and contact hosts directly — no platform booking fees.

Best niche:

- Cebu City condos
- Mactan / Lapu-Lapu airport stays
- staycation condos
- transient rooms
- weekly/monthly furnished rentals

## MVP site structure

Start with:

```text
/
/properties/
/properties/<listing-slug>/
/mactan-airport-accommodation/
/cebu-city-condo-daily-rental/
/cebu-weekly-monthly-stays/
/cebu-staycation-condos/
/list-your-property/
/contact/
```

## Lead capture over full booking engine

Do not start with payments, instant booking, or Airbnb-grade messaging.

Start with an embedded inquiry/chat-style lead form:

- Listing page shows a floating **Ask Host** button.
- User enters check-in, check-out, guest count, name, contact method, and message.
- Save inquiry in the database.
- Notify host and admin.
- Host replies via guest’s preferred channel.

This gives a chat-like website experience while avoiding full real-time messaging complexity.

For Keith, keep the tone direct/no-BS: if he asks whether SEO, host payments, or Facebook chat will “just work,” say what is realistic and what is not. Do not overpromise rankings, gateway approval, or Facebook automation.

See `references/cebu-direct-stay-session-notes.md` for Keith-specific Cebu strategy notes.
See `references/dockerized-django-directory-mvp.md` for the concrete Docker/Django MVP shape and pitfalls learned while starting Cebu Direct Stays.

## Recommended build stack for MVP

When Keith asks to actually build the product, default to a Dockerized server-rendered app rather than a static site:

- Django web container for SEO-friendly pages, forms, admin, image uploads, and listing management.
- PostgreSQL container for durable listings, hosts, amenities, photos, inquiries, and host applications.
- Docker Compose with restart policies; expose on a VPS port such as `8080` until a domain is ready.
- Django admin first for listing upload/editing and amenities checkboxes; do **not** build a full host dashboard before validating demand.

Minimum models:

- `Host`: name, email, phone, Facebook/Messenger URL, WhatsApp, verification flag.
- `Amenity`: standardized checkbox amenities.
- `Listing`: title, slug, host, area, property type, guests, bedrooms, beds, baths, prices, description, landmarks, rules, source/Airbnb URL, featured/published flags.
- `ListingPhoto`: uploaded image or external image URL.
- `Inquiry`: listing, guest contact info, dates, guest count, message, contacted flag.
- `HostApplication`: public “list your property” submission for manual review.

## Airbnb data extraction caution

If Keith provides his Airbnb links, attempt public extraction, but expect Airbnb to expose only partial data or block details behind JavaScript/login/rate limits. Do not pretend the scrape was complete.

Good pattern:

1. Extract what is public: title, resolved room ID, visible photos, visible location/type snippets.
2. Seed placeholder listing details where Airbnb does not expose enough.
3. Mark seeded details as needing admin verification.
4. Rewrite descriptions for the direct-booking site instead of blindly copying Airbnb text, both for SEO and ownership.

## Payment guidance

For unregistered/side-hustle hosts, recommend manual payment first:

- inquiry/request flow
- manual availability confirmation
- GCash/bank transfer/payment proof upload
- clear cancellation/refund rules

Do not imply that lack of Airbnb business-registration requirements means no tax/legal obligations. Encourage the user to verify local requirements when scaling.

For formal gateway integration later, prefer a gateway that supports GCash/Maya/cards, e.g. PayMongo or Xendit, but expect business/KYC requirements.

## Monetization path

Early:

- free host listings
- featured listings later
- manual listing setup service
- copywriting/photos/SEO setup service

Later:

- featured listing: low monthly fee
- homepage/area-page placement
- host mini-sites
- lead-management services
- optional pay-per-qualified-lead only after trust and tracking are strong

## Pitfalls

- Do not charge hosts before proving traffic/leads.
- Do not require every host to configure Facebook Messenger plugins; many will struggle.
- Do not overbuild host dashboards, real-time chat, payments, reviews, or dispute handling before validating demand.
- Add trust controls early: manual approval, host verification cues, report button, disclaimers, and contact transparency.
- SEO takes months; use Facebook groups and manual outreach for early traffic.
