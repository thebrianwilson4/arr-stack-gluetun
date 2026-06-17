# arr-stack-gluetun

A self-hosted media server stack built on Docker Compose. Automatically finds, downloads, and organizes movies, TV shows, music, and audiobooks — all routed through a WireGuard VPN so your download traffic stays private.

Tested on Unraid 7.x. Works on any Linux host running Docker Compose.

---

## What You Get

At its core, this stack gives you:

- **Radarr** — finds and manages your movie collection
- **Sonarr** — finds and manages your TV shows
- **Prowlarr** — connects to torrent and Usenet indexers so Radarr/Sonarr know where to look
- **Bazarr** — automatically downloads subtitles for everything
- **SABnzbd** — Usenet download client
- **Recyclarr** — keeps your quality settings in sync with community best practices
- **Decluttarr** — cleans up stalled or failed downloads automatically
- **Autoheal** — restarts any container that crashes or becomes unresponsive
- **Gluetun** — routes all download traffic through your WireGuard VPN

Everything is connected and talks to each other automatically once configured.

---

## Optional Add-Ons

You can turn these on or off in a single line in your `.env` file — no editing the compose file required.

| Add-on | What it does | Profile name |
| --- | --- | --- |
| [Lidarr](https://lidarr.audio) | Finds and manages your music collection | `music` |
| [Bookshelf](https://github.com/pennydreadful/bookshelf) | Automated audiobook acquisition | `audiobooks` |
| [Subgen](https://github.com/McCloudS/subgen) | Generates subtitles from audio using AI (NVIDIA GPU required) | `subtitles` |

Set `COMPOSE_PROFILES` in your `.env` to enable what you want:

```bash
COMPOSE_PROFILES=                           # movies + shows only (default)
COMPOSE_PROFILES=music                      # + music
COMPOSE_PROFILES=music,audiobooks           # + music + audiobooks
COMPOSE_PROFILES=music,audiobooks,subtitles # everything
```

---

## How the VPN Works

All download traffic routes through a single Gluetun container running WireGuard. Radarr, Sonarr, Prowlarr, Bazarr, Lidarr, FlareSolverr, and Bookshelf share Gluetun's network — they have no path to the internet except through the VPN tunnel. If the VPN goes down, the containers stop making requests until it recovers.

```
                    ┌──────────────────────────────────────────┐
                    │            arr_net (bridge)              │
                    │                                          │
                    │  ┌─────────┐    Routes through VPN      │
Internet ──────────►│  │ Gluetun │◄───────────────────────────┤
(WireGuard VPN)     │  └────┬────┘                            │
                    │       │  shares network namespace        │
                    │  ┌────▼────┐  ┌──────────┐              │
                    │  │ Radarr  │  │  Sonarr  │              │
                    │  └─────────┘  └──────────┘              │
                    │  ┌──────────┐  ┌──────────┐             │
                    │  │ Prowlarr │  │  Bazarr  │             │
                    │  └──────────┘  └──────────┘             │
                    │  ┌──────────┐  ┌─────────────┐          │
                    │  │  Lidarr  │  │ FlareSolverr│  (opt.)  │
                    │  └──────────┘  └─────────────┘          │
                    │  ┌────────────┐                          │
                    │  │ Bookshelf  │                  (opt.)  │
                    │  └────────────┘                          │
                    │                                          │
                    │  ┌───────────┐  ┌──────────────┐       │
                    │  │ Recyclarr │  │   SABnzbd    │       │
                    │  │(clearnet) │  │  (clearnet)  │       │
                    │  └───────────┘  └──────────────┘       │
                    │  ┌────────────┐  ┌──────────────┐      │
                    │  │ Decluttarr │  │    Subgen    │      │
                    │  │(clearnet)  │  │  (opt.)      │      │
                    │  └────────────┘  └──────────────┘      │
                    └──────────────────────────────────────────┘
                         Autoheal monitors all containers
```

SABnzbd, Recyclarr, and Decluttarr run on the regular network — Usenet traffic doesn't need VPN protection, and they only communicate with the other arr apps internally.

---

## Before You Start

You'll need:

1. **A server running Docker and Docker Compose** — Linux recommended. Unraid works great with [Compose Manager](https://forums.unraid.net/topic/114415-plugin-compose-manager/).
2. **A WireGuard VPN subscription** — [ProtonVPN](https://protonvpn.com), [Mullvad](https://mullvad.net), [AirVPN](https://airvpn.org), or any WireGuard-capable provider. You'll need to download a WireGuard config file from their website.
3. **A Usenet provider** (optional, for SABnzbd) — e.g. [UsenetPrime](https://www.usenetprime.com), Newshosting, Eweka.
4. **An NVIDIA GPU** (only if using the `subtitles` profile).

---

## Setup

### 1. Clone this repo

```bash
git clone https://github.com/thebrianwilson4/arr-stack-gluetun.git
cd arr-stack-gluetun
```

### 2. Create your config file

```bash
cp .env.example .env
```

Open `.env` in a text editor and fill in:

- Your WireGuard VPN credentials — see [docs/vpn-providers.md](docs/vpn-providers.md) for where to find these
- Your paths (`APPDATA_PATH`, `MEDIA_PATH`, `DOWNLOADS_PATH`)
- Your home network subnet (`LAN_SUBNET`)
- Which optional services you want (`COMPOSE_PROFILES`)

### 3. Create directories

```bash
# Core (always required)
mkdir -p /opt/appdata/{gluetun,radarr,sonarr,prowlarr,bazarr,recyclarr,sabnzbd,decluttarr}
mkdir -p /mnt/media/{movies,shows}
mkdir -p /mnt/downloads

# Only if using COMPOSE_PROFILES=music
mkdir -p /opt/appdata/lidarr /mnt/media/music

# Only if using COMPOSE_PROFILES=audiobooks
mkdir -p /opt/appdata/bookshelf /mnt/media/audiobooks

# Only if using COMPOSE_PROFILES=subtitles
mkdir -p /opt/appdata/subgen/models
```

Adjust paths to match `APPDATA_PATH` and `MEDIA_PATH` in your `.env`.

### 4. Start the stack

```bash
docker compose up -d
```

Watch Gluetun connect to the VPN first — everything else waits for it:

```bash
docker compose logs -f gluetun
```

You should see something like `connected to <server>` within 30 seconds.

### 5. Confirm VPN is active

```bash
docker exec radarr curl -s https://ifconfig.me
```

This must return your VPN provider's IP — **not your home IP**. If it returns your home IP, stop and check [docs/troubleshooting.md](docs/troubleshooting.md) before proceeding.

### 6. Configure each service

See [docs/post-install-config.md](docs/post-install-config.md) for step-by-step setup of Prowlarr, Radarr, Sonarr, SABnzbd, Recyclarr, and the optional services.

---

## Ports

| Service | Port | How to access |
| --- | --- | --- |
| Radarr | 7878 | `http://YOUR_SERVER_IP:7878` |
| Sonarr | 8989 | `http://YOUR_SERVER_IP:8989` |
| Prowlarr | 9696 | `http://YOUR_SERVER_IP:9696` |
| Bazarr | 6767 | `http://YOUR_SERVER_IP:6767` |
| SABnzbd | 8090 | `http://YOUR_SERVER_IP:8090` |
| Lidarr | 8686 | `http://YOUR_SERVER_IP:8686` *(music profile)* |
| Bookshelf | 8787 | `http://YOUR_SERVER_IP:8787` *(audiobooks profile)* |
| Subgen | 9000 | `http://YOUR_SERVER_IP:9000` *(subtitles profile)* |
| FlareSolverr | 8191 | Internal only — configured in Prowlarr |

> Radarr, Sonarr, Prowlarr, Bazarr, Lidarr, FlareSolverr, and Bookshelf all share Gluetun's network. When connecting them to each other in their settings, use `localhost` — not container names.

---

## VPN Provider Support

This stack uses Gluetun's `custom` WireGuard mode, which works with any WireGuard-capable provider:

- ✅ ProtonVPN
- ✅ Mullvad
- ✅ AirVPN
- ✅ IVPN
- ✅ Any provider that generates a WireGuard `.conf` file

See [docs/vpn-providers.md](docs/vpn-providers.md) for how to extract the right values from your provider's config file.

---

## Media Directory Structure

```
/mnt/media/              ← MEDIA_PATH
├── movies/
├── shows/
├── music/               ← music profile only
└── audiobooks/          ← audiobooks profile only

/mnt/downloads/          ← DOWNLOADS_PATH
└── ...

/mnt/cache/downloads/    ← CACHE_DOWNLOADS_PATH (optional — SSD staging)
└── ...
```

Radarr, Sonarr, and Lidarr need both the media directory and the downloads directory visible inside the container for hardlinks to work correctly. Hardlinks let the arr apps "move" files instantly without copying them — a significant time and disk I/O saver.

If you're pulling from a remote seedbox rather than downloading locally, hardlinks won't apply — configure Radarr/Sonarr to use remote path mappings instead.

---

## Unraid Notes

- Copy `docker-compose.yml` and `.env` to `/boot/config/plugins/compose.manager/projects/arr-stack/`
- `PUID=99`, `PGID=100` are correct for Unraid (already the defaults in `.env.example`)
- Use `/mnt/user/appdata` for `APPDATA_PATH`
- Use `/mnt/user/media` for `MEDIA_PATH`
- Use `/mnt/cache/downloads` for `CACHE_DOWNLOADS_PATH` if you have a cache drive; otherwise set it the same as `DOWNLOADS_PATH`
- Docker restart: `/etc/rc.d/rc.docker restart` (not `systemctl`)

---

## Troubleshooting

See [docs/troubleshooting.md](docs/troubleshooting.md) for help with:

- Gluetun not connecting / containers not starting
- VPN check returning your home IP
- FlareSolverr not working
- Downloads not importing into Radarr/Sonarr
- Can't reach local devices from inside containers
- Recyclarr failing to sync

---

## Credits

- [Gluetun](https://github.com/qdm12/gluetun) by qdm12 — the VPN container everything runs through
- [TRaSH Guides](https://trash-guides.info) — quality profile recommendations
- [Recyclarr](https://recyclarr.dev) — automates TRaSH Guide sync
- [LinuxServer.io](https://linuxserver.io) — Docker images for Radarr, Sonarr, Prowlarr, Bazarr, Lidarr, SABnzbd
- [ManiMatter/decluttarr](https://github.com/ManiMatter/decluttarr) — stalled download cleanup
- [pennydreadful/bookshelf](https://github.com/pennydreadful/bookshelf) — audiobook automation
- [McCloudS/subgen](https://github.com/McCloudS/subgen) — Whisper-based subtitle generation
- [willfarrell/docker-autoheal](https://github.com/willfarrell/docker-autoheal) — container health management
- [r/unraid](https://reddit.com/r/unraid) and [r/selfhosted](https://reddit.com/r/selfhosted) communities

---

## License

MIT — use freely, no warranty implied.
