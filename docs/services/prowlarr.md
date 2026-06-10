# Prowlarr

## What It Does

Prowlarr is an indexer manager. It maintains connections to torrent indexers (sites that catalogue torrents) and provides a unified API that Lidarr can query. Instead of configuring each indexer separately in every arr application, you configure them once in Prowlarr and it distributes them.

---

## How It Fits Into the Music Stack

```
Lidarr wants to find a release
        │
   Prowlarr queries all configured indexers simultaneously
        │
   Returns results to Lidarr
        │
   Lidarr picks the best match and sends it to qBittorrent
```

---

## Docker Network

Prowlarr is on the `arrstack` network. Lidarr reaches it by container name: `http://prowlarr:9696`.

---

## Compose File

See [`compose/prowlarr/docker-compose.yml`](../../compose/prowlarr/docker-compose.yml)

---

## Setup Steps

### 1. Deploy

```bash
cd /opt/docker/prowlarr
docker compose up -d
```

Navigate to `http://prowlarr.home` or `http://192.168.0.215:9696`.

### 2. Add indexers

Go to **Indexers → Add Indexer** and search for the indexers you want to use.

For each indexer, test the connection before saving.

> Some indexers use Cloudflare bot protection and will fail with "blocked by CloudFlare" — these cannot be used with Prowlarr without additional tools. Skip them and use other indexers instead.

### 3. Connect to Lidarr

Go to **Settings → Apps → Add Application → Lidarr**

| Field      | Value                        |
|------------|------------------------------|
| Lidarr URL | `http://lidarr:8686`         |
| API Key    | From Lidarr → Settings → General |

Click Test, then Save. Prowlarr will attempt to sync indexers to Lidarr automatically.

### 4. Get the Prowlarr API key

You will need this when adding indexers directly in Lidarr via Torznab (see `lidarr.md`).

**Settings → General → API Key**

---

## Nginx Proxy Manager Config

| Field            | Value             |
|------------------|-------------------|
| Domain           | `prowlarr.home`   |
| Scheme           | `http`            |
| Forward Hostname | `192.168.0.215`   |
| Forward Port     | `9696`            |
| Block Exploits   | ✅ On             |

---

## Firewall

```bash
sudo ufw allow from 10.8.0.0/24 to any port 9696 comment "Prowlarr VPN only"
```

---

## Troubleshooting

**Indexers not syncing to Lidarr automatically:**
Prowlarr's automatic sync only pushes indexers to an app if the indexer has categories matching the app's purpose. General trackers may not have music categories mapped and will be silently excluded. The reliable fallback is to add indexers directly in Lidarr via Torznab — see `lidarr.md` for instructions.

**Indexer shows "blocked by CloudFlare":**
That indexer cannot be used. Try the Base URL dropdown for mirror URLs — some mirrors don't use Cloudflare. If none work, use a different indexer. For music specifically, private trackers (Redacted, Orpheus) are far superior to public ones and don't have this problem.

**Indexer failing for hours:**
Check if the indexer's website is down. If it is, wait. If it has been failing for days, remove it and add an alternative.
