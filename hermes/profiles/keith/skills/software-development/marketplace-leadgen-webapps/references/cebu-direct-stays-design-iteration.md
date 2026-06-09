# Cebu Direct Stays design iteration notes

Session-derived notes for mobile-first travel/accommodation marketplace redesigns.

## User taste signals

- Keith prefers no-BS, direct critique and fast iteration over long theoretical design explanation.
- For accommodation/travel pages, “clean” is not enough; the page must create a vacation/travel feeling.
- Mobile is the priority because most guests will browse from phones.
- If an element looks like a button, it should be clickable. Decorative pills/chips that look like CTAs are clutter.

## Specific design lessons

### Category navigation

Do not duplicate stay-type/category links in multiple places on the homepage. If categories are already in the top nav/hamburger menu, remove large category cards from the homepage. On mobile they read as clutter and distract from the primary guest path.

Good homepage primary actions:

- Browse Cebu stays
- List your property

Keep detailed stay-type routes in the nav/hamburger:

- Mactan airport stays
- Cebu daily condos
- Weekly & monthly stays
- Staycation condos

### Mobile hamburger behavior

Mobile menus should close when:

- user taps outside the menu
- user taps a menu link
- user presses Escape

The hamburger icon should visibly switch to an X/open state. Verify this behavior in browser JS, not just by source inspection.

### Cinematic hero treatment

For a modern travel feel, make the hero background image full-bleed across the top of the site. Do not place the background image inside a rounded card if the user references cinematic travel-site screenshots.

Pattern:

- fixed/overlay nav on top of hero image
- full-width, full-viewport-ish hero section
- gradient scrim over image for headline legibility
- large emotional headline
- short supporting copy
- two real CTAs only
- nav becomes solid/blurred after scroll

Avoid:

- hero image trapped inside a card
- non-clickable trust pills designed like buttons
- repeated category cards below the hero
- too many click targets on first mobile screen

### Animation

Add subtle interaction, but don’t hide content behind JS:

- scroll reveal as progressive enhancement
- card hover/touch lift
- image zoom/saturation on card hover
- button press/hover states
- respect `prefers-reduced-motion`
- content should remain visible if JS fails

### Typography/spacing

Watch screenshots for:

- sections/cards touching or crowding each other
- overly tight letter spacing that makes headings feel compressed
- desktop spacing that collapses badly on mobile

Inter with relaxed tracking worked better than a rounded Airbnb-like font for this project after the user rejected the Airbnb-adjacent feel.

## Verification checklist

After each design pass:

1. Open homepage in browser.
2. Inspect the top hero visually.
3. Scroll down; ensure no hidden/blank sections.
4. Check mobile width or screenshot if possible.
5. Test hamburger open/close and outside-click behavior.
6. Confirm duplicate category cards/fake pills are absent when removed.
7. Confirm Docker app is still healthy if deployed.
