# Troubleshooting

Common issues and how to fix them.

---

## Gluetun not healthy / containers won't start

All arr containers depend on Gluetun being healthy before they start. If Gluetun fails, nothing else comes up.

```bash
# Check Gluetun logs
docker logs Gluetun

# Check health status
docker inspect Gluetun --format='{{.State.Health.Status}}'
```

**Common causes:**

| Symptom | Fix |
|---|---|
| `authentication failure` | Wrong private or public key — regenerate your WireGuard config |
| `dial udp: connect: network is unreachable` | Endpoint IP is wrong or unreachable |
| `permission denied` | Missing `cap_add: NET_ADMIN` or `privileged: true` |
| Container keeps restarting | Check `WIREGUARD_ADDRESSES` — must match what your provider assigned |

---

## VPN not routing traffic (home IP leaks)

```bash
docker exec radarr curl -s https://ifconfig.me
```

If this returns your real home IP instead of a VPN IP:

1. Confirm `network_mode: service:gluetun` is set on the container
2. Confirm Gluetun is healthy: `docker inspect Gluetun --format='{{.State.Health.Status}}'`
3. Restart the full stack: `docker compose down && docker compose up -d`

---

## FlareSolverr not solving Cloudflare challenges

FlareSolverr runs inside the Gluetun network. In Prowlarr, the proxy URL must be:

```
http://localhost:8191
```

**Not** `http://flaresolverr:8191` — because FlareSolverr shares the Gluetun network namespace, `localhost` is the correct address from Prowlarr's perspective.

Check FlareSolverr is running:
```bash
docker logs flaresolverr
curl http://localhost:8191/health
```

---

## Prowlarr can't reach Radarr / Sonarr

Prowlarr syncs indexers to Radarr and Sonarr over the internal network. In Prowlarr → Settings → Apps, use:

```
http://localhost:7878   # Radarr
http://localhost:8989   # Sonarr
```

Again, `localhost` — not container names — because they all share Gluetun's network.

---

## Downloads not importing

The most common cause is a volume mapping mismatch. Radarr and Sonarr need to see the **same path** for both the media library and downloads.

Check your volume mappings in `docker-compose.yml` — the path inside the container must be consistent between where files are downloaded and where the arr app expects to find them.

```bash
# See what paths Radarr sees
docker exec radarr ls /downloads
docker exec radarr ls /data
```

If using a remote seedbox with Syncthing, files land at a different path than direct downloads — make sure both paths are mounted and Radarr/Sonarr are configured to use the right remote path mapping.

---

## Recyclarr failing to sync

```bash
# Run manually and see output
docker exec recyclarr recyclarr sync

# Check logs
docker logs recyclarr
```

Common cause: Radarr/Sonarr API keys not set in `recyclarr.yml`. The config file lives at `${APPDATA_PATH}/recyclarr/recyclarr.yml` — see the [Recyclarr wiki](https://recyclarr.dev/wiki/) for setup.

---

## Container can't reach LAN devices

If arr containers can't reach your torrent client or NAS on the local network, check `FIREWALL_OUTBOUND_SUBNETS` in your `.env`.

It must match your actual LAN subnet:

```bash
# Find your LAN subnet
ip route | grep -v default
```

Set `LAN_SUBNET` in `.env` to the correct value (e.g. `192.168.1.0/24`) and restart.

---

## Port conflicts

If another service already uses one of the exposed ports, change the **host** port (left side of the colon) in `docker-compose.yml`:

```yaml
ports:
  - 17878:7878   # Changed host port to 17878, container still uses 7878
```

---

## Full stack restart

```bash
docker compose down
docker compose up -d

# Watch logs during startup
docker compose logs -f
```
