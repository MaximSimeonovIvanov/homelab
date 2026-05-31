# 🏠 Homelab

A self-hosted home server built on a Dell OptiPlex 7040 SFF, running Ubuntu Server 26.04 LTS. This repository documents the full setup — from bare metal to a running stack of privacy-first, self-hosted services.

Built as a learning project and portfolio piece. Everything is reproducible from this repo.

---

## Architecture Overview

```
                        [ Internet ]
                             │
                     WireGuard VPN (UDP 51820)
                             │
                    [ Dell OptiPlex 7040 ]
                    Ubuntu Server 26.04 LTS
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
   [AdGuard]          [Nginx Proxy]         [WireGuard]
   DNS + Ads           Rev. Proxy           Remote Access
         │                   │
   ┌─────┴──────┐            │
   │            │            │
[Nextcloud]  [Immich]        │
File Sync   Photo Mgmt       │
         │                   │
         ├───────────────────┤
         │                   │
   [Portainer]           [Homarr]
   Docker GUI            Dashboard
         │                   │
         ├───────────────────┤
         │                   │
    [Uptime Kuma]         [Ntfy]
     Monitoring        Push Alerts
         │                   │
         └───────────────────┘
               Docker Network
```

---

## Hardware

| Component    | Detail                          |
|--------------|---------------------------------|
| Machine      | Dell OptiPlex 7040 SFF          |
| CPU          | Intel Core i5-6500 (4C/4T)      |
| RAM          | 16 GB DDR4                      |
| Boot Drive   | 128 GB SSD                      |
| Data Storage | HDD (to be added)               |
| OS           | Ubuntu Server 26.04 LTS         |

---

## Services

| Service             | Purpose                          | Status     |
|---------------------|----------------------------------|------------|
| AdGuard Home        | DNS resolver + network ad block  | ✅ Running |
| Nginx Proxy Manager | Internal reverse proxy + HTTPS   | ✅ Running |
| WireGuard (wg-easy) | VPN for remote access            | ✅ Running |
| Nextcloud           | File sync (Google Drive alt.)    | ✅ Running |
| Immich              | Photo management (Google Photos) | ✅ Running |
| Portainer           | Docker GUI                       | ✅ Running |
| Homarr              | Home dashboard                   | ✅ Running |
| Ntfy                | Self-hosted push notifications   | ✅ Running |
| Uptime Kuma         | Service monitoring + alerting    | ✅ Running |

---

## Repository Structure

```
homelab/
├── README.md                  # This file
├── docs/
│   ├── hardware.md            # Hardware details and notes
│   ├── os-setup.md            # Ubuntu Server install + hardening
│   ├── networking.md          # Network architecture, DNS, VPN
│   └── services/
│       ├── adguard.md
│       ├── nginx-proxy-manager.md
│       ├── wireguard.md
│       ├── nextcloud.md
│       ├── immich.md
│       ├── portainer.md
│       ├── homarr.md
│       ├── ntfy.md
│       └── uptime-kuma.md
├── compose/
│   ├── adguard/
│   ├── nginx-proxy-manager/
│   ├── wireguard/
│   ├── nextcloud/
│   ├── immich/
│   ├── portainer/
│   ├── homarr/
│   ├── ntfy/
│   └── uptime-kuma/
└── scripts/
    └── (automation scripts)   # To be added
```

---

## Networking Architecture

### DNS Flow

AdGuard Home handles all DNS resolution. It runs with `network_mode: host` so it binds directly to the server's network interfaces on port 53.

Devices use AdGuard as their DNS server, configured manually per device (not via router — the router is shared). AdGuard resolves `.home` domain names to `192.168.0.215` via DNS rewrites, enabling clean local URLs like `nextcloud.home` instead of raw IP:port addresses.

```
Device DNS query → AdGuard (192.168.0.215:53) → Quad9 DoH upstream → answer
Device DNS query → AdGuard → DNS rewrite → 192.168.0.215 (for *.home domains)
```

### VPN + DNS — an important detail

WireGuard runs inside the `wg-easy` Docker container. When a VPN client sends a DNS query, it flows like this:

```
VPN client (10.8.0.2) → wg-easy container → Docker bridge (172.19.0.x) → AdGuard
```

