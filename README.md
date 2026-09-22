# Home Assistant on a Raspberry Pi with Docker

A minimal, copy-and-run setup: Home Assistant Container behind an nginx reverse
proxy, on a Raspberry Pi running 64-bit Raspberry Pi OS. Two files matter:
`docker-compose.yml` and `nginx.conf`. Everything else is optional.

## What you get

- **Home Assistant** (`ghcr.io/home-assistant/home-assistant`, pinned version) in
  host-network mode, so device discovery (mDNS/SSDP), Matter, Thread and Bluetooth
  work like on a native install.
- **nginx** in front of it on port 80 — a stable entry point, and the place to add
  TLS later (see below).
- Everything survives reboots and power cuts with no extra service: Docker starts
  at boot and `restart: unless-stopped` brings the containers back.

## Requirements

- Raspberry Pi 4 or 5 (2 GB RAM minimum, 4 GB recommended) with a good SD card or,
  better, an SSD.
- Raspberry Pi OS **64-bit** (Lite is fine — no desktop needed).
- Docker Engine + the compose plugin.

## Quick start

```bash
# 1. Docker (once)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"      # log out and back in afterwards
sudo systemctl enable --now docker

# 2. Get this repo
git clone https://github.com/TimotejLabsky/ha-pi-docker.git
cd ha-pi-docker
cp .env.example .env                 # optional: change HA_CONFIG_DIR / TZ

# 3. Start
docker compose up -d
```

Open `http://<pi-ip>/` (nginx) or `http://<pi-ip>:8123/` (Home Assistant directly)
and finish onboarding. Your configuration lives in `./config` (or wherever
`HA_CONFIG_DIR` points) — back that directory up.

### Tell Home Assistant it sits behind a proxy

Once onboarding is done, add this to `config/configuration.yaml` and restart the
container (`docker compose restart homeassistant`). Without it, HA refuses proxied
requests with a `400 Bad Request` and logs `A request from a reverse proxy was
received … but your HTTP integration is not set-up for reverse proxies`.

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - ::1
```

## Why there is no systemd unit

Docker itself is the boot service. With `restart: unless-stopped` the containers
come back after every reboot or power loss — including ones that happened
mid-write — as long as Docker is enabled (`systemctl enable docker`, done above).
A `docker compose up -d` unit on top of that only re-does what Docker already did,
and it breaks the moment the checkout moves. The one legitimate reason for a unit
is ordering against something Docker cannot see (for example `/config` on an NFS
mount that comes up late); if you need that, add `RequiresMountsFor=` to a small
unit rather than reintroducing the compose one.

## USB sticks (Zigbee, Z-Wave, Bluetooth)

Do **not** run the container privileged. Pass the device through instead — find it
with `ls -l /dev/serial/by-id/` and uncomment the `devices:` block in
`docker-compose.yml`:

```yaml
    devices:
      - /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus-if00-port0:/dev/ttyUSB0
```

The `by-id` path is stable across reboots; `/dev/ttyUSB0` is not.

## HTTPS / remote access

The Home Assistant companion apps and many integrations want HTTPS. Options, from
simplest:

1. **Keep it LAN-only** and reach it over a VPN (WireGuard on the router or
   Tailscale) — nothing to expose, nothing to renew. This is what nginx on :80 is
   sized for.
2. **Caddy instead of nginx** with a real domain: replace the `nginx` service with
   `caddy:2` and a two-line `Caddyfile` (`ha.example.com { reverse_proxy
   localhost:8123 }`); Caddy fetches and renews Let's Encrypt certificates
   itself. Requires port 80/443 forwarded to the Pi and a DNS record.
3. **Nabu Casa** (Home Assistant Cloud) — paid, zero configuration, supports the
   project.

## Updating

The image tag is pinned on purpose: Home Assistant ships monthly and breaking
changes are announced per release. To update:

```bash
# edit docker-compose.yml: image tag -> the version from https://github.com/home-assistant/core/releases
docker compose pull
docker compose up -d
```

Read the release notes' "Breaking changes" section first, and take a copy of
`./config` before a major jump. Avoid `:latest`/`:stable` on a device you cannot
easily roll back.

## SD-card longevity

Container logs are capped in `docker-compose.yml` (`logging:`), so a chatty
integration cannot fill the card. Consider moving `HA_CONFIG_DIR` to a USB SSD, and
enable Home Assistant's `recorder` purge (default 10 days) — the SQLite database is
the biggest writer.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Home Assistant + nginx |
| `nginx.conf` | reverse proxy with WebSocket support (needed by the HA frontend) |
| `.env.example` | `HA_CONFIG_DIR`, `TZ` — copy to `.env` |

Originally a corner of a private infrastructure repo; split out so it can be
shared as-is.
