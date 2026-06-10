# Homarr

## What It Does

Homarr is a customisable home dashboard for your homelab. It gives you a single page showing all your services with clickable links, status indicators, and optional widgets (search bar, date/time, system stats).

It serves as the landing page for the homelab — instead of memorising IP:port combinations, everything is one click away from `http://homarr.home`.

---

## Compose File

See [`compose/homarr/docker-compose.yml`](../../compose/homarr/docker-compose.yml)

---

## Setup Steps

```bash
cd /opt/docker/homarr
docker compose up -d
```

Navigate to `http://homarr.home` or `http://192.168.0.215:7575`.

On first visit you will be prompted to create an admin account.

---

## Adding Services

In the Homarr UI:

1. Click **Edit mode** (pencil icon)
2. Click **Add a tile**
3. Fill in the service name, URL, and icon
4. Save

Useful tile settings:
- **URL:** use the `.home` domain so it works both locally and over VPN
- **Icon:** Homarr has a built-in icon library — search by service name

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `homarr.home`     |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `7575`            |
| Block Exploits   | ✅ On             |

---

## Firewall

```bash
sudo ufw allow from 10.8.0.0/24 to any port 7575 comment "Homarr VPN only"
```

Homarr is VPN-only — not accessible from the internet.

---

## Notes

- Config stored at `/opt/docker/homarr/configs/`
- Icons stored at `/opt/docker/homarr/icons/`
- Both directories should be bind-mounted in the compose file so config survives container updates
