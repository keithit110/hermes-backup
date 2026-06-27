# VPN Proxy Setup for Polymarket Intel

Added 2026-06-26. Ensures ALL Docker containers (not just crypto engine) route Polymarket API calls through UK VPN.

## Problem

Before this change, only the `crypto` container used `network_mode: service:gluetun` (WireGuard tunnel). The `scanner` (which runs Copy Wallet, Copy Consensus, and Research Buy strategies) and `web` containers were on the default Docker bridge network — their outbound Polymarket API calls (`gamma-api.polymarket.com`, `clob.polymarket.com`, `data-api.polymarket.com`) came from the VPS's US IP.

When Keith switches to real trading, Polymarket blocks US IPs. All order placement must come from a non-US IP.

## Solution: gluetun's built-in HTTP proxy

Gluetun includes a built-in HTTP proxy that routes traffic through the VPN tunnel. Any container on the same Docker network can use it by setting `HTTP_PROXY` and `HTTPS_PROXY` environment variables.

### Step 1: Enable proxy in `.env.vpn`

Add to `/root/polymarket-intel/.env.vpn`:

```
HTTPPROXY=on
HTTPPROXY_LISTENING_ADDRESS=:8888
```

Without this, gluetun's proxy port returns network errors (wget exit code 4 → changed to exit code 8 after enabling — exit code 8 means proxy is running, just not handling direct GET without proxy headers, which is correct).

### Step 2: Add proxy env vars to docker-compose.yml

Each service that makes Polymarket API calls:

```yaml
environment:
  <<: *polymarket-env
  HTTP_PROXY: "http://gluetun:8888"
  HTTPS_PROXY: "http://gluetun:8888"
  NO_PROXY: "localhost,127.0.0.1,.local"
```

`NO_PROXY` prevents the proxy from intercepting local DB/file access.

### Step 3: Restart services

```bash
cd /root/polymarket-intel
docker compose up -d gluetun --force-recreate
# Wait for healthy
docker compose up -d web crypto --force-recreate
```

## Why no code changes needed

Python's `requests` library automatically reads `HTTP_PROXY` and `HTTPS_PROXY` from the environment. The scanner (`app/main.py`) and crypto engine (`app/crypto_engine.py`) both use `requests` for all Polymarket API calls. No import or session configuration needed.

## Verification results (2026-06-26)

| Container | Method | Exit IP | City | Country |
|-----------|--------|---------|------|---------|
| crypto | WireGuard tunnel | 185.44.76.189 | London | GB |
| scanner (test container) | HTTP proxy | 185.44.76.189 | London | GB |
| web (test container) | HTTP proxy | 185.44.76.189 | London | GB |

Test command for proxy-routed containers:

```bash
docker run --rm --network polymarket-intel_default \
  -e HTTP_PROXY=http://gluetun:8888 \
  -e HTTPS_PROXY=http://gluetun:8888 \
  python:3.11-slim sh -c 'pip install -q requests; python3 -c "
import requests
r = requests.get(\"https://ipinfo.io/json\", timeout=10)
print(r.json().get(\"city\"), r.json().get(\"country\"), r.json().get(\"ip\"))
"'
```

## Caveats

- **Gluetun HTTP proxy exit IP may differ from WireGuard tunnel IP.** In testing, the WireGuard tunnel exited via 103.138.124.11 (Manchester) while the HTTP proxy exited via 185.44.76.189 (London). Both are UK — acceptable for geoblocking evasion. The difference is because gluetun's HTTP proxy uses a separate routing path.
- **WebSocket connections do NOT go through HTTP proxy.** The crypto engine's WebSocket connections (Polymarket RTDS, Binance) use the WireGuard tunnel directly via `network_mode: service:gluetun`. HTTP proxy only handles REST API calls.
- **Scanner is one-shot.** Its proxy env vars are set in `docker-compose.yml` under the `scanner` service definition. Each invocation (`docker compose up scanner` or manual `docker compose run`) picks them up automatically.
