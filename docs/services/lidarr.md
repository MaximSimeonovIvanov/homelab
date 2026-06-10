# Lidarr

## What It Does

Lidarr is a music library manager. You tell it which artists and albums you want, it searches indexers (via Prowlarr) for matching torrents, sends them to qBittorrent for downloading, and then imports the completed files into your music library.

It is the brain of the music stack — orchestrating search, download, and import automatically.

---

## How It Fits Into the Music Stack

```
You add an artist/album to Lidarr
        │
   Lidarr searches Prowlarr indexers for a matching release
        │
   Sends the best match to qBittorrent as a download job
        │
   Monitors qBittorrent for completion
        │
   Imports completed files to the music library directory
        │
   Navidrome picks them up on next scan
```

---

## Docker Network

Lidarr is on the `arrstack` network. It reaches qBittorrent as `http://qbittorrent:8090` and Prowlarr as `http://prowlarr:9696`.

---

## Compose File

See [`compose/lidarr/docker-compose.yml`](../../compose/lidarr/docker-compose.yml)

Key volume mounts:
- Music library: same path as Navidrome's music mount (e.g. `/mnt/ssd2/music` → `/music`)
- Downloads: same path as qBittorrent's download directory (e.g. `/downloads` → `/downloads`)
- Config: `/opt/docker/lidarr/config` → `/config`

> Lidarr and qBittorrent must share the same download directory path, otherwise Lidarr can't find the files after qBittorrent completes.

---

## Setup Steps

### 1. Deploy

```bash
cd /opt/docker/lidarr
docker compose up -d
```

Navigate to `http://lidarr.home` or `http://192.168.0.215:8686`.

### 2. Add qBittorrent as a download client

**Settings → Download Clients → Add → qBittorrent**

| Field    | Value                  |
|----------|------------------------|
| Host     | `qbittorrent`          |
| Port     | `8090`                 |
| Username | `admin`                |
| Password | your qBittorrent password |

Click Test. A green tick means the connection works. If Lidarr shows a warning about "torrent removal on ratio", go to qBittorrent → Tools → Options → BitTorrent and disable the ratio limit — Lidarr treats this as a validation error and won't save the client until it's fixed.

### 3. Add indexers via Torznab

Prowlarr can sync indexers to Lidarr automatically, but this is unreliable for music-specific categories. The reliable method is to add each indexer manually via Torznab.

**Settings → Indexers → Add → Torznab**

| Field      | Value                                              |
|------------|----------------------------------------------------|
| URL        | `http://prowlarr:9696/{indexer_id}/api`            |
| API Key    | Prowlarr API key (Settings → General → API Key)    |
| Categories | `3000` (Audio) and/or `100101` (Music)             |

To find an indexer's ID: Prowlarr → Indexers → click the indexer → the ID is in the URL or the Details tab.

### 4. Set a quality profile

**Settings → Quality Profiles**

| Profile    | What it accepts                              |
|------------|----------------------------------------------|
| Standard   | MP3 at various bitrates (128–320 kbps) — recommended default |
| Lossless   | FLAC only                                    |
| Any        | Everything including low quality — avoid     |

Quality profiles are set per-artist in Lidarr. If you download a FLAC file but the artist is set to Standard, Lidarr will not import it. Use Manual Import for one-off exceptions.

### 5. Add artists

**Library → Add Artist**

When adding an artist:
- Set **Monitor** to **None** initially — "All Albums" will immediately queue the entire discography for download
- Add the artist, then go into their page and toggle monitoring only for the specific albums you want
- Trigger a manual search for those albums only

> Always manage per-album monitoring manually. Selecting "All Albums" on a prolific artist will queue dozens of albums and fill your storage.

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `lidarr.home`     |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `8686`            |
| Block Exploits   | ✅ On             |

---

## Firewall

```bash
sudo ufw allow from 10.8.0.0/24 to any port 8686 comment "Lidarr VPN only"
```

---

## Troubleshooting

**Download client test fails:**
Check the Docker network first — both containers must be on `arrstack`. Then check the ratio limit setting in qBittorrent (see Step 2). Then check the IP whitelist in qBittorrent covers `172.0.0.0/8`.

**Files downloaded but not imported:**
Lidarr will ignore files that don't match the artist's quality profile. Go to **Library → Unmapped Files** to manually import them, or temporarily change the artist profile to "Any".

**Lidarr stops importing after a qBittorrent restart:**
Re-test the download client connection: **Settings → Download Clients → test**. A restart breaks the connection and Lidarr won't detect completed downloads until it is re-established.

**Deleting an album:**
Always delete through Lidarr — go to the artist page, click the album, Delete, tick "Delete files". Never delete music files manually from the filesystem. Lidarr will see them as "missing" and re-download them at the next search cycle.
