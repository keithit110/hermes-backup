# Host calendar magic links and iCal lockout pattern

Use this for early direct-booking accommodation MVPs where hosts need to manage availability but a full host account/dashboard is premature.

## Recommended pattern

- Create one private calendar-edit URL per approved live listing.
- Use a long random token in the URL, but store only a hash of the token in the database.
- Ensure the token is unique and revocable.
- Admin must have a regenerate action for compromised links.
- Regenerating should revoke old active tokens and notify/email the host with the new link.
- Scope each token to exactly one listing; do not let a token enumerate or edit other listings.
- Treat the URL like a password: never expose it in public pages, logs, screenshots, or listing cards.

## Django model shape

Typical entities:

- `CalendarAccess`: listing FK, token hash, active flag, created/revoked timestamps, last-used metadata if needed.
- `CalendarFeed`: listing FK, provider label, iCal URL, active flag, last sync/error metadata.
- `AvailabilityBlock`: listing FK, start/end dates, source/manual-vs-ical, notes, timestamps.

## Approval flow

When admin approves a public host submission:

1. Create or attach the `Host`.
2. Create the live `Listing`.
3. Mark the submission approved and link it to the live listing.
4. Generate the listing's `CalendarAccess` token.
5. Email the private magic link to the host.

Email delivery should be backed by real SMTP/provider config before claiming production-ready delivery. Console email backend is fine for local/dev verification only.

## iCal source-of-truth rule

If a listing has an active iCal feed from Airbnb, Booking.com, Agoda, etc.:

- Do not allow manual blocking/unblocking in the host magic-link UI.
- Show a clear warning on the calendar page:
  - "This calendar is linked through iCal; blocking and unblocking dates should be managed in your Airbnb listing or the linked booking platform."
- Explain that iCal is import/sync-only for this site; the site should not pretend it can write back to Airbnb.

## Verification checklist

- Token page loads for the active token.
- Old token returns 404/forbidden after regeneration.
- Token for listing A cannot edit listing B.
- Manual block/unblock works when no active iCal feed exists.
- Manual block/unblock controls disappear when an active iCal feed exists.
- iCal warning appears when an active feed exists.
- Admin regeneration creates a new token and deactivates/revokes previous active links.
- Tests cover approval flow, token uniqueness/revocation, and iCal lockout behavior.
