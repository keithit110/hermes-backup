# Docker Compose VPN routing (gluetun + Surfshark)

When a container needs to appear from a non-US IP (e.g., to access Binance WebSocket which geoblocks US IPs), route the container through **gluetun**.

## docker-compose.yml pattern

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    cap_add: [NET_ADMIN]
    environment:
      VPN_SERVICE_PROVIDER: surfshark
      VPN_TYPE: wireguard
      WIREGUARD_PRIVATE_KEY: ${WIREGUARD_PRIVATE_KEY}
      WIREGUARD_ADDRESSES: ${WIREGUARD_ADDRESSES}
      SERVER_COUNTRIES: United Kingdom
    # expose ports through gluetun if the vpn-routed container needs them

  app:
    network_mode: "service:gluetun"  # inherits VPN IP stack
    depends_on:
      gluetun:
        condition: service_healthy
```

## Surfshark WireGuard credentials

From Surfshark → Manual Setup → WireGuard. The config file provides:

```ini
[Interface]
PrivateKey = <your-key>
Address = 10.14.0.2/16
DNS = 162.252.172.57, 149.154.159.92

[Peer]
PublicKey = <server-key>
Endpoint = uk-lon.prod.surfshark.com:51820
```

Store credentials in `.env.vpn`:

```text
WIREGUARD_PRIVATE_KEY=<your-key>
WIREGUARD_ADDRESSES=10.14.0.2/16
VPN_SERVICE_PROVIDER=surfshark
VPN_TYPE=wireguard
SERVER_COUNTRIES=United Kingdom
```

Run with: `docker compose --env-file .env.vpn up -d`

## Pitfalls

- **Country names must be full:** "United Kingdom" not "uk", "Netherlands" not "nl". Wrong name → container fails health check with "country specified is not valid".
- **Verify the VPN IP:** `docker exec <container> curl -s ifconfig.me`
- **gluetun DNS leaks:** the crypto engine's Binance WS connection will fail with HTTP 451 (geoblocked) if VPN isn't routing properly. Check logs for `Handshake status 451`.

## Polymarket CLOB geoblock — VPN country MUST be allowed

Polymarket's CLOB API accepts connections from anywhere but **rejects all order placement with HTTP 403 when the user's country is restricted**. This is checked via Cloudflare's `CF-IPCountry` header — the VPN exit IP's GeoIP location determines access.

### Blocked countries (orders return 403)
| Country | Reason |
|---------|--------|
| **United States** | Primary regulatory ban |
| **United Kingdom** | UK Gambling Commission action (2025-2026) |
| **France** | Local regulatory action |
| OFAC-sanctioned countries | Iran, North Korea, Cuba, Syria, Crimea, etc. |

### Allowed countries (known working)
Netherlands, Switzerland, Canada, Germany, Spain, Italy, Japan, Singapore, Australia, Brazil, Mexico — most of EU, LatAm, and APAC.

### Diagnosis — check if your orders are geoblocked
```bash
# Check live_orders.jsonl for 403 errors
docker exec polymarket-intel-crypto tail -20 /logs/live_orders.jsonl | python3 -c "
import sys,json
for line in sys.stdin:
    d = json.loads(line.strip())
    if d.get('status') == 'error' and 'geoblock' in str(d.get('error','')):
        print(f\"GEOBLOCKED: trade_id={d['trade_id']} price={d['price']}\")
"
```

### Fix
Change `SERVER_COUNTRIES` in `.env.vpn` to an allowed country and restart:
```bash
# .env.vpn
SERVER_COUNTRIES=Netherlands   # or Switzerland, Canada

# Then:
cd /root/polymarket-intel
docker compose stop crypto gluetun
docker compose up -d gluetun
# Wait for gluetun to become healthy
docker compose up -d crypto
# Verify new IP
docker exec polymarket-intel-crypto python3 -c "
import urllib.request; print(urllib.request.urlopen('https://api.ipify.org').read().decode())
"
```

**Pitfall — `SERVER_COUNTRIES=United Kingdom` silently blocks ALL live trading.** The engine initializes, connects to CLOB, logs `[LIVE] ClobClient initialised`, and even detects entry opportunities — but every `place_buy_limit()` call returns 403. The only symptom is entries in `live_orders.jsonl` with status=error and "Trading restricted in your region". The engine logs show `[SCALP DBG]` evaluations happening correctly, and paper trades continue fine — the geoblock is invisible unless you check `live_orders.jsonl`.

## Multiple service pattern

When one container needs VPN but others don't:

```yaml
services:
  gluetun:     # VPN-only gateway
  crypto:      # network_mode: "service:gluetun" → through VPN
  web:         # normal host network → no VPN
  scanner:     # normal host network → no VPN
```

The scanner and web dashboard stay on the host network — only the crypto engine goes through VPN.