Because wg-easy NATs all traffic through its own Docker IP before it reaches the host, the UFW firewall sees DNS queries arriving from `172.19.0.x`, not from `10.8.0.x`. The UFW rule allowing DNS from VPN clients must therefore target the wg-easy Docker network (`172.19.0.0/16`), not the VPN subnet (`10.8.0.0/24`).

This is a non-obvious consequence of running WireGuard in a container rather than natively on the host.

### Split Tunnel

VPN clients are configured with split tunnel — only traffic destined for the home network (`192.168.0.0/24`) and VPN subnet (`10.8.0.0/24`) goes through the tunnel. Regular internet traffic goes directly through the client's local connection. This keeps latency low for normal browsing while still allowing access to all homelab services remotely.

### Firewall (UFW)

UFW is configured with a strict default-deny incoming policy. All private services (Portainer, AdGuard UI, Homarr, etc.) are restricted to VPN clients only (`10.8.0.0/24`). Only ports 22 (SSH), 80 (HTTP), 443 (HTTPS), and 51820 (WireGuard) are open to the internet.

```
Internet-facing:  22/tcp, 80/tcp, 443/tcp, 51820/udp
VPN-only:         3000, 3001, 5555, 7575, 9000, 51821
DNS:              53 — local network + wg-easy Docker network only
```

---

## Guiding Principles

- **Privacy first** — self-hosted replacements for Google Drive, Google Photos
- **Documented and reproducible** — every step written up, no tribal knowledge
- **Security minded** — VPN over open ports, firewall enforced, no secrets committed
- **Educational** — built to understand the *why*, not just copy-paste

---

## Lessons Learned

Real problems encountered and understood during this build. Not just what broke, but why.

**Docker containers and host networking are separate.**
Services running inside Docker containers communicate through Docker bridge networks, not the host's network interfaces. When wg-easy forwards VPN traffic to AdGuard, it uses its own Docker IP as the source address. Firewall rules targeting the VPN subnet (`10.8.0.0/24`) never match — you have to target the container's Docker network instead. Understanding this required tracing packets with `tcpdump` to see what IP the firewall actually saw.

**`network_mode: host` exists for a reason.**
AdGuard needs to listen on port 53 on the server's real IP. Running it in a normal Docker network would isolate it — other containers and the host itself couldn't reach it for DNS. `network_mode: host` bypasses Docker networking entirely and makes the container part of the host's network stack. This is one of the few legitimate use cases for it.

**systemd-resolved is the glue between wg-quick and DNS.**
When WireGuard connects, `wg-quick` applies the `DNS =` setting from the config by calling `resolvectl` (part of systemd-resolved). If systemd-resolved is disabled, wg-quick silently skips DNS setup — WireGuard comes up but DNS never switches. The symptom is that everything works when you manually specify a DNS server but fails with normal browsing. `/etc/resolv.conf` must be a symlink to systemd-resolved's stub resolver, not a static file.

**UFW's `DEFAULT_FORWARD_POLICY` and containerised WireGuard.**
WireGuard in a Docker container using `WG_IPTABLES_BACKEND=nftables` manages its own NAT and routing rules inside the container via nftables. This completely bypasses UFW's forward policy. Setting `DEFAULT_FORWARD_POLICY=DROP` in UFW does not affect WireGuard traffic in this setup — but it should still be set to `ACCEPT` for correctness and to avoid silently blocking any future service that needs packet forwarding at the host level.

---

## Progress Log

| Date       | Milestone                                                        |
|------------|------------------------------------------------------------------|
| 2026-05-23 | Repo initialized, planning done                                  |
| 2026-05-23 | Ubuntu 26.04 LTS installed, server hardened                      |
| 2026-05-23 | Docker installed, AdGuard + NPM deployed                         |
| 2026-05-24 | WireGuard deployed, remote access confirmed                      |
| 2026-05-24 | Nextcloud 33 + Immich deployed, all services live                |
| 2026-05-25 | Homarr, Portainer added                                          |
| 2026-05-25 | Ntfy + Uptime Kuma deployed, monitoring live                     |
| 2026-05-31 | Full network audit — DNS, UFW, WireGuard stack debugged and fixed |

---

## Author

**Maxim Simeonov Ivanov** — CS student building a homelab from scratch.  
📧 [maksimivanov@tutamail.com](mailto:maksimivanov@tutamail.com)  
[github.com/MaximSimeonovIvanov](https://github.com/MaximSimeonovIvanov)
