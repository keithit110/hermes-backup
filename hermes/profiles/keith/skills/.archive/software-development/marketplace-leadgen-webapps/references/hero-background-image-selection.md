# Hero background image selection for travel/direct-booking marketplaces

Session-derived notes from Cebu Direct Stays hero replacement.

## Trigger

Use this when Keith dislikes a public landing-page hero/background image as low quality, pixelated, generic, or not emotionally compelling.

## Pattern that worked

1. Treat the hero background as a conversion/design asset, not decoration.
2. Generate a broad numbered set of options before applying one. For Keith, 20 numbered samples worked well.
3. Use a single contact sheet so the user can compare options quickly instead of opening many links.
4. Ask the user to choose by number, then apply exactly that selected image.
5. Before applying, inspect the chosen image for the specific subject/emotion the user named.
   - In the Cebu session, Keith picked #18 because the hammock “screams” siesta/paradise.
   - Verified the hammock was visible before using it.
6. Prefer saving the selected image as a local static asset instead of hotlinking generated/external URLs.
7. Export/upscale to a hero-appropriate size such as 1920x1080 JPEG, then cache-bust CSS.
8. Verify in real browser screenshots on desktop and mobile.

## Prompt guidance

For accommodation/travel marketplace hero options, include:

- premium tropical/vacation-paradise mood
- realistic photographic look, not illustration
- wide 16:9 composition
- clean/darker area for white headline text
- no people, no text, no logos, no watermark
- explicit local vibe when relevant: Cebu/Philippines/tropical island/resort/condo staycation

Generate variations across beach, hammock, resort pool, balcony, skyline/ocean, lagoon, villa, and sunset/blue-hour moods.

## Implementation notes

For Django/static apps:

```text
app/static/img/<descriptive-hero-name>.jpg
app/static/css/site.css -> --cinema-img:url('../img/<descriptive-hero-name>.jpg')
base template -> bump ?v=<n> on CSS
```

Use PIL if needed to resize/export:

```python
from PIL import Image, ImageFilter
img = Image.open(src).convert('RGB')
img = img.resize((1920, 1080), Image.Resampling.LANCZOS)
img = img.filter(ImageFilter.UnsharpMask(radius=1.2, percent=115, threshold=3))
img.save(out, quality=92, optimize=True, progressive=True)
```

## Verification checklist

- Tests pass after adding the asset.
- Homepage and health endpoint return 200 after rebuild/restart.
- Static image URL returns 200 and non-trivial byte size.
- CSS version is bumped and loaded by the page.
- Browser computed `backgroundImage` references the new static asset.
- Desktop screenshot: subject is visible, headline readable.
- Mobile screenshot: subject is still visible enough after cover-cropping, headline/readability intact.

## Pitfall

A selected image may look great in the raw contact sheet but crop poorly under `background-size: cover`, especially on mobile. Always verify mobile after applying; if the emotional subject disappears, adjust `background-position`, overlay, crop, or ask for a better variant.