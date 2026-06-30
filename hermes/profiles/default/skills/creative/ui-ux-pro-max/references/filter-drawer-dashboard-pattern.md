# Filter Drawer Dashboard Pattern

## When to use

When a dashboard has filter controls (dropdowns, checkboxes, date pickers) that consume permanent vertical space and the user wants to see the data table sooner without scrolling. Especially effective on mobile where every row of chrome pushes the table further down.

## Pattern

Replace the full-time filter bar with a **filter icon button** in the section header. Clicking it opens a **slide-in drawer** from the right side with a semi-transparent overlay backdrop. All filter controls live inside the drawer. Filter changes apply immediately; no separate "Apply" step needed (simpler UX).

## Anatomy

```
Section header:  [Title] [subtitle] [(#) Filters btn]
                                        ↓ click
Drawer (slides from right):
  ┌─────────────────────────────┐
  │  Filters               ✕   │  ← header with close
  ├─────────────────────────────┤
  │  Strategy   [dropdown    ]  │
  │  Status     [dropdown    ]  │  ← filter controls
  │  Side       [dropdown    ]  │
  ├─────────────────────────────┤
  │  [Clear]         [Done]     │  ← footer actions
  └─────────────────────────────┘
  (overlay behind — click to close)
```

## Key CSS

```css
/* Overlay: full-screen, fade in/out */
.drawer-overlay {
  position: fixed; inset: 0;
  background: rgba(0,0,0,.5);
  z-index: 100;
  opacity: 0; visibility: hidden;
  transition: opacity .2s, visibility .2s;
}
.drawer-overlay.open { opacity: 1; visibility: visible; }

/* Panel: slides from right */
.drawer-panel {
  position: fixed; top: 0; right: 0; bottom: 0;
  width: 340px; max-width: 90vw;
  background: var(--surface);
  z-index: 101;
  transform: translateX(100%);
  transition: transform .25s cubic-bezier(.4,0,.2,1);
  display: flex; flex-direction: column;
  box-shadow: -8px 0 30px rgba(0,0,0,.4);
}
.drawer-panel.open { transform: translateX(0); }

/* Mobile: full-width drawer */
@media(max-width: 900px) {
  .drawer-panel { width: 100%; max-width: 100%; }
}
```

## Key JS

```javascript
function toggleDrawer() {
  let panel = document.getElementById('drawerPanel');
  let overlay = document.getElementById('drawerOverlay');
  if (panel.classList.contains('open')) { closeDrawer(); }
  else { openDrawer(); }
}
function openDrawer() {
  document.getElementById('drawerPanel').classList.add('open');
  document.getElementById('drawerOverlay').classList.add('open');
}
function closeDrawer() {
  document.getElementById('drawerPanel').classList.remove('open');
  document.getElementById('drawerOverlay').classList.remove('open');
}
// Escape key closes drawer
document.addEventListener('keydown', e => {
  if (e.key === 'Escape') closeDrawer();
});
```

## Active filter badge

Show a small blue badge on the filter button indicating how many filters are active:

```javascript
function updateFilterBadge() {
  let f = getPaperFilters();
  let count = (f.strategy ? 1 : 0) + (f.status ? 1 : 0) + (f.side ? 1 : 0);
  let badge = document.getElementById('filterCount');
  if (count > 0) { badge.style.display = 'flex'; badge.textContent = count; }
  else { badge.style.display = 'none'; }
}
// Call on each filter change AND after resetPaperFilters()
```

## Edge cases

- **Mobile**: drawer goes full-width (`width: 100%; max-width: 100%`) since 340px panels feel cramped on phones
- **Keyboard**: Escape key closes drawer
- **Click-outside**: overlay click closes drawer (`onclick="closeDrawer()"`)
- **Filter changes**: fire immediately on `onchange` (no "Apply" step); simpler UX for dropdowns
- **State preservation**: filter values persist in the select elements even when drawer is closed

## Flask inline-template specific

When the dashboard HTML lives inside a Python raw string (`PAGE = r"""..."""`), the safest overhaul approach:

1. Write the new template to `/tmp/new-template.html`
2. Assemble the new Python file: `head -37 web.py` (pre-PAGE) + `PAGE = r"""` + new template + `"""` + `tail -n +490 web.py` (Flask routes)
3. Syntax-check: `python3 -c "compile(open(...).read(), 'web.py', 'exec')"`
4. Backup original: `cp web.py web.py.bak`
5. Swap in new file
6. Rebuild Docker: `docker compose build web` (COPY layer picks up changed file)
7. Restart: `docker compose up -d web` (**NOT `restart`** — `restart` reuses the old image; `up -d` recreates the container with the new image)
8. Verify: `curl -s localhost:8095/ | grep -c 'drawer-overlay'`

## Pitfall: `docker compose restart` doesn't pick up new images

After `docker compose build web`, `docker compose restart web` keeps the OLD image — it stops and starts the same container. You MUST use `docker compose up -d web` (or `docker compose stop web && docker compose up -d web`) to recreate the container with the new image.

## Pitfall: Browser tool cannot reach VPN-protected VPS

If the web service sits behind a VPN proxy (gluetun, WireGuard), external browser tools (Browserbase) will get 502 errors. Fall back to **curl-based verification** from the VPS itself:
```bash
curl -s http://localhost:8095/ | grep -c 'new-feature-class'
```
This is the ground truth for deployments. Keith verifies on his phone separately.
