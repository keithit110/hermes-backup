# Scroll-animation UI verification

Use this when adding scroll-triggered animation to server-rendered pages or static marketing/listing UI.

## Test-first hooks

Before editing templates/CSS/JS, add a failing rendered-page test that asserts durable animation hooks exist on the actual sections/items, for example:

- section/container class: `scroll-pop-section`
- item class: `scroll-pop-item`
- `data-scroll-reveal`
- semantic animation style: `data-reveal-style="unit-type"` / `data-reveal-style="listing-card"`

Also add an asset-level test that reads CSS/JS and asserts:

- named version comment for the change
- expected keyframe name, e.g. `scrollPopIn`
- `prefers-reduced-motion: reduce` in CSS
- reduced-motion guard in JS

## Implementation pattern

- Put `data-scroll-reveal` on both grouped containers and individual cards/chips when both should reveal.
- Use IntersectionObserver to add `.is-visible` once, then unobserve the target.
- Preserve visibility/fallback behavior: if IntersectionObserver is unavailable or reduced motion is enabled, immediately add `.is-visible`.
- Calibrate motion to the brand/site. For Keith's marketplace/listing UI, avoid both extremes: barely visible fade-ups are not enough, but gimmicky 3D pop/rotate/blur effects do not fit. Prefer clearly noticeable travel-marketplace motion: 28–50px vertical entry, .94–.97 starting scale, 0.7–1.0s duration, short stagger, optional clip-path/unmask on listing cards, and a tiny overshoot before settling.
- For unit/category chips, use a visible sweep/lift-in with mild scale and stagger.
- For listing cards, make the card reveal obvious: start lower, scale in, optionally unmask the bottom with `clip-path`, and verify mid-animation computed styles — final screenshots alone can look static.
- Add hover/tap feedback separately from scroll reveal.
- Bump static cache-busters in the base template when CSS/JS changes.

## Runtime verification checklist

After tests pass and the running site is rebuilt/restarted:

1. Confirm the served HTML references the new cache-busted assets.
2. Confirm the served CSS contains the new version comment/keyframe.
3. In browser automation, scroll to the target sections and inspect DOM/computed style:
   - `document.body.classList.contains('scroll-motion-ready')`
   - visible item count via `.is-visible`
   - computed `animationName` equals the intended keyframe for revealed cards
   - console has no JS errors
4. Capture mobile screenshots at the unit-type strip and listing-card positions.
5. Visually verify no broken layout, sticky nav overlap, or hidden cards.

## Pitfall

A static screenshot after the animation finishes only proves the final layout, not that the animation ran. Pair screenshots with DOM/computed-style checks while/after scrolling.