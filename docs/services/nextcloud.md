# Nextcloud

## What It Does

Nextcloud is a self-hosted file sync and collaboration platform — a replacement for Google Drive / Dropbox. It runs on your server and lets you sync files across devices, share them, and access them from anywhere via WireGuard.

In this homelab, Nextcloud handles **general file storage only**. Photos are managed by Immich.

---

## Stack

Nextcloud runs as two containers:

- `nextcloud` — the main application (PHP/Apache)
- `nextcloud-db` — MariaDB database

**Why MariaDB?** It's the officially recommended database for Nextcloud production installs. Best tested, best performance, most documentation.

---

## Compose File

See [`compose/nextcloud/docker-compose.yml`](../../compose/nextcloud/docker-compose.yml)

Copy `.env.example` to `.env` and fill in your values before deploying.

---

## Setup Steps

```bash
cd /opt/docker/nextcloud
cp .env.example .env
nano .env   # Fill in passwords
docker compose up -d
```

Navigate to `http://nextcloud.home` — login with the admin credentials from your `.env`.

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `nextcloud.home`  |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `8080`            |
| Websockets       | ✅ On             |
| Block Exploits   | ✅ On             |

---

## Notes

- Nextcloud version: 33.0.3
- Data stored at `/opt/docker/nextcloud/data` on the SSD for now — move to HDD mount when drive is added
- Trusted domain set to `nextcloud.home` via environment variable
