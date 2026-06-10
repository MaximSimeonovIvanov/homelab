# slskd

## What It Does

slskd is a self-hosted Soulseek client. Soulseek is a peer-to-peer file sharing network focused on music — particularly useful for rare releases, live recordings, and music not available on public torrent indexers.

slskd runs as a server-side daemon with a web UI, so Soulseek is always running in the background on the homelab rather than requiring a desktop client open on your laptop.

---

## How It Complements the Torrent Stack

Lidarr + qBittorrent + Prowlarr handle mainstream music via public torrent indexers. slskd fills the gaps:

- Rare or obscure releases not indexed by torrent sites
- Live recordings and bootlegs
- Music only shared by specific users on the Soulseek network
- Faster access to niche content that has few or no torrent seeders

---

## Docker Network

slskd is on the `arrstack` network alongside qBittorrent, Lidarr, and Prowlarr.

---

## Compose File

See [`compose/slskd/docker-compose.yml`](../../compose/slskd/docker-compose.yml)

---

## Setup Steps

### 1. Deploy

```bash
cd /opt/docker/slskd
docker compose up -d
```

Navigate to `http://slskd.home` or `http://192.168.0.215:5030`.

### 2. Create a Soulseek account

If you don't have one, register at [https://www.slsknet.org/news/node/1](https://www.slsknet.org/news/node/1).

Soulseek operates on a social reciprocity model — users who share more get better download speeds from others. Configure a shared music folder in slskd to contribute back to the network.

### 3. Configure credentials

In slskd settings, enter your Soulseek username and password.

### 4. Set download and share directories

Configure paths for:
- **Downloads:** where slskd saves completed downloads
- **Shared:** the folder(s) you share with other Soulseek users (your music library)

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `slskd.home`      |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `5030`            |
| Block Exploits   | ✅ On             |

---

## Firewall

```bash
sudo ufw allow from 10.8.0.0/24 to any port 5030 comment "slskd VPN only"
```

slskd is VPN-only. The Soulseek protocol itself uses its own outbound connections managed by slskd — no additional firewall rules are needed for downloads to work.

---

## Notes

- slskd does not integrate with Lidarr — downloads are manual via the web UI
- After downloading, move files to your music library directory and trigger a Navidrome rescan (Full Scan if you want it to appear immediately)
- The Soulseek network is entirely peer-to-peer — availability depends on other users being online and sharing the files you want
