# Cebu Direct Stays design iteration notes

Session-derived lessons for marketplace/direct-stay homepage work, especially for Keith's Cebu accommodation directory.

## Design corrections that mattered

- Mobile is the primary viewport. Most guests will likely arrive from phones/Facebook/Messenger, so the homepage should prioritize mobile clarity over desktop decoration.
- Do not duplicate navigation choices as large homepage cards when those same choices already live in the hamburger/top navigation. It creates clutter and makes users unsure what to do.
- If category options are in the hamburger/dropdown, the homepage should focus on the primary user actions:
  - Browse stays
  - List your property
  - Message/contact host from listing pages
- Avoid fake button-looking badges/pills that are not clickable. If an element looks like a button, it should do something. Otherwise use plain text, metadata, or remove it.
- Text spacing and vertical rhythm matter. Watch for boxed sections touching card grids, overly tight tracking, and cramped mobile layouts.
- Do not copy Airbnb's coral/pink palette by default. For Cebu Direct Stays, a teal/navy/gold direction felt more ownable.
- The user wants no-BS design critique and practical iteration: if he says it feels clunky/bland, treat that as a design failure, not as a request for tiny tweaks.

## Hero pattern lesson

The user referenced a travel landing page where the background image covers the full top of the page and the text carousel sits *inside the hero image*.

Correct pattern:

```text
full-bleed vacation image
→ centered hero text slide
→ small carousel dots below the text
→ text auto-rotates; dots are clickable
→ primary CTAs below the dots
```

Incorrect pattern that frustrated the user:

```text
full-bleed image hero
→ separate story/card carousel below the hero
```

If the user points to circled text inside a screenshot, inspect whether the requested element is inside the hero/image area rather than a separate section.

## Mobile nav behavior

Hamburger/dropdown menus should close when:

- tapping outside the menu
- tapping a menu link
- pressing Escape

The hamburger icon should visually transition to an X when open. Verify this behavior in browser, not only by reading code.

## Verification checklist for this class of design pass

- Browser visual check of the hero and below-the-fold content.
- Mobile-width check if possible; at minimum inspect mobile CSS and nav behavior.
- Confirm removed text/cards are no longer present in homepage HTML.
- Confirm cache-bust version changed for CSS/JS so the browser sees updates.
- Confirm Docker app health endpoint returns `ok` after restart.
- Confirm no blank gaps caused by scroll reveal animations.

## Copy direction

Prefer customer-facing travel/booking copy over internal/product labels.

Avoid:

- "MVP"
- "directory MVP"
- fake UI labels
- overly technical explanations in hero copy

Better:

- "Where will you stay in Cebu?"
- "Find Cebu condos without the platform-fee headache."
- "Browse stays, compare essentials, and message the host directly."
