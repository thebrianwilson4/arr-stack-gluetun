# Post-Install Configuration

After the stack is running, configure each service in order.

---

## 1. Prowlarr — Indexer Manager

Prowlarr is the hub — configure it first, then it syncs indexers to Radarr and Sonarr automatically.

**Access:** `http://YOUR_SERVER_IP:9696`

### Add your indexers

Settings → Indexers → Add Indexer. Search for your indexer by name.

**Priority guide** (lower number = higher priority):

| Type | Suggested Priority |
|---|---|
| Private tracker (ratio-based) | 5–10 |
| Semi-private / ratioless | 15–20 |
| Public trackers | 25–30 |
| Usenet | 25 (Interactive Search only recommended) |

### Add FlareSolverr proxy

For Cloudflare-protected indexers (TorrentGalaxy, HD-Space, etc.):

Settings → Indexers → Add Proxy → FlareSolverr
- URL: `http://localhost:8191`
- Assign to any Cloudflare-protected indexers

### Connect to Radarr and Sonarr

Settings → Apps → Add Application:

| Field | Radarr | Sonarr |
|---|---|---|
| Server URL | `http://localhost:7878` | `http://localhost:8989` |
| API Key | (from Radarr Settings → General) | (from Sonarr Settings → General) |
| Sync Level | Full Sync | Full Sync |

---

## 2. Radarr — Movie Manager

**Access:** `http://YOUR_SERVER_IP:7878`

### Root folder

Settings → Media Management → Add Root Folder → `/data/movies`

### Download client

Settings → Download Clients → Add:
- For **torrent**: add your torrent client (qBittorrent, rTorrent, Deluge, etc.)
- For **Usenet**: add SABnzbd at `http://YOUR_SERVER_IP:8090`

> If using a remote seedbox, use the seedbox's hostname/IP and credentials instead.

### Quality profiles

Recommended: install [Recyclarr](https://recyclarr.dev/wiki/) and sync TRaSH Guide profiles automatically. See [Recyclarr setup](#5-recyclarr).

### Recycling bin

Settings → Media Management → Recycling Bin: `/config/recycling`

This keeps deleted files for recovery before permanent deletion.

### Import Extra Files

Settings → Media Management → Import Extra Files: `srt,nfo`

---

## 3. Sonarr — TV Show Manager

**Access:** `http://YOUR_SERVER_IP:8989`

Same setup as Radarr — root folder, download client, quality profiles, recycling bin.

Root folder: `/data/shows`

---

## 4. Bazarr — Subtitle Manager

**Access:** `http://YOUR_SERVER_IP:6767`

Settings → Sonarr → connect to `http://localhost:8989`
Settings → Radarr → connect to `http://localhost:7878`

Then configure your subtitle providers (OpenSubtitles, Subscene, etc.) and language profiles.

---

## 5. Recyclarr

Recyclarr automatically syncs [TRaSH Guide](https://trash-guides.info) quality profiles and custom formats into Radarr and Sonarr.

Create `${APPDATA_PATH}/recyclarr/recyclarr.yml`:

```yaml
# Minimal example — see https://recyclarr.dev/wiki/ for full reference
radarr:
  movies:
    base_url: http://localhost:7878
    api_key: YOUR_RADARR_API_KEY
    quality_definition:
      type: movie
    quality_profiles:
      - name: Any

sonarr:
  series:
    base_url: http://localhost:8989
    api_key: YOUR_SONARR_API_KEY
    quality_definition:
      type: series
```

Manual sync:
```bash
docker exec recyclarr recyclarr sync
```

Automatic sync runs daily at 3 AM via the `CRON_SCHEDULE` env var.

---

## 6. SABnzbd — Usenet Client (optional)

**Access:** `http://YOUR_SERVER_IP:8090`

Only needed if you use Usenet. Add your Usenet provider(s) under Config → Servers.

Recommended settings:
- SSL on port 563
- 20–50 connections depending on your provider plan
- Set a download speed limit to avoid saturating your connection

---

## Verifying Everything Works

```bash
# Confirm VPN is active for arr containers
docker exec radarr curl -s https://ifconfig.me
# Must return your VPN provider's IP, not your home IP

# Check all containers are healthy
docker ps --format "table {{.Names}}\t{{.Status}}"

# Test Prowlarr indexer search
# Prowlarr → Indexers → Search icon on any indexer
```
