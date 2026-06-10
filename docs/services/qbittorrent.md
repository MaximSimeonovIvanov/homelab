# qBittorrent

## What It Does

qBittorrent is the download engine for the music stack. It receives torrent jobs from Lidarr, downloads the files, and places them in a directory where Lidarr can pick them up, verify them, and import them into the music library.

It is controlled entirely by Lidarr — you rarely need to interact with the qBittorrent UI directly.

---

## How It Fits Into the Music Stack

```
Lidarr finds a release → sends torrent to qBittorrent → qBittorrent downloads
        → Lidarr detects completion → imports to /mnt/ssd2/music/ → Navidrome reads
```

---

## Docker Network

qBittorrent is on the `arrstack` network along with Lidarr and Prowlarr. This allows Lidarr to reach qBittorrent by container name (`http://qbittorrent:8090`) rather than IP address.

```bash
# If the network doesn't exist yet
docker network create arrstack
```

---

## Compose File

See [`compose/qbittorrent/docker-compose.yml`](../../compose/qbittorrent/docker-compose.yml)

---

## Setup Steps

### 1. Deploy

```bash
cd /opt/docker/qbittorrent
docker compose up -d
```

### 2. Get the temporary password

On first start, qBittorrent generates a temporary password and prints it to the logs:

```bash
docker logs qbittorrent | grep "temporary password"
```

Log in at `http://qbit.home` or `http://192.168.0.215:8090` with username `admin` and the temporary password.

### 3. Set a permanent password immediately

**Tools → Options → Web UI → Password** — set a strong permanent password and save.

> If you restart the container before setting a permanent password, the temporary password changes and you are locked out. Set it on first login without exception.

### 4. Configure for Lidarr compatibility

In **Tools → Options → BitTorrent → Seeding Limits**:
- **Untick "When ratio reaches"** entirely

> Lidarr treats ratio-based torrent removal as a validation error and will refuse to save qBittorrent as a download client if this is enabled. Lidarr manages cleanup itself after importing files — this setting is not needed.

### 5. Configure the IP whitelist

In **Tools → Options → Web UI**:
- **Bypass authentication for clients on localhost and whitelisted IPs:** `172.0.0.0/8`

> This allows Lidarr (on the `arrstack` network) to communicate with qBittorrent without authentication errors. `172.0.0.0/8` covers all Docker internal networks.

### 6. Set download paths

In **Tools → Options → Downloads**:
- Default save path: your downloads directory (e.g. `/downloads`)
- Keep completed downloads in: a `complete` subfolder

These paths must match the volume mounts in the compose file and the paths configured in Lidarr.

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `qbit.home`       |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `8090`            |
| Block Exploits   | ✅ On             |

---

## Firewall

```bash
sudo ufw allow from 10.8.0.0/24 to any port 8090 comment "qBittorrent WebUI VPN only"
```

The torrent port (6881) does not need a UFW rule — Docker handles the port mapping directly.

---

## Troubleshooting

**Lidarr can't connect to qBittorrent:**
Check in order:
1. Both containers are on the `arrstack` network — `docker network inspect arrstack`
2. The ratio limit setting is disabled (see Step 4)
3. The IP whitelist includes `172.0.0.0/8` (see Step 5)
4. qBittorrent hasn't banned Lidarr's IP from failed login attempts — restart qBittorrent to clear the ban: `docker restart qbittorrent`

**Torrents show 0B size with hash names:**
qBittorrent added the magnet link but couldn't find peers to fetch metadata. Right-click → Force Reannounce. If still stuck after a few minutes, the torrent likely has no seeders — remove it and search for a better-seeded one in Lidarr via Interactive Search.

**Downloads complete but Lidarr doesn't import:**
Re-test the download client connection in Lidarr (**Settings → Download Clients → test**). A qBittorrent restart breaks the connection and Lidarr won't detect completed downloads until the connection is re-established.

**All torrents paused after a container restart:**
In qBittorrent, select all torrents → right-click → Resume.
