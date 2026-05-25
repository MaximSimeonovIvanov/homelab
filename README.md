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
    └── (automation scripts)
```

---

## Guiding Principles

- **Privacy first** — self-hosted replacements for Google Drive, Google Photos
- **Documented and reproducible** — every step written up, no tribal knowledge
- **Security minded** — VPN over open ports, firewall enforced, no secrets committed
- **Educational** — built to understand the *why*, not just copy-paste

---

## Progress Log

| Date       | Milestone                                          |
|------------|----------------------------------------------------|
| 2026-05-23 | Repo initialized, planning done                    |
| 2026-05-23 | Ubuntu 26.04 LTS installed, server hardened        |
| 2026-05-23 | Docker installed, AdGuard + NPM deployed           |
| 2026-05-24 | WireGuard deployed, remote access confirmed        |
| 2026-05-24 | Nextcloud 33 + Immich deployed, all services live  |
| 2026-05-25 | Homarr, Portainer added                            |
| 2026-05-25 | Ntfy + Uptime Kuma deployed, monitoring live       |

---

## Author

**Maxim Simeonov Ivanov** — CS student building a homelab from scratch.  
[github.com/MaximSimeonovIvanov](https://github.com/MaximSimeonovIvanov)
