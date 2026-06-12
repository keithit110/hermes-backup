# Marketplace scroll motion guidance

Use this when adding scroll-triggered animations to travel, accommodation, marketplace, or listing-card interfaces.

## Durable lesson

For premium marketplace UIs, avoid demo-like motion. A technically working reveal can still feel wrong if it fights the site’s visual language.

Bad fit for Airbnb-style / premium travel listing pages:
- 3D `translate3d(..., z)` or perspective effects
- `rotateX` / card flipping
- heavy blur-in effects
- exaggerated scale or “flying toward the user” motion
- pulse/glow effects on category cards unless the rest of the brand already uses them

Better baseline:
- category/unit chips: short fade + 12–16px upward glide
- listing cards: subtle fade + 18–24px upward lift, tiny scale only if needed (`.985` → `1`)
- restrained stagger: 40–80ms between items, capped
- hover: small lift (`-3px` chips, `-5px/-6px` cards), shadow increases naturally
- duration: ~0.5s for chips, ~0.65–0.75s for listing cards
- easing: soft marketplace curve like `cubic-bezier(.2,.72,.22,1)`

## Example CSS pattern

```css
.scroll-pop-section{--marketplace-ease:cubic-bezier(.2,.72,.22,1)}
.scroll-pop-item{will-change:transform,opacity;transform-origin:center bottom}
body.scroll-motion-ready .scroll-pop-item:not(.is-visible){
  opacity:.001!important;
  transform:translateY(var(--reveal-y,22px)) scale(.985)!important;
}
.scroll-pop-item.is-visible{
  opacity:1!important;
  animation:listingLiftIn .68s var(--marketplace-ease) both!important;
  animation-delay:var(--reveal-delay,0ms)!important;
}
.unit-type-filter.scroll-pop-item:not(.is-visible){transform:translateY(14px)!important}
.unit-type-filter.scroll-pop-item.is-visible{animation-name:categoryGlideIn!important;animation-duration:.52s!important}
.listing-card.scroll-pop-item:hover{transform:translateY(-6px)!important;box-shadow:var(--shadow)!important}
@keyframes categoryGlideIn{from{opacity:.001;transform:translateY(14px) scale(.985)}to{opacity:1;transform:none}}
@keyframes listingLiftIn{from{opacity:.001;transform:translateY(var(--reveal-y,22px)) scale(.985)}to{opacity:1;transform:none}}
@media (prefers-reduced-motion: reduce){
  .scroll-pop-item,.scroll-pop-item.is-visible{animation:none!important;opacity:1!important;transform:none!important}
}
```

## Verification checklist

- Verify hooks and reduced-motion behavior with tests.
- Verify served cache-buster actually changed.
- Scroll the real mobile page and inspect screenshots, not just computed animation names.
- Ask: “Does this motion belong on this brand/site?” If not, reduce complexity before reporting done.
