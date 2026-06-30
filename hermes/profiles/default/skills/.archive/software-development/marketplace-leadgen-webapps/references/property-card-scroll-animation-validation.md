# Property card scroll animation validation

Session-derived note from Cebu Direct Stays homepage/property-card animation work.

## Trigger

Keith said the individual property/listing cards on the homepage looked like they merely appeared/disappeared on scroll and did not feel special. He explicitly said the unit-type/category cards were okay and should not be changed.

## Durable lesson

When Keith critiques a frontend animation, preserve the parts he explicitly approves. Scope the change narrowly to the target component and make the new motion visibly different enough to pass a human screenshot/video review. A tiny opacity/translate reveal is often too subtle for property marketplace cards.

## Fix pattern

For accommodation listing cards, use a more distinctive but tasteful scroll reveal:

- Keep content visible without JS; animation should be progressive enhancement.
- Leave unrelated category/unit-type animations untouched when the user approves them.
- Animate the listing card as a composed object, not just text opacity:
  - lower starting position
  - slight alternating left/right entry offset per card
  - small scale-up/settle
  - reveal/mask or clip-path that uncovers the lower card content
  - subtle overshoot before final resting state
- Animate the primary photo separately with a gentle zoom/settle so the card feels alive without gimmicks.
- Stagger cards lightly, but avoid long delays that make content feel broken.
- Respect `prefers-reduced-motion` and avoid hiding core listing info behind JavaScript-only state.

## Verification pattern

Do not verify this class of change only at page load. Scroll to the actual listing-card region and capture/check during the animation window.

Recommended checks:

1. Run the app's targeted tests for the animation names/classes if available.
2. Rebuild/restart the Docker service if CSS/JS cache-busted assets are served from the container.
3. Use a mobile/touch viewport and scroll until the first listing card is entering the viewport.
4. Inspect computed animation state for the listing card and photo, e.g. `animationName`, transform/clip-path, and custom entry offset variables.
5. Confirm the approved unit-type/category animation name did not change.
6. Capture a screenshot during the animation and inspect visually before saying done.
7. Check browser console errors.

## Reporting back

Be concise: state that the prior animation was too subtle, list what changed, explicitly mention that approved category/unit-type cards were untouched, and provide verification evidence plus screenshot/media if available.
