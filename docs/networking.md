# Networking

## Overview

The network stack has three layers working together:

1. **AdGuard Home** — local DNS resolver. Resolves internal hostnames (e.g. `nextcloud.home`) and blocks ads/trackers on configured devices.
2. **Nginx Proxy Manager** — internal reverse proxy. Routes `service.home` hostnames to the correct Docker containers.
3. **WireGuard** — VPN for remote access. No private services are exposed to the internet. Remote clients tunnel in and access services as if on the local network.

---

## DNS Flow (Local)

```
Device asks: "what is nextcloud.home?"
        │
   AdGuard Home (192.168.0.215:53)
        │ returns: 192.168.0.215 (the server itself)
        │
   Nginx Proxy Manager (port 80)
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

| Host              | IP                                   | Notes                       |
|-------------------|--------------------------------------|-----------------------------|
| Router            | 192.168.0.1                          | Default gateway             |
| Server (OptiPlex) | 192.168.0.215                        | Static via DHCP reservation |
| AdGuard Home      | 192.168.0.215:3000 (UI), :53 (DNS)   | Runs on server              |
| WireGuard subnet  | 10.8.0.0/24                          | VPN clients get IPs here    |
| arrstack network  | 172.31.0.0/16                        | Music stack Docker network  |
| wg-easy network   | 172.19.0.0/16                        | WireGuard Docker network    |

---

## ISP / CGNAT Status

- **WAN IP matches router IP:** ✅ No CGNAT detected
- **Port forwarding available:** ✅ Yes
- **Remote access method:** WireGuard self-hosted (wg-easy)

---

## Firewall Rules (UFW)

Default policy: deny incoming, allow outgoing, allow routed (forward).

| Rule | Port/Protocol | Action   | From                | Purpose                          |
|------|---------------|----------|---------------------|----------------------------------|
| 1    | 22/tcp        | ALLOW IN | Anywhere            | SSH                              |
| 2    | 51820/udp     | ALLOW IN | Anywhere            | WireGuard VPN                    |
| 3    | 80/tcp        | ALLOW IN | Anywhere            | HTTP (NPM + Cloudflare)          |
| 4    | 443/tcp       | ALLOW IN | Anywhere            | HTTPS (NPM + Cloudflare)         |
| 5    | 9000          | ALLOW IN | 10.8.0.0/24         | Portainer — VPN only             |
| 6    | 7575          | ALLOW IN | 10.8.0.0/24         | Homarr — VPN only                |
| 7    | 5555          | ALLOW IN | 10.8.0.0/24         | Ntfy — VPN only                  |
| 8    | 3001          | ALLOW IN | 10.8.0.0/24         | Uptime Kuma — VPN only           |
| 9    | 51821         | ALLOW IN | 10.8.0.0/24         | WireGuard UI — VPN only          |
| 10   | 3000          | ALLOW IN | 10.8.0.0/24         | AdGuard UI — VPN only            |
| 11   | 53            | ALLOW IN | 192.168.0.0/24      | AdGuard DNS — local network      |
| 12   | 53            | ALLOW IN | 172.19.0.0/16       | AdGuard DNS — wg-easy container  |
| 13   | 2222/tcp      | ALLOW IN | Anywhere            | SSH — GitHub Actions deployments |
| 14   | 4533          | ALLOW IN | 10.8.0.0/24         | Navidrome — VPN only             |
| 15   | 8090          | ALLOW IN | 10.8.0.0/24         | qBittorrent WebUI — VPN only     |
| 16   | 9696          | ALLOW IN | 10.8.0.0/24         | Prowlarr — VPN only              |
| 17   | 8686          | ALLOW IN | 10.8.0.0/24         | Lidarr — VPN only                |
| 18   | 5030          | ALLOW IN | 10.8.0.0/24         | slskd — VPN only                 |

**Why rules 11 and 12 both exist for port 53:**
wg-easy runs inside a Docker container. When a VPN client sends a DNS query to AdGuard, wg-easy NATs it through its own Docker IP (`172.19.0.x`) before it reaches the host. UFW sees the query coming from the Docker network, not from the VPN subnet. Rule 11 covers devices on the local network; rule 12 covers VPN clients routing through the wg-easy container.

---

## Local Domain: `.home`

All services are accessible via `.home` subdomains on the local network and over VPN. These are configured in AdGuard Home as DNS rewrites, all pointing to `192.168.0.215`.

| Domain              | Service              | Port  |
|---------------------|----------------------|-------|
| `adguard.home`      | AdGuard Home UI      | 3000  |
| `npm.home`          | Nginx Proxy Manager  | 81    |
| `nextcloud.home`    | Nextcloud            | 8080  |
| `immich.home`       | Immich               | 2283  |
| `wireguard.home`    | WireGuard UI         | 51821 |
| `portainer.home`    | Portainer            | 9000  |
| `homarr.home`       | Homarr               | 7575  |
| `ntfy.home`         | Ntfy                 | 5555  |
| `uptime.home`       | Uptime Kuma          | 3001  |
| `music.home`        | Navidrome            | 4533  |
| `qbit.home`         | qBittorrent WebUI    | 8090  |
| `prowlarr.home`     | Prowlarr             | 9696  |
| `lidarr.home`       | Lidarr               | 8686  |
| `slskd.home`        | slskd                | 5030  |

All URLs are `http://` — no SSL configured yet for local services. See `known-limitations.md`.

---

## Public-Facing Site (sim-obleklo.bg)

This server also hosts a production e-commerce site proxied through Cloudflare. This is why ports 80 and 443 are open to the internet — they are required for Cloudflare to reach NPM.

```
Browser → Cloudflare (HTTPS) → NPM (ports 80/443) → application containers
```

NPM routes traffic by domain name: `sim-obleklo.bg` goes to the Next.js frontend container, `api.sim-obleklo.bg` goes to the Django backend container. SSL certificates are issued by Let's Encrypt via Cloudflare DNS challenge and terminate at NPM.

For full details see the [workwear catalog repository](https://github.com/MaximSimeonovIvanov/workwear-catalog).

---

## Docker Networks

| Network           | Subnet         | Members                                      |
|-------------------|----------------|----------------------------------------------|
| `arrstack`        | 172.31.0.0/16  | qbittorrent, prowlarr, lidarr, slskd         |
| `wireguard_default` | 172.19.0.0/16 | wg-easy                                     |
| `navidrome_default` | isolated      | navidrome (reads music from disk, no peers) |
| host              | —              | adguard (network_mode: host)                |

Inter-container communication within `arrstack` uses container names (e.g. `http://qbittorrent:8090`), not IP addresses. Docker resolves container names automatically within a shared network.
