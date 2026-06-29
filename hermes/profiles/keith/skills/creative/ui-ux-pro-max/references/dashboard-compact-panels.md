# Compact dashboard summary panels

Pattern for shrinking metric/summary cards to reduce vertical scrolling on dashboards.

## When to apply

- Dashboard has 4+ summary/strategy cards stacked above a data table
- Cards take up >40% of viewport before the table starts
- User complains about scrolling to reach the data

## Pattern

1. **Switch from flex-wrap to auto-fill grid** — creates side-by-side tiles instead of stacked rows
2. **Reduce card internals** — smaller padding, fonts, gaps
3. **Inline the section header** — remove the full card box around titles
4. **Collapse explain/FAQ sections** by default (CSS `display:none`)

## CSS recipe

```css
/* Before: flex row wrapping */
.pnl-summary{display:flex;gap:12px;flex-wrap:wrap;margin:10px 0 16px}
.pnl-card{padding:12px 16px;min-width:160px}
.pnl-card .val{font-size:22px}
.pnl-card .kind{font-size:10.5px;margin-bottom:4px}

/* After: compact auto-fill grid */
.pnl-summary{display:grid;grid-template-columns:repeat(auto-fill,minmax(175px,1fr));gap:8px;margin:0 0 12px}
.pnl-card{padding:8px 12px}
.pnl-card .val{font-size:18px}
.pnl-card .kind{font-size:9.5px;margin-bottom:2px}

/* Mobile: tighter grid */
@media(max-width:900px){
  .pnl-summary{grid-template-columns:repeat(auto-fill,minmax(140px,1fr));gap:6px}
  .pnl-card{padding:6px 10px}
  .pnl-card .val{font-size:16px}
}
```

## Section header compaction

```html
<!-- Before: bulky card -->
<div class="card"><h2>Paper Bets</h2><p class="muted">description</p></div>

<!-- After: inline header -->
<div style="display:flex;align-items:baseline;gap:8px;margin-bottom:10px">
  <h2 style="margin:0">Paper Bets</h2>
  <span class="muted" style="font-size:11px">description</span>
</div>
```

## Pitfalls

- Don't shrink cards below 140px min-width on mobile — text becomes unreadable
- Don't remove the labels entirely — users need to know what each card represents
- Verify no horizontal overflow after changing from flex to grid (grid can sometimes cause overflow if minmax is too wide)
- The `auto-fill` keyword creates as many tracks as fit; use `auto-fit` if you want empty tracks to collapse

## Main metric grid (top summary cards)

The top-of-page metric cards (e.g. "22 open bets", "-6.90% P/L", "0 arb", "last scan") are the biggest vertical-space offenders on mobile. Fix:

```css
/* Before: fixed 4-column, big cards, single-column on mobile */
.grid{display:grid;grid-template-columns:repeat(4,1fr);gap:12px;margin-bottom:16px}
.card{background:var(--surface);border:1px solid var(--border);border-radius:14px;padding:14px 16px}
.metric{font-size:28px}
.label{font-size:11px;margin-bottom:4px}
.sub{font-size:12px;margin-top:4px}

/* After: auto-fill grid, compact cards, never single-column */
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(160px,1fr));gap:8px;margin-bottom:12px}
.card{background:var(--surface);border:1px solid var(--border);border-radius:10px;padding:8px 12px}
.metric{font-size:22px}
.label{font-size:10px;margin-bottom:2px}
.sub{font-size:11px;margin-top:2px}

/* Mobile: still auto-fill, smaller but never single-column */
@media(max-width:900px){
  .grid{grid-template-columns:repeat(auto-fill,minmax(130px,1fr));gap:6px}
  .card{padding:6px 10px}
  .metric{font-size:18px}
  .sub{font-size:10px}
  .label{font-size:9px}
}
```

Key: use `auto-fill` on BOTH desktop and mobile. Never let `.grid` collapse to `1fr` (single-column) or `grid-template-columns:1fr` — that makes 4 tall cards stacked.

## Filter row anti-stacking (critical for mobile)

The most common mobile regression: filter dropdowns that stack vertically because of a `width:100%` override.

```css
/* BEFORE — THIS IS THE BUG: forces stacking on mobile */
@media(max-width:900px){
  .filters select,.filters input,.filters button{width:100%}
}

/* AFTER — REMOVE the width:100% rule entirely */
@media(max-width:900px){
  .filters select,.filters input{min-height:34px}  /* only tighten height */
  /* NO width:100% — lets flex-wrap handle naturally */
}
```

Why this works: `.filters` is already `display:flex;flex-wrap:wrap`. Without `width:100%`, the 3 dropdowns + clear button share the row until they naturally wrap. With `width:100%`, each element takes the full row — guaranteed 4 rows of filters before the table.

## Whole-page density tightening checklist

When reducing vertical scrolling, touch EVERY element — not just the obvious cards:

| Element | Default | Compact |
|---------|---------|---------|
| `.wrap` padding | 20px 24px | 12px 16px |
| Title `h1` | 26px | 22px |
| Filter bar padding | 10px; gap 8px | 8px; gap 6px |
| Form element min-height | 38px | 34px |
| Form element padding | 8px 12px | 6px 10px |
| Tab buttons margin | 16px 0 18px | 10px 0 12px |
| Tab buttons padding | 8px 14px | 6px 14px |
| Table cell padding | 8px 10px | 6px 10px |
| Chart card height | 240px | 200px |
| Logbox padding | 14px 16px | 10px 14px |
| Logbox font-size | 13px | 12px |
| Logbox max-height | 500px | 350px |
| `h2` font-size | 18px | 16px |
| Explain toggle padding | 10px 16px | 7px 14px |
| Empty state padding | 40px | 32px |

