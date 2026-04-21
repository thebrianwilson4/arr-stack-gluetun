# arr-stack-gluetun

> A production-ready Docker Compose stack for Radarr, Sonarr, Prowlarr, Bazarr, Recyclarr, SABnzbd, and FlareSolverr — all routed through a Gluetun VPN tunnel.

Tested on Unraid 7.2.4. Works on any Linux host running Docker Compose.

---

## What's Included

| Service | Purpose | VPN |
|---|---|---|
| [Gluetun](https://github.com/qdm12/gluetun) | WireGuard VPN tunnel — all arr traffic routes through this | — |
| [Radarr](https://radarr.video) | Movie collection manager | ✅ Through VPN |
| [Sonarr](https://sonarr.tv) | TV show collection manager | ✅ Through VPN |
| [Prowlarr](https://prowlarr.com) | Indexer manager — syncs to Radarr/Sonarr | ✅ Through VPN |
| [Bazarr](https://bazarr.media) | Subtitle automation | ✅ Through VPN |
| [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) | Cloudflare bypass for protected indexers | ✅ Through VPN |
| [Lidarr](https://lidarr.audio) | Music collection manager | ✅ Through VPN |
| [Decluttarr](https://github.com/ManiMatter/decluttarr) | Stalled download cleaner | Clearnet |
| [Recyclarr](https://recyclarr.dev) | Syncs TRaSH Guide profiles to Radarr/Sonarr | Clearnet |
| [SABnzbd](https://sabnzbd.org) | Usenet download client | Clearnet |

---

## How the VPN Tunnel Works

This is the key design pattern that makes this stack different from running containers individually:

```
                    ┌─────────────────────────────────────┐
                    │           arr_net (bridge)          │
                    │                                     │
                    │  ┌─────────┐   Routes through VPN  │
Internet ──────────►│  │ Gluetun │◄──────────────────────┤
(WireGuard VPN)     │  └────┬────┘                       │
                    │       │  network_mode: service:     │
                    │  ┌────▼────┐  ┌──────────┐         │
                    │  │ Radarr  │  │  Sonarr  │         │
                    │  └─────────┘  └──────────┘         │
                    │  ┌──────────┐ ┌──────────┐         │
                    │  │ Prowlarr │ │  Bazarr  │         │
                    │  └──────────┘ └──────────┘         │
                    │  ┌─────────────┐                   │
                    │  │ FlareSolverr│                   │
                    │  └─────────────┘                   │
                    │                                     │
                    │  ┌───────────┐  ┌──────────────┐  │
                    │  │ Recyclarr │  │   SABnzbd    │  │
                    │  │(clearnet) │  │  (clearnet)  │  │
                    │  └───────────┘  └──────────────┘  │
                    └─────────────────────────────────────┘
```

**Three key things that make this work:**

1. **`network_mode: service:gluetun`** — Radarr, Sonarr, Prowlarr, Bazarr, and FlareSolverr share Gluetun's network namespace entirely. Their traffic has no path to the internet except through the VPN tunnel.

2. **`condition: service_healthy`** — Nothing starts until Gluetun confirms the VPN tunnel is up. If the VPN drops and Gluetun restarts, the dependent containers restart with it.

3. **`FIREWALL_OUTBOUND_SUBNETS`** — Allows arr containers to still communicate with local services (torrent clients, NAS, etc.) on your LAN without going through the VPN.

**Recyclarr and SABnzbd intentionally run on clearnet** — Recyclarr only communicates with Radarr/Sonarr on the internal Docker network, and Usenet traffic doesn't require VPN protection.

---

## Quick Start

### 1. Clone the repo

```bash
git clone https://github.com/thebrianwilson4/arr-stack-gluetun.git
cd arr-stack-gluetun
```

### 2. Create your .env file

```bash
cp .env.example .env
```

Edit `.env` and fill in your values — at minimum:
- Your WireGuard VPN credentials (see [docs/vpn-providers.md](docs/vpn-providers.md))
- Your paths (`APPDATA_PATH`, `MEDIA_PATH`, `DOWNLOADS_PATH`)
- Your LAN subnet (`LAN_SUBNET`)
- Your user/group IDs (`PUID`, `PGID`)

### 3. Create required directories

```bash
# Adjust paths to match your APPDATA_PATH and MEDIA_PATH
mkdir -p /opt/appdata/{gluetun,radarr,sonarr,prowlarr,bazarr,recyclarr,sabnzbd}
mkdir -p /mnt/media/{movies,shows}
mkdir -p /mnt/downloads
```

### 4. Start the stack

```bash
docker compose up -d
```

Watch Gluetun come up first — everything else waits for it:

```bash
docker compose logs -f gluetun
```

### 5. Verify VPN is active

```bash
docker exec radarr curl -s https://ifconfig.me
# Must return your VPN provider's IP, not your home IP
```

### 6. Configure the services

See [docs/post-install-config.md](docs/post-install-config.md) for step-by-step setup of each service.

---

## Port Reference

| Service | Port | URL |
|---|---|---|
| Radarr | 7878 | `http://YOUR_SERVER_IP:7878` |
| Sonarr | 8989 | `http://YOUR_SERVER_IP:8989` |
| Prowlarr | 9696 | `http://YOUR_SERVER_IP:9696` |
| Bazarr | 6767 | `http://YOUR_SERVER_IP:6767` |
| SABnzbd | 8090 | `http://YOUR_SERVER_IP:8090` |
| FlareSolverr | 8191 | Internal only (Prowlarr → Proxy) |
| Gluetun HTTP proxy | 8888 | Internal |

> **Note:** Because Radarr, Sonarr, Prowlarr, Bazarr, and FlareSolverr share Gluetun's network, they communicate with each other via `localhost`, not container names.

---

## VPN Provider Support

This stack uses Gluetun's `custom` WireGuard mode, which works with **any WireGuard-capable VPN provider**:

- ✅ ProtonVPN
- ✅ Mullvad
- ✅ AirVPN
- ✅ IVPN
- ✅ Any provider that generates a WireGuard config file

See [docs/vpn-providers.md](docs/vpn-providers.md) for provider-specific instructions.

---

## Media Directory Structure

Radarr and Sonarr both mount your media path as `/data` internally. Organize it however you like — here's the recommended layout:

```
/mnt/media/          ← your MEDIA_PATH
├── movies/
│   └── ...
└── shows/
    └── ...

/mnt/downloads/      ← your DOWNLOADS_PATH
└── ...              (completed downloads land here)
```

Both paths must be visible to Radarr/Sonarr for hardlinks to work (if your download client is local). If using a remote seedbox, hardlinks won't work — configure Radarr/Sonarr to copy instead.

---

## Unraid Notes

If running on Unraid via Compose Manager:

- Copy `docker-compose.yml` and `.env` to `/boot/config/plugins/compose.manager/projects/arr-stack/`
- `PUID=99`, `PGID=100` are the standard Unraid nobody user values
- Use `/mnt/user/appdata` for `APPDATA_PATH`
- Docker restart command: `/etc/rc.d/rc.docker restart` (not systemctl)

---

## Troubleshooting

See [docs/troubleshooting.md](docs/troubleshooting.md) for common issues:
- Gluetun not healthy
- VPN IP leaking
- FlareSolverr not working
- Downloads not importing
- LAN connectivity issues

---

## Credits

- [Gluetun](https://github.com/qdm12/gluetun) by qdm12 — the VPN container that makes all of this possible
- [TRaSH Guides](https://trash-guides.info) — essential reading for quality profiles and custom formats
- [Recyclarr](https://recyclarr.dev) — automates TRaSH Guide sync
- [LinuxServer.io](https://linuxserver.io) — maintainers of the Radarr, Sonarr, Prowlarr, Bazarr, and SABnzbd images
- [r/unraid](https://reddit.com/r/unraid) and [r/selfhosted](https://reddit.com/r/selfhosted) communities

---

## License

MIT — use freely, no warranty implied.
