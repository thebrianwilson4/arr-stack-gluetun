# VPN Provider Setup

This stack uses Gluetun with a **custom WireGuard** configuration, which works with any WireGuard-capable VPN provider. Here's how to get your credentials for the most common providers.

---

## ProtonVPN

1. Log in at [protonvpn.com](https://protonvpn.com)
2. Go to **Downloads → WireGuard configuration**
3. Select a server and download the config file
4. Extract the following into your `.env`:

```ini
# From [Interface] section:
PrivateKey  →  WIREGUARD_PRIVATE_KEY
Address     →  WIREGUARD_ADDRESSES

# From [Peer] section:
PublicKey   →  WIREGUARD_PUBLIC_KEY
Endpoint    →  WIREGUARD_ENDPOINT_IP (IP only, not the :port)
              →  WIREGUARD_ENDPOINT_PORT (the port after the colon, usually 51820)
```

> ⚠️ Regenerate your WireGuard config periodically. Treat `WIREGUARD_PRIVATE_KEY` like a password — never share it or commit it to git.

---

## Mullvad

1. Log in at [mullvad.net](https://mullvad.net)
2. Go to **Wireguard configuration** under your account
3. Generate a config for your preferred server
4. Extract values the same way as above

For Mullvad, set `VPN_SERVICE_PROVIDER=mullvad` in the compose file instead of `custom` and follow the [Gluetun Mullvad guide](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/mullvad.md).

---

## AirVPN

Use `VPN_SERVICE_PROVIDER=airvpn`. See [Gluetun AirVPN setup](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/airvpn.md).

---

## Other Providers

Gluetun supports many providers natively. Check the full list:
👉 [github.com/qdm12/gluetun-wiki](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers)

If your provider supports WireGuard, the `custom` provider type in this compose file will work with any of them.

---

## Verifying Your VPN is Active

After starting the stack, confirm all arr traffic routes through the VPN:

```bash
# Should return your VPN provider's IP, NOT your home IP
docker exec radarr curl -s https://ifconfig.me

# Check Gluetun health
docker inspect Gluetun --format='{{.State.Health.Status}}'
```

If it returns your real home IP, Gluetun is not working correctly — check `docker logs Gluetun`.
