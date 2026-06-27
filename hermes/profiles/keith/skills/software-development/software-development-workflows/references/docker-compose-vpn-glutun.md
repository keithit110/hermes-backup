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
