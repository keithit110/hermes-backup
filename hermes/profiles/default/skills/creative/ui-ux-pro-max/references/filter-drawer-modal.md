# Filter drawer modal pattern

Replace permanent filter bars with a centered floating modal activated by a filter icon button. Saves vertical space and keeps the main page clean.

## When to use

- Dashboard has 3+ filter dropdowns taking permanent space above the data table
- User complained about scrolling past filters to reach data
- Mobile: filter bars stacking vertically is the #1 space waster

## Pattern: centered floating card with backdrop-blur

```css
/* Overlay: dark + blur, not just dark */
.drawer-overlay{
  position:fixed;inset:0;
  background:rgba(0,0,0,.35);
  backdrop-filter:blur(6px);
  -webkit-backdrop-filter:blur(6px);
  z-index:100;
  opacity:0;visibility:hidden;
  transition:opacity .2s,visibility .2s
}
.drawer-overlay.open{opacity:1;visibility:visible}

/* Panel: centered, NOT full-height side panel */
.drawer-panel{
  position:fixed;top:50%;left:50%;
  transform:translate(-50%,-50%) scale(.95);
  width:380px;max-width:92vw;max-height:85vh;
  background:var(--surface);
  border:1px solid var(--border);
  border-radius:12px;
  z-index:101;
  display:flex;flex-direction:column;
  box-shadow:0 12px 48px rgba(0,0,0,.5);
  opacity:0;
  transition:opacity .2s,transform .2s cubic-bezier(.4,0,.2,1)
}
.drawer-panel.open{
  transform:translate(-50%,-50%) scale(1);
  opacity:1
}
```

## Anti-pattern: side panel (DO NOT USE)

```css
/* WRONG — covers the right side of the page, blocks content */
.drawer-panel{
  position:fixed;top:0;right:0;bottom:0;
  width:340px;max-width:90vw;
  transform:translateX(100%);
  border-left:1px solid var(--border);
}
.drawer-panel.open{transform:translateX(0)}
```

The side panel covers the ENTIRE right portion of the viewport and blocks all content behind it. User feedback: "The filters are basically covering everything in the main page."

## Filter button in section header

```html
<div class="section-header">
  <h2>Paper Bets</h2>
  <span class="muted">click headers to sort</span>
  <div class="filter-badge">
    <button class="btn-icon" onclick="toggleDrawer()">Filters</button>
    <span class="filter-count" id="filterCount" style="display:none">0</span>
  </div>
</div>
```

The filter-count badge shows number of active filters (positioned absolute, top-right of button).

## JS: toggle, open, close

```javascript
function toggleDrawer(){
  let panel = document.getElementById('drawerPanel');
  panel.classList.contains('open') ? closeDrawer() : openDrawer();
}
function openDrawer(){
  document.getElementById('drawerPanel').classList.add('open');
  document.getElementById('drawerOverlay').classList.add('open');
}
function closeDrawer(){
  document.getElementById('drawerPanel').classList.remove('open');
  document.getElementById('drawerOverlay').classList.remove('open');
}
// Escape key closes
document.addEventListener('keydown', e => { if(e.key === 'Escape') closeDrawer(); });
```

## HARD RULE: filters must NOT auto-apply on select

**User explicitly corrected this.** The selects in the drawer preview their state (badge count updates) but do NOT trigger re-rendering. The "Done" button applies all filters at once and closes the drawer. The "Clear" button resets and applies.

```javascript
// Select onchange: badge preview ONLY — no render, no close
document.getElementById('paperStrategy').onchange = () => { updateFilterBadge(); };
document.getElementById('paperStatus').onchange   = () => { updateFilterBadge(); };
document.getElementById('paperSide').onchange     = () => { updateFilterBadge(); };

// Done button: applies all filters THEN closes
function applyFilters(){
  paperPage = 1;
  updateFilterBadge();
  renderPaperTable();
  closeDrawer();
}

// Clear button: resets, applies, closes
function resetPaperFilters(){
  document.getElementById('paperStrategy').value = '';
  document.getElementById('paperStatus').value = '';
  document.getElementById('paperSide').value = '';
  paperPage = 1;
  updateFilterBadge();
  renderPaperTable();
  closeDrawer();
}
```

```html
<div class="drawer-footer">
  <button onclick="resetPaperFilters()">Clear</button>
  <button class="btn-primary" onclick="applyFilters()">Done</button>
</div>
```

**Why this matters:** If onchange fires renderPaperTable() immediately, the user sees the table flicker/change behind the open drawer for every dropdown selection. This is confusing and feels broken. The "Done" pattern gives the user control over when changes take effect.

## Badge count update

```javascript
function updateFilterBadge(){
  let f = getPaperFilters();
  let count = (f.strategy ? 1 : 0) + (f.status ? 1 : 0) + (f.side ? 1 : 0);
  let badge = document.getElementById('filterCount');
  if(count > 0){ badge.style.display = 'flex'; badge.textContent = count; }
  else { badge.style.display = 'none'; }
}
```

## Pitfalls

- Don't make the drawer full-height on mobile — use `max-width:92vw` + `max-height:85vh` for the centered card, works on all screen sizes
- Don't auto-apply filters on select change — this is the #1 UX bug users complain about
- Don't forget to call `updatePaperFilters()` (populates select options from data) in `loadAll()` before the user opens the drawer
- The drawer-panel selector references MUST be in the DOM at page load (not dynamically created) — otherwise `getElementById` bindings fail
- Drawer panel and overlay sit OUTSIDE `.wrap` to keep z-index stacking clean
