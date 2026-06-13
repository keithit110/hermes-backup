# Property card horizontal carousel pattern

Session-derived lesson from Cebu Direct Stays: Keith rejected vertical featured-property browsing with per-card scroll animation because it still made users scroll down through a stack. He wanted the Airbnb-style homepage behavior: browse properties left/right.

## When to use

Use this pattern for accommodation/marketplace homepages when the user says property cards should browse like Airbnb or when vertical featured-card stacks make the page feel long.

## Implementation pattern

- Keep category/unit-type filters separate; do not remove or redesign them if the user explicitly approved them.
- Remove listing-card scroll-reveal hooks from the reusable card partial so property cards do not fade/deal in on vertical scroll.
- Wrap only the homepage featured listings in a carousel section:
  - `data-property-carousel`
  - `data-property-carousel-track`
  - optional previous/next buttons for desktop/tablet
- Use native horizontal scrolling with CSS grid columns, not transform-only fake sliders:
  - `display: grid`
  - `grid-auto-flow: column`
  - mobile `grid-auto-columns: minmax(282px, 84vw)` or similar
  - `overflow-x: auto`
  - `scroll-snap-type: x mandatory`
  - each `.listing-card { scroll-snap-align: start }`
  - hide scrollbars but preserve touch scrolling
- On desktop/tablet, add small arrow buttons that call `track.scrollBy({ left: direction * step(), behavior: 'smooth' })`.
- Keep the full properties/index page as a normal grid unless the user asks for carousel browsing there too.

## Verification checklist

After deployment, verify in a mobile/touch viewport:

- rendered homepage includes the carousel region and track
- cards are visible side-by-side/peeked, making sideways browsing obvious
- `getComputedStyle(track).gridAutoFlow == 'column'`
- `getComputedStyle(track).overflowX == 'auto'`
- `getComputedStyle(track).scrollSnapType` includes `x mandatory`
- `track.scrollWidth > track.clientWidth`
- manual horizontal scroll changes `track.scrollLeft`
- first card has `animationName == 'none'`
- card class is plain `card listing-card`, not `listing-card scroll-pop-item`
- browser console has no JS errors
- capture and inspect a mobile screenshot before saying done

## Pitfall

Do not answer this request by making the vertical scroll animation more dramatic. If the user says “like Airbnb” and “swipe left/right,” the interaction model changed: remove the per-card vertical animation and implement horizontal browsing.