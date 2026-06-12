# Mobile hero dead-space validation

Session-derived note from Cebu Direct Stays mobile landing-page work.

## Trigger

Keith marked a mobile screenshot with arrows pointing at wasted vertical space inside a landing-page hero. The issue was not merely aesthetics: on a mobile accommodation/lead-gen homepage, above-the-fold pixels are conversion-critical.

## Durable lesson

When a mobile hero uses a full-height/full-bleed image carousel, the content can look technically centered but still waste valuable vertical real estate. Treat this as a layout/conversion bug.

## Fix pattern

- Scope changes to mobile breakpoints first; do not wreck desktop hero composition.
- Reduce oversized hero/slide `min-height` or vertical padding when it pushes the CTA below the fold.
- Move the headline/lead/CTA cluster upward as a group rather than only shrinking text.
- Tighten gaps between:
  - nav/header and kicker/headline
  - lead text and carousel dots
  - dots and CTA buttons
- Keep the overlay legible; if moving content upward crosses a bright image region, adjust gradient/overlay instead of abandoning the tighter layout.
- Keep primary CTA visible and visually dominant on initial mobile viewport.

## Verification pattern

Use a real rendered mobile viewport and screenshot review, not just tests or `curl`.

Recommended checks:

1. Capture at the approximate height shown by the user, not only a tall modern phone viewport.
2. Inspect the screenshot visually: headline, lead, dots, and buttons should read as one compact action group.
3. Measure DOM geometry for objective spacing, e.g. nav bottom, kicker top, headline bounds, lead bounds, dot bounds, button bounds, and lead-to-dot/button gaps.
4. Verify the active hero slide is still horizontally centered if carousel code changed.
5. Run the app's targeted tests and restart/rebuild the Docker service if static assets are bundled.

## Reporting back

Keep the response concise. Say what caused the dead space, what changed, and what was verified. Include the screenshot/media artifact when available.