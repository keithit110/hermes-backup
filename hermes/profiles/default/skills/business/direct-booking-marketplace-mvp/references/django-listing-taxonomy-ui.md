# Django listing taxonomy + Airbnb-style unit type UI

Use this when extending the Cebu/direct-booking Django MVP with property categories or browse filters.

## Durable pattern

Keep two separate concepts:

- `unit_type`: broad guest-facing category, similar to Airbnb browse categories.
  - Suggested values: `condo`, `apartment`, `house`, `private_room`, `shared_room`.
  - Display labels: Condo, Apartment, House, Private room, Shared room.
  - Good for homepage/browse icon filters and quick listing-card badges.
- `property_type` or subtype/layout: more specific listing detail.
  - Examples: Studio condo, 1-bedroom condo, Condo, Room, House.
  - Good for detail pages and host/admin precision.

Do not overload one field for both. Guests need simple category filters; admin/host workflows need layout precision.

## UI placement

- Public submission form: add `unit_type` near area/address and before the more specific layout/subtype field.
- Homepage: show a horizontal icon/category strip below the hero and above featured listings.
- Browse page: show the same strip as filters, with active styling for `?unit_type=<value>`.
- Listing cards: show a compact badge with icon + broad unit type.
- Listing detail: show a pill combining broad unit type + subtype/layout near the guest/bed/bath facts.

## Implementation checklist

1. Add choices/constants in `listings/models.py`, including a small icon map.
2. Add `unit_type` to both `Listing` and `HostListingSubmission`.
3. Add migration and run it in the deployed container/environment.
4. Add the field to `HostListingSubmissionForm` labels/help text.
5. Add the field to Django admin `list_display`, `list_filter`, and host-submission fieldsets.
6. Pass unit type options into shared template context.
7. Add filtering in the listings view for `request.GET['unit_type']`.
8. Add CSS cache-buster bump when static CSS changes.
9. Verify with tests and browser against the actual running Docker/reverse-proxy target.

## Test shape

Add Django `TestCase` coverage for:

- submission form includes required `unit_type` and expected choices;
- valid host submission saves `unit_type`;
- detail page displays `unit-type-pill`, broad type, and subtype;
- homepage shows `unit-type-strip`/`unit-type-filter` and all categories;
- listing cards show `unit-type-card-label` and `unit-type-icon`.

## Pitfall

Avoid adding only a backend field. For this class of feature, it is incomplete unless the form, admin, homepage, browse page, cards, detail page, tests, migration, deployed container, and browser-visible UI all agree.