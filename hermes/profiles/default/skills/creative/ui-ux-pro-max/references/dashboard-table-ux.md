# Dashboard Table UX Patterns

Hard-won patterns from Polymarket Intel dashboard iteration. These are the patterns
that caused multiple rounds of user frustration before being corrected.

## 1. pointer-events trap — invisible elements still intercept clicks

**Symptom:** Clicking a table column header opens a filter modal instead of sorting.
**Root cause:** A hidden drawer panel (opacity:0, visibility:hidden) still occupies
space and intercepts pointer events. Its `<select>` elements sit on top of the table.

**Fix:** Always set `pointer-events:none` on hidden panels:

```css
.drawer-panel {
  opacity: 0;
  pointer-events: none;
  transition: opacity .2s, transform .2s, pointer-events 0s .2s;
}
.drawer-panel.open {
  opacity: 1;
  pointer-events: auto;
  transition: opacity .2s, transform .2s, pointer-events 0s 0s;
}
```

The delayed `pointer-events` transition (0s .2s on close) waits for the fade-out
animation to complete before disabling interaction. On open, it's immediate (0s 0s).

**Verification:** In Playwright, check `getComputedStyle(el).pointerEvents` on closed
panel is `"none"`. Click table headers and verify drawer stays closed.

## 2. Sticky-bottom pagination with CSS flex

**Symptom:** Pagination buttons have empty space below them, or require scrolling.
**Goal:** Pagination pinned to viewport bottom, table fills all remaining space.

```css
.wrap {
  display: flex;
  flex-direction: column;
  height: 100dvh;               /* use dvh not vh for mobile browsers */
  padding: 6px 16px 0;          /* zero bottom padding */
}
/* The scrollable section */
#paper {
  flex: 1;                      /* fill all remaining space */
  min-height: 0;                /* CRITICAL: allow flex shrink below content height */
  overflow: hidden;             /* section itself doesn't scroll */
  display: flex;
  flex-direction: column;
}
/* The scrollable content */
.table-wrap {
  flex: 1;
  min-height: 0;
  overflow: auto;              /* THIS is what scrolls */
}
/* The pinned footer */
#pagination {
  flex-shrink: 0;              /* never shrink */
}
```

Key gotchas:
- `min-height:0` on flex children is ESSENTIAL — without it, flex:1 won't shrink below
  the content's natural height
- `100dvh` handles mobile browser chrome (address bar) correctly; `100vh` doesn't
- Zero bottom padding on the wrap eliminates the gap below pagination

## 3. Raw Python strings + inline JavaScript

**Symptom:** Page loads but shows "Loading…" with empty containers. No data renders.
**Root cause:** Python `r"""..."""` raw strings preserve backslashes literally.
Inside inline JavaScript, `\"` becomes a literal backslash-double-quote that the JS
engine can't parse. The entire `<script>` block fails silently.

**Example of BROKEN code in a raw string:**
```python
PAGE = r"""
function esc(v){return String(v||'').replace(/[&<>\"']/g,s=>({'"':'&quot;'}[s]))}
"""
```
The `\"` in the regex is fine (JS regex accepts `\"`). But `'"'` in the object literal
key becomes `\'\"\'` which is a JS syntax error.

**Fix: Use if-statements instead of object literals with quote keys:**
```javascript
function esc(v){
  v = String(v||'');
  return v.replace(/[&<>"']/g, function(s){
    if(s === '&') return '&amp;';
    if(s === '<') return '&lt;';
    if(s === '>') return '&gt;';
    if(s === '"') return '&quot;';
    if(s === "'") return '&#39;';
    return s;
  });
}
```

**Verification:** After any template edit, ALWAYS run:
```bash
curl -s http://localhost:PORT/ | sed -n '/<script>/,/<\/script>/p' | sed '1d;$d' > /tmp/js.js
node -c /tmp/js.js && echo "PASS" || echo "SYNTAX ERROR"
```

## 4. Verification discipline for dashboard UI

When the user requests a UI fix, THIS is the minimum verification:

1. `docker compose build web && docker compose down web && docker compose up -d web`
2. Wait for container ready: `curl -s -o /dev/null -w '%{http_code}' http://localhost:PORT/`
3. `node -c` the extracted JS (see §3)
4. Playwright at 390×844 viewport:
   - Check status text is NOT "Loading…" (data loaded)
   - Check stat pills have non-zero values
   - Check table has rows
   - Check computed font sizes match CSS declarations
   - Click table headers → verify drawer stays closed (getComputedStyle pointerEvents)
   - Check pagination position: gap from viewport bottom ≤ 10px
5. Take screenshot, share with user

**Grep alone is NEVER verification.** The user considers screenshots without data as
"empty" and will call it a failure.
