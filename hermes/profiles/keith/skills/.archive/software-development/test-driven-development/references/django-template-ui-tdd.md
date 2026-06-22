# Django template/UI TDD notes

Use this when changing Django-rendered pages, forms, or lightweight JavaScript interactions.

## RED: assertions that catch template regressions

Write a Django `TestCase` before editing templates/models/views. Useful assertions:

- `assertContains(response, 'Visible copy')` for required UX text.
- `assertContains(response, 'data-action-name')` for JS hooks.
- `assertNotIn('old-wrapper-class', response.content.decode())` when removing a UI block.
- `response.content.index(b'First section') < response.content.index(b'Second section')` to lock important page order, e.g. amenities before map.
- Create minimum fixture models in `setUp()` and hit the real URL via `self.client.get(model.get_absolute_url())`.

## GREEN: minimal production changes

For template-only changes, prefer small model properties over view-specific string assembly when the value belongs to the object, e.g. approximate map label/bounds for a `Listing`.

## Browser verification after tests

Django tests prove server-rendered markup, but not actual browser JS behavior. After tests pass:

1. Run the dev server or rebuilt container that the user will actually inspect.
2. Open the real page in the browser tool.
3. Click the interaction target, e.g. sticky `Message host` button or `See all amenities` inline expander.
4. Verify the state transition via DOM inspection, not just a screenshot: `aria-expanded`, changed button text, hidden element count, visible class changes, or modal open state.
5. Toggle back/collapse if applicable and verify the reverse state too.
6. Check console errors.

For compact inline list/expander UI, good DOM probes include initial hidden extra count, delimiter-separated visible text, `data-*` hooks, `aria-expanded=false -> true -> false`, and button text `See all ... -> See less -> See all ...`.

## Static asset verification

When CSS/JS changes are involved, bump the cache query string in the base template and run `collectstatic` for production/staticfiles-backed serving. If the browser still serves stale or 404 static, verify the served static URL directly and restart the dev server in DEBUG mode for local browser QA if needed.

## Privacy-preserving maps for real-estate listings

For public property pages, prefer approximate area maps over exact unit/building coordinates unless the product explicitly wants exact location disclosure. Show copy explaining that exact location is shared later for host privacy and guest safety.