Every 2-4px saved across 10+ elements compounds into 60-100px of recovered vertical space.

## Beyond compaction: replacing metric cards with a stats strip

When even compact cards feel heavy, replace the entire `.grid` of cards with a **stats strip** — horizontal pill-shaped indicators that fit on one line:

```css
.stats-strip{display:grid;grid-template-columns:repeat(auto-fill,minmax(150px,1fr));gap:6px;margin-bottom:8px}
.stat-item{background:var(--surface);border:1px solid var(--border);border-radius:8px;padding:6px 10px}
.stat-val{font-size:17px;font-weight:800;letter-spacing:-.02em;line-height:1}
.stat-label{color:var(--text3);font-size:9.5px;text-transform:uppercase;letter-spacing:.06em;font-weight:600}
.stat-sub{color:var(--text2);font-size:10px;margin-top:1px}
```

Render in JS:
```javascript
function renderMetrics(s){
  document.getElementById('metrics').innerHTML=[
    ['Open Bets', s.open_paper, 'active'],
    ['Closed P/L', cp.toFixed(2)+'%', 'realized'],
    ['Arb', s.arb_count, 'gaps detected'],
    ['Last Scan', latest, 'scanner']
  ].map(x=>`<div class="stat-item"><div class="stat-label">${x[0]}</div><div class="stat-val ${x[2]==='realized'?(cp>=0?'green':'red'):''}">${x[1]}</div><div class="stat-sub">${x[2]}</div></div>`).join('');
}
```

This saves ~50% more vertical space than even compact `.card`-based grids. Combined with the filter drawer (see `references/filter-drawer-modal.md`), the data table becomes visible almost immediately on page load — no permanent filter bars, no bulky metric cards, just a tight header and a filter icon that opens a centered floating modal with backdrop-blur.

**IMPORTANT UX RULE**: Filters in the drawer must NOT auto-apply on each select change. The selects preview state (badge count) but only apply when the user clicks "Done". Auto-applying on every select causes the table to flicker behind the open drawer — users find this confusing and broken. See `references/filter-drawer-modal.md` for the full Done-button pattern.

## Modern header unification

Merge the title, status badge, global controls (time range, keyword, refresh, auto-refresh, export) into a single slim header bar instead of a separate `.top` section + `.filters` bar:

```css
.header{display:flex;align-items:center;gap:10px;margin-bottom:10px;flex-wrap:wrap}
.header-logo{font-size:20px;font-weight:800;margin-right:auto}
.header-actions{display:flex;align-items:center;gap:6px;flex-wrap:wrap}
```

```html
<div class="header">
  <div class="header-logo">Polymarket Intel <span>paper trading</span></div>
  <div class="badge-status"><span class="dot"></span><span id="status">Updated ...</span></div>
  <div class="header-actions">
    <select id="range">...</select>
    <input id="q" placeholder="Keyword…">
    <button onclick="loadAll()">Refresh</button>
    <button id="autoRefreshBtn">Auto</button>
    <a href="/api/export/paper_trades.csv">Export CSV</a>
  </div>
</div>
```

Mobile: use `flex-wrap:wrap` (never `flex-direction:column`) — lets items share the row until they naturally wrap. On the narrowest screens (<400px), the actions wrap to a second row via `width:100%`.

## Sticky-bottom pagination (table fills viewport, buttons pinned)

When the data table + pagination leaves empty space below the Prev/Next buttons, restructure the section as a flex column that fills the viewport:

```css
/* Wrap must fill viewport height */
.wrap{display:flex;flex-direction:column;min-height:100vh;min-height:100dvh}
```

```html
<!-- Paper/tab section: flex column, no fixed height -->
<section id="paper" style="display:flex;flex-direction:column;min-height:0">
  <div class="section-header">...</div>             <!-- fixed height -->
  <div class="pnl-summary" id="pnlSummary"></div>   <!-- fixed height -->
  <div id="paperTable" class="table-wrap" style="flex:1;min-height:0"></div>  <!-- FILLS REMAINING -->
  <div id="paperPagination" style="flex-shrink:0">  <!-- pinned to bottom -->
</section>
```

Key rules:
- `flex:1; min-height:0` on the table-wrap — this is what makes it fill remaining space
- `flex-shrink:0` on the pagination — prevents it from being squeezed when table has many rows
- `min-height:0` is CRITICAL — without it, flex children default to `min-height:auto` which prevents shrinking below content height
- `min-height:100dvh` on wrap (not `100vh`) — `dvh` accounts for mobile browser address bar

**Before**: pagination floated mid-page with empty space below → required scrolling to see buttons
**After**: pagination pinned to viewport bottom, always visible — gap typically 8-10px

## Table row/page sizing for pagination visibility

Reduce rows-per-page and tighten cell padding so the pagination bar fits above the fold:

```css
/* Before: 12.5px font, 5px padding */
table{font-size:12.5px}
th,td{padding:5px 8px}

/* After: 12px font, 4px padding — saves ~12px per row saving ~144px total */
table{font-size:12px}
th,td{padding:4px 8px}
th{font-size:10px}
```

```javascript
// Before: 20 rows fills viewport, pushes pagination off-screen
const PAPER_PAGE_SIZE = 20;

// After: 12 rows leaves room for pagination at bottom
const PAPER_PAGE_SIZE = 12;
```

## Sort arrow visibility

Default sort arrows (▴/▾) at 8px are invisible on mobile. Make them 10px with hover opacity:

```css
th .sort-arrow{font-size:10px;margin-left:3px;opacity:.6}
th:hover .sort-arrow{opacity:1}
```
