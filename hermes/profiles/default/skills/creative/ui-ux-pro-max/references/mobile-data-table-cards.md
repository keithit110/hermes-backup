# Mobile data table → stacked card pattern

Use when an existing desktop table must work on mobile without horizontal scrolling.

## Pattern

1. Render BOTH a `<table>` and a `<div class="mobile-cards">` — same data, two layouts
2. On desktop (≥769px): hide cards, show table
3. On mobile (≤768px): hide table, show cards as vertically stacked expandable rows
4. Each card has a visible header (title + key badges + P/L%) and a hidden body (detail rows)
5. Tap card header → `classList.toggle('open')` on the body div
6. Filters stack vertically (`flex-direction: column`) on mobile

## CSS skeleton

```css
@media(max-width:768px){
  .table-wrap{border:none;background:transparent;overflow:visible}
  .table-wrap table{display:none}
  .mobile-cards{display:flex;flex-direction:column;gap:8px}
  .mobile-card{background:var(--surface);border:1px solid var(--border);border-radius:10px;overflow:hidden}
  .mobile-card-header{display:flex;justify-content:space-between;padding:12px;cursor:pointer;gap:8px}
  .mobile-card-header:active{background:var(--surface2)}
  .mobile-card-body{display:none;border-top:1px solid var(--border);padding:8px 12px}
  .mobile-card-body.open{display:block}
  .filters{flex-direction:column;align-items:stretch}
}
@media(min-width:769px){
  .mobile-cards{display:none}
}
```

## JS pattern (inside shared `tbl()` function)

```javascript
// Build mobile cards alongside the table
let cards = '<div class="mobile-cards">' + data.map(r => {
  // Extract key fields for the card header
  let title = esc(r.event_title || '');
  let pnlRaw = Number(r.pnl_pct || 0);
  let sideVal = fmtSidePill(r._side);
  let statusVal = fmtStatus(r.status);
  
  // Build detail rows from remaining columns
  let detailRows = '';
  for (let c of cols) {
    if (['event_title','pnl_pct','_side','status'].includes(c[1])) continue;
    detailRows += `<div class="kv"><span class="k">${esc(c[0])}</span><span class="v">${fmtVal(c,r)}</span></div>`;
  }
  
  return `<div class="mobile-card">
    <div class="mobile-card-header" onclick="this.nextElementSibling.classList.toggle('open')">
      <div class="left"><span class="title">${title}</span><div class="meta">${sideVal} ${statusVal}</div></div>
      <div class="right"><span class="pnl-big ${pnlRaw>=0?'pos':'neg'}">${(pnlRaw*100).toFixed(2)}%</span></div>
    </div>
    <div class="mobile-card-body">${detailRows}</div>
  </div>`;
}).join('') + '</div>';

return table + cards;
```

## Pitfalls

- Do NOT make cards the only layout — desktop users need sortable columns
- Do NOT render all rows as cards with full details visible — progressive disclosure (tap to expand) prevents endless scrolling
- Do NOT use horizontal scroll as a fallback — users hate it
- Verify `scrollWidth > clientWidth` is false on mobile viewport after deploying
- **Before switching to cards, try column optimization first**: reorder columns so the most-scanned data (market name, side, P/L) is first; shorten timestamps from ISO to compact format (e.g. "06-26 15:52"); use short labels for strategy names; push wide metadata columns to the right where natural scrolling accommodates them. Cards are a last resort, not a first response to "the table is too wide."
