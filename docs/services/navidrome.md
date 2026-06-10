# Navidrome

## What It Does

Navidrome is a self-hosted music streaming server — a replacement for Spotify. It reads a music library from disk and streams it to any device via a web interface or compatible mobile app.

It implements the Subsonic API, which means it works with a large ecosystem of existing clients (DSub, Symfonium, Ultrasonic on Android; Amperfy on iOS; Sublime Music on desktop).

---

## How It Fits Into the Music Stack

Navidrome is the **playback layer** only. It does not download or manage music — that is handled by Lidarr and qBittorrent. Navidrome simply reads whatever files exist in the music directory and makes them streamable.

```
Lidarr → qBittorrent downloads → /mnt/ssd2/music/
                                         │
                                    Navidrome reads
                                         │
                                  Stream to any device
```

---

## Compose File

See [`compose/navidrome/docker-compose.yml`](../../compose/navidrome/docker-compose.yml)

Key volume mounts:
- Music library: bind mount your music folder (e.g. `/mnt/ssd2/music`) to `/music` inside the container (read-only)
- Data directory: bind mount for Navidrome's database and cache (e.g. `/opt/docker/navidrome/data` to `/data`)

---

## Setup Steps

### 1. Pre-create the data directory

```bash
mkdir -p /opt/docker/navidrome/data
sudo chown -R 1000:1000 /opt/docker/navidrome/data
```

> Docker would create this directory automatically if you skipped this step, but it would be owned by `root`. Navidrome runs as user `1000:1000` and would fail with a permission denied error on startup. Always pre-create and chown data directories for containers that run as non-root.

### 2. Deploy

```bash
cd /opt/docker/navidrome
docker compose up -d
```

### 3. Create your admin account

Navigate to `http://music.home` or `http://192.168.0.215:4533`.

On first visit, Navidrome will prompt you to create an admin username and password. Save these in your password manager.

### 4. Wait for the library scan

Navidrome automatically scans the music directory on startup. The first scan may take a few minutes depending on library size. Progress is visible in the web UI.

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `music.home`      |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `4533`            |
| Block Exploits   | ✅ On             |

---

## Firewall

```bash
sudo ufw allow from 10.8.0.0/24 to any port 4533 comment "Navidrome VPN only"
```

Navidrome is VPN-only. Access it over WireGuard when away from home.

---

## Mobile App Setup

Navidrome works with any Subsonic-compatible app. Recommended:

**Android:** DSub or Symfonium
**iOS:** Amperfy

Setup in the app:
1. Connect to WireGuard VPN
2. Server URL: `http://music.home` (or `http://192.168.0.215:4533` if DNS isn't working)
3. Username and password: your Navidrome credentials
4. Test connection

---

## Library Rescans

Navidrome rescans the music directory on a schedule. To trigger a manual rescan:

- Web UI → top right user icon → **Rescan library**
- Choose **Quick Scan** to pick up new files only
- Choose **Full Scan** to also remove database entries for deleted files (always use Full Scan after deleting music)

---

## Notes

- Navidrome runs on its own Docker network (`navidrome_default`), isolated from the `arrstack` network. This is correct — Navidrome only needs to read files from disk, it does not need to communicate with Lidarr or qBittorrent directly.
- Data (database, cache, transcoding) stored at `/opt/docker/navidrome/data` — lightweight, no need to move to HDD
- Music library should be moved to the HDD when it is installed (`/mnt/data/music`). Update the volume mount in the compose file accordingly.
