# Networking

## Overview

The network stack has three layers working together:

1. **AdGuard Home** — local DNS resolver. Resolves internal hostnames (e.g. `nextcloud.home`) and blocks ads/trackers network-wide.
2. **Nginx Proxy Manager** — internal reverse proxy. Routes `service.home` hostnames to the correct Docker containers with HTTPS.
3. **WireGuard** — VPN for remote access. No ports exposed to the internet except UDP 51820. Remote clients tunnel in and access services as if on the local network.

---

## DNS Flow (Local)

```
Device asks: "what is nextcloud.home?"
        │
   AdGuard Home (192.168.1.100:53)
        │ returns: 192.168.1.100 (the server itself)
        │
   Nginx Proxy Manager (port 80/443)
        │ routes based on hostname
        │
   Nextcloud container (internal Docker port)
```

---

## Remote Access Flow

```
Phone/laptop (outside home network)
        │
   WireGuard VPN (UDP 51820 → router → server)
        │ encrypted tunnel established
        │
   Now on the home network virtually
        │
   Same DNS + NPM flow as above
```

---

## IP Planning

| Host              | IP               | Notes                          |
|-------------------|------------------|--------------------------------|
| Router            | 192.168.1.1      | Default gateway                |
| Server (OptiPlex) | 192.168.1.100    | Static via DHCP reservation    |
| AdGuard Home      | 192.168.1.100:3000 (UI), :53 (DNS) | Runs on server |
| WireGuard subnet  | 10.8.0.0/24      | VPN clients get IPs here       |

> Update these IPs to match your actual network once confirmed.

---

## ISP / CGNAT Status

- **WAN IP matches router IP:** ✅ No CGNAT detected
- **Port forwarding available:** ✅ Yes
- **Remote access method:** WireGuard self-hosted (wg-easy)

---

## Firewall Rules (UFW)

| Port      | Protocol | Purpose              |
|-----------|----------|----------------------|
| 22        | TCP      | SSH                  |
| 51820     | UDP      | WireGuard VPN        |
| 80        | TCP      | NPM HTTP (local only)|
| 443       | TCP      | NPM HTTPS (local only)|

> Ports 80/443 should only be accessible from the local network, not forwarded through the router. WireGuard is the only port that needs router port forwarding.

---

## Local Domain: `.home`

All services are accessible via `.home` subdomains on the local network:

| URL                        | Service              |
|----------------------------|----------------------|
| `http://adguard.home`      | AdGuard Home UI      |
| `https://nextcloud.home`   | Nextcloud            |
| `https://immich.home`      | Immich               |
| `https://npm.home`         | Nginx Proxy Manager  |
| `https://wireguard.home`   | WireGuard UI         |

These are configured in AdGuard Home as DNS rewrites, all pointing to `192.168.1.100`.
