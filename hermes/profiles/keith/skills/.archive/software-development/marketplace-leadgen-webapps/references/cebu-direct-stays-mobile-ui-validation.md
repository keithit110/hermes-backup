# Cebu Direct Stays — Mobile UI Validation Lessons

Session-derived notes for marketplace/directory frontend work, especially Keith's Cebu Direct Stays project.

## What went wrong

Several iterations changed CSS/JS and passed basic health checks, but still looked wrong on the user's phone:

- Hero text carousel looked centered in code but was visually shifted right on mobile.
- Swipe carousel used transform-style movement inside a centered wrapper, causing the left edge to be clipped during swipe.
- Homepage duplicated stay-type category cards that were already inside the hamburger menu, creating clutter.
- Hamburger/dark-mode controls were misaligned on mobile.
- Listing detail pages dumped too many photos vertically before the description, burying the actual listing content.

## Required validation pattern before saying “done”

For mobile-first web UI changes:

1. Load relevant design skills first (`popular-web-designs`, `sketch`, this marketplace skill, etc.).
2. Run the app and verify `/health/`, but do not stop there.
3. Use a real/touch mobile viewport, e.g. Playwright or browser emulation around common phone sizes such as `393x852`.
4. Capture/inspect a mobile screenshot, not just desktop.
5. Measure DOM geometry for the risky elements:
   - text center offset vs viewport center should be near `0`
   - carousel/track left should be `0` and right should equal viewport width when intended full-bleed
   - active slide left/right should align with the viewport, not a centered inner wrapper
6. Test interactions:
   - hamburger open/close
   - click/tap outside closes mobile menu
   - dark mode toggle remains aligned
   - carousel dot click works
   - touch/drag/swipe changes slide/index
   - no auto-moving behavior where user explicitly asked for manual swipe
7. Only then report completion.

## Mobile hero carousel pattern that worked

For the full-bleed hero text carousel inside the background image:

- Use native horizontal scrolling with `overflow-x` and `scroll-snap-type` instead of fake transform-only swiping inside a centered wrapper.
- The carousel track must span the viewport width on mobile:
  - `track.left === 0`
  - `track.right === viewportWidth`
  - `track.clientWidth === viewportWidth`
- Each slide should be `100vw` or the exact carousel viewport width.
- Update dots based on `scrollLeft / clientWidth` or equivalent active-slide calculation.
- Let users swipe manually; auto-rotation is acceptable only if it does not block manual swipe and pauses appropriately.

## Listing detail photo layout pattern

For property detail pages:

- Do not dump all photos vertically above the description.
- Use an Airbnb-style compact mosaic:
  - one large primary image
  - four smaller secondary images on desktop when available
  - compact mobile mosaic that still lets the listing title/description appear in the first viewport or shortly after
- Clicking a photo opens a fullscreen lightbox.
- Lightbox should support manual swipe/scroll left-right; do not auto-advance photos.
- Validate on mobile that listing content starts soon after the photo mosaic.

## UX principle for Keith

Keith’s default expectation for this project is mobile-first polish. His phone screenshots are the source of truth. If a rendered screenshot looks clunky, off-center, cramped, or misaligned, treat that as a failed implementation even if the code and health checks pass.
