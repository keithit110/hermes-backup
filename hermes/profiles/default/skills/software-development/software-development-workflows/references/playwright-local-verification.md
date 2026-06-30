# Playwright Local Verification

Use when Browserbase can't reach the VPS (502 on non-standard ports, VPN/proxy blocking) but you need to prove a UI change is live.

## Setup (one-time)

```bash
pip3 install playwright
python3 -m playwright install chromium
```

## Verification script template

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    b = p.chromium.launch(headless=True, args=['--no-sandbox','--disable-gpu','--single-process'])
    page = b.new_page(viewport={'width': 390, 'height': 844})
    page.goto('http://localhost:8095/', timeout=15000, wait_until='networkidle')
    page.wait_for_timeout(4000)  # allow JS loadAll() to complete

    # Verify data loaded
    status = page.locator('#status').text_content()
    print(f'Status: {status}')  # Should show "Updated ..." not "Loading..."

    stat_vals = page.locator('.stat-val')
    for i in range(stat_vals.count()):
        print(f'Stat {i}: {stat_vals.nth(i).text_content()}')

    # Click Paper Bets tab
    page.locator('button[data-tab="paper"]').click()
    page.wait_for_timeout(2000)

    # Count table rows
    rows = page.locator('#paperTable table tbody tr')
    print(f'Table rows: {rows.count()}')

    # Screenshot
    page.screenshot(path='/tmp/pm-home.png')

    b.close()
```

## Quick JS syntax check

After any template change that touches JavaScript:

```bash
curl -s http://localhost:8095/ | sed -n '/<script>/,/<\/script>/p' | sed '1d;$d' > /tmp/js.js
node -c /tmp/js.js  # Must return 0
```

## Docker rebuild checklist

```bash
cd /root/polymarket-intel
docker compose build --no-cache web    # MUST be --no-cache if source changed
docker compose stop web
docker compose up -d web
# Verify
curl -s -o /dev/null -w '%{http_code}' http://localhost:8095/
# Then run Playwright or node -c
```

## Pitfalls

- `docker compose restart` does NOT pick up new images — use `stop` then `up -d`
- Docker COPY layer caching: a successful build with "CACHED" on the COPY step is serving OLD code
- `curl | grep` only proves Flask served the file — it does NOT prove JS executed or CSS rendered
- The raw Python string `r"""..."""` in web.py preserves literal backslashes in JS output